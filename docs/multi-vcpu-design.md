# 多 vCPU 支持 — 设计

状态：在 `multi-vcpu` 分支上进行中

- [x] 阶段 0a — 能力查询、`vcpu_count` 设置、`--vcpus`、硬错误校验（4aa27ba1）
- [x] 阶段 0b — `x86_64_cpu` 抽取（5611f1d4）；钩子回调携带触发 CPU（a68338eb）；`vcpu_context` 取代 `process_context::active_thread`，`CURRENT_THREAD` 解析显式接收调用线程，快照格式不变（9d6be10b）
- [x] 阶段 1 — WHP 驱动 N 个 VP：每 VP 一个 `whp_vcpu` 对象、参数化的 VP 索引、`partition_mutex_`（shared_mutex，copy-then-invoke 回调纪律）、`ProcessorCount = N`、工厂函数接收 `vcpu_count`（eef2e558）。Clang-tidy 干净。
- [~] 阶段 2 — BEL + vcpu_worker 调度器。已完成：`kernel_lock`（owner 追踪、不可递归、基于 assert 的内部检查）在每个入口点获取（钩子回调、UI 事件投递、调度器循环 —— 在客体执行和空闲睡眠间释放）；logger 打印互斥；启动冲突校验（指令精度 / 相对时间 / 指令预算 / N>1 时的 gdb）；每 vCPU 一个宿主机 worker 线程（`vcpu_worker`），带每 vCPU 线程所有权（加载在某 vCPU 上的客体线程会被其他 vCPU 跳过；vCPU 空闲时分离）；定时器线程踢醒所有运行中的 vCPU；`stop()` 取消所有 vCPU；WHP 每 vCPU 延迟 TLB 刷新（标志 + 取消，在运行循环顶部应用），取代跨 vCPU CR3 重写。N=1 保持其久经验证的内联循环（worker 只在 N>1 时生成）；N=1 + 所有测试不受影响。
- [x] 阶段 3（核心）— **并行执行已稳定**。Threads 测试的竞争是虚假的访问违例：在多 vCPU 下，一个以允许该访问的权限支撑的页面仍会触发故障，因为这个 vCPU 持有*过期转换*（一个对等 vCPU 刚映射/提交/重保护了它 —— 例如 ntdll 的 `.mrdata`/`.00cfg`/`.rsrc` 在加载器初始化期间翻转 RO↔RW）。升级的故障投递了一个假的 0xC0000005。修复：`try_repair_spurious_fault` 在所有三个 WHP 故障路径中，只要页面已支撑且其预期权限允许该操作，就重新建立映射 + 刷新这个 vCPU 的 TLB + 重试（守卫/不可访问页以权限 `none` 映射，所以真正的违例仍会投递）；每 vCPU (page,rip) 重试计数器度过并发再清除；上限只适用于访问类型靠猜的不可恢复退出路径，而带真实读/写的良类型路径（#PF、内存访问）无上限重试，因为那里已支撑+允许的故障永远是瞬时的。结果：`--vcpus 1` 100%（不变），`--vcpus 2`/`4` 100%，`--vcpus 8` 约 99%（重度超额订阅下仍有罕见失败）；N=1 和 23 个单元测试不受影响。客体 CPU 身份完成：客体现在看到 `max(fake_env.number_of_processors, vcpu_count)` 个处理器（KUSD ActiveProcessorCount、PEB、SystemBasicInformation 全部一致），且 NtGetCurrentProcessorNumber[Ex] 返回正在作用的 vCPU 索引。阶段 3 剩余（非阻塞，延迟/功能）：针对*运行中*线程的 alert/APC/suspend/terminate 跨 vCPU 踢（当前在下一次约 20ms 定时器抢占时生效 —— 功能正确）；运行中线程的 NtGet/SetContextThread 强制保存；stop-the-world 快照静默（快照仍为 N=1）；以及罕见的 `--vcpus 8+` 失败。更早的寄存器/分发修复细节见下文。
- [~] 阶段 3（更早的修复）— 修复了使非零 vCPU 上的每个系统调用都碰到 vCPU 0 寄存器的寄存器路由 bug：`syscall_context::emu`（以及参数/寄存器辅助、`callback_frame`、异常分发、rdtsc/rdtscp/interrupt/violation 钩子）现在使用正在作用的 vCPU 的 `x86_64_cpu` 而不是 `windows_emulator::emu()`（vCPU-0 门面）。客体内存是共享的，所以 `x86_64_cpu` 获得了到共享 `memory_interface&` 的隐式转换。遗留的 `current_thread()` 观察者 API 通过 `scoped_dispatch` 变得正确：`windows_emulator::dispatch_vcpu_` 成员在 BEL 下的每个处理程序入口设置（无竞争，因为 BEL 串行化处理程序 —— 不是全局或线程局部）。还修复了一个 WHP 提前终止 bug：`flush_virtual_address_mappings` 取消其他运行中的 VP（`WHvCancelRunVirtualProcessor`）但不设置 `stop_requested_`，且 `run()` 的 Canceled 处理程序在 rip 前进时返回（而不是重启）—— 所以一个为对等 vCPU 页表变更而被取消的 vCPU 返回到 worker，worker 看到 `!switch_thread` 并结束了整个运行（"Emulation terminated without status"）。现在不带 `stop_requested_` 的取消永远是 TLB 刷新重入，绝不停止。有了这些修复，N>1 可以正确跑完整个冒烟套件约 15 个测试，verbose 运行到达干净的 `NtTerminateProcess`（状态 0）。**Threads** 测试在 `-s`（静默、最快/最并行）下仍失败 —— 一个真实的时序竞争（Heisenbug：宿主机日志串行化到足以隐藏它）。一个客体线程在竞争下抛出未处理异常（联系 WER）。主要嫌疑：WHP 节区首次执行钩子中分区全局 EPT 单步舞步（两个 vCPU 在同一共享页上翻转/恢复 exec —— §4 中接受的那个竞争；在真实竞争下可能并非良性）—— 但 `--whp-exec-hook int3` 失败得一模一样，所以 exec-hook 机制被**排除**。此后修复了一个真实的锁外竞争：KUSD MMIO 读取（`kusd_mmio::read`/`update`）在客体执行期间在 worker 线程上运行（内核锁已释放）且没有防护，所以两个读时间页的 vCPU 撕裂了 `kusd_`；现在由内部互斥锁防护。这个修复加上 cancel-return 修复，把 `--vcpus 2` 冒烟套件通过率从几乎总是失败提到约 6/8（完整套件完成，干净的 `NtTerminateProcess`）。加入了事后异常追踪器（环境变量 `SOGEN_TRACE_EXCEPTIONS=1`，运行后转储以免扰动竞争）：它把崩溃精确定位为对内存管理器报告为**已提交**页面的虚假访问违例（0xC0000005）—— 出错 vCPU 上的客体页表项缺失。确定的故障地址（ntdll+0x1ea598，一个映像 `.data`/COW 页）但时好时坏。加入了过期 TLB 重试：一个 not-present 的 #PF 若其客体 PTE 实际存在，则刷新该 vCPU 的 TLB 并继续，而不是投递假的 AV。这只在 PTE 存在时有效；剩余情况是**已提交页面其急切的 WHP 映射在出错 vCPU 上缺失** —— 很可能是映像/COW 映射路径（而不是急切的 `map_memory`）未跨 vCPU 建立/刷新。下一步：给 WHP 违例 fall-through 打日志，记录 GvaValid + mapped_pages_ 状态 + is_virtual_mapping_present。根因定位：客体页故障升级为 WHP 的*不可恢复异常*退出（客体 IDT 无法分发 #PF），该路径读取 CR2 并投递内存违例而**没有**任何虚假故障检查 —— 所以对一个实际已支撑页面的故障（对等 vCPU 的 map/commit，或另一个 vCPU 上清除这个 vCPU PTE 的瞬时保护/COW 重映射）变成了假的 AV。修复：`try_repair_spurious_fault`（重新建立映射 + 刷新这个 vCPU 的 TLB + 重试，带每 vCPU (page,rip) 防护，使真正的违例 —— 空指针解引用、错误权限页 —— 在一个周期后仍被投递）现在在全部三个故障路径中运行（`handle_exception` #PF、`handle_memory_access` GpaUnmapped、`handle_unrecoverable_exception`）。这把 `--vcpus 2` 冒烟通过率提到约 16/24（此前约 1/2）。尝试了 `apply_memory_protection` 中的推测性刷新并回退（无明显帮助；Hyper-V 处理 EPT 级一致性；增加了扰动）。
  **残留**：约 1/3 的静默 `--vcpus 2` 运行仍在 **Threads** 测试失败 —— 修复触发但页面有时被并发再清除（守卫触发 → 投递 AV），指向共享页上更紧的 map/unmap/protect 竞争，或客体级加载器锁 / .mrdata 竞争。仍是 Heisenbug（热路径日志下消失）；环境门控的事后追踪器（`SOGEN_TRACE_EXCEPTIONS=1`）是工具。阶段 3 剩余：那个竞争；跨 vCPU 踢；快照 stop-the-world；客体 CPU 数 = vcpu_count。阶段 2 打磨剩余：ready_cv 取代 sleep-poll；设备泵移到 tick；N=1 启动基准；clang-cl -Wthread-safety 注解。
