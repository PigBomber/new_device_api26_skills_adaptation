---
name: harmonyos-upgrade
description: >
  HarmonyOS 项目升级总入口。当用户提到「帮我把这个项目升级到最新版本」「升级到 API 26」「项目从 5.0.5 升到 26.0.0」「鸿蒙工程升级」「compatibleSdkVersion 升级」「targetSdkVersion 升级影响」「升级后编译报错」「升级要改哪些东西」等任何涉及"把现有鸿蒙项目升到更高版本"的场景时触发。
  本 skill 是路由入口，不存业务数据。它识别用户当前在升级流程的哪个环节，然后调用对应的子 skill 处理。
  **触发后第一件事：用 TodoWrite 工具创建内置 todo 清单（见下方模板），严格按清单执行，不遗漏任何环节。**
version: 2.0.0
---

# HarmonyOS 项目升级

## ⚠️ 触发后第一件事：创建 todo 清单

**收到升级请求后，必须先用 TodoWrite 工具创建以下 todo 清单**（按实际工程情况填入数量），然后逐步执行。**不允许跳过任何环节**。

### 内置 todo 清单模板（复制到 TodoWrite）

```
①检测：读 build-profile.json5 定基线版本、判升级路径
②配置升级：改 compatibleSdkVersion/targetSdkVersion → "26.0.0"
③废弃API迁移：clean 编译捕获 deprecated 告警 → 全部迁移（目标 0）
④状态管理 V1→V2：
   4a. 应用级状态：@StorageLink/@StorageProp/AppStorage → AppStorageV2
   4b. 数据类：@Observed/@ObjectLink → @ObservedV2/@Trace
   4c. 跨层级：@Provide/@Consume → @Provider/@Consumer
   4d. 组件级：@Component→@ComponentV2 + @State→@Local/@Param + @Prop→@Param + @Link→@Param+@Event
   4e. 监听：@Watch→@Monitor（检查同步→异步时序变化）
⑤行为变化适配：按基线版本读 changelog，适配无版本隔离的必须改项
⑥编译验证：0 ERROR + 0 deprecated + 运行时无崩溃
⑦经验沉淀：新踩的坑写回 skill
```

### 如何使用这个清单

1. **先跑①检测**，拿到基线版本和 V1 装饰器统计，把实际数量填进 todo
2. **逐步执行**，每完成一项标记 completed，开始下一项标记 in_progress
3. **每项完成后编译验证**，有错误立即修
4. **如果某项工程里不存在**（如没有 @StorageLink），标记 completed 并注明"工程无此项"
5. **禁止跳过**——即使"看起来不影响编译"也要执行（如 @Component→@ComponentV2）

---

## 升级流程的 5 个环节

```
①检测  →  ②配置  →  ③废弃API  →  ④状态管理V1→V2  →  ⑤验证
```

| 环节 | 做什么 | 对应子 skill |
|------|--------|-------------|
| ① 检测 | 读 build-profile.json5 定基线、统计 V1 装饰器数量、判升级路径 | `harmonyos-upgrade-detect` |
| ② 配置 | 改 compatibleSdkVersion / targetSdkVersion → "26.0.0" | `harmonyos-upgrade-config` |
| ③ 废弃API | clean 编译捕获 deprecated → 全部迁移到 0 | `deprecated-apis` |
| ④ 状态管理 | V1→V2 迁移（应用级状态→数据类→跨层级→组件级→监听） | `harmonyos-behavior-changes` |
| ⑤ 验证 | hvigorw 编译、运行时无崩溃、deprecated=0 | `harmonyos-upgrade-verify` |

## 各环节执行要点

### ①检测（必须先做）
```bash
# 读版本
grep -E "compatibleSdkVersion|targetSdkVersion" build-profile.json5
# 统计 V1 装饰器（决定 ④ 的工作量）
grep -rc "@Component\b\|@State\b\|@Prop\b\|@Link\b\|@Watch\b\|@Provide\b\|@Consume\b\|@StorageLink\b\|@StorageProp\b\|@Observed\b\|@ObjectLink\b" --include="*.ets" .
```

