# Windows UI 模拟现状

## 目标

让客体 USER/GDI 拥有 UI 行为和像素。

目标终态：
- 客体窗口过程正常运行
- 客体 USER/对话框/控件逻辑决定行为
- 客体 GDI 绘制产生像素
- 后端只呈现像素并转发输入
- 宿主机侧的控件镜像和回退绘制可以移除

## 这项工作存在的原因

跨平台 UI 原型证明了后端接缝可行。
历史路径包括：
- Win32 后端原型
- SDL 后端
- web 后端

当前分支使用 SDL3 做原生宿主机 UI，web 后端用于 playground/浏览器路径。
可见控件仍依赖 hack 或宿主机原生控件。
真正的模拟目标更强：内置控件、对话框和自定义 `WM_PAINT` 应该基于客体语义工作，而不是后端特定的绘制。

## 总体状态

### 已完成

- 通用 `ui_backend` 接缝存在
- 当前分支接入了 SDL 和 web 后端
- Win32 后端曾作为原型存在，但已为 SDL3 路径移除
- SDL 现在是 SDL3 可用时的原生默认后端
- 客体对话框流程比以前好得多
- `MessageBox` 不再依赖导出钩子 hack
- 内置控件回调分发到达 ntdll 客户端 thunk，但这些 thunk 在内置控件真正绘制之前仍需要正确的 user32 客户端初始化
- 内置 `Button` / `Static` 在绘制期间不再崩溃，因为现在分配了正确的额外字节：
  - `Button` -> `cbWndExtra = 8`
  - `Static` -> `cbWndExtra = 8`
  - `#32770` -> `cbWndExtra = 30`
- 临时自定义样本已添加：
  - `src/samples/custom-paint-sample/`
- GDI 系统调用脚手架：
  - `NtGdiLineTo`
  - `NtGdiMoveToEx`
  - `NtGdiRectangle`
  - `NtGdiPatBlt`
  - `NtGdiExtTextOutW`
  - `NtGdiFlush`
  - `NtGdiGetRealizationInfo`
  - `NtGdiSelectBrushLocal`
  - `NtGdiSelectPenLocal`
- 临时调试位图字体已添加：
  - `src/windows-emulator/debug_font.hpp`
- `NtUserGetClientRect` 已添加，使 `GetClientRect` 返回正确尺寸
- DC 支撑的绘制表面存储在 `gdi_dc_state` 中
- 子控件绘制现在被合成进顶层窗口表面，而不是依赖独立的宿主机子 HWND
- SDL 后端：客体绘制表面经由 `EndPaint` → `present_surface` → SDL 纹理呈现
- **根因已修复**：`completion_NtUserMessageCall` 在 `WM_PAINT` 后调用了 `present_window_surface`（宿主机回退），覆盖了客体绘制的纹理。现在只调用 `validate_window`。

### 当前事实

- `custom-paint-sample` 现在能再次 visibly 渲染几何图形：
  - 边框线显示
  - 十字线显示
- `custom-paint-sample` 绘制路径完整执行：
  - `BeginPaint`
  - 通过 `NtGdiSelectBrushLocal` / `NtGdiSelectPenLocal` 创建并选择 GDI 画笔/画刷
  - `MoveToEx` 经由 `NtGdiMoveToEx`
  - 线系统调用（`NtGdiLineTo`）
  - 填充系统调用（`NtGdiPatBlt`）
  - `EndPaint` → `present_surface` → SDL 纹理更新并渲染
- 子控件绘制可以喂给顶层合成表面
- 宿主机回退不再在 `WM_PAINT` 后覆盖客体绘制
- 经由 `NtGdiExtTextOutW` 中的临时 8x8 调试字体的直接文本渲染路径现已存在
- 用户已在视觉上验证强制的直接文本路径：文本出现且字形镜像 bug 已修复
- 当前文本差距是较小文本的 GDI 批处理行为，而不是原始顶层表面呈现
- `manual-messagebox-sample` 比以前更多地操练 USER/GDI 路径，但内置控件渲染仍不能依赖手动回退绘制。
- 内置 `MessageBox` 已越过之前缺失的 `NtUserSetThreadState`、`NtUserGetIconSize` 和 `NtUserGetCursorFrameInfo` 障碍，但还不能正确工作。
  - `NtUserSetThreadState(value, mask)` 现在被建模为每线程 USER 状态位字段。
  - `NtUserGetThreadState(10)` 返回该位字段，用于对话框清理路径。
  - user32 创建对话框和按钮，然后为消息框图标/静态控件使用预定义类 atom `#2032`。
  - `#2032` 现在归一化为内置 `Static` 类，匹配 `SS_ICON` 控件路径。
  - 当前的障碍不再是单个缺失的系统调用；而是正确的 user32 客户端回调初始化和内置控件绘制。

## 2026-06-06 更新 — 内置 `MessageBox` 端到端渲染并可用

`messagebox-sample`（`MessageBoxA(NULL, "Proceed?", "Question", MB_YESNO | MB_ICONQUESTION)`）
现在渲染出对话框，带 **Yes/No 按钮、它们的标签和消息文本**，且**点击 Yes 返回 `IDYES`**（样本打印 `clicked: yes`）。真实的 user32 控件 wndproc 运行并通过客体 GDI 路径绘制 —— 没有手动回退绘制。

下面的章节（"当前事实"、"内置 MessageBox 调查笔记"、"Ntdll/User32 回调发现"）早于本次更新，为历史保留；它们的一些"障碍"已解决。实际阻塞可见、可交互控件的根本原因原来是四个独立的问题（全部在**宿主机** `C:\Windows\System32\user32.dll` 上用 Ghidra 验证 —— 注意模拟器在没有 `-e root` 时加载的是*宿主机* live 系统 DLL，而不是捕获的 `root` 副本；路由全局量在不同构建之间不同，所以要始终分析模拟器实际映射的那个 DLL）：

1. **控件在对话框变为可见后从不重绘。** user32 的控件 wndproc 把它们的绘制主体门控在父链整体可见上
   （`ButtonWndProcW` → worker `FUN_18000a540` → 可绘制性检查 `FUN_180008030`，它遍历窗口 + 每个祖先，要求 `WS_VISIBLE`，WND `+0x1f` 处的样式字节位 `0x10` =
   `0x10000000`；这映射到 `USER_WINDOW dwStyle@0x1c`、`fnid@0x2a`、`spwndParent@0x30`）。
   MessageBox 在 `WM_INITDIALOG` 期间创建其子控件，它们在对话框仍隐藏时绘制，所以它们正确地跳过绘制。`NtUserShowWindow` 当时只使对话框本身失效。修复：`invalidate_window_tree()`（在 `user.cpp` 中）递归地重新使可见子树失效，从 `NtUserShowWindow` 和 `NtUserSetWindowPos(SWP_SHOWWINDOW)` 在设置 `WS_VISIBLE` 之后调用。

