---
name: hmos-pen-adaptation
description: 鸿蒙手写笔（Pen Kit）适配开发。当开发者提到手写笔、画布书写、笔记、压感、报点预测、跟手性、一笔成形、图形识别、全局取色、取色器、笔身轻捏、笔身双击、stylusInteraction、sourceTool 区分笔和手指、HandwriteComponent、PointPredictor、InstantShapeGenerator 时触发。适用于手写套件集成、自定义画布压感绘制、跟手性优化、图形识别、全屏取色、笔身硬件事件等场景。不包含鼠标/键盘/触摸交互归一化（见 hmos-multidevice-interaction-methods skill）。
---

## Agent Interface

```yaml
symptom_keywords:
  - pen strokes don't show pressure variation, line width is fixed
  - finger touch also draws on canvas, pen and finger not distinguished
  - handwriting lags behind pen movement, poor responsiveness
  - drawn shape not recognized as regular geometry after pause
  - global color picker fails to launch
  - pen body squeeze or double tap does nothing
  - getPredictionPoint returns null or stale coordinates
  - InstantShapeGenerator throws 1010410001 engine released error
  - pressure property is undefined when reading touch.pressure

hard_constraints:
  - Always check canIUse('SystemCapability.Stylus.Handwrite') before using suite/PointPredictor/InstantShapeGenerator; check SystemCapability.Stylus.ColorPicker before imageFeaturePicker; check SystemCapability.Stylus.StylusService before stylusInteraction — never call these APIs on unsupported devices
  - In custom Canvas drawing, must branch on event.sourceTool === SourceTool.Pen; never mix pen input and finger touch into the same drawing path
  - touch.pressure may be undefined — always null-check before using it for line width mapping; fall back to base width when undefined
  - HandwriteComponent already enables point prediction and instant shape internally; do NOT separately integrate PointPredictor or InstantShapeGenerator when using the suite — avoid duplicate integration
  - InstantShapeGenerator and PointPredictor are finite engine resources — must call release() in aboutToDisappear; calling any method after release() throws 1010410001
  - processTouchEvent() must be called inside the onTouch callback, not elsewhere — it feeds the raw TouchEvent to the recognition engine
  - stylusInteraction.off() must receive the same callback reference passed to on(); anonymous functions create new references and cannot be unregistered
  - imageFeaturePicker.pickForResult() must not be called again before the previous Promise resolves — duplicate calls throw 1013900004

diagnostic_checklist:
  - Is canIUse() checked for the corresponding SystemCapability before calling any Pen Kit API?
  - Is event.sourceTool used to branch pen vs finger in custom Canvas onTouch?
  - Is touch.pressure null-checked before being used for line width?
  - Is PointPredictor/InstantShapeGenerator released in aboutToDisappear?
  - Is processTouchEvent called inside onTouch for InstantShapeGenerator?
  - Is the same callback reference passed to stylusInteraction.off() as was passed to on()?
  - If using HandwriteComponent, is PointPredictor/InstantShapeGenerator NOT separately integrated (suite already includes them)?
```

# 手写笔适配

## 适配领域

本 skill 覆盖应用集成鸿蒙 Pen Kit（`@kit.Penkit`）的手写笔全链路适配，包含五大能力的选择策略、底层笔触事件（sourceTool 区分、压感读取）的正确用法，以及引擎生命周期管理。

具体覆盖：手写套件（HandwriteComponent/Controller）集成、报点预测（PointPredictor）跟手性优化、一笔成形（InstantShapeGenerator）图形识别、全局取色（imageFeaturePicker）、手写交互（stylusInteraction 笔身轻捏/双击）、自定义 Canvas 笔触事件适配。

**不覆盖**：触摸/鼠标/键盘交互归一化（见 `hmos-multidevice-interaction-methods` skill）；纯视觉动画微调；与手写笔无关的业务流程。

> Pen Kit 仅限中国境内使用（港澳台除外），不支持模拟器，适用设备为 Phone/Tablet/2in1 且需支持手写笔硬件。

