---
name: harmonyos-project-upgrade
description: 鸿蒙项目升级总入口。当开发者提到「帮我把这个项目升级到最新版本」「升级到 API 26」「项目从 5.0.5 升到 26.0.0」「升级到 6.1」「升到 6.1.0」「鸿蒙工程升级」「compatibleSdkVersion 升级」「targetSdkVersion 升级影响」「升级后编译报错」「升级要改哪些东西」等任何涉及把现有鸿蒙项目升到更高版本的场景时触发。支持两条固定路径：升到 6.1.0(API23) 或升到 26.0.0(API26)，按用户指定的目标版本走对应路径。本 skill 是路由入口，不存业务数据，识别用户当前在升级流程的哪个环节后读取对应 reference 处理。触发后第一件事是用 TodoWrite 创建内置 todo 清单。不包含具体的技术实现细节（在 references 中）。
---

## Agent Interface

```yaml
symptom_keywords:
  - upgrade HarmonyOS project to 26.0.0 (API26)
  - upgrade HarmonyOS project to 6.1.0 (API23)
  - compatibleSdkVersion / targetSdkVersion upgrade
  - don't know what to change for upgrade
  - compile errors after upgrade
  - deprecated warnings after upgrade
  - state management V1 to V2 migration (only for path B → 26.0.0)
  - behavior changes after targetSdkVersion bump

hard_constraints:
  - Two fixed upgrade paths ONLY — path A (→6.1.0/API23) or path B (→26.0.0/API26); determine which by the user's stated target version; never invent other target versions
  - Must create a TodoWrite checklist immediately after triggering — never start modifying code without a checklist, or steps will be missed
  - SDK version is a hard prerequisite checked at step 0 — local SDK apiVersion must be ≥ the target path's API level (path A needs ≥23, path B needs ≥26); if insufficient, STOP and tell the user to upgrade DevEco Studio/SDK first, never proceed to config changes (config will be set to a version the SDK can't compile → "SDK component missing" → whole flow stalls)
  - Compilation is a hard dependency — deprecated API detection relies on compiler warnings, final verification relies on compile results; never skip compilation or abort with "no build tool found"
  - DevEco Studio includes hvigorw — if DevEco is installed, hvigorw exists; configure environment variables instead of asking user to open DevEco Studio
  - Never skip any step in the todo checklist — even if "looks like it won't affect compilation" (e.g. @Component→@ComponentV2 must still be done)
  - WidgetCard (form modules, widget/ directories) must be excluded from V1→V2 migration — @ComponentV2 compatibility is poor for cards
  - Pure V1 projects (0 @ComponentV2) must do full migration with no V1 residual — do not use dual-write bridge intermediate states

diagnostic_checklist:
  - What is the current baseline version (compatibleSdkVersion in build-profile.json5)?
  - Is hvigorw configured and runnable (DevEco Studio located)?
  - Is the local SDK version sufficient for the target path (apiVersion ≥ 23 for path A, ≥ 26 for path B, in sdk-pkg.json)? If not, STOP and tell the user to upgrade DevEco Studio/SDK first — otherwise compile will fail with "SDK component missing"
  - Which step of the upgrade flow is the user currently on?
  - How many V1 decorators exist in the project (determines step 4 workload)?
  - Does the project contain WidgetCard modules (must be excluded from V2 migration)?
  - Is the project FA model or Stage model (FA must migrate to Stage first)?
```

# HarmonyOS 项目升级

## 适配领域

本 skill 是升级流程的路由入口，覆盖从版本检测到编译验证的完整升级链路。识别用户当前在升级流程的哪个环节后，读取对应 reference 处理。不存业务数据——对照表、迁移档案、错误码表都在 references 里。

**不覆盖**：非升级场景（如新项目开发）；单个环节的具体技术实现（在对应 reference 中）。

---

## 路径选择（第一步必做）

收到升级请求后，**先确定走哪条路径**（不关心当前基线版本，只看目标版本）：

