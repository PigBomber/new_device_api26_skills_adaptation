---
name: harmonyos-camera-rear
description: 后置摄像头预览与叠加 UI 的适配。当开发者提到后置摄像头/后摄 适配、切到后摄黑屏/无画面/预览不显示、前后摄切换后黑屏、折叠后后摄不可用/黑屏、后摄相机初始化失败、后摄下 UI/控件/按钮/布局 错乱或被遮挡、后摄切到前摄或前摄切回后摄后控件跑位、后摄预览区在折叠/旋转/分屏后控件没跟着重排、控件被状态栏/导航条/挖孔遮挡需避让安全区时触发。适用于后置摄像头预览能正常显示（黑屏排查、相机生命周期、切换/折叠重建）以及预览之上叠加控件的跨折叠态/旋转/分屏响应式重布局与安全区避让。不包含按相机能力查询驱动功能按钮启停（变焦范围/闪光灯/人脸检测/exposure/focus）、通用 HarmonyOS 一多布局教程（断点/GridRow）、图片编辑与滤镜算法。
---

## Agent Interface

```yaml
symptom_keywords:
  - black screen / no preview after fold/unfold (expand↔fold) on rear camera
  - black screen after switching front↔rear camera (preview shows nothing)
  - shutter / photo button misaligned or clipped after switching camera or folding
  - mode switch bar or parameter slider wrong position after preview area resized
  - overlay controls don't reflow when preview area size/ratio changes (front↔rear, fold, rotate, split)
  - camera UI controls overlapped by status bar / nav indicator / cutout
  - need responsive relayout of overlay controls across fold/rotate/split-screen
  - controls show stale layout after camera switch transition
  - controls clip into preview or each other on large screens / expanded foldables

hard_constraints:
  - Camera init MUST follow the canonical lifecycle order: releaseCamera FIRST → getCameraManager → getSupportedCameras → pick rear device by cameraPosition CAMERA_POSITION_BACK (NOT hardcoded cameraId; on not-found fall back to cameras[0]) → createCameraInput + open → verify getSupportedSceneModes contains NORMAL_PHOTO → getSupportedOutputCapability(device, NORMAL_PHOTO) pick previewProfile (NEVER hand-build a Profile) → createPreviewOutput(profile, surfaceId) → createSession(NORMAL_PHOTO) → beginConfig → addInput → addOutput → commitConfig → start. Skipping release first or reordering causes 7400107/7400103/7400201.
  - On camera switch (front↔rear) or foldStatusChange you MUST release then fully rebuild — NEVER incrementally swap input/output on a running session (session.removeInput during a switch throws 7400201). Use the reloadXComponentFlag double-instance swap so the old session is released inside the new instance's onLoad before init.
  - surfaceId MUST be obtained inside XComponent.onLoad via getXComponentSurfaceId() and confirmed non-empty BEFORE createPreviewOutput — an empty/not-yet-generated surfaceId fails createPreviewOutput (401/7400101) → black screen.
  - Rear device may be ABSENT on wide-foldable outer screen (e.g. PuraX) or 2in1 — findIndex for CAMERA_POSITION_BACK must fall back to cameras[0], never createCameraInput on undefined.
  - HALF_FOLDED↔EXPANDED and HALF_FOLDED↔FOLDED transitions return an IDENTICAL camera list — SKIP rebuild there; only genuine camera-set changes trigger rebuild. Rebuilding on hover transitions causes flicker/failure.
  - Overlay controls (shutter, mode bar, zoom slider, etc.) MUST live in the SAME Stack as the XComponent preview and use responsive positioning (Blank / layoutWeight / width('100%') / justifyContent / aspectRatio) — NEVER absolute px offsets, because the preview area box changes across form/rotate/split and fixed coordinates will misalign
  - Any change to the preview-area box (form switch, rotation, split-screen, front↔rear profile ratio change) MUST trigger a relayout of the overlay controls — do not assume the control layer stays correct when the preview box changes
  - Overlay controls MUST avoid system safe areas via expandSafeArea (background fill) + getWindowAvoidArea / avoidAreaChange (content inset) — never let the shutter or mode bar sit under the status bar, nav indicator, or cutout
  - Rotation listening MUST pair windowSizeChange with display.on('change') — windowSizeChange does NOT fire on 180° rotation, so listening to only one leaves overlay controls in a stale layout after a half-turn
  - Rotation strategy is breakpoint-scoped: sm does not rotate, md/lg support rotation (window min dimension ≥ 600vp) — overlay layout must consider both portrait and landscape arrangements on md/lg
  - Foldable detection MUST use display.isFoldable() — NOT deviceInfo.deviceType (foldables report 'phone'); foldStatusChange is the signal that the preview box changed and overlay must relayout
  - During camera-switch transition, overlay UI MUST stay coherent with the reloadXComponentFlag double-instance swap (the XComponent swap itself is hardware-access scope; this skill only ensures controls don't dangle/stale during the swap window)

diagnostic_checklist:
  - Black screen after fold (expand↔fold) or front↔rear switch? Root cause: available camera set changed but session still bound to the old device.
  - On switch/fold, is the session fully released then rebuilt (not incrementally swapped via removeInput)? Is the rear device fallback to cameras[0] present?
  - Is foldStatusChange registered in aboutToAppear (cameraManager.on, which carries supportedCameras)? Is HALF_FOLDED↔EXPANDED rebuild skipped?
  - Error codes on fold/switch black screen: 7400107 (zombie session not released) / 7400201 (removeInput on running session, or device unavailable).
  - Are overlay controls in the same Stack as XComponent using responsive props, not hardcoded offsets?
  - Does every preview-box change (fold/rotate/split/camera switch) trigger an overlay relayout?
  - Do controls avoid status bar / nav indicator / cutout via expandSafeArea + getWindowAvoidArea?
  - Is rotation listened with BOTH windowSizeChange and display.on('change') (covers 180°)?
  - Is foldable form detected via display.isFoldable(), and foldStatusChange wired to relayout?
  - Does the overlay stay correct (no dangling/stale controls) across the camera-switch double-instance swap?
  - On md/lg does the overlay work in both portrait and landscape?
```

