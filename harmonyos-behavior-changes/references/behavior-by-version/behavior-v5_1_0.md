# 行为变化 — 版本 5.1.0（共 23 条）

> 数据来源：`changelogs-for-all-apps-*.md`
> 「版本隔离=是」表示 targetSdkVersion ≥ 对应版本才生效；「否」表示所有应用都受影响。


## 🔴 无版本隔离（必须适配，11 条）

### 默认不合并依赖混淆规则变更说明
- **Kit**: ArkTS
- **原因**: 混淆默认合并依赖混淆规则，由于部分开发者不了解混淆机制，在发布的三方库中的obfuscation.txt文件中引入了混淆开关选项，其它应用依赖这些三方库后出现应用启动崩溃，且开发者无法直接感知这些三方库引入了混淆开关。
- **影响接口**: 不涉及

### 当使用自定义组件名和内置属性重名时编译报错变更
- **Kit**: ArkUI
- **原因**: 当使用自定义组件名和内置属性重名时，编译会根据指定的白名单进行拦截报错，如果白名单中不存在，编译就拦截不到，导致编译转换后的产物出现问题。
- **影响接口**: ArkUI 内置组件属性API。

### setSpecificSystemBarEnabled接口在横屏的行为变更
- **Kit**: ArkUI
- **原因**: 修正接口规格，保证接口调用在不同场景下均可正常控制状态栏显隐。
- **影响接口**: Window#setSpecificSystemBarEnabled(name: SpecificSystemBar, enable: boolean, enableAnimation?: boolean): Promise<void>

### ArkUI双指长按行为变更
- **Kit**: ArkUI
- **原因**: 在手机、平板设备上，优先响应系统双指操作，默认禁用应用内双指长按。
- **影响接口**: 长按手势 LongPressGesture

### osAccount.distributedAccount.DistributedAccountAbility.getOsAccountDistributedInfo接口返回值生成规则变更
- **Kit**: Basic Services Kit
- **原因**: 原接口返回值生成规则与设计不符。
- **影响接口**: @ohos.account.distributedAccount.d.ts中如下接口:

### 原hiai\_foundation目录下的头文件废弃，替换为CANNKit下的头文件
- **Kit**: CANN Kit（原HiAI Foundation Kit）
- **原因**: 命名优化。
- **影响接口**: 不涉及

### 卡片支持的接口能力增加校验
- **Kit**: Form Kit
- **原因**: 针对支持卡片能力的接口，增加校验。
- **影响接口**: 不涉及

### 应用创建SoundPool时调用media.createSoundPool接口变更
- **Kit**: Media Kit
- **原因**: 原来的应用创建SoundPool实例，一个应用进程只能够创建一个SoundPool实例，为单实例模式，无法满足应用的使用场景。例如很多应用在使用SoundPool的能力的同时，还需要使用TimePicker组件(该组件中封装了SoundPool的能力)，单实例模式会使得SoundPool对象之间互相干扰，影响应用的使用体验。
- **影响接口**: 无变更接口

### 用户态trace打点格式变更
- **Kit**: Performance Analysis Kit
- **原因**: 优化调整原有用户态trace打点格式，提高用户态trace打点格式的前向兼容性。
- **影响接口**: -   OH\_HiTrace\_StartAsyncTrace
-   OH\_HiTrace\_FinshAsyncTrace
-   OH\_HiTrace\_CountTrace
-   startTrace
-   finishTrace
-   traceByValue

### JIT功能默认关闭，需申请权限证书并通过审核后启用
- **Kit**: NDK开发
- **原因**: JIT（Just In Time）即时编译功能可能带来任意代码注入的风险。为了保护应用安全并维护鸿蒙系统的纯净生态，系统默认关闭JSVM的JIT功能，采用解释执行方式执行JS代码。
- **影响接口**: OH\_JSVM\_CompileWasmModule：无JIT权限会返回JIT\_MODE\_EXPECTED状态码

### hiperf命令行调试工具使用权限缩小
- **Kit**: 调试工具
- **原因**: 基于对应用的安全隐私保护，调试能力遵循以下原则进行调整：
- **影响接口**: hiperf命令行工具。


