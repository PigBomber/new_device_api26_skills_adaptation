# 行为变化 — 版本 6.1.0（共 7 条）

> 数据来源：`changelogs-for-all-apps-*.md`
> 「版本隔离=是」表示 targetSdkVersion ≥ 对应版本才生效；「否」表示所有应用都受影响。


## 🔴 无版本隔离（必须适配，6 条）

### 横屏悬浮窗尺寸优化
- **Kit**: ArkUI
- **原因**: 应用横屏布局尺寸按横屏屏幕大小布局，旧悬浮窗尺寸过小导致要额外适配显示布局。
- **影响接口**: 不涉及

### 新增状态管理V1校验报错
- **Kit**: ArkUI
- **起始 API Level**: 不涉及
- **原因**: 自定义组件struct中使用状态管理V1的装饰器装饰Function类型的变量时，会出现运行时crash异常，提示开发者不支持装饰Function类型。为提升开发者运行时代码的稳定性，新增编译报错：如果自定义组件struct中使用状态管理V1的装饰器装饰Function类型的变量，则编译报错并中断编译。
- **影响接口**: ArkUI状态管理V1装饰器：

### WordBreak.HYPHENATION移除以下六种语言连字符“-”断词换行能力的支持：捷克语、印度尼西亚语、拉脱维亚语、马其顿语、斯洛伐克语、塞尔维亚语
- **Kit**: ArkGraphics 2D（该变更同时涉及ArkUI）
- **起始 API Level**: API 18
- **原因**: Hyphen库将移除部分不兼容语种，这些语种的连字符“-”进行断词换行能力将会失效。
- **影响接口**: -   ArkUI：文本组件text.d.ts文件TextAttribute中wordBreak属性对应属性值：WordBreak.HYPHENATION
-   ArkGraphics 2D：
    -   ArkTS API：@ohos.graphics.text.d.ts文件interface ParagraphStyle中wordBreak属性对应属性值：WordBreak.BREAK\_HYPHEN
    -   C API：drawing\_text\_typography.h文件enum OH\_Drawing\_WordBreakType中类型：WORD\_BREAK\_T

### 全球化接口与运行时接口本地化显示优化及换行能力增强
- **Kit**: Localization Kit
- **原因**: ICU版本已从72.1升级至74.2，本次升级基于[ICU 74.2数据](https://icu.unicode.org/download/74)，优化了本地化表达习惯，使部分全球化接口及运行时接口的行为更贴合各地区的语言使用规范，同时提升了系统的文本换行处理能力。
- **影响接口**: | 接口 | 变更场景举例 | 变更前返回值 | 变更后返回值 |
| --- | --- | --- | --- |
| [intl.DateTimeFormat.format](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-intl#formatdeprecated) | 繁体中文下格式化2024年4月23日 | 2024年4月23日 | 2024 年 4 月 23 日 |
| [intl.DateTimeFormat.formatRange](https://developer.huaw

### 动态照片资源的PhotoKeys.DURATION字段值变更
- **Kit**: Media Library Kit
- **起始 API Level**: API 14
- **原因**: 图库应用需获取动态照片中附带视频片段的时长，并根据该时长判断是否支持生成3D动态照片。因此动态照片PhotoKeys.DURATION字段的取值由图片资源的默认值0变更为动态照片附带视频片段的时长。
- **影响接口**: PhotoKeys.DURATION

### JS OOM异常崩溃栈形式变更
- **Kit**: Performance Analysis Kit
- **起始 API Level**: API 12
- **原因**: 变更前应用发生JS OOM异常时，既可能产生JS Crash，也可能产生CPP Crash，影响三方生态应用对OOM异常的数据统计。
- **影响接口**: 不涉及


## 🟡 版本隔离（targetSdkVersion ≥ 6.1.0 生效，1 条）

### Progress组件color属性设置渐变色规格变更
- **Kit**: -
- **原因**: Progress的渐变色能力增强，新增支持通过color设置Linear和Capsule进度条前景色为渐变色。
- **影响接口**: Progress组件的color属性。
