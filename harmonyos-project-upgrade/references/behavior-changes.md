# 行为变化适配 + 状态管理迁移

> 本文件为 `../SKILL.md` 的补充，覆盖升级流程步骤4「状态管理V1→V2迁移」与步骤5「行为变化适配」。当 SKILL.md 路由到这两个环节时读取本文件。

## Agent Interface

```yaml
symptom_keywords:
  - Toast/Dialog style changed after upgrade
  - touch hot zone height changed (28→32vp)
  - text layout changed (orphan line wrap, syllable break)
  - shadow radius=0 now shows shadow without blur
  - rawDeltaX/rawDeltaY values changed
  - NodeAdapter onAttachToNode callback timing changed
  - matchParent layout broke after upgrade
  - need to migrate @State to @Local / @Component to @ComponentV2
  - @Watch to @Monitor — timing changed (sync to async)
  - @Local property cannot be initialized externally (error 10905324)
  - V2 component cannot use V1 @Link system component (error 10905213)

hard_constraints:
  - State management V1→V2 migration is mandatory during upgrade — not optional; pure V1 projects must do full migration with no V1 residual (no dual-write bridge)
  - WidgetCard (form modules, widget/ directories) must NOT be migrated to V2 — @ComponentV2 compatibility is poor for cards; keep V1 decorators
  - A single component cannot mix V1 and V2 decorators — but parent and child components can each use different versions
  - @Local cannot be externally initialized — if a parent passes the value, use @Param (or @Param @Once); only pure internal state uses @Local (otherwise error 10905324)
  - @Watch (V1, synchronous) vs @Monitor (V2, asynchronous) — @Watch executes N times for N changes, @Monitor executes once after the event; code depending on synchronous execution must be adjusted
  - V2 components using V1 @Link system components (SegmentButton, TextReaderIcon, etc.) will error 10905213 — must replace with V2 versions; components without V2 alternatives (ComposeTitleBar, Chip, TreeView, etc.) keep @Component
  - Block B behavior changes: only adapt items where grep hits project code — never do preventive modification without grep hits; two-step gating: grep hit → read context to confirm it matches the change description → only then modify

diagnostic_checklist:
  - Is this a V1→V2 migration question (Block A) or a behavior change question (Block B)?
  - For Block A: Has the migration-guide.md been read for the 20-item decorator mapping table?
  - For Block B: Has grep been run for each of the 25 change items' keywords?
  - For grep hits: Has the hit line's context been read to confirm it matches the change description (two-step gating)?
  - For @State→@Local: Has it been checked whether the property is externally initialized by parent (use @Param instead)?
  - For @Watch→@Monitor: Has the sync→async timing change been reviewed for business logic impact?
  - For V2 migration: Are there V1 system components (SegmentButton, TextReaderIcon, etc.) that need V2 replacement?
```

# 行为变化适配 + 状态管理迁移

## 适配领域

本文件覆盖升级流程的两个环节：**块 A**（步骤4）状态管理 V1→V2 迁移——升级时必须迁移，不是可选项；**块 B**（步骤5）接口行为变化适配——基于 25 项变更，用 grep 扫项目代码，命中的才适配。

**不覆盖**：API 废弃替换（见 `deprecated-apis.md`）；版本检测（见 `upgrade-detect.md`）；配置修改（见 `upgrade-config.md`）；编译验证（见 `upgrade-verify.md`）。

---

## 两块内容矩阵

| | 块 A：状态管理 V1→V2 迁移 | 块 B：接口行为变化 |
|---|---|---|
| 管什么 | V1 装饰器迁 V2（@State→@Local 等） | API **还在**，但行为变了 |
| 怎么发现 | 查装饰器对照表；V1 装饰器 grep 统计 | 用 25 项变更的 grep 关键词扫项目代码 |
| 适配原则 | 升级时**必须迁移**，不留 V1 残留 | **grep 命中才适配**，不允许预防性修改 |
| 与 deprecated-apis 的关系 | — | deprecated-apis 管"API 没了要换替代"，本块管"API 还在但行为变了" |
| 参考文件 | `references/migration-guide.md` | 本 SKILL.md 内嵌表格 |

### 如何判断用户要哪块

| 用户问的 | 读哪个 |
|---------|--------|
| "@State 怎么改" / "@Watch 改 @Monitor" / "@Component 改 V2" | 块 A：读 migration-guide.md |
| "升级后行为变了啥" / "Toast 样式不对" | 块 B：用下方表格的 grep 关键词扫项目，命中的才适配 |
| 不确定 | 块 B 逐项 grep 扫一遍；状态管理看 migration-guide.md |

---

## 适配流程

### 新适配

#### 块 A：状态管理 V1→V2 迁移

> **仅路径B（→26.0.0）需要迁移**。路径A（→6.1.0）**不涉及 V2 迁移**——V1 装饰器在 6.1.0 正常运行，该环节跳过。