---

## 能力与场景矩阵

| 能力 | 核心 API | 系统能力 | 起始版本 | 典型场景 |
|------|---------|---------|---------|---------|
| 手写套件 | HandwriteComponent / HandwriteController | Stylus.Handwrite | 5.0.0(12) | 开箱画布+工具栏，笔记/签名应用 |
| 报点预测 | PointPredictor.getPredictionPoint | Stylus.Handwrite | 5.0.0(12) | 自定义 Canvas 提升跟手性 |
| 一笔成形 | InstantShapeGenerator | Stylus.Handwrite | 5.0.0(12) | 抬笔停顿后识别规则图形 |
| 全局取色 | imageFeaturePicker.pickForResult | Stylus.ColorPicker | 5.0.0(12) | 全屏取色，获取色值/色域 |
| 手写交互 | stylusInteraction.on('squeeze'/'doubleTap') | Stylus.StylusService | 5.1.1(19) | 笔身硬件事件切换工具 |
| 底层笔触 | event.sourceTool / touch.pressure | ArkUI.Full | 15+(pressure) | 自定义绘制区分笔/指、压感线宽 |

> **套件已内置报点预测与一笔成形**：使用 HandwriteComponent 时无需单独接入 PointPredictor / InstantShapeGenerator，仅在自定义 Canvas 场景才独立集成。

---

## 适配流程

### 新适配

**第 1 步：选择手写能力方案**

根据场景选择：

| 场景 | 推荐方案 |
|-----|---------|
| 需要开箱即用画布+工具栏（笔刷/橡皮/套索/保存加载） | **手写套件**：HandwriteComponent + HandwriteController |
| 自定义 Canvas 绘制，需提升跟手性 | **报点预测**：PointPredictor.getPredictionPoint |
| 自定义 Canvas，抬笔停顿后识别规则图形 | **一笔成形**：InstantShapeGenerator |
| 全屏取色，获取色值与色彩空间 | **全局取色**：imageFeaturePicker.pickForResult |
| 监听笔身轻捏/双击切换工具 | **手写交互**：stylusInteraction.on |
| 自定义绘制需区分笔/指、读取压感 | **底层笔触**：event.sourceTool + touch.pressure |

各能力 API 参数细节见 `references/api-reference.md`。

**第 2 步：能力检测**

使用任何 Pen Kit 能力前，先用 `canIUse` 检测对应系统能力，不支持时给出降级路径：

```typescript
// 手写套件/报点预测/一笔成形
if (!canIUse('SystemCapability.Stylus.Handwrite')) {
  console.warn('当前设备不支持手写能力，降级为普通触摸交互')
}
// 全局取色
if (!canIUse('SystemCapability.Stylus.ColorPicker')) { /* 降级 */ }
// 笔身交互
if (!canIUse('SystemCapability.Stylus.StylusService')) { /* 降级 */ }
```

**第 3 步：实现核心能力**

详见"代码模式"。套件需配置 EntryAbility Context；自定义 Canvas 需在 onTouch 内分流 sourceTool；引擎类需管理 release 生命周期。

**第 4 步：引擎生命周期管理**

InstantShapeGenerator、PointPredictor 等引擎实例为有限资源，必须在组件销毁时释放：

```typescript
aboutToDisappear() {
  this.generator?.release()   // 一笔成形引擎
  // PointPredictor 无显式 release，但应停止调用
}
```

---

### 问题定位

**1. 收集现象**
- 是"压感不生效线宽固定"、"手指也能画"、"跟手性差"还是"识别不触发"？
- 是套件场景还是自定义 Canvas 场景？
- 控制台是否有 1010410001 / 1013900004 / 801 等错误码？

**2. 确认触发条件**

