# src/ 模块说明

本文档梳理 `src/` 下每一个模块（目录 = CMake 目标）的作用。模块按架构分层组织：先有依赖关系总览，再按层分组详述每个模块的职责与关键文件。

## 总览（按字母序）

| 模块 | 一句话作用 |
|---|---|
| `analyzer/` | 统一入口可执行文件 `analyzer.exe`，按环境变量分发到 Windows/Linux/macOS 主函数。 |
| `android-runtime/` | Android 应用的原生运行时（JNI 会话、UI 后端、键位映射、伪终端）。 |
| `backend-selection/` | 运行时 CPU 后端选择（`EMULATOR_*` 环境变量 → 后端工厂）。 |
| `backends/` | 七个 CPU 执行后端（Unicorn/Icicle/WHP/KVM/FEX/HVF/WHP-ARM64）。 |
| `common/` | 跨平台共享工具：网络封装与通用工具集。 |
| `debugger/` | 浏览器集成调试器核心（`debug_session` + FlatBuffers 事件协议）。 |
| `disassembler/` | Capstone 反汇编器封装。 |
| `emulator/` | 后端无关的模拟器核心接口（机器/CPU/内存/钩子/序列化）。 |
| `emulator-platform/` | 平台抽象层（进程、线程、UI、音频、控制台等宿主能力接口）。 |
| `fuzzer/` | 独立的模糊测试可执行入口（驱动 windows-emulator + Icicle）。 |
| `fuzzing-engine/` | 可复用的模糊测试引擎（输入生成、随机源）。 |
| `gdb-stub/` | 后端无关的 GDB 远程串行协议实现。 |
| `gpu-bridge-protocol/` | GPU 半虚拟化的线上协议头（IOCTL、请求/响应结构、特性链映射）。 |
| `linux-analyzer/` | Linux 模拟的命令行前端。 |
| `linux-debugger/` | Linux 模拟的调试器事件处理（Emscripten 路径）。 |
| `linux-emulator/` | Linux 用户空间模拟器本体（ELF 加载、系统调用、信号、procfs、vDSO）。 |
| `linux-emulator-test/` | Linux 模拟器的 gtest 套件（含 torture 压测）。 |
| `linux-gdb-stub/` | Linux 模拟器的 GDB 桩处理器（x64/x86 寄存器映射）。 |
| `macos-analyzer/` | macOS 模拟的命令行前端。 |
| `macos-dyld-test/` | dyld 共享缓存解析的测试工具。 |
| `macos-emulator/` | macOS 用户空间模拟器本体（Mach-O、dyld 缓存、Mach 端口、MIG）。 |
| `macos-emulator-test/` | macOS 模拟器的 gtest 套件。 |
| `posix-emulator/` | Linux/macOS 模拟器共享的 POSIX 层（fd 表、文件系统、路径解析）。 |
| `process-manager/` | 宿主进程管理抽象（通过 FlatBuffers 协议驱动真实子进程）。 |
| `python-bindings/` | nanobind Python 绑定（`sogen` 模块的全部 Windows/Linux API）。 |
| `samples/` | 客体侧样例与 shim（test-sample、hook-sample、vulkan-shim、steam-shim 等）。 |
| `steam-bridge-protocol/` | Steam 桥的线上协议头（IOCTL 与请求/响应结构）。 |
| `steam-generated/` | Steam 桥生成器输出（**gitignore**，Valve 衍生，本地重新生成）。 |
| `steam-host-backend/` | Steam 桥宿主机侧静态库（纯 C ABI + 反向回调）。 |
| `tools/` | 离线工具与脚本：两个代码生成器、sandbox 启动器、根目录准备脚本。 |
| `version/` | 构建期版本信息生成（git 描述等 → C++ 常量）。 |
| `vulkan-bridge-marshal/` | GPU 桥的 Vulkan 编组运行时 + 生成代码。 |
| `windows-analyzer/` | Windows 模拟的命令行前端（分析引擎、上报器、快照、GDB）。 |
| `windows-emulator/` | Windows 内核模拟核心（系统调用、对象管理器、设备、模块、注册表、UI）。 |
| `windows-emulator-fuzz/` | windows-emulator 的 libFuzzer 目标（io_device 模糊测试）。 |
| `windows-emulator-test/` | windows-emulator 的大型 gtest 套件（50+ 测试文件）。 |
| `windows-gdb-stub/` | Windows 模拟器的 GDB 桩处理器（x86/x64/ARM64 寄存器映射）。 |