- [~] 阶段 3 — 跨 vCPU 正确性（WHP：64 位 + WoW64 32 位在 N=1..8 运行；真实游戏 open-iw5/MW3 在 N=2 运行到自行退出。已落地修复：每 vCPU GDT、SDL UI 封送到泵线程、syscalls/rdtsc/rdtscp/cpuid/read-hook 的作用 vCPU 归属。已收敛，未关闭 —— 更多工作负载会暴露更多竞争。）
- [x] 阶段 4 — 压力测试 + 竞争分析（结果见 §9 阶段 4）
- [ ] 阶段 5 — KVM 对齐、linux-emulator

范围：WHP 后端优先，KVM 其次；windows-emulator 优先，linux-emulator 其次。目标：让客体线程真正并行运行 —— 一个宿主机线程驱动一个 vCPU —— 而不是把所有客体线程协作式复用到单个 vCPU 上。

---

## 1. 现状

整个栈围绕**一个 vCPU、一个宿主机执行线程**构建：

- 每个后端实例*就是*一个 vCPU。WHP 把 `constexpr UINT32 vp_index = 0`（`whp_x86_64_emulator.cpp:29`）硬编码进每个 `WHvCreateVirtualProcessor` / `WHvRunVirtualProcessor` / `WHvGet/SetVirtualProcessorRegisters` / `WHvCancelRunVirtualProcessor` 调用，并设置分区 `ProcessorCount = 1`（`:1866`）。KVM 镜像了这个做法（`kvm_x86_64_emulator.cpp:62`，一个 `vcpu_fd_`）。
- 客体线程被协作式复用：一次上下文切换通过 `save_registers()`/`restore_registers()` 把整个寄存器文件序列化进 `emulator_thread::last_registers`（`emulator_thread.hpp:335`），并把另一个线程的数据块换进单个 vCPU（`switch_to_thread`、`windows_emulator.cpp:385`）。
- 调度位于 `windows_emulator::start()`（`windows_emulator.cpp:1023`）：轮询（`switch_to_next_thread`、`:442`），按每条指令 / 每个基本块（unicorn/icicle）抢占，或由一个每 20 ms 调用 `emu().stop()` → `WHvCancelRunVirtualProcessor` 的中断线程抢占（WHP/KVM，`:1052-1068`）。
- **几乎没有同步**，因为它从不需要。今天整个并发面就是：`switch_thread_` / `should_stop` 原子量、`memory_manager::layout_version_` 计数器、中断线程和一个 web-UI 互斥锁。

第二个并发执行的 vCPU 会竞争的状态，按爆炸半径排序：