| 现象 | 可能原因 |
|-----|---------|
| 压感不生效，线宽固定 | 未用 `event.sourceTool` 区分笔；或读取 `touch.pressure` 未判空导致 NaN |
| 手指触摸也画出线 | 未在 onTouch 内分流 `sourceTool`，笔与手指混入同一绘制主流程 |
| 跟手性差有延迟 | 自定义 Canvas 未集成 PointPredictor；或未用 getHistoricalPoints 补全历史点 |
| 一笔成形不触发 | `processTouchEvent` 未在 onTouch 内调用；或 `setPauseTime` 过短；或引擎已 release |
| 1010410001 引擎已释放 | release 后仍调用识别接口；或 release 时机不在 aboutToDisappear |
| 取色器拉不起来 | 未检测 ColorPicker 能力（801）；后台调用（1013900005）；未返回前重复调用（1013900004） |
| 笔身事件不触发 | 设备/版本不支持 StylusService（5.1.1+）；或 off 传了不同回调引用 |
| 套件 save/load 失败 | 路径错误（1010400001/0002）；Controller 方法调用时机不在 onInit 之后 |

**3. 修复与验证**

修复后执行验证清单，确认：能力检测到位、笔/指区分正确、压感生效、引擎无泄漏、取色器可正常拉起。

---

## 关键 API

### HandwriteComponent

**用途**：开箱即用的手写画布+工具栏组件，内置 7 种笔刷、橡皮、套索，默认开启报点预测与一笔成形。

**注意**：仅 Stage 模型；需先在 EntryAbility 中设置 GlobalContext；Controller 方法需在 `onInit` 回调后调用。

```typescript
import { HandwriteController, HandwriteComponent, PenType, PenHspInfo } from '@kit.Penkit'

HandwriteComponent({
  handwriteController: this.controller,
  defaultPenType: PenType.PEN,            // 默认笔刷
  onInit: () => {
    this.controller?.load(this.initPath)  // ✅ 初始化完成后加载笔记
  },
  onDidScroll: (yOffset: number) => { /* 滚动监听 */ }
})
```

### PointPredictor.getPredictionPoint()

**用途**：在自定义 Canvas 的 onTouch Move 中预测下一报点，提前绘制以降低延迟。

**注意**：套件已内置，仅自定义 Canvas 场景单独集成。

```typescript
case TouchType.Move:
  // ✅ 获取预测点提前绘制
  let point = this.pointPredictor?.getPredictionPoint(event)
  if (point) {
    this.context.lineTo(point.x, point.y)
    this.context.stroke()
  }
```

### InstantShapeGenerator

**用途**：抬笔停顿后将手绘笔迹识别为规则几何图形，输出 Path2D 与可持久化的 shapeString。

**关键原则**：`processTouchEvent` 必须在 onTouch 内调用；`release()` 必须在 aboutToDisappear 调用。

```typescript
aboutToAppear() {
  this.generator?.setPauseTime(280)              // 设置停顿时间
  this.generator?.onShapeRecognized((info) => {
    this.shapePath = info.shapePath              // 识别结果 Path2D
  })
}
// onTouch 内：
this.generator?.processTouchEvent(event)         // ✅ 必须在 onTouch 内
aboutToDisappear() {
  this.generator?.release()                      // ✅ 释放引擎
}
```

### imageFeaturePicker.pickForResult()

**用途**：启动全屏取色器，抬起后返回色值与色彩空间。

**注意**：设备需支持手写笔；未返回结果前不可重复调用。

```typescript
// ✅ 检测能力后调用
imageFeaturePicker.pickForResult(x, y, true)     // showValue=true 实时显示 RGB
  .then((info: imageFeaturePicker.PickedColorInfo) => {
    // info.color / info.colorSpace / info.timestamp
  })
  .catch((err: BusinessError) => { /* 处理 801/1013900004 等 */ })
```

### stylusInteraction.on('squeeze'/'doubleTap')

**用途**：监听笔身硬件事件（轻捏/双击），触发工具切换等应用内行为。

**注意**：off 必须传入与 on 相同的回调引用；起始版本 5.1.1(19)。