## 分层与依赖

```
入口层    analyzer ──► windows-analyzer / linux-analyzer / macos-analyzer
                      fuzzer ──► fuzzing-engine
                      tools/sandbox、android-runtime（JNI 宿主）
                    │
OS 模拟层  windows-emulator ──► emulator、process-manager、gdb-stub、debugger
           linux-emulator  ──► posix-emulator、emulator
           macos-emulator  ──► posix-emulator、emulator
                    │
桥接层    gpu-bridge-protocol + vulkan-bridge-marshal（GPU）
          steam-bridge-protocol + steam-host-backend + steam-generated（Steam）
                    │
接口层    emulator（核心接口）、emulator-platform（宿主平台抽象）、
          disassembler、common、version
                    │
后端层    backends/*（经 backend-selection 选择）
                    │
测试/绑定 python-bindings、windows-emulator-test、linux-emulator-test、
          macos-emulator-test、windows-emulator-fuzz、*-gdb-stub、samples
```

---

## 1. 接口层（后端无关的核心抽象）

### `emulator/` — 模拟器核心接口

整个项目的中枢抽象，定义"机器"与"CPU"的边界：

- `emulator.hpp` — 机器接口：`memory_interface + hook_interface`，`supports_multiple_vcpus()`、`serialize_state()`/快照的新进程可恢复性。
- `cpu_interface.hpp` / `typed_emulator.hpp` / `typed_cpu.hpp` — 每 vCPU 接口与类型化寄存器辅助（`reg<T>()`）。
- `x86_register.hpp` / `arm64_register.hpp` — x86/x64 与 ARM64 寄存器枚举。
- `memory_interface.hpp` / `memory_permission.hpp` / `memory_region.hpp` — 客体内存读写、权限、保留区域查询。
- `hook_interface.hpp` / `scoped_hook.hpp` — 执行/读写/指令/基本块钩子注册（回调携带触发的 CPU）与 RAII 注销。
- `serialization.hpp` / `serialization_helper.hpp` / `binary_writer.hpp` — `buffer_serializer`/`buffer_deserializer`，快照与 IPC 的统一序列化格式。
- `logger.hpp` / `generic_logger.hpp` — 日志接口（多 vCPU 下内部互斥）。
- 其他：`machine_type.hpp`（x64/arm64）、`stop_reason.hpp`、`address_utils.hpp`、`function_calling_convention.hpp`、`arch_emulator.hpp`、`interface.hpp`（`EMULATOR_INTERFACE` 导出宏）、`empty.cpp`。

### `emulator-platform/` — 平台抽象层

把宿主能力抽象成接口，使 windows/linux/macos 模拟器与具体 OS 解耦。`platform/` 下每个头一个能力域：

- 进程/线程/同步：`process.hpp`、`threading.hpp`、`synchronisation.hpp`。
- UI/外设：`ui_backend.hpp`、`audio_backend.hpp`、`console_backend.hpp`、`gdi.hpp`、`user.hpp`、`window.hpp`。
- 系统服务：`file_management.hpp`、`memory.hpp`、`registry.hpp`、`network.hpp`、`port.hpp`、`status.hpp`、`debug.hpp`。
- 二进制格式：`elf.hpp`、`win_pefile.hpp`。
- 杂项：`primitives.hpp`、`traits.hpp`、`unicode.hpp`、`compiler.hpp`、`namespace.hpp`、`kernel_mapped.hpp`。
- `segment_utils.hpp` — 段基址（fs/gs）计算。

### `disassembler/` — 反汇编器

Capstone 封装：`disassembler.hpp/.cpp` 提供 Intel/AT&T 语法的反汇编与 `resolve_jump_target`（跳转目标解析）。被调试器、analyzer 的 trace、UI 反汇编面板使用。

### `common/` — 共享工具

