# 行为变化 — 版本 26.0.0（共 21 条）

> 数据来源：`changelogs-for-all-apps-*.md`
> 「版本隔离=是」表示 targetSdkVersion ≥ 对应版本才生效；「否」表示所有应用都受影响。


## 🔴 无版本隔离（必须适配，10 条）

### JSVM基于上游社区的Chromium/v8内核从132升级为144版本
- **Kit**: ArkTS
- **起始 API Level**: 不涉及
- **原因**: 为了提升使用JSVM应用的安全性，为开发者提供最新的W3C HTML5特性，Chromium上游社区从133到144版本共发布300+项特性，带来了更多W3C特性、安全、性能、稳定性的提升，故本次进行内核升级 (132 -> 144)。
- **影响接口**: ArkTS/JSVM

### async函数类型判定修复
- **Kit**: ArkTS
- **起始 API Level**: API 8
- **原因**: 在类中getter/setter之后定义的async函数，实际被初始化的类型为普通函数，导致无法在运行时正确地判定函数类型。
- **影响接口**: 1.  new util.types().isAsyncFunction
2.  Function.constructor.name

### 修复ConvertXML的fastConvertToJSObject接口解析时丢失同级text节点的问题
- **Kit**: ArkTS
- **起始 API Level**: API 14
- **原因**: ConvertXML的[fastConvertToJSObject](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-convertxml#fastconverttojsobject14)接口在解析XML时，若XML内容包含子元素、与子元素同级的文本节点，会丢失与子元素同级的文本节点。此变更修复了该问题。
- **影响接口**: fastConvertToJSObject(xml: string, options?: ConvertOptions) : Object;

### 鼠标事件rawDeltaX和rawDeltaY的返回值变更
- **Kit**: ArkUI
- **原因**: 鼠标事件rawDeltaX和rawDeltaY的返回值的含义为鼠标设备在二维平面的物理移动偏移量，其数值为鼠标硬件的原始移动数据，使用物理世界中鼠标移动的距离单位进行表示，上报由鼠标硬件本身决定。当前实现返回值并非是原始移动数据，而是原始移动数据缩小了X倍，X为系统的显示大小比例。因此需要变更返回值使其符合本身的含义。
- **影响接口**: [rawDeltaX](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-mouse-key#mouseevent对象说明)、[rawDeltaY](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-mouse-key#mouseevent对象说明)、[OH\_ArkUI\_MouseEvent\_GetRawDeltaX](https://developer.huawei.com

### ArkUI接口新增仅支持Stage模型的约束
- **Kit**: ArkUI
- **起始 API Level**: API 10
- **原因**: ArkUI存在API接口未明确标注所支持的模型标签，导致DevEco Studio和开发者无法准确感知FA模型和Stage模型的支持情况。
- **影响接口**: ArkUI API version 10及以上的全量接口。

### 主页NavDestination中使用queryNavDestinationInfo接口和onResult接口的行为变更
- **Kit**: ArkUI
- **起始 API Level**: API 20
- **原因**: 1.  通过queryNavDestinationInfo(isInner: Optional<boolean>)接口无法获取到主页NavDestination的信息，不符合接口设计规格。
- **影响接口**: 1.  [queryNavDestinationInfo(isInner: Optional<boolean>)](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-custom-component-api#querynavdestinationinfo18)接口。
2.  NavDestination的[onResult](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-basic-components-navdes

### @ReusableV2组件复用的reuse属性支持动态复用标识
- **Kit**: ArkUI
- **起始 API Level**: API 18
- **原因**: 当前[@ReusableV2](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-new-reusablev2)装饰器装饰的自定义组件的reuse属性不支持使用动态的reuseId，变更后可增强V2组件复用能力，支持动态复用标识。
- **影响接口**: 涉及接口：[ReuseOptions](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-attributes-reuse#reuseoptions)里的reuseId参数。

### ArkWeb基于上游社区的Chromium内核从132升级为144版本
- **Kit**: ArkWeb
- **起始 API Level**: 不涉及
- **原因**: 为了提升使用ArkWeb内核应用的安全性，开发者使用最新的W3C HTML5特性，以及获得Chromium上游社区最新的性能体验优化成果，故本次进行内核升级(132 -> 144)。
- **影响接口**: ArkWeb

### ohos.permission.READ\_IMAGEVIDEO权限变更
- **Kit**: Media Library Kit
- **起始 API Level**: API 9
- **原因**: 限制网盘等应用访问云上的图片或视频，避免因为访问云上资产导致隐私安全与设备功耗高、发热等问题。
- **影响接口**: -   PhotoAccessHelper.getAssets(options: FetchOptions, callback: AsyncCallback<FetchResult<PhotoAsset>>): void;
    
-   PhotoAccessHelper.getAssets(options: FetchOptions): Promise<FetchResult<PhotoAsset>>;
    
-   PhotoAccessHelper.getBurstAssets(burstKey: string, options: FetchOptions): Promise<F

### 权限管控
- **Kit**: Network Kit


## 🟡 版本隔离（targetSdkVersion ≥ 26.0.0 生效，11 条）

### 部分公共事件行为变更，增加管控
- **Kit**: -
- **起始 API Level**: API 9
- **原因**: Ability Kit部分公共事件中包含应用信息，存在信息泄露的安全风险，需要增加管控。
- **影响接口**: 变更的公共事件列表：

### JSVM支持Wasm解释器，jitless默认行为发生变更
- **Kit**: -
- **起始 API Level**: API 12
- **原因**: 需要支持jitless条件下执行Wasm能力，支撑坚盾守护模式下特效/直播等业务场景。
- **影响接口**: ArkTS/JSVM

### NodeAdapter的onAttachToNode回调触发时机变更
- **Kit**: -
- **起始 API Level**: API 12
- **原因**: NodeAdapter绑定的宿主节点在主树挂载时，ArkTS API会触发onAttachToNode回调，C API会触发OH\_ArkUI\_NodeAdapter\_RegisterEventReceiver绑定的、事件类型为NODE\_APPEAR\_EVENT\_WILL\_ATTACH\_TO\_NODE的回调，与文档描述的在绑定时触发回调的时机不符。
- **影响接口**: [onAttachToNode](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-arkui-framenode#onattachtonode12)

### 属性字符串段落首个占位为CustomSpan或ImageAttachment时，支持设置段落样式
- **Kit**: -
- **起始 API Level**: API 12
- **原因**: 历史版本中，[CustomSpan](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-styled-string#customspan)或[ImageAttachment](https://developer.huawei.com/consumer/cn/doc/harmonyos-ref
- **影响接口**: [ParagraphStyle](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-styled-string#paragraphstyle)

### LayoutPolicy.matchParent父组件为Row、Column、Flex组件时，单方向设置matchParent的子组件布局行为变更
- **Kit**: -
- **起始 API Level**: API 15
- **原因**: 当前Row、Column、Flex组件布局过程中，单方向设置matchParent的子组件不会参与父组件的尺寸计算。变更后，Row、Column、Flex组件会自适应单方向matchParent子组件尺寸，布局效果更符合API语义。
- **影响接口**: 涉及组件：[Row](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-row)、[Column](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-column)、[Flex](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-flex)。

### EmbeddedComponent获焦能力变更
- **Kit**: -
- **起始 API Level**: API 18
- **原因**: [EmbeddedComponent](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-embedded-component)获焦后，其拉起的EmbeddedUIExtensionAbility窗口内焦点不会停留在容器组件，而是下发到容器内第一个可获焦子节点。当获焦节点为[TextInpu
- **影响接口**: [EmbeddedComponent](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-embedded-component)

### WithTheme相关组件行为变更
- **Kit**: -
- **原因**: 为提升主题功能的一致性与完整性，对下述场景下的组件行为进行优化适配。
- **影响接口**: WithThemeInterface、setDefaultTheme。

### NODE\_SWIPER\_EVENT\_ON\_CONTENT\_DID\_SCROLL事件回调的返回值行为变更
- **Kit**: -
- **原因**: [NODE\_SWIPER\_EVENT\_ON\_CONTENT\_DID\_SCROLL](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/capi-native-node-h#arkui_nodeeventtype)事件回调返回值中的主轴方向上页面的长度ArkUI\_NodeComponentEvent.da
- **影响接口**: [NODE\_SWIPER\_EVENT\_ON\_CONTENT\_DID\_SCROLL](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/capi-native-node-h#arkui_nodeeventtype)的返回参数值ArkUI\_NodeComponentEvent.data\[3\].f32。

### 组件的阴影模糊半径规格变更
- **Kit**: -
- **起始 API Level**: API 7
- **原因**: 修改阴影模糊半径生效范围，使shadow属性中的所有阴影类型新增支持模糊半径为0时的阴影能力。
- **影响接口**: 通用属性[shadow](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-attributes-image-effect#shadow)的[ShadowOptions.radius](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-attributes-image-effect#shadowoptions对象说明)参数。

### getUidRxBytes、getUidTxBytes接口权限变更
- **Kit**: -
- **起始 API Level**: API 10
- **原因**: getUidRxBytes、getUidTxBytes接口需要添加权限管控。
- **影响接口**: statistics.getUidRxBytes(uid: number, callback: AsyncCallback<number>): void

### 权限策略变更说明
- **Kit**: -
- **起始 API Level**: API 14
- **原因**: 当前PC/2in1设备上已支持为binary二进制文件赋予权限。针对kernelpermission，即权限名中带有“.kernel”的权限，如ohos.permission.kernel.ALLOW\_WRITABLE\_CODE\_MEMORY，从ROM版本7.0开始，binary（以下简称bin）的进程若想要使用kernelpermission的权限，需要应用主动为binary声明权限并签名
- **影响接口**: 不涉及
