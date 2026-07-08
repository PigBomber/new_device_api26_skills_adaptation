---
name: harmonyos-behavior-changes
description: >
  HarmonyOS 项目升级中的行为变化适配 + 状态管理 V1→V2 迁移环节。当用户说「升级后哪些接口行为变了」「升到 26 后 Toast 样式不对」「触摸热区变了」「组件默认效果变了」「targetSdkVersion 升级影响」「版本隔离是什么意思」「状态管理 V1 迁 V2」「@Component 改 @ComponentV2」「@State 改 @Local」「@Watch 改 @Monitor」时触发。
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

### 块 B：接口行为变化（速查）

升级到 API 26 后，以下行为变化需要注意。**只有 🔴 无版本隔离的必须适配代码，🟡 版本隔离的只需测试确认效果。**

#### 🔴 26.0.0 无版本隔离（必须适配）

| 行为变化 | 影响 | 适配方式 |
|---------|------|---------|
| **JSVM/ArkWeb 内核 132→144** | Web 组件行为变化，300+ W3C 特性变更 | 查 ArkWeb 差异总结，逐项核对 Web 页面 |
| **async 函数类型判定修复** | `util.types().isAsyncFunction` / `Function.constructor.name` 返回值变化 | 排查依赖 async 类型判断的代码 |
| **fastConvertToJSObject** | XML 解析保留同级 text 节点（之前丢失） | 检查 XML 解析结果是否依赖旧的丢失行为 |
| **鼠标 rawDeltaX/rawDeltaY** | 返回值从"原始数据/缩放比例"变为真实硬件数据 | 如需旧行为用 `px2vp()` 转换 |
| **READ_IMAGEVIDEO 权限收窄** | 只能读本地图片/视频，不含云端 | 如需云端资源用 PhotoViewPicker |
| **ArkUI 接口仅支持 Stage 模型** | FA 模型编译报错 | 确认工程已是 Stage 模型 |

#### 🟡 26.0.0 版本隔离（targetSdkVersion ≥ 26 才生效，需测试确认）

| 行为变化 | 影响 | 测试要点 |
|---------|------|---------|
| 公共事件管控增强 | COMMON_EVENT_PACKAGE_* 需配置 | 如监听了这些事件，确认权限 |
| 触摸热区 28→32vp | Button/Toggle/Select/Chip 点击区域变大 | 确认布局不被 4vp 增幅撑破 |
| Dialog/Toast 沉浸式材质 | 默认开启系统材质效果 | 确认弹窗视觉效果可接受 |
| 内置文本样式优化 | 孤字换行/小语种行高/音节换行 | 确认文本布局正常 |
| NodeAdapter onAttachToNode 时机 | 回调从"上树时"变为"绑定时"（更早） | 在 attachNodeAdapter 前设回调 |
| matchParent 布局 | 单方向 matchParent 参与父组件尺寸计算 | 确认布局不被撑破 |
| 阴影模糊半径 | radius=0 从无阴影变为有阴影无模糊 | 确认阴影效果 |

> 以上行为变化来自华为官方 changelogs-for-all-apps-7001 和 changelogs-ux-7001。

## 如何判断用户要哪块

| 用户问的 | 读哪个 |
|---------|--------|
| "@State 怎么改" / "@Watch 改 @Monitor" / "@Component 改 V2" | 块 A：读 migration-guide.md |
| "升级后行为变了啥" / "Toast 样式不对" | 块 B：看上方速查表 |
| 不确定 | 先看块 B 速查表是否有命中；状态管理看 migration-guide.md |

## 与 harmonyos-deprecated-apis skill 的关系

| | 本 skill | harmonyos-deprecated-apis |
|---|---|---|
| 管什么 | API **还在**，但行为变了；V1 装饰器迁 V2 | API **废弃了**，要换替代 |
| 怎么发现 | 查行为变化速查表；V1 装饰器 grep 统计 | 编译 deprecated 告警 |

## See Also

- [references/state-management/migration-guide.md](references/state-management/migration-guide.md) — V1→V2 迁移指南（20项对照+5步流程+踩坑记录）
