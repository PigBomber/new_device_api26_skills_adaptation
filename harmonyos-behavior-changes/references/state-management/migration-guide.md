# 状态管理 V1 → V2 迁移要点

> 升级鸿蒙项目时，状态管理从 V1 迁移到 V2 的核心参考。
> 详细机制差异见 [v1-v2-difference.md](v1-v2-difference.md)，
> 完整概述见 [overview.md](overview.md)。

## 为什么要迁：V1 的限制

V1 用"代理观察"机制，存在 4 个硬伤：
1. **状态变量不能独立于 UI** — 同一数据被多个视图代理时，一个视图改了不通知其他视图
2. **只能观察第一层** — 嵌套对象的深层属性变化感知不到
3. **冗余更新** — 改对象某个属性会触发整个对象相关 UI 刷新
4. **装饰器配合限制多** — 组件没有明确的输入/输出，不利于组件化

V2 让"数据本身可观察"，解决了以上全部问题。

## 装饰器对照表（V1 → V2）

| 用途 | V1 | V2 | 迁移注意 |
|------|----|----|---------|
| 组件装饰器 | `@Component` | `@ComponentV2` | 一个组件内 V1/V2 装饰器**不能混用**（父子组件可分别用 V1/V2） |
| 组件内状态（纯内部） | `@State` | `@Local` | @Local 不可外部初始化，是组件私有状态 |
| 组件内状态（外部传入一次） | `@State` | `@Param` + `@Once` | 需要从外部初始化一次的情况 |
| 父→子单向 | `@Prop` | `@Param` | 注意：@Prop 深拷贝，@Param 是引用传递（复杂类型行为不同） |
| 父↔子双向 | `@Link` | `@Param` + `@Event` | V2 拆成数据(@Param)和回调(@Event)，父子组件都要改 |
| 跨层级提供 | `@Provide` | `@Provider` | 改名 |
| 跨层级消费 | `@Consume` | `@Consumer` | 改名 |
| 可观察类 | `@Observed` | `@ObservedV2` | @ObservedV2 本身无观测能力，需配 @Trace |
| 类属性追踪 | `@Track` | `@Trace` | V1 @Track 只能一层；V2 @Trace 支持深度观测 |
| 嵌套对象 | `@ObjectLink` | 直接用 `@Param` | V2 不需要 @ObjectLink，@ObservedV2+@Trace 即可深度观测 |
| 变化监听 | `@Watch` | `@Monitor` | **同步→异步**：@Watch 同步执行多次，@Monitor 异步执行一次（⚠️ 最易踩坑） |
| 计算属性 | 无 | `@Computed` | V2 新增 |
| 应用级状态 | `AppStorage` + `@StorageLink/@StorageProp` | `AppStorageV2` + `connect()` | 写法完全不同，需定义 @ObservedV2 类，详见第 3 步 |
| 持久化 | `PersistentStorage` | `PersistenceV2` + `connect()` | 解耦 AppStorage，可独立使用 |
| 页面级状态 | `LocalStorage` + `@LocalStorageLink/@LocalStorageProp` | 全局 `@ObservedV2` + `@Trace` 类 | V2 不再需要 LocalStorage 容器 |
| 双向绑定语法糖 | `$$` | `!!` | V2 建议用 !! |
| 自定义弹窗 | `@CustomDialog` | `openCustomDialog` | V2 建议用 openCustomDialog |
| 组件复用 | `@Reusable` | `@ReusableV2` | |
| 系统环境变量 | `Environment` | 直接调用 Ability 接口 | V2 解耦 AppStorage |

## 机制差异（迁移时最易踩坑的点）

### 1. 观测深度
- **V1**：只观察第一层。`this.objA.propA` 改了能感知，但 `this.objA.objB.propB` 改了**感知不到**
- **V2**：只要被 `@Trace` 装饰，嵌套多深都能感知

