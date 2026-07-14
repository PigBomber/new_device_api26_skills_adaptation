---
name: harmonyos-project-upgrade
description: 鸿蒙项目升级总入口。当开发者提到「帮我把这个项目升级到最新版本」「升级到 API 26」「项目从 5.0.5 升到 26.0.0」「鸿蒙工程升级」「compatibleSdkVersion 升级」「targetSdkVersion 升级影响」「升级后编译报错」「升级要改哪些东西」等任何涉及把现有鸿蒙项目升到更高版本的场景时触发。本 skill 是路由入口，不存业务数据，识别用户当前在升级流程的哪个环节后调用对应子 skill 处理。触发后第一件事是用 TodoWrite 创建内置 todo 清单。不包含具体的技术实现细节（在子 skill 中）。
---

## Agent Interface

```yaml
symptom_keywords:
  - upgrade HarmonyOS project to API 26
  - compatibleSdkVersion / targetSdkVersion upgrade
  - don't know what to change for upgrade
  - compile errors after upgrade
  - deprecated warnings after upgrade
  - state management V1 to V2 migration
  - behavior changes after targetSdkVersion bump

hard_constraints:
  - Must create a TodoWrite checklist immediately after triggering — never start modifying code without a checklist, or steps will be missed
  - Compilation is a hard dependency — deprecated API detection relies on compiler warnings, final verification relies on compile results; never skip compilation or abort with "no build tool found"
  - DevEco Studio includes hvigorw — if DevEco is installed, hvigorw exists; configure environment variables instead of asking user to open DevEco Studio
  - Never skip any step in the todo checklist — even if "looks like it won't affect compilation" (e.g. @Component→@ComponentV2 must still be done)
  - WidgetCard (form modules, widget/ directories) must be excluded from V1→V2 migration — @ComponentV2 compatibility is poor for cards
  - Pure V1 projects (0 @ComponentV2) must do full migration with no V1 residual — do not use dual-write bridge intermediate states

diagnostic_checklist:
  - What is the current baseline version (compatibleSdkVersion in build-profile.json5)?
  - Is hvigorw configured and runnable (DevEco Studio located)?
  - Which step of the upgrade flow is the user currently on?
  - How many V1 decorators exist in the project (determines step 4 workload)?
  - Does the project contain WidgetCard modules (must be excluded from V2 migration)?
  - Is the project FA model or Stage model (FA must migrate to Stage first)?
```

# HarmonyOS 项目升级

## 适配领域

本 skill 是升级流程的路由入口，覆盖从版本检测到编译验证的完整升级链路。它识别用户当前在升级流程的哪个环节，然后调用对应的子 skill 处理。本 skill 不存业务数据——对照表、迁移档案、错误码表都在子 skill 的 references 里。

**不覆盖**：非升级场景（如新项目开发）；单个环节的具体技术实现（在对应子 skill 中）。

---

## 升级环节矩阵

```
1. 检测  →  2. 配置  →  3. 废弃API  →  4. 状态管理V1→V2  →  5. 验证
```

| 环节 | 做什么 | 对应子 skill |
|------|--------|-------------|
| 1. 检测 | 读 build-profile.json5 定基线、统计 V1 装饰器数量、判升级路径 | `harmonyos-upgrade-detect` |
| 2. 配置 | 改 compatibleSdkVersion / targetSdkVersion → "26.0.0" | `harmonyos-upgrade-config` |
| 3. 废弃API | clean 编译捕获 deprecated → 全部迁移到 0 | `harmonyos-deprecated-apis` |
| 4. 状态管理 | V1→V2 迁移（应用级状态→数据类→跨层级→组件级→监听） | `harmonyos-behavior-changes` |
| 5. 验证 | hvigorw 编译、运行时无崩溃、deprecated=0 | `harmonyos-upgrade-verify` |

### 场景 → 子 skill 快速路由