# 后置摄像头叠加 UI 多形态适配

## 适配领域

本 skill 解决**后置摄像头预览能正常显示**与**叠加在预览之上的 UI 跨形态正确布局**两类问题。两个问题域并列——一个是"有没有画面"，一个是"控件跑没跑位"。

具体覆盖：
- **Q1 折叠/前后摄切换后预览黑屏**：展开↔折叠、切到后摄/切回前摄后，可用相机变了但 session 没正确重建（绑在旧设备上）。见「后摄预览黑屏」。
- **Q2 叠加控件响应式布局**：拍照按钮、模式切换条、变焦滑块等控件与 XComponent 同 `Stack`，用响应式属性定位；预览区变化时控件随 Stack 自动重排。
- **Q3 安全区避让**：控件避让状态栏、导航指示条、挖孔，用 `expandSafeArea` + `getWindowAvoidArea` / `avoidAreaChange`。
- **Q4 形态切换后叠加 UI 刷新**：监听折叠/旋转/分屏/切相机事件，触发叠加控件重布局；切换过渡态配合双实例翻转。

**不覆盖**：
- 按相机**能力查询**驱动功能按钮启停（变焦范围/闪光灯/人脸检测/exposure/focus）—— 那是相机能力差异，本 skill 不管。
- 通用 HarmonyOS **一多布局教程**（断点/`GridRow`/媒体查询）—— 不在本 skill；本 skill 只在需要时一句话点到（见下方）。
- 图片编辑、滤镜算法。

---

## 核心范式

### A：预览能显示（黑屏的根治）

黑屏的根因几乎都在**相机生命周期**上：要么初始化顺序错，要么切换/折叠时没 release 就重建（僵尸会话冲突），要么 surfaceId 还没生成就建预览流，要么设备缺失没回退。根治方法是一条固定链：

```
切到后摄 = 一次完整生命周期：
  releaseCamera(先) → 枚举 cameras → 选后摄(cameraPosition BACK，缺失回退 cameras[0])
  → createCameraInput+open → getSupportedOutputCapability(device, NORMAL_PHOTO) 选 profile
  → createPreviewOutput(profile, surfaceId 非空) → createSession → beginConfig
  → addInput → addOutput → commitConfig → start
```

切换前后摄 / 折叠态变更 = 重新走整条链（release 在前），**不能在运行中的 session 上增量换 input/output**。`HALF_FOLDED↔EXPANDED` 镜头集不变，跳过重建。完整每步的 guard 与错误码见「后摄预览黑屏排查」与 `references/rear-blackscreen-reference.md`。

### B：形态驱动重布局（控件错乱的根治）

所有"叠加控件错乱"问题的根治方法是同一套范式：

```
1. 预览区那个矩形的尺寸/比例/位置发生变化
   → 触发源：折叠展开 / 旋转 / 分屏 / 前后摄切换（profile 比例不同）
2. 监听对应事件（foldStatusChange / windowSizeChange+display.on('change') / onAreaChange）
3. 叠加控件随 Stack 响应式重排（不要手动算坐标）
   - 用 Blank/layoutWeight/width('100%')/justifyContent/aspectRatio 等响应式属性
4. 避让安全区（expandSafeArea 背景 + getWindowAvoidArea 内容内缩）
5. 切换过渡态配合 reloadXComponentFlag 双实例翻转，控件不悬空
```