- `network/` — 宿主机网络封装：`address`（IP/端口解析）、`tcp_client_socket`、`tcp_server_socket`、`udp_socket`、`socket`。GDB stub 与 AFD 套接字设备共用。
- `utils/` — 约 30 个工具：序列化（`serialization.hpp`）、并发（`concurrency.hpp` 异步安全容器、`container.hpp`）、`host_pipe`/`host_watchpoints`（宿主机命名管道与硬件观察点）、`nt_handle.hpp`、`io.hpp`、`string.hpp`、`time.hpp`/`timer.hpp`、`cpu_features.hpp`、`compression.hpp`（zstd）、`stat.hpp`、`executable_path.hpp`、`finally.hpp`、`interupt_handler.hpp`、`lazy_object.hpp`、`win.hpp` 等。

### `version/` — 版本生成

`generate-version.cmake`：构建期把 git 描述/提交等信息生成 C++ 版本常量（快照兼容性头部用其校验构建一致性）。

---

## 2. 后端层

### `backend-selection/` — 后端选择

`backend_selection.hpp/.cpp`：`create_x86_64_emulator(backend_type, settings{ .vcpu_count })` 工厂与 `EMULATOR_UNICORN/ICICLE/WHP/KVM/FEX/HVF` 环境变量解析；不支持多 vCPU 的后端上请求 N>1 是构造期硬错误。

### `backends/` — CPU 执行后端

每个子目录一个独立后端，统一实现 `emulator`/`x86_64_cpu` 接口：

- `unicorn-emulator/` — 默认软件后端，完全模拟客体地址空间（沙箱性最好）；含 x86_64 与 ARM64 两个模拟器（`unicorn_x86_64_emulator`、`unicorn_arm64_emulator`）及细粒度钩子封装。
- `icicle-emulator/` — Rust 编写的 Icicle 软件后端（fuzzer 默认用它），含 Rust 桥（`icicle-bridge/`）。
- `whp-emulator/` — Windows Hyper-V 平台（硬件虚拟化），多 vCPU 主力；分区级 GPA/EPT 管理与 `partition_mutex_`。
- `whp-arm64-emulator/` — WHP 的 ARM64 客体变体。
- `kvm-emulator/` — Linux KVM 硬件虚拟化后端（合成 IDT、HLT 系统调用页等技术，见 `docs/kvm-backend.md`）。
- `fex-emulator/` — FEX-Emu 进程内 JIT（macOS Apple Silicon / Android AArch64），客体/宿主机地址空间共享，见 `docs/fex-backend.md`。
- `hvf-arm64-emulator/` — Apple Hypervisor 框架（ARM64）后端。

共性约束：硬件虚拟化/JIT 后端的细粒度内存钩子不触发（guest 原生运行，除单步外无插桩点）。

---

## 3. Windows 模拟栈

### `windows-emulator/` — Windows 内核模拟核心

项目最大的模块（扮演客体进程的"内核 + win32k"）。关键组成：

