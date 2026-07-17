# 后摄折叠/切换黑屏 实现细节

> 本文件为 `SKILL.md`「后摄预览黑屏（折叠 / 前后摄切换场景）」的补充，给出重建所需的完整相机生命周期、release 顺序、折叠/切换重建序列、后摄缺失回退、相关错误码。SKILL.md 讲场景与定位，本文件讲实现细节。
>
> **范围限定**：只覆盖「展开↔折叠」「前后摄切换」两类"可用相机变了但 session 没重建"导致的黑屏。其它黑屏根因（权限、surfaceId 时机、被抢占、旋转第二次、双路预览、服务崩溃等）不在本 skill。

---

## 为什么折叠/切换会黑屏

折叠态变化、前后摄切换，都会让 `cameraManager.getSupportedCameras()` 返回**不同的设备集**：
- 折叠：阔折叠外屏（如 PuraX）只剩前置；2in1 内置仅前置；折叠/展开态后摄的 connectionType 可能变。
- 切换：前后摄是不同 `CameraDevice`（`cameraPosition` 不同）。

如果 session 还绑在旧设备上（旧 `CameraInput` 没释放），而旧设备在新形态下已不可用，预览流就收不到帧 → XComponent 可见但**无画面（黑屏）**。根治方法：**监听变化 → 释放整条链 → 按新设备集重建**。

---

## 相机初始化生命周期（重建的基础）

一次完整 init 的顺序（release 必须在前，顺序不可乱）：

| 步 | API | 关键点 / guard |
|----|-----|------|
| 0 | `releaseCamera()`（先） | 释放上一次会话，否则僵尸会话冲突 7400107 |
| 1 | `cameraManager.getSupportedCameras()` | 重新枚举——**切换/折叠后设备集变了**，不能缓存 |
| 2 | 选后摄：`cameras.findIndex(d => d.cameraPosition === CAMERA_POSITION_BACK && d.connectionType === CAMERA_CONNECTION_BUILT_IN)` | **按 cameraPosition 选，不硬编码 cameraId**；`-1` 回退 `cameras[0]` |
| 3 | `cameraManager.getSupportedSceneModes(device)` | 须含 `NORMAL_PHOTO`，否则 releaseCamera() 并 return |
| 4 | `cameraManager.getSupportedOutputCapability(device, NORMAL_PHOTO)` → 取 `previewProfiles[...]` | profile **从这里取，不能手造** |
| 5 | `cameraManager.createCameraInput(device)` + `await input.open()` | open 异步 |
| 6 | `cameraManager.createPreviewOutput(profile, surfaceId)` | surfaceId 须来自 XComponent.onLoad 且非空 |
| 7 | `cameraManager.createSession(NORMAL_PHOTO)` as PhotoSession | — |
| 8 | `session.beginConfig()` → `addInput(input)` → `addOutput(previewOutput)` → `await commitConfig()` | beginConfig 后必须 commitConfig 才能再次 beginConfig（否则 7400105） |
| 9 | `await session.start()` | 启动预览 |

```typescript
import { camera } from '@kit.CameraKit';

async function initRearCamera(
  cameraManager: camera.CameraManager,
  surfaceId: string,
  cameraPosition: camera.CameraPosition = camera.CameraPosition.CAMERA_POSITION_BACK,
): Promise<{ session: camera.PhotoSession; input: camera.CameraInput; previewOutput: camera.PreviewOutput } | null> {
  // 0. 先释放上一次会话（关键，否则 7400107）
  await releaseCamera();

  // 1. 重新枚举
  const cameras = cameraManager.getSupportedCameras();
  if (cameras.length === 0) return null;

  // 2. 按 cameraPosition 选后摄，缺失回退 cameras[0]
  let idx = cameras.findIndex(d =>
    d.cameraPosition === cameraPosition &&
    d.connectionType === camera.CameraConnectionType.CAMERA_CONNECTION_BUILT_IN);
  if (idx === -1) idx = 0;
  const device = cameras[idx];

  // 3. 验证场景支持
  const modes = cameraManager.getSupportedSceneModes(device);
  if (!modes.includes(camera.SceneMode.NORMAL_PHOTO)) { await releaseCamera(); return null; }

  // 4. profile 从能力查询取
  const cap = cameraManager.getSupportedOutputCapability(device, camera.SceneMode.NORMAL_PHOTO);
  const profile = cap.previewProfiles[0];   // 生产应按目标比例/分辨率筛选

  // 5-6. 建输入 + 预览流
  const input = cameraManager.createCameraInput(device);
  await input.open();
  const previewOutput = cameraManager.createPreviewOutput(profile, surfaceId);

  // 7-9. 建会话并配置启动
  const session = cameraManager.createSession(camera.SceneMode.NORMAL_PHOTO) as camera.PhotoSession;
  session.beginConfig();
  session.addInput(input);
  session.addOutput(previewOutput);
  await session.commitConfig();
  await session.start();

  return { session, input, previewOutput };
}
```

---

## releaseCamera 顺序

重建前必须完整释放，顺序：

```typescript
let session: camera.PhotoSession | null;
let input: camera.CameraInput | null;
let previewOutput: camera.PreviewOutput | null;

async function releaseCamera(): Promise<void> {
  try { if (session) await session.stop(); } catch (e) {}
  try { if (input) await input.close(); } catch (e) {}
  try { if (previewOutput) await previewOutput.release(); } catch (e) {}
  try { if (session) await session.release(); } catch (e) {}
  session = null; input = null; previewOutput = null;
}
```

