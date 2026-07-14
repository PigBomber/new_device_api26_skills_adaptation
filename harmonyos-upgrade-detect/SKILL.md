---
name: harmonyos-upgrade-detect
description: 鸿蒙项目升级的第一步：检测当前版本与升级路径。当开发者提到「帮我升级项目」「这个项目现在是什么版本」「从我的版本升到 26.0.0 要经过哪些版本」「升级路径怎么定」「compatibleSdkVersion 是多少」「FA 模型还是 Stage 模型」时触发。读取 build-profile.json5 / oh-package.json5 / module.json5，输出当前 API level 和到 26.0.0 的升级路径。属于升级流程的步骤1检测环节。不包含配置修改、废弃API迁移、行为变化适配。
---

## Agent Interface

```yaml
symptom_keywords:
  - don't know current project API level
  - don't know upgrade path to 26.0.0
  - unsure if project is FA or Stage model
  - module targetSdkVersion values inconsistent across modules
  - project contains WidgetCard (form module) — need to know V2 migration exclusion
  - API level too low, large version gap

hard_constraints:
  - Read compatibleSdkVersion and targetSdkVersion from build-profile.json5, not guess — module name is in modules[].name and is not always "entry"
  - FA model projects must migrate to Stage model before upgrading — FA-specific APIs are deprecated at API 10+
  - WidgetCard (form modules, widget/ or widgetcard/ directories) must be excluded from V1→V2 migration — @ComponentV2 compatibility is poor for cards
  - API level < 12 projects should upgrade to 5.0.0(API12) first before continuing — large jumps are risky
  - All modules' targetSdkVersion must be unified to the same value before proceeding to config step

diagnostic_checklist:
  - What is the current compatibleSdkVersion in build-profile.json5?
  - What is the current targetSdkVersion (project-level and per-module)?
  - Is the project FA model or Stage model (check module.json5 for srcEntry pointing to UIAbility)?
  - Does the project contain WidgetCard modules (type: "form" or widget/ directories)?
  - What is the version interval to digest from baseline to 26.0.0?
```

# 升级检测：版本基线与升级路径

## 适配领域

本 skill 覆盖升级流程的第一步：读取鸿蒙工程的配置文件，确定当前 API level（基线版本）和到 26.0.0 的升级路径。

**不覆盖**：配置文件修改（见 `harmonyos-upgrade-config`）；废弃 API 迁移（见 `harmonyos-deprecated-apis`）；行为变化适配与状态管理迁移（见 `harmonyos-behavior-changes`）；编译验证（见 `harmonyos-upgrade-verify`）。

---

## 版本号对照表

| SDK 版本号 | API Level | 说明 |
|-----------|-----------|------|
| 5.0.0 | 12 | |
| 5.0.5 | 17 | |
| 5.1.0 | 18 | |
| 5.1.1 | 19 | |
| 6.0.0 | 20 | |
| 6.0.1 | 21 | |
| 6.0.2 | 22 | |
| 6.1.0 | 23 | |
| 6.1.1 | 24 | |
| 26.0.0 | 26 | 版本号改为 SemVer 格式 |

---

## 适配流程

### 新适配

**第 1 步：读 build-profile.json5**

```bash
# 项目级
grep -E "compatibleSdkVersion|targetSdkVersion|compileSdkVersion" build-profile.json5

# 模块级（所有模块，模块名不一定是 entry，从 build-profile.json5 的 modules[].name 读取）
find . -name "build-profile.json5" ! -path "*/node_modules/*" ! -path "*/oh_modules/*" -exec grep -lH "targetSdkVersion\|apiType" {} \;
```

**第 2 步：读 oh-package.json5（依赖版本）**

```bash
grep '@ohos/' oh-package.json5
```

**第 3 步：读 module.json5（模型类型）**

```bash
find . -name "module.json5" -exec grep -E "type|srcEntry" {} \;
```

- `type: "entry"` 且有 `srcEntry` 指向 UIAbility → Stage 模型
- 无 srcEntry 或指向旧式 PageAbility → FA 模型（升级时需迁移）

