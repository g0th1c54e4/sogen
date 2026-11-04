<p align="center">
  <a href="https://github.com/momo5502/sogen"><img src="https://momo5502.com/sogen/sogen.svg" height="200" alt="Sogen" /></a>
</p>

Sogen 为 Windows 和 Linux 用户空间模拟暴露了 Python 绑定。Python API 用于脚本化运行、构建小型分析辅助工具，以及快速迭代回调；Windows 绑定还暴露了用于更深入分析的钩子。

从 PyPI 安装：

```bash
pip install sogen
```

项目链接：

- PyPI: https://pypi.org/project/sogen/
- 仓库: https://github.com/momo5502/sogen
- 现成的模拟根目录: https://sogen.dev/root.zip

## 你需要什么

ARM64 可执行文件使用相同的 API，配一个 ARM64 模拟根目录；见 `examples/python/arm64_application.py`。

Python 包在运行时仍需要一个**模拟根目录**。在此下载现成的 root：

- https://sogen.dev/root.zip

把它解压到方便的地方，例如：

```text
./root
```

本文档大多数示例使用：

```python
emulation_root="./root"
```

## 快速开始

```python
import sogen

app = sogen.windows.create_application(
    "c:/test-sample.exe",
    emulation_root="./root",
)

app.callbacks.on_stdout = lambda text: print(text, end="")
app.start()
print("exit status:", app.process.exit_status)
```

Linux 快速开始：

```python
import sogen

syscalls = []


def on_syscall(syscall_id, syscall_name):
    syscalls.append((syscall_id, syscall_name))
    return sogen.HookContinuation.run


app = sogen.linux.create_application(
    "/bin/true",
    emulation_root="/",
    backend=sogen.Backend.unicorn,
)

app.callbacks.on_syscall = on_syscall
app.start()
print("exit status:", app.process.exit_status)
print("syscalls observed:", len(syscalls))
```

`sogen.linux` 暴露它自己的 Linux 作用域工厂函数，包括 `sogen.linux.create_application(...)` 和 `sogen.linux.create_empty(...)`；它不使用根级别的兼容别名。

## 带文件 + 端口映射的最小示例

Windows 示例：

```python
from pathlib import Path
import sogen

app = sogen.windows.create_application(
    "c:/test-sample.exe",
    emulation_root="./root",
    root_mode="ro",
    mounts=[(Path("./a.txt"), "c:/a.txt", "rw")],
    network="live",
    port_mappings={28970: 28980},
)

app.callbacks.on_stdout = lambda text: print(text, end="")
app.start()
print("exit status:", app.process.exit_status)
```

`root_mode` 设置模拟根目录中驱动器的访问模式。`mounts` 中每项是 `(host_path, guest_path)` 或 `(host_path, guest_path, mode)`。宿主机路径可以是文件或目录。模式是 `"ro"` 或 `"rw"`，两种设置默认都为 `"ro"`。客体对 `"ro"` 路径的写入失败并返回 `STATUS_ACCESS_DENIED`，宿主机文件不变。`path_mappings={guest_path: host_path}` 等同于默认模式的 `mounts`。

应用之后的位置参数被拼进进程命令行。为空或包含空格、制表符或引号的参数会被加引号，且它自身的引号被转义。`command_line="..."` 改为逐字设置整条命令行，包含程序名，正如 `CreateProcess` 会收到的那样。给出 `command_line` 时，位置参数被忽略。

Linux 应用还接受客体 cwd、文件系统映射和 TCP 端口映射 kwargs：

```python
from pathlib import Path
import sogen

app = sogen.linux.create_application(
    "/bin/my-tool",
    emulation_root="./linux-root",
    working_directory="/work",
    path_mappings={"/work/input": Path("./fixtures/input")},
    read_only_path_mappings=[("/work/readonly", Path("./fixtures/readonly"))],
    port_mappings={8080: 18080},
)

assert app.get_host_port(8080) == 18080
assert app.get_emulator_port(18080) == 8080
```

