# Steam 桥（半虚拟化 Steamworks）

Steam 桥让运行在 Sogen 内的客体 Windows 游戏能够与**宿主机上真实的 Steam 客户端**通信。它是 GPU 桥的 Steam 对应物：不是重新实现 Steamworks，而是把 Steamworks 接口调用转发到宿主机自己的 `steamclient64.dll`，让客体看到宿主机的已登录账号、好友、统计、匹配等。

设计镜像 Proton 的 `lsteamclient`，但宿主机是 Windows，所以两侧 ABI 一致，调用约定由编译器 —— 而不是手写编组 —— 处理。

## 架构

```
  guest game
      │  loads its own untouched steam_api(64).dll   (never modified)
      ▼
  our steamclient(64).dll  shim         src/samples/steam-shim/
      │  CreateInterface / Steam_BGetCallback / Steam_GetAPICallResult
      │  proxies derive from the real ISteam* interfaces (exact vtable)
      ▼
  \\.\SogenSteam  io_device             src/windows-emulator/devices/steam_bridge.cpp
      │  IOCTL: get_version / create_interface / release / invoke /
      │        run_callbacks / get_api_call_result
      ▼
  steam-host-backend  (C ABI)           src/steam-host-backend/
      │  owns the SDK headers + generated thunks; loads the host steamclient64
      ▼
  host steamclient64.dll  →  real Steam client
```

**为什么是 `steamclient` shim 而不是 `steam_api` shim。** 游戏自带它们自己的、带版本的、有时带 DRM 包装的 `steam_api(64).dll`，我们绝不能改动游戏文件。`steamclient` 边界（`CreateInterface` + `Steam_*` 泵）是稳定的接缝。游戏自己的 `steam_api` 加载我们的 `steamclient` 并做它正常的 `CCallback` 分发；我们只服务接口调用以及它底下的回调/调用结果泵。

## 组件

| 路径 | 作用 |
|------|------|
| `src/tools/steam-bridge-generator/` | 解析 SDK 头文件并生成客体代理 + 宿主机 thunk。 |
| `src/steam-generated/` | 生成器输出（`*.generated.hxx`）。**被 gitignore —— Valve 衍生，绝不提交。** 在本地重新生成。 |
| `src/steam-bridge-protocol/` | 线上协议：IOCTL 代码和请求/响应结构。架构无关（固定 8 字节标量槽）。 |
| `src/steam-host-backend/` | 纯 C ABI 背后的静态库；拥有 SDK 头文件 + thunk + 宿主机 `steamclient64` 引导。 |
| `src/windows-emulator/devices/steam_bridge.cpp` | `\\.\SogenSteam` io_device。已加固：把所有客体输入视为敌对。 |
| `src/samples/steam-shim/` | 客体 `steamclient(64).dll` shim（32 位和 64 位）。 |

## 代码生成

生成器解析 SDK 的 `isteam*.h` 头文件（而不是 `steam_api.json` —— 它列出非虚的内联辅助函数并省略 `STEAM_PRIVATE_API` 方法，所以不是 vtable），以恢复每个接口真实的有序虚函数列表。它生成：

- **客体代理**，派生自真实的 `ISteam*` 接口，因此 vtable 布局精确。
- **宿主机 thunk**，直接调用 `((ISteamX*)p)->Method(...)`。

两半都 `#include` 真实的 SDK 头文件，因此编译器发出正确的 ABI（寄存器 vs 隐藏指针结构体返回、XMM 中的浮点、32 位的 `thiscall`、按值传递的 `CSteamID`、…）。这与 `lsteamclient` 相同，避免在约 900 个方法上手卷 ABI。

覆盖率约 95%（865/903）。剩下的是真正需要定制的尾部：函数指针参数（宿主机回调无法跨越边界）、带类型标签的 `void*` 配置，以及少数双指针 —— 每个都需要按方法处理，而不是更多的通用机制。

### 接口版本在快照之间不唯一

接口版本字符串**不能**标识单一的 vtable 布局。Valve 会在不提升版本的情况下给接口追加方法，所以同一个字符串在不同 SDK 快照中含义不同。以 `SteamFriends005` 为例：

| 快照 | 方法数 |
|----------|---------|
| 1.02x、1.03 | 22 |
| 1.04–1.08 | 23（`ActivateGameOverlayToStore` 追加在索引 22） |
| 1.09 | 24（`SetPlayedWith` 追加在索引 23） |