### 2. @Watch（V1 同步） vs @Monitor（V2 异步）⚠️ 最易出 bug
```typescript
// V1: @Watch 同步执行，变量改 N 次回调执行 N 次
@State @Watch('onChange') count: number = 0;
onChange() { /* count 每次改都立即执行 */ }

// V2: @Monitor 异步执行，变量改 N 次回调只执行 1 次（在当前事件结束后）
@Local count: number = 0;
@Monitor('count') onChange(mon: IMonitor) { /* 事件结束后执行一次 */ }
```
**迁移影响**：如果 V1 代码依赖"@Watch 同步执行多次"的逻辑（比如在回调里立即读最新值做判断），迁到 V2 后时序变了，必须调整。

### 3. 组件内不能混用 V1/V2 装饰器
一个 `@Component` 内只能用 V1 装饰器（@State/@Prop/@Link...），一个 `@ComponentV2` 内只能用 V2 装饰器（@Local/@Param/@Event...）。**父子组件可以分别用 V1 和 V2**。

### 4. @Local 不可外部初始化（编译报错 10905324）⚠️ 高频踩坑

V1 的 `@State` 既可以内部初始化，也可以从父组件传入初始化。但 V2 的 `@Local` **只允许内部初始化，禁止外部传入**。如果父组件调用时传了该属性，编译报错：

```
ERROR 10905324: The '@Local' property 'xxx' in the custom component cannot be initialized here (forbidden to specify).
```

**判断规则**（决定用 @Local 还是 @Param）：
- 该属性**只在本组件内部读写**，父组件不传 → `@Local`
- 该属性**需要父组件传入初始值** → `@Param`
- 该属性需要父组件传入，且**只同步一次**（后续子组件自己维护）→ `@Param @Once`
- 该属性需要**父子双向同步** → `@Param` + `@Event`

**错误示例：**
```typescript
// ❌ currentIndex 从父组件传入，却用了 @Local
@ComponentV2
struct Child {
  @Local currentIndex: number = 0;  // 报错：父组件传值时 forbidden to specify
}
build() { Child({ currentIndex: this.idx }) }  // 父组件传值触发报错

// ✅ 需要外部传入用 @Param；只同步一次加 @Once
@ComponentV2
struct Child {
  @Param @Once currentIndex: number = 0;
}
```

**迁移要点**：V1 的 `@State` 迁 V2 时，**先检查这个属性是否被父组件传值初始化**。如果被传值，用 `@Param`（或 `@Param @Once`），不能用 `@Local`。只有纯内部状态才用 `@Local`。

### 5. V2 组件不能包含 V1 的 @Link 系统组件（编译报错 10905213）⚠️ 系统组件阻塞迁移

部分官方/系统组件内部使用了 V1 的 `@Link`（双向绑定）。在 `@ComponentV2` 组件里使用这些 V1 系统组件会编译报错：

```
ERROR 10905213: A V2 component cannot be used with any member property decorated by '@Link' in a V1 component.
```

**已知的 V1 高级组件**（内部含 @Link / @State / @Prop 等 V1 装饰器，**在 V2 工程中会导致数据流断裂或编译报错**）。注意 kit 来源不同：

**`@kit.ArkUI`（`@ohos.arkui.advanced.*`）**：

| V1 组件 | V2 替代 | since | 说明 |
|---------|---------|-------|------|
| `SegmentButton` | `TabSegmentButtonV2` / `CapsuleSegmentButtonV2` / `MultiCapsuleSegmentButtonV2` | 18 | 单选/多选见下方示例 |
| `ProgressButton` | `ProgressButtonV2` | 18 | 进度按钮，含 `progress` 状态 |
| `SubHeader` | `SubHeaderV2` | 18 | 分组标题栏，含 `select` 双向状态 |
| `ToolBar` | `ToolBarV2` | 18 | 底部工具栏，含选中态 `ItemState` |
| `AlertDialog`/`TipsDialog` 等 | `AlertDialogV2`/`TipsDialogV2`/`ConfirmDialogV2`/`LoadingDialogV2`/`SelectDialogV2`/`CustomContentDialogV2`/`PopoverDialogV2` | 18 | 来自 `@ohos.arkui.advanced.DialogV2` |