**路径B 升级时必须迁移**——这是升级到 26.0.0 的必要环节，不是可选项。详见 `migration-guide.md`（20 项装饰器对照表 + 机制差异 + 5 步迁移步骤 + 每条规则的代码对比 + 迁移易错点）。

迁移顺序（基础先行）：

1. **应用级状态**：@StorageLink/@StorageProp/AppStorage → 定义 @ObservedV2 数据类 + AppStorageV2.connect
2. **数据类**：@Observed → @ObservedV2，属性加 @Trace；@ObjectLink → @Param
3. **跨层级**：@Provide → @Provider，@Consume → @Consumer
4. **组件级**：@Component → @ComponentV2（叶子优先，根在后）
   - @State → @Local（纯内部）或 @Param（外部传入）
   - @Prop → @Param
   - @Link → @Param + @Event（父子双向）
5. **监听**：@Watch → @Monitor（注意：同步→异步时序变化）

> **排除 WidgetCard**：`widget/`、`widgetcard/` 目录或 module.json5 里 `type: "form"` 的模块不迁 V2——WidgetCard 对 @ComponentV2 兼容性不好，保留 V1 装饰器。
>
> **纯 V1 工程**（0 个 @ComponentV2）：全量迁移，不留 V1 残留（不用双写桥等中间态），组件级按依赖关系排序，叶子组件先迁。

#### 块 B：接口行为变化（grep 命中才适配）

**按路径累积扫多张表**（版本隔离是 `targetSdk ≥ X`，设得越高触发越多，必须累积）：
- 路径A（→6.1.0）：扫「6.0.0 表」+「6.1.0 表」
- 路径B（→26.0.0）：扫「6.0.0 表」+「6.1.0 表」+「26.0.0 表」

见下方「行为变化速查表（按版本累积）」章节。

**铁律：只适配项目代码命中的变更项。不允许没命中就预防性修改。** 用对应版本速查表的「grep 关键词」扫项目代码，命中的才按「适配方法」处理。没命中的跳过。

**如何判定命中（两步门控，缺一不可）：**

只允许在 grep 命中且二次确认符合变更描述时修改代码。不允许凭"可能受影响"就动代码。

**第 1 步：grep 扫描（证明项目用了这个 API/组件）**

```bash
cd <工程根目录>
# 示例：查项目是否用了 rawDeltaX（鼠标事件行为变化）
grep -rn "rawDeltaX\|rawDeltaY" --include="*.ets" --exclude-dir=oh_modules --exclude-dir=node_modules .
# 有输出 = 进入第 2 步；无输出 = 跳过该项，不改任何代码
```

**第 2 步：二次确认命中行的上下文是否符合变更描述（证明真的受影响）**

grep 命中只能证明"用了这个 API"，不能证明"用了变更影响的那个具体场景"。必须读命中行的上下文，对照变更描述逐项确认：

| 常见需二次确认的情况 | 确认方法 |
|-------------------|---------|
| 变更只影响特定参数值 | 读命中行，确认是否真用了那个参数值。例：`shadow` 变更只影响 radius=0 或负数，如果命中行 radius 是 50 → 不受影响，不改 |
| 变更只影响特定调用形式 | 确认是废弃的调用形式还是同名自定义方法。例：grep 到 `getContext(` 要看 import 来源，排除项目自定义的同名方法 |
| 变更只影响特定场景 | 确认命中代码是否真的在该场景下运行。例：`onAttachToNode` 变更只影响 NodeAdapter 绑定场景，如果命中是别的回调 → 不改 |

**铁律**：
- grep **无命中** → 跳过该项，不改任何代码。不允许"预防性修改"
- grep **有命中但二次确认不符合变更描述** → 不改，在交付清单标注"grep 命中 N 处但经确认不受变更影响"
- grep **有命中且二次确认符合** → 按「适配方法」修改，并在交付清单列出改了哪些文件/行
- **交付清单必须逐项列出**：命中的变更项、grep 命中数、是否实际修改及原因。没命中的也要列（标注"未命中"），让用户可核查

---

### 问题定位

**块 A（V1→V2 迁移）常见问题：**

| 现象 | 可能原因 | 修复 |
|------|---------|------|
| 编译报 10905324 @Local 不能外部初始化 | @State 迁 @Local 但属性被父组件传值 | 改用 @Param 或 @Param @Once |
| 编译报 10905213 V2 组件含 V1 @Link | V2 组件内用了 V1 系统组件（SegmentButton 等） | 有 V2 替代的必须替换；无替代的该 struct 保留 @Component |
| V2 迁移后 UI 不刷新 | @Trace 漏加 / @Watch→@Monitor 时序变化 / ForEach key 不变断链 | 查 migration-guide.md 运行时陷阱 |
| @Watch 迁 @Monitor 后逻辑异常 | @Watch 同步执行多次，@Monitor 异步执行一次 | 调整依赖同步执行的逻辑 |