所以 tag 查找不能取第一个声称拥有该版本的快照 —— 那可能交给游戏一个比编译它的 `steam_api` 所针对的 vtable *更短* 的 vtable，而对靠后槽位的调用会跑出 vtable 末尾，落入 `.rdata` 中紧随其后的任何内容。（MW2 调用了 `ActivateGameOverlayToStore`，落到下一个类的 `vtable[-1]`，也就是它的 RTTI Complete Object Locator 上，然后把 RTTI 数据当代码执行。）

这些变化永远只是*追加*，所以最长的变体是满足每个游戏的超集：**两侧都选取对请求版本方法数最多的快照**（shim 中的 `sogen_make_proxy`、后端中的分发）。它们必须使用相同的规则，否则方法索引在两侧含义不同。每个 tag 为此导出 `sogen_steam_method_count_<tag>`，由每 tag 的 `steam_versions.generated.hxx` 表支撑。

这个"只追加"属性是承重的，所以对照所有快照做了检查而不是想当然。34 个版本字符串提供了不同布局；每个中较短的布局都是最长布局的前缀。没有任何快照在固定版本字符串下重排或插入方法 —— 唯一的同索引差异是弃用*重命名*，它们保留槽位和签名（`SteamGameServer010` 槽位 12 `SetGameType`→`SetGameTags`；`SteamUser021` 槽位 3/4 `InitiateGameConnection`→`..._DEPRECATED`；`SteamClient017` 槽位 31/32 `Set_SteamAPI_…`→`DEPRECATED_…`）。如果未来某个快照在固定版本下*真的*重排，取最长会静默错误分发，所以添加快照时要重新运行那个检查。

### 构建桥

生成的代码不提交。构建由 `SOGEN_ENABLE_STEAM` CMake 选项门控（默认 ON）：

1. CMake 通过 `FetchContent` 获取 Valve `ValveSoftware/Proton` 仓库中内置的 Steamworks SDK 头文件快照（tag `proton_7.0`），因此头文件直接来自 Valve，从不存储在本仓库中。
2. 配置时 `generate.py`（需要 `libclang` pip 包）解析每个快照，把版本精确的代理 + 宿主机 thunk 发射到构建目录。
3. 如果缺少 Python 或 `libclang`，或获取没有得到快照，桥会警告并自行禁用，设备回退到空后端（客体 Steam 调用干净地失败）。

## 安全模型

客体不可信（它可能是恶意软件）。桥转发到宿主机**真实**的 Steam 客户端，它运行在沙箱*之外*，拥有宿主机用户的账号和文件系统 —— 因此有两个不同的攻击面：编组中的内存损坏，以及*语义上*不安全的 Steam 方法。

### 线上路径的内存安全

`\\.\SogenSteam` 设备对来自客体的每个长度/偏移做边界检查：invoke 参数 blob 受输入缓冲区约束，回复受输出缓冲区约束；客体提供的分配大小有上限（`cap_bytes`/`cap_count`，16 MiB）。缓冲编组在宿主机侧是两遍的（读取所有自定界输入 → 大小已知 → 分配受限缓冲 → 调用 → 写回）。无大小的 `char*`/`void*` 指针从不被当作单元素处理 —— 除非有显式大小属性或相邻的大小参数，否则它们被桩掉。

> **缓冲计数被钳制到分配大小，而不仅仅是分配被钳到上限。** 缓冲的大小参数是一个独立的按值参数，它同样传给真实的 Steam 方法。如果只钳制分配而传原始计数，客体就可以让计数 `> 16 MiB`，使方法读写越过（更小的）宿主机缓冲区 —— 这是宿主机堆越界，也是审计中发现的唯一真实损坏 bug。因此生成器会把传给 Steam 的计数钳到已分配容量，使声明的大小和缓冲大小永不分离。（`generate.py` 中的 `clamp_count`。）

### API 级策略（默认转发 + 黑名单）

编组正确还不够：有些 Steamworks 方法无论如何安全编组，都是逃逸/外泄原语。生成器拒绝转发它们的黑名单 —— 宿主机 thunk 发出 `blocked(...); return steam_host_unsupported;`（失败关闭；客体看到未实现的方法）。见 `generate.py` 中的 `BLOCKED_INTERFACES` / `BLOCKED_METHODS`。黑名单重点：