```typescript
// ✅ 用成员变量保存回调引用
private squeezeCb = (e: stylusInteraction.SqueezeEvent) => { /* ... */ }
aboutToAppear() { stylusInteraction.on('squeeze', this.squeezeCb) }
aboutToDisappear() { stylusInteraction.off('squeeze', this.squeezeCb) }  // 同一引用
```

### event.sourceTool / touch.pressure

**用途**：在自定义 Canvas 的 onTouch 中区分手写笔/手指/鼠标输入，并读取压感。

**关键原则**：pressure 可能为 undefined，必须判空后使用。

```typescript
.onTouch((event: TouchEvent) => {
  if (event.sourceTool === SourceTool.Pen) {
    const touch = event.touches[0]
    // ✅ 压感判空，undefined 时回退默认线宽
    const w = touch.pressure != null ? 1 + touch.pressure * 10 : this.baseWidth
    this.context.lineWidth = w
    this.context.lineTo(touch.x, touch.y)
    this.context.stroke()
  }
})
```

---

## 代码模式

### 模式 1：手写套件集成（开箱画布+保存加载）

```typescript
import { HandwriteController, HandwriteComponent, PenType, PenHspInfo } from '@kit.Penkit'

@Entry @Component
struct NotePage {
  controller: HandwriteController = new HandwriteController()
  initPath: string = this.getUIContext().getHostContext()?.filesDir + '/note'
  @State yOffset: number = 0

  aboutToAppear() {
    this.controller.onLoad((err, data) => {
      if (err) { console.error('load failed: ' + err.message) }
    })
  }

  build() {
    Stack({ alignContent: Alignment.TopStart }) {
      HandwriteComponent({
        handwriteController: this.controller,
        defaultPenType: PenType.PEN,
        defaultPenInfo: [
          { penType: PenType.PEN, penWidth: 5 },
          { penType: PenType.BALLPOINT_PEN, penWidth: 6 }
        ] as PenHspInfo[],
        maxCanvasHeight: 5000,               // 长画布最大高度
        scaleDisabled: false,
        onInit: () => { this.controller?.load(this.initPath) },  // 初始化后加载
        onDidScroll: (y: number) => { this.yOffset = y }
      })
      Button('保存')
        .onClick(async () => {
          const path = this.getUIContext().getHostContext()?.filesDir + '/note'
          await this.controller?.save(path)  // 保存笔迹
          this.controller?.getThumbnail(this.controller?.getContentRange())  // 缩略图
            ?.then((pm: PixelMap) => { pm?.release() })
        })
    }.width('100%').layoutWeight(1)
  }
}
```

### 模式 2：自定义 Canvas 笔/指区分 + 压感 + 报点预测

```typescript
import { PointPredictor } from '@kit.Penkit'

@Entry @Component
struct PenDrawPage {
  private ctx: CanvasRenderingContext2D = new CanvasRenderingContext2D(new RenderingContextSettings(true))
  private predictor: PointPredictor = new PointPredictor()
  @State baseWidth: number = 4

  build() {
    Canvas(this.ctx)
      .width('100%').height('100%')
      .onTouch((event: TouchEvent) => {
        const touch = event.touches[0]
        if (touch == null) return
        switch (event.sourceTool) {
          case SourceTool.Pen:
            // ✅ 手写笔分支：压感调线宽
            if (event.type === TouchType.Down) {
              this.ctx.beginPath(); this.ctx.moveTo(touch.x, touch.y)
            } else if (event.type === TouchType.Move) {
              // 压感判空后映射线宽
              const w = touch.pressure != null ? this.baseWidth * (1 + touch.pressure * 2) : this.baseWidth
              this.ctx.lineWidth = w
              // 历史报点补全，减少丢点
              event.getHistoricalPoints().forEach((hp) => {
                this.ctx.lineTo(hp.touchObject.x, hp.touchObject.y)
              })
              this.ctx.lineTo(touch.x, touch.y); this.ctx.stroke()
              // 报点预测：提前绘制预测点提升跟手性
              const p = this.predictor?.getPredictionPoint(event)
              if (p) { this.ctx.lineTo(p.x, p.y); this.ctx.stroke() }
            }
            break
          case SourceTool.Finger:
            // ✅ 手指分支：固定线宽，不读取压感
            if (event.type === TouchType.Down) {
              this.ctx.beginPath(); this.ctx.lineWidth = this.baseWidth; this.ctx.moveTo(touch.x, touch.y)
            } else if (event.type === TouchType.Move) {
              this.ctx.lineTo(touch.x, touch.y); this.ctx.stroke()
            }
            break
        }
      })
  }
}
```