| 用户要升到 | 路径 | 版本号格式 | 废弃API | 行为变化 | V2状态管理迁移 | SDK 要求 |
|-----------|------|----------|---------|---------|--------------|---------|
| **6.1.0** | **路径A** | 旧格式 `6.1.0(23)` | since ≤ 23 | 查 6.0.0 + 6.1.0 速查表（累积） | **不做** | apiVersion ≥ 23 |
| **26.0.0** | **路径B** | 纯 SemVer `26.0.0` | since ≤ 26（全部） | 查 6.0.0 + 6.1.0 + 26.0.0 速查表（累积） | **必做** | apiVersion ≥ 26 |

> 用户说"升到最新版"= 路径B（当前最新是 26.0.0）。用户说"升到 6.1"= 路径A。两条路径的流程骨架相同（检测→配置→废弃API→行为变化→验证），差异在每个环节的数据/参数，以及路径B 多一个 V2 迁移环节。

---

## 升级环节矩阵

```
1. 检测  →  2. 配置  →  3. 废弃API  →  4. 状态管理V1→V2(仅路径B)  →  5. 行为变化  →  6. 验证
```

| 环节 | 做什么 | 路径A（→6.1.0） | 路径B（→26.0.0） | 下一步（Read） |
|------|--------|---------------|-----------------|-------------|
| 1. 检测 | 读 build-profile.json5 定基线、统计 V1 装饰器数量 | 同 | 同 | `references/upgrade-detect.md` |
| 2. 配置 | 改 compatibleSdkVersion / targetSdkVersion | `6.1.0(23)` 旧格式 | `26.0.0` SemVer | `references/upgrade-config.md` |
| 3. 废弃API | clean 编译捕获 deprecated → 迁移到 0 | since ≤ 23 的 | since ≤ 26（全部） | `references/deprecated-apis.md` |
| 4. 状态管理 | V1→V2 迁移 | **跳过**（不做） | 必做 | `references/behavior-changes.md` 块A |
| 5. 行为变化 | grep 扫项目代码，命中的才改 | 查 6.0+6.1 表 | 查 6.0+6.1+26 表 | `references/behavior-changes.md` 块B |
| 6. 验证 | hvigorw 编译、运行时无崩溃、deprecated=0 | 同（行为变化验证项不同） | 同 | `references/upgrade-verify.md` |

### 场景 → reference 快速路由

| 用户说的话 | Read |
|---------------------|------|
| "帮我把这个项目升级到 26.0.0" / "升到最新版" | **路径B**：创建 todo 清单，从步骤0开始（先检查 SDK≥26） |
| "帮我把这个项目升级到 6.1" / "升到 6.1.0" | **路径A**：创建 todo 清单，从步骤0开始（先检查 SDK≥23），跳过步骤4 V2迁移 |
| "升级到最新版本" / "升到最新版" | 先确认「最新版」对应什么（当前是 26.0.0/API26），再从步骤0开始检查 SDK 是否够；**SDK 不够先提醒升级 DevEco Studio，别急着改配置** |
| "compatibleSdkVersion 怎么改" | `references/upgrade-config.md` |
| "deprecated 警告怎么修" | `references/deprecated-apis.md` |
| "@State 改 @Local" / "@Watch 改 @Monitor" | `references/behavior-changes.md` |
| "编译报错" / "hvigorw ERROR" | `references/upgrade-verify.md` |
| "SDK component missing" / "找不到目标版本接口" | 本地 SDK 版本不足（apiVersion 低于路径要求）——提醒用户升级 DevEco Studio/SDK（见步骤0） |

---

## 适配流程

### 新适配

**第 0 步：配置 hvigorw 编译环境 + SDK 版本门控（必须最先完成）**

**编译是升级流程的硬依赖**——废弃 API 检测靠编译告警、最终验证靠编译结果。**只要装了 DevEco Studio，本地就一定有 hvigorw**，配好环境变量即可。

> 完整的环境探测脚本、`DEVECO_SDK_HOME` 配置细节、常见环境报错及修复见 `references/upgrade-verify.md` 第 1 步。以下是核心流程：

1. **探测 hvigorw**：检查 PATH → 定位 DevEco Studio 安装路径 → 导出 `DEVECO_SDK_HOME` + PATH → `hvigorw --version` 验证
2. **SDK 版本门控**（按路径填 `NEED_API`）：读 `$DEVECO_SDK_HOME/default/sdk-pkg.json` 的 `apiVersion`，确保 ≥ 目标路径要求（路径A ≥ 23；路径B ≥ 26）

