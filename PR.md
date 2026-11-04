# Pull Request 提交指南

本文根据 [README.md](README.md)、[AGENTS.md](AGENTS.md) 和当前 [GitHub Actions 工作流](.github/workflows) 整理，用于提交 PR 前的准备、自检和 CI 排查。

文中的“仓库要求”来自现有规范，“CI 行为”来自工作流配置；分支命名、描述模板等属于提交建议。工作流会变化，发生不一致时请以对应源文件为准。仓库分支保护、必需检查项、审批人数和合并方式无法仅凭这些文件确定，不应把下文的检查列表视为 GitHub 合并权限配置。

## 1. 仓库明确要求

### PR 范围与描述

- **一个 PR 只处理一个主题。** 无关修复、重构和性能优化应分别提交，不要顺手混入。
- **标题与描述必须准确覆盖完整 diff。** 不要遗漏行为变化、附带改动或兼容性影响。
- 优先采用干净、符合项目习惯的方案，不使用临时绕过或补丁式 workaround。
- **禁止提交二进制文件**，包括可执行文件、DLL、目标文件、压缩包、图片和编译后的样本；Git LFS 指针或生成产物也不能作为绕过方式。复现材料优先提供源码、生成步骤或合法可访问的链接，不要把产物加入 Git。

### 代码与注释

- 项目使用 C++20；保持周围代码的命名、结构和风格，使用清晰的小函数、RAII、容器和合理抽象。
- 修改过的 C/C++ 文件需要运行 `clang-format`。
- 默认**不写注释**。只有代码本身无法表达的外部约束、刻意偏离常规方案的原因，以及能节省研究时间的规范引用，才适合写注释。
- 不要添加复述代码、函数内分节横幅、修改过程叙述或仅将函数签名改写成自然语言的注释。修改动机应写进 PR 描述，而不是留在源码里。

### 测试

- 不为覆盖率数字而添加测试；没有价值的测试不如不加。
- 优先使用 Google Test 单元测试。只有行为确实跨越模拟器子系统、无法在单元层有效验证时，才扩展在模拟器内运行的 `test-sample.exe`。
- 测试应验证客观行为，例如 Windows/NT 语义、guest 可见结果、公共 API、ABI、文件格式和架构不变量。
- 不要固定偶然实现细节或开发者自行选择的日志措辞、诊断内容与输出格式，除非它们确实是稳定接口或被其他组件消费。
- 测试必须确定且高效：不使用无意义的 sleep、固定延时或依赖时序的断言；需要等待时应等待实际条件。

完整说明见 [AGENTS.md](AGENTS.md)。

## 2. 开始开发前

1. 没有上游写权限时，先 fork 仓库并克隆自己的 fork；通常以 `momo5502/sogen` 的 `main` 为 PR 目标分支，特殊情况以维护者要求为准。
2. 克隆时包含子模块，已有工作区则同步子模块。
3. 从适当的基线新建工作分支，避免把其他主题的提交带进 PR。

README 中的上游克隆命令如下；使用 fork 时替换仓库地址：

```bash
git clone --recurse-submodules https://github.com/momo5502/sogen.git
cd sogen
git submodule update --init --recursive
git switch -c fix/short-topic
```

`fix/short-topic` 只是命名示例，现有文档没有规定分支名前缀或强制使用 Conventional Commits。不要无意更新子模块指针，也不要提交本机路径、凭据、注册表导出或 emulation root。

## 3. 本地自检

以下命令除特别说明外均从仓库根目录执行。根据实际改动选择额外检查，并在 PR 中记录运行环境、命令和结果；未执行或不适用的检查应明确说明。

### 3.1 构建环境与 release 构建

基础工具包括支持 C++20 的编译器、CMake **3.26.4 或更新版本**，以及 `release` / `tidy` preset 使用的 Ninja。平台依赖和特殊后端依赖请参考工作流及对应文档，不要假设只安装 CMake 就足够。

在 Windows 上，按 README 使用匹配架构的 Developer Command Prompt。README 的 Visual Studio 入门方式为 `cmake --preset=vs2022`，再构建生成的 `build/vs2022/sogen.sln`；日常快速迭代按 AGENTS 使用：

```bash
cmake --preset=release
cmake --build --preset=release
```

首次构建或配置变化后需要先 configure。`release` preset 实际设置的是 `CMAKE_BUILD_TYPE=RelWithDebInfo`，不要仅凭名称认定它是无调试信息的纯 Release 配置。配置详情见 [CMakePresets.json](CMakePresets.json)。

