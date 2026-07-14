# 手写笔适配 API 完整参考

> 本文件为 `SKILL.md` 的补充，记录笔刷枚举、笔宽配置、Controller 方法、图形类型、原生接口、响应区枚举与完整错误码。

---

## 系统能力检测（canIUse）

使用任何 Pen Kit 能力前，必须先用 `canIUse` 检测对应系统能力：

| 能力 | 系统能力 | 起始版本 | 说明 |
|------|---------|---------|------|
| 手写套件/报点预测/一笔成形 | `SystemCapability.Stylus.Handwrite` | 5.0.0(12) | 笔尖触屏绘制相关 |
| 全局取色 | `SystemCapability.Stylus.ColorPicker` | 5.0.0(12) | 全屏取色 |
| 手写交互 | `SystemCapability.Stylus.StylusService` | 5.1.1(19) | 笔身硬件事件 |

```typescript
if (!canIUse('SystemCapability.Stylus.Handwrite')) {
  console.warn('当前设备不支持手写能力，降级为普通触摸交互')
}
```

---

## 手写套件工具栏功能

HandwriteComponent 集成后获得开箱即用的画布与工具栏，默认已开启报点预测与一笔成形。

**画布**：笔迹绘制、笔迹保存、画布缩放、一笔成形（内置）。

**工具栏**：
- **笔刷**：圆珠笔、钢笔、铅笔、马克笔、荧光笔、马赛克笔、激光笔 7 种，5 档笔宽，100+ 颜色。
- **橡皮擦**：笔划擦除、像素擦除、仅擦除荧光笔、清空画布。
- **套索**：框选、移动、剪切粘贴、复制粘贴、删除、调整大小。
- **预置图形工具**（26.0.0+）：矩形、圆形、箭头、直线，线条粗细 1-10，默认 4。
- **手写波轮**（26.0.0+）：环形工具栏，轻捏笔身隐藏原工具栏并展开环形工具栏，再次轻捏切回。
- **其他**：撤销、重做、禁止手指书写。

> 套件仅支持上下滑动，不支持左右滑动。

### 套件 Context 初始化

套件依赖全局 Context，需在 EntryAbility 中设置，并通过 GlobalContext 暴露给页面：

```typescript
// EntryAbility
import { UIAbility } from '@kit.AbilityKit'
import { window } from '@kit.ArkUI'
import GlobalContext from '../utils/ContextConfig'

export default class EntryAbility extends UIAbility {
  onWindowStageCreate(windowStage: window.WindowStage): void {
    windowStage.loadContent('pages/HandWritingDemo', (err) => {
      if (err.code) { return }
    })
    GlobalContext.setContext(this.context)  // 存入全局供套件页面读取
  }
}
```

```typescript
// GlobalContext 单例
import { common } from '@kit.AbilityKit'
declare namespace globalThis {
  let _brushEngineContext: common.UIAbilityContext
}
export default class GlobalContext {
  static getContext(): common.UIAbilityContext { return globalThis._brushEngineContext }
  static setContext(context: common.UIAbilityContext): void { globalThis._brushEngineContext = context }
}
```

---

## PenType 笔刷枚举

| 名称 | 值 | 说明 |
|------|---|------|
| `PEN` | 1 | 钢笔 |
| `BALLPOINT_PEN` | 2 | 圆珠笔 |
| `PENCIL` | 3 | 铅笔 |
| `MARKER` | 4 | 马克笔 |
| `HIGHLIGHTER_BRUSH` | 5 | 荧光笔 |
| `MOSAIC` | 7 | 马赛克笔 |
| `RUBBER` | 8 | 橡皮擦 |
| `LASSO` | 9 | 套索 |
| `LASER` | 10 | 激光笔 |

---

## PenHspInfo 笔宽默认值与范围