**如果 hvigorw 探测失败**（极少见）：向用户索取 DevEco Studio 安装路径后重新配置。**不允许放弃编译**。

**如果 SDK 版本不足（常见）**：**先停下来提醒用户升级 DevEco Studio/SDK**。这是硬前提——SDK 不够，配置改成目标版本后编译必失败（`SDK component missing` / 接口找不到），废弃 API 检测（步骤3）也无法进行。让用户升级 DevEco 到最新版后，再从这里继续。

确定路径并配好 hvigorw + SDK 门控后，**用 TodoWrite 创建内置 todo 清单**。**不允许跳过任何环节**。

内置 todo 清单模板（复制到 TodoWrite，todo 文本保持简洁专业，**不带括号内的技术参数**——路径分支、阈值、grep 命令等执行细节按下文「使用说明」填充）：

```
0. 配置编译环境与 SDK 校验
1. 检测项目基线版本
2. 配置版本号升级
3. 废弃 API 迁移
4. 状态管理 V1→V2 迁移
5. 行为变化适配
6. 编译验证
7. 经验沉淀
```

**使用说明（agent 内部参考，不写入 todo 文本）：**

- **步骤0**：配置 hvigorw + 检查 `apiVersion`（路径A≥23；路径B≥26）；不足先提醒升级 DevEco，再继续
- **步骤1**：读 `build-profile.json5` 定基线版本、统计 V1 装饰器数量，把实际数量填进 todo 备注
- **步骤2**：改 `compatibleSdkVersion`/`targetSdkVersion`——路径A 用 `"6.1.0(23)"`（旧格式带 API level）；路径B 用 `"26.0.0"`（纯 SemVer）
- **步骤3**：clean 编译捕获 deprecated → 按对照表迁移（路径A只迁 since≤23；路径B迁全部）；目标=0
- **步骤4**：**仅路径B**（路径A 跳过，标记 completed 注明"路径A不涉及"）。迁移顺序：
  - 4a 应用级：@StorageLink/@StorageProp/AppStorage → AppStorageV2
  - 4b 数据类：@Observed/@ObjectLink → @ObservedV2/@Trace
  - 4c 跨层级：@Provide/@Consume → @Provider/@Consumer
  - 4d 组件级：@Component→@ComponentV2 + @State→@Local/@Param + @Prop→@Param + @Link→@Param+@Event
  - 4e 监听：@Watch→@Monitor（注意同步→异步时序变化）
- **步骤5**：grep 扫项目代码，命中的才改（累积查表：路径A查 6.0.0+6.1.0 表；路径B查 6.0.0+6.1.0+26.0.0 表）。**禁止预防性修改**
- **步骤6**：目标 = 0 ERROR + 0 deprecated + 运行时无崩溃
- **步骤7**：新踩的坑写回对应 reference

如何执行这个清单：
1. **先跑步骤1检测**，拿到基线版本和 V1 装饰器统计
2. **逐步执行**，每完成一项标记 completed，开始下一项标记 in_progress
3. **每项完成后编译验证**，有错误立即修
4. **如果某项工程里不存在**（如没有 @StorageLink），标记 completed 并注明"工程无此项"
5. **禁止跳过**——即使"看起来不影响编译"也要执行（如 @Component→@ComponentV2）

**第 1 步：检测（必须先做）**

```bash
# 读版本
grep -E "compatibleSdkVersion|targetSdkVersion" build-profile.json5
# 统计 V1 装饰器（决定步骤4的工作量）——排除 widget/卡片目录（WidgetCard 不迁 V2）
grep -rc "@Component\b\|@State\b\|@Prop\b\|@Link\b\|@Watch\b\|@Provide\b\|@Consume\b\|@StorageLink\b\|@StorageProp\b\|@Observed\b\|@ObjectLink\b" \
  --include="*.ets" --exclude-dir=widget --exclude-dir=widgetcard --exclude-dir=oh_modules --exclude-dir=node_modules .
```

**第 2 步：配置升级**

改 build-profile.json5：`compatibleSdkVersion` 和 `targetSdkVersion` 都改成目标版本——**路径A** 用 `"6.1.0(23)"`（旧格式），**路径B** 用 `"26.0.0"`（SemVer）。详见 `references/upgrade-config.md`。