**`@kit.SpeechKit`（AI 朗读组件，注意不是 ArkUI kit）**：

| V1 组件 | V2 替代 | since | 说明 |
|---------|---------|-------|------|
| `TextReaderIcon` | `TextReaderIconV2` | 24 | 朗读听筒图标，华为官方明确：V1 工程用 TextReaderIcon，V2 工程必须用 TextReaderIconV2 |

**铁律（按映射表有无分两种情况）**：
- **映射表有 V2 替代的**（上表 SegmentButton/ProgressButton/SubHeader/ToolBar/Dialog/TextReaderIcon 等）→ **必须替换成 V2 版本，禁止把所在 struct 回退到 @Component**。V1 高级组件在 V2 数据流中会出现：
  - 状态不同步（V1 的 `@State`/`@Link` 不被 V2 的 `@Local`/`@Param` 观测）—— 华为官方原文："使用 V1 装饰器时用 TextReaderIcon，使用 V2 装饰器时**需要**用 TextReaderIconV2"
  - 编译报错 10905213（V2 组件嵌套 V1 @Link 组件）
  - 选中态/进度等交互态丢失
- **映射表没有 V2 替代的**（见下方「无 V2 替代」白名单，如 ComposeTitleBar/Chip/TreeView 等）→ 该 struct **可以保留 @Component**，父子组件 V1/V2 可混用，不影响外层 V2 化

**处理策略**：
1. 全局搜 V1 用法（注意两个 kit 都要查）：
   - `import.*advanced\.(SegmentButton|ProgressButton|SubHeader|ToolBar|Dialog)['"]`（ArkUI）
   - `import.*TextReaderIcon['"]` 和 `import.*\{.*TextReaderIcon['"]`（SpeechKit，注意不要误伤 `TextReaderIconV2`）
2. 命中后查上方映射表：
   - **在映射表里（有 V2 替代）**→ 必须替换。ArkUI 高级组件仍从 `@kit.ArkUI` 导入；TextReaderIcon 从 `@kit.SpeechKit` 导入 `TextReaderIconV2` + `UpReadState`
   - **不在映射表里（查下方白名单确认无 V2 替代）**→ 该 struct 保留 @Component，在迁移记录里标注「无 V2 替代，struct 保留 V1」

> **TextReaderIcon → TextReaderIconV2 关键差异**（来自华为官方文档）：
> - 导入：`import { TextReaderIconV2, UpReadState } from '@kit.SpeechKit'`
> - V2 用 `@Param readState` + `@Event upReadState`（回调函数，类型 `UpReadState = (readState: ReadStateCode) => void`），替代 V1 的内部状态绑定
> - 配合 `TextReader.init()` / `TextReader.start()` 使用（这两个是 kit 的全局 API，V1/V2 通用，不用改）

> **无 V2 替代的 ArkUI 高级组件**（这些在当前 SDK API 26 的 `@ohos.arkui.advanced.*` 下**没有对应 V2 文件**，遇到时**该 struct 保留 @Component 即可**，父子组件 V1/V2 可混用）：
> `ComposeTitleBar`、`EditableTitleBar`、`SelectTitleBar`、`TabTitleBar`、`Filter`、`GridObjectSortComponent`、`TreeView`、`Counter`、`SwipeRefresher`、`SelectionMenu`、`Popup`、`ComposeListItem`、`Chip`/`ChipGroup`、`ArcButton`、`FormMenu`、`DownloadFileButton` 等。
>
> **判断方法**：遇到 10905213 报错，先查上方映射表。在表里 → 必须替换成 V2；不在表里 → 查本白名单，确认无 V2 替代后该 struct 保留 @Component，并在迁移记录里标注「无 V2 替代」。

### SegmentButton → SegmentButtonV2 迁移（API 经 SDK d.ets 核实）

V1 `SegmentButton` 是 `@Component`，内部用 `@Link` 双向绑定 `selectedIndexes`，是 V2 化最常见的阻塞点。V2 系列**从 API 18 起提供**，三种样式：

