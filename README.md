<h1 align="center">
	<a href="https://github.com/momo5502/sogen"><img src="https://momo5502.com/sogen/sogen.svg" height="200" /></a>
	<br>
	<a href="https://discord.sogen.dev"><img src="https://img.shields.io/badge/discord-come%20chat-00B0F8?logo=discord&logoColor=white"/></a>
	<a href="https://github.com/momo5502/sogen/actions"><img src="https://img.shields.io/github/actions/workflow/status/momo5502/sogen/build.yml?branch=main&label=build"/></a>
	<a href="https://github.com/momo5502/sogen/issues"><img src="https://img.shields.io/github/issues/momo5502/sogen?color=F8B000"/></a>
	<img src="https://img.shields.io/github/commit-activity/m/momo5502/sogen?color=FF3131"/>
</h1>

Sogen 无需真实操作系统即可运行 Windows、Linux 和 [macOS](#macos-程序初步支持) 程序：它让你看到并控制它们所做的一切。

Sogen 不重写数千个 OS API，而是在 CPU 和系统调用层级模拟二进制程序并运行**真实的系统 DLL**，因此行为与真实 OS 高度一致。

每条指令、每次内存访问和每个 API 调用都可以被挂钩、检查或改写；运行完全确定，且整个模拟器状态可快照并恢复。

基于 C++ 构建，由你选择的 CPU 后端驱动：

- [Unicorn Engine](https://github.com/unicorn-engine/unicorn)
- [icicle-emu](https://github.com/icicle-emu/icicle-emu)
- [Hyper-V (WHP)](https://learn.microsoft.com/en-us/virtualization/api/hypervisor-platform/hypervisor-platform)
- [KVM](https://www.kernel.org/doc/html/latest/virt/kvm/api.html)
- [FEX](https://fex-emu.com)

在线试用：<a href="https://sogen.dev">sogen.dev</a>  
加入社区：<a href="https://discord.sogen.dev">Discord</a>  
&nbsp;  

## 核心特性

- **真实系统 DLL**：运行真实的 ntdll、kernel32 和 user32，而非重写的桩
- **挂钩与改写**：拦截并修改内存、指令、系统调用和 API 调用
- **忠实的 Windows 内部机制**：PE 加载（重定位、TLS）、Windows 内存类型、SEH、线程、注册表、文件系统和网络
- **快照与恢复**：完整状态序列化、快速内存内快照和 minidump 载入
- **处处可运行**：Windows、Linux、macOS、Android、iOS 和浏览器，支持 x86-64 和 arm64
- **确定性**：每次运行都可复现，精确到指令  
&nbsp;  

## 预览

<img src="https://momo5502.com/sogen/preview.svg" width="650" alt="Preview" />

## 不可检测的调试

用你已经熟悉的工具（如 IDA Pro 或 GDB）通过 GDB 协议调试，或使用内置的浏览器内调试器。  
调试器运行在模拟器层、进程之外，因此对反调试检查不可见。

<img src="https://momo5502.com/sogen/debugger.png" width="650" alt="Debugging a process running in Sogen from an IDA Pro remote GDB session" />
&nbsp;  

## 在沙箱中运行游戏

原生 GUI 应用可运行，窗口、对话框和控件都可用。  
GPU 半虚拟化让你的真实 GPU 提供 3D 加速，Hyper-V 后端在你的 CPU 上原生运行代码，快到能玩游戏。  
Direct3D 8/9/10/11 游戏经由 [DXVK](https://github.com/doitsujin/dxvk) 运行，它在 GPU 桥之上把 Direct3D 翻译为 Vulkan。

<img src="https://momo5502.com/sogen/game.png" width="650" alt="A game running inside the Sogen emulator" />
&nbsp;  

## 项目概览

<a href="https://www.youtube.com/watch?v=wY9Q0DhodOQ" target="_blank">
  <img src="https://momo5502.com/sogen/video.png" alt="YouTube Video" width="600" />
</a>

点击<a href="https://docs.google.com/presentation/d/1pha4tFfDMpVzJ_ehJJ21SA_HAWkufQBVYQvh1IFhVls/edit">此处</a>查看幻灯片。  
&nbsp;  

## Python 绑定

安装：

```bash
pip install sogen
```

Python 绑定需要一个模拟根目录。可以[在此](https://sogen.dev/root.zip)下载现成的 root，或按照 [wiki](https://github.com/momo5502/sogen/wiki/Run-The-Emulator#emulation-root-environment) 中的说明创建自己的 root。

示例：

```python
import sogen

emu = sogen.windows.create_application("c:/test-sample.exe", emulation_root="./root")


def on_module_load(module):
    if module.name.lower() == "test-sample.exe":
        emu.hooks.memory_execution_at(module.entry_point, lambda address: print(f"hit entry point: 0x{address:x}"))

emu.callbacks.on_module_load = on_module_load
emu.start()
print(emu.process.exit_status)
```

设置细节和更大的示例见 `examples/python/README.md`。  
&nbsp;  

## 非官方绑定

[Dart 绑定](https://github.com/Wdestroier/sogen_dart)在单独的仓库中提供。  
&nbsp;  

## 快速开始（Windows + Visual Studio）

> [!TIP]  
> 查看 [Wiki](https://github.com/momo5502/sogen/wiki) 了解如何在 Windows、Linux、macOS 等平台上构建和运行模拟器的更多细节

1\. 检出代码：

```bash
git clone --recurse-submodules https://github.com/momo5502/sogen.git
```

2\. 在克隆目录中的 x64 Developer Command Prompt 里运行以下命令：

```bash
cmake --preset=vs2022
```

3\. 构建生成在 `build/vs2022/sogen.sln` 的解决方案

4\. 以管理员身份运行 [grab-registry.bat](https://github.com/momo5502/sogen/blob/main/src/tools/grab-registry.bat) 创建注册表转储，并把它放到 `analyzer.exe` 旁边的 artifacts 目录中

5\. 运行你选择的程序：

```bash
analyzer.exe C:\example.exe
```

默认情况下，客体无法访问宿主机网络。客体套接字使用进程内回环，DNS 查询解析为空地址，且 Steam 桥被禁用。要让客体访问宿主机网络，加 `--network live`：

```bash
analyzer.exe --network live C:\example.exe
```

客体文件系统默认只读，客体写入失败并返回 `STATUS_ACCESS_DENIED`。用 `--root-mode rw` 允许客体修改模拟根目录（未设置根目录时为宿主机驱动器）。用 `--mount HOST:GUEST[:MODE]` 在客体路径挂载宿主机文件或目录，例如 `--mount .\out:C:\out:rw`。挂载与模式的细节见 [docs/filesystem.md](docs/filesystem.md)。

Windows ARM64 可执行文件以同样方式运行，使用在 Windows ARM64 机器上用 [create-root.bat](https://github.com/momo5502/sogen/blob/main/src/tools/create-root.bat) 创建的模拟根目录。它们在 Windows ARM64 上走 WHP，在 Apple Silicon 上走 Hypervisor.framework，在其他地方走 Unicorn。

`--backend` 覆盖基于环境变量的后端选择。在 Windows ARM64 之外，如果未构建 Unicorn 的 ARM64 后端，`EMULATOR_UNICORN=1` 会让 ARM64 客体运行在 Unicorn 上而不是 hypervisor 上。

## macOS 程序（初步支持）

macOS 支持处于早期阶段。Sogen 通过真实的 dyld 和系统的 dyld 共享缓存运行 x86-64 macOS 命令行程序，足以运行 `echo` 这类简单工具。大多数其他程序仍会停在某个未实现的系统调用上。

1\. 在系统仍附带 x86-64 slice 的 Mac 上构建模拟根目录。该脚本会拷贝 dyld、`/bin/echo` 和 x86-64 共享缓存：

```bash
src/tools/create-macos-root.sh /tmp/macos-root
```

macOS 许可不允许再分发 root，也不允许在非 Apple 硬件上运行它，因此它无法下载，必须留在构建它的那台 Mac 上。

2\. 设置 `EMULATOR_MACOS=1`，从 root 运行程序：

```bash
EMULATOR_MACOS=1 build/release/artifacts/analyzer --root /tmp/macos-root /tmp/macos-root/bin/echo hello
```