### 模式 3：一笔成形图形识别

```typescript
import { InstantShapeGenerator, ShapeInfo } from '@kit.Penkit'

@Entry @Component
struct ShapePage {
  private generator: InstantShapeGenerator = new InstantShapeGenerator()
  private ctx: CanvasRenderingContext2D = new CanvasRenderingContext2D(new RenderingContextSettings(true))
  private drawPath: Path2D = new Path2D()
  private recognized: boolean = false

  aboutToAppear() {
    this.generator?.setPauseTime(280)                    // 停顿时间
    this.generator?.onShapeRecognized((info: ShapeInfo) => {
      this.recognized = true
      this.ctx.beginPath(); this.ctx.reset()
      this.ctx.lineWidth = 8; this.ctx.strokeStyle = '#ED1B1B'
      this.ctx.stroke(info.shapePath)                     // 绘制识别图形
    })
  }

  aboutToDisappear() { this.generator?.release() }        // ✅ 释放引擎

  build() {
    Canvas(this.ctx)
      .width('100%').height('100%')
      .onAreaChange((_o, n) => {
        this.generator?.notifyAreaChange(Number(n.width), Number(n.height))  // 同步尺寸
      })
      .onTouch((event: TouchEvent) => {
        this.generator?.processTouchEvent(event)           // ✅ 必须在 onTouch 内
        const t = event.touches[0]
        if (t == null) return
        if (event.type === TouchType.Down) {
          this.drawPath = new Path2D(); this.drawPath.moveTo(t.x, t.y); this.recognized = false
        } else if (event.type === TouchType.Move) {
          this.drawPath.lineTo(t.x, t.y)
          if (!this.recognized) { this.ctx.beginPath(); this.ctx.reset(); this.ctx.stroke(this.drawPath) }
        }
      })
  }
}
```

### 模式 4：全局取色联动画笔颜色

```typescript
import { imageFeaturePicker } from '@kit.Penkit'
import { BusinessError } from '@kit.BasicServicesKit'

@Entry @Component
struct ColorPickerPage {
  @State brushColor: string = '#333333'

  build() {
    Column() {
      Button('取色')
        .onClick((event) => {
          if (!canIUse('SystemCapability.Stylus.ColorPicker')) {  // ✅ 能力检测
            console.warn('不支持取色'); return
          }
          imageFeaturePicker.pickForResult(event.displayX, event.displayY, true)
            .then((info: imageFeaturePicker.PickedColorInfo) => {
              // 将取色结果应用到画笔颜色
              this.brushColor = `rgb(${info.color.red}, ${info.color.green}, ${info.color.blue})`
            })
            .catch((err: BusinessError) => {
              console.error(`取色失败: ${err.code} ${err.message}`)
            })
        })
    }.width('100%').height('100%')
  }
}
```

### 模式 5：笔身双击切换工具

```typescript
import { stylusInteraction } from '@kit.Penkit'
import { BusinessError } from '@kit.BasicServicesKit'

@Entry @Component
struct ToolSwitchPage {
  @State tool: string = '钢笔'
  // ✅ 成员变量保存回调引用，确保 off 时传入同一引用
  private doubleTapCb = (e: stylusInteraction.DoubleTapEvent) => {
    this.tool = this.tool === '钢笔' ? '橡皮' : '钢笔'
  }

  aboutToAppear() {
    if (!canIUse('SystemCapability.Stylus.StylusService')) return  // ✅ 能力检测
    try { stylusInteraction.on('doubleTap', this.doubleTapCb) }
    catch (e) { console.error((e as BusinessError).message) }
  }

  aboutToDisappear() {
    try { stylusInteraction.off('doubleTap', this.doubleTapCb) }   // ✅ 同一引用
    catch (e) { console.error((e as BusinessError).message) }
  }

  build() {
    Column() { Text('当前工具: ' + this.tool).fontSize(20) }
      .width('100%').height('100%')
  }
}
```