| V1 用法 | V2 替代 | 选择模式 | 关键 Param |
|---------|---------|---------|-----------|
| `SegmentButton`（tab 或 capsule 单选） | `TabSegmentButtonV2` / `CapsuleSegmentButtonV2` | 单选 | `selectedIndex: number` |
| `SegmentButton`（capsule 多选） | `MultiCapsuleSegmentButtonV2` | 多选 | `selectedIndexes: number[]` |

**V1→V2 关键差异（必读，否则编译/运行出错）**：
1. **数据模型变了**：V1 用 `SegmentButtonOptions`/`SegmentButtonItemOptionsArray`；V2 用 `SegmentButtonV2Items`（一个继承自 `Array<SegmentButtonV2Item>` 的 `@ObservedV2` 类），每项是 `SegmentButtonV2Item`（构造参数为 `SegmentButtonV2ItemOptions`，字段：`text` / `icon` / `symbol` / `enabled`）
2. **双向绑定改成单向 + 回调**：V1 的 `selectedIndexes: this.idx`（@Link）→ V2 改成 **`@Param selectedIndex` 传入 + `@Event $selectedIndex` 回调接收变更**。回调名是字段名加 `$` 前缀（不是 `onSelectedChange`）
3. **`@Require @Param`**：`items` 和 `selectedIndex`/`selectedIndexes` 都是 `@Require @Param`，**必须传**，否则编译报错
4. **导入路径**：V2 全部通过 `@kit.ArkUI` 导入（与 V1 同一个 kit，不用改 import 来源）

**完整迁移示例**（tab 样式单选）：
```typescript
// 导入：与 V1 同一个 kit
import { TabSegmentButtonV2, SegmentButtonV2Items, SegmentButtonV2Item } from '@kit.ArkUI';

// ============ 迁移前：V1 ============
@Component
struct SegDemoV1 {
  @State selectedIndex: number = 0;
  private options: SegmentButtonOptions = SegmentButtonOptions.tab({
    buttons: [{ text: '首页' }, { text: '发现' }, { text: '我的' }],
    multiply: false
  });

  build() {
    // ❌ SegmentButton 是 V1，内部 @Link selectedIndexes，阻塞外层 V2 化
    SegmentButton(this.options)
      .selectedIndexes([this.selectedIndex])  // 双向绑定，V1 风格
      .onSelectedChange((indexes) => { this.selectedIndex = indexes[0]; });
  }
}

// ============ 迁移后：V2 ============
@ComponentV2
struct SegDemoV2 {
  @Local selectedIndex: number = 0;  // 内部状态用 @Local
  // SegmentButtonV2Items 必须用构造函数实例化（它是 @ObservedV2 类，不是字面量）
  @Local items: SegmentButtonV2Items = new SegmentButtonV2Items([
    { text: '首页' }, { text: '发现' }, { text: '我的' }
  ]);

  build() {
    // ✅ 单向 + 回调：selectedIndex 传入，$selectedIndex 接收变更
    TabSegmentButtonV2({
      items: this.items,
      selectedIndex: this.selectedIndex,
      $selectedIndex: (idx: number) => { this.selectedIndex = idx; }
    });
  }
}
```

**capsule 多选**（multi-select）改用 `MultiCapsuleSegmentButtonV2`：
```typescript
@ComponentV2
struct SegMultiV2 {
  @Local selectedIndexes: number[] = [0];
  @Local items: SegmentButtonV2Items = new SegmentButtonV2Items([
    { text: '红' }, { text: '绿' }, { text: '蓝' }
  ]);

  build() {
    // 多选用 selectedIndexes（数组）+ $selectedIndexes 回调
    MultiCapsuleSegmentButtonV2({
      items: this.items,
      selectedIndexes: this.selectedIndexes,
      $selectedIndexes: (idxs: number[]) => { this.selectedIndexes = idxs; }
    });
  }
}
```