- **宿主机文件系统** —— `ISteamUGC`（宿主机路径读取 → 公开上传）、`ISteamScreenshots`（宿主机路径读取）、`ISteamHTMLSurface`（`file://`、JS、宿主机剪贴板）、`ISteamRemoteStorage::UGCDownloadToLocation`（写入任意宿主机路径）、`ISteamApps::{GetFileDetails,GetAppInstallDir,MarkContentCorrupt,InstallDLC,UninstallDLC,Request*ProofOfPurchaseKey*}`、`ISteamInput::SetInputActionManifestFilePath`。普通的 `ISteamRemoteStorage` 文件 I/O（`FileRead`/`FileWrite`/流/`Delete`）**不**被封 —— 那是 Steam Cloud，客体无法逃逸到达任意宿主机文件的按应用隔离虚拟文件系统；封掉它只会破坏合法的云存档。
- **凭证铸造 / 账号变更** —— `ISteamUser` 认证票据（`GetAuthSessionTicket`/`GetAuthTicketForWebApi`/`RequestStoreAuthURL`/`*EncryptedAppTicket`/`AdvertiseGame`）、`ISteamGameServer::GetAuthSessionTicket`、`ISteamFriends` overlay-web / 协议注册 / persona 改名 / 社交发送方法，以及 `ISteamRemoteStorage` 的 `FileShare`/`Publish*`（以宿主机账号公开发布文件）。

网络可达（`ISteamHTTP`、networking/sockets 接口）是故意不封的：有了 `--network live`，客体已经通过模拟的 AFD 套接字设备拥有宿主机网络访问，所以在这里封没有收益。`--network offline`（默认）下设备使用空后端，因此没有客体 Steam 调用到达宿主机。身份读取和 `ISteamFriends::ActivateGameOverlayToStore`（DLC/商店按钮）被有意允许。匹配按方法*名称*逐快照进行，所以缺少所列方法的版本只是永不匹配，且该清单能渡过桥已在处理的版本漂移。

反向回调（`callback_token`）参数也会拒绝已知范围之外的客体 `type`，而不是把一个空代理交给 Steam（那会在下一次虚分发时崩溃宿主机）。

## 回调

`steamclient64.dll` 直接导出 `Steam_BGetCallback` / `Steam_FreeLastCallback` / `Steam_GetAPICallResult` —— 不需要 `SteamAPI_Init`；我们泵自己的管道。宿主机后端排出普通回调并序列化；设备在 `run_callbacks` IOCTL 上转发它们，以游戏的管道为键。异步的 `SteamAPICall_t` 结果通过 `get_api_call_result` IOCTL 端到端工作。

**反向回调**（匹配服务器浏览器，Steam 调用*进入*游戏实现的 `ISteamMatchmaking*Response` 对象）以通用方式接线 —— 宿主机响应代理、反向记录通道和客体分发 —— 但**投递未经验证**：测试中 Steam 从未调用宿主机代理。疑似修复是在宿主机侧泵匹配帧（`SteamAPI_ManualDispatch_RunFrame`）和/或对着 LAN 列表测试。把服务器浏览器当作"形状在了，不崩"，而不是"可用"。

## 运行真实游戏

游戏加载它**自己的** `steam_api`，不做修改；该 DLL 通过注册表找到我们的 `steamclient`。游戏文件夹里没有任何东西被修改。用 `--network live` 运行；离线模式下桥使用空后端。

