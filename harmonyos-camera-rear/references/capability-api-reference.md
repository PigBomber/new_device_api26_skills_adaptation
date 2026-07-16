# 相机能力查询 API 完整参考

> 本文件为 `SKILL.md` 的补充，记录能力查询 API（方法/归属/版本）、相关枚举、`CameraOutputCapability` 结构、Flash vs Torch 对照、`setXComponentSurfaceRect` 几何计算详解。SKILL.md 讲流程与模式，本文件讲 API 细节。

> **来源**：本文件内容来自鸿蒙官方 Camera Kit 文档。其中 session 级能力查询方法（`getZoomRatioRange` / `isFlashModeSupported` / `isExposureModeSupported` / `isFocusModeSupported`）的精确定义在 mixin 文档：[zoomquery](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-zoomquery) / [flashquery](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-flashquery) / [autoexposurequery](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-autoexposurequery) / [focusquery](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-focusquery)。本文件给出归属与版本，参数细节请查官方对应 mixin 文档。

---

## 能力查询 API 分层

相机能力查询分两层，**两步缺一不可**：

| 层 | 查什么 | 何时查 | API |
|----|-------|-------|-----|
| **输出流能力** | preview/photo/video profiles、元数据检测类型 | 选定 device 后即可查 | `getSupportedOutputCapability(device, mode)` |
| **运行时功能能力** | 变焦范围、闪光灯、曝光、对焦 | session `commitConfig` 后才能查 | session 级 `getZoomRatioRange` / `isFlashModeSupported` 等 |

> `getSupportedOutputCapability` **不**返回 flash/zoom/exposure 等运行时能力。只查第一层会导致功能按钮状态缺失。

---

## CameraManager 级 API

### getSupportedOutputCapability(camera, mode) — 输出流能力

**入参**：`CameraDevice`（前后摄是不同 device）+ 必填 `mode`（`SceneMode`，常用 `NORMAL_PHOTO`）。

**返回** `CameraOutputCapability`：

| 字段 | 类型 | 说明 |
|------|------|------|
| `previewProfiles` | `Array<Profile>` | 预览分辨率列表（width/height/format） |
| `photoProfiles` | `Array<Profile>` | 拍照分辨率列表 |
| `videoProfiles` | `Array<Profile>` | 录像分辨率列表 |
| `supportedMetadataObjectTypes` | `Array<MetadataObjectType>` | 支持的元数据类型（含 `FACE_DETECTION` 等） |

> 前后摄返回的 profiles 不同——这是预览比例适配要按 device 查的原因。

### getSupportedFullOutputCapability (23+) — 增强版

含未压缩图（YUV）/HEIF/HDR。**用 YUV/HEIF/HDR 前必须先调它**。Stage 模型约束。

### Torch（手电筒，设备级）

| 方法 | 版本 | 说明 |
|------|------|------|
| `CameraManager.isTorchSupported()` | 11+ | 是否支持手电筒 |
| `CameraManager.isTorchModeSupported(mode)` | 11+ | 是否支持指定 TorchMode |
| `CameraManager.setTorchMode(mode)` | 11+ | 设置手电筒模式 |
| `CameraManager.isTorchLevelControlSupported()` | 26.0.0 | 是否支持亮度控制 |
| `CameraManager.setTorchModeOnWithLevel(level)` | 26.0.0 | 开手电筒并设亮度（level 范围 `[0.0, 1.0]`） |

> Torch 在 **CameraManager** 层，与拍照闪光灯（Flash，session 层）是两套 API。

---

## PhotoSession 级能力查询（mixin 接口）

PhotoSession 通过继承多个 mixin 接口获得能力查询方法。这些方法在 session `commitConfig` 之后调用。

### 变焦（Zoom / ZoomQuery mixin）

| 方法 | 版本 | 说明 |
|------|------|------|
| `getZoomRatioRange()` | 11+ | 变焦范围（返回 min/max，**前后摄不同**） |
| `isZoomSupported()` | Zoom mixin | 是否支持变焦 |
| `getZoomRatio()` | — | 当前变焦倍数 |
| `setZoomRatio(ratio)` | — | 设置变焦 |

> 精确定义见官方文档：[zoomquery](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-zoomquery)（getZoomRatioRange）/ [zoom](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-zoom)（setZoomRatio）。

### 闪光灯（Flash mixin）

| 方法 | 说明 |
|------|------|
| `hasFlash()` | 是否有闪光灯硬件 |
| `isFlashModeSupported(mode)` | 是否支持指定 FlashMode |
| `getFlashMode()` / `setFlashMode(mode)` | 读/设闪光模式 |

