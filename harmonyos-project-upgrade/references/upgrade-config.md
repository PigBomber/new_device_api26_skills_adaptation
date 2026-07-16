# 配置文件升级

> 本文件为 `../SKILL.md` 的补充，覆盖升级流程步骤2「配置」环节：compatibleSdkVersion/targetSdkVersion 改为目标版本。当 SKILL.md 路由到配置环节时读取本文件。

## Agent Interface

```yaml
symptom_keywords:
  - api version parameter is illegal
  - compatibleSdkVersion format error
  - targetSdkVersion format wrong
  - API version mismatch across modules
  - Module not found in SDK after version bump
  - unsure which config fields to change

hard_constraints:
  - Version format depends on the upgrade path — path A (→6.1.0) uses OLD format "X.Y.Z(API N)" with parenthesis API level (e.g. "6.1.0(23)"); path B (→26.0.0) uses pure SemVer "X.Y.Z" WITHOUT parenthesis (e.g. "26.0.0"); using the wrong format for either path causes "api version parameter is illegal"
  - Both compatibleSdkVersion and targetSdkVersion must be changed to the target version — not just one
  - targetSdkVersion controls version isolation behavior — behaviors marked "targetSdk≥X" only take effect after targetSdkVersion reaches that version
  - All modules' targetSdkVersion must be unified to the same value — inconsistent versions cause "API version mismatch"
  - Do not proactively upgrade or modify ohpm dependencies (third-party libraries) — keep original versions; third-party libs are out of scope, only upgrade a specific dependency if compile reports incompatibility and the user agrees

diagnostic_checklist:
  - Which path is this? Path A → "6.1.0(23)" old format; path B → "26.0.0" pure SemVer
  - Is compatibleSdkVersion changed to the target version in the correct format?
  - Is targetSdkVersion changed to the target version in both project-level and module-level build-profile.json5?
  - Are all modules' targetSdkVersion values consistent?
  - Has oh-package.json5 been checked for @ohos/ dependency compatibility?
```

## 适配领域

本文件覆盖升级流程的第二步：把鸿蒙工程的配置文件改成目标版本，包括 build-profile.json5 版本号修改和 oh-package.json5 依赖检查。**版本号格式由路径决定**。

**不覆盖**：版本检测（见 `upgrade-detect.md`）；废弃 API 迁移（见 `deprecated-apis.md`）；行为变化适配（见 `behavior-changes.md`）；编译验证（见 `upgrade-verify.md`）。

---

## 版本号格式选择（按路径）

| 路径 | 目标版本 | 格式 | 示例 | 说明 |
|------|---------|------|------|------|
| **路径A** | 6.1.0 | 旧格式 `X.Y.Z(API N)` | `"6.1.0(23)"` | 带 API level，与该版本 SDK 要求一致 |
| **路径B** | 26.0.0 | 纯 SemVer `X.Y.Z` | `"26.0.0"` | 从 API 26 起统一 SemVer，**不带括号** |

> ⚠️ **格式用错会报 `api version parameter is illegal`**：路径A 用纯 SemVer（漏 API level）或路径B 用旧格式（带括号）都会报错。

SemVer 字段含义（仅路径B适用）：

| 版本号字段 | 含义 | 适配要求 |
|-----------|------|---------|
| X (major) | 架构级变更 | 必须适配 |
| Y (minor) | 功能新增 | 向后兼容 |
| Z (patch) | 缺陷修复 | 无风险 |

---

## 适配流程

### 新适配

**第 1 步：改项目级 build-profile.json5**

路径B（→26.0.0，纯 SemVer）：
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

路径A（→6.1.0，旧格式带 API level）：
```diff
  {
    "app": {
      "products": [
        {
          "name": "default",
-         "compatibleSdkVersion": "5.0.5(17)",
+         "compatibleSdkVersion": "6.1.0(23)",
-         "targetSdkVersion": "5.0.5(17)",
+         "targetSdkVersion": "6.1.0(23)",
        }
      ]
    }
  }
```

- `compatibleSdkVersion` 和 `targetSdkVersion` 都改成目标版本（格式按路径选）
- `targetSdkVersion` 决定版本隔离行为是否生效（路径A 触发 6.1.0 的版本隔离项；路径B 触发 26.0.0 的）

**第 2 步：改模块级 build-profile.json5**

```diff
  // 路径B（纯 SemVer）：
-   "targetSdkVersion": "5.0.0(12)"
+   "targetSdkVersion": "26.0.0"

  // 路径A（旧格式）：
-   "targetSdkVersion": "5.0.0(12)"
+   "targetSdkVersion": "6.1.0(23)"
```

如果有多个模块，所有模块的 targetSdkVersion 要保持一致（格式也一致）。

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
- [ ] 项目级 build-profile.json5 中 compatibleSdkVersion 改为目标版本（路径A=`"6.1.0(23)"`；路径B=`"26.0.0"`）
- [ ] 项目级 build-profile.json5 中 targetSdkVersion 改为目标版本（同上）
- [ ] 所有模块级 targetSdkVersion 统一为目标版本且一致
- [ ] 版本号格式正确（路径A 旧格式带括号；路径B 纯 SemVer 不带括号）

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

- `behavior-changes.md`：行为变化适配——targetSdk 升到 26 后，版本隔离的行为变化开始生效，需扫代码
- `upgrade-verify.md`：编译验证——验证配置是否正确