2. **按钮点击命中测试失败。** `is_button_window`（在 `windows_emulator.cpp` 中）比较字面量 `"Button"`，但 `window::class_name` 存储的是原始序号 atom 类（`#1`）。
   修复：同时匹配 `#1`，使 `find_button_child_at` 工作且点击能关闭对话框。

3. **错误的返回码 + 空白按钮标签 —— 两者都来自 `SERVERINFO.MBStrings`。**
   user32 `SoftModalMessageBox` 通过从 `gpsi->MBStrings` 读取每个按钮的 `{caption, control id}` 来构建对话框模板：`gpsi + 0x3A4` 处的一个数组，条目为 `{ WCHAR achName[16]; UINT id @ +0x20; UINT uStr @ +0x24 }`（0x28 字节条目），固定顺序 OK、Cancel、Abort、Retry、Ignore、Yes、No、Close、Help、Try Again、Continue（id 1..11；按 MB 类型的索引表 `DAT_…b3c00/b3c5c/b3c64`）。我们的 SERVERINFO 把它留零 → 按钮得到控件 id 0（所以对话框总是返回 `IDOK`）和空标题（空白标签）。对话框过程（`MB_DlgProc` @ `0x1800530a0`）在 `WM_INITDIALOG` 上把 MSGBOXDATA 存入对话框的 `GWLP_USERDATA`，并在 `WM_COMMAND` 上调用 `EndDialog(control_id)`。修复：`win32k_userconnect.cpp` 中的 `seed_messagebox_button_strings()` 填充 `MBStrings`，从 `try_copy_client_pfn_arrays` **之后**调用（`apfnClientWorker` 填充会清零重叠的尾部）。已在运行时确认 user32 的 `gpsi` == 我们的 `user_handles.get_server_info()`。脆弱的 caption→ID 启发式（`map_dialog_button_caption_to_id`）已移除。

4. **控件真正开始绘制后浮现的缺失 GDI/USER 系统调用：**
   - `NtGdiGetDCDword` —— gdi32 为 `GdiGetIsMemDc` 调用它；此前未实现并中止。返回 0（真实绘制 DC 不是内存 DC）。
   - `NtGdiPolyPatBlt` —— 按钮边框/按下重绘。**条目布局已在运行时对着本构建的 gdi32 验证：`{ int x; int y; int cx; int cy; HBRUSH hbr@+0x10 }`（位置 + 大小），不是带 right/bottom 的 RECT** —— 按钮边框绘制传递像 `{x=96,y=4,cx=1,cy=20}` 这样的薄边，只有理解为宽/高才说得通。
   - `NtUserDrawIconEx` —— 桩为返回成功，使 `SS_ICON` 静态控件完成绘制（图标像素尚未绘制）。

### 本次更新后仍开放

- 消息框**图标**未绘制（`NtUserDrawIconEx` 桩）。
- 经典复选框/单选**勾选符号**未绘制（`NtUserBitBltSysBmp` 桩）；控件框、标签和点击/状态处理可用。

## 系统内置类 atom 解析（已解决，可移植）

**已修复** 于 `resolve_builtin_class_atom`（`syscalls/user.cpp`）。之前的权宜之计让 `normalize_builtin_window_class_name` 硬编码特定构建的类 atom 值（`#2032`/`#2160`/`#12192`→Static）；这在 Windows 构建之间会坏，因为同一个样本在一个系统的 user32 上把 static 解析为 `#2032`，在另一个系统上却是 `#12192`（"missing class"）。

user32 如何获取 atom（宿主机 `user32.dll`，对话框创建工作线程 `FUN_18002a070`）：对话框模板按**系统类序号**存储控件（`0x80`=Button、`0x81`=Edit、`0x82`=Static、`0x83`=ListBox、`0x84`=ScrollBar、`0x85`=ComboBox，写成 `0xFFFF,<ord>`）。user32 通过用 `(ordinal & 0x7F)` 索引 **`gpsi + 0x364`** 处的 WORD 表 `SERVERINFO.atomSysClass[ICLS]` 把序号变成类 atom，然后把这个 atom 传给 `NtUserCreateWindowEx`：

```c
class_atom = *(WORD *)(gpsi + (ordinal & 0x7F) * 2 + 0x364);
```

atom 值按构建分配（此处 Static = `0x7F0`，别处 `0x2FA0`），这就是硬编码列表不可移植的原因。

修复通过读取**同一个** `gpsi + 0x364` 表并把 atom 与 ICLS 索引的条目（0=Button、1=Edit、2=Static、3=ListBox、4=ScrollBar、5=ComboBox）匹配，把一个传入的整数类 atom 反向映射回其规范内置类名。因为查找经由 user32 用于正向映射的同一块内存往返，所以无论构建的 atom 值如何都与 user32 一致 —— 没有硬编码 atom。已验证：消息框对话框现在创建 `Button` 和 `Static` 控件，没有 "missing class" 错误，也没有 `#NNNN` 别名。

已在运行时用 `NtUserCreateWindowEx` 处的运行时探针实证验证：传入的 Button atom `0x1` 匹配 `atomSysClass[0]`，传入的 Static atom `0x7F0` 匹配 `atomSysClass[2]`。

## 内置 MessageBox 调查笔记

当前检查点的 IDA/user32 发现：

- `StartTaskModalDialog` / `EndTaskModalDialog` 不使用 `NtUserSetThreadState`；它们只是禁用再重新启用顶层窗口。
- 找到的唯一 `NtUserSetThreadState` 调用方在 `DefDlgProcWorker` 中。
- `DefDlgProcWorker` 以 `NtUserSetThreadState(value, 0x4000)` 的形式使用调用，设置或清除一个对话框相关的线程状态位。在所检查的调用点返回值被忽略。
- `DialogBox2` 稍后在 owner/前台窗口清理路径之前调用 `NtUserGetThreadState(10)`。
- 盲目的 no-op `NtUserSetThreadState` 不够：它让样本继续，但 `MessageBox` 立即返回 `0`。
- 加入线程状态位字段后，verbose 报告显示 `MessageBox` 走到了对话框/控件创建。然后它因类 atom `#2032` 失败，销毁对话框并打印 `clicked: other (0)`。
- 把 `#2032` 映射到 `Static` 把故障推进到图标/静态设置。