| # | 共享状态 | 位置 |
|---|--------------|----------|
| 1 | 单个 vCPU 寄存器文件（save/restore 数据块交换） | `emulator_thread.hpp:335,379-388` |
| 2 | 调度器状态：`threads`、`active_thread`、`thread_handles_by_id` | `process_context.hpp:468,483-484` |
| 3 | 所有 `handle_store` 对象表（事件、互斥体、文件、节区、窗口、…）+ 非原子引用计数 | `handles.hpp:184,147`、`process_context.hpp:449-467` |
| 4 | `memory_manager::reserved_regions_`（普通 `std::map`，被 NtAllocate/Protect/FreeVirtualMemory 修改） | `memory_manager.hpp:155` |
| 5 | WHP 分区表：`mapped_pages_`、`guest_pages_by_gpa_`、`free_guest_gpa_indices_`、`page_table_views_`、`mmio_regions_`、所有钩子映射 | `whp_x86_64_emulator.cpp:1809-1839` |
| 6 | Wait/APC 簿记跨线程写入（NtQueueApcThread、NtAlertThreadByThreadId 目标是*其他*线程） | `syscalls/thread.cpp:555,974` |
| 7 | KUSD（`kusd_mmio::update()` 每次读取时重算） | `kusd_mmio.cpp:157-179` |
| 8 | 设备（`io_device::work()` 从 CPU 线程轮询、`block_mutation` 重入 hack） | `windows_emulator.cpp:301-310`、`io_device.hpp:190` |
| 9 | 每个后端的钩子容器（未加锁的 map/vector，钩子从回调内部删除） | 如 `whp_x86_64_emulator.cpp:1826-1833`、`windows_emulator.cpp:748-752` |

两个事实使这个项目可行：

1. **客体线程状态已经与 vCPU 解耦。** 寄存器数据块模型（`last_registers`）正是 N vCPU 调度器所需要的 —— 一个线程的上下文可以恢复到*任何* vCPU 上。
2. **WHP/KVM 已经把昂贵的状态保持在分区作用域。** EPT/GPA 映射、页表、MMIO 区域和钩子是每分区的，这正是多个 vCPU 必须共享的东西。每 vCPU 部分（寄存器、运行/取消、单步状态）很小且可以干净地分离。

---

## 2. 两个核心设计决策

### 决策 1 — N 个 vCPU + 一个真正的调度器（不是一个客体线程一个宿主机线程）

两种可能的执行模型：

- **(a) 1 客体线程 = 1 vCPU = 1 宿主机线程。** 完全没有调度器；阻塞的客体线程停在宿主机条件变量上。被拒绝：客体进程创建几十到几百个线程（工作工厂、线程池）；WHP `ProcessorCount` 在分区设置时固定且 vCPU 重量级；客体在表观 CPU 数量巨大时表现也不好。
- **(b) N 个 vCPU（N = 配置的处理器数，如 4），M 个客体线程由调度器复用到其上 —— 像真正的内核。** 每个 vCPU 由一个专用宿主机 worker 线程驱动：挑一个就绪客体线程，把它的寄存器数据块恢复到*这个* vCPU 上，运行到让出/抢占/等待，把数据块存回，重复。

**(b) 是设计。** 它保留了现有、久经验证的 `save/restore` 上下文切换机制，保持客体可见的 CPU 数与我们已伪造的一致（`fake_environment_config::number_of_processors = 4`、`windows_emulator.hpp:85`），并在 N=1 时优雅降级到今天的行文。

### 决策 2 — 给"内核"加大模拟器锁（BEL），客体用户态并行

windows-emulator 层实际上是一个内核：`process_context`、句柄存储、内存管理器、设备以及数百个系统调用处理程序，全都假定串行执行。选项：

- **每结构细粒度锁。** 被拒绝作为起点：系统调用处理程序假定*跨*结构的原子性（例如 `NtWaitForMultipleObjects` 同时读取多个对象表和线程等待状态；`is_thread_ready` / `consume_object_signal` 依赖"我做决定时没有其他东西在运行"）。追补每结构锁意味着审计每个处理程序的跨结构不变量并招致死锁。这是一个长达数月的审计，带着长长的 bug 尾巴。
- **专用内核线程（actor 模型）** —— vCPU 退出向一个内核线程发消息并阻塞。被拒绝：与锁相同的串行化，但给每个系统调用增加两次线程切换延迟。
- **一个全局内核互斥锁（BEL）。** 客体代码在锁*释放*的情况下执行 —— 这就是并行收益所在，因为程序把绝大多数时间花在用户态。每个进入模拟器 C++ 代码的 VM 退出（系统调用、exec 钩子、MMIO、异常、内存违例）获取 BEL，处理退出，释放它，恢复客体。

**BEL 是设计。** 这正是 QEMU 走过的演进（单线程 TCG → "Big QEMU Lock" + MTTCG），它使几乎所有现有 windows-emulator 代码无需修改就保持正确，因为所有触碰共享状态的东西已经运行在退出上下文中。竞争被系统调用/退出频率所限，剖析已表明对我们关心的工作负载（游戏）而言它远低于客体执行时间。锁粒度可以*稍后*由剖析引导缩减，而无需再一次架构变更（§9）。

客体侧同步不需要我们帮助：客体用户态互锁操作通过共享 GPA 空间作用于真实共享内存，跨 vCPU 原生工作；它们的竞争慢路径是系统调用，由 BEL 串行化。

---

## 3. 核心接口变更（`src/emulator`）

把"机器"和"一个 CPU"分开。今天 `emulator` 多继承 `cpu_interface + memory_interface + hook_interface`（`emulator.hpp:12`）。新形态：

```
emulator (machine: memory_interface + hook_interface + vCPU registry)
 ├── supports_multiple_vcpus() const -> bool
 ├── vcpu_count() const -> size_t              // configured at creation
 └── get_cpu(size_t index) -> x86_64_cpu&

x86_64_cpu (per-vCPU: cpu_interface + typed register helpers + machine access)
 ├── start(count) / stop()
 ├── read/write_raw_register, reg<T>(), save_registers()/restore_registers()
 ├── memory access, delegated to the shared machine
 └── index() const
```

要点：