**块 B（行为变化）常见问题：**

| 现象 | 可能原因 | 修复 |
|------|---------|------|
| Toast/Dialog 样式变了 | 沉浸式系统材质默认开启（targetSdk≥26） | 确认视觉效果可接受 |
| 触摸热区变大 | 表单类组件热区最小高度 28→32vp（targetSdk≥26） | 确认布局不被 4vp 增幅撑破 |
| shadow radius=0 出现阴影 | radius=0 从无阴影变为有阴影无模糊（targetSdk≥26） | 如需无阴影改 radius 为 -1 |
| rawDeltaX/Y 数值变了 | 鼠标事件返回值从"原始数据/缩放比例"变为真实硬件数据 | 如需旧行为用 px2vp() 转换 |

---

## 关键 API

### V1→V2 装饰器对照

**用途**：升级时将 V1 装饰器迁移到 V2。完整 20 项对照表见 `references/migration-guide.md`。

**核心映射**：
- `@Component` → `@ComponentV2`
- `@State` → `@Local`（纯内部）或 `@Param`（外部传入）
- `@Prop` → `@Param`
- `@Link` → `@Param` + `@Event`
- `@Watch` → `@Monitor`（同步→异步）
- `@Provide` → `@Provider`，`@Consume` → `@Consumer`
- `@Observed` → `@ObservedV2` + `@Trace`
- `@ObjectLink` → 直接用 `@Param`

**注意**：@Local 不可外部初始化（否则报 10905324）；@Watch 同步→@Monitor 异步是时序变化（最易踩坑）。

### grep 扫描命令

**用途**：块 B 行为变化的两步门控第一步——证明项目用了某个 API/组件。

**注意**：每项变更的「grep 关键词」是 changelog 里「变更的接口/组件」字段列出的 API/组件名。无命中 = 跳过，不允许预防性修改。

---

## 行为变化速查表（按版本累积）

⚠️ **版本隔离的阈值是 `targetSdkVersion ≥ 某版本`，不是 `=`**。所以 targetSdk 设得越高，触发的版本隔离项越多——**必须累积适配所有 ≤ 目标版本的行为变化**，不能只查目标版本自己那张表。

| 路径 | targetSdk | 要扫的版本（累积） |
|------|-----------|------------------|
| **路径A**（→6.1.0） | 6.1.0(23) | 6.0.0 的表 + 6.1.0 的表 |
| **路径B**（→26.0.0） | 26.0.0 | 6.0.0 的表 + 6.1.0 的表 + 26.0.0 的表 |

> 路径B 是路径A 的超集（多一个 26.0.0 的表）。每张表内分「针对所有应用」(无版本隔离,必扫) 和「版本隔离」(targetSdk 到该版本阈值才触发) 两类。路径A 的 targetSdk=6.1.0(23) 已 ≥ 6.0.0(20) 和 6.1.0(23)，所以两个版本的版本隔离项都会触发。

---

### 6.0.0（API20）的行为变化

> 路径A 和路径B 都要扫这张表。来源：6.0.0 官方 changelog（changelogs-for-all-apps-6001~6004）。

#### 针对所有应用（无版本隔离，任何 targetSdk 生效）