- **主控**：`windows_emulator.hpp/.cpp`（facade + 调度器/线程切换）、`process_context.hpp/.cpp`（进程状态、句柄表）、`emulator_thread.hpp/.cpp`（线程 + 寄存器数据块）、`syscall_dispatcher.hpp/.cpp` + `syscalls.cpp`（系统调用路由表）。
- **系统调用处理**（`syscalls/`，~25 个文件）：`process`、`thread`、`memory`、`file`、`section`、`event`、`mutant`、`semaphore`、`timer`、`port`（ALPC）、`registry`、`token`、`object`、`debug`、`exception`、`trace`、`user`（窗口/消息）、`gdi`、`composition`、`namespace`、`license`、`locale`、`system`、`worker_factory`、`io_completion`。
- **对象管理器**：`handles.hpp`（类型化对象存储 + 引用计数）、`windows_objects.hpp`（窗口/USER 对象）、`user_handle_table.hpp`（win32k 句柄表 + 逐消息位表）。
- **设备**（`devices/`，`io_device` 实现）：`afd_endpoint`（客体套接字）、`gpu_bridge` + `vulkan_host`（GPU 半虚拟化宿主侧，含 MSAA 快照采样着色器 `snapshot_shaders/` 与 `spirv_clip_distance`）、`steam_bridge`、`console`、`cryptography`（DPAPI）、`named_pipe`、`mount_point_manager`、`security_support_provider`、`network_store_interface`。
- **模块/加载**（`module/`）：PE 加载与映射（`module_manager`、`mapped_module`、`module_mapping`）；`apiset/`（API set schema 解析）、`sxs/`（SxS 清单/激活上下文）、`version/`（版本资源与版本欺骗）。
- **文件系统**（`filesystem/`）：挂载模型（`mount_spec`、`directory_mount`、`document_mount`）、COW 图层（`layered_provider`）、宿主机目录提供器、文档代理（`document_broker`/`document_tree_provider`）、`guest_file`、`i_file`。
- **UI**：`ui_backends/`（SDL / web / Android 后端）、`win32k_userconnect.hpp/.cpp`（SHAREDINFO/SERVERINFO 播种，含 MessageBox 按钮串）、`wow64_heaven_gate.hpp`、`window_destroy_orchestrator`/`window_show_orchestrator`、`debug_font.hpp`（临时 8x8 字体）、`gdi_font_signature.hpp`、`emulated_display_adapter.hpp`。
- **基础设施**：`exception_dispatch.hpp/.cpp`、`debug_port`/`debug_object`、`code_page.hpp`、`cpu_context`/`thread_context`、`scheduler_clock.hpp`、`frame_pacer.hpp`、`section_view_manager`、`inherited_pipes`、`port.hpp/.cpp`、`registry/`（hive 解析与覆盖层）、`network/`、`ports/`（CSRSS ApiPort 等）、`minidump_loader`（崩溃转储载入分析）、`emulator_process_target.hpp`（对接 process-manager）、`user_callback_dispatch.hpp`、`syscall_utils.hpp`、`windows_path.hpp`。
- **快照**：`snapshot.hpp/.cpp`（`serialize()`/`deserialize()`、事务性保存、`snapshot::controller`，见 `docs/process-snapshots.md`）。

### `windows-analyzer/` — Windows 模拟的命令行前端

`analyzer.exe`（Windows 路径）的主体：

- `main.cpp` — CLI11 参数解析（`--mount`/`-p`、`--network`、`--env`、`--vcpus`、`--suspend-to`/`--resume` 等），装配 windows_emulator + 后端 + GDB 桩 + 子进程管理器。
- `analysis.cpp/.hpp` + `analysis_event.hpp` — 分析事件循环（每 0x20000 条指令泵一次调试器事件）。
- 上报器：`analysis_reporter`、`console_reporter`、`jsonl_reporter`、`stdout_file_reporter`、`tenet_tracer`（Tenet 时间旅行 trace 格式）。
- 其他：`object_watching.hpp`（对象监视）、`disassembler.cpp`（分析用反汇编）、`reflect_extension.hpp`/`reflect_type_info.hpp`（反射类型信息）。
- `test.py` — 前端冒烟脚本。

### `windows-gdb-stub/` — Windows 的 GDB 桩处理器

实现 `gdb-stub` 的 `debugging_handler`：`win_gdb_stub_handler`（Windows 语义：断点、单步、线程）、`cpu_gdb_stub_handler`（CPU 通用部分）、x86/x64/ARM64 寄存器映射与 target description XML、osabi 模板、`windows_filesystem`（GDB 远程文件访问宿主机文件）。

### `windows-emulator-test/` — Windows 模拟 gtest

50+ 测试文件，覆盖：ALPC（完成/节区）、AFD connect、API set、ARM64（后端/调试器/模拟）、音频、代码完整性、代码页、控制台、节区创建/视图、加密后端、文档代理、 elevated token、Emscripten HTTP 套接字、事件日志、异常、FEX 寄存器/子页保护、文件（压缩信息/控制/重命名）、……以及 `emulation_test_utils.hpp` 公共夹具。

### `windows-emulator-fuzz/` — io_device 模糊测试

`device_fuzz.cpp`（以 libFuzzer 或独立 main 驱动，`libfuzzer_main.cpp`/`standalone_main.cpp`）+ `mock_emulator.hpp`（模拟器 mock）+ `fuzz_target.hpp`。对每个 `io_device` 的 IOCTL 处理程序做无状态 fuzz。

---

## 4. Linux / macOS / POSIX 模拟栈

### `linux-emulator/` — Linux 用户空间模拟器