**第 3 步：废弃API迁移（铁律：deprecated 必须降到 0）**

```bash
hvigorw clean --no-daemon
hvigorw assembleHap ... 2>&1 | sed 's/\x1b\[[0-9;]*m//g' > /tmp/build.txt
TOTAL=$(grep -c "has been deprecated" /tmp/build.txt)
OH=$(grep -B1 "has been deprecated" /tmp/build.txt | grep -c "oh_modules")
echo "工程代码 deprecated: $((TOTAL-OH))（目标 0）"
```
查 `references/deprecated-apis.md` 的对照表和迁移档案，逐个迁移。**工具类的 getContext 也要改**。

> 编译器自动只报"当前目标 SDK 已废弃"的告警——路径A 只会报 since≤23 的，路径B 报全部。对照表的 since 列用于人工核对分类。

**第 4 步：状态管理 V1→V2（仅路径B；路径A 跳过此步）**

> **路径A（→6.1.0）**：**跳过**——V1 装饰器在 6.1.0 正常运行，不涉及 V2 迁移。标记 completed 注明"路径A不涉及"。
>
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

详见 `references/behavior-changes.md` 和 `references/migration-guide.md`。

**第 5 步：行为变化适配（grep 命中驱动，只改命中的）**

详见 `references/behavior-changes.md` 块B。**累积查表**：路径A扫6.0.0+6.1.0表；路径B扫6.0.0+6.1.0+26.0.0表（版本隔离阈值是≥，targetSdk越高触发越多）。核心规则：
1. 用对应路径速查表的 grep 关键词扫项目代码
2. **两步门控**：grep 命中 → 读命中行上下文二次确认符合变更描述 → 才允许改
3. grep 无命中 = 跳过，**不允许预防性修改**
4. 交付时逐项列出命中清单（命中的变更项、grep 命中数、是否实际修改及原因），让用户可核查

**第 6 步：验证**

- `hvigorw assembleHap`：0 ERROR
- deprecated 告警：工程代码 0
- 运行时：启动不崩溃（注意字段初始化不能用 UIContext）
- 多设备/UX 检查清单

**第 7 步：经验沉淀**

升级过程中新踩的坑（对照表未覆盖的废弃 API、新增的行为变化、编译错误新解法等）写回对应的 reference 文件，便于后续升级复用。详见 `references/` 下各文件已有的踩坑记录格式。

---

### 断点续跑：如何判断当前进度

如果用户中途中断升级后重新回来，按以下检测顺序判断当前处于哪个环节，从该环节继续（不需要重做已完成的步骤）：

| 检测命令 | 判断逻辑 | 结论 |
|---------|---------|------|
| `hvigorw --version` | 不在 PATH 或报错 | → 步骤0：配 hvigorw |
| 读 `$DEVECO_SDK_HOME/default/sdk-pkg.json` 的 `apiVersion` | < 目标路径要求（A<23 / B<26） | → 步骤0：升级 SDK |
| `grep compatibleSdkVersion build-profile.json5` | 不是目标版本（路径A≠`6.1.0(23)` / 路径B≠`26.0.0`） | → 步骤2：配置升级 |
| `hvigorw clean && hvigorw assembleHap` | 有 deprecated 告警（工程代码 > 0） | → 步骤3：废弃API迁移 |
| `grep -rn "@Component\b\|@State\b" --include="*.ets" --exclude-dir=widget --exclude-dir=oh_modules .` | 仍有 V1 装饰器（仅路径B检查；路径A 跳过） | → 步骤4：V2迁移 |
| 对照行为变化速查表逐项 grep | 有命中项未适配 | → 步骤5：行为变化适配 |
| `hvigorw assembleHap` | 有 ERROR / deprecated > 0 | → 步骤6：验证修复 |

> 检测顺序从上到下，命中第一条即当前进度。如果所有检测都通过，说明升级已完成，跳到步骤6做最终验证确认。

---

### 问题定位

