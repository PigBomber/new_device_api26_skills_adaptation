---
name: harmonyos-deprecated-apis
description: 鸿蒙项目升级中的废弃 API 检查与替换环节。当开发者提到「升级后哪些 API 废弃了」「deprecated 警告怎么修」「API 找不到了」「接口被移除了」「deprecated API」「@deprecated 标记」「旧接口替代方案」「getContext 怎么改」「px2vp 废弃」「promptAction 废弃」「router 废弃」时触发。提供检测项目中废弃 API 使用的方法、API 26 核心废弃对照表（全局函数→UIContext）、迁移规则与易错点。与 harmonyos-behavior-changes（行为变化）互补。不包含版本检测、配置修改、编译验证。
---

## Agent Interface

```yaml
symptom_keywords:
  - deprecated API warning after upgrade
  - cannot find name / API not found after upgrade
  - global function deprecated (px2vp, animateTo, getContext, promptAction, router)
  - Resource overload deprecated (getMediaContent, getStringSync)
  - module path deprecated (@ohos.promptAction → @kit.ArkUI)
  - getContext() in utility class needs migration
  - getHostContext() returns undefined in closures
  - field initialization crashes with getUIContext()

hard_constraints:
  - Every single deprecated warning from the compiler must be fixed — no exceptions, no skipping with "tool class is hard to change" or "false positive"
  - Compiler warnings are the ONLY reliable detection method — never use grep to find deprecated APIs (it produces false positives and false negatives)
  - Utility class getContext() must also be migrated — add context parameter to function signature, caller passes UIContext; never keep getContext() accepting the warning
  - getHostContext() returns Context | undefined — always null-check or use non-null assertion; in closures, control flow analysis resets, so use ! even after null check
  - Never call getUIContext() in field initialization phase (property declaration) — it returns undefined because the component hasn't mounted; move to aboutToAppear
  - ContextManager UIContext injection must be inside loadContent callback — not before loadContent, or getMainWindowSync().getUIContext() will null-pointer
  - @ohos.xxx old import paths (e.g. @ohos.file.fs, @ohos.hilog) do NOT report deprecated warnings in API 26 — they are NOT required to change; only @deprecated-marked APIs must be changed

diagnostic_checklist:
  - Is hvigorw clean + full compile run to capture all deprecated warnings?
  - Are deprecated warnings counted correctly (TOTAL minus oh_modules)?
  - Is the import source checked before migrating promptAction.showToast (third-party alias vs real deprecated API)?
  - Is getHostContext() null-checked or asserted when used as function argument?
  - Is getUIContext() called in field initialization (will crash at runtime)?
  - Is ContextManager.setUIContext called inside loadContent callback (not before)?
```

# 废弃 API 检查与替换

## 适配领域

本 skill 覆盖升级流程的第三步：找出代码中使用的已被废弃（deprecated / removed）的 API，并找到替代方案。包含 API 26 核心废弃对照表（全局函数→UIContext）、迁移规则（4 种场景获取 UIContext）、迁移易错点（7 条）、各版本速查。

**不覆盖**：API 还在但行为变了（见 `harmonyos-behavior-changes`）；版本检测（见 `harmonyos-upgrade-detect`）；配置修改（见 `harmonyos-upgrade-config`）；编译验证（见 `harmonyos-upgrade-verify`）。

---

## 废弃 API vs 行为变化

| | 废弃 API（本 skill） | 行为变化（harmonyos-behavior-changes） |
|---|---|---|
| API 状态 | **已废弃或移除**，编译可能报错或告警 | API **还在**，但返回值/默认效果/触发时机变了 |
| 典型表现 | `cannot find name 'xxx'` / `@deprecated` 警告 | 运行时行为不符合预期 |
| 处理方式 | 找替代 API 替换 | 按行为变化适配代码 |

一个升级任务通常两个都要查：先查废弃的（不改编译不过），再查行为变化的（不改运行不对）。

---

## 适配流程

### 新适配

**第 1 步：编译检测（唯一权威依据，必须执行）**

**铁律：编译器报的每一条 deprecated 告警都必须修改，无一例外。**