## 🟡 版本隔离（targetSdkVersion ≥ 5.1.0 生效，12 条）

### getKeyboardAvoidMode接口返回值变更
- **Kit**: -
- **原因**: getKeyboardAvoidMode接口实际返回值为字符串，与文档描述返回值为KeyboardAvoidMode枚举值类型不符。
- **影响接口**: getKeyboardAvoidMode

### XComponent组件上使用renderFit接口显示效果变更
- **Kit**: -
- **原因**: 优化XComponent组件上使用renderFit接口显示效果的正确性。
- **影响接口**: 涉及接口: renderFit。

### CanvasRenderingContext2D方法传NaN和Infinity值后执行的其他绘制方法由不绘制变更为正常绘制
- **Kit**: -
- **原因**: CanvasRenderingContext2D绘制路径时，若存在方法的number类型参数传入NaN或Infinity值，部分内容无法绘制或绘制异常，与Html5的行为不一致。
- **影响接口**: CanvasRenderingContext2D

### CanvasRenderingContext2D的drawImage接口默认单位变更
- **Kit**: -
- **原因**: 当drawImage传入9个参数时，若首个参数（image）为PixelMap类型，则第2至第5个参数（sx、sy、sw和sh）以px为单位进行解析。与文档描述不一致，且绘制得到的图片大小存在问题。
- **影响接口**: CanvasRenderingContext2D的drawImage接口

### 修复blendMode接口离屏模式会影响组件设置的不透明度的问题
- **Kit**: -
- **原因**: blendMode离屏模式与不透明度属性（opacity）同时使用时，组件的不透明度并不等于设置的不透明度，效果异常。
- **影响接口**: blendMode

### XComponent设置为Texture模式使用blendMode接口的行为由不生效变更为正常生效
- **Kit**: -
- **原因**: 用户使用XComponent组件并设置为Texture模式时，使用blendMode接口没有效果，不符合接口正常规格，需要变更为blendMode接口正常生效。
- **影响接口**: common.d.ts文件的blendMode接口。

### 滚动组件通用接口backToTop属性默认值变更
- **Kit**: -
- **原因**: List、Grid、Scroll和WaterFlow组件默认不支持点击状态栏回到顶部，为了推广点击状态栏回到顶部特性，提升应用体验，List、Grid、Scroll和WaterFlow组件修改为默认支持点击状态栏回到顶部。
- **影响接口**: 涉及接口：backToTop属性。

### 页面退出场景自定义组件删除前移
- **Kit**: -
- **原因**: 在页面退出动画的过程中，UI处于空闲状态。动画结束后，由于释放大量组件导致页面卡顿。可以将页面中自定义组件的释放提前，显著减轻卡顿并优化性能。
- **影响接口**: 自定义组件的onDisappear生命周期回调。

### V1和V2组件冻结能力增强
- **Kit**: -
- **原因**: 开发者使用组件冻结功能后，以下场景冻结功能实际未生效：支持冻结的组件嵌套使用的解冻场景、V2自定义组件的解冻场景、Repeat VirtualScroll的复用场景。
- **影响接口**: freezeWhenInactive。

### 音频框架识别USB音频设备类型行为变更
- **Kit**: -
- **原因**: 原有的USB音频设备在系统侧均识别为耳机输入/输出设备。为了提高识别的准确度，满足应用的UX显示需求，系统侧区分USB耳机和普通的USB音频设备(如音箱)。
- **影响接口**: ArkTS：@ohos.multimedia.audio.d.ts中DeviceType的USB\_HEADSET接口。

### Device Security Kit优化错误码上报
- **Kit**: -
- **原因**: Device Security Kit优化错误码类型，新增更明细的错误码。
- **影响接口**: -   checkUrlThreat
-   checkSysIntegrity

### fetch接口新增错误码返回
- **Kit**: -
- **原因**: 系统决策使用蜂窝网络发送网络请求时，如果不满足拉起蜂窝网络的条件，返回1007900986错误码。
- **影响接口**: fetch接口
