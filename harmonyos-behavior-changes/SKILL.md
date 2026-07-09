---
name: harmonyos-behavior-changes
description: >
  HarmonyOS 项目升级中的行为变化适配 + 状态管理 V1→V2 迁移环节。当用户说「升级后哪些接口行为变了」「升到 26 后 Toast 样式不对」「触摸热区变了」「组件默认效果变了」「targetSdkVersion 升级影响」「版本隔离是什么意思」「状态管理 V1 迁 V2」「@Component 改 @ComponentV2」「@State 改 @Local」「@Watch 改 @Monitor」时触发。
  块 B 基于官方 changelog 的 25 项变更，用 grep 扫项目代码，命中的才适配——不允许没命中就预防性修改。
  属于升级流程的步骤4状态管理 V1→V2 + 步骤5行为变化环节，通常由 harmonyos-upgrade 总 skill 路由调用。
version: 2.0.0
---

# 行为变化适配 + 状态管理迁移

## 两块内容

### 块 A：状态管理 V1→V2 迁移（核心）

升级鸿蒙项目时，把状态管理从 V1 迁到 V2。**升级时必须迁移**——这是升级流程的必要环节，不是可选项。

**入口文件**：[references/state-management/migration-guide.md](references/state-management/migration-guide.md)

包含：
- V1→V2 装饰器对照表（20 项）
- 机制差异（观测深度、@Watch 同步 vs @Monitor 异步，最易踩坑）
- 5 步迁移步骤（应用级状态→数据类→跨层级→组件级→监听时序）
- 每条规则的代码对比（改前/改后）
- 迁移易错点（@Local 不可外部初始化、V2 组件不能含 V1 @Link 系统组件、字段初始化崩溃等）

### 块 B：接口行为变化（基于官方 changelog，grep 命中才适配）

升级到 API 26 后，官方 changelog 列出了 25 项行为变化（来源：`changelogs-for-all-apps-7001` + `changelogs-ux-7001`）。

**铁律：只适配项目代码命中的变更项。不允许没命中就预防性修改。** 用下方表格的「grep 关键词」扫项目代码，命中的才按「适配方法」处理。没命中的跳过。

#### 如何判定命中（两步门控，缺一不可）

**只允许在 grep 命中且二次确认符合变更描述时修改代码。** 不允许凭"可能受影响"就动代码。

**第 1 步：grep 扫描（证明项目用了这个 API/组件）**

```bash
cd <工程根目录>
# 示例：查项目是否用了 rawDeltaX（鼠标事件行为变化）
grep -rn "rawDeltaX\|rawDeltaY" --include="*.ets" --exclude-dir=oh_modules --exclude-dir=node_modules .
# 有输出 = 进入第 2 步；无输出 = 跳过该项，不改任何代码
```

每项变更的「grep 关键词」是官方 changelog 里「变更的接口/组件」字段列出的 API/组件名。

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

#### 无版本隔离（targetSdk < 26 也生效，必须适配）

| grep 关键词 | 变更描述 | 命中后的适配方法 |
|------------|---------|----------------|
| `ArkWeb` + `Web(` | ArkWeb 内核 132→144，100+ W3C 规格变更 | 查 [ArkWeb 132→144 差异总结](https://gitcode.com/openharmony-tpc/chromium_src/blob/master/web/ReleaseNote/ArkWeb_132_144.md)，逐项核对 Web 页面 |
| `JSVM` `import.*jsvm` | JSVM 内核 132→144，38 项需注意的变更（W3C 特性/安全/性能） | 查 [JSVM 132→144 差异总结](https://gitcode.com/openharmony/arkcompiler_jsvm/blob/master/ReleaseNote/JSVM_132_144.md)，逐项核对 |
| `isAsyncFunction` | getter/setter 后的 async 函数，`util.types().isAsyncFunction` 返回值从 false→true，`constructor.name` 从 Function→AsyncFunction | 排查依赖 async 类型判断的代码逻辑 |
| `fastConvertToJSObject` | XML 解析不再丢失与子元素同级的 text 节点 | 检查 XML 解析结果是否依赖旧的丢失行为 |
| `rawDeltaX` `rawDeltaY` | 鼠标事件返回值从"原始数据/缩放比例"变为真实硬件数据 | 如需旧行为用 `this.getUIContext().px2vp()` 转换 |
| `READ_IMAGEVIDEO` | 该权限只能读本地图片/视频，不含云端 | 如需云端资源改用 `PhotoViewPicker` |
| `queryNavDestinationInfo` `onResult` | 主页 NavDestination 的 queryNavDestinationInfo 现在能获取信息了；onResult 不再误触发 | 审视主页 NavDestination 的 onResult 逻辑是否依赖旧行为 |
| `reuse(` `@ReusableV2` | reuseId 支持动态回调，复用标识从组件名变为回调返回值 | 检查 V2 复用组件的 reuseId，确认复用行为符合预期 |
| `defaultFocus` `requestFocus` | FA 模型的 ArkUI API（version 10+）编译报错 | 确认工程已是 Stage 模型（detect 步骤会检查） |

#### 版本隔离（仅 targetSdkVersion ≥ 26.0.0 生效，命中后需测试确认）

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

#### UX 变更（来自 changelogs-ux-7001，targetSdk ≥ 26 生效）

| grep 关键词 | 变更描述 | 命中后的适配方法 |
|------------|---------|----------------|
| `Button(` `Toggle(` `Select(` `Chip(` `ChipGroup(` | 表单类组件触摸热区最小高度 28→32vp | 确认布局不被 4vp 增幅撑破 |
| `showToast` `showAlertDialog` `showActionSheet` `showDialog` `openCustomDialog` `bindPopup` `bindSheet` | Dialog/Toast/AlphabetIndexer/文本选择菜单默认开启沉浸式系统材质 | 确认弹窗视觉效果可接受 |
| `Text(` `TextInput(` `TextArea(` `Search(` 等（大量内置文本组件） | 内置文本样式优化：孤字换行/小语种行高/音节换行 | 确认文本布局正常（此项命中面广，重点测试） |
| （无特定 grep 关键词，凡含文本即涉及） | notofonts 三方件小语种字体升级，影响小语种（阿拉伯语/泰语等）文本渲染 | 如应用支持小语种，重点测试文本渲染效果 |

> 以上 25 项变更来自华为官方 `changelogs-for-all-apps-7001` 和 `changelogs-ux-7001`。完整变更说明（含变更前/后对比、适配代码示例）在 `/doc/roadmap/changelogs-for-all-apps-7001.md` 和 `/doc/roadmap/changelogs-ux-7001.md`。

## 如何判断用户要哪块

| 用户问的 | 读哪个 |
|---------|--------|
| "@State 怎么改" / "@Watch 改 @Monitor" / "@Component 改 V2" | 块 A：读 migration-guide.md |
| "升级后行为变了啥" / "Toast 样式不对" | 块 B：用上方表格的 grep 关键词扫项目，命中的才适配 |
| 不确定 | 块 B 逐项 grep 扫一遍；状态管理看 migration-guide.md |

## 与 harmonyos-deprecated-apis skill 的关系

| | 本 skill | harmonyos-deprecated-apis |
|---|---|---|
| 管什么 | API **还在**，但行为变了；V1 装饰器迁 V2 | API **废弃了**，要换替代 |
| 怎么发现 | 查行为变化速查表；V1 装饰器 grep 统计 | 编译 deprecated 告警 |

## See Also

- [references/state-management/migration-guide.md](references/state-management/migration-guide.md) — V1→V2 迁移指南（20项对照+5步流程+踩坑记录）