> **为什么 stop 在前、release 在后**：stop 停止数据流，close/release 释放资源。逆序或漏一步都会留僵尸会话 → 下次 init 报 7400107。

---

## 折叠重建序列（展开↔折叠黑屏的修复）

```typescript
import { camera } from '@kit.CameraKit';

@Entry
@Component
struct CameraPage {
  @State reloadXComponentFlag: boolean = false;
  mXComponentController: XComponentController = new XComponentController();
  private cameraManager: camera.CameraManager | null = null;
  private mSurfaceId: string = '';

  aboutToAppear() {
    this.cameraManager = camera.getCameraManager(this.getUIContext());
    // 关键：注册相机层 foldStatusChange（回调带 supportedCameras）
    this.cameraManager.on('foldStatusChange', (err, foldStatusInfo: camera.FoldStatusInfo) => {
      if (err) return;
      this.onFoldStatusChange(foldStatusInfo.foldStatus);
    });
  }

  private onFoldStatusChange(foldStatus: camera.FoldStatus) {
    // 豁免：半折叠↔展开/折叠 镜头集不变，跳过重建
    if (foldStatus === camera.FoldStatus.HALF_FOLDED) return;  // 视实际态机调整判定
    // 真正的相机集变化 → 翻转 XComponent 触发重建
    this.reloadXComponent();
  }

  reloadXComponent() {
    this.reloadXComponentFlag = !this.reloadXComponentFlag;
  }

  build() {
    Stack() {
      if (this.reloadXComponentFlag) {
        XComponent({ type: XComponentType.SURFACE, controller: this.mXComponentController })
          .onLoad(() => this.loadXComponent())
          .width('100%').height('100%')
      } else {
        XComponent({ type: XComponentType.SURFACE, controller: this.mXComponentController })
          .onLoad(() => this.loadXComponent())
          .width('100%').height('100%')
      }
      // 叠加控件层（见 overlay-layout-reference.md）...
    }
  }

  async loadXComponent() {
    this.mSurfaceId = this.mXComponentController.getXComponentSurfaceId();
    // onLoad 内：先 release 旧 session，再 init 新的（完整生命周期）
    await initRearCamera(this.cameraManager!, this.mSurfaceId);
  }
}
```

> **为什么用 `cameraManager.on('foldStatusChange')` 而非 `display.on`**：相机层的 foldStatusChange 回调直接带 `foldStatusInfo.supportedCameras`，免再 `getSupportedCameras()` 一次。只监听 display 变化而漏掉相机层监听，是折叠黑屏的最常见漏点。

---

## 前后摄切换重建序列

用户主动切前后摄，机制与折叠相同（可用相机变了），同样走 release + 完整重建：

```typescript
switchToFront() {
  // 不要在运行中 session 上 removeInput 增量换（抛 7400201）
  // 翻转 XComponent → onLoad → releaseCamera(旧) → initRearCamera(新 cameraPosition)
  this.reloadXComponentFlag = !this.reloadXComponentFlag;
}

async loadXComponent() {
  this.mSurfaceId = this.mXComponentController.getXComponentSurfaceId();
  // initRearCamera 内部已先 releaseCamera()，再按当前 cameraPosition 完整重建
  await initRearCamera(this.cameraManager!, this.mSurfaceId, this.currentTargetPosition);
}
```

> **为什么不能增量换**：`session.removeInput` / `addInput` 在运行中的 session 上操作会抛 7400201（"切换时未先释放相机流就直接移除输入"）。安全路径只有 release + 完整重建。

---

## 后摄缺失回退

阔折叠外屏（PuraX）、2in1 等形态后摄可能不存在。`findIndex` 得 `-1` 时必须回退：

```typescript
let idx = cameras.findIndex(d =>
  d.cameraPosition === camera.CameraPosition.CAMERA_POSITION_BACK &&
  d.connectionType === camera.CameraConnectionType.CAMERA_CONNECTION_BUILT_IN);
if (idx === -1) idx = 0;          // 后摄缺失 → 用第一个可用（通常是前置）
// 绝不能 createCameraInput(cameras[-1] / undefined) → 崩溃
const device = cameras[idx];
```

> `cameraPosition` 区分前后摄，**不要**用 `cameraType`（那是镜头类型 wide/telephoto 等，没有 FRONT/BACK 值）。

---

## 相关错误码

只列折叠/切换黑屏相关的：

| 码 | 含义 | 触发场景 | 处理 |
|----|------|------|------|
| 7400107 | CONFLICT_CAMERA | 上一次会话没 release（僵尸会话）/ 被别的应用占用 | 确保 releaseCamera 完整执行；跨进程协调 |
| 7400201 | SERVICE_FATAL_ERROR | 在运行中 session 上 `removeInput` 增量换设备；设备不可用仍建会话 | 切换前先 release 整条链再重建；设备缺失回退 |

> 错误回调：`CameraInput.on('error')` 与 session error 事件。重建失败/设备不可用错误由此送达，据此清理并重试。

---

## 生命周期监听清理

折叠监听须在 `aboutToDisappear` 取消：

```typescript
aboutToDisappear() {
  this.cameraManager?.off('foldStatusChange');
  releaseCamera();
}
```

> 不清理会导致页面销毁后折叠事件仍回调，对已释放组件操作 → 崩溃。