> **为什么不能写死坐标**：预览区域在折叠/旋转/分屏/切相机后尺寸与比例都会变。用绝对像素定位的控件，一旦预览区变了就会错位、被遮挡或飘到屏外。响应式属性让控件随容器自动重排，是唯一可靠的跨形态做法。
>
> 通用一多（断点 `GridRow`）用于「不同断点下控件排列结构本身要变」（如模式条在 sm 横排、在 lg 分栏）。这种结构性切换属通用一多范畴，不在本 skill；本 skill 聚焦相机叠加控件的响应式重排与避让。

---

## 后摄预览黑屏（折叠 / 前后摄切换场景）

折叠态切换、前后摄切换是两个典型的"可用相机变了"的一多场景。共同症状：**XComponent 可见但无画面（黑屏）**；共同根因：**可用相机设备变了，但 session 还绑在旧设备上没重建**。

| 场景 | 根因 | 关键错误码 | 修复 |
|------|------|------|------|
| 展开 → 折叠 后黑屏 | 折叠后 `getSupportedCameras()` 返回的设备集变了（如阔折叠外屏只剩前置），旧后摄设备已不可用，但 session 还在用旧设备 | 7400201 / 7400107 | 注册 `foldStatusChange`，收到后 `releaseCamera()` → 重新枚举选后摄（缺失回退 `cameras[0]`）→ 完整重建 session |
| 折叠 → 展开 后黑屏 | 同上，展开后设备集又变回来，但 session 没跟着重建 | 7400201 / 7400107 | 同上：foldStatusChange → release → 重新枚举重建 |
| 切到后摄 / 切回前摄 后黑屏 | 在运行中的 session 上**增量换** input/output（`removeInput`/`addInput`），旧设备没先 release（僵尸会话冲突） | 7400201（removeInput）/ 7400107（僵尸会话） | 不要增量换。先 `releaseCamera()` 释放整条链，再按目标 cameraPosition 完整重建 |

**关键约束**：
- 折叠监听须在 `aboutToAppear` 注册 `cameraManager.on('foldStatusChange', cb)`——回调直接带 `supportedCameras`，免再枚举。只监听 `display` 变化而没监听相机层 foldStatusChange 是常见漏点。
- 重建必须是**完整 release + 重建**，不能在运行中 session 上 `removeInput` 增量换（抛 7400201）。
- 后摄可能在某些折叠态缺失（阔折叠外屏 / 2in1 仅前置）：选后摄 `findIndex(CAMERA_POSITION_BACK) === -1` 时回退 `cameras[0]`，不要 `createCameraInput(undefined)`。
- `HALF_FOLDED`（悬停态）↔ `EXPANDED` / `FOLDED` 之间镜头集不变，**跳过重建**，否则不必要的重建反而闪黑/失败。

> 完整的 initCamera 生命周期（每步 guard）、releaseCamera 顺序、折叠/切换重建序列、后摄缺失回退代码见 `references/rear-blackscreen-reference.md`。

---

## 适配流程（叠加 UI 布局）

### 新适配

**第 1 步：叠加控件与 XComponent 同 Stack，用响应式属性**

XComponent 与拍照按钮/模式切换条/滑块放在同一个 `Stack`，控件用 `Blank()` / `layoutWeight` / `width('100%')` / `justifyContent` / `aspectRatio` 等响应式属性定位。预览区比例变化时，控件随 Stack 自动重排，无需手算位置。见「代码模式 1」。

**第 2 步：安全区避让**

控件避让状态栏、导航指示条、挖孔：背景层用 `expandSafeArea(true, true)` 撑满全屏，内容层按 `getWindowAvoidArea` / `avoidAreaChange` 取避让区域做内缩（`padding`）。否则快门/模式条会被系统栏遮挡。见「代码模式 1」的 padding 部分。

**第 3 步：接形态变化事件触发重排**

监听形态变化，在预览区变化后触发控件层重布局（通常是更新一个 `@State` 触发 ArkUI 重渲染）：
- 折叠：`display.isFoldable()` 判断 + `foldStatusChange` 监听。
- 旋转：`windowSizeChange` **配** `display.on('change')`（`windowSizeChange` 在 180° 旋转**不**触发，须双监听）。
- 预览区比例变化（含前后摄 profile 不同、`onSurfaceChanged`）：也是重排触发源——预览区几何本身属硬件适配范畴，但「它变了 → 控件要重排」由本 skill 负责。

建议对高频事件 debounce（100–200ms），避免抖动期间反复重排。见「代码模式 3」。