| grep 关键词 | 变更描述 | 命中后的适配方法 |
|------------|---------|----------------|
| `AbilityDelegator` 相关错误码（`COMMON_EVENT_PACKAGE_ADDED`/`PACKAGE_REMOVED`/`PACKAGE_CHANGED`/`PACKAGE_RESTARTED`/`PACKAGE_DATA_CLEARED`/`PACKAGE_CACHE_CLEARED`/`QUICK_FIX_APPLY_RESULT`） | Ability Kit 公共事件（PACKAGE_ADDED 等 7 个）增加管控：三方应用由「可监听所有应用事件」变为「仅能监听自身应用事件」 | 若用这些事件判断其他应用是否安装，改用 `bundleManager.canOpenLink` 查询 |
| `delete arr[`（字面量数组 + delete） | 字面量定义的数组 delete 元素后再次使用该字面量，新数组由「保留删除后状态」变为「字面量原始值」 | 排查三方库/js 文件中「字面量数组 + delete 元素后复用」的逻辑，按修复后行为校正 |
| `CustomDialogController = {`（字面量初始化） | 因 `CustomDialogController` 新增 `getState()`，字面量 `{}` 初始化实例由「可编译」变为「编译报错」 | 改用 `new CustomDialogController()` 创建实例 |
| `Web(` 组件 / ArkWeb | ArkWeb 内核由 Chromium 114 升级到 132（50 项 W3C 规格变更） | 若用 Web 组件展示网页，按 ArkWeb_114_132 差异总结核查 W3C 规格变更并适配 |
| `userAuth` `.start()` / `ACCESS_BIOMETRIC` / `USER_AUTH_FROM_BACKGROUND` | 后台发起带交互界面身份认证：变更前凭 `ACCESS_BIOMETRIC` 即可前后台调用 `start()`；变更后后台调用需系统权限 `USER_AUTH_FROM_BACKGROUND`，三方应用不能后台发起 | 不涉及后台认证则无需适配；系统应用需申请 `USER_AUTH_FROM_BACKGROUND`，三方应用需改为前台发起 |
| `LocationButton` / `location_button`（位置控件） | 位置控件接口从 SDK 删除（API15 已废弃）；存量应用升级镜像后仍可用但每次点击都弹确认框 | 移除位置控件，改用权限弹窗申请位置授权 |
| `drawModifier(` / `.modifier = undefined` / `invalidate(` | drawModifier 两点变更：(1) 参数置 undefined 由「不清除原 modifier」变为「清除为 undefined」；(2) 重绘规则由「设了 drawModifier 即重绘」变为「仅未跳过测量的节点才重绘」 | (1) 若依赖置 undefined 不生效的旧行为，注释掉该行；(2) 自定义绘制内容变化后手动调用 `modifier.invalidate()` 触发重绘 |
| `bindSheet(` `SheetType.SIDE` `keyboardAvoidMode` | 半模态 SIDE 侧边样式由「不避让软键盘」变为「默认避让软键盘（TRANSLATE_AND_SCROLL）」 | 若需自定义避让，设 `keyboardAvoidMode = SheetKeyboardAvoidMode.NONE` |
| `context.font =`（CanvasRenderer / OffscreenCanvas font） | CanvasRenderer 的 font 接口设置自定义字体由「不生效显示默认字体」变为「生效显示自定义字体」 | 若需保持默认字体行为，不设置自定义字体即可 |
| `SaveButton` `tipPosition(` / `SaveButtonTipPosition` / `ABOVE_BOTTOM` / `BELOW_TOP` | 保存控件系统提示弹框取消，`tipPosition` 接口及 `SaveButtonTipPosition` 枚举删除 | 移除对 `tipPosition` 的调用（否则编译/运行失败），改由应用自行实现保存提示 |
| `zlib.unzipFile(` / `zlib.decompressFile(` | 格式有误的压缩包由「能解压成功但内容不符」变为「解压失败并抛文件格式错误码」 | 调用时 try-catch 捕获异常，按错误码检查压缩包格式 |
| `harmonyShare.on('dataReceive'` / `RecvCapabilityRegistry` / `capabilities` | 沙箱接收 `on('dataReceive')` 的 `RecvCapabilityRegistry` 新增必填参数 `capabilities`（配置可接收数据类型及最大数量） | 注册/解注册时补填 `capabilities`（含 utd + maxSupportedCount），未填编译报错 |
| Car Kit 接口（`getNavigationController(` / `updateNavigationStatus` / `getSmartMobilityAwareness` / `on(type: 'smartMobilityEvent'` 等） | Car Kit 公共接口新增错误码 801（设备不支持）、1003810001（非法参数）、1003810002（参数超限），未 try-catch 会崩溃 | 调用这些接口时 try-catch 并按需处理新错误码 |
| `rag.streamRun(` `StreamType.THOUGHT` | rag.streamRun 思考过程输出变更：(1) 由固定「正在思考中...」变为动态正式思考内容；(2) 由「首 token 后输出」变为「提问后立即输出」 | 若 config 配了 THOUGHT 且解析了思考内容或用固定时间做界面等待，改为依据 `stream.type` 刷新界面，不解析固定文案/不等固定时间 |
| `@interface`（JS HAR 包中的注解） | 编译产物为 JS 的 HAR 包（release+混淆）中由「可使用注解」变为「编译报错」（JS 无注解机制，会被移除导致 AOP 失效） | 删除 JS HAR 中的注解声明/调用，或改编译为字节码 HAR |

#### 版本隔离（仅 targetSdkVersion ≥ 6.0.0(20) 生效）

