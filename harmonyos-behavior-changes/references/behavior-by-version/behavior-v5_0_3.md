# 行为变化 — 版本 5.0.3（共 17 条）

> 数据来源：`changelogs-for-all-apps-*.md`
> 「版本隔离=是」表示 targetSdkVersion ≥ 对应版本才生效；「否」表示所有应用都受影响。


## 🔴 无版本隔离（必须适配，9 条）

### installSource字段规格变更
- **Kit**: Ability
- **原因**: 规格变更，支持开发者判断应用是否是预置应用。
- **影响接口**: ApplicationInfo中的installSource字段。

### kit.StoreKit.d.ts文件废弃，替换为kit.AppGalleryKit.d.ts文件
- **Kit**: AppGallery Kit
- **原因**: 命名优化
- **影响接口**: 变更前：kit.StoreKit.d.ts文件

### 信号处理方法注册接口sigaction支持SA\_RESETHAND标志位变更
- **Kit**: ArkTS
- **原因**: sigaction是由C库提供的用于注册信号处理方法的接口，开发者可通过调用此接口,指定应用在接收到特定信号时采取的处理方式。
- **影响接口**: musl/signal.h中sigaction接口

### 富文本组件RichEditor的onCopy回调中设置preventDefault()时的行为变更
- **Kit**: ArkUI
- **原因**: 在富文本组件RichEditor的onCopy回调中，当调用preventDefault()时，复制操作完成后，选中区消失。这导致了一个接口同时管理了复制功能和选中区关闭这两个相互独立的行为。
- **影响接口**: 富文本组件RichEditor

### 变更缅甸文，马来文和泰文的显示名称
- **Kit**: Localization Kit
- **原因**: 缅甸文，马来文和泰文显示名称错误。
- **影响接口**: i18n.System.getDisplayLanguage

### hilog日志在商用版本（nolog版本）开发者模式下默认日志级别由info变为warning
- **Kit**: 调试工具
- **原因**: hilog目前商用版本在打开开发者模式时日志级别设置为Info，为加强日志落盘安全性及提高性能，默认日志级别修改Warning。
- **影响接口**: 不涉及接口/组件变更。

### PC/2in1设备上，悬浮窗层级由低于dock栏调整为高于dock栏
- **Kit**: ArkUI
- **原因**: 应用创建的TYPE\_FLOAT类型的悬浮窗，因为层级低于Dock，会被Dock栏遮挡，在视频会议等场景，体验不符合应用预期。
- **影响接口**: @ohos.window.d.ts

### HiAppEvent模块onReceive、OH\_HiAppEvent\_OnReceive、takeNext接口支持应用分身故障日志订阅隔离
- **Kit**: Performance Analysis Kit
- **原因**: 应用分身从设计原则上要求数据隔离，当前应用分身的故障日志和主应用未进行隔离，不易于对分身应用的日志维护。
- **影响接口**: @ohos.hiviewdfx.hiAppEvent.d.ts中的[onReceive](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-hiviewdfx-hiappevent#watcher)、[takeNext](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-hiviewdfx-hiappevent#takenext)接口。

### C API轴事件接口OH\_ArkUI\_UIInputEvent\_GetSourceType和OH\_ArkUI\_UIInputEvent\_GetToolType接口返回值变更
- **Kit**: ArkUI
- **原因**: 通过鼠标滚轮或触控板触发的轴事件无法正确获取到触发源设备类型。
- **影响接口**: OH\_ArkUI\_UIInputEvent\_GetSourceType和OH\_ArkUI\_UIInputEvent\_GetToolType


## 🟡 版本隔离（targetSdkVersion ≥ 5.0.3 生效，8 条）

### FrameNode被UINode包裹时isVisible接口返回值发生变更
- **Kit**: -
- **原因**: 用户使用FrameNode的isVisible接口时，节点会依次向上层父节点查询可见性，如果父节点不可见，子节点也不可见。但如果子节点被UINode类型节点（如IfElse、ForEach、LazyForEach等）包裹，则向上查找过程被阻塞，无法继续查询父节点可见性。
- **影响接口**: FrameNode.d.ts文件isVisible接口。

### OH\_AVCodecOnStreamChanged在音频解码场景的默认行为变更
- **Kit**: -
- **原因**: 音频解码存在采样率等参数发生变化的场景，需要回调通知调用者。
- **影响接口**: native\_avcodec\_audiocodec.h 下的接口OH\_AudioCodec\_RegisterCallback注册的native\_avcodec\_base.h 下的OH\_AVCodecOnStreamChanged。

### 系统录屏应用调用的截屏接口变更
- **Kit**: -
- **原因**: 系统录屏在结束时会调用窗口模块提供的截屏API，生成屏幕缩略图，此时会返回一个截屏事件。多个三方应用通过窗口模块订阅了屏幕截屏事件，系统录屏产生的截屏事件会对三方应用的监听造成干扰。
- **影响接口**: 无变更接口

### supportWindowMode选项配置fullscreen和split时，窗口全屏启动
- **Kit**: -
- **原因**: PC/2in1设备上，在module.json5的[supportWindowMode](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/module-configuration-file#abilities标签)属性配置fullscreen和split时，或在startOptions的[supportWindowMode
- **影响接口**: module.json5的supportWindowMode属性从API version 9开始支持。

### Image、Text和ListItem组件onDragStart接口默认行为变更
- **Kit**: -
- **原因**: Image、Text和ListItem组件存在设置onDragStart接口DragItemInfo返回值中的builder属性后，返回值中pixelMap和extraInfo属性不生效的问题。
- **影响接口**: 涉及组件：Image、Text和ListItem。

### 轴事件支持BEGIN、END及CANCEL类型回调触发
- **Kit**: -
- **原因**: 开发者无法监听到轴事件的BEGIN、END、CANCEL类型的事件回调。
- **影响接口**: OH\_NativeXComponent\_RegisterUIInputEventCallback接口。

### TextController的SetStyledString接口支持保存设置的属性字符串信息到调用的TextController中
- **Kit**: -
- **原因**: 优化属性字符串与Text组件的绑定时机。

### 通话应用在前台时不显示通话胶囊
- **Kit**: -
- **原因**: 通话体验优化。
- **影响接口**: voipCall.reportIncomingCall（上报来电）