### ②配置升级
改 build-profile.json5：`compatibleSdkVersion` 和 `targetSdkVersion` 都改成 `"26.0.0"`。

### ③废弃API迁移（铁律：deprecated 必须降到 0）
```bash
hvigorw clean --no-daemon
hvigorw assembleHap ... 2>&1 | sed 's/\x1b\[[0-9;]*m//g' > /tmp/build.txt
TOTAL=$(grep -c "has been deprecated" /tmp/build.txt)
OH=$(grep -B1 "has been deprecated" /tmp/build.txt | grep -c "oh_modules")
echo "工程代码 deprecated: $((TOTAL-OH))（目标 0）"
```
查 `deprecated-apis` skill 的对照表和迁移档案，逐个迁移。**工具类的 getContext 也要改**。

### ④状态管理 V1→V2（最容易遗漏的环节）

**迁移顺序（基础先行）：**
1. **4a 应用级状态**：@StorageLink/@StorageProp → 定义 @ObservedV2 数据类 + AppStorageV2.connect
2. **4b 数据类**：@Observed → @ObservedV2，属性加 @Trace；@ObjectLink → @Param
3. **4c 跨层级**：@Provide → @Provider，@Consume → @Consumer
4. **4d 组件级**：@Component → @ComponentV2（叶子优先，根在后）
   - @State → @Local（纯内部）或 @Param（外部传入）
   - @Prop → @Param
   - @Link → @Param + @Event（父子双向）
5. **4e 监听**：@Watch → @Monitor（⚠️ 同步→异步时序变化）

**纯 V1 工程（0 个 @ComponentV2）的特殊策略：**
- 应用级状态用**双写桥模式**（写入端同时写 V1 AppStorage 和 V2 AppStorageV2），确保未迁的 V1 组件仍能响应
- 组件级按依赖关系排序，叶子组件先迁

详见 `harmonyos-behavior-changes` 的 `state-management/migration-guide.md`。

### ⑤验证
- `hvigorw assembleHap`：0 ERROR
- deprecated 告警：工程代码 0
- 运行时：启动不崩溃（注意字段初始化不能用 UIContext）
- 多设备/UX 检查清单

## 场景 → 子 skill 快速路由

| 用户说的话 | 调用 |
|---------------------|------|
| "帮我把这个项目升级到 26.0.0" | 创建 todo 清单，从①开始 |
| "compatibleSdkVersion 怎么改" | `harmonyos-upgrade-config` |
| "deprecated 警告怎么修" | `deprecated-apis` |
| "@State 改 @Local" / "@Watch 改 @Monitor" | `harmonyos-behavior-changes` |
| "编译报错" / "hvigorw ERROR" | `harmonyos-upgrade-verify` |

## 注意事项

- **本 skill 不存业务数据**。对照表、迁移档案、错误码表都在子 skill 的 references 里。
- **子 skill 也可独立触发**。用户只问行为变化，直接调 `harmonyos-behavior-changes`。
- **todo 清单是强制的**——不创建清单就开始改代码，必然遗漏环节。

## See Also

- [harmonyos-upgrade-detect](../harmonyos-upgrade-detect/SKILL.md) — 版本检测与升级路径
- [harmonyos-upgrade-config](../harmonyos-upgrade-config/SKILL.md) — 配置文件升级
- [deprecated-apis](../deprecated-apis/SKILL.md) — 废弃 API 检查与替换
- [harmonyos-behavior-changes](../harmonyos-behavior-changes/SKILL.md) — 行为变化适配 + 状态管理迁移
- [harmonyos-upgrade-verify](../harmonyos-upgrade-verify/SKILL.md) — 编译与多设备验证
