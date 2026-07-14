---
name: harmonyos-upgrade-config
description: 鸿蒙项目升级的配置文件修改环节。当开发者提到「compatibleSdkVersion 怎么改」「targetSdkVersion 报错」「build-profile 版本号格式不对」「升级 SDK 版本」「oh-package 依赖要升级吗」「版本号格式错误 api version parameter is illegal」时触发。负责改 build-profile.json5 / oh-package.json5 中的版本相关字段，把项目从旧版本升到 26.0.0。属于升级流程的步骤2配置环节。不包含版本检测、废弃API迁移、编译验证。
---

## Agent Interface

```yaml
symptom_keywords:
  - api version parameter is illegal
  - compatibleSdkVersion format error with parenthesis
  - targetSdkVersion still using old X.Y.Z(API N) format
  - API version mismatch across modules
  - Module not found in SDK after version bump
  - unsure which config fields to change for 26.0.0

hard_constraints:
  - From API 26.0.0, version numbers must use SemVer X.Y.Z format — never use the old "X.Y.Z(API N)" format with parentheses
  - Both compatibleSdkVersion and targetSdkVersion must be changed to "26.0.0" — not just one
  - targetSdkVersion controls version isolation behavior — behaviors marked "targetSdk≥26" only take effect after this is set to 26.0.0
  - All modules' targetSdkVersion must be unified to the same value — inconsistent versions cause "API version mismatch"
  - Do not proactively upgrade ohpm dependencies — keep original versions, only upgrade if compile reports incompatibility

diagnostic_checklist:
  - Is compatibleSdkVersion changed to "26.0.0" (without parenthesis API level)?
  - Is targetSdkVersion changed to "26.0.0" in both project-level and module-level build-profile.json5?
  - Are all modules' targetSdkVersion values consistent?
  - Has oh-package.json5 been checked for @ohos/ dependency compatibility?
```

# 配置文件升级

## 适配领域

本 skill 覆盖升级流程的第二步：把鸿蒙工程的配置文件从旧版本格式升级到 26.0.0，包括 build-profile.json5 版本号修改和 oh-package.json5 依赖检查。

**不覆盖**：版本检测（见 `harmonyos-upgrade-detect`）；废弃 API 迁移（见 `harmonyos-deprecated-apis`）；行为变化适配（见 `harmonyos-behavior-changes`）；编译验证（见 `harmonyos-upgrade-verify`）。

---

## 版本号格式变更（SemVer）

**从 API 26.0.0 开始，版本号统一采用语义化版本（SemVer）`X.Y.Z` 格式**，不再用旧的 `X.Y.Z(API N)` 格式。

| 版本号字段 | 含义 | 适配要求 |
|-----------|------|---------|
| X (major) | 架构级变更 | 必须适配 |
| Y (minor) | 功能新增 | 向后兼容 |
| Z (patch) | 缺陷修复 | 无风险 |

---

## 适配流程

### 新适配

**第 1 步：改项目级 build-profile.json5**

```diff
  {
    "app": {
      "products": [
        {
          "name": "default",
-         "compatibleSdkVersion": "5.0.5(17)",
+         "compatibleSdkVersion": "26.0.0",
-         "targetSdkVersion": "6.0.2(22)",
+         "targetSdkVersion": "26.0.0",
        }
      ]
    }
  }
```

- `compatibleSdkVersion` 和 `targetSdkVersion` 都改成 `"26.0.0"`（纯 SemVer，不要括号里的 API level）
- `targetSdkVersion` 决定版本隔离行为是否生效（行为变化 skill 里标注"targetSdk≥26生效"的就是它控制的）

**第 2 步：改模块级 build-profile.json5**

```diff
  {
-   "targetSdkVersion": "5.0.0(12)"
+   "targetSdkVersion": "26.0.0"
  }
```

如果有多个模块，所有模块的 targetSdkVersion 要保持一致。

**第 3 步：检查 oh-package.json5**

```bash
grep '@ohos/' oh-package.json5
```

检查每个 `@ohos/` 依赖是否有 API 26 兼容版本。常见依赖：
- `@ohos/hypium`（测试框架）
- `@ohos/axios`（HTTP）

保留原版本，编译时若有不兼容报错再升级对应依赖。

**第 4 步：验证配置**

```bash
# 确认改动
grep -E "compatibleSdkVersion|targetSdkVersion" build-profile.json5
# 应输出: "compatibleSdkVersion": "26.0.0" 和 "targetSdkVersion": "26.0.0"
```

---

### 问题定位

| 报错 | 原因 | 修复 |
|-----|------|------|
| `api version parameter is illegal! Expected format: <major>[.<minor>][.<patch>]` | 版本号带了括号格式如 `"6.0.0(26)"` | 改成纯 SemVer `"26.0.0"` |
| `compatibleSdkVersion must be greater than or equal to x.x.x` | 版本号低于要求 | 确认改成 `"26.0.0"` |
| `API version mismatch` | 各模块版本不同步 | 统一所有模块的 targetSdkVersion |
| `Module xxxx not found in SDK` | 旧版 SDK 包不兼容 | 升级 ohpm 依赖 |

---

## 关键 API

### compatibleSdkVersion / targetSdkVersion

**用途**：build-profile.json5 中的版本号字段，控制 SDK 兼容级别和版本隔离行为。

**注意**：`targetSdkVersion` 决定版本隔离行为是否生效——行为变化 skill 里标注"targetSdk≥26生效"的就是它控制的。

---

## 验证清单

**基础验证（每次必做）：**
- [ ] 项目级 build-profile.json5 中 compatibleSdkVersion = `"26.0.0"`
- [ ] 项目级 build-profile.json5 中 targetSdkVersion = `"26.0.0"`
- [ ] 所有模块级 targetSdkVersion = `"26.0.0"` 且一致
- [ ] 版本号无括号格式（纯 SemVer）

**压力验证（发布前做）：**
- [ ] oh-package.json5 中 @ohos/ 依赖已检查兼容性
- [ ] grep 确认改动输出正确

---

## 常见问题

**Q：版本号报 "api version parameter is illegal"**
A：版本号带了括号格式如 `"6.0.0(26)"`。从 API 26.0.0 起版本号统一采用 SemVer `X.Y.Z` 格式，改成纯 `"26.0.0"`，不要括号里的 API level。

**Q：报 "API version mismatch"**
A：各模块的 targetSdkVersion 不同步。检查所有模块级 build-profile.json5，统一改成 `"26.0.0"`。

**Q：oh-package.json5 的依赖要主动升级吗**
A：不要主动升级。保留原版本，编译时若有不兼容报错再升级对应依赖。

---

## 延伸阅读

- `../harmonyos-behavior-changes/SKILL.md`：行为变化适配——targetSdk 升到 26 后，版本隔离的行为变化开始生效，需扫代码
- `../harmonyos-upgrade-verify/SKILL.md`：编译验证——验证配置是否正确