Linux `working_directory` 被规范化为绝对客体路径，并在应用加载/启动前生效，因此它同时适用于解释器/加载器工作以及 `getcwd()` 和 cwd 相对系统调用如 `open()` 和 `openat(AT_FDCWD, ...)`。`path_mappings` 和 `read_only_path_mappings` 也在加载/启动前安装，使映射的客体路径对解释器启动和后续文件系统操作可见。它们接受 `{guest_path: host_path}` 字典或 `[(guest_path, host_path), ...]` 序列，最长匹配的客体前缀胜出。只读映射拒绝变更性文件操作，包括写打开、写入、截断、chmod、unlink 和 rename。`port_mappings` 接受 `{emulator_port: host_port}` 或 `[(emulator_port, host_port), ...]`；被映射的 IPv4 回环 TCP `connect()` 调用被代理到配置的宿主机回环端口，而未映射端口保持模拟的 Linux 套接字行为。

路径和端口映射显式选择加入宿主机文件系统和网络访问。它们是测试和受控分析工作流的便利桥梁，而不是恶意软件沙箱边界。

Windows 模拟器接受 `network="offline"`（默认）或 `network="live"`。其他字符串抛 `ValueError`，非字符串值抛 `TypeError`。离线模式下，客体套接字使用无法到达宿主机网络的进程内回环，DNS 查询解析为空地址列表，且 Steam 桥被禁用。`port_mappings` 只在 live 模式生效。返回 `None` 的 `dns_resolver` 回调会回退到所选模式的解析器，在离线模式下解析为空地址。

`app.callbacks.on_socket_activity` 在客体调用 `connect`、`bind` 或 `sendto` 时接收 `(action, address)`。action 是 `"connect"`、`"bind"` 或 `"sendto"` 之一，address 是像 `"127.0.0.1:28970"` 这样的字符串，端口为客体所见的端口。

## 选择后端

`sogen.windows.create_empty()`、`sogen.windows.create_application()`、`sogen.linux.create_empty()` 和 `sogen.linux.create_application()` 接受 `backend=sogen.Backend.unicorn`。

```python
import sogen

emu = sogen.windows.create_empty(
    emulation_root="./root",
    backend=sogen.Backend.unicorn,
)
```

可用值：

- `sogen.Backend.unicorn`
- `sogen.Backend.icicle`
- `sogen.Backend.whp` (Windows)
- `sogen.Backend.kvm` (Linux x86_64)

默认是 `sogen.Backend.unicorn`。

## 高层结构

Windows 入口点：

- `sogen.windows.create_empty(...)`
- `sogen.windows.create_application(...)`

Linux 入口点：

- `sogen.linux.create_empty(...)`
- `sogen.linux.create_application(...)`

Windows 兼容别名当前保留在顶层，且是 Windows 特有的：

- `sogen.create_empty(...)` -> `sogen.windows.create_empty(...)`
- `sogen.create_application(...)` -> `sogen.windows.create_application(...)`

Windows 绑定暴露的对象：

- `sogen.windows.Emulator` / `sogen.windows.WindowsEmulator`
- `ProcessContext`
- `Thread`
- `MemoryManager`
- `Hooks`
- `Callbacks`

绑定暴露的 Linux 特有对象：

- `sogen.linux.Emulator` / `sogen.linux.LinuxEmulator`
- `sogen.linux.ProcessContext`
- `sogen.linux.MemoryManager`
- `sogen.linux.Thread`
- `sogen.linux.MemoryRegionInfo`
- `sogen.linux.Hook`
- `sogen.linux.Hooks`
- `sogen.linux.Callbacks`
- `sogen.linux.ExportedSymbol`
- `sogen.linux.MappedSection`
- `sogen.linux.LinuxMappedModule`
- `sogen.linux.SymbolHooks`
- `sogen.linux.LinuxSymbolCall`
- `sogen.linux.ThreadWaitState`

Linux 回调槽：

- `app.callbacks.on_stdout`
- `app.callbacks.on_stderr`
- `app.callbacks.on_syscall`
- `app.callbacks.on_memory_violate`
- `app.callbacks.on_memory_allocate`
- `app.callbacks.on_memory_protect`
- `app.callbacks.on_memory_release`
- `app.callbacks.on_module_load`
- `app.callbacks.on_thread_create`
- `app.callbacks.on_thread_terminated`
- `app.callbacks.on_thread_switch`

常见 Windows 工作流：

- 用 `app.start()` 运行应用
- 用 `app.callbacks.on_stdout` 观察输出
- 用 `app.callbacks.on_module_load` 对模块加载做出反应
- 用 `app.hooks.apis[...]` 拦截 WinAPI 调用
- 用 `read_memory()` / `write_memory()` 读/写模拟器内存
- 用 `save_snapshot()` / `restore_snapshot()` 保存和恢复状态