**第 4 步：输出升级路径**

检测完基线后，输出该工程的升级路径。例如基线是 `5.0.5(17)`：

```
当前: compatibleSdkVersion = 5.0.5(17), API level 17
目标: 26.0.0

升级路径（要消化的版本区间）:
  5.0.5(17) → 5.1.0(18) → 5.1.1(19) → 6.0.0(20) → 6.0.1(21)
  → 6.0.2(22) → 6.1.0(23) → 6.1.1(24) → 26.0.0

后续环节:
  2. 配置升级 → 用 harmonyos-upgrade-config
  3. 废弃API迁移 → 用 harmonyos-deprecated-apis
  4. 状态管理 V1→V2 → 用 harmonyos-behavior-changes
  5. 验证 → 用 harmonyos-upgrade-verify
```

---

### 问题定位

| 检测结果 | 处理 |
|---------|------|
| 已是 26.0.0 | 无需升级，退出 |
| FA 模型 | 警告：升级前需先迁移到 Stage 模型（API 10+ 接口要求） |
| API level < 12 | 警告：跨度大，建议先升到 5.0.0(API12) 再继续 |
| 各模块 targetSdkVersion 不一致 | 提示统一 |
| 工程含 WidgetCard（`type: "form"` 模块或 `widget/` 目录） | 提示：卡片代码不迁 V2（兼容性不好），步骤4状态管理迁移时排除卡片目录 |

---

## 关键 API

### build-profile.json5

**用途**：读取 `compatibleSdkVersion` / `targetSdkVersion` / `compileSdkVersion` 确定基线版本。

**注意**：模块名从 `build-profile.json5` 的 `modules[].name` 读取，不一定是 `entry`。

### module.json5

**用途**：判断 FA 模型还是 Stage 模型。

**注意**：`type: "entry"` 且有 `srcEntry` 指向 UIAbility → Stage 模型；无 srcEntry 或指向旧式 PageAbility → FA 模型。

---

## 验证清单

**基础验证（每次必做）：**
- [ ] 已读 build-profile.json5 确认 compatibleSdkVersion 和 targetSdkVersion
- [ ] 已读 module.json5 确认模型类型（FA/Stage）
- [ ] 已检查是否有 WidgetCard 模块（type: "form" 或 widget/ 目录）
- [ ] 各模块 targetSdkVersion 已确认是否一致

**压力验证（发布前做）：**
- [ ] 升级路径已列出要消化的完整版本区间
- [ ] FA 模型工程已警告需先迁移 Stage 模型
- [ ] API level < 12 的工程已建议先升到 5.0.0(API12)

---

## 常见问题

**Q：项目是 FA 模型，能直接升级到 26.0.0 吗**
A：不能直接升。FA 模型特有的 API（PageAbility 相关）在 API 10+ 已标记废弃，升级到 26.0.0 时 ArkUI 接口新增了"仅支持 Stage 模型"的约束。必须先迁移到 Stage 模型再升级。

**Q：各模块 targetSdkVersion 不一致怎么办**
A：提示用户统一所有模块的 targetSdkVersion 到同一个值。在配置升级环节（harmonyos-upgrade-config）统一改为 `"26.0.0"`。

**Q：工程含 WidgetCard，状态管理迁移时怎么处理**
A：WidgetCard（服务卡片）当前对 @ComponentV2 兼容性不好，不迁 V2。检测时识别 `type: "form"` 模块或 `widget/`/`widgetcard/` 目录，在后续步骤4状态管理迁移时排除这些目录，保留 V1 装饰器。

---

## 延伸阅读

- `../harmonyos-upgrade-config/SKILL.md`：配置文件升级——知道目标版本号后改配置
- `../harmonyos-deprecated-apis/SKILL.md`：废弃 API 检查与替换——知道版本区间后查对应废弃项
- `../harmonyos-behavior-changes/SKILL.md`：行为变化适配 + 状态管理迁移——知道版本区间后读对应行为变化
- `../harmonyos-upgrade-verify/SKILL.md`：编译验证与多设备检查——验证时知道目标版本