| 笔刷 | 默认笔宽 | 取值范围 |
|------|---------|---------|
| 钢笔 PEN | 4 | 2-10 |
| 圆珠笔 BALLPOINT_PEN | 4 | 2-10 |
| 铅笔 PENCIL | 8 | 4-20 |
| 马克笔 MARKER | 32 | 16-80 |
| 荧光笔 HIGHLIGHTER_BRUSH | 32 | 16-80 |
| 马赛克笔 MOSAIC | 40 | 8-40 |
| 橡皮擦 RUBBER | 50 | 10-120 |
| 激光笔 LASER | 20 | 8-48 |

---

## HandwriteComponent 参数

| 参数 | 类型 | 必填 | 起始版本 | 说明 |
|------|------|------|---------|------|
| `handwriteController` | HandwriteController | 是 | 5.0.0(12) | 套件功能类实例 |
| `onInit` | () => void | 否 | 5.1.0(18) | 初始化完成回调，此时可调用 load |
| `onScale` | (scale: number) => void | 否 | 5.1.0(18) | 画布缩放回调 |
| `defaultPenType` | PenType | 否 | 5.1.0(18) | 默认笔刷 |
| `defaultPenInfo` | PenHspInfo[] | 否 | 5.1.0(18) | 各笔刷默认宽度 |
| `widthRatio` | number | 否 | 6.0.0(20) | 画布宽度占比 [0,1]，默认 1 |
| `heightRatio` | number | 否 | 6.0.0(20) | 画布高度占比 [0,1]，默认 1 |
| `maxCanvasHeight` | number | 否 | 6.1.0(23) | 长画布最大高度(vp) |
| `scaleDisabled` | boolean | 否 | 6.1.0(23) | 是否禁用缩放，默认 false |
| `onDidScroll` | (yOffset: number) => void | 否 | 6.1.0(23) | 画布滚动回调(vp) |

> Controller 方法需在 `onInit` 回调后或自定义方法中调用，不可在构造时直接调用。

---

## HandwriteController 方法

| 方法 | 签名 | 起始版本 | 说明 |
|------|------|---------|------|
| `load` | (path: string): void | 5.0.0(12) | 加载笔记文件；路径不存在则新建 |
| `save` | (path: string): Promise\<void\> | 5.0.0(12) | 保存笔记，Promise 异步 |
| `onLoad` | (callback: AsyncCallback\<string\>): void | 5.0.0(12) | 注册加载完成回调 |
| `getContentRange` | (): Rect | 6.0.0(20) | 获取笔迹矩形范围 |
| `getThumbnail` | (rect: Rect): Promise\<PixelMap\> | 6.0.0(20) | 获取缩略图 |
| `scrollTo` | (yOffset: number): void | 6.1.0(23) | 设置长画布滚动位置(vp) |

**Rect 结构**：`{ left, top, right, bottom }`，6.0.0(20) 起可用。

**onLoad 回调说明**：加载成功时 `err.message` 为 `load success`，加载失败时为 `load failed`；回调参数 `data` 为加载的路径。

---

## PointPredictor（报点预测）

| 方法 | 签名 | 起始版本 | 说明 |
|------|------|---------|------|
| `getPredictionPoint` | (event: TouchEvent): TouchPoint | 5.0.0(12) | 传入当前触摸事件，返回预测点信息 |

返回值 `TouchPoint` 含 `x`、`y` 坐标字段，可直接用于提前绘制。系统能力 `SystemCapability.Stylus.Handwrite`，仅 Stage 模型。

使用要点：在 `onTouch` 的 `TouchType.Move` 分支调用；`TouchType.Down` 时新建路径，`TouchType.Up` 时结束路径；预测点用于提前绘制，实际点到达后以实际点为准。

> 手写套件已默认开启报点预测，仅自定义 Canvas 场景才单独集成。

---

## imageFeaturePicker（全局取色）

### pickForResult 重载