**第 4 步：切换过渡态配合双实例翻转**

切相机（前后摄/折叠态变更镜头）时，XComponent 用 `reloadXComponentFlag` 双实例翻转（实现属硬件适配范畴）。本 skill 要确保：**叠加控件层在翻转窗口内不悬空、不停留在上一个相机的布局**——通常把控件层放在双实例 `if/else` 之外的同级 `Stack` 层，使其独立于 XComponent 实例切换；或随 flag 同步刷新布局状态。见「代码模式 2」。

> **折叠态豁免**：折叠设备在半折叠态（`HALF_FOLDED`）与展开态（`EXPANDED`）之间切换时，若可用镜头集不变，预览区几何可能不变，叠加控件重排可跳过。只有真正改变预览区尺寸/比例的形态切换才触发重排。

---

### 问题定位

**现象 → 可能原因**：

| 现象 | 可能原因 | 定位 |
|------|---------|------|
| 折叠/旋转/分屏后拍照按钮/模式条错位 | 控件没用响应式布局，预览区变了但控件位置写死 | 检查控件是否在 Stack 内用响应式属性，是否绝对像素 |
| 控件被状态栏/导航条/挖孔遮挡 | 没避让安全区 | 检查是否用 expandSafeArea + getWindowAvoidArea 内缩 |
| 旋转 180° 后控件没重排 | 只监听了 windowSizeChange（180° 不触发） | 检查是否配了 display.on('change') 双监听 |
| 切相机后控件停留在旧布局 | 切换过渡态没刷新控件层，或控件层与 XComponent 实例绑死 | 检查控件层是否独立于双实例翻转、切换后是否刷新布局状态 |
| 大屏/展开后控件挤在一起或飘边 | 缺响应式占位（Blank/layoutWeight）或没用断点结构 | 检查是否用响应式属性 + 是否需 GridRow 断点结构 |

**3 步定位法**：
1. **收集现象**：是错位、被遮挡、还是切换后停留旧布局？在哪种形态（折叠/旋转/分屏/切相机）下复现？
2. **确认响应式**：相关控件是写死坐标，还是响应式定位？形态变化有没有触发重排？安全区避让了没？
3. **修复与验证**：按对应代码模式改为响应式 + 避让 + 事件触发，按「验证清单」反复切形态验证。

---

## 关键 API

### 相机生命周期（黑屏根治）

| API | 用途 | 关键点 |
|------|------|------|
| `cameraManager.getSupportedCameras()` | 枚举设备 | 切换/折叠后**重新枚举**，旧设备可能已不可用 |
| 选后摄：`d.cameraPosition === CAMERA_POSITION_BACK` | 选后置 | **按 cameraPosition 选，不硬编码 cameraId**；找不到回退 `cameras[0]` |
| `cameraManager.getSupportedSceneModes(device)` | 查支持场景 | 须含 `NORMAL_PHOTO` 才继续 |
| `cameraManager.getSupportedOutputCapability(device, NORMAL_PHOTO)` | 取 previewProfile | profile **从这里取，不能手造** |
| `cameraManager.createCameraInput(device)` + `input.open()` | 建输入 | open 异步，await |
| `cameraManager.createPreviewOutput(profile, surfaceId)` | 建预览流 | surfaceId 须**非空**（onLoad 取），空则 401/7400101 |
| `cameraManager.createSession(NORMAL_PHOTO)` | 建 session | cast 为 PhotoSession |
| `session.beginConfig()` → `addInput` → `addOutput` → `commitConfig()` → `start()` | 配置并启动 | beginConfig 后必须 commitConfig 才能再次 beginConfig（否则 7400105） |
| `releaseCamera()`：`stop` → `input.close` → `previewOutput.release` → `session.release` → 置 null | 释放 | **重建前必须完整 release**，否则 7400107 僵尸会话 |
| `cameraManager.on('foldStatusChange', cb)` | 折叠监听（推荐） | 回调带 `supportedCameras`，免再枚举 |
| `CameraInput.on('error')` / session error | 错误回调 | 重建失败/设备不可用错误由此送达，据此清理重建 |

> 完整生命周期顺序、guard 条件、各 API 为何此序、相关错误码见 `references/rear-blackscreen-reference.md`。

### 安全区避让

| API | 用途 |
|------|------|
| `expandSafeArea(edges?, enable?)` | 组件背景撑满到安全区外（沉浸式），常用于 XComponent 预览层 |
| `window.getLastWindow()` → `getWindowAvoidArea(type)` | 取指定 `AvoidAreaType` 的避让矩形（px），内容层据此内缩 |
| `window.on('avoidAreaChange')` / 窗口实例 `on('avoidAreaChange')` | 安全区变化（如键盘弹出、横竖屏系统栏变）时回调，重算内缩 |