完整构建的功能也受依赖影响：CI 安装 Python 和 `libclang` 供 Steam bridge 代码生成使用；Rust 后端及 ARM64 上的 FEX 另有依赖。修改这些功能时，应核对构建日志，确认相关组件实际参与了构建，而不是被禁用或跳过。

### 3.2 格式检查

CI 的 C/C++ 格式检查使用 **clang-format 22** 和仓库的 [.clang-format](.clang-format)。本地也应使用相同主版本，避免工具版本不同造成来回修改。

对实际修改的文件执行以下操作，将示例路径替换为真实文件：

```bash
clang-format --version
clang-format -i src/path/to/changed_file.cpp
clang-format --dry-run --Werror src/path/to/changed_file.cpp
```

部分系统的可执行文件名为 `clang-format-22`。只格式化本次相关文件，避免把无关的全仓格式调整混入 PR。CI 会扫描 `src/` 下的 `.cpp`、`.hpp`、`.cc`、`.cxx`、`.h`、`.c` 文件，并遵循 [.clang-format-ignore](.clang-format-ignore)。

### 3.3 Smoke test 与 CTest

AGENTS 给出的 Windows smoke test 如下，在 **cmd / Developer Command Prompt** 中执行：

```bat
cmd /c "cd build\release\artifacts\ && analyzer.exe --network live -s test-sample.exe"
```

`test-sample.exe` 会解析 `google.com`，因此需要 `--network live`。这会让 guest 使用宿主网络，不是离线测试。