- 不能以"工具类难改/调用链太长"为由跳过
- 不能以"误报/命名遮蔽/非废弃重载"为借口不改 —— 编译器报了就是事实，必须核实并改
- 不能保留 deprecated 告警说"功能不受影响" —— 废弃 API 在后续版本会被移除
- 工具类的 `getContext()` 也要改：改函数签名加 context 参数，由调用者传入 UIContext
- 如果某条确实不确定替代方案，用 `// TODO: deprecated XXX` 标注并说明原因，但**必须给出改的方向**

升级到 API 26 的项目，`hvigorw clean` 后全量编译的 deprecated 告警数必须降到 **0**（或只剩第三方库 oh_modules 的告警）。

```bash
cd <工程根目录>
# clean 后全量编译才能看到完整告警（增量用缓存会跳过）
hvigorw clean --no-daemon
hvigorw assembleHap --mode module -p module=<模块名>@default -p product=default --no-daemon 2>&1 | sed 's/\x1b\[[0-9;]*m//g' > /tmp/build.txt
# 统计（减去第三方库）
TOTAL=$(grep -c "has been deprecated" /tmp/build.txt)
OH=$(grep -B1 "has been deprecated" /tmp/build.txt | grep -c "oh_modules")
echo "工程代码 deprecated: $((TOTAL-OH))（目标 0）"
```

> 模块名从 `build-profile.json5` 的 `modules[].name` 读取，不一定是 `entry`。
> **hvigorw 配置**：见 `harmonyos-upgrade-verify` 的编译环境配置章节。

> **绝对禁止输出"本地没有构建工具，请在 DevEco Studio 中打开"然后中止**——装了 DevEco Studio 就一定有 hvigorw，配好环境变量即可（用户已踩过此坑）。

**第 2 步：按对照表逐个迁移**

查下方「API 26 核心废弃对照表」和 `references/migration-archive.md`（每个 API 的代码示例+踩坑记录），逐个替换。

**第 3 步：工具类 getContext 迁移**

工具类/纯函数（无 this，无 UIContext）的 `getContext()` 必须改，不能跳过。两种方式：
- **优先**：给函数加 `uiContext: UIContext` 参数，调用者传入
- **替代**：工具类加静态 `setUIContext()` 持有者，在 EntryAbility 注入（见 migration-archive 附录模式 B/C）

**第 4 步：重点复查易错点**

查下方「迁移易错点」7 条，逐项核验。

---

### 问题定位

| 现象 | 可能原因 | 修复 |
|------|---------|------|
| 编译报 `cannot find name 'xxx'` | API 已移除或改名 | 查对照表找替代 |
| deprecated 告警不为 0 | 工具类 getContext 未改 / Resource 重载未改 / 第三方库告警 | 工具类也必须改；Resource 重载用 .id；第三方库(oh_modules)告警可忽略 |
| 运行时崩溃 `getRouter of undefined` | 字段初始化阶段调用了 getUIContext() | 移到 aboutToAppear（见易错点 #6） |
| 闭包内报 `context is possibly undefined` | 闭包内控制流分析重置 | 用非空断言 `!`（见易错点 #3） |
| ContextManager 注入后仍空指针 | setUIContext 在 loadContent 之前调用 | 移到 loadContent 回调内（见易错点 #7） |

---

## 关键 API

### UIContext 迁移口诀

**用途**：API 18 起大规模废弃全局函数，统一改用 UIContext 实例方法。

**口诀**：全局 `xxx()` → `this.getUIContext().xxx()`，路由类是 `router.xxx()` → `this.getUIContext().getRouter().xxx()`，弹窗类是 `promptAction.xxx()` → `this.getUIContext().getPromptAction().xxx()`。

### getHostContext()

**用途**：替代废弃的全局 `getContext()`，获取 ability context。

**注意**：返回 `Context | undefined`。组件生命周期内必然有值，可加判空或非空断言 `!`。闭包内控制流分析重置，仍认为可能 undefined，需用 `!`。

---

## API 26 核心废弃对照表（since 18 起大规模废弃）

### 全局函数 → UIContext 实例方法（最大类，since 18）