已实现的窄化系统调用进展：

- `NtUserGetIconSize(icon, frame, &cx, &cy)`
  - 对非空图标返回 `TRUE`
  - 写 `cx = 32`
  - 写 `cy = 64`
- `NtUserGetCursorFrameInfo(icon, frame, &rate, &frame_count)`
  - 对非空图标返回图标句柄
  - 写 `rate = 0`
  - 写 `frame_count = 1`
- `NtUserDestroyCursor(icon, flags)`
  - 对非空句柄返回 `TRUE`

`cy = 64` 的值是刻意的，针对在 IDA 中检查过的 user32 静态图标路径：`xxxSetStaticImage` 在布局/绘制前把返回的高度减半，所以这为 32x32 图标建模了一个由 32x64 图标/掩码位图支撑的情形。

当前运行时状态：

- 内置 `messagebox-sample` 现在创建对话框窗口和子窗口。
- 真实的客体绘制路径仍不能让内置按钮/静态控件可见。
- 曾短暂添加一个手动的宿主机侧/模拟器侧绘制实验：
  - 拦截内置 `Button` / `Static` 的 `WM_PAINT`
  - 直接调用模拟器 GDI 原语
  - 从存储的 `window.name` 绘制按钮矩形和文本
- 该实验让文本/按钮可见了，但它是 hack，已被移除。
- 移除该 hack 后，可见的内置控件渲染仍未解决。

## Ntdll/User32 回调发现

重要发现是：内置类 WndProc 的设置不只是指针查找问题。

观察到的当前行为：

- `SERVERINFO.apfnClientA/W` 当前包含 ntdll 客户端 thunk，如：
  - `NtdllButtonWndProc_A/W`
  - `NtdllStaticWndProc_A/W`
  - `NtdllDialogWndProc_A/W`
- 在 IDA 中，这些 ntdll thunk 调用 ntdll `NtUserPfn` 表中的条目。
- 这些表条目最初指向 `UninitUser32Proc`，它打印 `"User32 init not called"` 并中断。
- `user32!InitializeNtdllUserPfn` 用 user32 的 A/W/worker 回调数组调用 `ntdll!RtlInitializeNtUserPfn`。

实验与结论：

- 直接用 user32 的公共 WndProc 数组替换 `SERVERINFO.apfnClient*` 不正确。
- 它让类 WndProc 指向 user32，但 `messagebox-sample` 很快返回 `clicked: other (0)`。
- IDA 解释了原因：`user32!ButtonWndProcW` 立即调用 `ValidateHwnd`。
- `ValidateHwnd` 依赖 user32 客户端/全局/共享状态：
  - `TEB.Win32ClientInfo[8/9]`
  - `gSharedInfo`
  - `gpsi`
  - user32 句柄表元数据
- 我们的合成窗口对象对这个公共 WndProc 路径不够用。
- 从模拟器补丁 ntdll 的 `NtUserPfn` 表也只作为调试实验探索过，不是可接受的解决方案。

可能的正确方向：

- 不添加手动控件绘制回退。
- 不从模拟器代码 ad-hoc 补丁 ntdll 内存。
- 让 user32 客户端初始化路径以类似 Windows 的方式发生，或在 USER 边界以有原则的模型模拟相关效果：
  - `UserClientDllInitialize`
  - `InitializeNtdllUserPfn`
  - `RtlInitializeNtUserPfn`
  - `RtlRetrieveNtUserPfn`
  - user32 共享信息 / 句柄验证状态
- 让回调分发通过 ntdll/user32 回调约定，而不是从宿主机侧代码直接调用公共 user32 WndProc。

## 剩余障碍

由上面的 2026-06-06 更新解决：真实 user32 绘制路径上的内置 `MessageBox` 控件渲染、按钮自绘、点击→`WM_COMMAND`→`EndDialog`、正确返回码、按钮标题、可移植的系统内置类 atom 解析（见"系统内置类 atom 解析"），以及窗口/对话框**背景**填充（顶层表面现在用类背景刷填充 —— 灰色对话框面而不是白色）。仍开放：

- 消息框**图标**像素（`NtUserDrawIconEx` 是成功桩）—— 仍开放；见更新 (8) 了解可行性约束（没有存储的图标像素数据 / 图标身份）
- 用更正确的字体/文本处理逐步替换临时调试字体文本路径 —— 仍开放（文本通过固定单元格的 8x8 `debug_font` 渲染，忽略选中的 `HFONT`）
- 添加更正确的剪切/区域处理
- 在内置绘制需要的地方添加更正确的 ROP / 画刷 / 画笔 / DC 语义
- `NtGdiGetDCDword` 只返回默认值（0）；其他索引（MapMode、GraphicsMode、…）如果控件依赖它们需要真实值

自（更新 (8) 起）已解决：经典复选框/单选**勾选符号**（`NtUserBitBltSysBmp` 现在通过 `draw_system_button_glyph` 绘制），以及小文本 / **批处理 GDI 文本**路径（`flush_gdi_text_batch` 处理 TEB GDI 批，使普通 `TextOutA/W` 无需强制超大 `ExtTextOutW` 即可工作）。

## 2026-06-07 更新 — 输入路由整合进模拟的 win32k 层

指针命中测试是 win32k 的（内核）工作，不是宿主机后端的，也不是客体 user32 的：在真实 Windows 中，原始输入线程对窗口树做命中测试，发送 `WM_NCHITTEST`，并投递**已经定位到特定子 HWND** 的鼠标消息。`DefWindowProc` 从不会把点击转发给子控件，所以模拟器（扮演内核角色）必须做命中测试。早期的 web/SDL 后端各自重新推导了这一点，那才是真正的 hack。

现在已整合：

- `windows_emulator::route_pointer` 是鼠标捕获、子命中测试和窗口本地坐标转换的唯一权威。两个后端只转发 (顶层窗口, 顶层本地 x/y, 按钮状态) 三元组。
- 重复的 `findChildTarget` / `resolveClientTarget` / `windowOrigin` 树遍历已从两个 web 宿主（`page/src/web-ui-host.ts` 和 `ui_backends/web_ui_host.js`）中删除。
- **`GetMessage`/`PeekMessage` 的 `IsChild` 过滤器已修复**：`peek_pending_message` 现在匹配目标是指定过滤器 HWND *或其任何后代* 的消息，所以带非 NULL 顶层的 `GetMessage(&msg, hWnd, …)` 不再静默丢弃子控件消息。此前过滤器只精确相等，hwnd 过滤形式丢失了所有子输入。

