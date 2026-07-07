---
name: harmonyos-upgrade-config
description: >
  HarmonyOS 项目升级的配置文件修改环节。当用户说「compatibleSdkVersion 怎么改」「targetSdkVersion 报错」「build-profile 版本号格式不对」「升级 SDK 版本」「oh-package 依赖要升级吗」「版本号格式错误 api version parameter is illegal」时触发。
  本 skill 负责改 build-profile.json5 / oh-package.json5 / module.json5 中的版本相关字段，把项目从旧版本升到 26.0.0。属于升级流程的②配置环节，通常由 harmonyos-upgrade 总 skill 路由调用。
version: 1.0.0
---

# 配置文件升级

## 这个 Skill 做什么

把鸿蒙工程的配置文件从旧版本格式升级到 26.0.0。

## 核心变更：版本号格式（SemVer）

**从 API 26.0.0 开始，版本号统一采用语义化版本（SemVer）`X.Y.Z` 格式**，不再用旧的 `X.Y.Z(API N)` 格式。

| 版本号字段 | 含义 | 适配要求 |
|-----------|------|---------|
| X (major) | 架构级变更 | 必须适配 |
| Y (minor) | 功能新增 | 向后兼容 |
| Z (patch) | 缺陷修复 | 无风险 |

## build-profile.json5 修改

### 项目级 build-profile.json5

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

**关键**：
- `compatibleSdkVersion` 和 `targetSdkVersion` 都改成 `"26.0.0"`（纯 SemVer，不要括号里的 API level）
- `targetSdkVersion` 决定版本隔离行为是否生效（行为变化 skill 里标注"targetSdk≥26生效"的就是它控制的）

### 模块级 build-profile.json5

```diff
  {
-   "targetSdkVersion": "5.0.0(12)"
+   "targetSdkVersion": "26.0.0"
  }
```

如果有多个模块，所有模块的 targetSdkVersion 要保持一致。

## oh-package.json5 检查

```bash
grep '@ohos/' oh-package.json5
```

检查每个 `@ohos/` 依赖是否有 API 26 兼容版本。常见依赖：
- `@ohos/hypium`（测试框架）
- `@ohos/axios`（HTTP）

保留原版本，编译时若有不兼容报错再升级对应依赖。

## 常见报错与修复

| 报错 | 原因 | 修复 |
|-----|------|------|
| `api version parameter is illegal! Expected format: <major>[.<minor>][.<patch>]` | 版本号带了括号格式如 `"6.0.0(26)"` | 改成纯 SemVer `"26.0.0"` |
| `compatibleSdkVersion must be greater than or equal to x.x.x` | 版本号低于要求 | 确认改成 `"26.0.0"` |
| `API version mismatch` | 各模块版本不同步 | 统一所有模块的 targetSdkVersion |
| `Module xxxx not found in SDK` | 旧版 SDK 包不兼容 | 升级 ohpm 依赖 |

## 验证配置改对了

```bash
# 确认改动
grep -E "compatibleSdkVersion|targetSdkVersion" build-profile.json5
# 应输出: "compatibleSdkVersion": "26.0.0" 和 "targetSdkVersion": "26.0.0"
```

## 与其他子 skill 的衔接

配置改完后，进入：
- `harmonyos-behavior-changes`：targetSdk 升到 26 后，版本隔离的行为变化开始生效，需扫代码
- `harmonyos-upgrade-verify`：编译验证配置是否正确
