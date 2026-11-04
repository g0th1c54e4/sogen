# 目录用途说明

Sogen 是一个 C++20 Windows 用户空间模拟器（另含 Linux/macOS 用户空间模拟、浏览器 playground 和 Android 应用）。本文件说明仓库中每个目录的作用，**包含被 `.gitignore` 忽略的目录**。按字母序排列；`deps/` 与 `src/` 的子目录单独分节。

## 根目录

| 目录 | git 状态 | 作用 |
|---|---|---|
| `.github/` | 跟踪 | GitHub 配置：Dependabot 配置和 CI 工作流（`workflows/`）。 |
| `.git/`、`build/`、`page/node_modules/`、`page/public/*.wasm` 等 | **忽略** | 见下文"被忽略目录"。 |
| `app/` | 跟踪 | Android / Flutter 前端应用（游戏库、安装器、GOG 下载、触屏布局等）。 |
| `cmake/` | 跟踪 | 共享的 CMake 模块：编译器环境、FEX 与 Steam 桥的构建逻辑、工具链文件、杂项工具。 |
| `deps/` | 跟踪（子模块） | 第三方依赖，git 子模块。见下文专节。 |
| `docs/` | 跟踪 | 项目文档（架构、设计笔记、各子系统现状），即本目录。 |
| `examples/` | 跟踪 | 面向用户的示例，目前只有 `examples/python/`（Python API 用例）。 |
| `page/` | 跟踪 | 浏览器 playground：React + TypeScript + Vite 的 Web 前端，内含模拟器 worker、调试器 UI、文件系统浏览器、PWA 配置。 |
| `src/` | 跟踪 | 全部 C++ 源代码。见下文专节。 |
| `build/` | **忽略** | CMake 构建输出（所有预设：`release`、`tidy`、`vs2022`、Linux Docker 的 `build/linux` 等）。由 `cmake --build --preset=<name>` 生成，绝不提交。 |
| `dist_local/`、`devdocs/` | **忽略** | 本地开发产物：临时发布/打包输出、生成的开发文档。 |
| `page/node_modules/`、`page/public/{analyzer,debugger}{,.wasm}`、`page/public/{root,linux-root}.zip`、`page/public/{32,64}/…` | **忽略** | 前端依赖（npm install 生成）；编译出的 WebAssembly 二进制与模拟根 zip（构建生成，体积大，绝不提交）。 |
| `__pycache__/`、`.cache/`、`.vs/`、`.vscode/`、`compile_commands.json` | **忽略** | Python 缓存、Clang 缓存/编译数据库、IDE 本地状态。 |

## `src/` — 主源码（按字母序）

| 目录 | 作用 |
|---|---|
| `src/analyzer/` | 命令行 analyzer 可执行文件（`analyzer.exe`）的入口与资源；命令行选项见 `main.cpp`。 |
| `src/android-runtime/` | Android 运行时集成（被 Android 应用加载的原生侧）。 |
| `src/backend-selection/` | 运行时 CPU 后端选择逻辑（`EMULATOR_*` 环境变量、`backend_type`）。 |
| `src/backends/` | 各 CPU 模拟后端：`fex-emulator/`（FEX，macOS/Android ARM64）、`hvf-arm64-emulator/`、`icicle-emulator/`、`kvm-emulator/`（Linux）、`unicorn-emulator/`、`whp-arm64-emulator/`、`whp-emulator/`（Windows Hyper-V）。 |
| `src/common/` | 跨平台共享工具：网络（`network/`）、通用工具（`utils/`）。 |
| `src/debugger/` | 浏览器集成调试器的核心 `debug_session` 与通用命令协议。 |
| `src/disassembler/` | 基于 Capstone 的反汇编器封装。 |
| `src/emulator/` | 后端无关的模拟器核心接口（`emulator`、`x86_64_cpu`、钩子接口、类型化寄存器辅助）。 |
| `src/emulator-platform/` | 平台抽象层（SDL/控制台等平台支撑）。 |
| `src/fuzzer/` | 独立的模糊测试可执行入口与资源。 |
| `src/fuzzing-engine/` | 可复用的模糊测试引擎。 |
| `src/gdb-stub/` | x86/x64 的 GDB 远程调试桩。 |
| `src/gpu-bridge-protocol/` | GPU 半虚拟化的线上协议头（IOCTL 代码、请求/响应结构、特性链映射）。 |
| `src/linux-analyzer/` | Linux 客体 analyzer。 |
| `src/linux-debugger/` | Linux 模拟的调试器集成。 |
| `src/linux-emulator/` | Linux 用户空间模拟器本体（模块、系统调用等，`module/`、`syscalls/`）。 |
| `src/linux-emulator-test/` | Linux 模拟器的单元测试（含 Vulkan 编组往返测试）。 |
| `src/linux-gdb-stub/` | Linux 客体 GDB 桩。 |
| `src/macos-analyzer/` | macOS 客体 analyzer。 |
| `src/macos-dyld-test/` | macOS 动态链接器（dyld）测试。 |
| `src/macos-emulator/` | macOS 用户空间模拟器（MIG、`syscalls/`）。 |
| `src/macos-emulator-test/` | macOS 模拟器测试。 |
| `src/posix-emulator/` | POSIX 通用模拟层。 |
| `src/process-manager/` | 客体进程管理（子进程、句柄继承等）。 |
| `src/python-bindings/` | nanobind 编写的 Python 绑定（`sogen` 模块），Windows/Linux 模拟的全部 Python API。 |
| `src/samples/` | 客体侧样例与 shim：`arm64-hello-world`、`bad-sample`、`hook-sample`、`macos-hello-sample`、`steam-shim`（客体 steamclient shim）、`test-sample`（冒烟测试）、`vulkan-shim`（客体 vulkan-1.dll）、`vulkan-shim-test` 等。 |
| `src/steam-bridge-protocol/` | Steam 桥的线上协议（IOCTL 代码、请求/响应结构）。 |
| `src/steam-generated/` | Steam 桥生成器输出（**被 gitignore，Valve 衍生，绝不提交**；本地用生成器重建）。 |
| `src/steam-host-backend/` | Steam 桥宿主机侧：纯 C ABI 的静态库，持有 SDK 头文件 + thunk + 宿主机 steamclient64 引导。 |
| `src/tools/` | 离线代码生成/辅助工具：`sandbox`、`steam-bridge-generator`（解析 Steamworks SDK 头并生成桥代码）、`vulkan-bridge-generator`（解析 vk.xml 生成 Vulkan 编组）。 |
| `src/version/` | 版本号与构建信息。 |
| `src/vulkan-bridge-marshal/` | GPU 桥编组运行时（`writer`/`reader`/`arena`）与生成的编组代码。 |
| `src/windows-analyzer/` | Windows 客体 analyzer。 |
| `src/windows-emulator/` | Windows 用户空间模拟器本体：`apiset/`、`arch/`、`audio_backends/`、`console_backends/`、`crypt_backends/`、`devices/`（AFD、GPU 桥、Steam 桥等 io_device）、`filesystem/`、`module/`、`network/`、`ports/`、`registry/`、`sxs/`、`syscalls/`、`ui_backends/`、`version/`。 |
| `src/windows-emulator-fuzz/` | Windows 模拟器的模糊测试目标。 |
| `src/windows-emulator-test/` | Windows 模拟器单元测试（gtest）。 |
| `src/windows-gdb-stub/` | Windows 客体 GDB 桩。 |