- 主控：`linux_emulator.hpp/.cpp`、`linux_process_context.hpp/.cpp`、`linux_thread.hpp`。
- 系统调用：`linux_syscall_dispatcher.hpp/.cpp` + `syscalls/`（按类别组织）、`linux_syscall_numbers.hpp`、`linux_syscall_utils.hpp`。
- 加载：`module/`（ELF 加载器）、`vdso.hpp/.cpp`（可变 vDSO 元数据）。
- 进程语义：`signal_dispatch.hpp/.cpp`（信号投递）、`procfs.hpp/.cpp`（/proc 合成）、`linux_memory_layout.hpp`、`linux_stat.hpp`。
- `std_include.hpp`、工具头。

### `linux-analyzer/` — Linux 模拟的命令行前端

CLI11 入口（`--emulation-root`、可执行路径、端口映射等），装配 linux_emulator + linux_x64_gdb_stub；含 Emscripten 分支（`linux_event_handler`）。

### `linux-gdb-stub/` — Linux 的 GDB 桩处理器

`linux_x64_gdb_stub_handler`、`x64_gdb_stub_handler`、`x64_register_mapping`、`x64_target_descriptions`。

### `linux-debugger/` — Linux 调试器事件处理

`linux_event_handler.hpp/.cpp`：浏览器/Emscripten 路径下把 linux-emulator 的停止事件编码为 FlatBuffers 事件（对应 `windows-analyzer` 中的 `event_handler` 角色）。

### `linux-emulator-test/` — Linux 模拟 gtest

`emulation_test.cpp`、地址/映射测试（`network_address_test`、`path_port_mapping_test`、`thread_scheduler_test`）与一组 torture 压测：ELF 加载器、`poll/select/epoll`、信号/futex、fd/pipe、`mmap/brk`、`openat/getdents`。

### `posix-emulator/` — 共享 POSIX 层

Linux 与 macOS 模拟器共用的基础：`posix_fd_table`（fd 类型与生命周期）、`posix_file_system`、`posix_path_resolution`、`posix_memory_manager`、`posix_child_processes`。

### `macos-emulator/` — macOS 用户空间模拟器

- Mach 语义：`macos_mach_ports.hpp`、`mig/`（Mach IPC 存根生成物）、`macos_syscall_dispatcher`、`macos_syscall_context`。
- 加载：`macho.hpp`（Mach-O）、`dyld_cache_format.hpp` + `macos_shared_region.cpp/.hpp`（dyld 共享缓存/共享区域）、`macos_commpage.cpp/.hpp`（commpage 合成）、`macos_dyld_test` 的解析基础。
- 其他：`macos_file_ids.hpp`、`macos_memory_protection.hpp`、`macos_signals.hpp`、`syscalls/`、`macos_emulator.hpp/.cpp`。
- `std_include.hpp` 等。

### `macos-analyzer/` / `macos-emulator-test/` / `macos-dyld-test/`

- `macos-analyzer/`：macOS 模拟的 CLI 前端（`main.cpp`）。
- `macos-emulator-test/`：macOS 模拟 gtest。
- `macos-dyld-test/`：dyld 共享缓存格式解析的独立测试。

---

## 5. 桥接与半虚拟化

### `gpu-bridge-protocol/` — GPU 桥线上协议

- `gpu_bridge_protocol.hpp` — 零依赖线上协议：`CTL_CODE` IOCTL 编码、每命令固定大小请求/响应结构、`object_id`；宿主机与客体 shim 共享的唯一事实来源。
- `vk_feature_chain.hpp` — 跨位宽 `sType → sizeof` 映射，用于编组 `VkPhysicalDevice*Features` pNext 链（DXVK 设备引导）。

### `vulkan-bridge-marshal/` — Vulkan 编组

- `vk_bridge_serial.hpp` — 编组运行时：字节流 `writer`/`reader` + 持有解码对象的 `arena`。
- `vk_bridge_marshal.generated.hxx` — **生成代码**（由 `tools/vulkan-bridge-generator` 从 `vk.xml` 发射，勿手改）。

### `steam-bridge-protocol/` — Steam 桥线上协议

`steam_bridge_protocol.hpp`：IOCTL 代码与请求/响应结构（固定 8 字节标量槽，架构无关）。

