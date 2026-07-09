# 废弃 API 迁移档案（实战验证）

> 本档案记录在真实工程升级中迁过的每一个废弃 API，包含替代签名、代码示例、踩过的坑。
> 来源：ComprehensiveNews（5.0.1(13)→26.0.0）+ ComprehensiveTool（6.0.0(20)→26.0.0）两个工程实战。

---

## 1. px2vp / vp2px（since 18 废弃）

**废弃**：全局 `px2vp(value)` / `vp2px(value)`
**替代**：`UIContext.px2vp(value)` / `UIContext.vp2px(value)`

### 迁移示例

```typescript
// 组件内
this.getUIContext().px2vp(value)

// 有 windowClass
windowClass.getUIContext().px2vp(value)

// 工具函数（已有 UIContext 参数）
function foo(uiContext: UIContext, v: number) {
  return uiContext.px2vp(v)
}

// 工具函数（无 UIContext，需加参数）
// 改前：function getWidth(v: number) { return px2vp(v) }
// 改后：
function getWidth(uiContext: UIContext, v: number) {
  return uiContext.px2vp(v)
}
// 调用者：getWidth(this.getUIContext(), value)
```

### 实战数据
- ComprehensiveNews：px2vp 58 处 + vp2px 6 处
- ComprehensiveTool：px2vp 4 处 + vp2px 10 处

---

## 2. showToast（since 18 废弃）

**废弃**：`promptAction.showToast(options)`
**替代**：`UIContext.getPromptAction().showToast(options)`

### 迁移示例

```typescript
// 组件内
this.getUIContext().getPromptAction().showToast({ message: '提示' })

// ViewModel（有 uiContext 字段）
this.uiContext.getPromptAction().showToast({ message: '提示' })

// 通过 windowStageModel（继承 BaseViewModel 的 VM）
this.windowStageModel.windowStage?.getMainWindowSync().getUIContext().getPromptAction().showToast({ message: '提示' })
```

### 实战数据
- ComprehensiveNews：46 处
- ComprehensiveTool：58 处

### 易错：第三方库别名混淆
```typescript
// 这是第三方库别名，不是废弃 API，不能改！
import { ToastDialog as promptAction } from '@hw-agconnect/ui-toast'
promptAction.showToast({...})  // 实际是 ToastDialog.showToast
```
判断方法：看 import 来源，只有 `from '@kit.ArkUI'` 或 `from '@ohos.promptAction'` 才是废弃 API。

---

## 3. animateTo（since 18 废弃）

**废弃**：全局 `animateTo(value, event)`
**替代**：`UIContext.animateTo(value, event)`

### 迁移示例

```typescript
// 组件内
this.getUIContext().animateTo({ duration: 300 }, () => { ... })

// ViewModel（有 uiContext）
this.uiContext.animateTo({ duration: 300 }, () => { ... })

// module_transition 场景（有 WindowUtils.window 静态变量）
WindowUtils.window.getUIContext().animateTo({ duration: 300 }, () => { ... })
```

### 实战数据
- ComprehensiveNews：41 处
- ComprehensiveTool：3 处

### 易错：链式调用 vs 全局函数
```typescript
// 这是链式调用，不是全局函数，不要改！
someComponent.animateTo(...)

// 这才是废弃的全局函数
animateTo({ duration: 300 }, () => { ... })
```

---

## 4. getContext（since 18 废弃）

**废弃**：全局 `getContext(component?)`
**替代**：`UIContext.getHostContext()`

### 迁移示例

```typescript
// 组件内
this.getUIContext().getHostContext()

// 工具类（必须改，不能跳过）
// 改前：
class FileUtils {
  static readFile() {
    const ctx = getContext() as common.UIAbilityContext
  }
}
// 改后：加 context 参数
class FileUtils {
  static readFile(context: Context) {
    // 直接用传入的 context
  }
}
// 调用者更新：
FileUtils.readFile(this.getUIContext().getHostContext()!)
```

### getHostContext 返回 undefined
`getHostContext()` 返回 `Context | undefined`。两种处理：
```typescript
// 方式1：判空（推荐）
const ctx = this.getUIContext().getHostContext()
if (ctx) { ctx.resourceManager.getString(...) }

// 方式2：非空断言（组件生命周期内必然有值）
this.getUIContext().getHostContext()!.resourceManager.getString(...)
```

### 闭包内判空失效
```typescript
// 函数内判空后，闭包（回调）里仍报 undefined
let context = this.uiContext?.getHostContext()
if (!context) return
someApi.then(() => {
  fileIo.unlink(context.cacheDir)  // 错误：闭包内控制流重置，仍报 undefined
  fileIo.unlink(context!.cacheDir) // 正确：闭包内用非空断言
})
```