## `deps/` — 第三方依赖（git 子模块）

| 目录 | 作用 |
|---|---|
| `deps/base64/` | Base64 编解码。 |
| `deps/capstone/` | Capstone 反汇编框架（`src/disassembler` 使用）。 |
| `deps/CLI11/` | CLI11 命令行解析库。 |
| `deps/FEX/` | FEX-Emu（x86→ARM64 JIT，FEX 后端；固定到个人 fork）。 |
| `deps/flatbuffers/` | FlatBuffers（事件/调试器协议序列化）。 |
| `deps/googletest/` | Google Test / Google Mock（单元测试）。 |
| `deps/gtest-parallel/` | gtest 并行运行器。 |
| `deps/libadrenotools/` | Adreno（Android GPU）工具库。 |
| `deps/mbedtls/` | mbed TLS（加密/TLS）。 |
| `deps/minidump_cpp/` | C++ minidump（崩溃转储）读写。 |
| `deps/nanobind/` | nanobind（Python 绑定，替换 pybind11）。 |
| `deps/reflect/` | 反射库。 |
| `deps/reproc/` | reproc 进程库。 |
| `deps/SDL/` | SDL3（原生 UI 后端、输入、窗口/纹理呈现）。 |
| `deps/sogen-linux-files/` | Sogen 自维护的 Linux 客体根文件系统（`root/`）与测试文件（`test/`）。 |
| `deps/unicorn/` | Unicorn CPU 模拟器（默认软件后端）。 |
| `deps/Vulkan-Headers/` | Vulkan 头文件（浅子模块；GPU 桥 INTERFACE 目标）。 |
| `deps/zstd/` | Zstandard 压缩。 |

## `app/` — Android 前端

| 目录 | 作用 |
|---|---|
| `app/android/` | Android 宿主工程：Gradle 配置（`build.gradle.kts`、`settings.gradle.kts`、`gradle/`、属性）。 |
| `app/assets/` | 应用资源（如图标 `sogen_mark.svg`）。 |
| `app/lib/` | Flutter/Dart 代码（`main.dart`、`src/`）。 |
| `app/test/` | Dart 测试（模拟根、游戏库、GOG 下载/安装器、触屏布局）。 |

## `page/` — 浏览器 playground

| 目录 | 作用 |
|---|---|
| `page/src/` | React 前端源码：`debugger/`（调试器 UI）、`fb/`（FlatBuffers 生成代码）、`components/`、`hooks/`、`lib/`；以及 `emulator.ts`（worker 桥）、`web-ui-host.ts`、`playground.tsx`、`filesystem*.tsx`、`download.ts`、`Installer.ts` 等。 |
| `page/public/` | 静态资产；其中 `*.wasm`、`*.js`（emscripten 输出）与 `root.zip`、`linux-root.zip` 为**构建生成并被 gitignore**。 |
| `page/node_modules/` | **忽略**；npm 依赖（`npm install` 生成）。 |

## `examples/` — 用户示例

| 目录 | 作用 |
|---|---|
| `examples/python/` | Python API 示例（Windows、Linux、ARM64 用例；含自己的 README）。 |