### `steam-host-backend/` — Steam 桥宿主机侧

纯 C ABI 静态库（可被任何宿主加载）：

- `steam_host_backend.h/.cpp` — C ABI 入口（版本握手、接口创建、调用、回调泵）。
- `steam_host_runtime.hpp` — 宿主机运行时（加载真实 steamclient64、按 tag 分发）。
- `steam_host_tag.cpp.in` — 每个接口版本一个实例化的模板（编译期针对各 SDK 快照）。
- `steam_reverse.hpp/.cpp` — 反向回调（Steam 调用进入游戏实现的响应对象）。

### `steam-generated/` — 生成输出

**目录在树中为空且被 gitignore**：Steam 桥生成器（`tools/steam-bridge-generator`）的客体代理 + 宿主机 thunk 输出（Valve 衍生，绝不提交）。本地配置 CMake 后自动生成到此处或构建目录。

---

## 6. 调试设施

### `gdb-stub/` — GDB 远程协议（后端无关）

- `gdb_stub.hpp/.cpp` — 协议状态机与 `debugging_handler` 抽象（run/singlestep、寄存器读写、断点、内存/文件读写）。
- `connection_handler.hpp/.cpp` — TCP/串行连接。
- `async_handler.hpp/.cpp` — 异步停/断事件。
- `stream_processor.hpp/.cpp` + `checksum.hpp` — 数据包解析、`$…#cc` 校验。
- `filesystem_interface.hpp` — 远程文件系统操作抽象。

### `debugger/` — 浏览器调试器核心

- `debug_session.hpp/.cpp` — 一等调试子系统：断点注册表、步进引擎（into/over/out/run-to）、只读自省；后端无关。
- `event_handler.hpp/.cpp` — 解码 `DebugCommandRequest`（kind + JSON payload），分发到 `debug_session`，回发 `DebugCommandResponse`；同时把模拟事件编码进 FlatBuffers。
- `events.fbs` + `events_generated.hxx` — 事件协议 schema 与生成代码。
- `message_transmitter.hpp/.cpp` — 消息传输。
- 设计见 `docs/debugger/ARCHITECTURE.md`；被 `windows-analyzer`（`analysis.cpp` 泵驱动）与 `page/` 前端共用。

---

## 7. 进程管理与 Android

### `process-manager/` — 宿主进程管理

- `process_manager.hpp` — `managed_process` 抽象：跨进程读/写内存、查询内存区域、等待退出、枚举线程等（宿主进程的外部控制接口）。
- `subprocess_process_manager.hpp/.cpp` — 启动真实宿主机子进程、经管道用 FlatBuffers 协议（`process_protocol.fbs` / `process_protocol_generated.hxx`）控制（用于客体子进程 emulation 与 analyzer 的子进程模式）。

### `android-runtime/` — Android 原生运行时

Android 应用（`app/`）加载的原生侧：

- `session.hpp/.cpp` — 一次模拟会话：run_config（根目录/参数/环境/分辨率/网络）、SAF 文档树（`document_tree`，COW 挂载）、快照挂起/恢复。
- `android_ui_backend.hpp/.cpp` — Android 平台 `ui_backend` 实现（Surface 呈现、输入）。
- `java_dispatcher.hpp/.cpp` + `jni_bridge.cpp` — Java↔C++ 双向调用分发。
- `android_keymap.hpp/.cpp` — Android 键码 → 客体输入映射。
- `pseudo_terminal.hpp/.cpp` — 伪终端（控制台重定向）。

---

## 8. Python 绑定与样例

### `python-bindings/` — `sogen` Python 模块

nanobind 绑定（PyPI 包 `sogen` 的本体）：

- `sogen_module.cpp` — 模块初始化与枚举/类型导出。
- `sogen_wrappers.cpp` + `sogen_bindings_runtime.cpp` + `sogen_bindings.hpp` + `sogen_internal.hpp` — Windows 模拟器 facade（`sogen.windows.*`）。
- `sogen_linux_wrappers.cpp` + `sogen_bindings_linux_runtime.cpp` — Linux facade（`sogen.linux.*`）。
- `sogen_bindings_arm64.cpp` — ARM64 注册器/内存原语。
- `sogen_api_hooks.cpp` — `@api_call`/`@symbol_call` 装饰器与调用拦截。
- `sogen_callbacks.cpp` + `sogen_linux_callbacks.cpp` — 回调槽（stdout、模块加载、内存违例、线程生命周期…）。
- `sogen_bindings_types.cpp` + `sogen_helpers.cpp` — 值类型与辅助。
- `sogen.def` — Windows 导出定义。
- `test.py` / `test_linux.py` / `test_arm64.py` — 绑定的冒烟脚本（对应 PyPI 发布验证）。