| 签名 | 起始版本 | 说明 |
|------|---------|------|
| `pickForResult(x?: number, y?: number): Promise<PickedColorInfo>` | 5.0.0(12) | 启动取色器，移动时不显示 RGB 值 |
| `pickForResult(x?: number, y?: number, showValue?: boolean): Promise<PickedColorInfo>` | 5.1.0(18) | 启动取色器，`showValue=true` 时实时显示 RGB 值 |

`x`、`y` 为取色器初始位置(vp)，默认 100，超出屏幕范围取屏幕实际宽高。系统能力 `SystemCapability.Stylus.ColorPicker`，仅 Stage 模型。

### PickedColorInfo

| 字段 | 类型 | 说明 |
|------|------|------|
| `color` | common2D.Color | 色值 |
| `colorSpace` | colorSpaceManager.ColorSpace | 色域空间 |
| `timestamp` | number | 时间戳，自系统启动以来经过的时间(ms) |

> 全局取色要求设备支持手写笔。支持设备：Tablet、PC/2in1；从 5.1.1(19) 起新增 Phone。未返回结果前不可重复调用（否则报 1013900004）。

---

## stylusInteraction（手写交互）

### 接口

| 方法 | 签名 | 说明 |
|------|------|------|
| `on` | (type: 'squeeze', receiver: Callback\<SqueezeEvent\>): void | 监听笔身轻捏事件 |
| `off` | (type: 'squeeze', receiver?: Callback\<SqueezeEvent\>): void | 取消监听轻捏；不传 receiver 则取消所有 |
| `on` | (type: 'doubleTap', receiver: Callback\<DoubleTapEvent\>): void | 监听笔身双击事件 |
| `off` | (type: 'doubleTap', receiver?: Callback\<DoubleTapEvent\>): void | 取消监听双击；不传 receiver 则取消所有 |

起始版本 5.1.1(19)，系统能力 `SystemCapability.Stylus.StylusService`，仅 Stage 模型。

> `off` 取消单个监听时，传入的 receiver 必须与 `on` 注册时是**同一个回调引用**。匿名函数每次创建都是新引用，需以成员变量保存回调。

### 事件对象

**SqueezeEvent**

| 字段 | 类型 | 说明 |
|------|------|------|
| `timestamp` | number | 时间戳，自系统启动以来经过的时间(ms) |

**DoubleTapEvent**

| 字段 | 类型 | 说明 |
|------|------|------|
| `timestamp` | number | 时间戳，自系统启动以来经过的时间(ms) |

---

## 底层笔触事件枚举

### SourceTool（事件来源工具）

`TouchEvent.sourceTool` 字段标识当前触摸事件来源工具，自定义绘制时必须据此分流：

| 值 | 说明 |
|------|------|
| `SourceTool.Finger` | 手指触摸 |
| `SourceTool.Pen` | 手写笔 |
| `SourceTool.MOUSE` | 鼠标 |

### TouchType（触摸事件类型）

手写笔按下/移动/抬起与手指触摸类型一致：

| 值 | 说明 |
|------|------|
| `TouchType.Down` | 手写笔按下 |
| `TouchType.Move` | 手写笔移动 |
| `TouchType.Up` | 手写笔抬起 |
| `TouchType.Cancel` | 取消（手指按住屏幕时手写笔介入，或折叠屏切换等触发） |

> `TouchType.Cancel` 在手指触摸期间手写笔介入、或折叠屏切屏等场景触发，绘制逻辑需在此清理当前路径。

### pressure（压感）

`TouchObject.pressure` 表示按压压力值，起始版本 15+：

| 场景 | 取值范围 | 说明 |
|------|---------|------|
| 手写笔 | [0, 1) | 归一化值，压力越大值越大 |
| 通用（含手指） | [0, 65535) | 压力越大值越大 |

> `pressure` 可能为 `undefined`（设备不支持压感或非 15+ 版本），使用前**必须判空**，undefined 时回退默认线宽。

### getHistoricalPoints（历史报点）