- **显式 CPU 上下文 —— 没有线程局部状态。** 每个 vCPU worker 线程在运行期间拥有自己的 `x86_64_cpu`，并沿每条调用链显式下传。没有任何东西隐式解析"当前 CPU"；当代码运行在意外线程上没有环境状态会出错。
- **钩子回调获得作为参数的触发 CPU。** `hook_interface.hpp:44-55` 中每个回调类型获得 `cpu_interface&`（在架构层类型化为 `x86_64_cpu&`）作为第一个参数；后端用其退出正被处理的 vCPU 调用回调，始终在该 vCPU 的 worker 线程上。
- **`x86_64_cpu` 是"一个 CPU 看待机器的视图"。** 寄存器和运行控制是每 VP 原生的；内存表面委托给共享机器。今天的系统调用层把一个 `x86_64_emulator&` 同时用于寄存器和内存（`syscall_context::emu`），所以把这个成员以及退出处理路径换成 `x86_64_cpu&` 使数百个 `c.emu.reg(...)` / `c.emu.read_memory(...)` 调用点不变地编译，同时让目标 CPU 显式化。`typed_emulator.hpp` 中的类型化寄存器辅助被分解为一个 mixin 供 `x86_64_cpu` 复用。
- **能力查询：** 模拟器接口暴露 `bool supports_multiple_vcpus() const`。WHP 返回 `true`；unicorn 和 icicle 返回 `false`；KVM 暂时返回 `false`（hypervisor 本身支持，但多 vCPU KVM 推迟到阶段 5，报告不支持直到实现）。FEX 也返回 `false`：其每 vCPU 状态已就位，但只有 `docs/fex-backend.md` 中列出的跨 vCPU 差距关闭后才报告 `true`。
- **创建：** `create_x86_64_emulator(backend_type, emulator_creation_settings{ .vcpu_count })`（`backend_selection.hpp:10-19`）。`vcpu_count` 默认为 `1`。在不支持的后端上请求多于一个 vCPU 是创建时的硬错误 —— 不做静默钳制。
- **线程安全契约，在接口处声明：**
  - 每 vCPU 操作（`start`、`stop`、寄存器访问、save/restore）可以安全地在*不同* vCPU 上并发调用；对一个 vCPU 的寄存器访问只有从其拥有的 worker 线程或在已知停止时才有效。
  - 内存变更（`map/unmap/protect`）和钩子增删需要调用方外部串行化（BEL 提供它）。后端额外地防护其内部分区表（§4），因为一些变更发生在 `start()` 内部的退出处理路径上。
  - 任何 vCPU 上的 `stop()` 保持可从任何线程调用（这正是 `is_stop_thread_safe()` 已经为 WHP/KVM 承诺的）—— 它成为 **IPI/kick 原语**（§5.2）。

`memory_manager` 保持单一共享实例（它已经是，按值持有在 `windows_emulator` 上）；其元数据由 BEL 防护，因为它只从系统调用/退出上下文变更。

---

## 4. WHP 后端变更

参考：以下全部位于 `src/backends/whp-emulator/whp_x86_64_emulator.cpp`。

**每 vCPU（新 `whp_vcpu` 结构，每索引一个）：**
- `virtual_processor_handle`（`:96-118`）—— 按索引创建；`configure_partition()` 中分区 `ProcessorCount = N`（`:1866`）。
- 运行状态：`stop_requested_`、`run_active_`（`:1820-1821`）。
- 单步机制：`pending_execution_step_`（`:1822`）、`pending_mmio_step_`（`:1837`）、`deferred_patched_breakpoint_` / `deferred_execution_page_`（`:1834-1835`）。
- 退出上下文缓冲、每 vCPU 寄存器设置（LSTAR → 共享 HLT 系统调用页，GDT/IDT 基址可指向同一共享结构；系统调用拦截页 `:1948-1957` 是只读且共享的）。
- 所有 `WHvRunVirtualProcessor` / `WHvCancelRunVirtualProcessor` / `WHvGet/SetVirtualProcessorRegisters` 调用参数化为 vCPU 索引而不是文件作用域的 `vp_index`。

**分区共享，需要内部 `partition_mutex_`：**
`mapped_pages_`、`guest_pages_by_gpa_`、`free_guest_gpa_indices_`、`page_table_views_`、`pml4_gpa_`、`next_internal_gpa_`（`:1809-1819`）、`mmio_regions_` + `mmio_read_grace_deadlines_`（`:1836-1838`）、所有钩子映射 + `next_hook_id_` + `syscall_hook_`（`:1824-1839`）、`patched_execution_breakpoints_`（`:1833`）。

为什么即使有 BEL 后端也需要自己的锁：exec 钩子和 MMIO 退出处理（`handle_memory_access` → `remap_hook_page` / `remap_pages`、`:2174-2291`）在 `start()` *内部*修改这些表，即调用方**不**持有 BEL 时。避免锁倒置的纪律：

- 锁顺序是 **BEL → partition_mutex_**，绝不反过来。
- 后端绝不能在持有 `partition_mutex_` 时调用用户回调（回调获取 BEL）。现有的 copy-callbacks-into-a-local-vector-then-invoke 模式（`:3154`、`:3276`）已经支持这一点 —— 只在容器访问周围取锁。
- `WHvMapGpaRange`/`WHvUnmapGpaRange` 是分区级的；Hyper-V 跨运行中的 VP 执行自己的 EPT 失效。我们仍在 `partition_mutex_` 下串行化*我们自己的*调用，使簿记和 hypervisor 状态不会分离。（在阶段 1 中实证验证在其他 VP 运行时重映射的行为；如果不符合，unmap/protect 必须短暂踢醒所有 vCPU —— 一个无论如何都需要的 stop-the-world 辅助，§5.2。）

**exec 钩子单步窗口（已知的、接受的竞争）：**
节区首次执行钩子剥掉一页的 EPT exec；在故障时，处理程序触发回调，临时恢复 exec，通过 TF 单步，再重新保护（`effective_page_permissions :2162`、`arm_execution_single_step :2479`、`complete_execution_step :2527`）。EPT 权限是分区全局的 —— 当 vCPU A 单步越过被钩页时，vCPU B 可以不触发故障地执行该页。后果：

