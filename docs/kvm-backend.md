# KVM 后端 — 开发笔记

Linux KVM 模拟器后端（`sogen::kvm`）。这是一个独立的后端，结构与 `whp-emulator` 镜像对应（WHP 后端保持不动），在 Linux x86_64 上通过环境变量 `EMULATOR_KVM=1` 在运行时选择。

## 状态

`test-sample.exe` 在 KVM 下可以完整运行，全部 26 个测试通过（退出码 0），Intel 和 AMD 宿主机上都已验证。单线程和多线程执行、分页、系统调用（LSTAR→HLT）、异常投递和 MMIO 都可以工作。

## 架构

- `kvm_x86_64_emulator.hpp` — 工厂函数 `sogen::kvm::create_x86_64_emulator()`。
- `kvm_x86_64_common.hpp` — 内置的辅助代码（页表遍历、MMIO 区域查找、通用寄存器分类、`access_memory`）；只保留 KVM 用到的部分。
- `kvm_x86_64_emulator.cpp` — 后端本体。

客体模型：64 位长模式，Windows 用户态进程运行在 CPL3。后端自己构建客体的页表（`pml4`、`ensure_virtual_mapping`），通过 `KVM_SET_USER_MEMORY_REGION` memslot 把客体物理页映射进 KVM（`rebuild_mappings`，它会合并连续的同权限页），并拦截：

- **系统调用**：`MSR_LSTAR` 指向一个内部页，该页首字节为 `HLT`；运行循环识别 `syscall_hook_page_+1` 处的 HLT 并调用 `handle_syscall_halt`。
- **异常**：通过一个合成的 IDT（见下面的修复 2）。
- **`cpuid` / `rdtsc` / `rdtscp` / `ud2`**：通过 `handle_pre_run_instruction`，在每次 `KVM_RUN` 之前扫描 RIP 处的字节。

CMake 把该后端限定为 `Linux AND NOT ANDROID AND x86_64`；`backend-selection` 会加入 `backend_type::kvm` 和 `EMULATOR_KVM=1` 环境变量。

细粒度的 `hook_memory_read/write/execution/range_execution` 和 `hook_basic_block` 钩子仅为 API 兼容而注册，但永远不会触发：客体代码以原生方式运行，除了单步执行之外没有按访问/按指令插桩的切入点。依赖这些钩子的分析器功能在该后端下不受支持。

## 使其可用的修复

1. **抢占 / 挂起。** 调度器每 20 ms 通过辅助线程调用 `emu().stop()` 来抢占正在运行的线程。原来的 `stop()` 只设置 `run_->immediate_exit = 1`，这个标志从未被重置（所以第一次抢占之后每次 `KVM_RUN` 都立即返回），而且只在 `KVM_RUN` 入口处被检查（所以它无法中断已进入客体模式的 vCPU）。修复：在每个时间片重置 `immediate_exit`，并用一个无操作的 `SIGRTMIN` 信号对 vCPU 线程执行 `pthread_kill`（安装时未加 `SA_RESTART`），使进行中的 `KVM_RUN` 以 `EINTR` 返回。

2. **异常拦截。** KVM 不像 WHP 那样提供用户态的异常退出位图。安装一个内部 IDT，其 64 位中断门指向运行在 CPL0、使用 IST 栈的单 `HLT` 桩（IST 栈通过 `KVM_SET_SREGS` 设置的 TSS 提供）。在 HLT 退出时，运行循环从触发陷阱的 RIP 推导出向量，从 IST 栈重建出错现场帧，恢复出错现场，并路由到 `handle_exception`。

3. **MMIO / SSE。** KVM 通过模拟出错指令来服务 MMIO，而其内核内模拟器缺少 SSE/AVX 支持（ntdll 会对 `KUSER_SHARED_DATA` 执行向量化的 `wcslen`）。修复：MMIO 页由**只读 memslot** 支撑，内容由读取回调填充并在每次进入客体前刷新（`refresh_mmio_pages`），这样读取以原生方式执行，只有写入会陷入。

