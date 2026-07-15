---
name: harmonyos-camera-rear-adaptation
description: 鸿蒙后置摄像头 UX 与功能适配开发。当开发者提到切换到后置摄像头后预览画面拉伸/变形/裁切、后摄下拍照按钮/录像按钮/模式切换条位置错乱或被遮挡、后摄下人脸检测框消失、后摄变焦范围与前摄不同导致滑块不对、闪光灯按钮在前摄仍可点（本该禁用）、滤镜在后摄不可用、getZoomRatioRange 前后摄返回不同、getSupportedOutputCapability 能力差异、前后摄能力不同导致 UI 没跟着变、Capability-First 能力驱动 UI、setXComponentSurfaceRect 预览比例适配、preconfig/PreconfigRatio 选比例时触发。适用于后置摄像头场景下根据相机实际能力动态调整预览比例、重新布局叠加控件、按能力启用/禁用功能（人脸/变焦/闪光灯/曝光/对焦）等场景。不包含摄像头选择与会话重建、相机旋转角度补偿、图片编辑、与相机无关的 UI。
---

## Agent Interface

```yaml
symptom_keywords:
  - preview stretched or distorted after switching to rear camera
  - photo button / shutter / mode switch bar misaligned or clipped after camera switch
  - face detection box disappears on rear camera though front camera shows it
  - zoom slider range is wrong on rear camera (different from front)
  - flash button stays enabled on front camera where flash is unsupported
  - UI does not update after switching cameras (功能按钮状态没跟着相机变)
  - rear camera preview has wrong aspect ratio / black bars / cropped fill
  - need to enable or disable camera features based on actual camera capability
  - switched camera but exposure/focus/macro controls show stale state

hard_constraints:
  - Any feature UI enable/disable MUST be driven by capability queries — never hardcode "rear camera always supports zoom/flash"; query getSupportedOutputCapability(device, mode) for output profiles + metadata detection types, then session-level getZoomRatioRange / isFlashModeSupported / isExposureModeSupported / isFocusModeSupported to decide each control's enabled state
  - After switching cameras (front↔rear) you MUST re-query all capabilities — front and rear cameras return different getZoomRatioRange / previewProfiles / supportedMetadataObjectTypes; a cached capability object across a switch is a bug
  - Preview aspect-ratio fit is done via XComponentController.setXComponentSurfaceRect(rect) computing surfaceWidth/Height from sensor ratio with centered offset — there is NO official objectFit for camera Surface; not calling setXComponentSurfaceRect leaves the Surface stretched to the component bounds
  - Flash and Torch are DIFFERENT objects — Flash (isFlashModeSupported/hasFlash, session-level, for photo) vs Torch (CameraManager.isTorchSupported/isTorchModeSupported, device-level flashlight); query the right one for the right button
  - Whether face detection is supported is determined by CameraOutputCapability.supportedMetadataObjectTypes containing MetadataObjectType.FACE_DETECTION — do not assume both cameras support it
  - getZoomRatioRange (11+) returns a range that differs between front and rear cameras; after a camera switch re-fetch it and update the zoom slider bounds before the user can drag
  - AutoDeviceSwitch (13+) can ONLY auto-switch among multiple FRONT lenses on foldables — it CANNOT switch between front and rear cameras; front↔rear switching still requires manual release + session rebuild, and the capability-driven UI refresh afterwards is this skill's concern
  - Not all foldStatusChange transitions require re-querying capabilities — foldables report an IDENTICAL camera set for HALF_FOLDED↔EXPANDED transitions (the lens does not change), so re-query/rebuild can be skipped there; only genuine camera switches (front↔rear, or fold states that change the available lens set) trigger the capability refresh
  - Prefer preconfig (12+) with PreconfigRatio (1:1 / 4:3 / 16:9, default 4:3) to let the system pick a matching profile by ratio rather than hardcoding a resolution; call canPreconfig first to check support

diagnostic_checklist:
  - Is every feature control's enabled state driven by a capability query, not hardcoded?
  - Are capabilities re-queried after every front↔rear camera switch (getSupportedOutputCapability + getZoomRatioRange + isFlashModeSupported etc.)?
  - Is setXComponentSurfaceRect called with surfaceWidth/Height computed from the sensor ratio and a centered offset?
  - Is Flash (session) distinguished from Torch (CameraManager) when wiring the flash/flashlight button?
  - Is face-detection support checked via supportedMetadataObjectTypes before showing the face-frame UI?
  - Is the zoom slider bound to getZoomRatioRange's return value, refreshed on camera switch?
  - Is preconfig/canPreconfig used to select a profile by ratio instead of a hardcoded resolution?
  - After a camera switch, does the UI show a transition state (e.g. loading overlay) until the new session start() succeeds, since there is no official transition API?
```

