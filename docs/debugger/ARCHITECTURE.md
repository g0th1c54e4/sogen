# 集成调试器 — 架构

状态：第 1 阶段（核心抽象 + 设计）。增量实现；本文档是后续阶段所依据的契约。

## 目标

一个一流的、运行在浏览器内的调试器，它操作的是**真实**的模拟器执行引擎（通过 `windows_emulator` 的 Unicorn/WHP/icicle/KVM），而不是只有前端的 mock。UX 风格类似 IDA/x64dbg/Ghidra：反汇编、寄存器、堆栈、内存、断点、线程、调用栈、模块、CFG，以及一个 Python 控制台。

## 复用地图（不要重复造轮子）

| 关注点 | 复用的现有系统 |
|---|---|
| 单步执行 | `windows_emulator::start(1)`（见 `x86_64_gdb_stub_handler::singlestep`） |
| 断点 | `x86_64_emulator::hook_memory_execution(addr, cb)` + RAII `scoped_hook`（见 `x86_64_gdb_stub_handler::set_breakpoint`） |
| 反汇编 | `disassembler`（Capstone；Intel/AT&T 句柄；`resolve_jump_target`） |
| 寄存器 | `windows_emulator::emu().read_register/write_register`、`x86_register` |
| 内存 | `memory_manager::try_read_memory`、`get_reserved_regions`（已经由 `GetMemoryRegions` 暴露） |
| 模块 | `module_manager::modules()`、`find_name`、`mapped_module` |
| 线程 / PC | `process_context::threads`、`active_thread`、`current_thread().current_ip` |
| 传输 | FlatBuffers 事件通道 `events.fbs` ↔ `event_handler.cpp` |
| 事件泵（浏览器） | `analysis.cpp` 每 0x20000 条指令调用一次 `debugger::handle_events` |
| 客户端桥 | `page/src/emulator.ts` worker + id/tombstone 请求关联 |
| 脚本模型 | 现有的 nanobind `sogen` Python API（`src/python-bindings`） |

## 组件

```
                 ┌──────────────────────── browser ────────────────────────┐
 React debugger   │  Disasm | Regs | Stack | Mem | BP | Threads | CallStack   │
   UI panels      │  Modules | CFG (React Flow) | Python (Monaco)            │
        │        └───────────────┬──────────────────────────────────────────┘
        │  typed wrappers        │ Emulator.debugCommand(kind, json) -> Promise
        ▼                        ▼
  page/src/debugger/*  ──>  emulator.ts  ──(postMessage)──>  emulator-worker.js
                                                                  │ FlatBuffers
                                                                  ▼
                       event_handler.cpp  ──dispatch──>  debug_session (core)
                                                                  │
                              windows_emulator (emu/memory/modules/threads)
```

- **`debug_session`（核心，`src/debugger/debug_session.hpp`）** —— 一等子系统。拥有断点注册表、步进引擎和自省能力。包装一个 `windows_emulator&`。后端无关：只使用所有模拟器后端都实现的 `emu()` 接口，因此 Unicorn/WHP/icicle/KVM 行为一致。
- **通用命令协议** —— 不为每个功能做定制的 FlatBuffers 表，而是做一个可扩展的表对：
  - `DebugCommandRequest { id:uint32; kind:uint32; payload:[ubyte] }`
  - `DebugCommandResponse { id:uint32; ok:bool; payload:[ubyte] }`
  `payload` 是 UTF-8 JSON（与 `GetMemoryRegionsResponse` 相同的成熟做法）。新的调试器功能 = 新的 `kind` + JSON schema；**无需重新生成 FlatBuffers**。
- **`event_handler.cpp` 分发** —— 解码 `DebugCommandRequest`，把 `kind` 路由到 `debug_session` 方法，把 JSON 结果序列化进具有相同 `id` 的 `DebugCommandResponse`。

## 命令种类（JSON 子协议）

```
0  get_registers      -> { rip, registers:{name:value(hex)} }
1  disassemble {address,count} -> { insns:[{address,bytes,mnemonic,op,symbol,branch}] }
2  get_modules        -> { modules:[{name,base,size,entry}] }
3  get_threads        -> { threads:[{id,ip,active}] }
4  get_callstack {thread?} -> { frames:[{ip,sp,symbol,module}] }
5  set_breakpoint {address,type} -> { id }
6  clear_breakpoint {address}    -> { ok }
7  list_breakpoints   -> { breakpoints:[{address,type,enabled}] }
8  step_into          -> async stop event
9  step_over          -> async stop event
10 step_out           -> async stop event
11 run_to {address}    -> async stop event
12 continue / pause / restart
13 read_memory {address,size}    (already covered by ReadMemoryRequest)
```

异步停止复用现有的暂停/状态机制（`GetStateResponse`、`EmulationStatus`），因此 UI 更新是事件驱动且低延迟的。

## 事件流

1. UI 动作 → `Emulator.debugCommand(kind, json)` → 唯一 `id`、pending map
   （tombstone FIFO，与 `readMemory` 关联完全相同）。