运行依赖真实系统 DLL 和注册表的程序时，需要先准备运行环境。README 提供了注册表导出和 emulation root 的说明；也可参考 [运行指南](https://github.com/momo5502/sogen/wiki/Run-The-Emulator#emulation-root-environment)。不要把本机导出的环境加入 Git。

使用独立 root 复现 CI 的样本运行方式时，先确保 `build/release/artifacts/root` 已准备好，并将样本放到 `root/filesys/c/test-sample.exe`：

```bat
cmd /c "cd build\release\artifacts\ && analyzer.exe --network live --root root --root-mode rw c:/test-sample.exe"
```

**文件系统默认只读。** 样本涉及写入时，CI 会显式使用 `--root-mode rw`。本地应使用可丢弃的专用 root；没有 root 时开启写入可能影响宿主盘。挂载与权限详情见 [docs/filesystem.md](docs/filesystem.md)。

完成构建并准备相应测试环境后，可运行：

```bash
ctest --test-dir build/release --output-on-failure
```

CI 的常规测试使用 `ctest --verbose -j`。根据测试类型，还会设置以下环境变量；本地应使用实际绝对路径，而不是照抄 CI 的工作区路径：

| 变量 | 用途 |
| --- | --- |
| `EMULATOR_ROOT` | Windows emulation root；CI 将样本复制到该 root 的 `filesys/c/` 下 |
| `ANALYSIS_SAMPLE` | 构建出的 `test-sample.exe` 路径 |
| `LINUX_TEST_ROOT` | Linux 测试文件目录，CI 使用 `deps/sogen-linux-files/test/linux` |
| `MACOS_EMULATOR_ROOT` | macOS 测试所需的本机 root |
| `EMULATOR_ARM64_ROOT` | ARM64 guest 测试所需的 Windows ARM64 root |
| `EMULATOR_VERBOSE` | 详细日志开关，用于排查测试问题 |

只运行一个测试可用于开发迭代，但不能据此声称完整测试已通过。不要将缺少 root、样本或后端导致的跳过当成对应功能验证成功。

### 3.4 最后运行 tidy

按 AGENTS 要求，功能实现完成后运行带 clang-tidy 的构建；该配置较慢，不用于每次快速迭代：

```bash
cmake --preset=tidy
cmake --build --preset=tidy
```

CI 在 Windows 和 Linux 上使用 LLVM **22**。Windows tidy job 使用 `clang-cl`；本地对齐时，应在正确的 Developer Command Prompt 中为 tidy 配置选择该编译器。若已有 CMake 缓存使用另一编译器，应重新准备该构建目录，不要直接混用。

`tidy` preset 启用 Python bindings，但关闭 Steam 和 Rust 代码，因此 **tidy 通过不等于所有后端和完整功能构建通过**。

纯文档改动可在 PR 中说明本地编译和运行测试不适用；这不会使工作流自动跳过构建。

### 3.5 按改动范围增加检查

#### Web 页面：`page/`

在 `page/` 下执行：

```bash
npm ci
npx prettier . --check
npm run lint
npm run build
```

需要修复格式时，只对相关文件运行 Prettier 的 `--write`。CI 的页面构建还会提前放入 Emscripten 32/64 位产物、样本和 root；仅完成页面编译不代表浏览器中的模拟器运行已验证。影响交互或运行流程时，在 PR 中补充手动验证步骤与结果。

#### Flutter / Android 应用：`app/`

在 `app/` 下执行：

```bash
flutter pub get
dart format --output=none --set-exit-if-changed lib test
flutter analyze
flutter test
```

CI 另外构建 release APK，并在 `app/android/` 下运行 `./gradlew :app:testDebugUnitTest` 验证原生代码。APK 构建依赖已暂存的 ARM64 原生库和其他资源，本地准备方式见 [app/README.md](app/README.md)，不要将不完整环境下的构建结果当成完整应用验证。

#### Python bindings / 打包

CI 使用 Python **3.10** 构建并测试 wheel / sdist。涉及绑定、导出 API 或打包规则时，应检查源码包和二进制包，而不仅是在开发目录中 import：

```bash
python -m pip install --upgrade build setuptools-scm
python -m build --wheel --outdir dist
python -m build --sdist --outdir dist
```

CI 安装构建好的 wheel 后运行 `src/python-bindings/test.py`、`test_arm64.py`，并在 Linux 上运行 `test_linux.py` 的 Unicorn / KVM 场景；这些运行测试仍需对应样本、root 和后端环境。sdist job 会从源码包安装并验证 import，专门检查无桌面开发依赖的 headless 构建场景。

使用说明见 [docs/python/README.md](docs/python/README.md) 和 [examples/python/README.md](examples/python/README.md)。生成的 `dist/` 包不应提交到 Git。

#### 后端、guest 架构和系统语义

修改 CPU 后端、syscall、内存、线程、序列化或文件系统时，说明影响的 host 平台、guest 架构和后端，并验证相关组合。宿主架构与 guest 架构不是同一个维度，单一 Windows x86_64 构建不能覆盖 ARM64 guest 或其他后端。

macOS 支持仍处于早期阶段。macOS root 必须按 README 在符合条件的 Mac 上生成并保留在该 Mac 上，不得分发，也不得将 CI 中临时生成的 root 上传为产物。

## 4. 提交 PR 后，CI 会做什么

### 4.1 触发规则

| 工作流 | 触发条件与用途 |
| --- | --- |
| [Build PR](.github/workflows/build.yml) | PR 的 `opened`、`synchronize`、`reopened` 事件，目标分支匹配 `**`；调用共享流水线 |
| [Build Branch](.github/workflows/build-push.yml) | 分支 push 或手动 `workflow_dispatch`；调用同一共享流水线，并在满足条件时部署与发布 |
| [Reusable Build](.github/workflows/ci-reusable.yml) | 由上述入口通过 `workflow_call` 调用，执行主要构建、测试、打包和汇总 |
| [Fuzz](.github/workflows/fuzz.yml) | 每天 Europe/Berlin 时区 02:00 定时运行，也可手动触发；不挂在 PR 事件上 |
| [Claude Code](.github/workflows/claude.yml) | 在指定 issue、评论或 review 事件的相应正文或标题中出现 `@claude` 时触发；不是每个 PR 都自动运行的构建检查 |

注意以下行为：

- **当前没有 `paths` / `paths-ignore` 过滤。** 纯文档 PR 也会触发共享流水线。
- PR 入口没有排除 Draft，但没有监听 `ready_for_review`；仅将 Draft 改为 Ready 不会通过这个事件额外触发构建。
- 非 `main` 分支的 push 流水线会检查该分支是否已有 open PR；存在时跳过自己的共享构建，由 PR 流水线承担验证。首次未发现 PR 时会等待约 60 秒再检查一次。
- 共享流水线按 `github.ref` 分组，并设置 `cancel-in-progress: true`。继续推送同一 PR / 分支时，旧运行可能被取消，应查看最新提交对应的结果。

### 4.2 当前启用的主要检查

以下概括当前未被注释掉的配置，不代表所有平台都运行同一套测试：

| 检查 | 当前覆盖范围 |
| --- | --- |
| `Run Clang Tidy` | Windows、Linux，LLVM 22 |
| `Verify Formatting` | `src/` 的 C/C++ clang-format 检查，以及 `page/` 的 Prettier 检查 |
| 主构建矩阵 | Windows x86 / x86_64 / arm64、MinGW x86_64、Linux x86_64 GCC Sanitizer、Linux x86_64 / arm64 Clang、macOS arm64、Android x86_64 / arm64、Emscripten Web / Web Memory 64 / Node.js |
| `Build Isolate` | Windows x86_64 的 Debug 和 Release；关闭 LTO、Steam 和 Rust 以尽早开始测试 |
| Root 准备 | 构建 x86 / x64 root tools，创建 Windows 2025、Windows 2022 和 Windows 11 arm64 roots |
| 常规 `Test` | Windows x86、Linux x86_64 GCC Sanitizer / Clang、macOS arm64，使用 Windows 2025 root 和 Unicorn 配置 |
| `Windows Test` | Windows x86_64，Unicorn；Windows 2025 root 覆盖 Debug / Release，Windows 2022 root 只覆盖 Release |
| ARM64 测试 | Linux x86_64 Sanitizer / Clang 和 macOS arm64 上的 ARM64 guest 测试，以及 Windows ARM64 上的原生样本运行 |
| 后端 / 平台 smoke tests | Node.js、WHP、Icicle、KVM、Windows x86、MinGW、macOS arm64 的 FEX、Linux arm64 的 FEX（x86_64 / x86 guest）、Android x86_64 的 Unicorn / Icicle |
| `Build Page` | 暂存 Web 产物与 roots，执行 `npm ci`、lint 和页面构建 |
| `Build Android App` | Dart 格式、Flutter analyze / test、release APK 构建及 Android 原生单元测试；有签名密钥时额外生成 AAB |
| Python wheel | Linux x86_64、Windows x86_64、macOS arm64 / x86_64 的构建、安装和运行测试 |
| Python sdist | 构建源码包，在 Linux 无桌面开发依赖环境下安装并验证 import |
| `Pipeline Summary` | 汇总依赖 job；发现 `failure` 或 `cancelled` 时失败，并尝试删除 `Temp *` 临时产物 |

CI 的主构建矩阵目前只启用名为 Release 的配置，Windows isolate 测试另有 Debug 覆盖。iOS、主构建中的 macOS x86_64、普通 Linux GCC 等被注释的矩阵项不应算作已启用覆盖；但 macOS x86_64 的 Python wheel 构建与测试仍然启用。

汇总脚本不会把单纯的 `skipped` 自动判为失败。因此除了看汇总状态，还应检查相关测试是否因前置失败、条件或环境缺失而未执行。

### 4.3 PR 验证与发布的区别

- PR 会构建页面、Android APK、Python wheel 和 sdist 等产物，**不等于发布到生产环境**。
- `Build Branch` 中的 GitHub Pages 部署和 PyPI 发布，只在 `github.repository_owner == 'momo5502'`、`github.ref == 'refs/heads/main'` 且共享 CI 成功时运行。普通分支和 fork 的 `main` 不满足这些发布条件。
- 发布动作不依赖打 tag 触发；合入满足条件的 `main` 后即可进入发布流程。影响页面、绑定或打包规则时，应在 PR 中说明发布影响。
- fork 及 fork PR 无法取得上游签名 secrets；Android 构建无 release key 时使用 debug key，只有成功配置 release key 才额外构建 AAB。不要为了让 PR 产物“看起来正式”而提交密钥或签名配置。
- Pages 部署和 PyPI 发布分别串行化，且不会取消已经开始的同类发布。它们是分支流水线中的后续 job，不是 PR 构建步骤。

## 5. CI 失败如何排查

1. 确认正在查看最新提交对应的运行，而不是被新提交取消的旧运行。
2. 找到最先失败的 job / step。构建、root 准备失败常会使下游测试或页面构建跳过，不要只看最后的汇总失败。
3. 记录 job 名、平台、编译器、guest 架构、root 版本、后端和关键错误；在 PR 中附 CI 日志链接及复现命令。
4. 格式问题按 CI 工具版本修复；编译或行为失败优先定位根因，不通过禁用检查、放宽断言或增加固定 sleep 来“修绿”。
5. 下载、runner、权限或外部服务故障应与代码问题区分。有证据表明是临时基础设施问题时再重跑，并保留说明；反复重跑直到偶然通过不能代替修复。

可用的诊断入口：

- 多数常规构建、root、wheel / sdist 和页面产物只保留 **1 天**；`Temp *` 还会被汇总 job 尝试删除，应及时查看。
- 序列化不匹配时，相关 job 会尝试上传 `Serialization Mismatch ...` 产物，其中含 `*.a.bin` / `*.b.bin`，保留 **7 天**。
- `Build Branch` 手动触发支持 `verbose=true`，用于向相关测试传递详细日志开关；PR 入口没有暴露这一手动输入。
- 独立 `Fuzz` 工作流针对的是模拟器 IO device handlers，使用 libFuzzer + ASAN，不是 `src/fuzzer` 的 guest 应用 fuzzer。定时运行时长为 3600 秒，手动默认 600 秒；发现 crash / OOM / timeout / leak 复现文件时上传 `fuzz-crash`，保留 **30 天**，并使 job 失败。涉及该类处理器改动时，可说明是否补充了 fuzz 验证；普通 PR 不会自动跑它。

对外贴日志或复现材料前，检查并移除令牌、密钥、本机敏感路径、注册表内容及不应公开的样本或系统文件。

## 6. PR 标题、描述与评审

以下为提交建议，而不是额外的自动检查规则：

- 标题描述实际结果或修复的问题，避免只写“修复 bug”“一些优化”。现有文件没有要求标题采用某种固定前缀。
- 描述先说明问题与原因，再概括解决方式、影响范围和验证结果；有相关 issue 时附链接，只有确实解决该 issue 时才使用关闭关键字。
- 对行为变化写清修改前后差异，尤其是 guest 可见语义、API / ABI、默认权限、兼容性和性能影响。
- UI 改动可附合法可分享的截图或录屏链接，但不要把图片或视频提交到仓库。
- 将已执行、失败、跳过和待验证的项目分开写。缺少平台、硬件或 root 时如实说明，不要声称已完成全部验证。
- 收到评审意见后保持 PR 聚焦；确需增加新的独立主题时拆分 PR。新增提交扩大 diff 后，同步更新标题、描述和验证记录。

当前仓库没有自动 PR 描述模板。下面的模板用于手动复制到 PR 正文；这个 `PR.md` 文件本身不会成为 GitHub 的自动模板。

### 可复制的 PR 描述模板

```markdown
## 背景

- 要解决的问题 / 相关 issue：
- 修改前的行为与复现步骤：

## 改动

- 主要修改及其原因：
- 行为、API / ABI、兼容性或发布影响：
- 受影响的 host 平台 / guest 架构 / 后端：

## 验证

| 检查 | 环境与命令 | 结果 |
| --- | --- | --- |
| release 构建 | | |
| clang-format / 相关格式检查 | | |
| tidy 构建 | | |
| 单元测试 / CTest | | |
| smoke test / 手动复现 | | |
| Web / Android / Python 专项检查（如适用） | | |

- 新增或修改的测试验证了什么客观行为：
- 未运行、失败或跳过的检查及原因：
- CI 运行 / 日志链接：
- 截图或其他验证材料链接（如适用）：

## 风险与限制

- 已知问题、平台限制及需要评审者重点关注的事项：
```

## 7. 提交前最后检查

- [ ] PR 只处理一个主题，没有混入无关修改或子模块指针变化。
- [ ] 标题与描述覆盖完整 diff，相关行为和兼容性影响已说明。
- [ ] 代码采用干净、符合项目习惯的实现，没有临时绕过或冗余注释。
- [ ] 修改过的文件已按对应工具检查格式，没有无关的全仓格式改动。
- [ ] 代码 / 功能改动已完成 release 构建，并在实现完成后运行 tidy；不适用或未执行项已说明。
- [ ] 测试有明确价值、确定且高效；相关测试和 smoke test 的环境、命令、结果已记录。
- [ ] 涉及 Web、Android、Python、后端或 guest 架构时，已补充对应验证或明确覆盖缺口。
- [ ] 没有提交二进制、图片、压缩包、root、注册表导出、凭据或本机临时文件。
- [ ] 已检查完整 diff 和暂存内容，而不仅是自己记得修改过的文件。
- [ ] 提交 PR 后，确认最新提交的 CI 结果，并排查与本次改动有关的失败或跳过。

暂存前可先执行以下只读检查：

```bash
git status --short
git diff --check
git diff --stat
git diff
```

暂存后再执行 `git diff --cached --check`、`git diff --cached --stat` 和 `git diff --cached`，确认真正将要提交的内容。`git diff` 不显示未跟踪文件的正文，新增文件也要单独检查。只暂存与本次 PR 相关的文件，再提交并推送到自己的工作分支。