# 后置摄像头 UX 与功能适配

## 适配领域

本 skill 解决应用**切到后置摄像头后 UX/功能变形**的问题。根因是：**前后置摄像头的能力不同**（变焦范围、闪光灯支持、预览分辨率比例、人脸检测支持等），但应用的 UI 往往按某一个摄像头写死。切到另一个摄像头时，UI 没跟着实际能力调整，就出现拉伸、按钮错位、功能失效等"变形"。

本 skill 的核心是 **Capability-First**：以当前相机的实际能力驱动 UI，而非假设后摄一定支持哪些功能。

具体覆盖：
- **Q1 后摄预览拉伸/畸变/裁切**：按 sensor 比例用 `setXComponentSurfaceRect` 算 Surface 几何，居中显示防拉伸。
- **Q2 切相机后叠加 UI 布局错乱**：预览区域尺寸/比例变化时，拍照按钮、模式切换条、参数滑块等叠加控件如何随预览重新布局；切换过渡态处理。
- **Q3 后摄功能失效**：`getSupportedOutputCapability` + session 级 `getZoomRatioRange`/`isFlashModeSupported` 等查询能力差异，驱动功能按钮的启用/禁用。

**不覆盖**：摄像头**选择**与会话**重建**（切换镜头、`foldStatusChange` 监听、release/重建顺序）；相机预览/拍照/录像的**旋转角度**补偿；图片编辑、滤镜算法等与摄像头能力适配无关的内容。

> 本 skill 假设摄像头**已经选好、会话已经建好**。它只管：选好后，预览怎么正确显示、UI 控件怎么正确布局、功能按钮怎么按实际能力启用/禁用。

---

## 核心范式：Capability-First（能力驱动 UI）

这是整个 skill 的灵魂。所有"变形"问题的根治方法都是同一套范式：

```
1. 选定 CameraDevice（前后摄之一）
2. getSupportedOutputCapability(device, NORMAL_PHOTO)
   → 取 previewProfiles / photoProfiles / supportedMetadataObjectTypes
3. session 创建并 commitConfig 后
   → 逐项查运行时能力：getZoomRatioRange / isFlashModeSupported / isExposureModeSupported / isFocusModeSupported ...
4. 用查询结果驱动 UI：
   - 变焦滑块的范围 ← getZoomRatioRange 的返回
   - 闪光灯按钮启用/禁用 ← isFlashModeSupported
   - 人脸框开关显示/隐藏 ← supportedMetadataObjectTypes 含 FACE_DETECTION
   - 预览 Surface 几何 ← previewProfile 的宽高比
5. 切换相机（前后摄）后 → 重新执行整条链（能力变了，UI 必须刷新）
```

> **为什么不能写死**：同一台设备上，前置摄像头可能不支持闪光灯、变焦范围窄、预览比例不同；后置摄像头可能有多个镜头（广角/长焦）、变焦范围广、支持闪光灯。硬编码任何一个，切到另一个摄像头就会"变形"。查询 API 是唯一可靠的依据。

---

## 能力差异矩阵

前后摄能力差异（典型情况，**具体值以运行时查询为准**，不可写死）：