**易错点**：
- ❌ 直接传字面量数组 `items: [{ text: 'x' }]` —— 类型是 `SegmentButtonV2Items`，必须 `new SegmentButtonV2Items([...])`
- ❌ 用 `onSelectedChange` 接收回调 —— V2 没有这个属性，回调名是 `$selectedIndex` / `$selectedIndexes`
- ❌ 忘记传 `selectedIndex` —— 它是 `@Require @Param`，缺失直接编译报错
- ✅ 想加 icon：`new SegmentButtonV2Item({ text: '首页', icon: $r('app.media.home') })`
```

## 迁移步骤

### 第 1 步：识别需要迁移的范围

```bash
# 组件级 V1 装饰器（排除 widget/卡片目录——WidgetCard 不迁 V2，见下方说明）
grep -rn "@Component\b\|@State\|@Prop\|@Link\|@Provide\|@Consume\|@Watch\|@Observed\|@ObjectLink" \
  --include="*.ets" --exclude-dir=widget --exclude-dir=widgetcard --exclude-dir=oh_modules --exclude-dir=node_modules \
  <工程路径> | grep -v "@ComponentV2"

# 应用级 V1 状态（@StorageLink/@StorageProp/AppStorage/LocalStorage/PersistentStorage）
grep -rn "@StorageLink\|@StorageProp\|AppStorage\|LocalStorage\|PersistentStorage" \
  --include="*.ets" --exclude-dir=widget --exclude-dir=widgetcard --exclude-dir=oh_modules --exclude-dir=node_modules \
  <工程路径>
```

> **⚠️ 排除 WidgetCard（服务卡片）**：卡片目录（`widget/`、`widgetcard/`，或 module.json5 里 `type: "form"` 的模块）**不迁 V2**——WidgetCard 当前对 @ComponentV2 兼容性不好，保留 V1 装饰器。统计和迁移时必须排除，否则会把卡片的 @Component/@State 算进迁移量导致误迁。

### 第 2 步：确定迁移顺序

**原则：叶子组件优先，根组件在后；应用级状态（AppStorage）最先迁或最后迁都行，但要一次迁完。**

```
应用级状态（AppStorage → AppStorageV2）   ← 建议先迁，因为很多组件依赖它
    ↓
叶子组件（不引用其他自定义组件的组件）      ← 先迁
    ↓
中间组件                                       ← 再迁
    ↓
根组件 / 页面入口（@Entry）                   ← 最后迁
```

> 关键规则：**同一个组件内 V1/V2 装饰器不能混用**。但父子组件可以分别用 V1 和 V2——所以可以从叶子组件开始逐个迁，不需要一次性全改。

### 第 3 步：应用级状态迁移（@StorageLink / AppStorage）

这是最常被忽略但改动量最大的一块。

**V1 写法：**
```typescript
// 写入端（通常在 EntryAbility 或工具类）
AppStorage.setOrCreate('isHover', false);
AppStorage.setOrCreate('pageID', 0);

// 读取端（各组件）
@Component
struct MyView {
  @StorageLink('isHover') isHover: boolean = false;   // 双向同步
  @StorageProp('pageID') pageID: number = 0;          // 只读单向
}
```

**V2 写法：**
```typescript
// 1. 先定义可观察的数据类
@ObservedV2
class AppState {
  @Trace isHover: boolean = false;
  @Trace pageID: number = 0;
}

// 2. 写入端：用 AppStorageV2.connect
AppStorageV2.connect(AppState, 'appState', () => new AppState())!;

// 3. 读取端：组件内用 @Local + connect
@ComponentV2
struct MyView {
  @Local appState: AppState = AppStorageV2.connect(AppState, 'appState', () => new AppState())!;
  // 双向同步：直接改 this.appState.isHover 即可
  // 只读：不调用 connect，通过其他方式获取
}
```

**LocalStorage（页面级）→ V2：** 用全局 `@ObservedV2` + `@Trace` 类替代，不再需要 LocalStorage 容器。

**PersistentStorage（持久化）→ PersistenceV2：**
```typescript
// V1
PersistentStorage.persistProp('theme', 'light');
@StorageLink('theme') theme: string = 'light';