`AvoidAreaType`：`TYPE_SYSTEM`（状态栏等）/ `TYPE_NAVIGATION_INDICATOR`（导航指示条）/ `TYPE_CUTOUT`（挖孔）/ `TYPE_KEYBOARD`（键盘）。叠加控件常需避让前三者。

> 细节与枚举速查见 `references/overlay-layout-reference.md`。

### 形态 / 窗口监听

| API | 用途 | 注意 |
|------|------|------|
| `display.isFoldable()` | 是否折叠设备 | **不要**用 `deviceInfo.deviceType`（折叠设备返回 `'phone'`） |
| `FoldStatus` / `foldStatusChange` / `foldDisplayMode` | 折叠状态与变化事件 | `HALF_FOLDED` 悬停态常为过渡，可豁免 |
| 窗口 `on('windowSizeChange')` | 窗口尺寸变化（分屏、部分旋转） | **180° 旋转不触发**，须配 `display.on('change')` |
| `display.on('change')` / `display.rotation` | 屏幕旋转，`rotation` 0/90/180/270 | 补 `windowSizeChange` 的 180° 盲区 |
| `onAreaChange` | 组件自身尺寸/位置变化 | 预览区比例变化时也可由此感知重排 |

> 高频事件建议 debounce 100–200ms。预览 Surface 的几何/旋转角度计算属硬件适配范畴，不在本 skill；本 skill 只用上述事件作为「触发控件重排」的信号。

---

## 代码模式

### 模式 A1：相机初始化生命周期骨架（黑屏根治）

切到后摄 = 一次完整生命周期。**release 在前，顺序不可乱**。下面是骨架；每步的 guard、错误处理、releaseCamera 完整顺序见 `references/rear-blackscreen-reference.md`。

```typescript
import { camera } from '@kit.CameraKit';

async function initRearCamera(
  context: Context,
  surfaceId: string,                // 必须来自 XComponent.onLoad 且非空
  cameraManager: camera.CameraManager
): Promise<void> {
  // 0. 先释放上一次会话（关键！否则僵尸会话冲突 7400107）
  await releaseCamera();            // stop→input.close→previewOutput.release→session.release→置 null

  // 1. 枚举设备，按 cameraPosition 选后摄（不硬编码 cameraId）
  const cameras = cameraManager.getSupportedCameras();
  if (cameras.length === 0) return;
  let idx = cameras.findIndex(d =>
    d.cameraPosition === camera.CameraPosition.CAMERA_POSITION_BACK &&
    d.connectionType === camera.CameraConnectionType.CAMERA_CONNECTION_BUILT_IN);
  if (idx === -1) idx = 0;          // 后摄缺失（PuraX外屏/2in1）→ 回退 cameras[0]

  // 2. 验证场景支持
  const modes = cameraManager.getSupportedSceneModes(cameras[idx]);
  if (!modes.includes(camera.SceneMode.NORMAL_PHOTO)) { await releaseCamera(); return; }

  // 3. 从能力查询取 previewProfile（不可手造）
  const cap = cameraManager.getSupportedOutputCapability(cameras[idx], camera.SceneMode.NORMAL_PHOTO);
  const profile = cap.previewProfiles[0];   // 生产应按目标比例/分辨率筛选

  // 4. 建输入 + open
  const cameraInput = cameraManager.createCameraInput(cameras[idx]);
  await cameraInput.open();

  // 5. 建预览流（surfaceId 必须非空，否则 401/7400101）
  const previewOutput = cameraManager.createPreviewOutput(profile, surfaceId);

  // 6. 建会话并配置（beginConfig→addInput→addOutput→commitConfig→start）
  const session = cameraManager.createSession(camera.SceneMode.NORMAL_PHOTO) as camera.PhotoSession;
  session.beginConfig();
  session.addInput(cameraInput);
  session.addOutput(previewOutput);
  await session.commitConfig();
  await session.start();
}
```

> **切换/折叠重建**：不要在运行中的 session 上 `removeInput` 增量换（抛 7400201）。正确做法是用 `reloadXComponentFlag` 翻转触发新 XComponent 实例，在其 `onLoad` 里先 `releaseCamera()`（旧）再 `initRearCamera()`（新）。`HALF_FOLDED↔EXPANDED` 镜头集不变，跳过重建。完整序列与各根因修复代码见 `references/rear-blackscreen-reference.md`。

---

### 模式 B1：响应式叠加布局 + 安全区避让

XComponent 与控件同 `Stack`，控件用响应式属性，内容层避让安全区：

