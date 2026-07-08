---
name: harmonyos-upgrade-detect
description: >
  HarmonyOS 项目升级的第一步：检测当前版本、判定升级路径。当用户说「帮我升级项目」「这个项目现在是什么版本」「从我的版本升到 26.0.0 要经过哪些版本」「升级路径怎么定」时触发。
  本 skill 读 build-profile.json5 / oh-package.json5 / module.json5，输出当前 API level 和到 26.0.0 的升级路径。属于升级流程的步骤1检测环节，通常由 harmonyos-upgrade 总 skill 路由调用。
version: 1.0.0
---

# 升级检测：版本基线与升级路径

## 这个 Skill 做什么

读取鸿蒙工程的配置文件，确定：
1. **当前 API level**（基线版本）
2. **到 26.0.0 的升级路径**（要经过哪些中间版本）

## 检测方法

### 1. 读 build-profile.json5

```bash
# 项目级
grep -E "compatibleSdkVersion|targetSdkVersion|compileSdkVersion" build-profile.json5

# 模块级（所有模块，模块名不一定是 entry，从 build-profile.json5 的 modules[].name 读取）
find . -name "build-profile.json5" ! -path "*/node_modules/*" ! -path "*/oh_modules/*" -exec grep -lH "targetSdkVersion\|apiType" {} \;
```

### 2. 读 oh-package.json5（依赖版本）

```bash
grep '@ohos/' oh-package.json5
```

### 3. 读 module.json5（模型类型）

```bash
find . -name "module.json5" -exec grep -E "type|srcEntry" {} \;
```
- `type: "entry"` 且有 `srcEntry` 指向 UIAbility → Stage 模型
- 无 srcEntry 或指向旧式 PageAbility → FA 模型（升级时需迁移）

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

## 输出：升级路径

检测完基线后，输出该工程的升级路径。例如基线是 `5.0.5(17)`：

```
当前: compatibleSdkVersion = 5.0.5(17), API level 17
目标: 26.0.0

升级路径（要消化的版本区间）:
  5.0.5(17) → 5.1.0(18) → 5.1.1(19) → 6.0.0(20) → 6.0.1(21)
  → 6.0.2(22) → 6.1.0(23) → 6.1.1(24) → 26.0.0

后续环节:
  2. 配置升级 → 用 harmonyos-upgrade-config
  3. 废弃API迁移 → 用 deprecated-apis
  4. 状态管理 V1→V2 → 用 harmonyos-behavior-changes
  5. 验证 → 用 harmonyos-upgrade-verify
```

## 关键判断

| 检测结果 | 处理 |
|---------|------|
| 已是 26.0.0 | 无需升级，退出 |
| FA 模型 | 警告：升级前需先迁移到 Stage 模型（API 10+ 接口要求） |
| API level < 12 | 警告：跨度大，建议先升到 5.0.0(API12) 再继续 |
| 各模块 targetSdkVersion 不一致 | 提示统一 |
| 工程含 WidgetCard（`type: "form"` 模块或 `widget/` 目录） | 提示：卡片代码不迁 V2（兼容性不好），步骤4状态管理迁移时排除卡片目录 |

## 与其他子 skill 的衔接

检测完成后，把"升级路径"结果传递给：
- `harmonyos-upgrade-config`：知道目标版本号后改配置
- `harmonyos-behavior-changes`：知道版本区间后读对应的行为变化文件
- `harmonyos-upgrade-verify`：验证时知道目标版本