| 现象 | 可能原因 | 修复方向 |
|------|---------|---------|
| 升级后编译报错 | 废弃 API 未迁移 / 版本号格式错 / V2 迁移问题 | 查错误码（`references/upgrade-verify.md`）→ 对应环节的 reference |
| deprecated 告警不为 0 | 工具类 getContext 未改 / Resource 重载未改 / 第三方库告警 | 查 `references/deprecated-apis.md` 对照表，工具类也必须改 |
| 运行时崩溃 | 字段初始化阶段调用了 getUIContext() | 移到 aboutToAppear（见 `references/deprecated-apis.md` 迁移易错点） |
| V2 迁移后 UI 不刷新 | @Trace 漏加 / @Watch→@Monitor 时序变化 / ForEach key 不变断链 | 查 `references/behavior-changes.md` + `references/migration-guide.md` |
| 升级遗漏环节 | 未创建 todo 清单就开始改代码 | 创建 todo 清单后逐项执行 |

---

## 验证清单

**基础验证（每次必做）：**
- [ ] hvigorw 编译环境已配置，`hvigorw --version` 可运行
- [ ] 路径已确定（A→6.1.0 / B→26.0.0）
- [ ] SDK apiVersion 满足该路径要求（A≥23 / B≥26）
- [ ] TodoWrite 清单已创建，步骤按路径标注
- [ ] compatibleSdkVersion 和 targetSdkVersion 改为目标版本值（A=`6.1.0(23)` / B=`26.0.0`）

**压力验证（发布前做）：**
- [ ] `hvigorw assembleHap`：0 ERROR
- [ ] deprecated 告警：工程代码 0
- [ ] 运行时：启动不崩溃（字段初始化未用 UIContext）
- [ ] V1→V2 迁移（**仅路径B**）：无残留（WidgetCard 除外）；路径A 确认已跳过
- [ ] 行为变化：已查对应路径速查表，grep 命中项已逐项交付
- [ ] 多设备/UX 验证清单通过

---

## 常见问题

**Q：升级时能不能跳过某些步骤**
A：不能。todo 清单是强制的——不创建清单就开始改代码，必然遗漏环节。即使"看起来不影响编译"也要执行（如 @Component→@ComponentV2）。如果某项工程里不存在（如没有 @StorageLink），标记 completed 并注明"工程无此项"。

**Q：能不能不编译，直接用 grep 找废弃 API**
A：不能。编译器有完整类型信息和 SDK 的 @deprecated 注解，能精确发现所有废弃点（含 grep 查不到的 Resource 重载、import 别名、对照表外的新废弃）。**绝对禁止输出"本地没有构建工具，请在 DevEco Studio 中打开"然后中止**——装了 DevEco 就有 hvigorw，配好环境即可。

**Q：WidgetCard 要不要迁 V2**
A：不要。WidgetCard（服务卡片）当前对 @ComponentV2 兼容性不好，保留 V1 装饰器。`widget/`、`widgetcard/` 目录或 module.json5 里 `type: "form"` 的模块，在统计 V1 装饰器数量和迁移时都要排除。

**Q：reference 文件可以单独读吗**
A：可以。本 SKILL.md 是路由入口，但每个环节的细节在 `references/` 下对应文件里。用户只问行为变化，直接读 `references/behavior-changes.md`；只问编译报错，直接读 `references/upgrade-verify.md`。按需 Read，不必每次全读。

---

## 延伸阅读（references/ 文件清单）

| 文件 | 覆盖环节 | 何时读 |
|------|---------|--------|
| `references/upgrade-detect.md` | 步骤1 检测 | 定基线版本、判升级路径、识别 FA/Stage 模型时 |
| `references/upgrade-config.md` | 步骤2 配置 | 改 compatibleSdkVersion/targetSdkVersion 时 |
| `references/deprecated-apis.md` | 步骤3 废弃API | 编译捕获 deprecated 告警、按对照表迁移时 |
| `references/behavior-changes.md` | 步骤4 状态管理 + 步骤5 行为变化 | V1→V2 迁移、grep 扫行为变化时 |
| `references/migration-guide.md` | 步骤4 深度参考 | 需查 V1→V2 的 20 项装饰器对照表、机制差异、迁移易错点时（被 behavior-changes.md 引用） |
| `references/migration-archive.md` | 步骤3 深度参考 | 需查具体废弃 API 的替代签名、代码示例、踩坑记录时（被 deprecated-apis.md 引用） |
| `references/upgrade-verify.md` | 步骤6 验证 | hvigorw 编译、错误码修复、多设备验证时 |