| 废弃全局函数 | 替代（UIContext 实例方法） | since | 说明 |
|---|---|---|---|
| `px2vp(value)` | `uiContext.px2vp(value)` | 18 | 像素转 vp |
| `vp2px(value)` | `uiContext.vp2px(value)` | 18 | vp 转像素 |
| `lpx2px(x)` / `fp2px(x)` | `uiContext.lpx2px(x)` / `uiContext.fp2px(x)` | 18 | 同 px2vp/vp2px |
| `animateTo(value, fn)` | `uiContext.animateTo(value, fn)` | 18 | 动画 |
| `getContext(component?)` | `uiContext.getHostContext()` | 18 | 获取 Context |
| `promptAction.showToast(opt)` | `uiContext.getPromptAction().showToast(opt)` | 18 | Toast 提示 |
| `promptAction.showDialog(opt)` | `uiContext.getPromptAction().showDialog(opt)` | 18 | 对话框 |
| `promptAction.openCustomDialog(...)` | `uiContext.getPromptAction().openCustomDialog(...)` | 18 | 打开自定义弹窗 |
| `promptAction.closeCustomDialog(...)` | `uiContext.getPromptAction().closeCustomDialog(...)` | 18 | 关闭自定义弹窗 |
| `MeasureText.measureTextSize(opt)` | `uiContext.getMeasureUtils().measureTextSize(opt)` | 18 | 测量文本尺寸 |
| `mediaquery.matchMediaSync(condition)` | `uiContext.getMediaQuery().matchMediaSync(condition)` | 18 | 媒体查询 |
| `router.pushUrl(opt)` | `uiContext.getRouter().pushUrl(opt)` | 18 | 路由跳转 |
| `router.replaceUrl(opt)` | `uiContext.getRouter().replaceUrl(opt)` | 18 | 路由替换 |
| `router.pushNamedRoute(opt)` | `uiContext.getRouter().pushNamedRoute(opt)` | 18 | 命名路由跳转 |
| `router.replaceNamedRoute(opt)` | `uiContext.getRouter().replaceNamedRoute(opt)` | 18 | 命名路由替换 |
| `router.back(opt?)` | `uiContext.getRouter().back(opt?)` | 18 | 路由返回 |
| `router.getParams()` | `uiContext.getRouter().getParams()` | 18 | 获取路由参数 |
| `AlertDialog.show(opt)` | `uiContext.showAlertDialog(opt)` | 26 | 警告对话框 |

### 迁移规则（按调用上下文获取 UIContext）

迁移的关键是**如何拿到 UIContext 实例**，分 4 种场景。详见「代码模式」。

### 其他废弃 API（非 UIContext 类）

| 废弃 API | 替代 | since | 说明 |
|---|---|---|---|
| `imagePacker.packing(...)` | `imagePacker.packToData(...)` | 13 | 图片打包 |
| `getVolumeSync(AudioVolumeType.MEDIA)` | `getVolumeByStream(StreamUsage.STREAM_USAGE_MUSIC)` | 20 | 获取音量 |
| `audio.on('volumeChange', cb)` | `audio.on('streamVolumeChange', StreamUsage.XXX, cb)` | 20 | 音量变化监听 |
| `resourceManager.getMediaContent(resource)` | `getMediaContent(resource.id)` | 20 | Resource 重载废弃，用 resId 重载 |
| `resourceManager.getMediaContentSync(resource)` | `getMediaContentSync(resource.id)` | 20 | 同上 |
| `resourceManager.getStringSync(resource)` | `getStringSync(resource.id)` | 20 | Resource 重载废弃，用 resId 重载 |
| `resourceManager.getStringValue(resource)` | `getStringValue(resource.id)` | 9 | 同上模式 |
| `resourceManager.getPluralStringValueSync/Sync/ByName` | `getIntPluralStringValueSync` / `getIntPluralStringByNameSync` | 18 | **整个方法废弃**（不止 Resource 重载） |
| `intl.DateTimeFormat` | `Intl.DateTimeFormat`（全局命名空间） | 20 | 小写 intl 模块类废弃，用全局 Intl |
| `display.getDefaultDisplay()` | `display.getDefaultDisplaySync()` | 9 | 获取默认屏幕 |
| `i18n.language` / `i18n.Locale` | `i18n.System.getDisplayLanguage` / `getSystemLanguage` | 9 | 语言相关 |
| `tagSession.getTagInfo()` | `tag.getTagInfo(want)` | 9 | NFC 标签信息（实例方法废弃） |
| `@ohos.promptAction` 模块路径 | `@kit.ArkUI` | 20 | 旧模块路径废弃 |
| `picker.PhotoViewPicker` / `PhotoSelectOptions` | `photoAccessHelper.PhotoViewPicker` / `PhotoSelectOptions` | 20 | 类从 `@ohos.file.picker` 迁到 `@kit.MediaLibraryKit` |
| Web `.selectionMenuOptions()` | `.editMenuOptions()` | 20 | 参数类型也变了 |
| `TextDecoder.decodeWithStream()` | `decodeToString()` | 12 | 直接改名 |