| 能力 | 查询 API | 归属 | 前摄典型 | 后摄典型 | UI 影响 |
|------|---------|------|---------|---------|---------|
| 变焦范围 | `getZoomRatioRange()` (11+) | session | 窄或不支持 | 广（多镜头） | 滑块范围要变 |
| 闪光灯 | `isFlashModeSupported(mode)` / `hasFlash` | session (Flash) | 常不支持 | 支持 | 按钮要禁用/启用 |
| 手电筒 | `CameraManager.isTorchSupported()` (11+) | CameraManager | — | 支持 | 独立按钮（≠闪光灯） |
| 手电筒亮度 | `isTorchLevelControlSupported()` (26.0.0) | CameraManager | — | 支持 | 亮度滑块显隐 |
| 元数据检测 | `supportedMetadataObjectTypes` | output capability | FACE_DETECTION | FACE + CAT/DOG/SALIENT (26.0.0) | 检测框种类驱动 |
| 曝光模式 | `isExposureModeSupported(mode)` | session (AutoExposure) | — | — | 曝光 UI |
| 对焦模式 | `isFocusModeSupported(mode)` | session (Focus) | — | — | 对焦 UI |
| 预览比例 | `previewProfiles` / `preconfig` | output capability | — | — | Surface 几何 |

> **重点**：`getSupportedOutputCapability` 只返回**输出流能力**（profiles + 人脸检测类型），**不**返回 flash/zoom/exposure 等运行时功能能力。后者必须在 session 创建后逐项查。两步缺一不可。

---

## 适配流程

### 新适配

**第 1 步：建立能力查询层（Capability-First）**

封装一个能力查询函数，选定相机后一次性取回所有 UI 需要的状态。见「代码模式 A」。这是后续所有 UI 决策的数据源。

**第 2 步：后摄预览比例适配（防拉伸）**

用 `preconfig`（12+）按 `PreconfigRatio` 选 profile，或手动从 `previewProfiles` 选目标比例；然后用 `setXComponentSurfaceRect` 按 sensor 宽高比算 `surfaceWidth/surfaceHeight`，offset 居中。见「代码模式 B」。

> 官方没有相机层的 objectFit。不调 `setXComponentSurfaceRect`，Surface 默认按组件拉伸——这就是变形的根因。

**第 3 步：叠加 UI 响应式布局**

XComponent 与拍照按钮/模式切换条/参数滑块放在同一个 `Stack`，控件用相对定位或响应式属性。预览区域比例变化时（前后摄 profile 不同、折叠/旋转），控件随 Stack 重布局。见「代码模式 C」。

**第 4 步：切换相机后能力重查与 UI 刷新**

切换前后摄是手动 release + 重建（AutoDeviceSwitch 做不了前后摄切换）。重建后**必须重新查询能力并刷新 UI**——否则 UI 停留在上一个相机的状态（滑块范围错、闪光灯按钮该禁用没禁用）。见「代码模式 D」。

> **折叠态豁免**：并非所有 `foldStatusChange` 都需要重查能力。折叠设备在半折叠态（`HALF_FOLDED`）与展开态（`EXPANDED`）之间切换时，相机框架返回的镜头集完全一致，无需重建 session 或重查能力。只有真正改变了可用镜头集的切换（前后摄切换、折叠态导致前置镜头变更）才触发重查。

---

### 问题定位

**现象 → 可能原因**：

| 现象 | 可能原因 | 定位 |
|------|---------|------|
| 切到后摄预览拉伸/有黑边/裁切 | 没调 `setXComponentSurfaceRect`，Surface 按组件拉伸；或 profile 比例与组件不一致 | 检查是否按 sensor 比例算了几何 |
| 切前后摄后拍照按钮/模式条错位 | 控件没用响应式布局，预览区域变了但控件位置写死 | 检查控件是否在 Stack 内响应式定位 |
| 后摄变焦滑块范围和前摄一样（错） | 切换后没重新 `getZoomRatioRange`，用了缓存的旧范围 | 检查切换后是否重查能力 |
| 闪光灯按钮在前摄还能点 | 没按 `isFlashModeSupported` 启用/禁用，写死了 | 检查按钮 enabled 是否绑定能力查询 |
| 后摄没人脸框（该有的没有/不该有的有） | 没查 `supportedMetadataObjectTypes` | 检查人脸框开关是否由 FACE_DETECTION 支持性驱动 |
| 切相机后 UI 卡在旧状态 | 没在重建后刷新 UI 状态 | 检查第 4 步是否执行 |