### `samples/` — 客体侧二进制

构建为客体 PE/Mach-O 可执行文件或 DLL shim，由 analyzer 在模拟器内运行：

- `test-sample/` — 冒烟测试样本（26 项测试；`analyzer.exe -s test-sample.exe`）。
- `hook-sample/` — API 钩子演示目标。
- `bad-sample/` — 异常/崩溃路径样本。
- `arm64-hello-world/`、`macos-hello-sample/` — ARM64 / macOS 客体 smoke。
- `vulkan-shim/` — 客体 `vulkan-1.dll` shim（GPU 桥客户端，交付物）。
- `vulkan-shim-test/` — 无头回归测试（fill/clear 读回校验）。
- `vulkan-snapshot-test/` — GPU 快照往返校验样本。
- `steam-shim/` — 客体 `steamclient(64).dll` shim（Steam 桥客户端，32/64 位）。

---

## 9. 模糊测试

### `fuzzing-engine/` — 可复用引擎

`fuzzer.hpp/.cpp`（运行循环、覆盖率反馈接入）、`input_generator.hpp/.cpp`（变异/生成策略）、`random_generator.hpp/.cpp`（确定性随机源）。

### `fuzzer/` — 模糊测试可执行入口

`main.cpp`：以 Icicle 后端（要求 `SOGEN_ENABLE_RUST_CODE`）驱动 windows_emulator 的独立 fuzzer 可执行文件；`resources/` + `resource.rc`。

（`windows-emulator-fuzz/` 见第 3 节，是 io_device 级的 libFuzzer 目标。）

---

## 10. 工具

### `tools/` — 离线工具与脚本

- `steam-bridge-generator/` — 解析 Steamworks SDK 头（libclang），发射版本精确的客体代理 + 宿主机 thunk（→ `steam-generated/`）；设计见 `docs/steam-bridge-versioning.md`。
- `vulkan-bridge-generator/` — 解析 `vk.xml`，发射 Vulkan 编组代码（→ `vulkan-bridge-marshal/`）；见 `docs/gpu-paravirtualization.md`。
- `sandbox/` — `sandbox.exe`：受限的启动器可执行文件（Steam 桥宿主场景用，见 `docs/steam-bridge.md`）。
- 根目录准备脚本：`create-root.bat`、`grab-registry.bat`（注册表 hive 采集）、`create-macos-root.sh`、`create-profile-dirs.ps1`、`install-gpu-runtime.ps1`、`install-steam-shim.ps1`、`seed-audio-endpoint.ps1`。

---

## 依赖速查

| 若你修改… | 主要影响面 |
|---|---|
| `emulator/`（接口） | 全部后端 + 三个 OS 模拟器（改前读 `docs/multi-vcpu-design.md` §3 契约） |
| `backends/` | `backend-selection` + 对应平台的模拟器；KVM/FEX 先读对应开发笔记 |
| `windows-emulator/` | analyzer、python-bindings、调试器、page 前端、快照格式 |
| `windows-emulator/syscalls/` 或 `devices/` | `windows-emulator-test` 应有对应 gtest；设备另受 `windows-emulator-fuzz` 覆盖 |
| `linux-emulator/` | linux-analyzer、python-bindings（Linux facade）、`linux-emulator-test` |
| `gpu-bridge-protocol/` | 协议版本需 bump（`GET_VERSION` 握手）；同步改 `vulkan-shim` 样本 |
| `steam-bridge-protocol/` / 生成器 | 需本地重跑生成器（输出不提交） |
| `events.fbs` | 需重新生成 `events_generated.hxx` 并同步 `page/src/fb/` |
| `python-bindings/` | 同步更新 `docs/python/README.md` 与 `examples/python/` |