- 对**节区首次执行**钩子这是良性的：回调语义是"首次执行时触发一次"，由 BEL 下 `track_section_first_execution`（`windows_emulator.cpp:741`）中的 `first_execute` 标志检查强制。一次临近的重复故障被过滤；一次错过的额外执行无关紧要。
- 对**精确断点**（gdb stub、调试器、int3 补丁断点 —— 0xCC 字节在单步期间从共享内存移除、`:2408-2477`）这个竞争*不可*接受。解决方案：当精确执行钩子/断点激活时，要么以 `vcpu_count = 1` 运行（analyzer/调试器默认），要么在单步窗口周围使用 stop-the-world 辅助。不要尝试每 vCPU EPT 视图 —— WHP（和 KVM memslot）没有每 VP GPA 映射。

**多 vCPU 下不支持（不变的约束）：** 指令计数（`supports_instruction_counting()` 已经是 `false`、`:1734`）、带精确预算的 `start(count)`、指令精度模式。把这些与 `vcpu_count > 1` 组合是启动时的硬错误（决策：不静默钳制，与不支持的后端规则一致）。

---

## 5. windows-emulator 变更

### 5.1 调度器

`windows_emulator::start()`（`windows_emulator.cpp:1023`）变成协调者：

```
start():
    spawn N vcpu_worker threads (N = min(settings.vcpu_count, emu().vcpu_count()))
    main thread: pump UI events + user timers, until exit/stop     // SDL wants the main thread
    join workers

vcpu_worker(cpu):
    loop:
        lock BEL
        thread = pick_ready_thread()            // round-robin over process.threads, skipping
                                                // threads currently running on another vCPU
        if none: wait on ready_cv (with timeout for timed waits / host-condition polls)
        thread->restore(cpu)                    // blob -> this vCPU
        mark thread running-on(cpu); unlock BEL
        cpu.start()                             // guest runs in parallel, BEL released
        lock BEL
        thread->save(cpu)                       // exits for syscalls etc. acquired BEL internally
        handle wait state / APCs / termination; mark not-running
        unlock BEL
```

- **`vcpu_context` 取代全局"当前线程"概念。** 一个由 `windows_emulator` 拥有的每 vCPU 结构（在普通 vector 中，无全局或线程局部）：
  ```cpp
  struct vcpu_context
  {
      x86_64_cpu& cpu;
      emulator_thread* active_thread{};   // replaces process_context::active_thread
      std::atomic_bool switch_thread{};   // per-vCPU yield request (timer thread writes it)
      bool apc_alertable{};
  };
  ```
  `process_context::active_thread`（`process_context.hpp:484`）和全局 `switch_thread_` 消失。`windows_emulator::current_thread()` 被移除；每条需要它的路径 —— 系统调用处理程序（经由 `syscall_context`）、异常分发、用户回调分发、钩子 lambda —— 显式接收 `vcpu_context&`。钩子回调携带触发的 `x86_64_cpu&`（§3），且 `windows_emulator` 以 O(1) 把 `cpu.index()` 映射到其 `vcpu_context`。这是避免环境状态的主要改动成本：一次宽泛但由编译器引导的、纯粹机械的重构（删除 `current_thread()`，跟着编译错误走）。
- `should_stop` 保持为 `windows_emulator` 上的单一成员原子量。
- `ready_cv` 在线程变为就绪时收到信号：对象被信号（`NtSetEvent`、`NtReleaseMutant`、…）、APC 入队、alert 投递、线程被创建或恢复。这些全部已在 BEL 下运行，所以唤醒是无竞争的。
- **时间片：** 一个定时器线程（今天中断线程的演进，`windows_emulator.cpp:1052-1068`）踢（`cpu.stop()`）每个片到期的运行中 vCPU —— 但仅当 `ready_threads > running_vcpus`，因此提交不足的客体永远不用付抢占退出。
- **设备泵：** `perform_context_switch_work()`（`:301-310`，设备 `work()`、线程收割）移到调度器 tick，由下一个撞上它的 worker 在 BEL 下执行（基于截止期），因为 BEL 串行化它而保持 `block_mutation` 不变量。
- **`start(count)`**（指令预算，被 analyzer 的 `-c` 和 gdb 单步使用）与 `vcpu_count > 1` 不兼容 —— 启动时硬错误。
- **N=1 走同一个调度器**（决策）：一个 BEL 无竞争的单 `vcpu_worker` 完全取代旧循环，因此恰好只有一条调度代码路径。阶段 2 必须在合并前对 main 基准 N=1 启动墙钟时间。

### 5.2 跨 vCPU 操作需要踢（IPI 等价物）

今天，定位*另一个*线程总是修改一个已停放的线程。现在目标可能在另一个 vCPU 上执行。以下全部在 BEL 下运行，并用目标 vCPU 上的 `cpu.stop()` 作为踢：

- `NtAlertThreadByThreadId` / `NtQueueApcThread`（`syscalls/thread.cpp:555,974`）：设置标志/排队 APC（BEL 使其安全），然后踢目标的 vCPU，使可警报等待或 APC 投递迅速。
- `NtSuspendThread` / `NtTerminateThread`：踢，worker 在下一次退出观察到 `suspended`/`exit_status` 时停放该线程。
- 对*运行中*线程的 `NtGetContextThread` / `NtSetContextThread`：踢目标 vCPU，等待（在 BEL 下，经由一个条件）直到 worker 保存了数据块，然后操作 `last_registers` —— 对已停放线程已存在的机制。
- **Stop-the-world 辅助**（快照、精确断点步进、可能 unmap 需要）：设置全局暂停标志，踢所有 vCPU，等每个 worker 停放了它的线程，运行临界动作，释放。

### 5.3 BEL 不必修的东西

- 客体对客体内存竞争：与真实硬件相同；客体自己的问题。
- 系统调用处理程序读取另一个 vCPU 并发写入的客体内存：与真实内核相同的 TOCTOU 暴露；处理程序已经通过 `emulator_object` copy-in/copy-out。
- KUSD：读取是 MMIO 退出 → BEL 串行化的 `update()`（`kusd_mmio.cpp:157`）。最初无需变更。

### 5.4 较小事项