```typescript
@Entry
@Component
struct CameraPage {
  // 安全区内缩量（vp），由 getWindowAvoidArea / avoidAreaChange 填充
  @State safeInsetTop: number = 0;
  @State safeInsetBottom: number = 0;
  @State safeInsetLeft: number = 0;
  @State safeInsetRight: number = 0;
  mXComponentController: XComponentController = new XComponentController();

  async updateSafeInset(windowStage?: window.Window) {
    const win = windowStage ?? await window.getLastWindow(this.getUIContext());
    // 避让状态栏 + 导航指示条 + 挖孔（按需组合）
    const sys = win.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM);
    const cutout = win.getWindowAvoidArea(window.AvoidAreaType.TYPE_CUTOUT);
    const nav = win.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
    this.safeInsetTop = Math.max(
      px2vp(sys.topRect.height), px2vp(cutout.topRect.height));
    this.safeInsetBottom = px2vp(nav.bottomRect.height);
    // 横屏时左右也常需避让
    this.safeInsetLeft = px2vp(cutout.leftRect.width);
    this.safeInsetRight = px2vp(cutout.rightRect.width);
  }

  build() {
    Stack() {
      // 预览层：背景撑满安全区（沉浸式）
      XComponent({ type: XComponentType.SURFACE, controller: this.mXComponentController })
        .width('100%').height('100%')
        .expandSafeArea(true, true)   // 预览贯穿到状态栏/导航条下
        .onLoad(() => { /* initCamera（硬件适配范畴）；注册 avoidAreaChange 见模式 3 */ })

      // 叠加控件层 —— 响应式定位，预览区变了自动重排
      Column() {
        // 顶部：模式切换条
        Row() { /* 模式按钮 */ }
        .width('100%').justifyContent(FlexAlign.Center)

        Blank()  // 弹性占位，把底部控件推下去（随容器高度自适应）

        // 底部控件
        Row() {
          // 拍照按钮、其它按钮 ...
        }
        .width('100%').justifyContent(FlexAlign.SpaceEvenly)
      }
      .width('100%').height('100%')
      // 内容内缩安全区：控件不压到状态栏/导航条/挖孔
      .padding({
        top: this.safeInsetTop,
        bottom: this.safeInsetBottom,
        left: this.safeInsetLeft,
        right: this.safeInsetRight,
      })
    }
  }
}
```

> 控件用 `Blank()` / `width('100%')` / `justifyContent` / `padding` 等响应式属性，预览区域比例变了会自动重布局，不需要手算坐标。背景撑满 + 内容内缩是安全区避让的标准组合。

### 模式 B2：切换过渡态配合 reloadXComponentFlag 双实例

切相机时 XComponent 用 `reloadXComponentFlag` 双实例翻转（翻转实现属硬件适配范畴）。**叠加控件层放在双实例 `if/else` 之外的同级 `Stack` 层**，使其独立于实例切换、不在翻转窗口悬空：

```typescript
@Entry
@Component
struct CameraPage {
  @State reloadXComponentFlag: boolean = false;
  @State switching: boolean = false;  // 切换过渡态标记
  mXComponentController: XComponentController = new XComponentController();
  mXComponentOptions: XComponentOptions = {
    type: XComponentType.SURFACE,
    controller: this.mXComponentController
  }

  // 触发相机切换：翻转 flag（重建 XComponent 由硬件适配负责）
  reloadXComponent() {
    this.switching = true;
    this.reloadXComponentFlag = !this.reloadXComponentFlag;
  }

  onSessionStarted() {
    // 新 session start() 成功后撤掉过渡态
    this.switching = false;
  }

  build() {
    Stack() {
      // 双 XComponent 实例：if/else 各一个（重建/防残影属硬件适配范畴）
      if (this.reloadXComponentFlag) {
        XComponent(this.mXComponentOptions)
          .onLoad(() => { /* initCamera */ })
          .width('100%').height('100%')
          .expandSafeArea(true, true)
      } else {
        XComponent(this.mXComponentOptions)
          .onLoad(() => { /* initCamera */ })
          .width('100%').height('100%')
          .expandSafeArea(true, true)
      }

      // 叠加控件层 —— 独立于双实例，翻转期间不悬空
      Column() {
        Row() { /* 模式条 */ }.width('100%').justifyContent(FlexAlign.Center)
        Blank()
        Row() { /* 拍照按钮 */ }.width('100%').justifyContent(FlexAlign.SpaceEvenly)
      }
      .width('100%').height('100%')
      .padding({ top: this.safeInsetTop, bottom: this.safeInsetBottom })

      // 切换过渡遮罩（可选）：新 session start 成功后撤掉
      if (this.switching) {
        Column() {
          LoadingProgress().width(48).height(48)
        }
        .width('100%').height('100%')
        .justifyContent(FlexAlign.Center)
        .backgroundColor('rgba(0,0,0,0.4)')
      }
    }
  }
}
```