### 实战数据
- ComprehensiveNews：39 处（最初保留 25 处工具类，后纠正为全改）
- ComprehensiveTool：53 处（全部改为参数化传递）

---

## 5. openCustomDialog / closeCustomDialog（since 18 废弃）

**废弃**：`promptAction.openCustomDialog(...)` / `promptAction.closeCustomDialog(...)`
**替代**：`UIContext.getPromptAction().openCustomDialog(...)` / `.closeCustomDialog(...)`

### 迁移示例

```typescript
// 组件内
this.getUIContext().getPromptAction().openCustomDialog(contentNode, options)
this.getUIContext().getPromptAction().closeCustomDialog(contentNode)
```

### 实战数据
- ComprehensiveNews：6 处

---

## 6. AlertDialog.show（since 26 废弃）

**废弃**：`AlertDialog.show(options)`
**替代**：`UIContext.showAlertDialog(options)`

### 迁移示例

```typescript
// 组件内
this.getUIContext().showAlertDialog({
  title: '标题',
  message: '内容',
  buttons: [{ text: '确定', color: '#007aff' }]
})
```

### 易错：AlertDialog 组件 vs AlertDialog.show
```typescript
// 这是 ArkUI 的 AlertDialog 组件（构建器形式），不是废弃 API
AlertDialog({ title: 'xxx' })  // 用于 CustomDialogController，不要改

// 这才是废弃的全局函数
AlertDialog.show({ title: 'xxx' })  // 要改成 showAlertDialog
```

### 实战数据
- ComprehensiveNews：1 处
- ComprehensiveTool：1 处

---

## 7. MeasureText.measureTextSize（since 18 废弃）

**废弃**：`MeasureText.measureTextSize(options)`
**替代**：`UIContext.getMeasureUtils().measureTextSize(options)`

### 迁移示例

```typescript
this.getUIContext().getMeasureUtils().measureTextSize({ primaryKey, fontSize, maxWidth })
```

### 实战数据
- ComprehensiveNews：1 处

---

## 8. imagePacker.packing（since 13 废弃）

**废弃**：`imagePacker.packing(image, options)`
**替代**：`imagePacker.packToData(image, options)`

### 迁移示例

```typescript
// 改前
const data = await imagePacker.packing(pixelMap, { format: 'image/jpeg', quality: 90 })
// 改后
const data = await imagePacker.packToData(pixelMap, { format: 'image/jpeg', quality: 90 })
```

### 实战数据
- ComprehensiveNews：2 处

---

## 9. getVolumeSync / volumeChange（since 20 废弃）

**废弃**：
- `audio.getVolumeSync(AudioVolumeType.MEDIA)`
- `audio.on('volumeChange', callback)`

**替代**：
- `audio.getVolumeByStream(StreamUsage.STREAM_USAGE_MUSIC)`
- `audio.on('streamVolumeChange', StreamUsage.XXX, callback)`

### 迁移示例

```typescript
// 改前
const vol = audio.getVolumeSync(AudioVolumeType.MEDIA)
audio.on('volumeChange', (event: VolumeEvent) => { ... })
audio.off('volumeChange')

// 改后
const vol = audio.getVolumeByStream(StreamUsage.STREAM_USAGE_MUSIC)
audio.on('streamVolumeChange', StreamUsage.STREAM_USAGE_MUSIC, (event: StreamVolumeEvent) => { ... })
audio.off('streamVolumeChange')
```

### 实战数据
- ComprehensiveNews：3 处

---

## 10. resourceManager.getMediaContent（since 20 废弃）

**废弃**：`resourceManager.getMediaContent(resource: Resource)`（Resource 重载）
**替代**：`resourceManager.getMediaContent(resId: number)`（用 resId 重载）

### 迁移示例

```typescript
// 改前
const data = resourceManager.getMediaContent($r('app.media.icon'))
// 改后：用 .id 取 resId
const data = resourceManager.getMediaContent($r('app.media.icon').id)
```

### 实战数据
- ComprehensiveNews：1 处

---

## 11. getTagInfo（NFC，since 9 废弃）

**废弃**：`tagSession.getTagInfo()`（实例方法）
**替代**：`tag.getTagInfo(want)`（全局函数）

### 这个 API 比较特殊
实例方法废弃了，替代是全局函数但需要 `want` 参数。迁移时需确认 tagSession 实例的来源，是否能拿到 want。

### 实战数据
- ComprehensiveTool：4 处（MifareClassicReader/Iso7816/FeliCa/Nfc）

---

## 12. intl.DateTimeFormat / format（since 20 废弃）

**废弃**：`intl.DateTimeFormat`（@ohos.intl 模块的类）
**替代**：`Intl.DateTimeFormat`（全局 Intl 命名空间，首字母大写）

### 迁移示例