`TouchEvent.getHistoricalPoints(): Array<HistoricalPoint>` 返回当前帧所有历史点（起始版本 10+）。不同设备触摸上报频率不同，一帧 `onTouch` 通常只回调一次，若该帧收到多个 TouchEvent，最后一个点通过 onTouch 返回，其余作为历史点。

**HistoricalPoint 结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `touchObject` | TouchObject | 历史点的触摸信息（含 x/y/pressure 等） |
| `timestamp` | number | 时间戳(ns) |
| `force` | number | 压力值 |
| `size` | number | 触摸面积 |

> 历史报点补全"过去点"，报点预测预测"未来点"，二者可配合使用。

---

## ShapeInfo 图形类型（shapeType）

| shapeType | 图形 |
|-----------|------|
| 0 | 未知（识别失败） |
| 1 | 直线 |
| 2 | 圆（椭圆） |
| 3 | 折线 |
| 6 | 矩形 |
| 7 | 平行四边形 |
| 9 | 菱形 |
| 10 | 等腰三角形 |
| 11 | 等边三角形 |
| 12 | 五角星形 |
| 13 | 正五边形 |
| 14 | 抛物线 |
| 15 | 直线单向箭头（指向起点） |
| 16 | 直线单向箭头（指向终点） |
| 17 | 直线双向箭头 |
| 18 | 抛物线单向箭头（指向起点） |
| 19 | 抛物线单向箭头（指向终点） |
| 20 | 抛物线双向箭头 |

> 一笔成形仅支持规则几何图形，心形、云朵等自由形状不支持识别。

---

## InstantShapeGenerator 方法

| 方法 | 签名 | 说明 |
|------|------|------|
| `processTouchEvent` | (event: TouchEvent): void | 传递触摸事件，**必须在 onTouch 内调用** |
| `setPauseTime` | (time: number): void | 设置触发识别的停顿时间(ms)，默认 300 |
| `onShapeRecognized` | (callback: Callback\<ShapeInfo\>): InstantShapeGenerator | 注册识别完成回调 |
| `getPathFromString` | (shapeString: string, penSize: number): Path2D | 从形状字符串恢复 Path2D |
| `notifyAreaChange` | (width: number, height: number): void | 通知组件尺寸变化 |
| `release` | (): void | 销毁引擎，释放资源 |

---

## 报点预测 C/C++ 原生接口

| 项 | 值 |
|----|-----|
| 头文件 | `<handwrite/native_handwrite_api.h>` |
| 库 | `libhandwrite_ndk.z.so` |
| 起始版本 | 6.0.0(20) |

```cpp
// 预测下一报点
Handwrite_ErrCode HMS_HandWrite_GetPredictPoint(
    const HandWrite_HistoricalPoint* event, int32_t size,
    float* predictPointX, float* predictPointY);
```

**HandWrite_HistoricalPoint 结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `x` | float | x 坐标，相对元素左上角(像素) |
| `y` | float | y 坐标，相对元素左上角(像素) |
| `timeStamp` | int64_t | 时间戳(ns) |
| `force` | float | 压力值 |

> 历史报点通过 XComponent 的 `OH_NativeXComponent_GetHistoricalPoints` 获取，映射为该结构数组后传入预测接口。

---

## 全局取色 C/C++ 原生接口

| 项 | 值 |
|----|-----|
| 头文件 | `<color_picker/native_gcp_api.h>` |
| 库 | `libcolorpicker_ndk.z.so` |
| 起始版本 | 6.0.0(20) |

```cpp
// 启动取色器（不显示 RGB）
void HMS_GCP_StartColorPicker(float initialPosX, float initialPosY,
    HMS_GCP_OnResult callback, void* userData);

// 启动取色器（可显示 RGB，5.1.0/18）
void HMS_GCP_StartColorPickerWithColorValue(float x, float y, bool showValue,
    HMS_GCP_OnResult callback, void* userData);

// 回调原型
typedef void (*HMS_GCP_OnResult)(void* userData,
    HMS_GCP_PickedColorInfo colorInfo, const int32_t code);
```