> 关键：控件层与 XComponent 实例解耦。无论哪个 XComponent 实例在显示，控件层始终正确叠加；`switching` 标记保证过渡期不响应陈旧操作。`reloadXComponentFlag` 双实例翻转本身（防上一个相机画面残影）属硬件适配范畴。

### 模式 B3：形态变化触发重排

监听折叠/旋转/窗口变化，debounce 后更新触发 ArkUI 重渲染的状态（如安全区内缩、布局标记）：

```typescript
import { display } from '@kit.ArkUI';

@Entry
@Component
struct CameraPage {
  @State orientation: 'portrait' | 'landscape' = 'portrait';
  @State isFoldable: boolean = false;
  private resizeTimer: number = -1;
  private win: window.Window | null = null;

  aboutToAppear() {
    this.isFoldable = display.isFoldable();  // 折叠检测：不要用 deviceInfo.deviceType
    this.setupListeners();
  }

  async setupListeners() {
    this.win = await window.getLastWindow(this.getUIContext());

    // 1) 窗口尺寸变化（分屏、部分旋转）—— 注意 180° 旋转不触发
    this.win.on('windowSizeChange', (size) => {
      this.scheduleRelayout();
    });

    // 2) 屏幕旋转（补 180° 盲区）—— windowSizeChange 单独不够
    display.on('change', () => {
      const r = display.getDefaultDisplaySync().rotation;  // 0/90/180/270
      this.orientation = (r === 1 || r === 3) ? 'landscape' : 'portrait';
      this.scheduleRelayout();
    });

    // 3) 安全区变化（系统栏/键盘）
    this.win.on('avoidAreaChange', () => {
      this.updateSafeInset(this.win ?? undefined);
    });
  }

  // debounce：折叠/旋转过程会连续触发，避免抖动期反复重排
  private scheduleRelayout() {
    if (this.resizeTimer !== -1) clearTimeout(this.resizeTimer);
    this.resizeTimer = setTimeout(() => {
      this.updateSafeInset(this.win ?? undefined);
      // 触发 ArkUI 重渲染：@State 变化 → 控件层自动重排
      this.resizeTimer = -1;
    }, 150) as unknown as number;
  }

  aboutToDisappear() {
    display.off('change');
    this.win?.off('windowSizeChange');
    this.win?.off('avoidAreaChange');
  }

  build() {
    // 见模式 B1 的 Stack 结构；orientation 可用于切换横竖屏两套排列
    // ...
  }
}
```

> `windowSizeChange` 在 180° 旋转不触发是常见坑——必须配 `display.on('change')`。高频事件 debounce 100–200ms 防抖动。

---

## 验证清单

**预览能显示（折叠 / 切换黑屏）**：

- [ ] 展开→折叠 后无黑屏（foldStatusChange 注册、release 后完整重建）
- [ ] 折叠→展开 后无黑屏（同上）
- [ ] 切到后摄 / 切回前摄 后无黑屏（release 整条链后完整重建，非增量 removeInput）
- [ ] 后摄缺失设备（PuraX 外屏/2in1）回退 cameras[0]，不崩
- [ ] 半折叠↔展开不闪黑（HALF_FOLDED 镜头集不变，豁免重建）

**叠加 UI 布局（每次必做）**：

- [ ] 折叠展开后，拍照按钮/模式切换条/滑块位置正确，不被遮挡
- [ ] 旋转 0°/90°/180°/270° 四个方向，叠加控件都正确（尤其 180°）
- [ ] 分屏后窗口尺寸变化，控件随预览区重排，不错位
- [ ] 控件避让状态栏/导航指示条/挖孔（expandSafeArea + getWindowAvoidArea）
- [ ] 切相机后叠加控件层不悬空、不停留在旧布局

**压力验证（发布前做）**：

- [ ] 反复展开↔折叠、切前后摄，预览每次都正确重建无黑屏
- [ ] 快速反复折叠展开/旋转，控件每次都正确同步（无残留旧布局，debounce 生效）
- [ ] sm 断点设备不旋转行为符合预期；md/lg 横竖屏两套排列都正确
- [ ] 大屏/展开态控件不挤作一团、不飘出屏外（响应式占位生效）
- [ ] 安全区动态变化（如键盘弹出、横竖屏系统栏高度变）时内缩正确更新

---

## 常见问题