## 回调

示例：打印已加载模块。

```python
import sogen

app = sogen.windows.create_application(
    "c:/test-sample.exe",
    emulation_root="./root",
)


def on_module_load(module):
    print(f"loaded {module.name} @ 0x{module.entry_point:x}")


app.callbacks.on_module_load = on_module_load
app.start()
```

有用的 Linux 回调槽：

- `app.callbacks.on_stdout`
- `app.callbacks.on_stderr`
- `app.callbacks.on_syscall`
- `app.callbacks.on_memory_violate`
- `app.callbacks.on_memory_allocate`
- `app.callbacks.on_memory_protect`
- `app.callbacks.on_memory_release`
- `app.callbacks.on_module_load`
- `app.callbacks.on_thread_create`
- `app.callbacks.on_thread_terminated`
- `app.callbacks.on_thread_switch`

有用的 Windows 回调槽：

- `app.callbacks.on_stdout`
- `app.callbacks.on_syscall`
- `app.callbacks.on_memory_violate`
- `app.callbacks.on_module_load`
- `app.callbacks.on_module_unload`

Linux `on_syscall` 回调使用 `callback(syscall_id: int, syscall_name: str) -> sogen.HookContinuation | bool | None`。
`None`、`False` 和 `sogen.HookContinuation.run` 继续内建系统调用处理程序。
`True` 和 `sogen.HookContinuation.skip` 抑制内建处理程序，因此拦截回调必须自己更新客体寄存器和内存。

Linux `on_module_load` 回调使用 `callback(module: sogen.linux.LinuxMappedModule) -> None`。
赋值 `app.callbacks.on_module_load` 会按映像基址升序重放应用构造期间已映射的模块，然后观察未来从运行时文件支撑 `mmap()` 活动（例如由 Linux 动态加载器映射的 libc）中发现的 ELF 模块。Linux 当前不暴露模块卸载回调，因为模拟器中没有原生 Linux 卸载事件源。

Linux 线程控制暴露活跃的模拟线程和协作式原生调度器钩子：

- `app.current_thread` 返回活跃的 `sogen.linux.Thread` 或 `None`。
- `app.current_thread_id` 返回活跃 TID 或 `None`。
- `app.activate_thread(tid)` 切换到一个存活线程，保存/恢复 CPU 寄存器，并对缺失或已终止的线程返回 `False`。
- `app.perform_thread_switch()` 按 TID 顺序选择下一个可运行的未终止线程，并返回是否发生了切换。
- `app.yield_thread()` 执行该切换，并在发生切换时停止后端。
- `app.process.threads` 列出未终止的 Linux 线程；每个线程暴露 `tid`、`current_ip`、`start_address`、`wait_state`、`setup_done`、栈/TLS 字段、终止状态、退出码和已执行指令数。`previous_ip` 抛 `NotImplementedError`，直到 Linux 追踪上一条指令指针而不是静默返回占位值。

Linux 线程回调使用原生生命周期/切换路径：

- `app.callbacks.on_thread_create(thread)` 在 `clone(CLONE_THREAD)` 创建线程后触发。
- `app.callbacks.on_thread_terminated(thread)` 在 Linux 线程退出时触发。
- `app.callbacks.on_thread_switch(old_tid, new_tid)` 在活跃 TID 通过 `activate_thread()` 或调度器改变时触发。



Linux 模拟器在每次 `start()` 调用后暴露停止诊断：

- `app.last_stop_reason` 是字符串：`"none"`、`"unknown_syscall"`、
  `"unimplemented_syscall"`、`"syscall_exception"`、`"instruction_limit"`、
  `"normal_exit"`、`"signal_termination"`、`"unhandled_memory_violation"`、
  `"explicit_stop"`、`"backend_error"`、`"breakpoint"` 或 `"watchpoint"`。
- `app.last_stop_reason_code` 是整数枚举值，供偏好稳定数值存储的集成使用。
- `app.last_stop_detail` 包含特定原因细节，例如未处理内存违例的 `address=0x... size=...` 或指令限制的 `count=...`。