| 用户说的话 | 调用 |
|---------------------|------|
| "帮我把这个项目升级到 26.0.0" | 创建 todo 清单，从步骤1开始 |
| "compatibleSdkVersion 怎么改" | `harmonyos-upgrade-config` |
| "deprecated 警告怎么修" | `harmonyos-deprecated-apis` |
| "@State 改 @Local" / "@Watch 改 @Monitor" | `harmonyos-behavior-changes` |
| "编译报错" / "hvigorw ERROR" | `harmonyos-upgrade-verify` |

---

## 适配流程

### 新适配

**第 0 步：配置 hvigorw 编译环境（必须完成，编译是步骤3、6的前提）**

**编译是升级流程的硬依赖**——废弃 API 检测靠编译告警、最终验证靠编译结果。**只要装了 DevEco Studio，本地就一定有 hvigorw**，配好环境变量即可。

```bash
# 1. 先查 PATH 里是否已有
command -v hvigorw 2>/dev/null && echo "hvigorw 已在 PATH" || {
  # 2. 不在 PATH，定位 DevEco Studio 安装路径（多平台）
  # macOS:
  DEVECO_HOME=$(find /Applications ~/Applications -maxdepth 2 -name "DevEco*Studio*" -type d 2>/dev/null | head -1)
  # Windows (Git Bash/WSL): 在 /c/Program\ Files/ 下找
  [ -z "$DEVECO_HOME" ] && DEVECO_HOME=$(ls -d /c/Program\ Files/Huawei/DevEco-Studio 2>/dev/null | head -1)
  # Linux:
  [ -z "$DEVECO_HOME" ] && DEVECO_HOME=$(find /opt ~/ -maxdepth 2 -name "DevEco*Studio*" -type d 2>/dev/null | head -1)

  if [ -n "$DEVECO_HOME" ]; then
    # macOS 的可执行文件在 Contents/tools/ 下
    TOOLS_DIR="$DEVECO_HOME/Contents/tools"
    SDK_DIR="$DEVECO_HOME/Contents/sdk"
    [ ! -d "$TOOLS_DIR" ] && TOOLS_DIR="$DEVECO_HOME/tools"      # Win/Linux
    [ ! -d "$SDK_DIR" ]  && SDK_DIR="$DEVECO_HOME/sdk"
    export DEVECO_SDK_HOME="$SDK_DIR"
    export PATH="$TOOLS_DIR/hvigor/bin:$TOOLS_DIR/ohpm/bin:$PATH"
    echo "已配置 hvigorw（来自 $DEVECO_HOME）"
  fi
}

# 3. 验证 hvigorw 真的能用
hvigorw --version || echo "错误：hvigorw 仍不可用，请用户提供 DevEco Studio 安装路径"
```

**如果上述都失败**（极少见）：向用户索取 DevEco Studio 安装路径后重新配置。**不允许放弃编译**。

**第 1 步：用 TodoWrite 创建内置 todo 清单**

收到升级请求后，先配置好 hvigorw 编译环境，再用 TodoWrite 创建 todo 清单（按实际工程情况填入数量），然后逐步执行。**不允许跳过任何环节**。

内置 todo 清单模板（复制到 TodoWrite）：

```
1. 检测：读 build-profile.json5 定基线版本、判升级路径
2. 配置升级：改 compatibleSdkVersion/targetSdkVersion → "26.0.0"
3. 废弃API迁移：clean 编译捕获 deprecated 告警 → 按对照表全部迁移（目标 0）
4. 状态管理 V1→V2：
   4a. 应用级状态：@StorageLink/@StorageProp/AppStorage → AppStorageV2
   4b. 数据类：@Observed/@ObjectLink → @ObservedV2/@Trace
   4c. 跨层级：@Provide/@Consume → @Provider/@Consumer
   4d. 组件级：@Component→@ComponentV2 + @State→@Local/@Param + @Prop→@Param + @Link→@Param+@Event
   4e. 监听：@Watch→@Monitor（检查同步→异步时序变化）
5. 行为变化适配：按基线版本读 changelog，适配无版本隔离的必须改项
6. 编译验证：0 ERROR + 0 deprecated + 运行时无崩溃
7. 经验沉淀：新踩的坑写回 skill
```