> 能力查询（`hasFlash`/`isFlashModeSupported`）见 [flashquery](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-flashquery)，操作方法（`get/setFlashMode`）见 [flash](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-flash)。

### 曝光（AutoExposure mixin）

| 方法 | 说明 |
|------|------|
| `isExposureModeSupported(mode)` | 是否支持指定 ExposureMode |
| `getExposureMode()` / `setExposureMode(mode)` | 读/设曝光模式 |

> 能力查询（`isExposureModeSupported`）见 [autoexposurequery](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-autoexposurequery)，操作方法见 [autoexposure](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-autoexposure)。

### 对焦（Focus mixin）

| 方法 | 说明 |
|------|------|
| `isFocusModeSupported(mode)` | 是否支持指定 FocusMode |
| `getFocusMode()` / `setFocusMode(mode)` | 读/设对焦模式 |

> 能力查询（`isFocusModeSupported`）见 [focusquery](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-focusquery)，操作方法见 [focus](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-camera-focus)。

### 预配置（PhotoSession 自身）

| 方法 | 版本 | 说明 |
|------|------|------|
| `canPreconfig(type, ratio?)` | 12+ | 是否支持指定预配置 |
| `preconfig(type, ratio?)` | 12+ | 让系统按类型+比例选 profile |

---

## 枚举速查（来自 camera-e）

### CameraPosition

| 枚举值 | 含义 |
|--------|------|
| `CAMERA_POSITION_BACK` | 后置（折叠屏镜头 A，各态一致；2in1 后摄） |
| `CAMERA_POSITION_FRONT` | 前置（折叠屏展开 B / 折叠 C，身份会变） |
| `CAMERA_POSITION_UNSPECIFIED` | 未指定 |

> 用 `cameraPosition` 区分前后置，**不要**用 `cameraType`（后者是镜头类型，无 FRONT/BACK 值）。

### CameraType（镜头类型，与前后置无关）

| 枚举值 | 含义 |
|--------|------|
| `CAMERA_TYPE_DEFAULT` | 默认 |
| `CAMERA_TYPE_WIDE_ANGLE` | 广角 |
| `CAMERA_TYPE_ULTRA_WIDE` | 超广角 |
| `CAMERA_TYPE_TELEPHOTO` | 长焦 |
| `CAMERA_TYPE_TRUE_DEPTH` | 深感 |

> 后摄可能暴露多个 CameraType（广角+长焦+超广角），能力各不同。

### FlashMode（拍照闪光）

| 枚举值 | 含义 |
|--------|------|
| `FLASH_MODE_CLOSE` | 关闭 |
| `FLASH_MODE_OPEN` | 常开 |
| `FLASH_MODE_AUTO` | 自动 |
| `FLASH_MODE_ALWAYS_OPEN` | 始终开启 |

### FlashState (24+) — 闪光灯可用性状态

| 枚举值 | 含义 |
|--------|------|
| `UNAVAILABLE` | 不可用 |
| `READY` | 就绪 |
| `FLASHING` | 闪光中 |

### TorchMode（手电筒）

| 枚举值 | 含义 |
|--------|------|
| `TORCH_MODE_OFF` | 关 |
| `TORCH_MODE_ON` | 开 |
| `TORCH_MODE_AUTO` | 自动 |

### ExposureMode

| 枚举值 | 含义 |
|--------|------|
| `EXPOSURE_MODE_LOCKED` | 锁定 |
| `EXPOSURE_MODE_AUTO` | 自动 |
| `EXPOSURE_MODE_CONTINUOUS_AUTO` | 连续自动 |

### FocusMode

| 枚举值 | 含义 |
|--------|------|
| `FOCUS_MODE_MANUAL` | 手动 |
| `FOCUS_MODE_CONTINUOUS_AUTO` | 连续自动 |
| `FOCUS_MODE_LOCKED` | 锁定 |

### MetadataObjectType（元数据类型，含人脸检测）

| 枚举值 | 含义 |
|--------|------|
| `FACE_DETECTION` (0) | 人脸检测 |
| `CAT` / `DOG` / `SALIENT` (26.0.0) | 猫/狗/显著区域检测 |

> 检测类型是否支持，看 `supportedMetadataObjectTypes` 是否包含对应枚举值（人脸/宠物/显著区域等）。前后摄支持的类型可能不同。

### PreconfigType

