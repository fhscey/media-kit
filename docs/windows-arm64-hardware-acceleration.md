# Windows ARM64 硬件加速方案（media_kit 插件视角）

> 目标：在 Windows ARM64 上让 media_kit 的视频播放**真正使用硬件加速**，
> 而不是回退到软件渲染。
>
> 背景（已确认的事实）：
> 1. media_kit 插件 `video_output.cc` 硬编码用 `MPV_RENDER_API_TYPE_OPENGL`（ANGLE）
>    做 H/W 渲染；`enableHardwareAcceleration` 默认 `true`。
> 2. 你的 `mpv-winbuild-cmake` 在 `aarch64` 分支把
>    `mpv_gl = "-Dgl=disabled -Degl-angle=disabled"`，导致 ARM64 的 libmpv
>    **没有 OpenGL/ANGLE → `mpv_render_context_create(OPENGL)` 失败 → 回退 S/W**。
> 3. 回退到 S/W 后，`GetVideoWidth/GetVideoHeight` 的整数除法 bug（先除后乘）
>    在竖屏大视频下算出 0 → 黑屏有声。该 bug 已在 media-kit `test` 分支修复
>    （commit `34f5e3e2`），竖屏现已能播放（走 S/W）。
>
> 因此，真正"硬件加速"的问题是：**ARM64 的 libmpv 需要支持硬件渲染后端**。

---

## 两条可行路径

| 路径 | 改动 | 硬件加速 | 状态 |
|---|---|---|---|
| **路径 A：aarch64 启用 ANGLE**（推荐先做） | `mpv-winbuild-cmake` 一处配置 | ✅ media_kit 现有 `ANGLESurfaceManager` H/W 路径直接生效 | 可立即落地（见下方 patch） |
| **路径 B：DXGI 原生（wiliwili 式）** | libmpv 打 D3D11 render 后端补丁 + media_kit 新增 DXGI 分支 | ✅ 彻底避开 ANGLE | 依赖 libmpv 补丁源码，工作量最大（见下文蓝图） |

---

## 路径 A（推荐）：让现有 H/W 在 ARM64 生效

media_kit 现有 `ANGLESurfaceManager`（ANGLE → D3D11 共享纹理）在 x64 上工作正常。
ARM64 上不需要它存在 bug，只要 libmpv 提供 ANGLE 即可创建 OPENGL render context。

### 改动（在你的 `mpv-winbuild-cmake` 里）

`cmake/packages_check.cmake` 的 aarch64 分支：把

```cmake
elseif(TARGET_CPU STREQUAL "aarch64")
    ...
    set(mpv_gl "-Dgl=disabled -Degl-angle=disabled")
```

改为：

```cmake
elseif(TARGET_CPU STREQUAL "aarch64")
    ...
    set(mpv_gl "-Dgl=enabled -Degl-angle=enabled")   # 与 x86_64/i686 一致
```

配套检查：
- ANGLE 已在本仓构建支持（x86_64/i686 已通过 `-Degl-angle=enabled` 走通；`mpv-release` 已依赖 `angle-headers`）。
- ARM64 需确保 ANGLE 的构建依赖（`angle-headers`）在 `mpv.cmake` 的 `DEPENDS` 中，必要时加上。
- 重新 `ninja` 构建 `aarch64` libmpv，替换 media_kit 的 `libs/windows/media_kit_libs_windows_video` 里的 ARM64 `libmpv-2.dll`。

### media_kit 侧

**无需改插件代码**——`enableHardwareAcceleration: true`（默认）会自动走 H/W。
（已提交的 `angle_surface_manager` 的 GPU 拷贝完成同步 `f67e7d9d` 会让 H/W 纹理读取更稳。）

一份可直接应用的 patch 见本目录 `mpv-winbuild-cmake-arm64-enable-angle.patch`。

---

## 路径 B：DXGI 原生渲染（wiliwili 式）——完整实施蓝图

### 关键前提（阻塞点）

`MPV_RENDER_API_TYPE_DXGI` / `mpv_dxgi_init_params` / `render_dxgi.h` 是 wiliwili
给 libmpv 打的**私有补丁**，官方 `mpv-player/mpv` 只有 `OPENGL` 与 `SW` 两种 render API。
该补丁源码**未公开**。因此路径 B 必须先有支持 D3D11 render 后端的 libmpv。