### 仍待移入模拟 USER/win32k 路径的宿主机 USER 逻辑（TODO）

这些是模拟的 USER/win32k 层中的有界简化（它们已经住在模拟器核心中，而不是某个宿主机后端），应随时间成长为更有原则的 win32k 模型。它们对当前操练的情形是承重且正确的；下面的条目精确界定它们在何处偏离真实 win32k：

- **`route_pointer` 是简化命中测试**（`windows_emulator.cpp`）。它已经把控鼠标路由整合进 win32k 层（后端只转发 (顶层, 顶层本地 x/y, 按钮) 三元组），并正确跨顶层遵守 `SetCapture`。它与真实 win32k 的偏离：
  - **没有 `WM_NCHITTEST`** —— 所以 `HTTRANSPARENT`、可拖动客户区（`HTCAPTION`）和自定义命中测试不可用。
  - **没有 `WM_SETCURSOR` / `WM_MOUSEACTIVATE`** —— 没有光标变化或点击激活。
  - **近似 z 序** —— `find_child_window_at` 遍历 `process.windows`（按句柄键控的 map），"最后匹配者胜"，即句柄分配顺序，而不是真实 z 序兄弟链。兄弟控件不重叠（对话框）时无害；重叠兄弟时错误。
  - **无 `WS_EX_TRANSPARENT` 处理。**
  有原则的方向是把命中测试从急切的主机事件时刻（`handle_ui_event`，客体代码不能运行）移到**客体线程上下文中的消息弹出时刻**（`GetMessage`/`PeekMessage`），在那里可以同步 `SendMessage(WM_NCHITTEST)` / `WM_SETCURSOR` / `WM_MOUSEACTIVATE` 给 wndproc —— 镜像内核原始输入路径在投递鼠标消息之前发送它们的方式。这是一次真正的重构，单独跟踪。两个较小改进先做：`SetWindowPos`/`BringWindowToTop` 确认保持链同步之后的经由兄弟链的真实 z 序，以及 `WS_EX_TRANSPARENT` 跳过。
- **所有鼠标消息都经由 `route_pointer` 子路由**（`is_pointer_message` 覆盖 `WM_MOUSEMOVE` / `WM_LBUTTON*` / `WM_RBUTTON*`，遵守 `SetCapture`）；仍是上面的简化命中测试。
  （对话框管理器捷径 `handle_dialog_message` / `complete_dialog` 已在 2026-06-07 (4) 中**移除** —— 它结果是死的且有 bug；客体的真实 `DefDlgProc`/`IsDialogMessage` 路径正确地驱动键盘和 `WM_CLOSE`。见下文更新。）
  （`#32770` / `WC_DIALOG` 字面字符串匹配于 2026-06-07 (5) **集中**进 `builtin_dialog_class_name` 常量 + `windows_objects.hpp` 中的 `window::is_dialog()`；不再有散落的 `u"#32770"` 字面量。该值可移植 —— `WC_DIALOG = MAKEINTATOM(0x8002)` 是由 `get_atom_name` 格式化为 `"#32770"` 的固定 Win32 ABI atom —— 所以与经由 `resolve_builtin_class_atom` 解析的特定构建控件类 atom 不同，规范名是可靠的。见下文更新。）
- **`invalidate_window_tree` 是有界简化，不是可移除的作弊**（2026-06-07 (6) 验证承重 —— 禁用子递归会让消息框有绘制的对话框边框但按钮/文本空白）。在隐藏→可见转换（`NtUserShowWindow`、`NtUserSetWindowPos|SWP_SHOWWINDOW`）时，它递归地使窗口和每个 `WS_VISIBLE` 后代失效。它是必需的，因为 user32 控件 wndproc 把它们的绘制主体门控在*整个祖先链*可见上（`FUN_180008030`），所以在 `WM_INITDIALOG` 期间创建的子控件（对话框仍隐藏时）跳过绘制，且祖先之后显示时不会自行重绘。该行为实际上等于 `RedrawWindow(hwnd, RDW_INVALIDATE | RDW_ALLCHILDREN)`，且已住在模拟的 win32k 层（`syscalls/user.cpp`）中，所以*不是*待迁移的宿主机逻辑。它**对今天操练的情形是正确的**（子控件都在显示时新暴露的对话框）；只在还没有更新区域/剪切模型的地方偏离真实 win32k —— 部分暴露、重叠或被遮挡的兄弟、`WS_CLIPCHILDREN`、z 序。有原则的版本是真正的每窗口更新区域 + 剪切子系统（很大，与更广的 GDI 区域工作耦合），单独跟踪而不是作为快速移除项。

## 2026-06-07 (2) 更新 — 跨构建支持：控件 id + 对话框完成

点击在宿主机的 live 系统 DLL（Windows 11，build 26x00）下可用，但在捕获的 **Windows Server 2022 (20348)** 模拟根目录下失败。早期的 "web 后端" 点击 bug 其实是这个构建差异（web 用 Server 2022 根目录，SDL 用宿主机 DLL）；SDL 配 Server 2022 根目录能复现。两个构建特定的差距被修复，都通过对每个构建的 `user32.dll` 反编译验证：

1. **子控件 id 字段在构建之间不同。** 按钮的 `ButtonWndProc` 从一个构建特定的 WND 偏移读取其子控件 id 来构建它的 `WM_COMMAND`：Windows 11 在 **`WND+0x140`** 读 `wID`，Windows Server 2022 在 **`WND+0x98`** 读 `spmenu`（在通知辅助函数中为 `(short)pwnd[0x13]`；`spmenu` 是真实 Windows 存储子控件 id 的地方）。
   模拟器只写 `wID@0x140`，所以在 Server 2022 上每个 `WM_COMMAND` 携带 id `0`，应用的处理函数 fall through 到 `DefWindowProc`。修复（`NtUserCreateWindowEx` / `NtUserSetWindowLongPtr(GWLP_ID)`）：子窗口现在把 id 镜像进 **两个** `spmenu` 和 `wID`，所以内置 wndproc 无论加载哪个构建都能正确通知。这移除了旧的 `handle_ui_event` `find_button_child_at` `WM_COMMAND` 合成 —— 真实的 `ButtonWndProc` 现在在两个构建上都驱动 `WM_COMMAND`。