| 枚举值 | 含义 |
|--------|------|
| `PRECONFIG_TYPE_720P` | 720P |
| `PRECONFIG_TYPE_1080P` | 1080P |
| `PRECONFIG_TYPE_4K` | 4K |
| `PRECONFIG_TYPE_HIGH_QUALITY` | 高质量 |

### PreconfigRatio

| 枚举值 | 含义 |
|--------|------|
| `PRECONFIG_RATIO_1_1` | 1:1 |
| `PRECONFIG_RATIO_4_3` | 4:3（默认） |
| `PRECONFIG_RATIO_16_9` | 16:9 |

### Profile format（常用）

| 枚举值 | 值 | 用途 |
|--------|----|------|
| `CAMERA_FORMAT_YUV_420_SP` | 1003 | 预览常用 |
| `CAMERA_FORMAT_JPEG` | 2000 | 拍照常用 |
| `CAMERA_FORMAT_HEIC` | 2003 (13+) | 拍照 HEIF |

---

## Flash vs Torch 对照

两者完全不同，API 分属不同对象：

| 项 | Flash | Torch |
|----|-------|-------|
| 归属对象 | PhotoSession（Flash mixin） | CameraManager |
| 用途 | 拍照时闪光 | 设备级手电筒（持续照明） |
| 查询 API | `session.isFlashModeSupported(mode)` / `session.hasFlash()` | `cameraManager.isTorchSupported()` (11+) |
| 设置 API | `session.setFlashMode(mode)` | `cameraManager.setTorchMode(mode)` |
| 枚举 | `FlashMode`（CLOSE/OPEN/AUTO/ALWAYS_OPEN） | `TorchMode`（OFF/ON/AUTO） |
| 亮度控制 | 无 | `setTorchModeOnWithLevel` (26.0.0) |
| 典型场景 | 拍照按钮旁的闪电图标 | 单独的手电筒按钮 |

> UI 上闪光灯按钮和手电筒按钮是两个独立控件，分别绑各自的查询 API。

---

## setXComponentSurfaceRect 几何计算详解

官方没有相机层的 objectFit。预览比例适配完全靠 `XComponentController.setXComponentSurfaceRect(rect: SurfaceRect)`(12+) 算几何。

### SurfaceRect 结构

| 字段 | 类型 | 说明 |
|------|------|------|
| `offsetX` | number | Surface 左上角相对组件左上角的 x 偏移（px） |
| `offsetY` | number | y 偏移 |
| `surfaceWidth` | number | Surface 宽（px，上限 8192） |
| `surfaceHeight` | number | Surface 高（px，上限 8192） |

> 不设 offset 默认居中。不设 width/height 默认等于组件宽高（= 拉伸变形）。

### 几何计算（contain 模式：完整显示，留黑边）

目标：让 Surface 的宽高比 = sensor 的宽高比（来自 previewProfile 的 width/height），且在组件内居中。

```
componentAspect = componentWidth / componentHeight
sensorAspect    = profile.width / profile.height

if sensorAspect > componentAspect:
    // sensor 更宽 → 按宽度撑满，高度按比例缩
    surfaceWidth  = componentWidth
    surfaceHeight = componentWidth / sensorAspect
else:
    // sensor 更高 → 按高度撑满，宽度按比例缩
    surfaceHeight = componentHeight
    surfaceWidth  = componentHeight * sensorAspect

offsetX = (componentWidth  - surfaceWidth)  / 2   // 居中
offsetY = (componentHeight - surfaceHeight) / 2
```

### 何时重算

- 切换相机（前后摄 profile 比例不同）
- 折叠/展开（组件尺寸变）
- 屏幕旋转（组件宽高互换）

> 搭配 `onSurfaceChanged(surfaceId, rect)`(12+) 回调，在 Surface 大小变化时触发重算。

### cover 模式（填满，裁切超出部分）

如果希望预览填满组件（不留黑边，裁掉超出部分），把 contain 逻辑反过来——按短边撑满：

```
if sensorAspect > componentAspect:
    // sensor 更宽 → 按高度撑满，宽度溢出（左右裁切）
    surfaceHeight = componentHeight
    surfaceWidth  = componentHeight * sensorAspect
else:
    surfaceWidth  = componentWidth
    surfaceHeight = componentWidth / sensorAspect

offsetX = (componentWidth  - surfaceWidth)  / 2   // 负值 = 居中裁切
offsetY = (componentHeight - surfaceHeight) / 2
```

> 选择 contain（留黑边）还是 cover（裁切）取决于产品设计。两者都靠 setXComponentSurfaceRect 实现。

> 完整的能力查询代码实现见 SKILL.md「模式 A：能力查询层」。