// V2
@ObservedV2
class ThemeState { @Trace theme: string = 'light'; }
PersistenceV2.connect(ThemeState, 'themeState', () => new ThemeState())!;
```

**迁移决策：无公共模块的跨模块缓存**

工程没有 lib_common 公共模块时，跨模块共享的全局缓存（如 screenWidth/screenHeight）迁移到 V2 需要：
- 要么在每个使用模块各自建 model（重复定义）
- 要么新建一个公共模块（影响工程结构）
- 要么放 entry 模块（其他模块不一定依赖 entry）

**实战决策**：工程无公共模块时，新建一个公共 model 模块或放在 utils 里存放 @ObservedV2 数据类。**必须迁移，不能保留 V1 AppStorage**。

**纯 V1 工程的正确迁移策略（一步到位，不留 V1 中间态）**：

纯 V1 工程（0 个 @ComponentV2）迁移时，**不要用双写桥或任何保留 V1 的中间态**。正确做法是按顺序一步到位全迁：

1. **先定义 @ObservedV2 数据类**（如 AppGlobalState），集中管理所有全局状态字段
2. **写入端和读取端一起改**：AppStorage.setOrCreate → AppStorageV2.connect；@StorageLink/@StorageProp → @Local + connect
3. **组件同步迁 V2**：@Component → @ComponentV2，@State → @Local/@Param
4. **全部迁完后 V1 AppStorage 彻底清除**，不留任何 V1 残留

> ⚠️ **禁止用双写桥**（同时写 V1 AppStorage 和 V2 AppStorageV2）。这是分批迁移的妥协方案，但既然目标是全部迁完 V2，双写桥只会增加复杂度——迁完后还要记得删 V1 那行，容易遗漏。一步到位更干净。

### 第 4 步：组件级装饰器迁移（逐条配代码对比）

#### 4.1 组件装饰器
```typescript
// V1
@Component
struct MyView { }

// V2
@ComponentV2
struct MyView { }
```

#### 4.2 @State → @Local 或 @Param @Once
```typescript
// V1：@State 可外部初始化，也可内部
@Component
struct MyView {
  @State count: number = 0;
}

// V2：看是否需要外部传入
// 情况A：纯内部状态 → @Local（不可外部初始化）
@ComponentV2
struct MyView {
  @Local count: number = 0;
}
// 情况B：需要外部传入一次 → @Param @Once
@ComponentV2
struct MyView {
  @Param @Once count: number = 0;
}
```

#### 4.3 @Prop → @Param（父→子单向）
```typescript
// V1：@Prop 深拷贝
@Component
struct Child {
  @Prop title: string = '';
}

// V2：@Param 引用传递（注意：复杂类型不再深拷贝）
@ComponentV2
struct Child {
  @Param title: string = '';
}
```

#### 4.4 @Link → @Param + @Event（父↔子双向）⚠️ 跨文件改动
双向绑定在 V2 拆成"数据下行(@Param) + 事件上行(@Event)"两部分，父子组件都要改。

```typescript
// V1：@Link 自动双向同步
// 子组件
@Component
struct Child {
  @Link count: number;   // 改这里会同步回父组件
}
// 父组件
@State parentCount: number = 0;
Child({ count: this.parentCount })

// V2：@Param + @Event 手动实现双向
// 子组件
@ComponentV2
struct Child {
  @Param count: number = 0;
  @Event onCountChange: (n: number) => void = (n: number) => {};  // 上行事件
  build() {
    Button('inc').onClick(() => this.onCountChange(this.count + 1))
  }
}
// 父组件
@Local parentCount: number = 0;
Child({
  count: this.parentCount,
  onCountChange: (n: number) => { this.parentCount = n; }  // 接收上行
})
```

#### 4.5 @Observed + @ObjectLink → @ObservedV2 + @Trace
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

#### 4.6 @Watch → @Monitor ⚠️ 时序变化
```typescript
// V1：@Watch 同步执行，变量改 N 次回调执行 N 次
@Component
struct MyView {
  @State @Watch('onChange') count: number = 0;
  onChange() {
    // count 每次改变都立即同步执行
    console.info(`count = ${this.count}`);
  }
}