**Q：展开↔折叠 后预览黑屏？**
A：折叠态变化后，`getSupportedCameras()` 返回的设备集变了，旧后摄设备可能已不可用，但 session 还绑在旧设备上 → 无画面。须在 `aboutToAppear` 注册 `cameraManager.on('foldStatusChange', cb)`（回调直接带 `supportedCameras`），收到折叠变化后 `releaseCamera()` → 按新设备集重新选后摄（缺失回退 `cameras[0]`）→ 完整重建 session。`HALF_FOLDED↔EXPANDED/FOLDED` 镜头集不变，跳过重建。

**Q：切到后摄 / 切回前摄 后黑屏？**
A：在运行中的 session 上用 `removeInput`/`addInput` 增量换设备会抛 7400201；旧设备没先 release 会留僵尸会话冲突 7400107。正确做法：先 `releaseCamera()` 释放整条链，再按目标 `cameraPosition` 完整重建。用 `reloadXComponentFlag` 双实例翻转触发新实例 onLoad，在其中先 release 旧 session 再 init 新的。

**Q：阔折叠外屏（PuraX）/2in1 折叠后选后摄就崩？**
A：这些形态后摄不存在。`findIndex(CAMERA_POSITION_BACK)` 得 -1 时回退 `cameras[0]`，不要 `createCameraInput(undefined)`。

---

**Q：折叠展开后拍照按钮/模式条错位了？**
A：控件大概率用了写死坐标或绝对像素。把它们和 XComponent 放同一个 `Stack`，用 `Blank()` / `layoutWeight` / `width('100%')` / `justifyContent` 等响应式属性定位，预览区变化时控件随容器自动重排。同时监听 `foldStatusChange` 触发重渲染。

**Q：旋转 180° 后控件没重排，其它角度都正常？**
A：经典坑。`windowSizeChange` 在 180° 旋转时**不**触发（宽高没变）。必须再配一个 `display.on('change')`，用它读 `display.rotation`（0/90/180/270）判断朝向并触发重排。两个监听缺一不可。

**Q：控件被状态栏/挖孔/导航条挡住了？**
A：没避让安全区。预览背景层用 `expandSafeArea(true, true)` 撑满（沉浸式），内容控件层用 `getWindowAvoidArea(TYPE_SYSTEM/TYPE_CUTOUT/TYPE_NAVIGATION_INDICATOR)` 取避让矩形做 `padding` 内缩，并监听 `avoidAreaChange` 在安全区变化时更新。注意折叠检测用 `display.isFoldable()`，不是 `deviceInfo.deviceType`。

**Q：切相机后叠加控件还停在上一个相机的位置/状态？**
A：控件层与 XComponent 实例绑太紧了。把控件层放在双 XComponent `if/else` 之外的同级 `Stack` 层，使其独立于 `reloadXComponentFlag` 翻转；用一个 `switching` 过渡标记遮挡切换窗口，新 session `start()` 成功后撤掉并刷新布局状态。（XComponent 双实例翻转本身属硬件适配范畴。）

**Q：大屏/展开后控件挤在一起或飘到边上？**
A：缺响应式占位。用 `Blank()` 弹性占位或 `layoutWeight` 分配空间；若需要不同断点下控件排列结构本身变化（如模式条横排→分栏），用 `GridRow`/断点，属通用一多范畴、不在本 skill。

**Q：预览区比例变化（前后摄 profile 不同）对叠加 UI 有什么影响？**
A：预览区那个矩形一旦变了（比例变化是触发源之一），叠加控件要跟着重排——用模式 B1 的响应式布局即可自动应对。注意区分：预览画面**拉伸变形**（Surface 宽高比不对）是另一类问题，按黑屏排查表里的 surfaceId/分辨率根因处理；本 skill 管的是**控件怎么跟着预览区重排**。

---

## 延伸阅读

- **后摄预览黑屏（折叠/切换）实现细节**（initCamera 生命周期与每步 guard、releaseCamera 顺序、折叠重建序列、前后摄切换重建序列、后摄缺失回退、相关错误码 7400107/7400201）见 `references/rear-blackscreen-reference.md`。
- **叠加 UI 布局 API 速查**（安全区 `expandSafeArea`/`getWindowAvoidArea`/`avoidAreaChange`/`AvoidAreaType`、形态监听 `display.isFoldable`/`FoldStatus`/`windowSizeChange`/`display.on('change')`/`onAreaChange`、响应式布局属性 Stack/Blank/layoutWeight/aspectRatio）见 `references/overlay-layout-reference.md`。

> 以下主题不在本 skill 范围：按相机能力查询驱动功能按钮启停（变焦范围/闪光灯/人脸检测/exposure/focus）、通用一多布局（断点/`GridRow`/`GridCol`）、图片编辑与滤镜算法。
