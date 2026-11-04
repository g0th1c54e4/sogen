# Steam 桥 — 旧版接口版本管理（设计笔记）

状态：**已验证的原型，尚未集成。** 记录了用基于 libclang、版本精确（version-exact）的生成器替换当前旧版版本处理方案的计划，风格类似 Proton/lsteamclient。

## 问题所在

游戏请求它们编译时所针对的接口版本（MW2 → `SteamClient008`、`SteamUser012`、`SteamNetworking003`、…）。当前的桥只为最新 SDK 生成代理，并把旧版本转发到最新的线上格式（对追加的参数做前缀转发，用 `_DEPRECATED` 别名），**对没有兼容最新孪生方法的任何方法打桩**。这会丢失方法：在 `steamworks_sdk_105` 上，60/235 个方法被桩掉 —— 包括 `InitiateGameConnection`（IWNet 认证）和整个 `ISteamNetworking` 套接字 API。

## Proton/lsteamclient 的做法

**每个接口版本一个完全生成的包装器**（79 个 SDK 快照共 189 个 `.cpp`）。每个包装器都针对该版本拥有精确 vtable 的那个快照编译，并且是一个简单的直通：
`((ISteamNetworking*)p)->IsP2PPacketAvailable(pcubMsgSize, nChannel)`。它从不打桩，也从不转发。因为 lsteamclient 桥接的是 **Windows↔Linux**，它需要 61 个 `struct_converters_*.cpp` 来处理 ABI 差异 —— 而且因为它是进程内直通（Wine 共享地址空间），它完全**不需要**参数方向信息（它把客体指针直接转发过去）。

## 我们的方案：版本精确、libclang、无结构体转换器

采用其*技术*（而不是 Proton 的代码）：用 **libclang** 解析每个 SDK 快照的头文件，为每个版本同时生成版本精确的客体代理**和**宿主机 thunk，各自针对自己的快照编译，把 `dispatch(version)` 路由到匹配的版本，并调用宿主机已经通过 `GetISteamGenericInterface` 获取的真实版本精确对象。

有两件事让我们比 Proton *更简单*：

- 客体和宿主机都是 **Windows 且版本相同**，所以结构体布局逐字节一致 —— **不需要结构体转换器**（lsteamclient 的大头）。
- 我们**确实**需要参数方向（我们在模拟器 IOCTL 边界上编组，而不是进程内）—— 而 libclang 可以免费得到（见下）。

这**删除**了整套桩/转发/`_DEPRECATED`/前缀层。

### 关键的 libclang 事实（已验证）

- `STEAM_*` 注解（`STEAM_OUT_STRING_COUNT`、`STEAM_ARRAY_COUNT`、`STEAM_BUFFER_COUNT`、…）只有在 `-DAPI_GEN` 下才会展开为 `__attribute__((annotate("out_buffer_count:cbDest;")))`（见 `steam_api_internal.h`）。用 `-DAPI_GEN` 解析，每个注解都会作为 `PARM_DECL` 的一个 `ANNOTATE_ATTR` 子节点出现 —— 正是我们当前从 `steam_api.json` 提取的方向/大小信息。
- 解析参数（已验证的调用形式）：`-x c++ -std=c++14 -D_WIN32 -DAPI_GEN -fms-extensions -Wno-ignored-attributes`
  `-target {i686,x86_64}-pc-windows-msvc -I <sdk>`，源文件为 `#include "steam_api.h"` + `steam_gameserver.h`。
- 当前和旧快照都以 **0 错误**解析；vtable 顺序 = 虚函数 `CXX_METHOD` 的声明顺序；`STEAM_PRIVATE_API` 方法显示为非 `PUBLIC` 的 `access_specifier`。
- libclang 只是**生成期**依赖：`pip install libclang`（捆绑了 win/linux/mac 的原生库）。它从来不是构建或运行时依赖，也不会随任何东西发布。跨目标布局只是一个 `-target` 标志，所以 Linux 开发者也能得到 Windows 布局。

### 验证（复用现有发射器/分类器的原型前端）

| SDK | json+regex（当前） | libclang 前端 |
|---|---|---|
| 当前 SDK | 903 个方法，865 个被编组 | 903 个方法，**863 个被编组**（2 个边界情况待对齐） |
| `sdk_105` | 转发到最新：**60 个被桩掉** | 版本精确：**4 个被桩掉（98%）** |

`ISteamUser012` 和 `ISteamNetworking003` 从各有几个桩变为 **0 个** —— MW2 需要的每个方法都直接针对 1.05 头文件编组。

## 来源与许可（这是单独决策的原因）

这些头文件是 Valve 的 **Steamworks SDK**，受 **Steamworks SDK 许可**约束（Proton 的 `lsteamclient` 中的 `LICENSE` *就是*那份许可；头文件标注 `Copyright Valve, All rights reserved`）—— **不是** Proton 的 GPL。该许可只允许再分发 `redistributable_bin`（DLL），绝不允许再分发头文件，且第 2.4 条款限制逆向工程。因此：

- **绝不**提交/内置头文件或生成的（派生）代码 —— 像今天一样被 gitignore。
- 优先**从 Valve 自己的 `ValveSoftware/Proton` 做 `FetchContent`**（固定带扁平 `steamworks_sdk_*` 目录的旧 tag，`GIT_SHALLOW`，不带子模块）：由*用户*的构建直接从 Valve 获取头文件；项目本身只发布一个 URL（工具提供者，而不是分发者）。
- 把获取放在 Steam 选择加入（opt-in）开关之后，让默认/公共构建不会从 Valve 拉取任何东西。
- 旧版本是**冻结的**，所以较旧的 Proton tag 对它们是正确的来源；当前版本来自官方 SDK。
- 支撑这座桥的开放问题是在法定**互操作性**权利（欧盟软件指令第 6 条，这是无逆向工程条款无法放弃的；美国 §1201(f)）适用于*半虚拟化*桥——它转发到真正的 Steam——的背景下。我们认为这是可辩护的互操作性 —— 该桥没有重新实现、模拟或绕过 Steam 中的任何东西；它只是让沙箱内一个真实持有许可的客户端能够被访问 —— 但精确的契约与法条边界因司法辖区而异，所以这是一个项目在持续跟踪的开放法律问题，而非已定论的许可。

## 剩余实现工作

1. **前端**（原型：`src/tools/steam-bridge-generator/clang_frontend.py`）：并入 `generate.py`，弃用 `steam_api.json`/regex 路径。
2. **版本精确发射**：按版本发射带有各版本自己索引的客体代理 + 宿主机 thunk；`create_proxy`/`dispatch` 按 version→tag 路由；移除前缀转发、`_DEPRECATED` 别名和打桩。
3. **构建**：客体和宿主机两侧每个版本一个隔离的对象库（镜像现有的 `steam-shim-v105` 模式），各自针对自己的快照编译。
4. **CMake**：`SOGEN_ENABLE_STEAM` 选项（默认 ON，可禁用）；`FetchContent` 快照；运行生成器（需要 libclang）。