---

## 验证清单

**基础验证（每次必做）：**
- [ ] 能力检测：`canIUse` 检测对应 SystemCapability，不支持设备有降级提示
- [ ] 笔/指区分：手写笔可绘制，手指走独立分支或不绘制
- [ ] 压感生效：手写笔书写线宽随按压轻重变化
- [ ] 引擎释放：组件销毁后无 1010410001 错误

**压力验证（发布前做）：**
- [ ] 跟手性：报点预测开启后书写延迟明显降低
- [ ] 一笔成形：手绘圆/矩形停顿后正确识别为规整图形
- [ ] 取色器：支持设备可正常取色，不支持设备降级而非崩溃
- [ ] 笔身事件：双击/轻捏能切换工具，off 后不再触发
- [ ] 套件保存加载：保存后重新加载笔记内容完整

---

## 常见问题

**Q：压感不生效，手写笔线宽一直固定**
A：两个常见原因：一是未用 `event.sourceTool === SourceTool.Pen` 区分笔，导致压感读取分支未进入；二是读取 `touch.pressure` 未判空，pressure 为 undefined 时计算出 NaN。修复：在 onTouch 内按 sourceTool 分流，读取 pressure 前判空并回退默认线宽。

**Q：手指触摸也会画出线，本应只有手写笔能画**
A：onTouch 内未按 `event.sourceTool` 分流，笔与手指混入了同一绘制主流程。修复：用 `switch (event.sourceTool)` 将 Pen 和 Finger 走不同分支，Finger 分支不绘制或固定线宽。

**Q：一笔成形不触发，偶发 1010410001 错误**
A：`processTouchEvent` 未在 onTouch 回调内调用，或 `setPauseTime` 停顿时间过短导致识别不触发。1010410001 表示引擎已释放但仍被调用——确认 `release()` 仅在 `aboutToDisappear` 调用，不在其他时机提前释放。

**Q：全局取色器拉不起来，报 801 或 1013900005**
A：801 表示设备/版本不支持 ColorPicker 能力，需先 `canIUse` 检测后降级。1013900005 表示后台调用，取色器必须在前台调用。此外未返回结果前重复调用会报 1013900004。

**Q：笔身双击事件不触发**
A：确认设备/系统版本支持 `SystemCapability.Stylus.StylusService`（5.1.1+）；确认手写笔硬件支持笔身双击；确认应用在前台。若 off 不生效，检查是否传入了与 on 不同的回调引用（匿名函数每次创建新引用）。

**Q：使用手写套件还需要单独接入报点预测吗**
A：不需要。HandwriteComponent 已默认开启报点预测与一笔成形。仅在自定义 Canvas（不使用套件）时才单独集成 PointPredictor / InstantShapeGenerator。

---

## 延伸阅读

- `references/api-reference.md`：系统能力检测表、套件工具栏功能与 Context 初始化、PenType 笔刷枚举全表、PenHspInfo 笔宽默认值/范围表、HandwriteComponent 参数全表、HandwriteController 方法全表（含 onLoad 回调说明）、PointPredictor ArkTS 接口、imageFeaturePicker 双重载与 PickedColorInfo 字段、stylusInteraction 接口表与事件字段、SourceTool/TouchType/pressure 枚举与取值范围、getHistoricalPoints 与 HistoricalPoint 结构、ShapeInfo shapeType 图形类型全表、报点预测与取色的 C/C++ 原生接口、ResponseRegionSupportedTool 限笔响应区枚举、原生侧倾斜角、完整错误码表——当需要查阅 API 参数细节、枚举值或原生侧接口时读取。