**3 步定位法**：
1. **收集现象**：是拉伸、按钮错位、还是功能失效？在哪种摄像头（前/后）下复现？
2. **确认能力驱动**：相关 UI 控件的状态是写死的，还是由能力查询驱动的？切换相机后有没有重新查询？
3. **修复与验证**：按对应代码模式改为能力驱动，按「验证清单」反复切前后摄验证。

---

## 关键 API

### CameraManager.getSupportedOutputCapability(camera, mode)

**用途**：查询指定相机的输出流能力。**入参是具体的 CameraDevice + 必填的 SceneMode**（如 `NORMAL_PHOTO`）——前后摄是不同 device，返回的能力集天然不同。

**返回** `CameraOutputCapability`：`previewProfiles` / `photoProfiles` / `videoProfiles` / `supportedMetadataObjectTypes`。

> **注意**：只返回输出流能力，**不**含 flash/zoom/exposure 等运行时能力。后者在 session 创建后另查。23+ 有增强版 `getSupportedFullOutputCapability`（含 YUV/HEIF/HDR，用前必须先调）。

### PhotoSession 能力查询方法（session 级，commitConfig 后调用）

| 方法 | 用途 | 版本 |
|------|------|------|
| `getZoomRatioRange()` | 变焦范围（前后摄不同） | 11+ |
| `isFlashModeSupported(mode)` / `hasFlash` | 闪光灯模式支持 | Flash mixin |
| `isExposureModeSupported(mode)` | 曝光模式支持 | AutoExposure mixin |
| `isFocusModeSupported(mode)` | 对焦模式支持 | Focus mixin |

> 这些方法的精确定义在 mixin 文档（见 `references/capability-api-reference.md`）：[zoomquery](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-zoomquery) / [flashquery](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-flashquery) / [autoexposurequery](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-autoexposurequery) / [focusquery](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-focusquery)。本 skill 只给归属与版本，展开细节查官方 mixin 文档。

### XComponentController.setXComponentSurfaceRect(rect) (12+)

**用途**：设置 Surface 显示区域（宽高 + 相对组件左上角的 offset）。这是预览比例适配的核心——按 sensor 比例算宽高，offset 居中，避免拉伸。

**参数** `SurfaceRect`：`offsetX/offsetY/surfaceWidth/surfaceHeight`（px，上限 8192）。offset 不设默认**居中**。

> 不调此方法，Surface 默认按组件宽高拉伸——变形根因。搭配 `onSurfaceChanged(surfaceId, rect)`(12+) 在 Surface 大小变化时重算。

### PhotoSession.preconfig / canPreconfig (12+)

**用途**：让系统按类型+比例选 profile，免手动遍历 `previewProfiles`。

`preconfig(preconfigType, preconfigRatio?)`：`PreconfigType`（720P/1080P/4K/HIGH_QUALITY）+ `PreconfigRatio`（1:1 / 4:3 / 16:9，默认 4:3）。`canPreconfig` 先查是否支持。

### Flash vs Torch

Flash（session 级，拍照闪光）与 Torch（CameraManager 级，手电筒）是两套 API、两个按钮，不可混用。26.0.0 新增 `isTorchLevelControlSupported()` / `setTorchModeOnWithLevel(level)`（level 范围 `[0.0, 1.0]`）可调手电筒亮度。完整对照见 `references/capability-api-reference.md`。

---

## 代码模式

### 模式 A：能力查询层（Capability-First）

封装一次查询，产出驱动 UI 的状态对象：