- `executed_instructions_`（`windows_emulator.hpp:114`）→ 原子量（或每 vCPU 计数器在读时求和）。
- `handle_store` 引用计数（`handles.hpp:147`）保持非原子 —— 所有变更都在 BEL 下。在该类上记录这个不变量。
- `logger` 需要内部互斥锁（否则行交错）。
- 客体可见 CPU 身份（决策：**绑定到 `vcpu_count`**）：vCPU 数驱动 `PEB.NumberOfProcessors`（`process_context.cpp:360`）、`KUSER_SHARED_DATA.ActiveProcessorCount`（`kusd_mmio.cpp:72`）及派生的亲和性掩码；`fake_env.number_of_processors` 成为反分析实验的显式覆盖。`NtGetCurrentProcessorNumberEx`（`thread.cpp:916`）返回执行该系统调用的 vCPU 的索引（`c.cpu.index()`）。Affinity/priority 仍是 no-op。
- **序列化/快照**（决策：**v1 中限定 N=1**）：`vcpu_count > 1` 时 `save_snapshot()`/`serialize()` 报错，直到阶段 3 的 stop-the-world 静默落地。每线程寄存器数据块已经可以序列化；当前被序列化的中断线程原子量（`windows_emulator.cpp:1315`）变成恢复时重置的每 vCPU 状态。
- **gdb-stub / 调试器**（`x86_64_gdb_stub_handler.hpp`、`debug_session.cpp`）：与 `vcpu_count > 1` 不兼容 —— 组合时硬错误。多 vCPU 调试（每 vCPU 线程 id、stop-the-world 步进）是更晚的独立功能。
- **UI 事件：** `deliver_raw_input` / `handle_ui_event`（`windows_emulator.cpp:1103`）从主线程调用 —— 它们取 BEL。

---

## 6. 钩子系统线程安全

发现：钩子在每个后端都未加锁；钩子合法地从其自己的回调内部被删除（`track_section_first_execution`、`windows_emulator.cpp:748-752`）；只有 icicle 有延迟增删（`icicle_x86_64_emulator.cpp:468,645`）。

新模型下的规则：

1. **注册/删除由契约 BEL 串行化**（§3）。所有当前的安装/移除点（全在 `setup_hooks()` `windows_emulator.cpp:823` 和节区首次执行路径）已经运行在退出上下文或 worker 启动之前。
2. **分发容器获得后端 `partition_mutex_`**，用于从多个 worker 并发查询它们的退出处理路径（`handle_execution_hook :3144`、`handle_instruction_exit :3015`、`handle_memory_access :3190`、`handle_syscall_halt :1959`）。copy-then-invoke（已是 `:3154/:3276` 的模式）使回调在锁外运行。
3. **从回调删除继续工作**：回调在持有 BEL 时运行；删除在 `partition_mutex_` 下变更容器；其他 worker 只在同一把锁下读取容器并遍历它们的本地副本。一个被删除的钩子可能仍从另一个 vCPU 已取走的副本触发一次 —— 在乎的回调（节区首次执行）在 BEL 下按检查幂等。
4. `scoped_hook`（`scoped_hook.hpp`）无需变更 —— 析构发生在持 BEL 的路径上。
5. **系统调用钩子**保持特判：`syscall_hook_`（`:1839`）在 worker 存在之前于设置时写一次；并发分发是只读的 → 热路径无锁。

---

## 7. 并发纪律 — 无竞争与避免死锁

适用于整个项目的一般设计规则：**没有可变的全局状态，且 `thread_local` 也算全局。** 每一块共享状态都是一个具有明确所有者的对象的成员，且正在执行的 CPU/线程身份总是显式传递（§3、§5.1）。这不仅是风格 —— 它使下面的所有权分类*可检查*，因为没有环境状态可以从错误的线程不可见地被触碰。

### 7.1 数据竞争自由：每个对象恰好得到一个所有权类别

竞争自由不是靠撒锁实现；它靠把**所有**可变状态归入六个类别之一并强制每个类别的访问规则实现：

| 类别 | 规则 | 示例 |
|-------|------|----------|
| **vCPU 本地** | 只被其拥有的 worker 线程触碰。*唯一*的跨线程操作是 `stop()`（踢）。 | `x86_64_cpu` 寄存器/运行状态、WHP 每 VP 步进状态、`vcpu_context`（其原子量除外） |
| **内核状态（BEL 域）** | 只在持有 BEL 时触碰。 | `process_context` + 所有句柄存储、`memory_manager` 元数据、`module_manager`、设备、调度器队列、跨线程可见的 `emulator_thread` 字段 |
| **分区状态（后端）** | 只在持有后端 `partition_mutex_` 时触碰。 | WHP GPA/页表、钩子分发容器、MMIO 区域 |
| **设置后冻结** | 只在 vCPU worker 生成前写入；之后只读，可无锁并发安全读取。 | 系统调用处理程序表（`syscall_dispatcher::handlers_`）、模块导出、设置、系统调用钩子指针 |
| **指定原子量** | 逐个记录，各带顺序语义。 | `should_stop`、`layout_version_`、每 vCPU `switch_thread`、指令计数器 |
| **客体内存** | 按设计竞争，像物理 RAM。内核代码必须 copy-in/copy-out（`emulator_object` 已如此）且绝不假定两次读取之间稳定。 | 所有客体页 |

强制执行，使纪律在现实中存活：