2. **内置 `MessageBox`/对话框从不结束（真实 `EndDialog` 路径）。** 真实按钮 `WM_COMMAND` 取代合成后，内置消息框点击不再关闭。根因：`NtUserSetDialogPointer` 存储了 DLGINFO 指针，但从未设置 WND 的 "has-DLGINFO" 标志位，位于 **`WND+0x12` 位 0**。user32 的 ensure-dialog-info 辅助函数以该位为门，所以它在*每个*对话框消息上都重新分配一个全新 DLGINFO（观察到：数百次 `NtUserSetDialogPointer` 调用）。模态循环和 `EndDialog` 于是读/写不同的 DLGINFO 块，所以 `EndDialog`（设置 `DLGINFO+0x18 |= 1`、隐藏对话框、投递 `WM_NULL`）永远够不到循环。修复：`NtUserSetDialogPointer` 现在随指针设置/清除 `WND+0x12` 位 0，与内核一致。对话框现在经由真实的 `DefDlgProc` → `EndDialog` → 模态循环路径完成（无合成）；`messagebox-sample` 点击再次返回 `IDYES`。

两个修复都镜像真实 Windows 语义而不是绕过它：子控件 id 真的住在 `spmenu`，内核真的设置 `WND+0x12` DLGINFO 位。把 id 写进 `spmenu` 和 `wID` 两者是一个刻意的构建无关选择（模拟器在创建时不知道映射了哪个 `user32` 构建，且每个构建只读自己的偏移）。已在宿主机（Win11）和 Server 2022 根目录上用 `manual-controls-sample` 和 `messagebox-sample` 端到端验证。

用于追查的临时 `[ui-event]` / `[capture]` / `[sogen-ui]` 输入路由诊断在同一变更中移除；分支不遗留调试日志。

## 2026-06-07 (3) 更新 — 跨顶层的捕获路由 + 正确的 WM_SETTEXT A/W 编组

审查跨构建工作时的两个后续。

### 跨顶层遵守鼠标捕获

`route_pointer` 只经由 `get_window_origin_relative_to_ancestor(captured, top_level)` 转换被捕获窗口的坐标，当事件是为*另一个*不同于拥有捕获的顶层报告的顶层报告时返回 `nullopt` —— 然后回退把事件投递给那个其他顶层。这破坏了跨模拟器顶层的按下/拖拽/释放的 `SetCapture` 契约（被捕获控件可能错过它的 button-up 并保持卡住）。修复：经由**屏幕坐标**（相对于根的原点）转换，使被捕获窗口总能收到事件；对报告顶层之下的被捕获窗口，这退化为之前的偏移。

### WM_SETTEXT 被重编码到目标 wndproc 的编码

对内置控件（如 `Static`）的 `SetWindowTextA` 产生乱码文本。调查穿过几个误导层后找到真正原因：

- **`LARGE_STRING.bAnsi` 不是固定值。** user32 的内部 `_DefSetText(hwnd, text, bAnsi)` 从*wndproc 自己的编码*构建传给 `NtUserDefSetText` 的 `LARGE_STRING`：它为 ANSI proc 设置 `bAnsi` 位，为 wide proc 清除它（仅 Unicode 的标题路径 `xxxSetWindowText` 额外用 `& 0x7fffffff` 掩它）。所以 `bAnsi` 是权威的**当且仅当**到达 proc 的缓冲已匹配 proc 的编码。
- **实际的 bug 在我们这边。** 追踪 `SetWindowTextA` → `NtUserMessageCall(WM_SETTEXT)` 显示 `ansi=1` 且带 ANSI 缓冲（一致），但模拟器通过固定的 `apfnClientA[21]` 回调分发它，并把**原始 ANSI `l_param`** 直接转发给 wide 的 `StaticWndProcW`。wide proc 然后用 `bAnsi=0` 对 ANSI 字节调用 `_DefSetText` → `read_def_set_text_string` 看到 `bAnsi=0` + ANSI 文本 → 乱码。`read_def_set_text_string` 中早期的字节嗅探启发式只是掩盖了这一点。

真实 win32k 以 proc 自己的编码把消息的文本投递给 wndproc。修复记录每个窗口的 wndproc 编码（`window.unicode_proc` —— 内置控件为 wide，否则为注册的 proc），并在 `handle_NtUserMessageCall` 中，当调用者编码（`ansi`）与目标 proc 不同时，把 `WM_SETTEXT` 字符串重编码进一个暂存客体缓冲。暂存缓冲在 `completion_NtUserMessageCall` 中释放。随着载荷现在匹配 proc，`bAnsi` 在 `NtUserDefSetText` 处一致，所以 `read_def_set_text_string` 相信它，字节嗅探特例消失。已验证：状态标签的 `[defsettext]` 翻转为 `bAnsi=0` + UTF‑16，控件文本正确渲染。

用 `RegisterClassExA` 注册类的应用窗口保持 ANSI proc，所以 ANSI `WM_SETTEXT` 匹配且不转换 —— 常见情形无行为变化。（区分 `RegisterClassExW` 应用类需要注册期的 A/W 标志，尚未贯穿；受影响的内置控件已处理。）

## 2026-06-07 (4) 更新 — 宿主机作弊清理：死的 `WM_COMMAND` 路径 + 对话框捷径移除

开始清理"仍待移入模拟 USER/win32k 路径的宿主机 USER 逻辑"下所列的宿主机 USER 作弊。其中两个现在真实 user32 路径驱动控件通知后已是死代码，被移除：

- **`handle_ui_event` 中的 `WM_COMMAND` 控件 id 修正**（`windows_emulator.cpp`）用子控件的 `wID` 重写 `WM_COMMAND` 的 `wParam` 低字。没有 UI 后端发出 `WM_COMMAND` 事件 —— SDL 只投递 `WM_CLOSE` / `WM_KEYDOWN` / `WM_LBUTTONDOWN` / `WM_LBUTTONUP`，web 宿主只投递 `WM_LBUTTON*` + 按键 —— 所以该分支不可达。真实的 `ButtonWndProc` 已经从子控件 id 构建 `WM_COMMAND`（镜像进 `spmenu` 和 `wID` 两者，见更新 (2)）。已移除。
- **`handle_dialog_message` 中的 `WM_COMMAND(BN_CLICKED)` 分支**（`syscalls/user.cpp`）在按钮点击时调用 `complete_dialog`。按钮的 `WM_COMMAND` 是同线程同步 `SendMessage`，user32 在进程内直接分发到 `DefDlgProc` 而不经过 `NtUserMessageCall` 系统调用，所以它永远到不了 `handle_dialog_message`。完成已经走真实的 `DefDlgProc` → `EndDialog` → 模态循环（更新 (2)）。已移除；`l_param`（它唯一的消费者）从 `handle_dialog_message` 的签名中去掉。