// V2：@Monitor 异步执行，变量改 N 次回调只执行 1 次（取最终值）
@ComponentV2
struct MyView {
  @Local count: number = 0;
  @Monitor('count')
  onChange(mon: IMonitor) {
    // 当前事件结束后才异步执行一次
    console.info(`count = ${this.count}`);
    // mon.before 可以拿到变化前的值
    mon.dirty.forEach(path => console.info(`changed: ${path}`));
  }
}
```
**必须检查**：如果 V1 的 @Watch 回调里依赖"立即同步读到最新值"或"每次改变都触发"，迁到 V2 后行为变了，需调整逻辑。

#### 4.7 @Provide / @Consume → @Provider / @Consumer
```typescript
// V1
@Component
struct Parent {
  @Provide('theme') theme: string = 'light';
}
@Component
struct Descendant {
  @Consume('theme') theme: string = '';   // 跨层级双向同步
}

// V2（仅装饰器改名，用法基本一致）
@ComponentV2
struct Parent {
  @Provider('theme') theme: string = 'light';
}
@ComponentV2
struct Descendant {
  @Consumer('theme') theme: string = '';
}
```

### 第 5 步：编译验证 + 重点复查

```bash
hvigorw assembleHap --mode module -p module=entry@default -p product=default 2>&1 | grep -E "ERROR|error"
```

**重点复查清单：**
- [ ] 所有 `@Watch` 改 `@Monitor` 的地方，确认时序变化不影响业务逻辑
- [ ] `@Link` 改 `@Param + @Event` 的地方，确认父子组件双向同步正确
- [ ] `@StorageLink` 改 `AppStorageV2.connect` 的地方，确认写入端和读取端都用 V2
- [ ] `@Prop` 改 `@Param` 的地方，注意复杂类型从深拷贝变引用传递
- [ ] `@Observed` 类加了 `@Trace` 的属性是否完整（漏加会导致不触发 UI 更新）
- [ ] **每个 `@State` 改 `@Local` 前，先检查该属性是否被父组件传值**——被传值的必须用 `@Param`（否则报错 10905324）
- [ ] **目标 V2 组件内是否用了含 @Link 的 V1 系统组件**（如 SegmentButton、TextReaderIcon、ProgressButton、SubHeader、ToolBar）——查第 5 节映射表：**有 V2 替代的必须替换（禁止回退 @Component）**；**无 V2 替代的（见白名单）该 struct 保留 @Component**（否则报错 10905213）

> V1/V2 同一组件内混用会编译报错，按错误提示逐个修正即可。父子组件分别用 V1/V2 是允许的。

## 迁移决策

| 场景 | 建议 |
|------|------|
| 新开发的应用 | 直接用 V2 |
| 老应用升级到 API 26 | **必须迁移**。V1→V2 是升级流程的必要环节，不留 V1 残留 |
| 老应用暂不升级 | 不涉及迁移（未升级就不需要迁 V2） |
| **WidgetCard（服务卡片）** | **不迁移 V2**。WidgetCard 当前对 @ComponentV2 兼容性不好，卡片代码保留 @Component/@State 等 V1 装饰器 |

> V1 存在深度观测缺失、冗余更新、装饰器配合限制等问题。**升级时必须迁移 V2**——这是升级流程的必要环节，不留 V1 残留，不保留双写桥等中间态。
>
> **例外：WidgetCard（服务卡片）不迁 V2**。卡片代码（通常在 `*/widget/` 或 `*/widgetcard/` 目录，module.json5 里 `type: "form"`）当前对 @ComponentV2 兼容性不好，**保留 V1 装饰器**，不要改成 @ComponentV2。统计 V1 装饰器数量和迁移时要排除卡片目录，避免误迁。

## 参考文档

## 缺失文档（需手动补充）

以下关键迁移场景文档未下载，如需详细 step-by-step 迁移示例请查华为官方：
- [V1向V2迁移场景](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-state-management-v1-v2-migration-guide)
- [V1和V2混用场景](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/v1v2-mixing)