- **Owner 追踪锁包装。** BEL 和 `partition_mutex_` 被包装在一个记录拥有者 `std::thread::id` 的小类中（包装器的成员 —— 不是线程局部）。每个核心变更者在入口断言所有权：`handle_store::store/erase`、所有 `memory_manager` 变更者、`process_context::create_thread/terminate_thread`、设备容器变更。调试构建在违例时中止；该检查便宜到足以在 bring-up 阶段保持启用。
- **在边界获取，在内部断言。** BEL 在每个入口点恰好取一次（VM 退出处理程序、UI 事件投递、调度器循环迭代）。内部代码从不获取它 —— 它断言自己已持有。不用 `recursive_mutex`：尝试嵌套获取是一个 bug，owner 断言会立即抓住它，而不是一个递归锁会掩盖的设计异味。
- **静态检查：** 核心结构上的 clang `-Wthread-safety` 注解（`GUARDED_BY(bel_)` / `REQUIRES(bel_)`）。本地 clang-cl 安装（clang 21、`x86_64-pc-windows-msvc`）可用于开发；因为 clang-cl 兼容 MSVC，它也能构建 WHP 后端，所以分析覆盖 Windows 上真实的多 vCPU 代码路径。不需要专用预设 —— 在配置时覆盖编译器（`-DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl` 加上 `-Wthread-safety`）用于开发/验证构建，并在阶段 2-3 保持零警告。注解宏在 MSVC 上编译为空。
- **动态检查：** 阶段 5 落地后在 Linux/KVM 构建上用 ThreadSanitizer（Windows 上没有 TSan，clang-cl 下也没有）；在 Windows 上，阶段 4 客体压力测试加 owner 断言是实际的网。
- **评审规则：** 永远不允许新的静态/全局/线程局部可变状态；每个新成员字段恰好属于上面一个类别，不明显时注释说明。

### 7.2 避免死锁：一个固定、微小的锁层级

设计刻意只有**三级锁**，严格排序。持锁线程只能获取严格更高层的锁：

```
L0  BEL (windows_emulator kernel mutex)     — outermost
L1  partition_mutex_ (one per backend)      — may be taken under the BEL, never above it
L2  leaf mutexes (logger)                   — never call out while held
```

使层级充分的规则：

1. **BEL 绝不持着执行客体代码**（`cpu.start()`）或阻塞在宿主机 I/O 上。BEL 下的所有等待经由绑定到它的条件变量（`ready_cv`、stop-the-world 屏障、`NtGetContextThread` 中的强制保存等待）—— CV 等待原子地释放 BEL，所以被等待的线程总能获取它并推进。这从构造上关闭了经典的"A 持锁等 B，B 需要锁"循环。
2. **`partition_mutex_` 只是簿记锁**：在容器/EPT 表访问周围持有，绝不在用户回调间（回调获取 BEL → 会倒置 L0/L1）、绝不在 `WHvRunVirtualProcessor` 间、绝不在任何阻塞操作间持有。现有的 copy-callbacks-then-invoke 模式（`whp_x86_64_emulator.cpp:3154,3276`）形状已经正确。
3. **踢是非阻塞的。** `WHvCancelRunVirtualProcessor` / `pthread_kill` 从不阻塞，所以在任何锁下信令另一个 vCPU 都安全。任何必须*等待*被踢 vCPU 反应的事都通过 BEL 上的 CV 做（规则 1），绝不带锁自旋。
4. **叶子锁绝不外呼。** logger 互斥锁只包格式化+写入。
5. **增加互斥锁必须给它指定层级**（文档化在 BEL 旁边）。预期的稳态是没人需要：阶段 4 的竞争修复应该优先分片数据或缩小 BEL 范围，而不是引入带新排序义务的新锁。

跨锁顺序违例（持有 `partition_mutex_` 时获取 BEL）无法被包装器自己的 owner 字段单独检测，但如果规则 2 成立它们在构造上不可能 —— 后端只在从不向上调用的叶子簿记函数中取 `partition_mutex_`。该性质可通过对每个后端文件内一个小的、封闭的获取点集合的评审强制。

## 8. KVM 后端（次要）

在此阶段落地前，KVM 后端报告 `supports_multiple_vcpus() == false`，并像 JIT 后端一样拒绝 `vcpu_count > 1`，即使 hypervisor 本身可以支持。

§4 的镜像，结构上完全相同：

- N 个 `KVM_CREATE_VCPU` fd + 每 vCPU `kvm_run` mmap（`kvm_x86_64_emulator.cpp:1362-1380`）。
- 每 vCPU：`vcpu_thread_`、踢信号（`pthread_kill`、`:577-594` —— 已经是正确的每线程取消模型），以及关键的**寄存器缓存**（`regs_cache_`/`sregs_cache_`、`:2231-2266`），它们是每 vCPU 状态。
- VM 全局：memslot、GPA 表、钩子映射 → 同样的 `partition_mutex_` 处理。
- 设计刻意让所有 KVM 特有的东西不进 windows-emulator：调度器/BEL 只依赖 §3 的接口契约。

Unicorn 和 icicle 保持 `supports_multiple_vcpus() == false`；N=1 时统一调度器运行单个 worker，相同的让出点、无竞争的 BEL —— 功能上等同于今天的循环。

---

## 9. 实现阶段

每个阶段都要能构建、通过 `analyzer.exe -s test-sample.exe`，且可独立合并。

**阶段 0 — 机械重构，零行为变化。**
从模拟器接口抽取 `x86_64_cpu`（后端把它们的单个 CPU 暴露为索引 0）；给所有钩子回调签名加触发的 `x86_64_cpu&` 参数；引入 `vcpu_context` 并把它穿进 `syscall_context`、异常分发和用户回调分发，删除 `windows_emulator::current_thread()` 和 `process_context::active_thread`（编译器引导）；给所有后端加 `supports_multiple_vcpus()`；把 `vcpu_count`（默认 1）贯穿 `backend_selection` 和 `emulator_settings`，对不支持的后端硬报错。仍恰好一个 vCPU 一个宿主机线程。风险：低（无行为变化，编译器驱动重构）。改动：大但机械。

**阶段 1 — WHP 后端内部走向 N vCPU。**
`whp_vcpu` 结构、按索引创建 VP、`ProcessorCount = N`、参数化的寄存器/运行/取消调用、`partition_mutex_` 环绕分区表和钩子分发容器。windows-emulator 仍只用 CPU 0 → 无可见变化。加一个后端级单元测试：创建 N 个 VP、在两个宿主机线程上于不相交内存中运行琐碎代码、压测并发 map/unmap。实证验证在其他 VP 运行时 `WHvMapGpaRange`/`WHvUnmapGpaRange`。

