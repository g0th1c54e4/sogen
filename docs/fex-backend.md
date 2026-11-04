# FEX 后端 — 开发笔记

FEX-Emu 模拟器后端（`sogen::fex`）。这是一个独立的后端，结构与其他后端镜像对应，在运行时通过 `EMULATOR_FEX=1` 选择。其功能目标是 **Apple Silicon 上的 macOS** 和 **AArch64 上的 Android** —— [FEX](https://fex-emu.com) 只把 x86/x86-64 JIT 到 ARM64。

## 状态

在 Darwin/Apple Silicon 上可用：`test-sample.exe` 能完整运行，行为与 Unicorn 后端一致。Android/AArch64 现在也是一个功能目标，目前要求宿主机页面为 4KB。其他 AArch64 Linux 宿主机仅有构建覆盖，不期望能运行。

## 安全 / 地址空间模型

**FEX 共享客体与宿主机的地址空间 —— 两者之间没有隔离。** 与 Unicorn/Icicle（完全软件模拟的客体地址空间，在构造上与宿主机进程沙箱隔离）或 KVM/WHP（真正的硬件虚拟化，拥有真正独立的客体物理地址空间）不同，FEX 是进程内 JIT：它把 x86/x86-64 翻译为 ARM64 并直接在本进程内执行结果，把客体虚拟地址当作宿主机虚拟地址对待（1:1 映射，`guest VA == host VA`，WoW64 进程的低 4GiB 除外，见下文）。具体而言：

- `map_memory()` 是在客体地址上执行真正的 `mmap(MAP_FIXED)`；一旦知道范围已映射，`read_memory()`/`write_memory()` 就是直接的宿主机 `memcpy` —— 在客体访问与本进程自身内存之间没有页表或边界检查。
- 因此，客体的越界读或写可以直接落进本进程自身的内存 —— 包括 sogen 自己的运行时状态 —— 这在 Unicorn/Icicle 下是构造上不可能的。这是 FEX-Emu 设计工作方式的固有属性（它被构建为透明的 Linux binfmt 风格翻译层，与 `qemu-user`/`box64` 同类，"客体指针就是宿主机指针"正是实现系统调用直通和原生速度执行的关键），而不是这个移植特有的缺陷。参见 [FEX-Emu/FEX 自己的 `docs/ProgrammingConcerns.md`](https://github.com/FEX-Emu/FEX/blob/main/docs/ProgrammingConcerns.md) 和 `docs/allocator_usage.md`，FEX 自己也明确说明了这一点。
- 该后端自身的地址空间保护（`windows-emulator/memory_manager.cpp` 中的 `host_reserved` 区域追踪，以及仅 Apple 平台的 FEXCore 内部 arena）是缓解措施，而不是沙箱：它们阻止普通的客体分配*落在*该后端或宿主机进程需要的地址上，降低碰撞风险。它们无法、也不能阻止客体计算出指向任何其他有效的宿主机已映射内存的任意指针 —— 按照设计，对此没有硬故障边界。
- 实际意义上，这意味着 FEX 不适合运行你完全不信任其会留在自己内存范围内的客体代码 —— 这类工作负载正是 Unicorn/Icicle 沙箱模型所适合的。
- `deps/CMakeLists.txt` 显式禁用了 FEX 自己的内部分配器（`-DENABLE_FEX_ALLOCATOR=OFF`、`-DENABLE_JEMALLOC_GLIBC_ALLOC=OFF`）—— 客体分配由 sogen 的内存管理器拥有。在 Apple 上，该后端还会单独安装 FEXCore 分配器钩子，把 FEXCore 自己的匿名内部映射引导进一个专用的 4GiB 宿主机保留 arena，使其避开客体分配范围。这仍然只是碰撞缓解，不是隔离。
- WoW64 进程的低 4GiB 由一个独立的 4GiB 宿主机窗口支撑，FEX 会把低于 4GiB 的每个客体访问重定位到该窗口中，因为宿主机可能已经在使用该范围（Android 的 ART 堆）或者根本无法在该处映射（Darwin 的 `__PAGEZERO`）。别名到低 4GiB 的宿主机内存会在该窗口内被第二次映射，而不是出现在它自己的地址上。

## 架构

- `fex_x86_64_emulator.hpp`/`.cpp` — 工厂函数 `sogen::fex::create_x86_64_emulator()` 和后端本体；文件顶部的注释包含上述模型的实现级版本。
- `fex_x86_64_common.hpp` — 寄存器分类（`classify_gpr` 及其相关函数），把 `x86_register` 映射到 FEXCore 的 `CPUState`。
- 客体的 `syscall` 指令通过 `FEXCore::HLE::SyscallHandler` 路由回 sogen，由它调用已注册的系统调用指令钩子 —— 与其他所有后端使用的机制相同。
- 细粒度的 `hook_memory_read/write/execution/range_execution` 和 `hook_basic_block` 钩子仅为 API 兼容而被接受，但永远不会触发，与 KVM 后端完全一样：客体代码以原生方式运行，除了单步执行之外没有按访问/按指令插桩的切入点。
- `deps/FEX` 固定在一个个人 fork（`github.com/JackTYM/FEX`）上，而不是直接跟上游 FEX-Emu —— 它带有一个移除 10 个子模块的改动（精简 376MB）和几个小的 Darwin 可移植性/opdispatch 补丁；精确列表见该 fork 自己的提交历史。**固定到 fork 意味着这些补丁不会自动跟随上游 FEX-Emu 的安全修复** —— 定期变基到较新的上游版本是值得权衡的维护成本。

## 构建与测试

与 KVM 后端不同（KVM 需要 Docker-on-Windows 的变通方法才能构建或运行），无需特殊设置：在 ARM64 Clang 宿主机（Apple Silicon macOS 或 ARM64 Linux）上，一旦 `deps/FEX` 检出完成，普通的 `cmake --preset=release` 就会自动带上这个后端（见根 `CMakeLists.txt` 中的开关）。用 `EMULATOR_FEX=1` 运行。Android 目前要求宿主机页面为 4KB。

## 已知限制

- **对 `PAGE_NOACCESS` 页的读-改-写会被作为读错误报告给客体。** 故障按 ESR 的 WnR 位分类，而对于同时读和写的指令，Arm 会先报告触发故障的读操作。因此，在不可访问页面上执行的 `lock xadd`、`xchg`、`lock cmpxchg` 或普通的 `add [mem], reg` 会抛出 `STATUS_ACCESS_VIOLATION` 且 `ExceptionInformation[0]` = 0，而 Windows 会报告为写错误。对只读内存的存储（包括 store）是写故障。当没有可用的描述数据中止的 ESR 时，仅含 `STLR` 的 `decode_arm64_store` 仍是后备方案。
- **`sync_host_page_apple` 的权限并集可能在 PE 节边界附近剥掉 `PROT_EXEC` 或过度授予写权限。** 客体权限按每 4KB 影子槽追踪，但按每 16KB Apple 宿主机页面（四个槽共享一个宿主机页面）应用；当共享一个宿主机页面的各槽意见不一致时，代码对它们取并集，所以只读槽挨着可写槽时会保持可写。PE 的 `.text`（RX）紧跟 `.data`（RW）落入同一个宿主机页面时，代码页尾部可能丢失执行权限 —— 这是真实的执行故障，而不仅仅是过度宽松的只读页。
- **混合宿主机页面中只有少数几个守卫页和不可访问的 4KB 页面会触发故障，而保留页不会。** 在 macOS 上，与可访问内存共享一个 16KB 宿主机页面的守卫页或 `PAGE_NOACCESS` 4KB 页，是通过在运行客体的宿主机线程上用调试寄存器捕获的：按 `hw.optional.watchpoint` 报告的尽可能多的观察点，优先用客体堆栈指针下方最近的宿主机页填充。其他所有这类页面，以及已提交内存旁的每个保留页面，都像以前一样可访问。被监视页面上的 `clflush`/`clwb` 不会触发故障，因为 Arm 缓存维护不会触发观察点。当启动探测未看到被监视的 4KB 页面触发故障，或启动时附加了调试器时，观察点保持关闭；之后附加的调试器会自己收到命中。在它们被布防期间，每次 JIT 写保护切换会多花约 15 ns，这会稍微拖慢编译密集阶段。
- **MMIO 故障路径并非完全异步信号安全。** `handle_mmio_fault` 在真正的信号处理程序内部直接调用已注册的 MMIO 回调，而不是经由本文件自己的 `pending_fault_dispatch_` 机制（该机制正是为这类风险设计的，并且已用于 `memory_violation_hooks_`/`interrupt_hooks_`）。唯一的生产注册方 `kusd_mmio::read` 会获取 `kusd_mmio::mutex_`。页面被拦截期间，另外两个持有者是 `kusd_mmio::access` 的两个重载（系统调用在正常上下文中调用它们 —— QPC 频率、系统根目录、处理器数量和特性），以及（一旦 FEX 运行多个 vCPU）其他 vCPU 上的 `read`。因此，一个触发故障的 vCPU 的处理程序可能等待另一个 vCPU 的 `read` 或 `access` 内部的系统调用。之所以是等待而不是死锁，仅仅因为 `read` 最多以共享方式持有表锁，且任何 `access` 仿函数在持有互斥锁时都不会触碰客体内存或获取表锁。一个做这两件事中任何一件的 `access` 调用方都会与该处理程序死锁。该处理程序仍然阻塞在一个互斥锁上。
- **还有一个更小的、同类 bug 的差距：** `commit_memory` 对提交进 `section_kind`/`host_reserved` 区域有防护，但对客体直接探测其他后端保留范围的情况没有等价的防护。

## 多 vCPU

FEX 仍然报告单个 vCPU：`supports_multiple_vcpus()` 为 false，所以 `--vcpus` 大于 1 会被拒绝。每 vCPU 的基础工作已经就绪。`sogen::fex::create_x86_64_emulator(vcpu_count)` 会构建 1 到 64 个 vCPU（一个处理器组），每个都有自己独立的 FEXCore 线程、GDT、备用信号栈，在 macOS 上还有自己的 JIT 写保护重试槽和已布防的观察点；后端选择总是请求一个。系统调用由在调用线程上运行的 vCPU 服务，翻译后的代码在每个 vCPU 的 FEXCore 线程中失效，在 macOS 上，一个改变了不可访问页面的内存调用会暂停其他 vCPU 一次，让它们布防新的观察点。
只有在以下差距都关闭之后，`supports_multiple_vcpus()` 才能变为 true：

- **快照被拒绝。** `serialize_state`/`deserialize_state` 抛出 "Multi-vCPU snapshots are not supported yet"，与 WHP 相同。
- **宿主机拷贝可能为整个进程放宽保护。** 缺少所需权限的宿主机对客体内存的读或写会在拷贝时改变宿主机保护，因此拷贝期间另一个 vCPU 对该内存的客体访问不会触发故障。
- **Linux 被追踪内存存在刷新窗口。** 把过期的被追踪块刷新到最新状态的缓存刷新需要该块可读，所以另一个 vCPU 可以在保护变更和刷新之间读到它的旧缓存行。
- **为宿主机代码模拟的未对齐 `LDAR`/`STLR` 可能发生撕裂。** `handle_misaligned_atomic_fault` 中的后备方案保持了半屏障顺序，但用普通的 load 和 store 拷贝值，另一个 vCPU 可能观察到撕裂的值。
- **有支撑的 `KUSER_SHARED_DATA` 刷新与客体读取者存在竞争。** 在 WoW64 进程的低窗口内，该页面由每个 vCPU 在进入客体之前和系统调用之后都会刷新的宿主机内存支撑，而其他 vCPU 的客体代码可能正在读取它。该拷贝不遵循内核的 `KSYSTEM_TIME` 写入顺序。

## bring-up 和评审期间发现的修复

- **Apple `mmap(MAP_FIXED)` 语义下的宿主机保留地址空间追踪。** 两个真实的内存安全 bug：一个 MMIO 区域插入路径可能违反宿主机保留的重叠追踪不变量（直到一次 `mmap(MAP_FIXED)` 失败导致 analyzer 崩溃才被发现），以及一个 decommit→recommit 窗口可能让外来的宿主机分配落进客体仍被保留的范围。修复：正确地雕刻宿主机保留空洞，并增加 `memory_interface::release_guest_address_range()` 来区分 decommit（保留地址声明）和 release（真正释放）。
- **调度量子竞争**（与后端无关，拆分为独立修复 —— 见 `windows_emulator.cpp` 中 `start()` 的量子定时器看门狗）：非原子的 `switch_thread = true` / `cpu.stop()` 组合可能被调度器自己的消费步骤拆开，被误读为致命崩溃。该后端协作式的页面保护停止使这个竞争热到可以可靠命中，它就是这样被发现的。
- **`InterruptFaultPage` 展开的原子性。** 一个竞争中的中断故障页展开（该后端用来中断 JIT 编译代码的机制）在并发访问下可能被 `start()` 的运行循环误读为致命停止；改为打上标记并重新布防，而不是终止。
- **`int 2Dh`（Windows 调试服务陷阱）在错误的地址恢复。** FEXCore 只对真正的 `INT3`/`INT1` 报告 RIP 已经越过触发陷阱的指令（FEXCore `OpcodeDispatcher.cpp` 中的 `SetRIPToNext`）；该后端无法直接分发的通用 `INT n`（从一个合成的 `#GP` 重映射回它自己的向量）报告的已经是该指令自身的起始地址。一段无条件按 INT3 情形修正的共享代码修复方案破坏了这个情况。修复：在该后端中规范化，并以 FEXCore 的 `TrapNo == X86_TRAPNO_BP` 为条件（经验证，整个 FEXCore 中只有一处设置它 —— 真正的 `0xCC` 情形）—— 这使得共享代码的变通方案可以直接删除，而无需继续修补。
- **缺少 GPR 子寄存器宽度。** `classify_gpr` 没有 `r8b`-`r15b`/`r8w`-`r15w`/`r8d`-`r15d` 的分支（只有全部 64 位宽度和原有 8 个寄存器的每个宽度）—— 读取时静默零填充，写入时静默丢弃。在真实的 x86-64 代码中很常见（例如 `mov r9d, eax`）。
- **`setup_gdt` 丢弃了分配失败。** GDT 因此位于一个固定地址，而在 Apple 上，低地址和一系列典型的宿主机分配不可用/容易碰撞（见上面的地址空间模型），所以那里的碰撞是真实的，而不是理论上的，并且失败是静默的，而不是抛出可诊断的错误。GDT 现在通过内存管理器分配，后者会与宿主机确认该地址。