### 各版本主要废弃项速查

| 版本 | 重点废弃项 | 替代方案 |
|------|----------|---------|
| 5.0.0 | 部分 `@ohos.*` 旧路径 API | 迁移到 Kit 路径（`kit.*`） |
| 5.0.3 | `kit.StoreKit` | `kit.AppGalleryKit` |
| 6.0.0 | 部分 ArkUI 旧属性写法 | 新属性 API |
| 26.0.0 | FA 模型 ArkUI 接口 | 迁移到 Stage 模型 |

> **注意**：`@ohos.xxx` 旧 import 路径（如 `@ohos.file.fs`、`@ohos.hilog`）在 API 26 **仍可用，不报 deprecated 告警**，不属于必须修改范围。只有编译告警标记 `@deprecated` 的才必须改。

---

## 代码模式

### 模式 1：组件内获取 UIContext（有 this）

```typescript
// ✅ 组件内直接用 this.getUIContext()
this.getUIContext().px2vp(value)
this.getUIContext().getPromptAction().showToast({ message: 'xxx' })
this.getUIContext().animateTo({ duration: 300 }, () => { /* ... */ })
this.getUIContext().getRouter().pushUrl({ url: 'pages/Detail' })
```

### 模式 2：有 window.Window 实例可用

```typescript
// ✅ window.Window 有 getUIContext() 方法
windowClass.getUIContext().px2vp(value)
// 或从 windowStage 获取
windowStage.getMainWindowSync().getUIContext().animateTo({ duration: 300 }, () => { /* ... */ })
```

### 模式 3：工具函数加 UIContext 参数（推荐）

```typescript
// ✅ 给函数加 uiContext 参数，调用者传入
function myUtil(uiContext: UIContext, value: number) {
  return uiContext.px2vp(value)  // 直接用参数
}
// 调用者：
myUtil(this.getUIContext(), value)
```

### 模式 4：工具类静态持有者（调用者多时）

```typescript
// ✅ 工具类加静态 setUIContext()，在 EntryAbility 注入
class PopViewUtil {
  private static context?: Context
  static setContext(ctx: Context) { PopViewUtil.context = ctx }
  static showToast(msg: string) {
    PopViewUtil.context?.resourceManager.getString(/* ... */)
  }
}
// EntryAbility.onWindowStageCreate 的 loadContent 回调内注入
windowStage.loadContent('pages/Index', () => {
  PopViewUtil.setContext(windowStage.getMainWindowSync().getUIContext().getHostContext()!)
})
```

> 多模块工程推荐用 AppUtil 全局持有者模式（见 `references/migration-archive.md` 附录模式 C）。

---

## 迁移易错点

迁移 deprecated API 时，以下情况容易**误改**或**漏改**，必须核验：

**1. 第三方库别名混淆**
部分第三方库 import 时用 `as` 别名，与废弃 API 同名。
```typescript
// 这是第三方库别名，不是废弃的 promptAction —— 不能改！
import { ToastDialog as promptAction } from '@hw-agconnect/ui-toast';
promptAction.showToast({...})  // 实际是 ToastDialog.showToast，非废弃
```
**判断方法**：迁移 `promptAction.showToast` 前，先看 import 来源。只有 `from '@kit.ArkUI'` 或 `from '@ohos.promptAction'` 的才是废弃 API。

