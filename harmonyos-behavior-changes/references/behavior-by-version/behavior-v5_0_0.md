# 行为变化 — 版本 5.0.0（共 114 条）

> 数据来源：`changelogs-for-all-apps-*.md`
> 「版本隔离=是」表示 targetSdkVersion ≥ 对应版本才生效；「否」表示所有应用都受影响。


## 🔴 无版本隔离（必须适配，96 条）

### 包管理bundleManager/AbilityInfo中新增必选属性orientationId
- **Kit**: Ability
- **原因**: 按照终端设备用户的使用习惯，应用应当能够根据设备类型配置默认的窗口旋转方式，应用在orientation配置资源引用，使用时根据资源ID orientationId解析具体配置。

### 延迟加载（lazy import）影响异步任务执行时序变更为不影响异步任务执行时序
- **Kit**: ArkTS
- **原因**: 延迟加载（lazy import）特性在测试过程中发现问题，使用lazy import的变量，会改变异步任务的运行时序。
- **影响接口**: 不涉及

### 执行幂运算（\*\*）当底数是1，指数是NaN或ToNumber之后是NaN的情况的返回值变更
- **Kit**: ArkTS
- **原因**: ArkTS执行幂运算时，当底数为1，指数为NaN或ToNumber之后是NaN，返回值为1，与ECMAScript® 2021 Language Specification描述不符。
- **影响接口**: 不涉及

### String.prototype.lastIndexOf接口查找空字符串行为变更
- **Kit**: ArkTS
- **原因**: String.prototype.lastIndexOf接口查找空字符串时返回值为-1，与ECMAScript® 2021 Language Specification描述不符。
- **影响接口**: String.prototype.lastIndexOf