```typescript
import { camera } from '@kit.CameraKit';

// 驱动 UI 的能力状态
interface CameraUiCapability {
  zoomRange: [number, number] | null;          // getZoomRatioRange，null=不支持变焦
  flashSupported: boolean;                      // 闪光灯
  torchLevelSupported: boolean;                 // 手电筒亮度可调（26.0.0）
  detectionTypes: camera.MetadataObjectType[];  // 支持的检测类型（人脸/宠物等）
  exposureModes: camera.ExposureMode[];         // 支持的曝光模式
  previewAspectRatio: number;                   // 预览宽高比（驱动 Surface 几何）
}

async function queryCameraUiCapability(
  cameraManager: camera.CameraManager,
  session: camera.PhotoSession,
  device: camera.CameraDevice
): Promise<CameraUiCapability> {
  // 1. 输出流能力（profiles + 元数据检测类型）——mode 必填
  const outputCap = cameraManager.getSupportedOutputCapability(
    device, camera.SceneMode.NORMAL_PHOTO
  );
  const detectionTypes = outputCap.supportedMetadataObjectTypes ?? [];

  // ⚠ previewProfiles[0] 仅为示意——数组顺序不保证比例正确。
  //    生产代码应按目标比例/分辨率筛选 profile（findIndex 匹配 width/height/format），
  //    或用 preconfig(PreconfigType, PreconfigRatio) 让系统按比例选 profile。
  const previewAspectRatio = outputCap.previewProfiles[0].width / outputCap.previewProfiles[0].height;

  // 2. 运行时能力（session 级，commitConfig 后才能查）
  let zoomRange: [number, number] | null = null;
  try {
    const range = session.getZoomRatioRange();
    zoomRange = [range.min, range.max];
  } catch (e) { /* 不支持变焦 */ }

  const flashSupported = session.hasFlash?.() ?? false;

  // 3. 手电筒亮度（CameraManager 级，26.0.0）
  const torchLevelSupported = cameraManager.isTorchLevelControlSupported?.() ?? false;

  // 4. 曝光模式（遍历枚举查支持项）
  const exposureModes: camera.ExposureMode[] = [];
  for (const m of [camera.ExposureMode.EXPOSURE_MODE_LOCKED,
                    camera.ExposureMode.EXPOSURE_MODE_AUTO,
                    camera.ExposureMode.EXPOSURE_MODE_CONTINUOUS_AUTO]) {
    if (session.isExposureModeSupported?.(m)) exposureModes.push(m);
  }

  return {
    zoomRange, flashSupported, torchLevelSupported,
    detectionTypes, exposureModes, previewAspectRatio,
  };
}
```

> UI 控件的 enabled/range/visible 全部绑定到这个对象。切换相机后重新调用本函数刷新。

### 模式 B：预览比例适配（setXComponentSurfaceRect）

```typescript
import { camera } from '@kit.CameraKit';

// 按 sensor 比例算 Surface 几何，居中显示防拉伸
function fitPreviewSurface(
  controller: XComponentController,
  componentWidth: number,
  componentHeight: number,
  sensorAspectRatio: number   // 宽/高，如 16/9 或 4/3
): void {
  const componentAspect = componentWidth / componentHeight;
  let surfaceWidth: number;
  let surfaceHeight: number;

  if (sensorAspectRatio > componentAspect) {
    // sensor 更宽 → 按宽度撑满，高度按比例缩（上下留黑/裁切）
    surfaceWidth = componentWidth;
    surfaceHeight = componentWidth / sensorAspectRatio;
  } else {
    // sensor 更高 → 按高度撑满，宽度按比例缩（左右留黑/裁切）
    surfaceHeight = componentHeight;
    surfaceWidth = componentHeight * sensorAspectRatio;
  }

  // offset 居中
  controller.setXComponentSurfaceRect({
    surfaceWidth: Math.floor(surfaceWidth),
    surfaceHeight: Math.floor(surfaceHeight),
    offsetX: Math.floor((componentWidth - surfaceWidth) / 2),
    offsetY: Math.floor((componentHeight - surfaceHeight) / 2),
  });
}
```

> 配合 `onSurfaceChanged(surfaceId, rect)`(12+) 在 Surface 大小变化（折叠/旋转）时重算。sensorAspectRatio 来自 previewProfile 的 width/height。

### 模式 C：叠加 UI 响应式布局

XComponent 与控件同放 `Stack`，控件响应式定位：