```typescript
// 改前
import { intl } from '@kit.LocalizationKit'
const fmt = new intl.DateTimeFormat('zh-CN', { dateStyle: 'long' })
const str = fmt.format(date)

// 改后：用全局 Intl，去掉 intl import
const fmt = new Intl.DateTimeFormat('zh-CN', { dateStyle: 'long' })
const str = fmt.format(date)
```

### 易错：intl.DateTimeFormat vs Intl.DateTimeFormat
小写 `intl` 是模块导入的，大写 `Intl` 是全局命名空间。编译器对小写 `intl.DateTimeFormat` 报废弃。

### 实战数据
- ComprehensiveTool：4 处（DateTimeFormat 2 + format 2）

---

## 13. getStringSync（since 20 部分重载废弃）

**废弃**：`getStringSync(resource: Resource)` 重载
**替代**：`getStringSync(resId: number)` 重载

### 迁移示例

```typescript
// 改前
const str = resourceManager.getStringSync($r('app.string.xxx'))
// 改后
const str = resourceManager.getStringSync($r('app.string.xxx').id)
```

### 易错：Resource 可选类型 .id 访问报 undefined

如果变量是可选类型（`Resource?` 或 `Resource | undefined`），改成 `.id` 后会报错 `Object is possibly 'undefined'`（错误码 10605999）。改法：

```typescript
// 报错：res 是 Resource? 类型，res.id 可能 undefined
resourceManager.getStringSync(res.id)
// 正确：加非空断言
resourceManager.getStringSync(res!.id)
```

### 实战数据
- ComprehensiveTool：1 处
- ibest-ui：getStringSync 4 处 + getMediaContentBase64Sync 2 处

---

## 14. @ohos.promptAction 模块路径（since 20 废弃）

**废弃**：`from '@ohos.promptAction'` 模块路径
**替代**：`from '@kit.ArkUI'` 统一入口

### 迁移示例

```typescript
// 改前
import { promptAction } from '@ohos.promptAction'
// 或 import { LevelMode } from '@ohos.promptAction'

// 改后
import { promptAction, LevelMode } from '@kit.ArkUI'
```

### 实战数据
- ComprehensiveTool：1 处

---

## 迁移统计汇总

> 来源：5 个工程实战验证
> - ComprehensiveNews（5.0.1(13)→26.0.0，1080 文件）
> - ComprehensiveTool（6.0.0(20)→26.0.0，1146 文件）
> - ComprehensiveMall（5.0.4(16)→26.0.0，483 文件）
> - Recipes（5.0.4(16)→26.0.0，262 文件）
> - HMOSWorld（5.0.0(12)→26.0.0，420 文件）

| API | since | News | Tool | Mall | Recipes | HMOS | 总计 |
|-----|-------|------|------|------|---------|------|------|
| showToast | 18 | 46 | 58 | 24 | 31 | 15 | 174 |
| getContext | 18 | 39 | 53 | 20 | 18 | 60 | 190 |
| px2vp | 18 | 58 | 4 | 22 | 2 | 9 | 95 |
| animateTo | 18 | 41 | 3 | 13 | 0 | 18 | 75 |
| vp2px | 18 | 6 | 10 | 6 | 0 | 8 | 30 |
| router 路由类 | 18 | 0 | 0 | 0 | 0 | 22 | 22 |
| openCustomDialog/closeCustomDialog | 18 | 6 | 0 | 3 | 3 | 0 | 12 |
| AlertDialog.show/showDialog | 18/26 | 1 | 1 | 0 | 2 | 2 | 6 |
| matchMediaSync | 18 | 0 | 0 | 0 | 0 | 3 | 3 |
| measureTextSize | 18 | 1 | 0 | 0 | 0 | 0 | 1 |
| packing | 13 | 2 | 0 | 0 | 0 | 0 | 2 |
| getVolumeSync/volumeChange | 20 | 3 | 0 | 0 | 0 | 0 | 3 |
| getMediaContent/getMediaContentSync | 20 | 1 | 0 | 1 | 0 | 1 | 3 |
| getTagInfo | 9 | 0 | 4 | 0 | 0 | 0 | 4 |
| DateTimeFormat | 20 | 0 | 2 | 0 | 0 | 0 | 2 |
| getStringSync/getStringValue | 9/20 | 0 | 1 | 0 | 0 | 3 | 4 |
| @ohos.promptAction 路径 | 20 | 0 | 1 | 0 | 0 | 0 | 1 |
| **合计** | | **204** | **137** | **86** | **57** | **144** | **628** |

**规律观察**：showToast + getContext + px2vp 三者占总量的 **73%**（459/628）。路由类 API（router.xxx）在老工程（API 12 基线）出现，是 HMOSWorld 特有的大类。

---

## 15. getMediaContentSync（since 20 废弃）