如何使用这个清单：
1. **先跑步骤1检测**，拿到基线版本和 V1 装饰器统计，把实际数量填进 todo
2. **逐步执行**，每完成一项标记 completed，开始下一项标记 in_progress
3. **每项完成后编译验证**，有错误立即修
4. **如果某项工程里不存在**（如没有 @StorageLink），标记 completed 并注明"工程无此项"
5. **禁止跳过**——即使"看起来不影响编译"也要执行（如 @Component→@ComponentV2）

**第 2 步：检测（必须先做）**

```bash
# 读版本
grep -E "compatibleSdkVersion|targetSdkVersion" build-profile.json5
# 统计 V1 装饰器（决定步骤4的工作量）——排除 widget/卡片目录（WidgetCard 不迁 V2）
grep -rc "@Component\b\|@State\b\|@Prop\b\|@Link\b\|@Watch\b\|@Provide\b\|@Consume\b\|@StorageLink\b\|@StorageProp\b\|@Observed\b\|@ObjectLink\b" \
  --include="*.ets" --exclude-dir=widget --exclude-dir=widgetcard --exclude-dir=oh_modules --exclude-dir=node_modules .
```

**第 3 步：配置升级**

改 build-profile.json5：`compatibleSdkVersion` 和 `targetSdkVersion` 都改成 `"26.0.0"`。

**第 4 步：废弃API迁移（铁律：deprecated 必须降到 0）**

```bash
hvigorw clean --no-daemon
hvigorw assembleHap ... 2>&1 | sed 's/\x1b\[[0-9;]*m//g' > /tmp/build.txt
TOTAL=$(grep -c "has been deprecated" /tmp/build.txt)
OH=$(grep -B1 "has been deprecated" /tmp/build.txt | grep -c "oh_modules")
echo "工程代码 deprecated: $((TOTAL-OH))（目标 0）"
```
查 `harmonyos-deprecated-apis` skill 的对照表和迁移档案，逐个迁移。**工具类的 getContext 也要改**。

**第 5 步：状态管理 V1→V2（最容易遗漏的环节）**

> **注意：排除 WidgetCard（服务卡片）**：`widget/`、`widgetcard/` 目录或 module.json5 里 `type: "form"` 的模块**不迁 V2**——WidgetCard 当前对 @ComponentV2 兼容性不好，保留 V1 装饰器。

迁移顺序（基础先行）：
1. **4a 应用级状态**：@StorageLink/@StorageProp → 定义 @ObservedV2 数据类 + AppStorageV2.connect
2. **4b 数据类**：@Observed → @ObservedV2，属性加 @Trace；@ObjectLink → @Param
3. **4c 跨层级**：@Provide → @Provider，@Consume → @Consumer
4. **4d 组件级**：@Component → @ComponentV2（叶子优先，根在后）
   - @State → @Local（纯内部）或 @Param（外部传入）
   - @Prop → @Param
   - @Link → @Param + @Event（父子双向）
5. **4e 监听**：@Watch → @Monitor（注意：同步→异步时序变化）

纯 V1 工程（0 个 @ComponentV2）的特殊策略：
- 全量迁移，不留 V1 残留（不用双写桥等中间态）
- 组件级按依赖关系排序，叶子组件先迁

详见 `harmonyos-behavior-changes` 的 `references/migration-guide.md`。

**第 6 步：行为变化适配（grep 命中驱动，只改命中的）**

详见 `harmonyos-behavior-changes`。核心规则：
1. 用 25 项变更的 grep 关键词逐项扫项目代码
2. **两步门控**：grep 命中 → 读命中行上下文二次确认符合变更描述 → 才允许改
3. grep 无命中 = 跳过，**不允许预防性修改**
4. 交付时逐项列出命中清单（命中的变更项、grep 命中数、是否实际修改及原因），让用户可核查

**第 7 步：验证**