Linux 信号回调使用 `app.callbacks.on_signal(signum, fault_addr, si_code)`。
回调在原生信号传递开始时触发，在被模拟进程的处理程序或 Linux 默认终止路径运行之前。它是观察性的：返回值被忽略，无法抑制客体传递。`on_exception` 是同一回调槽的 Linux 别名，接收相同的 `(signum, fault_addr, si_code)` 载荷，因为 Linux 异常以信号形式出现。该别名也由 `repr(app.callbacks)` 显示，便于运行时发现。

Linux 内存违例回调使用
`callback(address: int, size: int, operation: sogen.MemoryOperation, type: sogen.MemoryViolationType) -> sogen.MemoryViolationContinuation | bool | None`。
`sogen.MemoryOperation` 是 `sogen.MemoryPermission` 的别名，与后端操作值匹配。`None`、`True` 和 `sogen.MemoryViolationContinuation.resume` 恢复执行；`False` 和 `sogen.MemoryViolationContinuation.stop` 以 `last_stop_reason == "unhandled_memory_violation"` 停止；`sogen.MemoryViolationContinuation.restart` 要求后端重试出错指令。如果没有安装 Linux 内存违例回调或钩子，模拟器保持其原生 SIGSEGV/默认停止路径。

## Linux 底层钩子

Linux 模拟器通过 `app.hooks` 暴露底层钩子。大多数方法是后端钩子；`memory_violation(callback)` 是与模拟器内部 SIGSEGV 传递路径共存的 Linux 托管观察者。钩子方法返回带只读 `active` 属性和幂等 `remove()` 方法的 `sogen.linux.Hook` 句柄。即使你不去存储返回的句柄，它们也保持活跃：`app.hooks` 持有钩子直到它被移除或模拟器被销毁。

```python
import sogen

emu = sogen.linux.create_empty(emulation_root="/")
emu.memory.allocate_memory_at(0x100000, 0x1000, sogen.MemoryPermission.exec)
emu.write_memory(0x100000, b"\x0f\xa2")  # cpuid
emu.write_register(sogen.Register.rip, 0x100000)


def on_cpuid(data):
    emu.stop()
    return sogen.HookContinuation.skip


hook = emu.hooks.instruction(sogen.Instruction.cpuid, on_cpuid)
emu.start(10)
hook.remove()
```

可用的 Linux 钩子方法：

- `app.hooks.memory_execution(callback)`
- `app.hooks.memory_execution_at(address, callback)`
- `app.hooks.memory_read(address, size, callback)`
- `app.hooks.memory_write(address, size, callback)`
- `app.hooks.instruction(sogen.Instruction.<name>, callback)`
- `app.hooks.interrupt(callback)`
- `app.hooks.memory_violation(callback)`
- `app.hooks.basic_block(callback)`

Linux `Hooks` 把 Linux 符号钩子暴露为 `app.hooks.symbols`。仅限 Windows 的 `hooks.apis` 在 `sogen.linux` 下仍缺席。

## Linux 模块与符号钩子

Linux 应用在 `app.start()` 之前暴露初始 ELF 模块，并随动态加载器映射而追加运行时加载的 ELF 模块：

```python
import ctypes
import sogen

app = sogen.linux.create_application("/bin/true", emulation_root="/")
for module in app.modules:
    print(module.name, hex(module.image_base), module.path)
```

每个 `sogen.linux.LinuxMappedModule` 有 `name`、`path`、`image_base`、
`size_of_image`、`entry_point`、`exports`、`needed_libraries`、`sections`、
`rpath` 和 `runpath`。Exports 是带 `name`、`rva` 和 `address` 的 `sogen.linux.ExportedSymbol` 对象；sections 是带 `name`、`start`、`length` 和 `permissions` 的 `sogen.linux.MappedSection` 对象。可执行 ELF 节区和区域暴露组合权限如 `sogen.MemoryPermission.read_exec`；写-执行映射使用 `sogen.MemoryPermission.write_exec`。使用 `app.find_module_by_address(address)` 或 `app.find_module_by_name(name)` 查找模块。

符号钩子用 `app.hooks.symbols["symbol"] = callback` 或 `app.hooks.symbols["module!symbol"] = callback` 注册。限定形式区分大小写地匹配 Linux 模块 `name` 或文件名字干。给键赋 `None` 删除该钩子，或调用 `app.hooks.symbols.clear()` 移除所有符号钩子。