**废弃**：`resourceManager.getMediaContentSync(resource: Resource)` 重载
**替代**：`resourceManager.getMediaContentSync(resId: number)` 重载

### 迁移示例

```typescript
// 改前
const data = context.resourceManager.getMediaContentSync(image)  // image 是 Resource 类型
// 改后：用 .id 取 resId
const data = context.resourceManager.getMediaContentSync((image as Resource).id)
```

### 实战数据
- ComprehensiveMall：1 处

---

## 16. 路由 API（router.xxx，since 18 废弃）

**废弃**：`router.replaceUrl / pushUrl / replaceNamedRoute / pushNamedRoute / back / getParams`
**替代**：`UIContext.getRouter().replaceUrl / pushUrl / ...`

### 迁移示例

```typescript
// 改前（全局 router 模块）
import router from '@ohos.router'
router.pushUrl({ url: 'pages/Detail' })
router.replaceUrl({ url: 'pages/Login' })
router.back()
router.getParams()

// 改后（UIContext 实例）
// 组件内：
this.getUIContext().getRouter().pushUrl({ url: 'pages/Detail' })
this.getUIContext().getRouter().replaceUrl({ url: 'pages/Login' })
this.getUIContext().getRouter().back()
this.getUIContext().getRouter().getParams()

// 工具类：用 ContextManager.getUIContext() 或加 uiContext 参数
```

### 实战数据
- HMOSWorld：22 处（replaceUrl 6 + getParams 5 + back 4 + replaceNamedRoute 4 + pushNamedRoute 2 + pushUrl 1）

---

## 17. matchMediaSync（since 18 废弃）

**废弃**：`mediaquery.matchMediaSync(condition)`
**替代**：`UIContext.getMediaQuery().matchMediaSync(condition)`

### 迁移示例

```typescript
// 改前
const listener = mediaquery.matchMediaSync('(dark-mode: true)')

// 改后
const listener = this.getUIContext().getMediaQuery().matchMediaSync('(dark-mode: true)')
```

### 实战数据
- HMOSWorld：3 处

---

## 18. getStringValue（since 9 废弃）

**废弃**：`resourceManager.getStringValue(resId: number)` 旧重载
**替代**：`resourceManager.getStringValue(resId: number, callback)` 或带 .id 的新重载

### 迁移示例

```typescript
// 改前
const str = resourceManager.getStringValue(resource)
// 改后
const str = resourceManager.getStringValue(resource.id)
```

### 实战数据
- HMOSWorld：1 处

---

## 通用易错：迁移时不要破坏原有控制流兜底

迁移废弃 API 时，如果改动涉及 `if/else` 分支，**不要把无条件的 `else` 改成带条件的 `else if`**——会破坏兜底赋值，导致变量在某个分支未赋值，编译报 `Variable 'xxx' is used before being assigned`（错误码 10605999）。

```typescript
// 原代码：else 是兜底，保证变量一定被赋值
let result: SomeType
if (typeof value == 'string') {
    result = processString(value)
} else {  // Resource 类型兜底
    result = processResource(resourceManager.getMediaContent(value))
}

// 错误改法：为了加 .id 判空把 else 改成 else if(value)
// 当 value 是 falsy 时，result 没赋值 → 编译报错
} else if (value) {
    result = processResource(resourceManager.getMediaContent(value.id))
}

// 正确改法：保持 else 兜底，在内部用非空断言
} else {
    result = processResource(resourceManager.getMediaContent(value!.id))
}
```

---

## 附录：工具类 getContext 的两种迁移模式

迁移工具类的 `getContext()` 有两种模式，按场景选择：

### 模式 A：参数化传递（推荐，ComprehensiveNews/Tool 用）
给函数加 `context: Context` 参数，调用者传入。

```typescript
class FileUtils {
  static readFile(context: Context) { ... }  // 加参数
}
// 调用者
FileUtils.readFile(this.getUIContext().getHostContext()!)
```
**适用**：调用者少、调用链清晰的场景。

### 模式 B：静态 setContext 持有者（ComprehensiveMall 用）
工具类加静态 `setContext()` 方法，在 EntryAbility.onCreate 注入。

```typescript
class PopViewUtil {
  private static context?: Context
  static setContext(ctx: Context) { PopViewUtil.context = ctx }
  static showToast(msg: string) {
    PopViewUtil.context?.resourceManager.getString(...)
  }
}
// EntryAbility.onCreate
PopViewUtil.setContext(this.getUIContext().getHostContext()!)
```
**适用**：调用者很多（几十处）、逐个改调用链成本太高的场景。

**注意**：模式 B 的 context 在 EntryAbility.onCreate 时注入，如果工具类在 onCreate 之前被调用会拿到 undefined。需确保注入时序。