### libmpv 侧（在 `mpv-winbuild-cmake` 里为 mpv 增加 D3D11 渲染后端）

需要补丁实现（在 mpv 的 `libmpv/` 与 `video/out/`）：

1. 新增 `libmpv/render_dxgi.h`，声明：
   ```c
   typedef struct mpv_dxgi_init_params {
       ID3D11Device    *device;      // 传入的 D3D11 设备
       IDXGISwapChain  *swapchain;   // 或 NULL（配合 FBO/DXGI render 参数渲到纹理）
   } mpv_dxgi_init_params;

   typedef struct mpv_dxgi_fbo {
       ID3D11RenderTargetView *rtv;  // 目标 render target（可绑定到共享纹理）
       ID3D11Texture2D        *texture; // 若 rtv 为空则据此创建
       int w, h;
   } mpv_dxgi_fbo;
   ```
2. 新增 `MPV_RENDER_API_TYPE_DXGI`（`"dxgi"`）、`MPV_RENDER_PARAM_DXGI_INIT_PARAMS`、
   `MPV_RENDER_PARAM_DXGI_FBO`。
3. 实现一个 `libmpv_gpu` 后端：`video/out/gpu/libmpv_gpu_d3d11.c`，复用 mpv 现有
   `ra` D3D11 后端（`ra_d3d11` / libplacebo）：
   - `init`：读 `mpv_dxgi_init_params`，创建 RA。
   - `wrap_fbo`：读 `MPV_RENDER_PARAM_DXGI_FBO` 的 `mpv_dxgi_fbo`，包成 `ra_tex` 作为渲染目标。
   - 复用 wiliwili 的思路：`mpv_render_context_report_swap` 后做 D3D11 同步。
4. 在 `video/out/libmpv.c` 里注册该后端到 `render_backend`，并让
   `mpv_render_context_create` 接收 `MPV_RENDER_API_TYPE_DXGI`。

> 提示：mpv 官方已有 `vo=gpu-next`(libplacebo) 的 D3D11 支持，可作为 RA 来源；
> 难点在 libmpv render API 层接入，工作量 ~几百行 C。

### media_kit 插件侧（在 media-kit `test` 分支）

新增一个 D3D11/共享纹理的 VideoOutput 硬件surface manager（可新增
`dxgi_surface_manager.h/.cc` 或扩展现有 ANGLESurfaceManager），核心：

1. 创建一个**共享 D3D11 纹理**（`D3D11_RESOURCE_MISC_SHARED_NTHANDLE` +
   `IDXGIResource1::CreateSharedHandle`），作为 `mpv_dxgi_fbo` 的 render target。
2. `VideoOutput` 增加检测：若 libmpv 支持 `MPV_RENDER_API_TYPE_DXGI`，则用
   `MPV_RENDER_API_TYPE_DXGI` 建 render context；否则回退 ANGLE(H/W) / SW。
3. `Render` 里 `mpv_render_context_render` 渲到该纹理，然后通过
   `FlutterDesktopGpuSurfaceDescriptor`(kFlutterDesktopGpuSurfaceTypeDxgiSharedHandle)
   把共享句柄交给 Flutter。`kFlutterDesktopPixelFormatBGRA8888`。
4. 交换/帧同步与 `read_pixel` 与现有 H/W 一致（可复用 `f67e7d9d` 的 query 同步）。

### 落地顺序

1. 先做路径 A（ANGLE 启用）——让 ARM64 尽快有硬件加速并验证（改动小、风险低）。
2. 等拿到/实现 libmpv D3D11 后端后再做路径 B（DXGI），在 `test` 分支新增 DXGI 分支，
   保留 ANGLE/SW 作为回退。

---

## 参考

- wiliwili 的 DXGI 用法：`wiliwili/source/view/mpv_core.cpp`（`BOREALIS_USE_D3D11`）
- media_kit 现有 H/W：`media_kit_video/windows/angle_surface_manager.*`
- media_kit 渲染分支选择：`media_kit_video/windows/video_output.cc`