**2. 命名遮蔽（变量名遮蔽模块名）**
```typescript
// tag 是构造函数参数（IsoDepTag 实例），遮蔽了 @ohos.nfc.tag 模块名
constructor(tag: tag.IsoDepTag) {
  tag.getTagInfo()  // 是实例方法，不是废弃的全局 tag.getTagInfo
}
```
**判断方法**：看到 `xxx.getMethod()` 报废弃时，检查 xxx 是模块名还是局部变量。

**3. 闭包内 context 判空（getHostContext 返回 undefined）**
`getHostContext()` 返回 `Context | undefined`。即使函数开头判空，闭包（回调函数）内仍报 undefined：
```typescript
// ❌ 闭包内报错：'context' is possibly 'undefined'
let context = this.uiContext?.getHostContext();
if (!context) return;
someApi.then(() => {
  fileIo.unlink(context.cacheDir)  // 闭包内控制流分析重置，仍认为可能 undefined
})

// ✅ 闭包内用非空断言
fileIo.unlink(context!.cacheDir)
```

**4. getStringSync 重载区分**
`resourceManager.getStringSync` 有多个重载，只有部分废弃：
- `getStringSync(resId: number)` — 非废弃
- `getStringSync(resource: Resource)` — 非废弃
- `getStringSync(resId: number, ...args)` — 部分参数组合废弃
迁移前确认用的是哪个重载。

**5. @ohos.xxx 模块路径迁移**
部分旧模块路径（如 `@ohos.promptAction`）本身已废弃，需迁移到 `@kit.ArkUI`。但这不报 deprecated 告警，需要主动检查 import。

**6. 字段初始化阶段不能用 UIContext（运行时崩溃！）**

这是 UIContext 迁移**最危险的坑**——编译能过，但运行时崩溃。

**原因**：`getUIContext()` / `ContextManager.getUIContext()` 在组件**字段初始化阶段**（属性声明时赋值）可能返回 undefined，因为此时组件还没挂载到窗口树。`!` 非空断言骗过编译器，但运行时 `getRouter of undefined` 崩溃。

```typescript
// ❌ 崩溃！字段初始化时组件还没上树，getUIContext() 返回 undefined
@Entry @Component
struct MyPage {
  params: object = ContextManager.getUIContext()!.getRouter().getParams()  // 崩溃
  width: number = ContextManager.getUIContext()!.vp2px(100)                // 崩溃
}

// ✅ 延迟到 aboutToAppear 赋值（此时组件已上树）
@Entry @Component
struct MyPage {
  params: object = {}
  width: number = 0
  aboutToAppear() {
    this.params = this.getUIContext().getRouter().getParams()
    this.width = this.getUIContext().vp2px(100)
  }
}
```

**排查方法**：
```bash
grep -rn "getUIContext()!\." --include="*.ets" . | grep -v "this\.getUIContext\|aboutToAppear\|build()"
grep -rn "ContextManager.getUIContext()!\." --include="*.ets" .
```

**7. ContextManager 必须在 loadContent 回调里注入 UIContext**

如果用了 ContextManager 静态持有者模式，UIContext 注入必须在 `loadContent` 的**回调函数里**——不能在 loadContent 之前调用，否则 `getMainWindowSync().getUIContext()` 会空指针：
```typescript
onWindowStageCreate(windowStage: window.WindowStage): void {
  windowStage.loadContent('pages/SplashPage', localStorage, () => {
    // ✅ 必须在回调里：此时窗口内容已加载，UIContext 已创建
    ContextManager.setUIContext(windowStage.getMainWindowSync().getUIContext())
  })
  // ❌ 不能在这里：loadContent 之前 UIContext 还没创建，会空指针
}
```
注意：loadContent 是异步的，回调执行时机晚于 SplashPage 的字段初始化。所以 SplashPage 字段初始化阶段仍然拿不到 ContextManager 注入的 UIContext——字段初始化的 UIContext 调用必须移到 aboutToAppear（见易错点 #6）。

---

## 常见废弃模式

### 1. Kit 更名 / 路径迁移
Kit 从一个模块迁移到另一个，旧 import 路径废弃。例如：
- `kit.StoreKit` -> `kit.AppGalleryKit`（5.0.3 变更）
- 旧版 `@ohos.xxx` -> 新版 Kit 路径

