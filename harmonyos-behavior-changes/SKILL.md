---
name: harmonyos-behavior-changes
description: >
  HarmonyOS 项目升级中的行为变化适配环节。当用户说「升级后哪些接口行为变了」「升到 26 后 Toast 样式不对」「触摸热区变了」「组件默认效果变了」「targetSdkVersion 升级影响」「版本隔离是什么意思」「状态管理 V1 迁 V2」「@Component 改 @ComponentV2」「@State 改 @Local」「@Watch 改 @Monitor」「帮我升级项目，要查行为变化」时触发。
  覆盖两类内容：(1) 从 5.0.5(17) 到 26.0.0 升级路径上的接口行为变化（216 条）和 UX 样式变化（77 条）；(2) 状态管理 V1→V2 迁移指导。属于升级流程的③行为变化环节，通常由 harmonyos-upgrade 总 skill 路由调用。
version: 1.0.0
---

# 升级中的行为变化适配

## 这个 Skill 做什么

在鸿蒙项目升级时，处理两类问题：
1. **接口行为变化** — 哪些 API 的行为变了（返回值、默认效果、触发时机等）
2. **状态管理 V1→V2 迁移** — 装饰器从 V1 迁到 V2 的对照和步骤

## 两大块内容

### 块 A：接口行为变化

覆盖从 5.0.5(17) 到 26.0.0 升级路径上的全量 216 条行为变化 + 77 条 UX 变化，按版本分组，区分是否版本隔离。

每条行为变化记录包含：Kit（如 ArkUI、ArkTS、Network Kit）、变更标题、是否版本隔离、变更原因、起始 API Level、影响接口。

> 数据说明：原始 changelogs 提取到 328 条，清洗后保留 216 条（去除重复计数、目录索引页等无效数据）。其中 155 条 Kit 已明确归属（如 ArkUI、ArkTS），剩余 61 条 Kit 字段为 `-`（changelog 源文件未明确标注 Kit，但变更标题和内容完整，可按标题识别归属）。

**核心概念：版本隔离**
- 🔴 **无版本隔离** — 所有应用都受影响，**必须适配**
- 🟡 **版本隔离** — 仅 targetSdkVersion ≥ 对应版本时生效，可用 `canIUse()` 做条件分支

数据文件：
- `references/behavior-by-version/behavior-v{版本}.md` — 按版本分组的行为变化
- `references/ux-changes.md` — UX 样式变化汇总
- `references/behavior-index.md` — 全量索引

使用方法：先读 `behavior-index.md` 确定基线后要查哪些版本文件，再读对应文件。

### 块 B：状态管理 V1→V2 迁移

升级鸿蒙项目时，把状态管理从 V1 迁到 V2。**升级时必须迁移**——这是升级流程的必要环节，不是可选项。详见 migration-guide。

**入口文件**：[references/state-management/migration-guide.md](references/state-management/migration-guide.md)

包含：
- V1→V2 装饰器对照表（@State→@Local, @Watch→@Monitor 等 14 项）
- 机制差异（观测深度、@Watch 同步 vs @Monitor 异步 ⚠️ 最易踩坑）
- 迁移步骤（识别→逐组件迁移→检查时序→编译验证）
- 迁移决策（要不要迁的判断标准）

详细参考（同目录下）：
- `references/state-management/v1-v2-difference.md` — V1/V2 机制差异
- `references/state-management/overview.md` — 状态管理概述
- `references/state-management/mvvm-v1.md` / `mvvm-v2.md` — MVVM 模式对比

## 如何判断用户要哪块

| 用户问的 | 读哪个 |
|---------|--------|
| "升级后行为变了啥" / "xx接口返回值变了" | 块 A：读 behavior-by-version |
| "Toast 样式不对" / "触摸热区变了" | 块 A：读 ux-changes.md |
| "状态管理要迁吗" / "@State 怎么改" / "@Watch 改 @Monitor" | 块 B：读 state-management/migration-guide.md |
| 不确定 | 先读 state-management/migration-guide.md 顶部的"迁移决策"判断要不要迁；行为变化读 behavior-index.md |

## 使用流程

### 第 1 步：确认基线版本
同 deprecated-apis skill，从 `build-profile.json5` 读 `compatibleSdkVersion`。