1. **让 shim 在客体中可见。** 把 `steamclient(64).dll` 放到一个客体路径（如 `C:\`），或用 `-p C:\steamclient64.dll <host-path>` 和 `-p C:\steamclient.dll <host-path>` 映射它。
2. **把注册表指向它。** 游戏的 `steam_api` 读取 `HKCU\Software\Valve\Steam\ActiveProcess\SteamClientDll64`（32 位为 `SteamClientDll`）。这些是真实的 hive 值 —— 客体打不开仅覆盖层（overlay-only）的键，所以它们必须被烘焙进 `NTUSER.DAT`。在 Windows 宿主机上：复制注册表目录，然后 `reg load` / `reg add` / `reg unload` `ActiveProcess` 块（`SteamClientDll`、`SteamClientDll64`、`Universe`、`ActiveUser`、`pid`），以及 `HKCU\Software\Valve\Steam\{SteamPath,SteamExe}`，然后运行 `analyzer -r <that-registry-dir>`。
3. **App ID。** 两个*独立*的 App ID 上下文都必须正确：
   - **客体侧** —— 客体进程必须知道它的 App ID，否则 `steam_api` 会因 "Steam must be running to play this game." 而使 `SteamAPI_Init` 失败。传 `--env SteamAppId <id>`（以及 `SteamGameId`），或放一个 `steam_appid.txt`。许多零售游戏（如 `iw4mp.exe`）也在内部烘焙了 id。
   - **宿主机侧** —— 设置宿主机环境变量 `SteamAppId=<id>`（在 `analyzer.exe`/`sandbox.exe` 进程上），使后端的 `steamclient64` 会话作用于正确的应用。`connect_locked()` 在首次使用时通过 `std::getenv` 读取它，如果未设置，**会带一条响亮的 stderr 警告回退到 480（Spacewar）**，以便 SDK 测试框架仍能工作。

   > **这个宿主机侧作用域对多人游戏不是可选的。** Steam 把大厅、匹配、rich presence 和 P2P 限定在正在运行的应用内。处于错误应用会话（如 480 回退）可以 `CreateLobby` 并自动进入它*自己的*大厅（`LobbyEnter_t response=1`），但真实 Steam 会以 `k_EChatRoomEnterResponseDoesntExist (2)` 拒绝加入*另一个应用的*大厅。在 MW2 中，这表现为即时被公共比赛踢出 `EXE_HOSTUNREACH`：失败的 `JoinLobby` 让游戏独自留在一个单人大厅（`GetNumLobbyMembers==1`），于是 CL 连接代码回退到一个空的默认大厅 id，`sscanf` 什么也没解析出来，连接就此放弃。把宿主机 `SteamAppId` 设为游戏真实 id（MW2 Multiplayer = **10190**）即可修复，账号在好友列表中会显示为游戏中。所以"用户作用域调用无论如何都能工作"只对身份/好友读取成立 —— **大厅/匹配加入需要正确的宿主机应用上下文。**

### 伪造 "Steam is running"

客体的 `steam_api` 从 `ActiveProcess\pid` 读一个 pid，`NtOpenProcess` 它，并检查它是否存活。Sogen 提供一个合成的 Steam 进程使这能通过：

- `handles.hpp`：`STEAM_FAKE_PROCESS_ID`（**必须等于预置注册表的 `pid`**）和一个伪 `STEAM_PROCESS_HANDLE`。
- `NtOpenProcess` 为该 pid 返回伪句柄（为客体自己的 pid 返回客体自己的句柄）；其他 pid 为 `STATUS_INVALID_CID`。
- 伪句柄永不置信号（存活性 `WaitForSingleObject` 超时 = 存活），`NtQueryInformationProcess(ProcessBasicInformation)` 报告 `ExitStatus = STILL_ACTIVE`，所以 `GetExitCodeProcess` 报告存活。`NtClose` 对伪句柄已经是 no-op。

## 状态

已在模拟器内对着真实 Steam 验证（32 位和 64 位客体都测过）：接口调用返回真实账号（persona、SteamID、好友）、普通回调和异步调用结果。注册表欺骗和合成 Steam 进程已端到端验证。

**MW2（`iw4mp.exe`，32 位 WoW64）能启动到菜单、上线并加入公共比赛。** 用正确的宿主机 `SteamAppId=10190`（见上文"App ID"），完整 Steam 匹配路径可用：`RequestLobbyList`/`CreateLobby`/`JoinLobby`、大厅数据/成员读取、`LobbyEnter_t`/`LobbyCreated_t` 回调，以及用于 IWNet Steam 认证握手的 `ISteamUser::InitiateGameConnection`。早前"卡在 D3D9 初始化模态对话框"的说明已过时（该对话框已通过播种游戏的首启状态消除；与桥无关）。

## WIP / 已知差距

- **反向回调投递（服务器浏览器）— 未验证。** Steam 调用*进入*游戏实现的 `ISteamMatchmaking*Response` 对象已通用接线（宿主机响应代理 + 反向记录通道 + 客体分发），但测试中从未观察到触发。疑似修复：在宿主机侧泵匹配帧（`SteamAPI_ManualDispatch_RunFrame`）和/或对着 LAN 列表测试。当作"形状在了，不崩"，而不是"可用"。MW2 的匹配路径不依赖它（它用大厅 + 直连 UDP），所以这只是驱动服务器浏览器的游戏的差距。
- **方法覆盖约 95%（865/903）。** 定制尾部（函数指针参数、带类型标签的 `void*` 配置、少数双指针）被桩掉，需要按方法处理。
- **通告的本地地址是回环。** 客户端在大厅数据中通告 `addrLoc=127.0.0.1`（`get_local_address`→`getsockname` 在模拟器下返回回环）。对已测试的路径无害，但对任何会拨号本地地址的对端是真实缺陷。
- **生成的代码绝不提交**（Valve 衍生；见"构建桥"）。只有手写的桥被跟踪；在本地重新生成 `src/steam-generated/`。
- **分发依赖于一个开放的互操作性问题**（Steamworks SDK 许可 vs 欧盟软件指令第 6 条 / 美国 §1201(f)）——在 `steam-bridge-versioning.md` 中跟踪，不是已定论的许可。