2. Worker 转发 FlatBuffers `DebugCommandRequest`。
3. `handle_events`（由模拟循环驱动）在暂停时排空它；`event_handler` 分发到 `debug_session`。
4. 响应 `DebugCommandResponse{id,...}` → worker → `emulator.ts` 按 `id` 兑现 promise。
5. 执行状态变化（命中断点、步进完成）翻转现有的 `emulation_state` → 现有的 `GetStateResponse` 路径 → React 重新渲染。

## 步进实现（第 3 阶段契约）

- **step_into**：`emu().start(1)` —— 一条指令，然后暂停。成熟路径（`gdb_stub_handler::singlestep`）。线程感知：步进应用于 `current_thread()`。
- **step_over**：通过 `disassembler` 解码 RIP 处的指令。如果是 `call`（或带 `rep` 前缀），在 `RIP + insn.size` 处设置**临时**执行钩子，`continue`，命中时自动移除。否则 == step_into。
- **step_out**：从栈帧读取返回地址（序言后的 `[RSP]` / 通过调用栈遍历），在那里设临时断点，`continue`。
- **run_to**：在目标处设临时断点，`continue`。
- **不失步规则**：步进绝不修改持久的模拟器状态；临时钩子是 RAII（`scoped_hook`），在报告暂停之前移除。所有步进决策都从一个已暂停的快照做出。

## 断点生命周期

```
set_breakpoint(addr)
  └─> debug_session stores breakpoint{addr,type,enabled}
      └─> scoped_hook = emu().hook_memory_execution(addr, on_hit)
on_hit (during emulation):
  └─> mark session paused -> emulation loop's handle_events blocks
      └─> emit state change -> UI shows stop @ addr
clear_breakpoint(addr) -> erase entry -> scoped_hook dtor removes emu hook
```

断点保存在 `utils::concurrency::container` 中（与 gdb stub 相同），以便模拟线程和命令处理程序之间异步安全访问。

## 状态同步

- 唯一事实来源 = 实时的 `windows_emulator`。UI 从不缓存权威状态；每个面板都通过针对当前暂停快照的命令读取。
- 一个单调的 `snapshot generation`（内存视图已在用）在每次新的暂停时递增，使面板/缓存一致失效。
- 所有自省都是**只读**的，且只在暂停时有效；运行期间发出的命令被拒绝（`ok:false`）—— 绝不阻塞模拟。

## 未来扩展点

`debug_session` 是以下功能接缝：污点追踪（钩子驱动）、轨迹记录（指令钩子环形缓冲）、时间旅行（周期性 `memory_manager` 快照 + 重放）、符号执行（解码流馈入表达式引擎）。它们都不需要协议变更 —— 只需要新的命令 `kind`。

## Python 脚本集成

浏览器的 Python 标签页不会分叉 API：它通过命令 `kind` 驱动**同一个** `debug_session`，暴露一个 `emu.debug.*` 门面，其名称/行为镜像现有的 nanobind `sogen` 模型（`emu.debug.breakpoint/step_into/step_over/continue_execution`、`emu.debug.registers.rip`、`emu.debug.disassemble(addr,n)`）。执行沙箱、输出流、traceback 和取消是专门的阶段；底层（通用命令通道 + `debug_session`）被设计为让 Python 层是一个瘦客户端，而不是一个并行的引擎。

## 阶段计划

1. **[done]** 核心抽象 + 本文档。
2. **[done]** 通用 `DebugCommand` 协议 + 分发 + 只读自省（寄存器/反汇编/模块/线程/调用栈）。
3. **[done]** 断点 + 步进引擎（into/over/out/run-to），通过持久控制钩子实现事件驱动停止。
4. **[done]** Web UI 调试器外壳 + 面板 + 热键（F5/F10/F11/Shift+F11/Ctrl+G）。
5. **[done]** CFG 视图（自包含 SVG，从解码块增量构建）。
6. **[done]** Monaco 编辑器脚本标签页 + 通往真实 `debug_session` 的 `emu.debug.*` 桥。

## 脚本标签页 — 诚实的范围说明

脚本标签页使用 Monaco 编辑器，并暴露一个 `emu` 对象，其方法名称/行为镜像现有的 nanobind `sogen` 模型（`emu.debug.breakpoint/step_into/step_over/continue_execution`、`emu.debug.registers()`、`emu.debug.disassemble(addr, n)`、`emu.memory.read`）。每次调用都通过同一个通用命令通道 / `debug_session` 驱动**真实的**模拟器 —— 它不是 mock。

宿主语言是 JavaScript（异步），而不是 CPython：在浏览器中运行真实的 nanobind `sogen` 模块需要一个 WASM 中的 Python 运行时（pyodide + nanobind-wasm）。该运行时是未来的即插即用方案，挂载在*完全相同*的 `emu` 门面之后 —— 不需要协议、`debug_session` 或 UI 变更。这使集成在今天就是真实且正确的，而不是交付一个假的 Python 解释器。