4. **系统调用返回的 RIP 前进。** `handle_syscall_halt` 只有在处理程序保持 RIP 不变时才会把 RIP 前进过 syscall 指令，但 `dispatch_callback` 的插桩回调重定向会把 RIP 设置为 `callback - 2`，期望再加回 syscall 长度。在非跳过分支中始终前进 2（与 WHP 一致），否则被重定向的线程会提前两字节恢复执行。

5. **AMD：挂起异常的重新注入。** 合成 IDT 投递异常并回退到出错现场之后，KVM 仍在 `kvm_vcpu_events` 中保留一个挂起的异常。在 AMD（SVM）上，它会在下一次进入时重新注入 → 三重故障（`KVM_EXIT_SHUTDOWN`）；Intel 对此可以容忍。处理完异常后清除 `kvm_vcpu_events`。这就是 WHP 后端 AMD 修复（PR #808）在 KVM 上的对应版本。

6. **断点 RIP 差一。** int3 是陷阱，所以 CPU 压入的是 `int3 + 1`。模拟器的异常分发期望 RIP 位于 int3 处（用于 `ExceptionAddress` 和 int-2Dh 调试服务模式），这与按指令精确的后端所报告的一致。在陷阱处理程序中对 `#BP` 把 RIP 回退一个字节。

修复 5 和 6 只在 Windows 2022 模拟根目录下复现（其 ntdll 与某些本地根目录不同），修复 5 只在 AMD 上复现。

## 构建与测试（Docker，从 Windows 宿主机）

KVM 无法在 Windows 上构建或运行。使用 Docker Desktop（WSL2 后端）；嵌套虚拟化默认开启，因此加上 `--device /dev/kvm` 就可以使用 `/dev/kvm`。

构建镜像：

```dockerfile
FROM gcc:14
RUN apt-get update -qq && apt-get install -y -qq cmake ninja-build linux-libc-dev >/dev/null
```

配置（单独的构建目录 `build/linux`），禁用 Rust/Python 并以控制台模式使用内置 SDL：

```sh
docker run --rm -v "<repo>:/src" -w /src sogen-kvm-build \
  cmake -S . -B build/linux -G Ninja -DCMAKE_BUILD_TYPE=Release \
    -DSOGEN_ENABLE_RUST_CODE=OFF -DSOGEN_ENABLE_PYTHON_BINDINGS=OFF \
    -DSOGEN_ENABLE_SDL3=ON -DSOGEN_USE_SYSTEM_SDL3=OFF \
    -DSDL_X11=OFF -DSDL_WAYLAND=OFF -DSDL_UNIX_CONSOLE_BUILD=ON \
    -DSOGEN_ENABLE_LTO=OFF -DSOGEN_BUILD_TOOLS=ON
```

构建并运行（`kvm-emulator` 是 analyzer 加载的共享库，因此可以单独重建以便快速迭代）：

```sh
docker run --rm -v "<repo>:/src" -w /src sogen-kvm-build \
  cmake --build build/linux --target analyzer -j

docker run --rm --device /dev/kvm -v "<repo>:/src" -w /src/build/linux/artifacts \
  sogen-kvm-build \
  env EMULATOR_KVM=1 ./analyzer -s -e /src/build/linux/artifacts/root c:/test-sample.exe
```

注意事项：

- Windows 模拟根目录位于 `build/linux/artifacts/root`（在 Windows 上生成；CI 会生成它，Linux 任务会下载它）。`-s` 是静默模式；去掉它以查看详细的系统调用日志。
- 基础镜像中没有 `clang-format`；改为在宿主机上运行。
- unicorn 后端（省略 `EMULATOR_KVM`）可以让同一样本完整运行，是用作 trace 对比的参照。
- AMD 相关的问题可能需要在 AMD 宿主机上复现（GitHub 的 AMD runner / 一台实体 AMD 机器）；Docker 构建的 Linux 二进制也可以直接在裸机 Linux 上运行。

### Git Bash 上 Docker 的 Windows 路径陷阱

绑定挂载路径必须是 Windows 风格；要加上 `MSYS_NO_PATHCONV=1` 前缀，防止 Git Bash 改写路径，例如 `MSYS_NO_PATHCONV=1 docker run ... -v "C:\path\to\repo:/src" ...`。