> **注意**：`@ohos.xxx` 旧 import 路径在 API 26 **仍可用，不报 deprecated 告警**，不属于必须修改范围。

### 2. FA 模型 API 废弃
Stage 模型全面替代 FA 模型。FA 模型特有的 API（PageAbility 相关）在 API 10+ 已标记废弃。

### 3. 状态管理 V1 装饰器（必须迁移）
V1 装饰器（@State/@Prop/@Link 等）在 26.0.0 虽然还能编译运行，但**升级时必须迁移到 V2**。详见 `harmonyos-behavior-changes` 的状态管理迁移指南。

### 4. 接口签名变更
部分接口的参数列表或返回类型变更，旧签名废弃。编译时通常报类型不匹配错误。

---

## 验证清单

**基础验证（每次必做）：**
- [ ] `hvigorw clean` + 全量编译已执行
- [ ] deprecated 告警已统计（TOTAL - oh_modules = 工程代码数）
- [ ] 工程代码 deprecated 告警 = 0

**压力验证（发布前做）：**
- [ ] 工具类 getContext() 已迁移（加 context 参数或静态持有者）
- [ ] 字段初始化阶段无 getUIContext() 调用（排查命令已跑）
- [ ] ContextManager 注入在 loadContent 回调内
- [ ] 闭包内 getHostContext() 返回值用非空断言 `!`
- [ ] 第三方库别名未误改（import 来源已核实）

---

## 常见问题

**Q：能不能用 grep 找废弃 API 代替编译**
A：不能。编译器有完整类型信息 + SDK 的 @deprecated 注解，能精确发现所有废弃点（含 Resource 重载废弃、import 别名、对照表外的新废弃 API——这些 grep 都查不到）。grep 还会误报（把已迁移的 `.px2vp(` 也匹配上）。

**Q：工具类的 getContext() 能不能保留告警不改**
A：不能。工具类的 `getContext()` 也要改：改函数签名加 context 参数，由调用者传入 UIContext。即使调用链长也必须改。或用静态持有者模式（AppUtil / ContextManager）。

**Q：迁移后运行时崩溃 "getRouter of undefined"**
A：字段初始化阶段调用了 `getUIContext()`——此时组件还没挂载到窗口树，getUIContext() 返回 undefined。`!` 非空断言骗过编译器但运行时崩溃。修复：移到 `aboutToAppear` 赋值。

**Q：闭包里报 "context is possibly undefined" 但函数开头已判空**
A：闭包（回调函数）内控制流分析会重置，即使函数开头判空了，闭包内仍认为可能 undefined。用非空断言 `!`：`context!.cacheDir`。

**Q：ContextManager 注入后还是空指针**
A：`setUIContext` 必须在 `loadContent` 的回调函数里调用，不能在 loadContent 之前。loadContent 是异步的，回调执行时机晚于 SplashPage 的字段初始化——所以 SplashPage 字段初始化的 UIContext 调用仍需移到 aboutToAppear。

**Q：@ohos.file.fs 这种旧 import 路径要改吗**
A：不需要。`@ohos.xxx` 旧 import 路径在 API 26 仍可用，不报 deprecated 告警。它们只是有了 `@kit.*` 别名。只有编译告警标记 `@deprecated` 的才必须改。

**Q：AlertDialog.show 和 AlertDialog 组件有什么区别**
A：`AlertDialog.show(options)` 是废弃的全局函数，要改成 `this.getUIContext().showAlertDialog(options)`。而 `AlertDialog({ title: 'xxx' })` 是 ArkUI 的组件构建器形式（用于 CustomDialogController），不是废弃 API，不要改。

---

## 延伸阅读

- `references/migration-archive.md`：废弃 API 迁移档案——每个 API 的替代签名、代码示例、踩坑记录（23 条 + 通用易错 + 附录 3 种工具类迁移模式）——当需要查阅具体 API 的迁移代码时读取。
- `../harmonyos-behavior-changes/SKILL.md`：行为变化适配——API 还在但行为变了时查
- `../harmonyos-upgrade-verify/SKILL.md`：编译验证与错误码——编译报错修复时查
- `../harmonyos-project-upgrade/SKILL.md`：升级总入口——废弃 API 检查在升级流程中的位置
