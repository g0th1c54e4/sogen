# GPU 半虚拟化现状

跟踪 issue：[#1032](https://github.com/momo5502/sogen/issues/1032)。工作位于 `gpu-bridge-device` 分支。

## 目标

给客体二进制真实的 GPU 图形支持，让会渲染的应用能在模拟器中运行和被分析。技术是 **GPU API 远程化**（又称 API 半虚拟化），作用在 **Vulkan** 边界：客体的 Vulkan 调用跨模拟器边界转发，在**宿主机真实 Vulkan 驱动**上执行。我们从不模拟 GPU 设备或解析专有的命令缓冲 —— 这正是"真正"的 GPU 虚拟化不可行的部分。

这与 Mesa 的 **Venus**（Vulkan over virtio-gpu）和 **virglrenderer** 是同一思路。

## 为什么这适合用户空间模拟器

Windows 图形栈有两个厂商组件：一个**用户态驱动 / ICD**（如 `nvoglv64.dll`）和 `dxgkrnl.sys` 背后的**内核态驱动**，后者构建厂商专有命令缓冲。因为 Sogen 只模拟**用户空间**（没有内核、没有 `dxgkrnl`、没有硬件），我们用**远程化 shim** 替换 UMD/ICD，并跳过整个内核/KMD/命令缓冲层。

DirectX/OpenGL 可以稍后在上面分层（DXVK 做 D3D→Vulkan，Zink 做 GL→Vulkan）而不需要新的桥工作。**DXVK 已大部分降级为低优先级**（重点是原生 Vulkan 应用），不过 DXVK 设备引导（bring-up）现在已初步可用 —— 见 M2 下的 "DXVK 设备引导" 切片。

## 架构

```
guest app  (or, later, DXVK d3d11.dll / dxgi.dll)
  -> guest vulkan-1.dll shim            (serialize Vulkan calls)            src/samples/vulkan-shim/
    -> CreateFile(\\.\SogenGpu) + DeviceIoControl   [emulator boundary]
      -> SogenGpu io_device              (dispatch IOCTL -> command)        devices/gpu_bridge.cpp
        -> vulkan_host                   (real vulkan-1.dll / libvulkan)    devices/vulkan_host.cpp
          -> host GPU
      <- opaque object ids / results / fence status
```

### 关键决策：虚拟驱动设备，而不是自定义系统调用

issue #1032 最初的草图用一个自定义的高编号系统调用。我们改为通过一个**虚拟驱动设备**远程化 —— 客体打开 `\\.\SogenGpu`（NT 路径 `\Device\SogenGpu`）并用 `NtDeviceIoControlFile` 与之通信。这复用了 Sogen 现有的 `io_device` 基础设施，并免费获得了系统调用路线本来要重新发明的一切：

| 关注点 | 驱动设备给我们的 |
| --- | --- |
| 分发管道 | `NtCreateFile` → `io_device_container` → `NtDeviceIoControlFile` → `io_control()` 已存在 |
| 客体 shim | 普通 `CreateFile` + `DeviceIoControl`；**没有 asm 系统调用桩**；在宿主机 DLL **和**根目录模式下都可用 |
| 命令 + 缓冲 | IOCTL 代码就是命令；输入/输出缓冲是内置的 |
| 轮询并让出 | `STATUS_PENDING` + `work()` 已被 AFD 套接字设备使用（`devices/afd_endpoint.cpp`） |
| 快照 | `io_device::serialize_object` 钩子已在设备容器中 |
| 分析钩子 | `callbacks.on_ioctrl(device, name, code)` 自动触发 |

更难设备行为（异步、轮询并让出）的蓝图是现有的 `devices/afd_endpoint.cpp`（套接字），它已经返回 `STATUS_PENDING` 并由模拟器主循环驱动的 `work()` 完成工作。

### 不透明对象 id

原始宿主机指针绝不跨边界。`vulkan_host` 把真实的 `VkInstance` / `VkPhysicalDevice` / `VkDevice` / … 保存在表中，交给客体一个不透明的 `object_id`（单调递增的 `uint64`）。**客体 shim 把 `object_id` 直接存进 Vulkan 句柄值** —— 所以 shim 不需要自己的句柄表；应用持有的值*就是*宿主机查找所用的 id。Vulkan 句柄有两种形态，shim 可移植地处理两种（见 `to_object_id`/`to_handle`）：*可分发*句柄（`VkInstance`、`VkDevice`、…）是指针 —— x64 上 64 位、32 位（WOW64）客体上 32 位 —— 所以 id 存在指针值里（id 是小的单调计数器，因此无损装进 32 位）；*不可分发*句柄（`VkBuffer`、`VkImage`、…）在每个平台上都是 `uint64_t`，携带完整 id。无论客体位数如何，id 跨线始终以 64 位值传输。

### 跨位宽可移植性（WOW64）

shim 和模拟器宿主机可以构建成不同的位数 —— 32 位（WOW64）客体对 64 位宿主机，或反之 —— 所以线上协议被刻意设计为**与指针大小无关**：`gpu_bridge_protocol.hpp` 中的每个结构只用定宽整数/浮点和 `object_id`（绝不用 `size_t` 或指针），MSVC 在 x86 和 x64 上布局一致（8 字节类型在两者上都保持 8 对齐）。一组 `static_assert` 钉住代表性结构的大小，使任何未来指针大小的成员至少在一种架构上构建失败。shim 通过 `.def` 导出（`src/samples/vulkan-shim/vulkan_shim.def`），使其入口点在 x86 上也保持**未修饰**的名字 —— `VKAPI_CALL` 是 `__stdcall`，否则会修饰它们（`_vkCreateInstance@16`），让客体的 `GetProcAddress` 找不到，正如真实的 `vulkan-1.dll` 用它们自己的 def 所避免的那样。已验证：32 位样本 + shim 通过其 WOW64 路径运行在 64 位模拟器（`analyzer.exe`）下 —— `vulkan-shim-test` 的 fill/clear 读回通过，窗口化样本能渲染。

### 线上协议

`src/gpu-bridge-protocol/gpu_bridge_protocol.hpp` 是一个零依赖头文件（只有 `<cstdint>`），是宿主机和客体共享的唯一事实来源：

- IOCTL 代码用标准 `CTL_CODE` 编码（FILE_DEVICE_UNKNOWN, METHOD_BUFFERED），每条命令一个，位于厂商函数范围（`0x800+`）。
- 每条命令有固定大小的请求/响应结构（`object_id`、计数、`VkResult` 为 `int32`）。
- 没有指针的原生 Vulkan 结构（如 `VkPhysicalDeviceProperties`、`VkQueueFamilyProperties`）以**原始字节**跨线 —— 两侧通过各自 `<vulkan/vulkan_core.h>` 就布局达成一致。这使协议头文件不含 `vulkan.h`。

对于**指针丰富**的结构（pNext 链、字符串数组、嵌套结构），原始字节不够，所以由生成的编组处理（见下文）。

## 状态 — 已完成

所有内容都对着真实 Vulkan 驱动端到端验证过 —— 开发机的真实 GPU（一块 Intel UHD 630 + 一块 Nvidia Quadro P2000）以及，在无 GPU 的机器上，放入作为软件 ICD 的 **SwiftShader**。窗口化样本还作为**原生** Vulkan 应用对真实 GPU 运行（加载 `vulkan-1.dll` 而不是 shim）。每个切片是 `gpu-bridge-device` 上的一个提交（或合并）；release 构建干净，`tidy`（clang-tidy）预设干净，且 `analyzer.exe -s test-sample.exe` 冒烟测试全程通过。

### M1 — 最小远程化（完成）

issue #1032 的 M1 目标是 `vkCreateInstance → vkEnumeratePhysicalDevices → vkCreateDevice`。已完成，外加实际使用它们所需的周边调用：

- **`3fa8096c` — 桥边界。** `SogenGpu` `io_device`；在 `create_device` / `get_io_device_name` 中注册，使 `\\.\SogenGpu` 可解析。`GET_VERSION` 握手（magic + 协议版本）。客体 `gpu-bridge-probe-sample` 通过 `DeviceIoControl` 做握手。
- **`bc1e2586` — 实例 + 物理设备查询。** 把 `Vulkan-Headers` 添加为浅 `deps/` 子模块，通过轻量 `vulkan-headers` INTERFACE 目标暴露。`vulkan_host` 动态加载宿主机加载器（`vulkan-1.dll` / `libvulkan.so`）并远程化 `vkCreateInstance`、`vkEnumeratePhysicalDevices`、`vkGetPhysicalDeviceProperties`、`vkDestroyInstance`。探针枚举出真实宿主机 GPU。
- **`8654f8ce` — 客体 `vulkan-1.dll` shim。** `src/samples/vulkan-shim/` 导出 Vulkan 入口点并转发到桥；应用像加载真实 ICD 一样通过 `LoadLibrary` + `vkGetInstanceProcAddr` 加载它。`vulkan-shim-test` 驱动它走真实的 Vulkan 流程。
- **`1fe7e75d` — 逻辑设备 + 队列。** `vkGetPhysicalDeviceQueueFamilyProperties`、`vkCreateDevice`、`vkGetDeviceQueue`、`vkDestroyDevice`。在宿主机 GPU 上创建真实 `VkDevice` + 图形队列。
- **`053944d1`** — 上述内容的 clang-tidy 清理。

### M2 — 提交、同步、资源和呈现（进行中）

- **`08cde6de` — 命令提交 + fence 同步（轮询并让出）。** 命令池/缓冲 + fence 对象模型、`vkQueueSubmit`，以及阻塞调用的解决方案：宿主机端点只做**非阻塞 `vkGetFenceStatus`**，客体 shim 的 `vkWaitForFences` 一边轮询一边让出给模拟器（`SwitchToThread` → `NtYieldExecution`）。真实 GPU 独立于模拟器线程推进，所以 fence 最终会信号 —— 单线程模拟器绝不会在 GPU 上阻塞。已验证：录制一个空命令缓冲，带着 fence 提交并等待（`vkQueueSubmit` 和 `vkWaitForFences` 都返回 `VK_SUCCESS`）。

- **`59103a59` — vk.xml 编组生成器（代码生成基础）。** 手写编组无法扩展到 Vulkan 指针丰富的结构，所以引入了一个 Venus 风格生成器：
  - `src/tools/vulkan-bridge-generator/generate.py` 解析 `vk.xml`，为一个白名单结构发射具体的 `encode`/`decode` 重载到一个检入的头文件。
  - `src/vulkan-bridge-marshal/` 是运行时：基于字节流的 `writer`/`reader`，外加一个 `arena` 持有解码出的被指向对象（这样宿主机可以把重建的结构传给真实驱动）。`encode`（客体）展平；`decode`（宿主机）重建；两者来自同一个生成的头文件所以不会漂移。
  - `vulkan-bridge-generate` CMake 目标离线重新生成（只有*重新生成*才需要 Python，构建从不需要）；已验证它逐字节复现检入的头文件。
  - 往返 gtest（`src/windows-emulator-test/vulkan_marshal_test.cpp`）覆盖标量、以 null 结尾的字符串、`len="count,null-terminated"` 字符串数组、嵌套结构指针，载体为 `VkApplicationInfo` 和 `VkInstanceCreateInfo`。

- **设备内存 + host-visible 缓冲 + 读回。** 第一个让 GPU*产出客体读回的数据*的切片 —— 所有渲染的基础。增加 `vkGetPhysicalDeviceMemoryProperties`、`vkAllocateMemory`/`vkFreeMemory`、`vkCreateBuffer`/`vkDestroyBuffer` + `vkGetBufferMemoryRequirements`/`vkBindBufferMemory`、一条真实录制命令（`vkCmdFillBuffer`）以及内存读回。因为 `VkDeviceMemory` 在宿主机侧，客体的 `vkMapMemory`/`vkUnmapMemory` 通过**暂存一份客体侧副本**来模拟：`download_memory` 在 map 时填充暂存缓冲，`upload_memory` 在 unmap 时刷回，因此宿主机指针永不跨桥（零拷贝映射是后续优化）。端到端验证：客体分配一个 host-visible 缓冲，GPU 通过 `vkCmdFillBuffer` 填充它，映射读回返回精确的模式（`vulkan-shim-test` 的 `fill+readback` 检查）。

- **图像 + clear + 图像→缓冲读回。** 离屏渲染目标 + 读回路径，窗口化 present 会复用它。增加 `vkCreateImage`/`vkDestroyImage` + `vkGetImageMemoryRequirements`/`vkBindImageMemory`、图像布局转换（`vkCmdPipelineBarrier`，仅限图像内存屏障）、`vkCmdClearColorImage` 和 `vkCmdCopyImageToBuffer`。端到端验证：GPU 把一个 16×16 `R8G8B8A8_UNORM` 图像 clear 成已知颜色，转换 `UNDEFINED → TRANSFER_DST → TRANSFER_SRC`，拷贝进 host-visible 缓冲，读回逐像素匹配（`vulkan-shim-test` 的 `clear+readback` 检查）。

- **WSI：向客体窗口窗口化呈现。** 第一个*可见的*里程碑 —— 一个客体 Vulkan 应用打开真实 Win32 窗口并显示其 GPU 渲染内容。增加 `vkCreateWin32SurfaceKHR`、`vkDestroySurfaceKHR`、`vkCreateSwapchainKHR`、`vkDestroySwapchainKHR`、`vkGetSwapchainImagesKHR`、`vkAcquireNextImageKHR` 和 `vkQueuePresentKHR`。**没有真实宿主机 WSI**：表面只是客体 HWND，交换链是 N 个离屏图像加一个 host-visible 读回缓冲。客体保持忠实的 WSI 应用（转换到 `PRESENT_SRC_KHR`，宿主机映射为 `TRANSFER_SRC_OPTIMAL` 以保持真实驱动有效）。在 `vkQueuePresentKHR` 上，宿主机把呈现图像拷入读回缓冲，等待，然后桥通过 `win_emu.ui().present_surface(hwnd, …)` 把 BGRA 像素交给客体窗口 —— 与 GDI `EndPaint` 用的同一个 UI 后端接缝。新客体样本 `vulkan-window-sample` 创建 320×240 窗口，每帧把交换链图像 clear 成动画颜色并呈现；窗口显示实时 GPU 输出（在 SwiftShader 上屏幕验证）。

- **图形管线：窗口中的三角形。** 第一个*真实渲染*（不只是传输 clear）。增加 `vkCreateShaderModule`、`vkCreateImageView`、`vkCreateRenderPass`、`vkCreateFramebuffer`、`vkCreatePipelineLayout`、`vkCreateGraphicsPipelines`（+ 销毁），以及绘制录制 `vkCmdBeginRenderPass` / `vkCmdBindPipeline` / `vkCmdDraw` / `vkCmdEndRenderPass`。宿主机硬编码大部分固定功能状态（无顶点输入、三角形列表、静态全尺寸视口/裁剪、一个无混合颜色附件）；客体提供 SPIR-V、渲染通道附件和视口尺寸。`vulkan-window-sample` 现在通过渲染通道把经典的 RGB 渐变三角形渲染进交换链图像并呈现 —— 在 SwiftShader 上屏幕验证。着色器 SPIR-V 离线编译（glslang）并检入为 `triangle_spirv.hxx`，所以构建不需要着色器编译器。

- **旋转立方体样本 + 宿主机 vs 模拟帧率基准。** 一个稍微丰富一点的样本，更狠地压测现有入口点：一个实体 3D 立方体（6 面 36 个烘焙顶点），由 CPU 计算的 64 字节 mat4 MVP push constant 变换。桥中没有深度附件和剔除，六个面每帧由 CPU 从后往前排序（画家算法）并用逐面 `vkCmdDraw` 调用绘制（`firstVertex = face*6`）。旋转由真实墙钟时间驱动（不是帧数），因此原生和模拟下以相同物理速率旋转，循环每真实秒向 stdout 打印一次 FPS，使同二进制的吞吐量能对比 shim 路径 vs 真实 `vulkan-1.dll`。无需新的桥/shim 工作。

- **Push constant + 旋转三角形样本。** 增加 `vkCmdPushConstants` 和管线布局上的 push constant 范围 —— 足够着色器无需顶点/均匀缓冲即可读取小的逐帧数据。新客体样本 `vulkan-spinning-triangle-sample` 用 push constant 角度旋转三角形并在窗口标题显示帧率。旋转角度按*帧*推进（而不是按墙钟秒），因为模拟的 tick 计数器不跨宿主机侧阻塞跟踪真实时间，且 FPS 从系统时间（`GetSystemTimeAsFileTime`）测量而非 `GetTickCount`。

- **顶点 + 索引缓冲 + 索引绘制。** 第一个几何来自真实缓冲而不是烘焙进顶点着色器（`gl_VertexIndex`）。`create_graphics_pipeline` 现在携带可变长度**顶点输入状态**（绑定 + 属性描述跟在请求后面，如同 SPIR-V 跟在 `create_shader_module` 后面），所以线上协议提升到 `protocol_version = 2`。增加绘制时命令 `vkCmdBindVertexBuffers`、`vkCmdBindIndexBuffer` 和 `vkCmdDrawIndexed`（全部像其他 `vkCmd*` 一样录制进批量命令流）。无需新的内存管道 —— 顶点/索引缓冲只是带 `VERTEX_BUFFER`/`INDEX_BUFFER` 用法的普通 `vkCreateBuffer`，通过现有 host-visible `vkMapMemory` 填充。空顶点输入（计数为 0）是先前行为，所以旧的着色器烘焙样本不变。新客体样本 `vulkan-vertex-buffer-sample` 上传一个 4 顶点 / 6 索引四边形（交错的 `vec2` 位置 + `vec3` 颜色），在其管线中声明顶点输入，用 `vkCmdDrawIndexed` 绘制 —— 在 SwiftShader 上和作为原生应用屏幕验证。

- **统一缓冲 + 描述符集。** 着色器现在可以通过绑定到描述符集的 uniform buffer 读取数据，而不只是 push constant。增加 `vkCreateDescriptorSetLayout`、`vkCreateDescriptorPool`、`vkAllocateDescriptorSets`、`vkUpdateDescriptorSets`（+ 销毁）以及录制的 `vkCmdBindDescriptorSets`；`vkCreatePipelineLayout` 现在携带描述符集布局的可变长度列表。`vkUpdateDescriptorSets` 每次写入建模一个描述符（缓冲或以后的组合图像采样器），描述符拷贝未建模；描述符池以 `FREE_DESCRIPTOR_SET` 创建。新客体样本 `vulkan-uniform-buffer-sample` 绘制顶点/索引缓冲四边形，但旋转角度和颜色着色来自 CPU 每帧重写的 `vec4` uniform buffer —— 在 SwiftShader 上和作为原生应用屏幕验证。

- **纹理（采样图像）。** 着色器现在可以采样纹理。增加 `vkCreateSampler` / `vkDestroySampler` 和录制的 `vkCmdCopyBufferToImage`（宿主机侧，组合图像采样器描述符写入已在描述符集切片中接线）。纹理是带 `SAMPLED | TRANSFER_DST` 用法的普通 `vkCreateImage`，从 host-visible 暂存缓冲通过 `vkCmdCopyBufferToImage` 上传，伴随通常的 `UNDEFINED → TRANSFER_DST → SHADER_READ_ONLY` 布局转换，然后通过绑定为组合图像采样器的 `VkImageView` + `VkSampler` 读取。拷贝是紧密打包的（mip 0 / 层 0，无行填充）。新客体样本 `vulkan-texture-sample` 上传 64×64 RGBA 棋盘格并把它采样到旋转的顶点/索引缓冲四边形上 —— 在 SwiftShader 上和作为原生应用屏幕验证。这个样本一次性触及整个绘制栈：顶点 + 索引缓冲、push constant、描述符集和采样纹理。

- **深度缓冲（正确的 3D 遮挡）。** 渲染通道现在可以有深度附件。现有 create-info 入口点增加可选深度字段（全部向后兼容 —— 关闭深度是先前行为）：`vkCreateRenderPass` 在子通道有 `pDepthStencilAttachment` 时增加深度附件；`vkCreateFramebuffer` 接受第二个（深度）附件；`vkCreateImageView` 遵循 `aspectMask`（因此 `DEPTH` 视图可用，不只是 `COLOR`）；`vkCreateGraphicsPipelines` 应用 `VkPipelineDepthStencilState`（深度测试/写入 + 比较操作）；`vkCmdBeginRenderPass` 携带深度 clear 值（仅当渲染通道有深度时使用）。深度图像是带 `DEPTH_STENCIL_ATTACHMENT` 用法的普通 `vkCreateImage`。新客体样本 `vulkan-depth-cube-sample` 绘制真实 3D 立方体（顶点/索引缓冲 + mat4 MVP push constant），通过深度缓冲正确遮挡，**无需 CPU 面排序** —— 不同于早于深度支持的 `vulkan-cube-sample`（它画家算法排序其面）。在 SwiftShader 上和作为原生应用屏幕验证。

- **在真实驱动上运行样本（不只是桥）。** 每个 Vulkan 样本同时是普通原生 Vulkan 应用：加载真实 `vulkan-1.dll` 而不是 shim（`… .exe vulkan-1.dll`），它就对着真实 GPU 运行。这是对 shim API 保真度的强正确性检查 —— *同一个*二进制既通过桥又对着真实加载器运行 —— 它暴露（并修复）了桥掩盖的几个真实 WSI 要求：
  - **`VK_SUBOPTIMAL_KHR`** 是真实驱动从 acquire/present 返回的成功码；样本把它当作非致命而不是中止。
  - **逐帧 fence 同步** —— 每帧在呈现前等待 fence，而不是依赖桥的隐式 `vkQueueWaitIdle`（真实驱动不会替你串行化帧）。
  - **交换链尺寸匹配** —— 远程化 `vkGetPhysicalDeviceSurfaceCapabilitiesKHR`，样本把交换链/视口/帧缓冲/渲染区域对齐到 `caps.currentExtent`（以*物理*像素表示的表面客户区）。硬编码窗口尺寸会让真实驱动以 `VK_ERROR_OUT_OF_DATE_KHR` 拒绝 present（客户区 < 窗口，可能被 DPI 缩放）。桥返回未定义的 `currentExtent`，此时客体回退到窗口客户矩形。样本窗口是固定尺寸的，因为交换链不在调整大小时重建。
  - 旋转样本用 `VK_PRESENT_MODE_IMMEDIATE_KHR` 呈现（无垂直同步），使其 FPS 反映真实吞吐而非刷新率（桥忽略呈现模式）。

  > 构建 FPS 读数还顺带发现一个无关的模拟器 bug：`GetTickCount[64]` 快了约 10000 倍（`kusd_mmio::update()` 从 100 ns 单位推导 tick 计数，而客体期望毫秒）。已在 [#1037](https://github.com/momo5502/sogen/pull/1037) 单独修复。

- **DXVK 设备引导：真实特性/扩展远程化。** 朝运行真实 D3D9 游戏的第一步（通过 `open-iw5.exe` 运行的《使命召唤 MW3》，其 `d3d9.dll` 是经过 shim 化的 `vulkan-1.dll` 之上的 DXVK 2.7.1）。DXVK 只有在桥报告宿主机驱动*真实*能力时才会选择并创建设备，所以三个入口点从桩提升为真实远程化（协议提升到 `protocol_version = 4`）：
  - `vkEnumerateDeviceExtensionProperties` —— 转发到宿主机设备的真实扩展列表。
  - `vkGetPhysicalDeviceFeatures2` —— 为调用者的 pNext 特性链查询宿主机并用真实值填充。新的零依赖相邻头文件 `src/gpu-bridge-protocol/vk_feature_chain.hpp` 是编组 `VkPhysicalDevice*Features` pNext 链的单一、**跨位宽**事实来源：`VkPhysicalDevice*Features` 结构是 `{sType, pNext}` 头部（x86 上 8 字节，x64 上 16 字节）后跟 `VkBool32` 序列，所以只有无填充的主体跨线 —— 32 位客体计算主体大小，64 位宿主机回显。增加一个新需要的特性结构是一行 `sType → sizeof` 的 switch 条目（目前的集合：核心 1.1/1.2/1.3、`VK_EXT_robustness2`、`VK_KHR_maintenance5`）。shim 还对未实现的入口点按 Vulkan 契约返回 **null**，使层能检测缺失的可选特性，并逐个记录。
  - `vkCreateDevice` —— 编组已启用扩展名 + 特性链，使真实设备带着 DXVK 所请求的精确配置被创建（此前被忽略，一旦 DXVK 用到这些特性就会成为 UB）。
  - 结果（无屏运行验证）：DXVK 枚举宿主机 GPU，**跳过缺少必需特性的设备**（Intel UHD 630 缺 `maintenance5`）并**在 Quadro P2000 上创建设备**，然后设置 D3D9 格式。`vulkan-shim-test` 和窗口化样本仍通过（`vkCreateDevice`/特性路径也在它们的路径上）。完整 DXVK 渲染（交换链 + DXVK 的大型运行时设备函数集）仍在前方；DXVK 其余部分仍为低优先级。
- **macOS / MoltenVK 宿主机引导（可移植性驱动）。** 宿主机此前只在原生 ICD（Linux）或 SwiftShader 上运行过；在 Apple Silicon Mac 上，宿主机 `vulkan-1.dll`/`libvulkan.dylib` 解析到 **MoltenVK** 可移植性驱动（Vulkan over Metal），Khronos 加载器拒绝使用它，除非调用方选择加入。两个可移植性要求被加入 `vulkan_host`：
  - `create_instance` 现在在加载器广告该实例扩展时启用 `VK_KHR_portability_enumeration` 并设置 `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR`；没有它 `vkCreateInstance` 返回 `VK_ERROR_INCOMPATIBLE_DRIVER`，永远枚举不到物理设备。该扩展由加载器而非驱动实现，所以它也可能出现在 Linux 加载器上 —— 但在任何地方启用都是无害的：它只是从枚举中解除隐藏非合规设备，从不谎报设备能力。
  - `create_device` 现在在物理设备广告它时追加 `VK_KHR_portability_subset`（Vulkan 要求只要设备广告就必须启用）；客体从不要求它。设备是否广告它检查一次并按物理设备缓存。
  在 macOS 上对真实 GPU 端到端验证：客体 `vulkan-shim-test` 通过 MoltenVK 枚举宿主机 **Apple M5 Pro**，其 `fill+readback`（`0xDEADBEEF`）和 `clear+readback`（`0xFF0000FF`）检查都通过 —— GPU 产出像素且客体精确读回。这运行在 Unicorn CPU 后端下（桥与 CPU 后端无关），且实际走了直接别名 `vkMapMemory` 路径（宿主机分配按页舍入，MoltenVK 返回页对齐映射指针，因此整个 `VkDeviceMemory` 直接别名进客体 VA）。在以下环境用 homebrew 加载器 + MoltenVK ICD 运行：`DYLD_LIBRARY_PATH=/opt/homebrew/lib VK_ICD_FILENAMES=/opt/homebrew/etc/vulkan/icd.d/MoltenVK_icd.json`。

- **可移植性设备上的 DXVK 特性欺骗。** DXVK 的适配器过滤器拒绝任何缺少其统一 D3D8/9/10/11 基线的 Vulkan 设备：`geometryShader` 和 `shaderCullDistance` 核心特性、`VK_EXT_depth_clip_enable`（+ 其 `depthClipEnable` 特性），以及 `VK_EXT_robustness2` 的 `robustBufferAccess2`/`nullDescriptor` 特性。MoltenVK 真实支持这些中的任何一个都不行（Apple GPU 没有几何着色器级或着色器剔除距离；MoltenVK 也没实现 depth-clip-enable 或完整 robustness2），所以不干预的话 DXVK 会拒绝 macOS 上唯一的适配器，完全无法 D3D9 渲染。D3D9 本身不需要这些能力中的任何一个 —— 该过滤器只是 DXVK 实现的所有 D3D 版本共享。

  因此 `vulkan_host` 在*查询*时撒谎，并在*设备创建*时撤回谎言：
  - `vkGetPhysicalDeviceFeatures2` 报告 `geometryShader`、`shaderCullDistance`、`depthClipEnable`、`robustBufferAccess2` 和 `nullDescriptor` 为受支持，`vkEnumerateDeviceExtensionProperties` 注入一条合成的 `VK_EXT_depth_clip_enable` 条目，使 DXVK 的过滤器通过。
  - `vkCreateDevice` 把请求的特性集掩码削减回设备真实支持的范围（重新查询相同的 pNext 链形状），并除非设备真正实现否则从已启用扩展中剥掉 `VK_EXT_depth_clip_enable`，因此 MoltenVK 永远不会被要求启用它无法提供的能力，设备创建成功。

  DXVK 观察不到撤回 —— 它相信查询并走正常代码路径 —— 这是安全的，因为 D3D9 不触及任何被欺骗的能力。没有任何 DXVK 配置能放松其必需特性集，MoltenVK 也长不出几何级，所以桥是唯一能解决该不匹配的地方。三个已知限制：
  - 欺骗不限定于 DXVK：可移植性设备上每个客体 Vulkan 客户端都会被告知这五个能力存在。一个真正使用其中之一（几何着色器、剔除距离，或通过 vkd3d 的 D3D12 游戏）的客户端会在管线创建时失败，而不是在适配器选择时干净地跳过设备。
  - 深度裁剪保持 Vulkan 默认，与 D3D9 的默认状态匹配，因此显式关闭 D3D9 深度裁剪的游戏会静默渲染错误。
  - DXVK 对未绑定资源的 `VK_NULL_HANDLE` 描述符绑定依赖 MoltenVK/Metal 在没有正式 `nullDescriptor` 特性时实际上容忍空描述符 —— 经验行为，而非保证契约。

  这一切都被门控在**可移植性设备** —— 广告 `VK_KHR_portability_subset` 的设备，这是 Vulkan 规范自己对不合规设备的定义；合规的原生驱动从不广告它。真正缺其中某个特性的原生 Linux/Windows 驱动（如没有 `geometryShader` 的软件光栅器）看到的是其真实能力，且客体所请求的会原样到达它 —— `vkCreateDevice` 在那里会恰好在没有 shim 时也会失败的地方失败。

## 已远程化的 Vulkan 入口点

实例/设备：`vkCreateInstance`、`vkDestroyInstance`、`vkEnumeratePhysicalDevices`、`vkEnumerateDeviceExtensionProperties`、`vkGetPhysicalDeviceProperties`、`vkGetPhysicalDeviceFeatures2`（真实 pNext 特性链）、`vkGetPhysicalDeviceQueueFamilyProperties`、`vkCreateDevice`（线程已启用扩展 + 特性链）、`vkDestroyDevice`、`vkGetDeviceQueue`、`vkGetInstanceProcAddr`、`vkGetDeviceProcAddr`（+ `vk_icdGetInstanceProcAddr`）。

命令/同步：`vkCreateCommandPool`、`vkDestroyCommandPool`、`vkAllocateCommandBuffers`、`vkFreeCommandBuffers`、`vkBeginCommandBuffer`、`vkEndCommandBuffer`、`vkCreateFence`、`vkDestroyFence`、`vkResetFences`、`vkGetFenceStatus`、`vkQueueSubmit`、`vkWaitForFences`。

内存/缓冲：`vkGetPhysicalDeviceMemoryProperties`、`vkAllocateMemory`、`vkFreeMemory`、`vkMapMemory`、`vkUnmapMemory`、`vkCreateBuffer`、`vkDestroyBuffer`、`vkGetBufferMemoryRequirements`、`vkBindBufferMemory`、`vkCmdFillBuffer`。

图像/传输：`vkCreateImage`、`vkDestroyImage`、`vkGetImageMemoryRequirements`、`vkBindImageMemory`、`vkCmdPipelineBarrier`、`vkCmdClearColorImage`、`vkCmdCopyImageToBuffer`、`vkCmdCopyBufferToImage`、`vkCreateSampler`、`vkDestroySampler`。

WSI：`vkCreateWin32SurfaceKHR`、`vkDestroySurfaceKHR`、`vkGetPhysicalDeviceSurfaceCapabilitiesKHR`、`vkCreateSwapchainKHR`、`vkDestroySwapchainKHR`、`vkGetSwapchainImagesKHR`、`vkAcquireNextImageKHR`、`vkQueuePresentKHR`。

图形管线：`vkCreateShaderModule`、`vkCreateImageView`（颜色或深度方面）、`vkCreateRenderPass`（颜色 + 可选深度附件）、`vkCreateFramebuffer`、`vkCreatePipelineLayout`（带 push constant 范围和描述符集布局）、`vkCreateGraphicsPipelines`（顶点输入状态 + 可选深度测试）（+ 销毁）、`vkCmdBeginRenderPass`、`vkCmdBindPipeline`、`vkCmdPushConstants`、`vkCmdBindVertexBuffers`、`vkCmdBindIndexBuffer`、`vkCmdDraw`、`vkCmdDrawIndexed`、`vkCmdEndRenderPass`。

描述符：`vkCreateDescriptorSetLayout`、`vkCreateDescriptorPool`、`vkAllocateDescriptorSets`、`vkUpdateDescriptorSets`（uniform-buffer 和组合图像采样器写入）（+ 销毁）、`vkCmdBindDescriptorSets`。

大多数手写入口点仍传最小/空的 create-info（如 `vkCreateInstance` 忽略层/扩展，`vkQueueSubmit` 忽略信号量）。`vkCreateDevice` 是例外 —— 它现在遵循已启用设备扩展和 pNext 特性链（见 DXVK 设备引导切片）。生成器是处处遵循完整结构的路径。

## 已知限制 / 简化

- **活跃 GPU 状态无快照支持。** `gpu_bridge_device` 序列化为 no-op；宿主机 Vulkan 对象无法序列化。在打开 GPU 句柄时拍摄快照的恢复不受支持（实验性）。
- **`pNext` 链被生成的编组丢弃**（encode 写"无链"，decode 产出 `nullptr`）。扩展结构需要 sType 分发链遍历。
- **被编组结构内部的句柄成员尚未处理** —— 它们需要客体↔宿主机 `object_id` 转换钩子。当前白名单（`VkApplicationInfo`、`VkInstanceCreateInfo`）没有句柄。
- 生成器尚不支持：非字符串结构数组、联合体、固定内联数组。
- **`vkQueueSubmit`** 每次提交远程化单个命令缓冲，无等待/信号信号量（fence 挂在最后一个命令缓冲上）。
- **`vkCreateDevice`** 从应用第一个 `VkDeviceQueueCreateInfo` 创建单个队列族，无设备扩展/特性。
- **`vkMapMemory` 是暂存拷贝，不是零拷贝。** map 时宿主机范围被下载进客体侧缓冲；unmap 时上传回。读写正确，但每次 map 一次宿主机↔客体拷贝。零拷贝（把宿主机 GPU 可见内存映射进客体 VA）是后续优化。
- **呈现基于读回（无真实交换链），现在是异步的（落后一帧）。** 每个 `vkQueuePresentKHR` 把一次图像→缓冲拷贝录进 host-visible 读回缓冲；桥随后把这些像素交给 UI 后端。未建模真实呈现信号量/获取 fence（`vkAcquireNextImageKHR` 忽略它们并轮询发放图像），且 `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR` 在宿主机上映射为 `TRANSFER_SRC_OPTIMAL`。
  - 拷贝**已提交但不内联等待**：它在客体渲染下一帧时在 GPU 上运行，而*下一次* present 排空它（其 fence 到那时基本总是已信号）。因此呈现落后一帧（无感）且宿主机线程从不在 `vkQueueWaitIdle` 上阻塞。读回缓冲分配为 `HOST_CACHED`（不只是 `HOST_COHERENT`），因此约 920 KB 的 CPU 读回快一个数量级。
  - *实测（WHP/Hyper-V 后端，开发机，`vulkan-cube-sample` 内置逐阶段分析器）。* 同步读回呈现最初主导了模拟帧：约 3.4 ms/帧（约 290 FPS），其中呈现约 2.7 ms（≈79%）—— 宿主机侧拆分为约 0.88 ms `vkQueueWaitIdle` + 约 1.25 ms 的 920 KB 读回 memcpy（缓冲是未缓存的）+ 约 0.45 ms SDL 上传。两个修复：`HOST_CACHED` 读回把 memcpy 从约 1.25 ms 砍到约 0.065 ms（→ 约 450 FPS），异步呈现移除了内联 `vkQueueWaitIdle`（→ 约 650 FPS，约 1.54 ms/帧）。净 **约 290 → 约 650 FPS（约 2.2 倍）**，对比原生约 0.21 ms。边界成本是真实的，但在这个命令数量级是次要的（WHP 下每次远程化调用约 7.5 µs）；现在它还被**批量化了** —— 整个命令缓冲的录制一次 IOCTL 跨线（见下文命令批量），因此立方体的录制从约 0.18 ms（12 个 IOCTL）降到约 0.06 ms（1 个）。剩余的大阶段现在是客体的每帧 fence 等待（约 0.6 ms，轮询并让出）和 SDL `present_surface` 上传（约 0.45 ms）；完全消除逐帧 GPU→CPU→GPU 往返（零拷贝 / GPU 侧呈现）是下一个杠杆。
- **命令录制被批量化（`ioctl_record_commands`）。** 命令缓冲录制命令（`vkBeginCommandBuffer`、各 `vkCmd*` 调用、`vkEndCommandBuffer`）不再每个一个 IOCTL。客体 shim 把每条追加到每命令缓冲字节流（一个 `command_record_header` + 该命令的正常请求结构作为载荷），并在 `vkEndCommandBuffer` 把整个 `begin → cmds → end` 流**单次** IOCTL 刷给桥；宿主机重放它（`execute_recorded_command`）。这使录制成本约为 O(1) 次边界穿越，与命令数无关 —— 对每帧发出数百/数千次绘制的真实应用必不可少（每次一个 VM 退出会主导）。600 次绘制/帧实测：批量时录制保持约 0.15 ms，而未批量估计约 4.5 ms（600 × 约 7.5 µs）。每命令 IOCTL 及其宿主机处理程序已移除；请求结构保留作为流载荷格式。
- **表面能力是合成的，交换链从不重建。** `vkGetPhysicalDeviceSurfaceCapabilitiesKHR` 返回固定 caps，`currentExtent` *未定义*（客体选择尺寸）且 min/max 宽松；`vkCreateSwapchainKHR` 请求的模式和 pre-transform 并未真正应用（读回路径是唯一呈现路径）。桥不实现交换链重建，所以 `VK_ERROR_OUT_OF_DATE_KHR`/调整大小未处理 —— 样本窗口固定尺寸以避免。表面格式和呈现模式的*枚举*也未远程化；样本直接传有效值（`B8G8R8A8_UNORM`、FIFO/immediate）。
- 宿主机驱动是宿主机解析到的任何 `vulkan-1.dll`。开发机有真实 ICD；无 GPU 的机器（如 CI）可以通过把加载器 + ICD 暂存（stage）到 `analyzer.exe` 旁边并指 `VK_ICD_FILENAMES` 到清单来放入 SwiftShader 作为软件回退。软件驱动在第一次 `vkQueueSubmit` 时 JIT 编译其提交路径，所以第一次 `vkWaitForFences` 需要宽裕的超时。

## 剩余工作（计划）

近期目标 —— 一个**窗口化** Vulkan 样本打开真实窗口并显示其 GPU 渲染内容 —— 已**达成**：`vulkan-window-sample` 通过图形管线把三角形渲染进客体窗口，`vulkan-spinning-triangle-sample` 增加了 push constant 旋转的三角形和 FPS 读数（下面的步骤 1–4 全部完成）。两个样本也作为普通原生 Vulkan 应用对真实 GPU 运行（加载 `vulkan-1.dll` 而不是 shim），验证了 shim 的 API 保真度。方法是**读回并呈现** —— 在宿主机 GPU 上渲染进离屏图像，把像素读回，并通过 Sogen 现有的 `present_surface` 接缝交给客体窗口的表面（SDL 后端已显示逐 HWND 的 BGRA 表面；见 `windows-ui-emulation.md`）。客体保持真实 Vulkan WSI 应用（`vkCreateWin32SurfaceKHR` / `vkCreateSwapchainKHR` / `vkQueuePresentKHR`），但桥把"交换链"实现为普通离屏 `VkImage`。这避免了零拷贝内存映射，避免了把宿主机 GPU 绑定到 SDL 窗口，并且意味着没有 Win32 WSI 的软件驱动（SwiftShader）仍可用。

通往该目标的阶段步骤：

1. **内存 + 读回基础** —— *done*（host-visible 缓冲、`vkCmdFillBuffer`、map 读回）。
2. **渲染目标 + clear** —— *done*（`VkImage` + 内存、布局屏障、`vkCmdClearColorImage`、`vkCmdCopyImageToBuffer` 读回对着已知 clear 颜色验证）。
3. **WSI → 第一个可见窗口** —— *done*（`vkCreateWin32SurfaceKHR` + 交换链 + `vkQueuePresentKHR`；呈现 = 读回 + `present_surface(hwnd, …)`；`vulkan-window-sample` 在屏幕上显示动画颜色）。
4. **三角形** —— *done*（渲染通道、帧缓冲、SPIR-V 着色器模块、图形管线、`vkCmdDraw`；`vulkan-window-sample` 把 RGB 渐变三角形渲染进窗口）。

横切（随着 create-info 变丰富而需要，可与上面的步骤并行落地）：

- **生成器：句柄转换** —— 句柄成员的 `object_id`↔宿主机句柄转换，使指针丰富的结构可以生成而不是手写编组。
- **生成器：`pNext` sType 分发** —— 遍历/重建白名单上的扩展链。
- **把手写编组命令迁移到生成的路径**，然后**逐命令生成**（完整的 Venus 模型）。
- **零拷贝 `vkMapMemory`** —— 把宿主机 GPU 可见内存映射进客体 VA（Sogen 用于 `KUSER_SHARED_DATA` 的机制），替换暂存拷贝 map。
- **更完整的 WSI** —— `VK_ERROR_OUT_OF_DATE_KHR`/调整大小时重建交换链、真实呈现和获取信号量、表面格式/呈现模式枚举。（今天样本固定尺寸并直接传有效交换链参数。）
- **绘制管道已就位** —— 顶点 + 索引缓冲 + `vkCmdDrawIndexed`、uniform buffer + 描述符集、采样纹理和深度缓冲都**完成**（见上面的 M2 切片）。剩余绘制细化是更丰富的固定功能/管线状态（模板、混合、多个颜色附件、MSAA、动态状态）以及真实应用需要的多/数组描述符。

更晚：通过 Zink 的 OpenGL、通过 DXVK 的 DirectX —— 无需新的桥工作，只需提供客体 DLL。

## 文件地图

| 路径 | 作用 |
| --- | --- |
| `src/gpu-bridge-protocol/gpu_bridge_protocol.hpp` | 零依赖线上协议（IOCTL 代码、请求/响应结构） |
| `src/gpu-bridge-protocol/vk_feature_chain.hpp` | 共享的跨位宽 `sType → sizeof` 映射 + 编组 `VkPhysicalDevice*Features` pNext 链的头部大小（仅宿主机 + shim；需要 Vulkan 头文件） |
| `src/windows-emulator/devices/gpu_bridge.{hpp,cpp}` | `SogenGpu` io_device；IOCTL → 命令分发 + 编组 |
| `src/windows-emulator/devices/vulkan_host.{hpp,cpp}` | 对真实驱动的无模拟器包装；对象 id 表。保持在自己的编译单元中，使宿主机 `<vulkan/vulkan_core.h>` + `<Windows.h>` 不与模拟的 Windows 类型冲突 |
| `src/vulkan-bridge-marshal/vk_bridge_serial.hpp` | 序列化运行时：`writer` / `reader` / `arena` + 字符串/数组辅助 |
| `src/vulkan-bridge-marshal/vk_bridge_marshal.generated.hxx` | **生成的** encode/decode（不要手改） |
| `src/tools/vulkan-bridge-generator/generate.py` | 生成器（解析 `vk.xml`）；`STRUCT_ALLOWLIST` 控制覆盖 |
| `src/samples/vulkan-shim/` | 客体 `vulkan-1.dll` shim（交付物） |
| `src/samples/vulkan-shim-test/` | 无头客体 exe，驱动 shim（实例→设备→fill/clear 读回） |
| `src/windows-emulator-test/vulkan_marshal_test.cpp` | 生成编组的往返 gtest |
| `deps/Vulkan-Headers` | 浅子模块；`vulkan-headers` INTERFACE 目标 |

注册点：`io_device.cpp`（`create_device`）和 `syscalls/file.cpp`（`get_io_device_name`）中的设备名；`src/CMakeLists.txt` 中的库；`deps/CMakeLists.txt` 中的 `vulkan-headers` 目标。

驱动开发的窗口化演示样本（`vulkan-window-sample`、立方体/三角形/缓冲/纹理/深度样本和 `gpu-bridge-probe-sample`）仅用于测试且已被移除；可从 git 历史恢复。`vulkan-shim-test` 保留作为无头回归检查。

## 构建、重新生成、验证

构建（release）：

```cmd
cmake --build --preset=release
```

从 `vk.xml` 重新生成编组（只在更改 `STRUCT_ALLOWLIST` 或升级 `Vulkan-Headers` 子模块之后）：

```cmd
cmake --build --preset=release --target vulkan-bridge-generate
```

运行 shim 测试（客体 app → `vulkan-shim.dll` → 桥 → 宿主机 GPU；枚举设备、创建逻辑设备 + 队列、提交命令缓冲、验证 `fill+readback` / `clear+readback`）：

```cmd
cmd /c "cd build\release\artifacts && analyzer.exe -s vulkan-shim-test.exe vulkan-shim.dll"
```

在无 GPU 机器（如 CI）上放入 **SwiftShader** 作为软件 ICD：把 `vulkan-1.dll`、`vk_swiftshader.dll` 和 `vk_swiftshader_icd.json` 暂存到 `analyzer.exe` 旁边，运行前把加载器指向清单：

```cmd
set VK_ICD_FILENAMES=%CD%\vk_swiftshader_icd.json
```

运行编组往返测试：

```cmd
cmd /c "cd build\release\artifacts && windows-emulator-test.exe --gtest_filter=VulkanMarshalTest.*"
```

冒烟测试（必须保持绿）：

```cmd
cmd /c "cd build\release\artifacts && analyzer.exe -s test-sample.exe"
```