| grep 关键词 | 变更描述 | 命中后的适配方法 |
|------------|---------|----------------|
| `AbilityDelegator.startAbility(` | startAbility 返回错误码全部变更（如 29360128→401、2097199→16000001 等，共 16 项） | 无需适配（错误码与 API 描述对齐）；若按旧错误码判断需更新对照 |
| `FLAG_AUTH_READ_URI_PERMISSION` / `FLAG_AUTH_WRITE_URI_PERMISSION` / `wantConstant.Params.PARAMS_STREAM` / `startAbility(` `startAbilityForResult(` `terminateSelfWithResult(`（文件分享场景） | 文件分享场景下，Want 的 uri scheme 为空或 PARAMS_STREAM 中 URI scheme 不为 file 时，由「不处理」变为「擦除该 URI」 | 排查 flags 是否设了文件 URI 读写 Flag 且 uri/PARAMS_STREAM 写入了非文件 URI；二选一：删除该 Flag，或用 `fileUri.getUriFromPath` 把沙箱路径转为文件 URI |
| `GridRow(` `columns:` / `GridCol(` `span:` | GridRow 不配 columns 时默认值由 12 变为 `{xs:2,sm:4,md:8,lg:12,xl:12,xxl:12}`；columns/span 配部分断点时未配置项由「取更小断点值（无则 12/1）」变为「无更小断点时取更大断点值」 | 默认行为变更，无需适配；若默认/补全布局不符预期，显式配置所有断点的 columns 与 span |
| `LayoutPolicy.matchParent`（width/height） | Row/Column 子组件 matchParent 时大小由「含父 padding/border/safeAreaPadding」变为「父内容区大小（不含 padding/border/safeAreaPadding）」 | 默认行为变更，无需适配 |
| `ble.startAdvertising(` / `ble.enableAdvertising(` / `ble.stopAdvertising(` / GATT Client/Server 接口 | BLE 广播/扫描/GATT 接口异常由「仅返回 2900099」变为「返回细分错误码（2900010/2902054/2902055 等）」 | 打印接口错误码辅助定位；若按 2900099 统一判断需更新为细分错误码处理 |
| `ImageInfo` `.mimeType` | raw/icon 格式图片 mimeType 返回值变更：icon `image/ico`→`image/x-icon`；dng/cr2/raf/nef 等 raw 由 `image/jpeg` 变为各自标准名 | 调用方式不变；若按 mimeType 判断 raw/icon 格式需更新判断字符串 |
| `AVPlayer` | 播放器使用内存归属由「计入操作系统内存」变为「计入本应用内存」 | 同时工作实例 <16 无需变更；超出则调整实例数，避免退后台被 RSS 查杀 |
| `TreeSet` / `TreeMap`（自定义比较器） | TreeSet/TreeMap 扩容导致自定义比较器丢失的 bug 正向修复：扩容前后比较器一致 | 仅当用了自定义比较器且把旧的错误结果当正确结果使用时，需按修复后结果校正 |

---

### 6.1.0（API23）的行为变化

> 路径A 和路径B 都要扫这张表（路径A targetSdk=6.1.0(23) 已触发本表的版本隔离项）。来源：6.1.0 官方 changelog（changelogs-for-all-apps-6101）。

#### 针对所有应用（无版本隔离，任何 targetSdk 生效）

| grep 关键词 | 变更描述 | 命中后的适配方法 |
|------------|---------|----------------|
| 横屏悬浮窗 / `scale` | 横屏悬浮窗布局宽度基准由「屏幕宽度」改为「屏幕高度」（通过调整 scale 值保持用户看到的窗口大小不变） | 若应用监听窗口 size 做了不同布局效果，需根据自身业务代码对应适配 |
| V1 装饰器装饰 Function 类型变量（`@State`/`@Prop`/`@Link`/`@Provide`/`@Consume`/`@StorageLink`/`@ObjectLink` + Function 类型） | struct 中用状态管理 V1 装饰器装饰 Function 类型变量：变更前编译通过但运行时 crash，变更后编译报错（`The V1 decorator 'xxx' cannot be applied to a Function-type variable 'yyy'`） | 将 Function 类型变量上的 V1 装饰器移除，改为常规变量声明 |
| `WordBreak.HYPHENATION` / `WordBreak.BREAK_HYPHEN` / `WORD_BREAK_TYPE_BREAK_HYPHEN` | 捷克语、印度尼西亚语、拉脱维亚语、马其顿语、斯洛伐克语、塞尔维亚语 6 种语言下连字符「-」断词换行不再生效，自动回落到按单词换行 | 审视相关文本展示效果，若受影响按业务代码适配 |
| `intl.DateTimeFormat.format` / `intl.Locale.maximize/minimize` / `i18n.System.getDisplayCountry` / `i18n.Transliterator.transform` / `i18n.TimeZone.getDisplayName` / `Intl.ListFormat.format` / `ubrk_next` 等 | ICU 由 72.1 升级至 74.2，全球化/运行时接口返回值与文本换行行为更贴合地区规范（如繁中日期「2024年4月23日」→「2024 年 4 月 23 日」、邮箱 @ 两侧识别为单词边界、新增 622 个中日韩表意字符等） | 审视依赖这些接口返回值或换行效果的业务逻辑，按需适配 |
| `PhotoKeys.DURATION` | 动态照片（`PhotoSubtype.MOVING_PHOTO`）的 DURATION 字段值由默认 0 变更为附带视频片段时长（无法解析则为 -1） | 若用 DURATION 判断图片/视频分类，改用 `mimeType`（`image/`、`video/`）或 `PhotoKeys.PHOTO_TYPE` 判断；判断动态照片改用 `PhotoKeys.PHOTO_SUBTYPE` |
| JS OOM / JS Crash / CPP Crash | worker/taskpool 线程堆与共享堆发生 JS OOM 时，崩溃类型由 CPP Crash 统一变为 JS Crash（主线程堆原本即 JS Crash 不变） | 审视 OOM 异常的崩溃类型分类与统计逻辑，按需适配 |