```typescript
@Entry
@Component
struct CameraPage {
  @State uiCap: CameraUiCapability | null = null;
  @State zoomValue: number = 1;
  mXComponentController: XComponentController = new XComponentController();

  build() {
    Stack() {
      // 预览层
      XComponent({ type: XComponentType.SURFACE, controller: this.mXComponentController })
        .width('100%').height('100%')
        .onLoad(() => { /* initCamera + queryCameraUiCapability + fitPreviewSurface */ })

      // 叠加控件层 —— 用响应式属性，预览区域变就跟着重布局
      Column() {
        // 顶部：模式切换条
        Row() { /* 模式按钮 */ }
        .width('100%').justifyContent(FlexAlign.Center)

        Blank()  // 弹性占位，把底部控件推下去

        // 底部：变焦滑块（范围绑定能力查询结果）
        if (this.uiCap?.zoomRange) {
          Slider({ value: this.zoomValue, min: this.uiCap.zoomRange[0], max: this.uiCap.zoomRange[1] })
            .onChange((v) => { this.zoomValue = v; /* session.setZoomRatio(v) */ })
        }

        // 拍照按钮 + 闪光灯（按能力启用/禁用）
        Row() {
          // 闪光灯按钮：前摄常不支持，按 flashSupported 决定 enabled
          Image($r('app.media.flash'))
            .opacity(this.uiCap?.flashSupported ? 1 : 0.3)
            .enabled(this.uiCap?.flashSupported ?? false)
            .onClick(() => { /* 切换闪光模式 */ })

          Image($r('app.media.shutter'))
            .onClick(() => { /* 拍照 */ })
        }
        .width('100%').justifyContent(FlexAlign.SpaceEvenly)
      }
      .width('100%').height('100%')
      .padding({ top: 48, bottom: 48 })  // 避让安全区
    }
  }
}
```

> 控件用 `Blank()`/`width('100%')`/`justifyContent` 等响应式属性，预览区域比例变了会自动重布局，不需要手算位置。

### 模式 C 补充：双 XComponent 防残影

切换相机时（前后摄/折叠态变更），旧相机的 Surface 释放与新相机的创建存在时间差，单个 XComponent 会残留上一个相机的画面。官方折叠屏示例用**双 XComponent 实例翻转**解决：用 `reloadXComponentFlag` 布尔翻转，`if/else` 分支各持一个 XComponent 实例，切换时翻转 flag 让旧实例连同旧 Surface 一起被 ArkUI 销毁。

```typescript
@Entry
@Component
struct CameraPage {
  @State reloadXComponentFlag: boolean = false;
  mXComponentController: XComponentController = new XComponentController();
  mXComponentOptions: XComponentOptions = {
    type: XComponentType.SURFACE,
    controller: this.mXComponentController
  }

  // 翻转 flag 触发 XComponent 重建
  reloadXComponent() {
    this.reloadXComponentFlag = !this.reloadXComponentFlag;
  }

  build() {
    Stack() {
      // 双实例：if/else 各一个 XComponent，翻转时旧实例自动销毁
      if (this.reloadXComponentFlag) {
        XComponent(this.mXComponentOptions)
          .onLoad(() => { /* initCamera + queryCameraUiCapability + fitPreviewSurface */ })
          .width('100%').height('100%')
      } else {
        XComponent(this.mXComponentOptions)
          .onLoad(() => { /* initCamera + queryCameraUiCapability + fitPreviewSurface */ })
          .width('100%').height('100%')
      }
      // ... 叠加控件层（同模式 C 主体） ...
    }
  }
}
```

> 双 XComponent 翻转与叠加控件响应式布局组合使用：前者解决切换残影，后者解决比例变化后的控件错位。

### 模式 D：切换相机后能力重查

```typescript
async function switchCameraAndRefreshUi(
  cameraManager: camera.CameraManager,
  newXComponentController: XComponentController,
  newDevice: camera.CameraDevice
): Promise<void> {
  // 1. 重建 session（release + 重建，AutoDeviceSwitch 做不了前后摄切换）
  //    重建流程（stop→close→release→undefined + 重建）不在本 skill 范围
  const session = await rebuildSession(cameraManager, newDevice, newXComponentController);

  // 2. 关键：重新查询能力 —— 前后摄能力不同，UI 必须刷新
  const newCap = await queryCameraUiCapability(cameraManager, session, newDevice);

  // 3. 重新适配预览比例
  fitPreviewSurface(
    newXComponentController,
    componentWidth, componentHeight,
    newCap.previewAspectRatio
  );

  // 4. 刷新 UI 状态（变焦滑块范围、闪光灯可用性、人脸框开关等）
  //    this.uiCap = newCap;  →  ArkUI 响应式自动更新
}
```