然后**整个对话框管理器捷径被移除**（`handle_dialog_message`、`complete_dialog`、`is_dialog_window`）。一个禁用捷径、只让客体真实 `DefDlgProc`/`IsDialogMessage` 路径运行的实验表明它正确处理对话框关闭 —— 而且该捷径不只是冗余，对 `MB_YESNO` 是**错误**的：

- **Enter** → 客体激活默认按钮（Yes）→ `IDYES` / `clicked: yes`。旧捷径硬编码 `VK_RETURN` → `IDOK`，而 YesNo 框根本没有这个按钮。
- **Esc** → 被忽略，匹配原生 Windows（YesNo 框没有 Cancel 按钮）。旧捷径强制 `VK_ESCAPE` → `IDCANCEL`。（对 `MB_OKCANCEL`/`MB_YESNOCANCEL`，客体路径发送 `WM_COMMAND(IDCANCEL)` 给真实 Cancel 按钮，所以那里也可用。）
- **窗口 X** → 对 YesNo 框正确禁用（变灰），匹配原生 —— X 没有效果因为没有 Cancel 命令。旧捷径伪造 `WM_CLOSE` → `IDCANCEL`。
- **鼠标 Yes/No** → 不变，仍由真实 `ButtonWndProc` → `DefDlgProc` → `EndDialog` 驱动。

所以该捷径被直接删除（在宿主机 Win11 DLL 上交互验证：Enter→`clicked: yes`，Esc→无操作，X 禁用，鼠标 Yes/No 可用）。随之而去的还有现已死亡的对话框簿记：`window` 字段 `dialog_pointer` / `dialog_flags` / `dialog_result`（及其序列化），以及 `NtUserSetDialogPointer` 中的 `dialog_flags` 重新应用分支 —— 它现在只设置 `WND+0x12` "has DLGINFO" 位并把指针写进窗口额外字节（它真正的内核工作，见更新 (2)）。标准冒烟套件（`analyzer.exe -s test-sample.exe`）仍通过。

也把 **`#32770` 字面字符串**的清理记入上面的 TODO 列表（集中 `WC_DIALOG` 魔法字符串；它可移植，不像构建特定的控件 atom）。

## 2026-06-07 (5) 更新 — 集中化 `#32770`（`WC_DIALOG`）字面量

对话框类 `#32770` 此前在两个编译单元（`syscalls/user.cpp`、`syscalls/gdi.cpp`）的七个位置被裸字符串 `u"#32770"` 匹配；`gdi.cpp` 甚至把该字面量*以原始形式*重复为 `class_name` 比较，因为 `normalize_builtin_window_class_name` 是 `user.cpp` 私有的。这是 `WC_DIALOG = MAKEINTATOM(0x8002)`，一个由 `get_atom_name` 格式化为 `"#32770"` 的固定 Win32 ABI atom，所以 —— 与经由 `SERVERINFO.atomSysClass` 解析的按构建控件类 atom 不同 —— 规范名可移植且可安全直接匹配。

集中进 `windows_objects.hpp` 中的单一事实来源：

- `inline constexpr std::u16string_view builtin_dialog_class_name = u"#32770";`（有文档）。
- 用于窗口实例检查的 `window::is_dialog()`（`NtUserShowWindow` 可见性同步、gdi.cpp 背景填充）。这也修复了 raw-vs-normalized 不一致 —— `is_dialog()` 是原始 `class_name` 比较，这完全等价，因为 `normalize_builtin_window_class_name` 从不把任何东西映射*到* `#32770`（它只规范化控件别名），所以 `normalize(x) == "#32770"` 当且仅当 `x == "#32770"`。对话框存储的 `class_name` 总是字面上的 `"#32770"`（它来自 `WC_DIALOG` atom）。
- 名称分类辅助函数（`is_builtin_window_class_name`、`get_builtin_window_fnid`、`ensure_builtin_window_class`）使用该常量。

纯重构，保持行为。除常量定义外不再有 `u"#32770"` 字面量。已验证：release + tidy（clang-tidy）构建干净，冒烟套件通过。

## 2026-06-07 (6) 更新 — `invalidate_window_tree` 确认承重（保留）

调查了宿主机作弊清单上的下一个条目，得出结论：它与对话框捷径不同，**不是**可移除的作弊。一个禁用子树递归（只使被显示窗口失效）的实验在 `messagebox-sample` 上交互验证：对话框边框仍绘制但 **Yes/No 按钮和消息文本空白**，因为 user32 控件 wndproc 把它们的绘制主体门控在整个祖先链可见上 —— 在 `WM_INITDIALOG` 期间创建的子控件（对话框仍隐藏）跳过绘制，且在祖先之后显示时不会自行重绘。

所以递归保留。作弊清单条目被重述：`invalidate_window_tree` 是有界简化（≈ `RedrawWindow(RDW_INVALIDATE | RDW_ALLCHILDREN)`），已住在模拟的 win32k 层中，对今天操练的情形正确；与真实 win32k 的唯一偏离是缺少更新区域/剪切模型（部分暴露、兄弟遮挡、`WS_CLIPCHILDREN`、z 序），那是一个单独的、更大的子系统，而不是快速移除项。无代码变更。

## 2026-06-07 (7) 更新 — `route_pointer` 界定范围为有界 win32k 简化（保留）

调查了宿主机作弊清单的最后一个条目。与 `invalidate_window_tree` 一样，`route_pointer` 是承重的，且已住在模拟的 win32k 层中（`windows_emulator.cpp`）—— 它是为了整合 SDL/web 后端曾经重复过的命中测试而创建的，所以不是待迁移的宿主机逻辑。它对今天操练的情形正确处理 `SetCapture`（包括跨顶层）和子命中测试。