**HMS_GCP_Color 结构**：`{ red, green, blue, alpha }`（均为 double）

**HMS_GCP_PickedColorInfo 结构**：`{ color: HMS_GCP_Color, colorSpace: HMS_GCP_ColorSpace, timestamp: int64_t }`

---

## ResponseRegionSupportedTool 限笔响应区

| 名称 | 值 | 说明 |
|------|---|------|
| `ALL` | 0 | 所有输入工具类型 |
| `FINGER` | 1 | 手指 |
| `PEN` | 2 | 手写笔 |
| `MOUSE` | 3 | 鼠标 |

起始版本 22+。通过组件 `responseRegion` 配合该枚举，可限制某区域仅响应手写笔：

```typescript
// ✅ 仅手写笔可触发该区域响应
Canvas(this.context)
  .responseRegion({
    x: 0, y: 0, width: '100%', height: '100%',
    type: ResponseRegionSupportedTool.PEN
  })
```

> 低版本无此枚举时，改用 `event.sourceTool === SourceTool.Pen` 在 onTouch 内手动区分作为降级。

---

## 原生侧倾斜角与工具类型

C/C++ 原生侧通过 XComponent 获取倾斜角与工具类型，适用于自渲染场景：

| 接口 | 起始版本 | 说明 |
|------|---------|------|
| `OH_NativeXComponent_GetTouchPointToolType` | 9 | 获取触摸点工具类型枚举 |
| `OH_NativeXComponent_GetTouchPointTiltX` | 9 | 倾斜角（相对 X 轴） |
| `OH_NativeXComponent_GetTouchPointTiltY` | — | 倾斜角（相对 Y 轴） |
| `OH_NativeXComponent_GetHistoricalPoints` | 10 | 获取历史点（上报频率可达 1ms） |

**OH_NativeXComponent_TouchPointToolType 枚举**：UNKNOWN / FINGER / PEN / RUBBER / BRUSH / PENCIL / AIRBRUSH / MOUSE / LENS

> ArkTS 侧 `TouchEvent` 暂不直接暴露倾斜角，需倾斜角数据时走原生 XComponent 方案。

---

## 完整错误码

### 手写套件/报点预测/一笔成形

| 错误码 | 含义 | 可能原因 | 处理步骤 |
|--------|------|---------|---------|
| 1010400001 | load failed（加载失败） | 路径不存在或未知错误 | 确认路径有效；重试或提单 |
| 1010400002 | save failed（保存失败） | 保存路径错误 | 检查路径合法性；重试或提单 |
| 1010410001 | 引擎已释放 | release 后仍调用识别接口 | 确认 release 时机在 aboutToDisappear；重试或提单 |

### 报点预测 C/C++ 错误码

| 错误码 | 含义 | 处理步骤 |
|--------|------|---------|
| E_NO_ERROR (0) | 成功 | — |
| E_PARAMS (401) | 参数错误 | 检查历史点数组与 size |
| E_INNER_ERROR (1010400001) | 内部错误 | 重试或提单 |

### 全局取色错误码

| 错误码 | 含义 | 可能原因 | 处理步骤 |
|--------|------|---------|---------|
| 401 | 参数错误 | 参数缺失/类型不对 | 检查 x、y、showValue |
| 801 | api not supported | 设备/版本不支持 | `canIUse` 检测后降级 |
| 1013900001 | IPC 连接失败 | 防火墙屏蔽 | 检查防火墙，重试 |
| 1013900002 | 内存不足 | 内存不足 | 排查内存，重试 |
| 1013900003 | 服务不合法 | 未知 | 重试或提单 |
| 1013900004 | 重复调用 | 未返回结果前重复调用 | 等待上次取色返回后再调用 |
| 1013900005 | 后台服务调用 | 在后台调用 | 确保前台调用 |

> 全局取色要求设备支持手写笔。支持设备：Tablet、PC/2in1；从 5.1.1(19) 起新增 Phone。