**阶段 2 — BEL + 调度器 worker（大项）。**
引入内核互斥锁；在每个由 `setup_hooks()` 安装的模拟器入口回调和 UI/外部入口点获取它；用 `vcpu_worker` + `ready_cv` + 定时器踢线程取代单个运行循环；每 vCPU `switch_thread_`；调度器 tick 上的设备泵；`logger` 互斥锁；原子指令计数器。以设置暴露（`--vcpus N`，默认 1）。N=1 走同一个调度器（一个 worker、无竞争 BEL）—— 合并前对 main 基准 N=1 启动墙钟时间。启动校验：当 `vcpu_count > 1` 与不支持的后端、gdb-stub、指令精度、指令预算、相对时间模式或快照（直到阶段 3）组合时硬错误。

**阶段 3 — 跨 vCPU 正确性。**
对运行中线程的 alert/APC/suspend/terminate 踢；`NtGet/SetContextThread` 的强制保存；stop-the-world 辅助；快照静默（解除 N=1 快照限制）；客体可见 CPU 身份（KUSD 计数、`NtGetCurrentProcessorNumberEx`）。

**阶段 4 — 验证 + 性能。** *(done)*
客体压力测试：`src/samples/stress-sample` —— 8 个线程猛砸互锁操作、一个 `CRITICAL_SECTION` 保护的计数器、一个信号量生产者/消费者、自动重置事件乒乓、以及带写/读验证的 `VirtualAlloc` 搅动，各自断言一个确定性不变量。在 WHP 上的 `--vcpus 1..8`、x64 和 WoW64（x86）都通过；未发现新竞争。

BEL 竞争分析：`kernel_lock` 在 `SOGEN_LOCK_PROFILE` 下插桩（获取次数 / 竞争次数 / 等待时间 / 持有时间，运行末汇总；未设置时零开销）。实测竞争（竞争获取 ÷ 总获取）：

| 工作负载                    | N=2   | N=4   | N=8   |
|-----------------------------|-------|-------|-------|
| 纯同步压力 | 43.5% | 51.6% | 53.7% |
| 真实游戏（open-iw5/MW3）    | 1.4%  | 2.6%  | —     |

持有时间跨 vCPU 数保持约不变（压力约 4.7 s，游戏约 5–6 s）—— BEL 串行化固定量的工作。压力数字是最坏情况（工作负载几乎全是系统调用）；游戏以客体计算为主，所以 BEL 只有 1–3% 的时间有竞争，**不是**目标工作负载的瓶颈。

定向锁细化：**不值得**。按上面的规则（"只在有测量时"），细化 1–3% 游戏负载竞争是投机性的，会带来新的锁顺序 bug，所以故意不做。如果未来一个系统调用密集型工作负载证明不然，剖析器会指明方向，候选仍是 KUSD 读取、热设备 ioctl（GPU 桥）和 `memory_manager` 读取路径（`shared_mutex` 拆分）。

**阶段 4b — VM 宿主机上的 WHP 映射成本。** *(done)*
在嵌套虚拟化宿主机上，已映射客体页的首次触摸在 hypervisor 内部花费约 90 µs，且每当页被解除映射再映射时会重新支付。分配搅动因此被 EPT 填充主导：

- 在每次 `WHvMapGpaRange` 后发出 `WHvAdviseGpaRange(Populate)`（运行时解析，尽力而为）。填充 32 MB 范围约需 5 ms，而不是约 2.7 s。
- 释放的读/写页保持映射。`unmap_memory` 把它们停进 `retired_pages_`（GPA、宿主机页和 EPT 映射完好），`map_memory` 在清零后重新发放。一页只有在每个在该页退役时正在运行客体代码的 vCPU 都应用了退役所在 epoch 的 TLB 刷新后才可复用（`requested_tlb_epoch_` / `acked_tlb_epoch_`，一个 Dekker 风格的 `pending_tlb_flush_` / `run_active_` 握手，而不是取消空闲 vCPU）。`remap_pages` 跳过 EPT 标志已匹配的页。

**阶段 5 — 次要目标。**
KVM 对齐（§8）；linux-emulator 采用相同的调度器/BEL 形态（它镜像 windows-emulator 结构；考虑在那时把 vCPU worker/调度器核心提取为共享组件，而不是之前）。

---

## 10. 风险与开放问题

- **WHP 运行中重映射语义**（§4）。如果并发 VP 执行期间 `WHvUnmapGpaRange` 行为异常，每次 unmap/protect 都需要 stop-the-world —— 正确但更慢；`remap_pages` 合并（`:2196`）限制了频率。阶段 1 解决。
- **BEL 竞争。** 如果一个工作负载结果是系统调用密集型的，并行收益会缩水。缓解路径是 §9 阶段 4；架构不变。
- **单 vCPU 回归。** BEL 在 N=1 无竞争（acquire 上的一次 fetch-and-store），但阶段 2 必须在合并前对 main 基准 N=1 启动墙钟时间，因为 N=1 走新调度器。
- **潜在的单线程假设。** Grep 审计找不全（如系统调用处理程序中的静态局部变量、跨调用复用的暂存缓冲 —— #1172 的 gpu-bridge 暂存缓冲是每 `windows_emulator` 的，没问题，但同样的模式在其他地方可能不行）。阶段 4 压力测试是真正的网。
- **并发下的等待语义。** `is_thread_ready` / `consume_object_signal`（`emulator_thread.cpp:72,176,935`）保持 BEL 防护，所以"恰好消费一个信号"保持原子。但*唤醒顺序*变化：今天轮询位置决定谁赢得一个已信号的自重置事件；有了并行 worker，是哪个 worker 先挑到该线程。Windows 不做顺序承诺，所以这是合法的 —— 但它可能暴露被确定性调度掩盖的潜在客体 bug。保留 `--vcpus 1` 作为确定性复现回退。
- **确定性 / 快照。** 多 vCPU 运行本质上不可复现；快照/恢复保持支持（静默），但重放式调试需要 N=1。
- **WOW64。** Heaven's-gate 转换（`wow64_heaven_gate.hpp`）是数据块内部的每线程寄存器状态 —— 预期无额外工作，但 32 位路径必须在阶段 4 压力矩阵中（open-iw5、t6mp 是实用试验台）。