### 第 2 步：读取行为变化文件
读取基线版本之后所有版本的 `behavior-v{版本}.md`。文件在 `references/behavior-by-version/` 下。

当前有数据的版本：5.0.0、5.0.3、5.1.0、6.0.0、6.1.0、26.0.0。

behavior 文件按 changelog 版本编号（如 v5_0_0、v5_1_0），与 build-profile.json5 中的 SDK 版本号不同。下表根据基线版本确定要读哪些文件：

| 基线 SDK 版本 | API Level | 需读取的 behavior 文件 |
|---|---|---|
| <= 5.0.0 | <= 12 | v5_0_0, v5_0_3, v5_1_0, v6_0_0, v6_1_0, v26_0_0 |
| 5.0.3 | 13-14 | v5_0_3, v5_1_0, v6_0_0, v6_1_0, v26_0_0 |
| 5.0.5 | 15-17 | v5_1_0, v6_0_0, v6_1_0, v26_0_0 |
| 5.1.0 | 18 | v5_1_0, v6_0_0, v6_1_0, v26_0_0 |
| 5.1.1 | 19 | v6_0_0, v6_1_0, v26_0_0 |
| 6.0.0 | 20 | v6_0_0, v6_1_0, v26_0_0 |
| 6.0.1-6.0.2 | 21-22 | v6_1_0, v26_0_0 |
| 6.1.0-6.1.1 | 23-24 | v26_0_0 |
| 26.0.0 | 26 | 无需升级 |

### 第 3 步：按优先级处理
- 先处理 🔴 无版本隔离的（必须改）
- 再处理 🟡 版本隔离的（如果 targetSdk 已升到对应版本）

### 第 4 步：UX 变化
单独查 `references/ux-changes.md`，关注组件样式/交互变化。

## 快速参考 — 各版本行为变化数量

| 版本 | 无版本隔离 | 版本隔离 | 文件 |
|------|----------|---------|------|
| 5.0.0 | 96 | 18 | behavior-v5_0_0.md（114 条） |
| 5.0.3 | 9 | 8 | behavior-v5_0_3.md（17 条） |
| 5.1.0 | 11 | 12 | behavior-v5_1_0.md（23 条） |
| 6.0.0 | 23 | 11 | behavior-v6_0_0.md（34 条） |
| 6.1.0 | 6 | 1 | behavior-v6_1_0.md（7 条） |
| 26.0.0 | 10 | 11 | behavior-v26_0_0.md（21 条） |
| **合计** | **155** | **61** | **216 条** |

完整索引见 [references/behavior-index.md](references/behavior-index.md)。

## 26.0.0 关键行为变化速查

### 🔴 无版本隔离（必须适配）
- **JSVM 内核 132→144**：Web 组件行为变化，300+ W3C 特性变更
- **async 函数类型判定修复**：`util.types().isAsyncFunction` 返回值变化
- **fastConvertToJSObject**：XML 解析保留同级 text 节点
- **鼠标 rawDeltaX/rawDeltaY**：返回值变为真实硬件数据
- **READ_IMAGEVIDEO 权限收窄**：仅本地媒体，不含云端
- **ArkUI 接口仅支持 Stage 模型**：FA 模型编译报错

### 🟡 版本隔离（targetSdkVersion ≥ 26.0.0）
- **公共事件管控**：COMMON_EVENT_PACKAGE_* 需配置 allowListenBundleChangedEvent
- **NodeAdapter onAttachToNode 回调时机提前**
- **matchParent 布局行为变更**
- **阴影模糊半径规格变更**（radius=0 从无阴影变为有阴影无模糊）

## 与 deprecated-apis skill 的关系

- **本 skill**：API 还在，但**行为变了**（返回值、默认效果、触发时机等）
- **deprecated-apis skill**：API **被废弃了**，需要找替代

一个升级任务可能两个都要查：先查废弃的（不改编译不过），再查行为变化的（不改运行不对）。

## See Also

- [references/behavior-index.md](references/behavior-index.md) — 全量索引
- [references/behavior-by-version/](references/behavior-by-version/) — 按版本分组的行为变化
- [references/ux-changes.md](references/ux-changes.md) — UX 样式变化汇总