- `hvigorw assembleHap`：0 ERROR
- deprecated 告警：工程代码 0
- 运行时：启动不崩溃（注意字段初始化不能用 UIContext）
- 多设备/UX 检查清单

---

### 问题定位

| 现象 | 可能原因 | 修复方向 |
|------|---------|---------|
| 升级后编译报错 | 废弃 API 未迁移 / 版本号格式错 / V2 迁移问题 | 查错误码（harmonyos-upgrade-verify）→ 对应子 skill |
| deprecated 告警不为 0 | 工具类 getContext 未改 / Resource 重载未改 / 第三方库告警 | 查 harmonyos-deprecated-apis 对照表，工具类也必须改 |
| 运行时崩溃 | 字段初始化阶段调用了 getUIContext() | 移到 aboutToAppear（见 harmonyos-deprecated-apis 迁移易错点） |
| V2 迁移后 UI 不刷新 | @Trace 漏加 / @Watch→@Monitor 时序变化 / ForEach key 不变断链 | 查 harmonyos-behavior-changes 迁移指南 |
| 升级遗漏环节 | 未创建 todo 清单就开始改代码 | 回到第1步，创建 todo 清单逐项执行 |

---

## 验证清单

**基础验证（每次必做）：**
- [ ] hvigorw 编译环境已配置，`hvigorw --version` 可运行
- [ ] TodoWrite 清单已创建，7 个步骤逐一标记
- [ ] 基线版本已检测，升级路径已确认
- [ ] compatibleSdkVersion 和 targetSdkVersion 改为 `"26.0.0"`

**压力验证（发布前做）：**
- [ ] `hvigorw assembleHap`：0 ERROR
- [ ] deprecated 告警：工程代码 0
- [ ] 运行时：启动不崩溃（字段初始化未用 UIContext）
- [ ] V1→V2 迁移无残留（WidgetCard 除外）
- [ ] 行为变化 grep 命中清单已逐项交付
- [ ] 多设备/UX 验证清单通过

---

## 常见问题

**Q：升级时能不能跳过某些步骤**
A：不能。todo 清单是强制的——不创建清单就开始改代码，必然遗漏环节。即使"看起来不影响编译"也要执行（如 @Component→@ComponentV2）。如果某项工程里不存在（如没有 @StorageLink），标记 completed 并注明"工程无此项"。

**Q：能不能不编译，直接用 grep 找废弃 API**
A：不能。编译器有完整类型信息和 SDK 的 @deprecated 注解，能精确发现所有废弃点（含 grep 查不到的 Resource 重载、import 别名、对照表外的新废弃）。**绝对禁止输出"本地没有构建工具，请在 DevEco Studio 中打开"然后中止**——装了 DevEco 就有 hvigorw，配好环境即可。

**Q：WidgetCard 要不要迁 V2**
A：不要。WidgetCard（服务卡片）当前对 @ComponentV2 兼容性不好，保留 V1 装饰器。`widget/`、`widgetcard/` 目录或 module.json5 里 `type: "form"` 的模块，在统计 V1 装饰器数量和迁移时都要排除。

**Q：子 skill 可以独立触发吗**
A：可以。本 skill 是路由入口，但子 skill 也可独立触发。用户只问行为变化，直接调 `harmonyos-behavior-changes`；只问编译报错，直接调 `harmonyos-upgrade-verify`。

---

## 延伸阅读

- `../harmonyos-upgrade-detect/SKILL.md`：版本检测与升级路径——步骤1检测环节
- `../harmonyos-upgrade-config/SKILL.md`：配置文件升级——步骤2配置环节
- `../harmonyos-deprecated-apis/SKILL.md`：废弃 API 检查与替换——步骤3废弃API迁移环节
- `../harmonyos-behavior-changes/SKILL.md`：行为变化适配 + 状态管理迁移——步骤4状态管理 + 步骤5行为变化环节
- `../harmonyos-upgrade-verify/SKILL.md`：编译与多设备验证——步骤6验证环节