通过 `grep` 确认 `WM_NCHITTEST` / `WM_SETCURSOR` / `WM_MOUSEACTIVATE` / `HTTRANSPARENT` 在全树中任何地方都不存在。与真实 win32k 的偏离（没有 `WM_NCHITTEST`、没有 `WM_SETCURSOR` / `WM_MOUSEACTIVATE`、句柄顺序 z 序而不是兄弟链、没有 `WS_EX_TRANSPARENT`）现已在上面的 TODO 条目中精确界定。深部分不是快速修复的架构原因：那些消息是*同步发送*的，但命中测试目前运行在宿主机事件时刻（`handle_ui_event`），客体代码不能在那里运行；有原则的修复把命中测试移到消息弹出时刻（`GetMessage`/`PeekMessage`）的客体线程上下文中，以便 wndproc 可以被同步调用。无代码变更。

至此，宿主机作弊清单上的每个条目都已解决或界定：#1/#2（`WM_COMMAND` 修正）和 #3（对话框捷径）作为死代码移除，`#32770` 已集中，剩余两项（`invalidate_window_tree`、`route_pointer`）验证为承重，并作为有界 win32k 简化重述，记录了它们有原则的修复方案。

## 2026-06-07 (8) 更新 — GDI 文本/位图状态同步 + ExtTextOutW 颜色修复

调查了 GDI 文本/位图路径，发现文档的"剩余障碍"清单已漂移 —— 列为开放的两项实际已实现：

- **批处理 GDI 文本** —— `flush_gdi_text_batch`（`syscalls/gdi.cpp`）遍历 TEB `GdiTebBatch`，在 `NtGdiFlush` 上重放 `TextOut`（`k_gdi_batch_cmd_text_out`）和 `PolyPatBlt` 记录，使用每条批记录携带的前/背景色。普通 `TextOutA/W` 流经此处，所以不需要超大 `ExtTextOutW`。
- **复选框/单选勾选符号** —— `NtUserBitBltSysBmp` 调用 `draw_system_button_glyph` 绘制该符号；框、标签和点击/状态已可用。

修复了**直接** `NtGdiExtTextOutW` 路径（非批处理路径）中一个真实 bug，它读取错误的 DC 颜色：文本用 DC **画笔**颜色绘制，`ETO_OPAQUE` 用 DC **画刷**颜色填充。Windows 使用 DC **文本颜色**（`SetTextColor` / `crForegroundClr`）绘制字形，用 DC **背景颜色**（`SetBkColor` / `crBackgroundClr`）做不透明填充 —— 批处理路径已经正确。增加 `get_dc_text_color` / `get_dc_background_color`（在 `0x28` / `0x20` 处读取 `DC_ATTR`，与文件中已有的 ReactOS 衍生 `DC_ATTR` 偏移一致），并让直接路径指向它们。Release + 冒烟通过。

该区域真正仍开放的：**`NtUserDrawIconEx`**（消息框图标）是成功桩，且图标模型没有存储的像素数据或追踪的图标身份 —— 绘制真实系统图标需要要么把标准消息框图标合成成字形（类似 `draw_system_button_glyph`，外加知道请求的是哪个图标的方法），要么加载并光栅化真实的 `imageres.dll`/`user32` 资源。且文本仍使用固定单元格的临时 8x8 `debug_font`，忽略选中的 `HFONT`（大小/字族/粗细）—— 真实字体处理仍是后续大项。

## 2026-06-08 (9) 更新 — `NtGdiStretchDIBitsInternal` present 路径（软件渲染窗口）

实现了 `NtGdiStretchDIBitsInternal`（`syscalls/gdi.cpp`，注册于 `syscalls.cpp`）。它此前是一个未实现的系统调用，会中止任何经由 `StretchDIBits` 向窗口 blit DIB 的应用。处理程序镜像 `NtGdiSetDIBitsToDeviceInternal`（32bpp `BI_RGB`，自顶向下/自底向上），但增加最近邻源→目标缩放，然后 —— 对窗口支撑的 DC —— 经由 `present_surface` 呈现 DC 的表面，与 `NtUserEndPaint` 使用的同一接缝（该路径上没有 `BeginPaint`/`EndPaint`；应用持有 `GetDC` 窗口 DC 并直接 blit）。

这使**客体内的软件 Vulkan/GL 驱动能显示**：加载 SwiftShader（`vk_swiftshader.dll`）的客体在客体内存中渲染帧，并经由 `GetDC` + `StretchDIBits` present，现在能显示在屏幕上。用 `vulkan-cube-sample.exe vk_swiftshader.dll` 验证 —— 旋转立方体显示在窗口中（在 SwiftShader 约 11 s 的客体 JIT 预热后约 1 FPS；软件光栅化完全在模拟器内运行，所以这是慢的正确性路径，而不是快路径 —— 经由 `vulkan-shim.dll` 的 GPU 桥是高性能路线）。

## 后端对齐笔记

SDL 和 web 仍需要独立的宿主机后端，因为它们的平台集成真正不同：

- SDL 拥有原生窗口、纹理和 SDL 事件轮询。
- web UI 拥有 canvas 合成、worker `postMessage` 和浏览器键盘/鼠标事件。

后端契约应保持小于 USER/GDI 语义：

- C++ USER/GDI 代码创建客体窗口、绘制像素、合成子表面并决定 Win32 消息。
- 后端为顶层窗口呈现 `ui_surface_desc` 像素。
- 后端以通用形状转发原始输入。

当前对齐工作：

- web 宿主现在消费完整 `ui_window_desc` 元数据，而不仅是 rect/title/visible/enabled。
- web 宿主在创建 `ImageData` 前把 `ui_surface_format::bgra8` 转成 canvas RGBA。
- web 宿主转发顶层本地鼠标坐标，与 SDL 的坐标形状匹配。
- SDL 不再拥有直接的 `Button` 点击到 `WM_COMMAND` 合成。
- 直接的子 `Button` 命中测试求 `WM_LBUTTONDOWN` 现在住在 `windows_emulator::handle_ui_event`，所以 SDL 和 web 共享该行为。

剩余的设计方向是继续把 Win32 行为移出宿主机后端。未来的绘制或控件修复一般应落在模拟器侧的 USER/GDI 路径中，然后只要后端呈现表面并转发输入，就能在 SDL 和 web 中都工作。

## 下一步

1. 绘制消息框图标（`NtUserDrawIconEx`）—— 需要先确定图标身份/像素数据方案（见更新 (8)）；对话框背景填充已完成。
2. 保持内置控件绘制在真实回调/user32 路径上；不要重新引入手动 `Button` / `Static` 绘制回退。
3. 用真实 `HFONT` 感知的字体/文本处理替换临时 8x8 `debug_font`（批处理文本管道已就位；差距是按选中大小/字族的光栅化）。

## 本检查点的非目标

现在不试图完成整个 USER/GDI。