用 `@sogen.linux.symbol_call(params=[...], restype=...)` 描述 x86_64 System V 整数/指针签名。支持 ctypes 为不超过 8 字节的整数和指针大小标量/指针 ctypes；不支持的 ctypes 在钩子注册时失败。参数从 `rdi`、`rsi`、`rdx`、`rcx`、`r8`、`r9` 解码，然后是返回地址之后的栈槽。

```python
@sogen.linux.symbol_call(params=[ctypes.c_int], restype=ctypes.c_int)
def on_target(call, params):
    print(call.module.name, call.name, params[0])
    return sogen.ApiContinuation.run_original


app.hooks.symbols["target_function"] = on_target
```

回调接收带只读 `module`、`name`、`address` 和 `return_address` 的 `sogen.linux.LinuxSymbolCall`，外加可读写 `return_value`。返回 `None`、`False` 或 `sogen.ApiContinuation.run_original` 继续原符号。返回 `True` 或 `sogen.ApiContinuation.intercept` 通过把 `return_value` 写入 `rax`、把 `rip` 设为保存的返回地址、并把 `rsp` 前进 8 来跳过原符号。

后端支持因引擎而异。Unicorn 和 Icicle 支持细粒度执行、读/写、指令、中断、基本块和符号钩子。KVM 也为 API 兼容注册这些钩子，但读/写、执行、基本块和符号回调可能不在该后端触发。

## Linux 调试器门面

Linux 模拟器暴露 `app.debug`，一个构建在用于底层 Linux 钩子的相同后端钩子和 CPU 状态之上的 Python 原生调试器门面。断点是执行钩子：`app.debug.set_breakpoint(address)` 布防一个，`clear_breakpoint(address)` 移除它，`list_breakpoints()` 返回当前已布防地址。执行到达断点时，`app.last_stop_reason == "breakpoint"`，`app.last_stop_detail` 包含地址，且 `app.debug.registers()["rip"]` 是断点地址。

```python
emu = sogen.linux.create_empty(emulation_root="/")
emu.memory.allocate_memory_at(0x100000, 0x1000, sogen.MemoryPermission.exec)
emu.write_memory(0x100000, b"\x0f\xa2\x0f\xa2")  # cpuid; cpuid
emu.write_register(sogen.Register.rip, 0x100000)
emu.debug.set_breakpoint(0x100002)
emu.debug.continue_execution()
assert emu.last_stop_reason == "breakpoint"
```

步进和运行控制方法是 `step_into()`、`step_over()`（当前是相同的单指令原语）、`run_to(address)`、`continue_execution()` 和 `pause()`。`step_out()` 使用帧指针遍历，并抛出区分零 RBP、不可读的保存返回地址槽和零保存返回地址的诊断性 `RuntimeError`。消息建议在显式目的地更可靠时使用 `run_to(address)`。自省方法返回 Python 原生数据：`registers()` 返回寄存器字典；`modules()` 和 `threads()` 返回字典列表；`disassemble(address, count_or_size)` 返回带地址、字节、助记符、操作数和大小的指令字典；`call_stack()` 返回有界的尽力而为帧指针遍历。

## API 钩子

API 钩子通过 `app.hooks.apis` 注册。

用 `@sogen.windows.api_call(...)` 描述调用约定和参数。顶层 `sogen.api_call(...)` 保留为兼容别名。

### 观察 API 调用，然后运行原函数

```python
import ctypes
import sogen

app = sogen.windows.create_application(
    "c:/test-sample.exe",
    emulation_root="./root",
)


@sogen.windows.api_call(cc=sogen.CallingConvention.stdcall, params=[ctypes.c_uint32])
def on_sleep(call, params):
    print(f"Sleep({params[0]})")


app.hooks.apis["Sleep"] = on_sleep
app.start()
```

### 拦截 API 调用并返回自定义值

```python
import sogen

app = sogen.windows.create_application(
    "c:/hook-sample.exe",
    emulation_root="./root",
)


@sogen.windows.api_call(cc=sogen.CallingConvention.stdcall, params=[])
def on_get_current_process_id(call, params):
    call.return_value = 0xC0FFEE01
    return sogen.ApiContinuation.intercept


app.hooks.apis["GetCurrentProcessId"] = on_get_current_process_id
app.start()
print(app.process.exit_status)
```

钩子键可以是：