### 轴事件分发机制变更
- **Kit**: ArkUI
- **原因**: 轴事件分发错误，开发者如果改了组件z序，组件显示、隐藏后不能正确分发到挂载轴事件的XComponent组件上。
- **影响接口**: [OH\_NativeXComponent\_RegisterUIInputEventCallback](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/capi-native-interface-xcomponent-h#oh_nativexcomponent_registeruiinputeventcallback)接口。

### @ohos.arkui.uiExtension中uiExtension命名空间下新增properties必选属性
- **Kit**: ArkUI
- **原因**: [EmbeddedComponent](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-embedded-component)所在的应用窗口移动的场景下，无法获取该组件的大小和位置信息，不满足开发者业务诉求。
- **影响接口**: WindowProxy的properties属性

### Navigation的menus接口、NavDestination的title和menus接口支持Resource类型资源
- **Kit**: ArkUI
- **原因**: 基础能力增强，Navigationd的menus接口、NavDestination的title和menus接口支持Resource类型资源。
- **影响接口**: Navigation的menus接口、NavDestination的title和menus接口

### setWindowLayoutFullScreen、setImmersiveModeEnabledState接口在PC/2in1设备的自由多窗模式上禁用
- **Kit**: ArkUI
- **原因**: 因为phone设备上的沉浸式是应用布局全屏且窗口与系统状态栏与导航条交叠，而PC/2in1设备上的沉浸式是应用布局全屏且隐藏系统状态栏和Dock栏，行为与phone设备不一致。所以在PC/2in1设备的自由多窗模式上禁用setWindowLayoutFullScreen、setImmersiveModeEnabledState接口，只能调用maximize接口设置进入/退出沉浸式，在进入最大化时通
- **影响接口**: @ohos.window.d.ts

### setWindowBrightness在PC/2in1设备的行为变更
- **Kit**: ArkUI
- **原因**: PC/2in1设备下，在视频播放页面，通过快捷键调节屏幕亮度不生效，原因是快捷键调节系统亮度，而在视频播放页面屏幕亮度跟随窗口亮度值。
- **影响接口**: @ohos.window.d.ts

### setAppAccess错误码变更
- **Kit**: Basic Service Kit
- **原因**: 为防止资源浪费，禁止应用无限制地通过调用该接口授予三方应用权限，当授权应用数量超过1024个时，返回新增的错误码（12400005）。
- **影响接口**: @ohos.account.appAccount.d.ts中如下接口：

### kit.CallKit.d.ts文件废弃，替换为kit.CallServiceKit.d.ts文件访问级别
- **Kit**: Call Service Kit
- **原因**: 命名优化
- **影响接口**: 变更前：kit.CallKit.d.ts文件

### 持久化权限激活接口实现从sandbox\_manager模块切换到UPMS模块
- **Kit**: Core File Kit
- **原因**: sandbox\_manager权限清理存在时序问题，需要在临时授权中添加时间戳，所有的临时授权调节到UPMS统一添加时间戳处理。
- **影响接口**: @ohos.fileshare.d.ts的ArkTS API：ohos.fileshare.activatePermission。

### @hms.ai.vision.objectDetection.d.ts和@hms.ai.vision.skeletonDetection.d.ts方法文件变更
- **Kit**: Core Vision Kit
- **原因**: 错误信息表述有误。
- **影响接口**: -   static create(): Promise<ObjectDetector>
-   process(request: visionBase.Request): Promise<ObjectDetectionResponse>
-   static create(): Promise<SkeletonDetector>
-   process(request: visionBase.Request): Promise<SkeletonDetectionResponse>

### FormLink的router事件允许拉起Ability类型范围变更
- **Kit**: Form Kit
- **原因**: FormLink的router事件当前未对被拉起的Ability类型进行校验，但实际此事件应只允许拉起UIAbility，针对使用router事件拉起非UIAbility的场景，需要做安全加固。
- **影响接口**: FormLink

### image.ImageSource.DecodingOptionsForPicture接口的desiredAuxiliaryPictures属性系统能力变更
- **Kit**: Image Kit
- **原因**: 接口DecodingOptionsForPicture与属性desiredAuxiliaryPictures归属的系统能力不一致，会影响对接口支持系统能力情况的判断，需要将desiredAuxiliaryPictures的SystemCapability中的SystemCapability.Multimedia.Image.Core改为SystemCapability.Multimedia.Ima
- **影响接口**: @ohos.multimedia.image中涉及修改的属性如下：

### hdc file recv命令不支持操作媒体库目录
- **Kit**: 调试工具
- **原因**: 由于业务演进方向不同，媒体和文档目录需要不同的权限策略，变更后禁止通过hdc file recv命令将媒体库目录内文件从远端设备接收至本地。
- **影响接口**: hdc命令行工具

### hdc的file recv命令及shell读取权限变更
- **Kit**: 调试工具
- **原因**: 为了更好的保护终端用户的隐私安全，加强hdc/shell对系统目录文件的权限管控。
- **影响接口**: hdc命令行工具

### hidumper组件内存输出显示每列后新增一个空格
- **Kit**: 调试工具
- **原因**: 解决生态调优测试的显示问题。
- **影响接口**: hidumper组件

### 安装的应用是已卸载的预置应用时校验签名是否一致
- **Kit**: 调试工具
- **原因**: 预置应用被卸载后可以安装一个bundleName相同、签名不同的hap仿冒，有安全风险。
- **影响接口**: [bm工具](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/bm-tool#安装命令install)安装命令。

### 禁止Extension进程拉起启动框架
- **Kit**: Ability Kit
- **原因**: Extension进程不应拉起启动框架，启动框架是用于优化UIAbility启动时的一些启动任务，当Extension可以拉起启动框架时，可能会导致应用未启动便执行启动框架，导致一些代码在不应执行的时间点执行。
- **影响接口**: AppStartup启动框架模块默认行为。

### 在字节码HAR中通过router.getState()获取的path内容变更
- **Kit**: ArkUI
- **原因**: 当开发者使用中间码HAR升级到字节码HAR时，通过router.getState()方法获取的path信息不正确。
- **影响接口**: router.getState()

### 优化getWindowProperties，增加返回值中drawableRect的实时性，调用行为变更
- **Kit**: ArkUI
- **原因**: 应用调用getWindowProperties可以获取窗口属性，返回的结构体中表示可绘制区域的字段为drawableRect，如果在on('windowSizeChange')回调中调用getWindowproperties，可能获得未更新的drawableRect。
- **影响接口**: @ohos.window.d.ts

### RichEditor（富文本）从组件外拖入内容onWillChange、onDidChange回调变更
- **Kit**: ArkUI
- **原因**: 从组件外拖入内容时，onWillChange、onDidChange多回调了一次相同的内容，不符合实际文本变化情况。
- **影响接口**: RichEditor

### RichEditor（富文本）onWillChange接口返回值变更
- **Kit**: ArkUI
- **原因**: 在添加Symbol时onWillChange接口返回值中缺少了SymbolId。
- **影响接口**: RichEditor

### RichEditor（富文本）TypingStyle默认字体大小变更
- **Kit**: ArkUI
- **原因**: 开发者在设置TypingStyle但是没有设置其中的字体大小时，TypingStyle的默认字体大小为14px，显示效果异常。
- **影响接口**: RichEditor

### RichEditor（富文本）onDidChange接口变更
- **Kit**: ArkUI
- **原因**: 在用户执行删除操作，但实际未删除内容时（例如在aboutToDelete接口中拦截了删除操作），也回调了OnDidChange接口，不符合接口行为定义。
- **影响接口**: RichEditor

### RichEditor（富文本）删除完成后光标位置变更
- **Kit**: ArkUI
- **原因**: 开发者在aboutToDelete回调中设置光标/选中区后，删除完成后，光标位置异常。
- **影响接口**: RichEditor

### 国家、地区本地化名称变更
- **Kit**: Localization Kit
- **原因**: 1、当前中国香港、中国澳门、中国台湾地区的本地化名称中使用逗号或空格作为分隔符，当多个国家并列时，存在歧义。
- **影响接口**: i18n.System.getDisplayCountry

### 时间日期格式“十一月”格式化结果错误问题修改
- **Kit**: Localization Kit
- **原因**: 参考业界标准，农历中“十一月”应该显示为“冬月”。
- **影响接口**: intl.DateTimeFormat.format

### 日期时间段格式化在zh-Hant-HK下结果错误问题修改
- **Kit**: Localization Kit
- **原因**: 当区域为zh-Hant-HK时，日期时间段格式化结果返回空字串。
- **影响接口**: intl.DateTimeFormat.formatRange

### 归属地获取接口对无效号码的归属计算错误问题修改
- **Kit**: Localization Kit
- **原因**: 归属地获取接口在传入无效号码时，会返回PhoneNumberFormat对象创建时传入的地区作为号码归属地。
- **影响接口**: i18n.PhoneNumberFormat.getLocationName

### 系统支持国家地区列表变更
- **Kit**: Localization Kit
- **原因**: 不丹当前不在系统支持的国家地区列表中。
- **影响接口**: i18n.System.getSystemCountries

### 禁止bm命令进行跨用户操作
- **Kit**: 工具
- **原因**: bm命令行工具未对sh调用方用户身份做校验，用户A可以通过bm命令安装、卸载其他用户空间下的应用，并且可以通过bm命令嗅探其他空间下已安装的应用，违反安全规范。
- **影响接口**: bm命令行工具

### 动画接口在播放次数为无限循环时的行为变更
- **Kit**: ArkUI
- **原因**: 非动画的闭包函数修改状态变量，存在被带入无限循环动画的风险，产生预期外的无限循环动画且无法停止。
- **影响接口**: 1、animateTo；

### 系统对单应用最大创建UIAbility数量限制变更
- **Kit**: ArkUI
- **原因**: 系统防护机制增强，系统会针对设备上单应用最大创建UIAbility数量进行限制，如果数量超过限制，系统可能会将最早未使用的任务删除。
- **影响接口**: 任务管理规格

### Video组件不再默认解析并自动播放拖拽信息中的视频资源
- **Kit**: ArkUI
- **原因**: Video组件默认允许拖入任意视频并自动播放的行为不符合终端用户预期，故需调整该默认规格。
- **影响接口**: Video组件。

### onErrorReceive接口在首次加载woff等在线字体资源不再触发的行为变更
- **Kit**: ArkWeb
- **原因**: woff等在线字体资源首次加载时不再指定仅从缓存中读取。
- **影响接口**: web.d.ts的onErrorReceive接口

### PostCardAction的router事件允许拉起Ability类型范围变更
- **Kit**: Form Kit
- **原因**: PostCardAction的router事件当前未对被拉起的Ability类型进行校验，但实际此事件应只允许拉起UIAbility，针对使用router事件拉起非UIAbility的场景，需要做安全加固。
- **影响接口**: PostCardAction

### app.json中bundleName字段正则匹配规则修改
- **Kit**: 公共能力
- **原因**: app.json中对bundleName的正则匹配规则较简单，不符合应用包名规范，进行整改。
- **影响接口**: Openharmony SDK目录下toolchains/modulecheck/app.json scheme文件。

### 应用发生OOM时Heapdump产物文件格式变更
- **Kit**: 工具
- **原因**: 为了提升应用发生OOM时Heapdump生成dump文件的概率和效率，支撑开发者定位内存泄漏问题：
- **影响接口**: OpenHarmony的SDK在toolchains目录下新增rawheap\_translator工具。

### 禁止通过wantAgent拉起ExtensionAbility
- **Kit**: Ability Kit
- **原因**: 三方应用创建拉起ExtensionAbility的wantAgent，并发送携带此wantAgent的通知。通过桌面点击通知拉起该ExtensionAbility，应用退出后，该ExtensionAbility进程无法退出从而影响功耗。
- **影响接口**: wantAgent.trigger接口

### startAbility/openLink锁屏时限制拉起行为
- **Kit**: ArkUI
- **原因**: 锁屏时对任意拉起应用的行为增加限制。
- **影响接口**: startAbility/openLink

### @ohos.file.photoAccessHelper相册添加资产、移除资产等接口行为变更
- **Kit**: Media Library Kit
- **原因**: 为了优化资产的保存逻辑，修改了媒体库对于“资产-相册”的对应关系。修改前一个图片/视频资源可以通过关系映射存在于多个逻辑相册中，修改后一张照片仅能归属一个实体相册。
- **影响接口**: 1.  向相册中添加资产：addAssets(assets: Array<PhotoAsset>): void
    
2.  从相册中移除资产：removeAssets(assets: Array<PhotoAsset>): void
    
3.  设置相册名称：
    
4.  setAlbumName(name: string):void，
    
    commitModify(callback: AsyncCallback<void>): void
    
5.  注册对指定uri的监听：registerChange(uri: string, forChildUris:

### 拉起UIExtensionAbility增加调用方为前台应用的校验
- **Kit**: Ability Kit
- **原因**: UIExtensionAbility仅支持拥有前台窗口的应用拉起，虽有该约束，但未校验调用方是否为前台应用，存在后台应用拉起UIExtensionAbility的风险。
- **影响接口**: 1、UIAbilityContext.startAbilityByType、UIExtensionContentSession.startAbilityByType

### 借助Want分享文件URI时无权限的URI会被拦截
- **Kit**: Ability Kit
- **原因**: 在want的flags字段设置了授权flag前提下，禁止在want中的URI字段和wantConstant.Params.PARAMS\_STREAM字段中传入无权限的[File URI](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-file-fileuri-V5#fileuri10)。
- **影响接口**: 不涉及

### childProcessManager增加子进程最大数量限制
- **Kit**: Ability Kit
- **原因**: 应用可以无限制启动子进程，这个场景存在恶意调用风险，因此需要增加子进程最大数量限制。
- **影响接口**: childProcessManager中的startChildProcess(非SELF\_FORK模式)、startArkChildProcess接口。

### 包管理@ohos.bundle.bundleManager.d.ts中的bundleManager.verifyAbc接口以及bundleManager.deleteAbc接口变更为system API
- **Kit**: Ability Kit
- **原因**: 因安全合规要求，包管理@ohos.bundle.bundleManager.d.ts中的bundleManager.verifyAbc接口以及bundleManager.deleteAbc接口变更为system API。
- **影响接口**: @ohos.bundle.bundleManager.d.ts中如下接口：

### startAbility从后台发起时，即使应用正在运行长时任务，也不能调用成功。
- **Kit**: Ability Kit
- **原因**: 应用从后台使用startAbility拉起自己或者其他应用，这种行为是禁止的。但是利用长时任务，当前可以达到应用在后台依然startAbility成功的效果。这个场景存在恶意弹窗等风险，因此现在禁止应用在运行长时任务时，从后台调用startAbility的行为。
- **影响接口**: 各个context中的startAbility接口。

### util.TextEncoder模块utf-16le和utf-16be编码数据行为变更
- **Kit**: ArkTS
- **原因**: TextEncoder编码器在编码格式设置为utf-16le和utf-16be时，获得的编码数据相反。
- **影响接口**: util.TextEncoder模块的接口：

### Worker模块中RestrictedWorker的接口变更为system API
- **Kit**: ArkTS
- **原因**: 因安全合规要求，RestrictedWorker的接口由public API变更为system API。
- **影响接口**: worker模块下的两个接口：

### 优化状态变量场景下ForEach的冗余刷新行为
- **Kit**: ArkUI
- **原因**: 开发者使用ForEach，在旧节点下树的时候会被刷新一次，属于规格外的行为，当刷新调用方法中存在对全局变量的修改时会发生异常。
- **影响接口**: ForEach

### OH\_AVCodecBufferAttr 接口行为变更
- **Kit**: AVCodec Kit
- **原因**: 针对视频轨获取的 OH\_AVCodecBufferAttr 中 pts 属性，会在文件实际封装信息的基础上减去轨道起始时间，使其从0开始，
- **影响接口**: | 名称 | 描述 |
| --- | --- |
| OH_AVCodecBufferAttr | 定义OH_AVCodec的缓冲区描述信息 |

### CameraPosition.CAMERA\_POSITION\_FOLD\_INNER接口废弃
- **Kit**: Camera Kit

### file-access-across-devices 接口行为变更
- **Kit**: Core File Kit
- **原因**: 分布式文件业务在检测到可信设备连接同一个Wi-Fi会自动触发建链，但该规格存在以下两个问题：
- **影响接口**: 本次变更仅涉及新增以下接口：

### setSecurityLabel接口行为变更
- **Kit**: Core File Kit
- **原因**: setSecurityLabel接口未对设置的安全等级进行校验，未阻止应用将高风险等级数据设置为低风险等级，有安全风险，需要进行整改。
- **影响接口**: | 接口声明 | 变更前 | 变更后 |
| --- | --- | --- |
| setSecurityLabel(path:string, type:DataLevel):Promise<void> | 数据风险等级由高向低设置未做拦截 | 数据风险等级由高向低设置时抛错 |
| setSecurityLabel(path:string, type:DataLevel, callback: AsyncCallback<void>):void | 数据风险等级由高向低设置未做拦截 | 数据风险等级由高向低设置时抛错 |
| setSecurityLabelSync(path:string,

### getSecurityLabel接口行为变更
- **Kit**: Core File Kit
- **原因**: getSecurityLabel接口获取未对设置过的数据风险等级的数据返回值为空字符串，不符合安全设计。需要修改实现返回“s3”。
- **影响接口**: | 接口声明 | 变更前 | 变更后 |
| --- | --- | --- |
| getSecurityLabel(path:string, callback:AsyncCallback<string>): void | 默认返回空字符串 | 默认返回“s3” |
| getSecurityLabel(path:string):Promise<string> | 默认返回空字符串 | 默认返回“s3” |
| getSecurityLabelSync(path:string):string | 默认返回空字符串 | 默认返回“s3” |

### 文件授权接口行为变更
- **Kit**: Core File Kit
- **原因**: 基于对用户个人文件资产保护，文件必须先授权后使用。文件授权接口实行强管控，提供目录授权和文件持久化授权。
- **影响接口**: 接口未变更，行为变更；变更后对文件/目录授权实行强管控。授权接口见下表：

### 使用udid-hash与appid和盐值基于sha256混淆截断后保留前16位值作为deviceid，调用行为变更
- **Kit**: Distributed Service Kit
- **原因**: 分布式设备管理deviceid接口存在安全漏洞，应用获取的deviceid在设备重置后或者应用卸载重装后不变，对于同一个应用可以标识设备。
- **影响接口**: 不涉及

### 命令行调试工具使用权限缩小
- **Kit**: 调试命令
- **原因**: 基于对应用的安全隐私保护，调试能力遵循以下原则进行调整：
- **影响接口**: | 组件 | 命令 | 变更前 | 变更后 |
| --- | --- | --- | --- |
| hidumper | hidumper --mem-smaps pid [-v] | 有内存信息打印，且能获取内存地址信息 | 将打印帮助信息，不支持该命令 |
| hidumper | hidumper -p [pid] | 可以导出任意进程的maps信息以及/proc/pid/mountinfo等信息 | 只支持导出debug应用的基础信息，所有进程均不支持导出maps信息 |
| hiprofiler | nativehook | nativehook插件可以对任意进程进行调优 | 使

### hdc工具tmode usb命令功能废弃
- **Kit**: 调试命令
- **原因**: 按照开发者需求，hdc应同时支持USB调试和无线调试。
- **影响接口**: | 组件 | 命令 |
| --- | --- |
| hdc | hdc tmode usb |

### aa工具不支持对Release模式的应用进行调试
- **Kit**: 调试命令
- **原因**: aa工具是DevEco Studio调试能力的底层依赖，正常的调试场景都通过DevEco Studio进行，正常情况下DevEco Studio会限制对Release模式的应用进行调试，但存在绕过DevEco Studio直接使用aa工具的调试命令进行调试的风险，应当从根本上限制此行为。
- **影响接口**: aa工具

### 入口图标和入口标签规格变动
- **Kit**: Ability Kit
- **原因**: [入口图标和名称](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/module-configuration-file#配置文件标签)（module.json5中abilities配置的icon和label）是应用安装后在桌面上展示的图标和名称，[应用图标和名称](https://developer.huawei.com/
- **影响接口**: bundleManager模块提供的查询接口返回的icon和label会按照优先级返回。

### TextEncoder.encodeInto()接口变更
- **Kit**: ArkTS
- **原因**: TextEncoder.encodeInto编码字符串，当字符串中每多一个'\\0'字符，返回的数组长度会增加2，长度异常。
- **影响接口**: TextEncoder.encodeInto

### Search/TextInput/TextArea onchange回调新增可选参数预上屏文本
- **Kit**: ArkUI
- **原因**: 在输入过程中，新增预上屏行为，需要将当前已提交上屏的文本和预上屏信息实时返回给开发者。
- **影响接口**: Search/TextInput/TextArea组件的onChange回调事件参数。

### 资源匹配逻辑变更
- **Kit**: Localization Kit
- **原因**: 插入SIM卡时未能获取到正确的mcc目录资源，影响开发者差异化定制资源。
- **影响接口**: SystemCapability.Global.ResourceManager获取资源相关接口。

### raw\_file模块头文件变更
- **Kit**: Localization Kit
- **原因**: raw\_file模块引用了string头文件，string头文件是C++标准库文件，导致raw\_file模块在C语言环境下无法使用。
- **影响接口**: | 变更前 | 变更后 |
| --- | --- |
| #include <string> | #include <stddef.h>#include <stdint.h>#include <stdbool.h> |

### webSocket.connect接口新增2302998错误码
- **Kit**: Network Kit
- **原因**: 为规范元服务请求域名范围，提升元服务上架审核效率和平台合规经营安全性，当用户使用元服务时，将根据该元服务的域名配置进行域名访问；若是访问未配置域名，则connect接口会返回该错误码。
- **影响接口**: @ohos.net.webSocket

### bm命令行工具变更
- **Kit**: Ability Kit
- **原因**: bm install、bm uninstall命令在-u（即用户）未指定情况下，默认为全部用户。基于安全考虑，变更为默认当前活跃用户。

### 键值型数据库跨设备自动同步的使用场景变更
- **Kit**: ArkData
- **原因**: 解决跨设备无效数据同步问题。
- **影响接口**: autoSync/分布式键值型数据库（distributedKVStore）

### util.TextDecoder模块ignoreBOM功能行为变更
- **Kit**: ArkTS
- **原因**: util.TextDecoder模块ignoreBOM参数未使能，无法对存在BOM标记的数据进行正常解析。
- **影响接口**: 为确保版本之间的兼容性，对util.TextDecoder模块ignoreBOM使能的相关接口进行废弃，并新增对应方法。

### Base64Helper、StringDecoder模块部分接口参数异常错误码由undefined变更为401
- **Kit**: ArkTS
- **原因**: 接口参数异常错误码规格为401，但实际为抛出错误码为undefined，开发者无法精确找到问题原因。
- **影响接口**: | 类名 | 接口 | 变更 |
| --- | --- | --- |
| util.Base64Helper | encodeToStringSync(src: Uint8Array, options?: Type): string; | 参数异常由undefined变更为401。 |
| util.Base64Helper | encode(src: Uint8Array, options?: Type): Promise<Uint8Array>; | 参数异常由undefined变更为401。 |
| util.Base64Helper | encodeSync(src: Uint8A

### reminderAgentManager.publishReminder权限管控
- **Kit**: Background Tasks Kit
- **原因**: 由于应用存在滥用后台代理提醒能力，利用该能力发送广告、营销类延时提醒，影响用户体验；因此针对此问题，后台代理提醒增加管控机制，未通过管控的应用无法使用后台代理提醒能力。
- **影响接口**: reminderAgentManager.publishReminder

### @ohos.multimedia.medialibrary变更
- **Kit**: Media Library Kit
- **原因**: 根据工信部关于应用软件调用行为记录能力要求，需要进行媒体库接口能力变更。
- **影响接口**: @ohos.multimedia.medialibrary模块中的所有接口。

### 匿名内存执行权限管控策略变更说明
- **Kit**: NDK开发
- **原因**: 为了维护生态的纯净，防止恶意应用向匿名内存注入指令，实现任意代码执行，以绕过代码签名管控，达到变脸或攻击系统的目的，系统要限制应用内设置匿名内存为可执行的行为。
- **影响接口**: 不涉及

### 应用编译构建对不支持命令强校验
- **Kit**: NDK开发
- **原因**: cmake从3.16.5版本升级到3.28.2版本，引入了该项变更。
- **影响接口**: 不涉及

### 应用编译构建建议cmake\_minimum\_required修改为不低于3.5.0的版本
- **Kit**: NDK开发
- **原因**: cmake从3.16.5版本升级到3.28.2版本，引入了该项变更。
- **影响接口**: 不涉及

### 应用编译构建harA链接harB的so，导致hap包so冲突
- **Kit**: NDK开发
- **原因**: cmake从3.16.5版本升级到3.28.2版本，引入了该项变更。
- **影响接口**: 不涉及

### fileAccess接口行为变更
- **Kit**: ArkWeb
- **原因**: API version 11及之前，fileAccess为false的时候，应用使用file协议依然可以加载部分可读可写目录，造成注入攻击的安全问题。同时将fileAccess默认值变更为false，提高应用安全性。
- **影响接口**: 变更前接口行为

### Asset属性类型变更
- **Kit**: ArkData
- **原因**: Asset的属性在HarmonyOS NEXT Developer Beta1阶段增加了undefined类型，破坏了Asset的通用性。
- **影响接口**: Asset/数据通用类型（commonType）

### setSessionId接口行为变更
- **Kit**: ArkData
- **原因**: 分布式数据对象的同步数据时，会尝试同步与同账号网络中的所有设备建立链接，并进行数据同步。但实际上组网中的其他设备未必创建了分布式数据对象，它们不需要同步数据。所以可能产生无效的数据同步，浪费系统资源。
- **影响接口**: setSessionId/分布式数据对象（data\_object）

### OH\_Drawing\_RegisterFont()、OH\_Drawing\_RegisterFontBuffer()接口增加报错码
- **Kit**: ArkGraphics 2D
- **原因**: OH\_Drawing\_RegisterFont()、OH\_Drawing\_RegisterFontBuffer()接口缺少对文件内容校验的报错。
- **影响接口**: OH\_Drawing\_RegisterFont()、OH\_Drawing\_RegisterFontBuffer()接口增加报错码：9 文件损坏。

### 编译语法检查场景补齐
- **Kit**: ArkTS
- **原因**: interface的属性名为数字字面量不符合ArkTS语法规则，编译语法检查场景遗漏，需要修复。
- **影响接口**: 不涉及。

### TreeSet.clear()和TreeMap.clear()接口行为变更
- **Kit**: ArkTS
- **原因**: 容器类TreeMap和TreeSet的构造函数支持传入比较器，用户可以通过传入比较器来控制元素在二叉树上的插入位置。当用户调用了TreeMap和TreeSet的clear方法时，预期结果为仅清除数据，
- **影响接口**: TreeMap.clear();

### TreeMap.setAll()接口行为变更
- **Kit**: ArkTS
- **原因**: 使用setAll接口添加一个空的TreeMap，预期长度+0，实际长度+1。
- **影响接口**: TreeMap.setAll();

### TreeMap、TreeSet的hasKey和has接口行为变更
- **Kit**: ArkTS
- **原因**: 1.  对于传入用户自定义比较器的TreeMap， 不能正确区分undefined或null与其他元素的比较关系，导致在没有插入null或undefined的情况下hasKey(null/undefined)错误返回true。
- **影响接口**: TreeMap.hasKey();

### URLParams类append接口添加的值中包含连续空格时行为变更
- **Kit**: ArkTS
- **原因**: URLParams类append接口添加的字符串中包含连续多个空格时，连续空格会被错误处理成只转换为一个'+'，该实现不符合URL的标准。
- **影响接口**: URLParams对象的append接口。

### URLParams类在入参字符串中包含大小写不同编码值时，toString()接口表现结果不一致变更
- **Kit**: ArkTS
- **原因**: URLParams类在入参字符串中包含大小写不同编码值时toString()接口返回值有误，主要涉及："%2b"和"%2B"的表现不一致，"%2B"被处理为"%2B"是符合标准的，但"%2b"会被错误处理成'+'。
- **影响接口**: URL模块URLParams类toString接口，

### URLParams类append接口行为变更
- **Kit**: ArkTS
- **原因**: URLParams使用append()方法时，对入参键值对中的特殊字符会错误进行encode，该行为与URL标准不一致，导致后续对键值对进行增删改查等操作出现异常。
- **影响接口**: URLParams类append接口。

### @Component和@ComponentV2修饰的自定义组件使用@Observed或者@ObservedV2修饰的类增加相关校验
- **Kit**: ArkUI
- **原因**: 此变更涉及应用适配。

### TextTimer的onTimer回调频率与参数单位调整
- **Kit**: ArkUI
- **原因**: 调整onTimer回调频率与参数的单位，使其符合文档描述。
- **影响接口**: textTimer组件的onTimer接口

### RichEditor组件builderSpan支持绑定自定义菜单
- **Kit**: ArkUI
- **原因**: 富文本支持builderSpan绑定自定义菜单。
- **影响接口**: RichEditor组件的RichEditorSpanType接口。

### RichEditor的lineHeight、letterSpacing、lineSpacing属性返回值单位变更
- **Kit**: ArkUI
- **原因**: 文本字体相关属性的返回值单位应默认使用fp类型。
- **影响接口**: RichEditor组件，lineHeight、letterSpacing、lineSpacing属性。

### RichEditor的fontSize属性的返回值单位变更
- **Kit**: ArkUI
- **原因**: 文本字体相关属性的返回值单位应默认使用fp类型。
- **影响接口**: RichEditor组件，fontSize属性。

### 文本计算接口fontSize参数默认单位实现修正
- **Kit**: ArkUI
- **原因**: fontSize参数在文档描述中number类型默认单位是fp，实际实现是vp。
- **影响接口**: measureText和measureTextSize接口。

### @ohos.multimedia.medialibrary变更
- **Kit**: Media Library Kit
- **原因**: 根据工信部关于应用软件调用行为记录能力要求，需要进行媒体库接口能力变更。
- **影响接口**: @ohos.multimedia.medialibrary模块中的所有接口。

### 开发态下代码签名强制管控策略变更
- **Kit**: 公共能力
- **原因**: 为统一底层代码签名管控逻辑，降低鸿蒙系统被攻击的安全风险，所有运行在设备端的应用代码都需要包含合法签名。应用进程加载代码时，系统会检查对应文件签名的合法性。禁止未签名及签名不合法的代码在设备上运行。
- **影响接口**: 不涉及


## 🟡 版本隔离（targetSdkVersion ≥ 5.0.0 生效，18 条）

### 数据库插入长度为0的Uint8Array的数据，getRow、getValue 接口返回值发生变化
- **Kit**: -
- **原因**: 当插入列类型是blob且数据为长度为0的Uint8Array时，数据库对应错误写入null值，导致使用getRow和getValue接口读取时，读取数值结果错误与插入数据不匹配。
- **影响接口**: @ohos.data.relationalStore.d.ts中getRow接口。

### 关系型数据管理@ohos.data.relationalStore.d.ts中getRdbStore接口新增错误码14800020，用于业务侧进行恢复重建数据库
- **Kit**: -
- **原因**: 根密钥和工作密钥不匹配时返回的错误码不正确，新增14800020错误码，此错误码用于业务侧进行恢复重建数据库。
- **影响接口**: @ohos.data.relationalStore.d.ts中如下接口：

### ImageAttributeModifier支持new方式创建ColorFilter对象传入colorFilter接口变更
- **Kit**: -
- **原因**: ImageAttributeModifier不支持new方式创建ColorFilter对象传入colorFilter接口，修复后即可增加colorFilter接口的支持范围，并且ColorFilter在业务代码中传递更加便捷。
- **影响接口**: ImageAttributeModifier的colorFilter接口。

### List组件首次创建布局时，Scroller控制器的跳转方法优先级变更为高于initialIndex的优先级
- **Kit**: -
- **原因**: ![](https://contentcenter-vali-drcn.dbankcdn.cn/pvt_2/DeveloperAlliance_scene_100_1/32/v3/A_z2eIT_Sz2kIqKjCgDghg/note_3.0-zh-cn.png?HW-CC-KV=V1&HW-CC-Date=20260706T023818Z&HW-CC-Expire=86400&HW-CC-Sig
- **影响接口**: List组件的initialIndex接口和Scroller控制器的跳转接口（scrollToIndex、scrollToItemInGroup和scrollEdge）。

### Image组件的borderRadius接口支持动态修改
- **Kit**: -
- **原因**: 为了增强功能的灵活性，Image组件的borderRadius接口支持动态修改。动态修改可以实时更新borderRadius的值，灵活地调整图片的圆角效果。例如，可根据用户交互或状态变化即时改变圆角半径。
- **影响接口**: Image组件的borderRadius接口。

### RichEditor（富文本）在光标处于文本起始位置情况时向前删除空文本onWillChange回调变更
- **Kit**: -
- **原因**: RichEditorController构造的富文本：光标位于文本起始位置时向前删除，触发onWillChange回调范围是\[-1, -1\]，不符合接口定义。
- **影响接口**: RichEditor

### 修复zIndex接口会影响组件在3D变换中的透视效果的错误行为
- **Kit**: -
- **原因**: zIndex接口在3D变换场景与translateZ耦合，导致zIndex值的改变会影响3D变换时的透视效果。
- **影响接口**: zIndex接口

### 屏幕Display对象rotation和orientation属性变更
- **Kit**: -
- **原因**: 手机、平板等不同设备，电子器件贴片方向具有差异，传感器自然方向与屏幕的物理方向具有偏差，导致平板上屏幕Display对象中的rotation和orientation的变化方向和手机不统一，给用户带来困扰，需要针对设备类型做特殊处理，影响使用。
- **影响接口**: @ohos.display.d.ts文件中屏幕Display对象的rotation和orientation属性。

### 在PC/2in1设备上getWindowStatus和on('windowStatusChange')接口在窗口最大化状态返回值变更
- **Kit**: -
- **原因**: getWindowStatus和on('windowStatusChange')接口在最大化状态返回值为WindowStatusType::FULL\_SCREEN，与实际情况不符合。
- **影响接口**: @ohos.window.d.ts

### image.Component.setAuxiliaryPictureInfo接口行为变更
- **Kit**: -
- **原因**: 通过setAuxiliaryPictureInfo设置辅助图信息时，会将辅助图信息中的size、pixelFormat等信息同步到pixelMap的ImageInfo中，需要对size和pixelFormat信息做合法校验，防止对pixelMap像素数据信息的越界访问。
- **影响接口**: | API类型 | 所在d.ts文件 | 接口名 | 接口起始版本 |
| --- | --- | --- | --- |
| ArkTS API | @ohos.multimedia.image.d.ts | setAuxiliaryPictureInfo(info: AuxiliaryPictureInfo): void | 13 |

### image.Component.OH\_AuxiliaryPictureNative\_SetInfo()接口行为变更
- **Kit**: -
- **原因**: 通过OH\_AuxiliaryPictureNative\_SetInfo设置辅助图信息时，会将辅助图信息中的size、pixelFormat等信息同步到pixelMap的ImageInfo中，需要对size和pixelFormat信息做合法校验，防止对pixelMap像素数据信息的越界访问。
- **影响接口**: | API类型 | 所在头文件 | 接口名 | 接口起始版本 |
| --- | --- | --- | --- |
| C API | picture_native.h | Image_ErrorCode OH_AuxiliaryPictureNative_SetInfo(OH_AuxiliaryPictureNative *auxiliaryPicture, OH_AuxiliaryPictureInfo *info) | 13 |

### image接口Heif格式类型变更
- **Kit**: -
- **原因**: 相机Heif编码时，使用图片标准定义image/heic参数编码失败，当前版本Image中的格式参数定义为image/heif，不符合图片标准定义，为提升规范性，将Heif格式图片mimeType变更为image/heic。
- **影响接口**: | API类型 | 所在d.ts/头文件 | 接口名 | 接口起始版本 |
| --- | --- | --- | --- |
| C API | image_packer_native.h | Image_ErrorCode OH_PackingOptions_SetMimeType(OH_PackingOptions *options, Image_MimeType *format) | 12 |
| C API | image_packer_native.h | Image_ErrorCode OH_PackingOptions_GetMimeType(OH_PackingOptions

### AVErrorCode枚举值变更
- **Kit**: -
- **原因**: 播放器当前上报的IO错误码只有一个，为了帮助开发者更好地了解播放失败的原因，本次细化了IO相关错误，提升生态应用友好度。
- **影响接口**: 1.  @ohos.multimedia.media.d.ts中接口：
    
    AVPlayer.on(type: 'error', callback: ErrorCallback)。
    
2.  avplayer\_base.h中接口：
    
    void (\*OH\_AVPlayerOnErrorCallback)(OH\_AVPlayer \*player, int32\_t errorCode, const char \*errorMsg)。

### 自定义界面扫码权限校验错误码变更
- **Kit**: -
- **原因**: 按照错误码规范要求，权限不足时报201错误码。
- **影响接口**: customScan模块下接口：

### 集成自定义界面扫码应用适配窗口子系统属性变更
- **Kit**: -
- **原因**: 窗口子系统[屏幕Display对象rotation和orientation属性变更](https://developer.huawei.com#屏幕display对象rotation和orientation属性变更)，会导致集成自定义界面扫码的应用相机预览画面和码图位置出现错误，影响用户使用。
- **影响接口**: @ohos.display.d.ts文件中屏幕Display对象的rotation和orientation属性。

### convertXml模块未支持parentKey属性的行为变更
- **Kit**: -
- **原因**: convertXml模块未实现parentKey属性，生成的object中不具有parentKey属性的值。
- **影响接口**: ConvertXML模块下的接口：

### 禁止在转场动画过程中，更新消失节点的属性。
- **Kit**: -
- **原因**: 在转场动画过程中改变正在消失节点的属性，可能造成数据访问异常，产生crash。例如，动画过程中将data置为undefined，Text组件增加默认转场不会立即被删除，在更新状态时，数据访问异常产生crash。因此，需要变更为在转场动画过程中，禁止更新消失节点的属性。
- **影响接口**: transition属性

### 鼠标按键处理行为变更
- **Kit**: -
- **原因**: 在开发者为组件配置鼠标事件后，若在组件区域内按下鼠标非左键并拖拽至组件区域外释放，此时将无法接收到按键释放事件，这可能导致事件配对失败，进而引发应用程序行为异常。此变更确保开发者能够接收到匹配的按键按下与释放事件。
- **影响接口**: ArkTS的onMouse接口，Native的OH\_NativeXComponent\_GetMouseEvent接口和registerNodeEvent接口的NODE\_ON\_MOUSE。
