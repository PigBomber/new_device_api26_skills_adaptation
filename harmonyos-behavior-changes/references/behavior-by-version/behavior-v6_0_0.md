# 行为变化 — 版本 6.0.0（共 34 条）

> 数据来源：`changelogs-for-all-apps-*.md`
> 「版本隔离=是」表示 targetSdkVersion ≥ 对应版本才生效；「否」表示所有应用都受影响。


## 🔴 无版本隔离（必须适配，23 条）

### 禁止在编译产物为JS的HAR包中使用注解
- **Kit**: ArkTS
- **原因**: 应用开发中，在[release模式下构建](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ide-hvigor-build-har#section19788284410)源码HAR，并同时[开启混淆](https://developer.huawei.com/consumer/cn/doc/harmonyos-guide
- **影响接口**: 不涉及

### 通过字面量定义的数组在删除元素后再使用该字面量定义数组时数组内容异常
- **Kit**: ArkTS
- **原因**: 通过字面量定义的数组在删除元素后再使用该字面量定义数组时数组内容异常。
- **影响接口**: 不涉及

### 文本与输入、信息展示、按钮与选择、滚动与滑动、图形绘制组件接口支持Resource类型
- **Kit**: ArkUI
- **原因**: 基础能力增强，文本与输入、信息展示、按钮与选择、滚动与滑动、图形绘制组件的接口支持Resource类型，可以使用资源对象设置默认选项的值。
- **影响接口**: | 组件名称 | 接口名称 | 参数名称 |
| --- | --- | --- |
| TextPicker | TextPickerOptions | value |
| Progress | CapsuleStyleOptions | content |
| TextPicker | TextPickerOptions | value |
| QRCode | QRCodeInterface | value |
| TextClock | format | value |
| TextTimer | fontWeight | value |
| Badge | BadgeStyle | 

### 使用字面量初始化CustomDialogController类实例导致的编译行为变更
- **Kit**: ArkUI
- **原因**: 在类CustomDialogController中新增接口getState()获取对应弹窗的状态。当原先使用字面量的方式初始化CustomDialogController实例时，会编译报错。字面量的初始化方式是指采用"{}"直接初始化类的实例，例如：
- **影响接口**: 编译行为，不涉及接口

### ArkWeb基于上游社区的Chromium内核从114升级为132版本
- **Kit**: ArkWeb
- **原因**: 为了提升使用ArkWeb内核应用的安全性，开发者使用最新的W3C HTML5特性，以及获得Chromium上游社区最新的性能体验优化成果，故本次进行内核升级（114 -> 132）。
- **影响接口**: ArkWeb

### @ohos.useriam.userAuth限制应用从后台发起带交互界面的身份认证变更
- **Kit**: User Authentication Kit
- **原因**: 需要限制应用从后台发起带交互界面的身份认证。
- **影响接口**: start(): void;

### mincore接口功能补齐至与Linux一致
- **Kit**: 其他
- **原因**: 由于部分应用需要使用mincore做内存相关的性能调优，出参结果影响使用者调优的准确性。
- **影响接口**: mincore

### ptrace syscall操作未被停住的线程由返回0改为返回ESRCH.1
- **Kit**: 其他
- **原因**: 部分外部应用反调试功能预期对未被ptrace停住的线程调用PTRACE\_DETACH会返回ESRCH否则会crash，本次修改鸿蒙返回值使其与Linux保持一致，解决此类应用因反调试功能crash的问题。
- **影响接口**: 内核ptrace系统调用

### 限制ptrace接口仅可在开发者调试模式下使用
- **Kit**: 其他
- **原因**: 为进一步保障安全隐私，将对非开发者模式，且不具备开发者证书情况下使用ptrace进行应用调试的行为进行限制。
- **影响接口**: sys/ptrace.h

### Ability Kit相关公共事件行为变更，增加管控
- **Kit**: Ability Kit
- **原因**: Ability Kit部分公共事件中包含应用信息，需要增加管控措施。
- **影响接口**: 变更的公共事件列表：

### 位置控件功能变更
- **Kit**: ArkUI
- **原因**: 从最新的大数据分析，在需要获取位置信息时，大部分应用使用地图picker或者权限弹窗来申请位置权限，仅有极少数应用使用位置控件，该特性的价值有限，经谨慎评估将该特性下架。
- **影响接口**: @internal/component/ets/location\_button.d.ts中所有接口。

### 通用属性drawModifier接口行为变更
- **Kit**: ArkUI
- **原因**: （1）当drawModifier接口参数从DrawModifier对象变为undefined时，实际生效的仍是原来的DrawModifier对象。开发者无法重置其值，这与通用属性接口的规范不符。
- **影响接口**: drawModifier

### 半模态SIDE侧边样式新增避让软键盘能力
- **Kit**: ArkUI
- **原因**: [bindSheet](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-attributes-sheet-transition#bindsheet)半模态弹窗侧边样式默认支持避让软键盘，提升易用性。
- **影响接口**: -   bindSheet的keyboardAvoidMode属性
    
-   SheetType的SIDE半模态侧边样式

### CanvasRenderer的font接口支持自定义字体行为变更
- **Kit**: ArkUI
- **原因**: 增强基础能力，CanvasRenderer的font接口支持设置自定义字体。
- **影响接口**: CanvasRenderingContext2D和OffscreenCanvasRenderingContext2D的font接口。

### 去除保存控件系统提示弹框变更
- **Kit**: ArkUI
- **原因**: 当前，保存控件支持自定义UI样式。应用选择使用自定义UI，当用户点击保存控件，成功保存媒体库文件时，系统将弹出系统弹框提示用户。在开发过程中，开发者可以调用指定API调整该系统弹框的位置。
- **影响接口**: 删除接口如下：

### zlib.unzipFile和zlib.decompressFile解压文件接口变更
- **Kit**: Basic Services Kit
- **原因**: 解压文件时，针对格式有误的压缩包进行拦截，避免解压之后的文件不符合预期。
- **影响接口**: zlib.unzipFile

### retrieval.VectorQuery接口value字段变更为可选
- **Kit**: Data Augmentation Kit
- **原因**: 支持在检索接口中，自动将query生成向量。
- **影响接口**: retrieval.VectorQuery接口的value字段

### 泰国、沙特阿拉伯、阿富汗和伊朗的默认历法变更
- **Kit**: Localization Kit
- **原因**: 泰国、沙特阿拉伯、阿富汗和伊朗的默认历法配置错误。
- **影响接口**: | 文件 | 接口 |
| --- | --- |
| @ohos.intl.d.ts | [intl.DateTimeFormat.prototype.format](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-intl#formatdeprecated) |
| @ohos.intl.d.ts | [intl.DateTimeFormat.prototype.formatRange](https://developer.huawei.com/consumer/cn/doc/harmony

### libc++ condition\_variable::wait\_for接口变更
- **Kit**: NDK开发
- **原因**: 变更前，libc++库condition\_variable::wait\_for接口使用系统墙上时间，受到修改系统时间的影响，和开发者预期不符合。

### on('dataReceive')接口新增必填参数capabilities
- **Kit**: Share Kit
- **原因**: 若应用在PC/2in1侧接入了沙箱接收，手机到PC/2in1碰一碰会将手机侧分享的所有数据都给到应用，考虑到不同应用能处理的数据类型存在差异。为防止手机侧分享的数据在PC/2in1侧应用无法处理带来的用户体验问题，针对PC/2in1侧沙箱接收的接口做出调整。RecvCapabilityRegistry新增必填参数capabilities，需应用配置支持的数据类型及最大数量。
- **影响接口**: on('dataReceive')/off('dataReceive')接口第二个参数RecvCapabilityRegistry新增必填参数capabilities。

### Car Kit接口新增801、1003810001、1003810002错误码
- **Kit**: Car Kit
- **原因**: 当传入无效入参或在不支持的设备调用接口时，通过相关错误码告知开发者接口调用失败的原因，方便开发者定位问题。
- **影响接口**: Car Kit 提供的公共接口在变更后新增 801、1003810001、1003810002 错误码：

### rag.streamRun接口思考过程输出变更
- **Kit**: Data Augmentation Kit
- **原因**: 思考中输出内容优化。
- **影响接口**: rag.streamRun接口回调函数输出。

### SECURITY\_AUDIT\_NOTIFY\_EVENT\_FILE\_INTERCEPTED、FILE\_INTERCEPTED枚举值变更
- **Kit**: Device Security Kit
- **起始 API Level**: API 20
- **原因**: 为了帮助开发者更直观的识别安全审计事件ID，并能使开发者依据事件ID的规划原则准确识别该事件提供的数据类别，需要将原本规划偏移的文件拦截事件对应ArkTS、C的枚举值进行调整，按事件类别、行为等重新规划。
- **影响接口**: 变更文件：hmscore\_sdk\_c/DeviceSecurityKit/security\_audit.h


## 🟡 版本隔离（targetSdkVersion ≥ 6.0.0 生效，11 条）

### AbilityDelegator.startAbility()接口错误码变更
- **Kit**: -
- **原因**: AbilityDelegator.startAbility()返回的所有错误码与API描述不一致。
- **影响接口**: AbilityDelegator提供的startAbility()接口。

### 借助Want进行文件分享时擦除不合法的URI
- **Kit**: -
- **原因**: 在文件分享场景下（[Want](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-app-ability-want#want)的flags字段中配置了[wantConstant.Flags.FLAG\_AUTH\_READ\_URI\_PERMISSION](https://developer.hua
- **影响接口**: 启动和退出应用的相关接口在文件分享场景下可触发该变更，涉及的接口如下：

### GridRow组件columns参数和GridCol组件span参数默认值变更
- **Kit**: -
- **原因**: 栅格组件默认值原规格容易引发页面压缩问题，变更后提升组件易用性。
- **影响接口**: GridRow组件、GridCol组件。

### width和height支持的matchParent接口规格变更
- **Kit**: -
- **原因**: 接口能力增强，使能Row和Column在设置matchParent时仅适应父组件内容区大小。
- **影响接口**: width(widthValue: Length | LayoutPolicy): T

### UI Input相关NDK接口行为变更
- **Kit**: -
- **原因**: 修复相关接口在如下场景下的返回值异常问题，以确保开发者能够获取正确的结果：
- **影响接口**: int32\_t OH\_ArkUI\_UIInputEvent\_GetType(const ArkUI\_UIInputEvent\* event);

### 蓝牙BLE接口错误码变更
- **Kit**: -
- **原因**: 蓝牙子系统BLE相关接口错误码不够清晰，开发者无法通过错误信息明确需要如何修复问题，影响开发效率。
- **影响接口**: | 模块名 | 命名空间 | 类名 | 接口声明 | 主要变更点说明 |
| --- | --- | --- | --- | --- |
| @ohos.bluetooth.ble.d.ts | ble |  | function startAdvertising(setting: AdvertiseSetting, advData: AdvertiseData, advResponse?: AdvertiseData): void; | 新增2900010错误码：广播资源已耗尽；新增2902054错误码：广播报文数据长度超限； |
| @ohos.bluetooth.ble.d.ts | b

### @ohos.file.fs.d.ts中copy接口在拷贝后，对目标文件原有数据处理方式发生变更
- **Kit**: -
- **原因**: 在目标文件存在且size大于源文件的场景下使用copy接口，拷贝后的目标文件的size会大于源文件且尾部存在冗余内容。
- **影响接口**: @ohos.file.fs.d.ts中的copy接口

### SendPipeRequest和SendPipeRequestWithAshmem传入错误参数时，返回值由USB\_DDK\_SUCCESS变更为USB\_DDK\_INVALID\_PARAMETER
- **Kit**: -
- **原因**: 调用OH\_Usb\_SendPipeRequest和OH\_Usb\_SendPipeRequestWithAshmem接口时，如果入参中的devMmap的bufferlen的长度大于设备的MaxPacketSize，会导致接口执行失败，但是之前未将错误上报，开发者无法感知。
- **影响接口**: drivers/external\_device\_manager: OH\_Usb\_SendPipeRequest、OH\_Usb\_SendPipeRequestWithAshmem。

### ImageInfo对象mimeType返回值变更
- **Kit**: -
- **原因**: 该接口为图片信息查询接口，当前返回值与实际不符。
- **影响接口**: ImageInfo对象mimeType返回值变更。

### 播放器所使用的内存归属变更
- **Kit**: -
- **原因**: 应用创建的播放器实例当前使用的内存统一归属到操作系统侧，导致内存无法根据应用做精细化管理，也容易导致应用滥用播放器行为。本次变更将播放器使用的内存缓存关联到创建播放器的应用，便于应用使用资源的统计。
- **影响接口**: AVPlayer

### TreeSet/TreeMap扩容导致比较器丢失问题正向修复
- **Kit**: -
- **原因**: 使用TreeSet/TreeMap模块的add接口触发扩容时，TreeSet/TreeMap自定义比较器会在扩容后丢失，导致扩容之后进行系统默认排序。
- **影响接口**: TreeSet、TreeMap