- 裸 API 名，例如 `"Sleep"`
- 限定模块形式，例如 `"kernel32!Sleep"`

## 内存与状态

模拟器暴露直接状态访问。Windows 和 Linux 模拟器都提供 `serialize_state()` / `deserialize_state(bytes)` 做显式字节检查点，以及 `save_snapshot()` / `restore_snapshot()` 做对象内快照槽。

```python
import sogen

emu = sogen.linux.create_empty(emulation_root="/")
base = emu.memory.allocate_memory(0x1000, sogen.MemoryPermission.read_write)
emu.write_memory(base, b"ABCD")
emu.write_register(sogen.Register.rax, 0x1234)

state = emu.serialize_state()
emu.write_memory(base, b"WXYZ")
emu.write_register(sogen.Register.rax, 0)
emu.deserialize_state(state)
assert emu.read_memory(base, 4) == b"ABCD"
assert emu.read_register(sogen.Register.rax) == 0x1234
```

`read_register` 和 `write_register` 移动 64 位整数。Windows 模拟器还提供 `read_register_bytes(register, size)` 和 `write_register_bytes(register, data)`，它们移动原始小端字节，因此能到达 x86-64 上的 128 位 XMM 寄存器和 ARM64 上的 `sogen.arm64.Register.v0` 到 `v31`。短于寄存器的值零扩展，超出寄存器宽度的字节读为零，且两者最多接受 64 字节。`sogen.Register.fs_base` 和 `gs_base` 命名段基址；64 位 Windows 上 `gs_base` 是当前线程的 TEB。

Linux 序列化状态使用版本标签 `linux-emulator-state-v1`，包含后端 CPU 状态、已执行指令数、停止诊断、映射内存字节和权限、进程 id、brk、argv/envp/auxv、线程、目录和 epoll 缓存、ELF 模块元数据、信号动作和可变 vDSO 元数据。打开的 Linux 文件描述符仅为 stdio、procfs/内存文件、eventfd、目录描述符、epoll 描述符和可按路径重新打开并恢复到其文件偏移的宿主机文件序列化。活动管道、套接字、未命名宿主机描述符和不可重开的宿主机描述符在序列化或恢复时抛出，而不是被静默丢弃。

Linux 内存管理器通过 `app.memory.mapped_regions`、`app.memory.get_mapped_regions()`、`app.memory.get_region_info(address)` 和 `app.memory.compute_memory_stats()` 暴露映射区域自省。`mapped_regions` 返回带 `start`、`length`、`permissions`、`allocation_base`、`allocation_length`、`is_reserved`、`is_committed`、`initial_permissions` 和 `kind` 的 `sogen.linux.MemoryRegionInfo` 对象。组合区域权限包括 `sogen.MemoryPermission.read_exec` 和 `sogen.MemoryPermission.write_exec`。Linux 没有 Windows 的 reserve/commit 拆分。每个映射区域报告：

- `allocation_base == start`
- `allocation_length == length`
- `is_reserved is False`
- `is_committed is True`
- `initial_permissions == permissions`
- `kind == sogen.MemoryRegionKind.private_allocation`

`compute_memory_stats()` 返回包含以下项的字典：

- `region_count`
- `mapped_bytes`
- `executable_bytes`

Linux 内存生命周期回调观察成功的原生内存操作：

- `app.callbacks.on_memory_allocate(address, length, permissions, committed)` 在 `mmap`/分配成功后触发。Linux 总是传 `committed=True`，因为没有仅保留状态。
- `app.callbacks.on_memory_protect(address, length, permissions)` 在保护变更成功后触发。
- `app.callbacks.on_memory_release(address, length)` 在 unmap/release 成功后触发。

对于检查点式工作流，使用快照：

```python
emu.save_snapshot()
# ... mutate state ...
emu.restore_snapshot()
```

## 示例

可运行的小示例：

- `examples/python/basic_usage.py` 用于 Windows
- `examples/python/linux_true.py` 用于 Linux

示例设置说明：

- `examples/python/README.md`

## 当前限制 / 预期

- Windows 绑定需要模拟根目录
- 本仓库中的 Windows 示例假定 Windows 风格客体路径如 `c:/...`
- Windows 工作流最易对着仓库样例二进制如 `test-sample.exe` 和 `hook-sample.exe` 验证
- 后端可用性取决于平台和 Sogen 的构建方式