#### 版本隔离（仅 targetSdkVersion ≥ 6.1.0(23) 生效）

| grep 关键词 | 变更描述 | 命中后的适配方法 |
|------------|---------|----------------|
| `Progress` / `.color()` / `LinearGradient` | Progress 的 Linear 和 Capsule 样式通过 color 设置渐变色：变更前仅显示默认主题色（蓝色），变更后显示为指定的渐变色 | 想要单色则 color 传 `ResourceColor`；想要渐变色则 value 参数传 `LinearGradient`，按实际需求选择 |

---

### 26.0.0（API26）的行为变化

> **仅路径B** 要扫这张表。来源：26.0.0 官方 changelog（changelogs-for-all-apps-7001）。

#### 无版本隔离（targetSdk < 26 也生效，必须适配）

| grep 关键词 | 变更描述 | 命中后的适配方法 |
|------------|---------|----------------|
| `ArkWeb` + `Web(` | ArkWeb 内核 132→144，100+ W3C 规格变更 | 查 [ArkWeb 132→144 差异总结](https://gitcode.com/openharmony-tpc/chromium_src/blob/master/web/ReleaseNote/ArkWeb_132_144.md)，逐项核对 Web 页面 |
| `JSVM` `import.*jsvm` | JSVM 内核 132→144，38 项需注意的变更 | 查 [JSVM 132→144 差异总结](https://gitcode.com/openharmony/arkcompiler_jsvm/blob/master/ReleaseNote/JSVM_132_144.md)，逐项核对 |
| `isAsyncFunction` | getter/setter 后的 async 函数，`util.types().isAsyncFunction` 返回值从 false→true | 排查依赖 async 类型判断的代码逻辑 |
| `fastConvertToJSObject` | XML 解析不再丢失与子元素同级的 text 节点 | 检查 XML 解析结果是否依赖旧的丢失行为 |
| `rawDeltaX` `rawDeltaY` | 鼠标事件返回值从"原始数据/缩放比例"变为真实硬件数据 | 如需旧行为用 `this.getUIContext().px2vp()` 转换 |
| `READ_IMAGEVIDEO` | 该权限只能读本地图片/视频，不含云端 | 如需云端资源改用 `PhotoViewPicker` |
| `queryNavDestinationInfo` `onResult` | 主页 NavDestination 的 queryNavDestinationInfo 现在能获取信息了；onResult 不再误触发 | 审视主页 NavDestination 的 onResult 逻辑是否依赖旧行为 |
| `reuse(` `@ReusableV2` | reuseId 支持动态回调，复用标识从组件名变为回调返回值 | 检查 V2 复用组件的 reuseId，确认复用行为符合预期 |
| `defaultFocus` `requestFocus` | FA 模型的 ArkUI API（version 10+）编译报错 | 确认工程已是 Stage 模型（detect 步骤会检查） |

### 版本隔离（仅 targetSdkVersion ≥ 26.0.0 生效，命中后需测试确认）

| grep 关键词 | 变更描述 | 命中后的适配方法 |
|------------|---------|----------------|
| `COMMON_EVENT_PACKAGE_ADDED` `COMMON_EVENT_PACKAGE_REMOVED` `COMMON_EVENT_PACKAGE_CHANGED` `COMMON_EVENT_PACKAGE_CACHE_CLEARED` | 订听这些公共事件需 In-House 应用在 app.json5 配置 allowListenBundleChangedEvent | 如监听了这些事件，确认权限配置 |
| `CompileWasmModule` `CompileWasmFunction` `CreateWasmCache` | JSVM Wasm 接口不再返回 JSVM_JIT_MODE_EXPECTED，改为返回 JSVM_OK | 审视返回值从错误码变 JSVM_OK 对逻辑的影响 |
| `onAttachToNode` `attachNodeAdapter` `RegisterEventReceiver` | NodeAdapter 回调从"上树时"变为"绑定时"（更早触发） | 在 attachNodeAdapter **之前**设置 onAttachToNode 回调；依赖主树的操作移到 onAppear |
| `ParagraphStyle` `ImageAttachment` `CustomSpan` | 属性字符串段落首个占位为 CustomSpan/ImageAttachment 时，段落样式现在生效 | 审视段落样式展示效果是否符合预期 |
| `LayoutPolicy.matchParent` | Row/Column/Flex 内单方向 matchParent 的子组件现在参与父组件尺寸计算 | 确认布局不被撑破；如需旧行为手动设父组件尺寸 |
| `EmbeddedComponent` | 层级页面切换导致焦点转移时，焦点不再下发到子节点（停留在根容器） | 如需子节点获焦，设 defaultFocus(true) 或 requestFocus |
| `WithTheme` `setDefaultTheme` | 弹窗类组件跟随宿主 WithTheme 样式；动态添加的组件响应主题变化；TextArea 新增 WithTheme 支持 | 审视 WithTheme 下组件样式是否符合预期；如需旧行为用 ThemeColorMode.SYSTEM |
| `NODE_SWIPER_EVENT_ON_CONTENT_DID_SCROLL` | 回调值 data[3].f32 现在符合实际页面主轴长度 | 去掉之前手动乘屏幕像素密度的补偿代码 |
| `shadow(` `.shadow` `itemShadow` `symbolShadow` | 阴影 radius=0 从无阴影变为有阴影无模糊；负数从按0处理变为无阴影 | 如需无阴影改 radius 为 -1；如需有阴影无模糊用 0 |
| `getUidRxBytes` `getUidTxBytes` | 查询非自身应用流量需申请 GET_NETWORK_STATS 权限（仅 MDM） | 审视是否查其他应用流量；非 MDM 需调整逻辑 |
| `.kernel` `INHERIT_PARENT_PERMISSION` | binary 不再自动继承应用的 kernelpermission，需主动声明 | 为 bin 配置权限策略段并签名（见 changelog 适配步骤） |

### UX 变更（targetSdk ≥ 26 生效）

| grep 关键词 | 变更描述 | 命中后的适配方法 |
|------------|---------|----------------|
| `Button(` `Toggle(` `Select(` `Chip(` `ChipGroup(` | 表单类组件触摸热区最小高度 28→32vp | 确认布局不被 4vp 增幅撑破 |
| `showToast` `showAlertDialog` `showActionSheet` `showDialog` `openCustomDialog` `bindPopup` `bindSheet` | Dialog/Toast/AlphabetIndexer/文本选择菜单默认开启沉浸式系统材质 | 确认弹窗视觉效果可接受 |
| `Text(` `TextInput(` `TextArea(` `Search(` 等（大量内置文本组件） | 内置文本样式优化：孤字换行/小语种行高/音节换行 | 确认文本布局正常（此项命中面广，重点测试） |
| （无特定 grep 关键词，凡含文本即涉及） | notofonts 三方件小语种字体升级，影响小语种文本渲染 | 如应用支持小语种，重点测试文本渲染效果 |

---

## 代码模式

### 模式 1：@State → @Local 或 @Param

```typescript
// V1：@State 可外部初始化，也可内部
@Component
struct MyView {
  @State count: number = 0;
}

// V2 情况A：纯内部状态 → @Local（不可外部初始化）
@ComponentV2
struct MyView {
  @Local count: number = 0;
}

// V2 情况B：需要外部传入一次 → @Param @Once
@ComponentV2
struct MyView {
  @Param @Once count: number = 0;
}
```

### 模式 2：@Link → @Param + @Event（父子双向）

```typescript
// V1：@Link 自动双向同步
@Component
struct Child {
  @Link count: number;   // 改这里会同步回父组件
}
// 父组件
@State parentCount: number = 0;
Child({ count: this.parentCount })

// V2：@Param + @Event 手动实现双向
@ComponentV2
struct Child {
  @Param count: number = 0;
  @Event onCountChange: (n: number) => void = (n: number) => {};
  build() {
    Button('inc').onClick(() => this.onCountChange(this.count + 1))
  }
}
// 父组件
@Local parentCount: number = 0;
Child({
  count: this.parentCount,
  onCountChange: (n: number) => { this.parentCount = n; }
})
```

### 模式 3：@Watch → @Monitor（时序变化）

```typescript
// V1：@Watch 同步执行，变量改 N 次回调执行 N 次
@Component
struct MyView {
  @State @Watch('onChange') count: number = 0;
  onChange() { /* count 每次改都立即执行 */ }
}

// V2：@Monitor 异步执行，变量改 N 次回调只执行 1 次（事件结束后）
@ComponentV2
struct MyView {
  @Local count: number = 0;
  @Monitor('count')
  onChange(mon: IMonitor) {
    // 当前事件结束后才异步执行一次
    mon.dirty.forEach(path => console.info(`changed: ${path}`));
  }
}
```

### 模式 4：@Observed + @ObjectLink → @ObservedV2 + @Trace

```typescript
// V1：@Observed 配 @ObjectLink，只能观测一层
@Observed
class User {
  name: string = '';
}
@Component
struct Child {
  @ObjectLink user: User = new User();
}

// V2：@ObservedV2 + @Trace，支持深度观测，不需要 @ObjectLink
@ObservedV2
class User {
  @Trace name: string = '';   // 需追踪的属性加 @Trace
}
@ComponentV2
struct Child {
  @Param user: User = new User();   // 直接 @Param，不再需要 @ObjectLink
}
```

### 模式 5：应用级状态 @StorageLink → AppStorageV2

```typescript
// V1
AppStorage.setOrCreate('isHover', false);
@Component
struct MyView {
  @StorageLink('isHover') isHover: boolean = false;
}

// V2
@ObservedV2
class AppState {
  @Trace isHover: boolean = false;
}
AppStorageV2.connect(AppState, 'appState', () => new AppState())!;
@ComponentV2
struct MyView {
  @Local appState: AppState = AppStorageV2.connect(AppState, 'appState', () => new AppState())!;
}
```

---

## 验证清单

**基础验证（每次必做）：**
- [ ] 块 A：所有 V1 装饰器已迁移到 V2（WidgetCard 除外）
- [ ] 块 A：@Watch 改 @Monitor 的地方，时序变化不影响业务逻辑
- [ ] 块 A：@Link 改 @Param + @Event 的地方，父子双向同步正确
- [ ] 块 B：25 项变更逐项 grep 扫描完成
- [ ] 块 B：grep 命中项已二次确认上下文

**压力验证（发布前做）：**
- [ ] 块 A：@State 改 @Local 前已检查是否被父组件传值（被传值的用 @Param）
- [ ] 块 A：V2 组件内无 V1 @Link 系统组件（有 V2 替代的已替换，无替代的保留 @Component）
- [ ] 块 A：@ObservedV2 类的 @Trace 属性完整（漏加导致不触发 UI 更新）
- [ ] 块 B：交付清单逐项列出（命中的变更项、grep 命中数、是否实际修改及原因）
- [ ] 块 B：grep 无命中的项也列出（标注"未命中"）

---

## 常见问题

**Q：@State 改 @Local 后编译报 10905324 "cannot be initialized here"**
A：@Local 只允许内部初始化，禁止外部传入。如果父组件调用时传了该属性，改用 @Param（需要外部传入）或 @Param @Once（只同步一次）。迁移前先检查该属性是否被父组件传值初始化——被传值的必须用 @Param，只有纯内部状态才用 @Local。

**Q：@Watch 改 @Monitor 后逻辑不对了**
A：@Watch 是同步执行（变量改 N 次回调执行 N 次），@Monitor 是异步执行（变量改 N 次回调只执行 1 次，在当前事件结束后）。如果 V1 代码依赖"@Watch 同步执行多次"的逻辑（比如在回调里立即读最新值做判断），迁到 V2 后时序变了，必须调整。

**Q：V2 组件报 10905213 "cannot be used with @Link"**
A：V2 组件内用了含 V1 @Link 的系统组件（如 SegmentButton、TextReaderIcon、ProgressButton、SubHeader、ToolBar、Dialog 等）。查 migration-guide.md 第5节映射表：有 V2 替代的必须替换成 V2 版本（禁止回退 @Component）；无 V2 替代的（ComposeTitleBar/Chip/TreeView 等，见白名单）该 struct 保留 @Component，父子组件 V1/V2 可混用。

**Q：块 B 的行为变化能不能预防性修改**
A：不能。块 B 铁律：只适配 grep 命中的变更项。grep 无命中 = 跳过，不允许预防性修改。grep 有命中但二次确认不符合变更描述 = 不改，标注"经确认不受影响"。只有 grep 有命中且二次确认符合才改。

**Q：WidgetCard 要不要迁 V2**
A：不要。WidgetCard（服务卡片）对 @ComponentV2 兼容性不好，保留 V1 装饰器。`widget/`、`widgetcard/` 目录或 module.json5 里 `type: "form"` 的模块在统计和迁移时都要排除。

**Q：纯 V1 工程能不能用双写桥分批迁移**
A：不能。纯 V1 工程（0 个 @ComponentV2）迁移时不要用双写桥或任何保留 V1 的中间态。正确做法是按顺序一步到位全迁：先定义 @ObservedV2 数据类 → 写入端和读取端一起改 → 组件同步迁 V2 → 全部迁完后 V1 AppStorage 彻底清除。

---

## 延伸阅读

- `references/migration-guide.md`：状态管理 V1→V2 迁移指南——20 项装饰器对照表、机制差异（@Watch 同步 vs @Monitor 异步）、5 步迁移步骤、每条规则的代码对比、迁移易错点（@Local 不可外部初始化、V1 系统组件阻塞、运行时陷阱 6 条）、SegmentButton→SegmentButtonV2 完整迁移示例——当需要查阅 V1→V2 具体迁移代码时读取。
- `deprecated-apis.md`：废弃 API 检查与替换——API 没了要换替代时查
- `upgrade-verify.md`：编译验证与错误码——V2 迁移编译报错时查
- `../SKILL.md`：升级总入口——行为变化适配在升级流程中的位置