## 推荐的验证命令

自定义样本：

```cmd
build\release\artifacts\analyzer.exe -s build\release\artifacts\custom-paint-sample.exe
```

手动消息框样本：

```cmd
build\release\artifacts\analyzer.exe -s build\release\artifacts\manual-messagebox-sample.exe
```

内置消息框样本（渲染 Yes/No + 文本；点击 Yes 打印 `clicked: yes`）：

```cmd
build\release\artifacts\analyzer.exe -s build\release\artifacts\messagebox-sample.exe
```

带根映射的 MessageBox 样本：

```cmd
build\release\artifacts\analyzer.exe -s -e root -p c:\messagebox-sample.exe build\release\artifacts\messagebox-sample.exe c:\messagebox-sample.exe
```

## 2026-06-14 更新 — MW2 (iw4x) `DefWindowProc` 崩溃：空消息表全局量 + `NtUserMessageCall` 之墙

在 WHP 下把《使命召唤 MW2》（32 位 `iw4x.exe`，WoW64）带进游戏中时发现（`EMULATOR_WHP=1 analyzer -vc -ss "...\iw4x.exe" -console -stdout +devmap mp_rust`）。游戏在启动时显示"硬件已更改，重置设置？"对话框；为它处理任何窗口消息都会以 `C0000005` 崩溃。（与单独落地的 x87/3D 投影修复无关。）

### 根因（已确认）

Sogen 运行**真实**的 `user32.dll`。游戏的窗口过程（`iw4x.exe` `FUN_004731f0`，一个以 `DefWindowProcA` 结尾的普通 `LRESULT(HWND,msg,wParam,LPARAM)`）调用 `DefWindowProcA` → `DefWindowProcWorker`（`user32.dll FUN_10029310`，Ghidra 基址 0x10000000）。在 `user32+0x29378` 处，它在消息 switch 之前执行：

```asm
CMP  msg, [user32+0xccbc0]        ; DAT_100ccbc0 = message-table limit
JA   skip
MOV  EAX, [user32+0xccbc8]        ; DAT_100ccbc8 = pointer to per-message bit array
TEST byte ptr [(msg>>3) + EAX], 1<<(msg&7)   ; <-- FAULT
```

转储活全局量（客体 `user32_base = fault_ip - 0x29378`；读 `+0xccbc0/+0xccbc8/+0xccbd0/+0xccbd8`）显示**四个全是 0**。所以整个 DefWindowProc 消息表从未到达 user32。上限为 0 时，只有 `DefWindowProc(WM_NULL=0)` 到达读取且 `[0+0]` 故障 —— 而加载循环泵 `WM_NULL`，所以可靠命中。

### 已解决：这些全局量是 `SHAREDINFO.DefWindowMsgs` / `DefWindowSpecMsgs`

这四个全局量是 `_gSharedInfo.DefWindowMsgs` (0x218) 和 `.DefWindowSpecMsgs` (0x228)。32 位 `user32!UserClientDllInitialize` 把 0x238 字节的 user-connect 回复原样拷贝进 `_gSharedInfo`，并用与 64 位客户端相同的 8 字节槽 `SHAREDINFO` 布局读取它（因此 `…bc0/…bc8/…bd0/…bd8` 的 8 字节间距）。WoW64 connect 过去用单独的 `WIN32K_USERCONNECT32` 布局编写，其 `wndmsg_count`/`wndmsg_table` 字段落进了错误的槽，所以消息表从未到达 user32。两个 WoW64 写入方（CSRSS ApiPort 回复在 `ports/api_port.cpp`，以及 `NtUserProcessConnect`）现在都经由与 64 位路径相同的 `populate_user_shared_info`，`WIN32K_USERCONNECT32` 已消失。

复现：创建一个窗口并调用 `DefWindowProcA(hwnd, WM_NULL, 0, 0)` 的 32 位客体，在修复前于 user32 内部对地址 0（空位数组指针）的读取处故障，修复后返回 0。

在运行时播种这四个全局量（对 `user32+0x29310` 的定向**执行钩子**；写 `DAT_100ccbc0=0x63F`、`DAT_100ccbc8`=客户端消息位表地址，对 `…bd0/…bd8` 同样）已经**消除了崩溃** —— 对话框渲染 —— 这就是在源字段被找到之前验证诊断的方式。

### 更深的墙：`NtUserMessageCall` 服务器回调（`call to 0x2`）

用**非零**位表（填充的 `client_message_bits`），DefWindowProc 把被标记的消息转发给服务器（`NtUserMessageCall`）—— 而那条路径崩溃：`ntdll.dll+0xd6f1c`（64 位，经由 `wow64cpu.dll`）处 `call to 0x2`，即**通过一个空/垃圾函数指针的服务器→客户端回调分发**（`apfnClient*` / `gapfnScSendMessage` 回调表 —— 见上文"Ntdll/User32 回调发现"）。位表**为零**时，没有东西转发：不崩溃，但对话框的**按钮从不被创建**（控件需要该服务器处理），所以对话框不可操作。

所以可用对话框需要两者：(1) 消息表全局量从正确的 connect 字段播种（完成，见上），以及 (2) `NtUserMessageCall` 服务器回调路径穿过 WoW64 转换解析出一个有效客户端回调而不是 `0x2`。(2) 是真正的障碍，是它自己的 win32k 回调工程。

### 有用的面包屑

- 故障函数：user32 `DefWindowProcWorker` = `FUN_10029310`；故障指令 `user32+0x29378`。
- 全局量：`user32+0xccbc0`（上限）、`+0xccbc8`（位数组指针）、`+0xccbd0`/`+0xccbd8`（第二个表）。
- 游戏 wndproc：`iw4x.exe` `FUN_004731f0` (0x4731f0)。
- `NtUserMessageCall` 崩溃：`ntdll+0xd6f1c`，`call to 0x2`，栈经由 `wow64cpu.dll` + `user32+0x3b880`。
- 逐消息位表住在 `user_handle_table.hpp`（`WND_MESSAGE_BITS`，经由 `get_awm_control_message`/`get_def_window_messages`/`get_def_window_spec_messages` 读取）；`populate_user_shared_info` 为原生和 WoW64 connect 都把 `awmControl[]`/`DefWindowMsgs`/`DefWindowSpecMsgs` 指向它们。
- WHP 调试笔记：数据观察点（GDB stub 和 `hook_memory_read/write`）在 WHP 下**不**触发；只有 `hook_memory_execution(address, …)` 被强制执行。