> 切换期间（release→重建→start 完成）没有官方过渡态 API。防残影首选双 XComponent 翻转（见模式 C 补充），让旧 Surface 随旧实例一起销毁；loading 遮罩作为补充，在新 session `start()` 成功后撤掉。

---

## 验证清单

**基础验证（每次必做）**：

- [ ] 切换前后摄，预览均无拉伸/畸变（setXComponentSurfaceRect 几何正确）
- [ ] 切换前后摄，拍照按钮/模式切换条/参数滑块位置正确，不被遮挡
- [ ] 闪光灯按钮在支持时启用、不支持时禁用（按 isFlashModeSupported/hasFlash，不是写死）
- [ ] 变焦滑块范围在前后摄下都正确（按 getZoomRatioRange，切换后重取）
- [ ] 人脸检测框的显示由 supportedMetadataObjectTypes 驱动

**压力验证（发布前做）**：

- [ ] 快速反复切前后摄 5 次以上，UI 状态每次都正确同步（无残留旧状态）
- [ ] 后摄变焦滑块拖到边界，行为正确（范围来自 getZoomRatioRange）
- [ ] 闪光灯按钮在手电筒按钮旁时，两者不混淆（Flash ≠ Torch）
- [ ] 折叠/旋转后预览比例重新适配（onSurfaceChanged 重算）
- [ ] 不支持变焦的摄像头（如某些前摄），变焦滑块正确隐藏或禁用

---

## 常见问题

**Q：后摄变焦滑块范围和前摄不一样，是 bug 吗？**
A：不是。`getZoomRatioRange`(11+) 对前后摄返回不同范围（后摄通常更广，有多镜头）。正确做法：切换相机后重新调用 `getZoomRatioRange`，用返回值更新滑块的 min/max。缓存旧范围是 bug。

**Q：闪光灯按钮在所有摄像头都亮着（可点），对吗？**
A：不对。前摄常不支持闪光灯。按钮的 enabled 应由 `isFlashModeSupported` / `hasFlash` 查询结果决定，不能写死。另外注意 Flash（session 级，拍照用）和 Torch（CameraManager 级，手电筒）是两套 API、两个按钮，别混用。

**Q：后摄没人脸检测框？**
A：查 `CameraOutputCapability.supportedMetadataObjectTypes` 是否含 `MetadataObjectType.FACE_DETECTION`。前后摄对这个的支持可能不同，人脸框 UI 的显隐要由这个查询驱动。

**Q：预览总是拉伸，调了 profile 还是变形？**
A：选对 profile 只是一半。关键是调 `setXComponentSurfaceRect`(12+) 按 sensor 宽高比算 `surfaceWidth/surfaceHeight`，offset 居中。官方没有相机层的 objectFit，不调这个方法 Surface 就按组件拉伸。

**Q：AutoDeviceSwitch(13+) 能帮我自动切前后摄吗？**
A：不能。文档原文：`AutoDeviceSwitch` 仅用于"有多个前置镜头的折叠设备，在不同折叠状态下自动切换到当前可使用的前置镜头"，**无法实现前后置镜头的切换**。前后摄切换必须手动 release + 重建。它的价值范式（`isDeviceCapabilityChanged` 信号 → 重新查能力 → 刷新 UI）可借鉴，但切换本身得手动做。

---

## 延伸阅读

能力查询 API 完整表（方法/归属/版本）、枚举（CameraPosition/CameraType/FlashMode/TorchMode/ExposureMode/FocusMode/MetadataObjectType/PreconfigType/PreconfigRatio）、`CameraOutputCapability` 结构、Flash vs Torch 对照、`setXComponentSurfaceRect` 几何计算详解，见 `references/capability-api-reference.md`。
