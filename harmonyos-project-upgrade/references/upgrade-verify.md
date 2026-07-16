# 编译验证与多设备检查

> 本文件为 `../SKILL.md` 的补充，覆盖升级流程步骤6「编译验证」环节：hvigorw 编译、错误码修复、多设备验证。当 SKILL.md 路由到验证环节时读取本文件。

## Agent Interface

```yaml
symptom_keywords:
  - hvigorw not found / not in PATH
  - SDK component missing
  - Invalid value of DEVECO_SDK_HOME
  - compile ERROR after upgrade
  - deprecated warnings count not reaching zero
  - Cannot find module 10505001 batch errors (single-module compile false errors)
  - Object is possibly undefined 10605999
  - V2 component cannot be used with @Link 10905213
  - @Local property cannot be initialized 10905324
  - multi-module project compile errors

hard_constraints:
  - Compiling is a hard dependency of the upgrade flow — deprecated API detection relies on compiler warnings, final verification relies on compile results; never skip compilation or abort with "no build tool found"
  - DevEco Studio includes hvigorw — if DevEco is installed, hvigorw exists; configure environment variables instead of asking user to open DevEco Studio
  - DEVECO_SDK_HOME must point to Contents/sdk (macOS) or sdk (Win/Linux) — never to Contents/sdk/default; let hvigor find the default subdirectory itself
  - Multi-module projects (with file: HAR dependencies) must use full-project compile (no --mode module) — single-module compile produces batch false errors (10505001/10905204/10903329)
  - deprecated warning statistics must be based on the latest clean compile log — never reuse old log files; never use paste to mix lines from different log files
  - ANSI color codes must be stripped with sed before processing build logs

diagnostic_checklist:
  - Is hvigorw in PATH or has DevEco Studio been located via the detection script?
  - Is DEVECO_SDK_HOME pointing to Contents/sdk (not Contents/sdk/default)?
  - Is the project multi-module with file: HAR dependencies? (If so, use full-project compile, not --mode module)
  - Has ohpm install been run at project root and every module with oh-package.json5?
  - Was hvigorw clean run before assembleHap to get full deprecated warnings?
  - Are deprecated warnings counted correctly (TOTAL minus oh_modules)?
```

# 编译验证与多设备检查

## 适配领域

本文件覆盖升级流程的最后一步：跑 hvigorw 编译捕获并分类错误、按错误码指引修复、多设备/UX 验证清单逐项确认。

**不覆盖**：版本检测（见 `upgrade-detect.md`）；配置修改（见 `upgrade-config.md`）；废弃 API 迁移（见 `deprecated-apis.md`）；行为变化适配（见 `behavior-changes.md`）。

---

## 错误码分类矩阵

| 错误码 | 类别 | 原因 | 修复 |
|--------|------|------|------|
| **10605080** | ArkTS 语法 | `for..in` / `any` / 类型断言等不支持 | 改用 `Object.keys().forEach()` 或 `for..of`；声明明确类型 |
| **10605029** | ArkTS 类型 | 索引访问 `obj[key]` 不支持 | 用 `Record<string, T>` 类型声明 |
| **10605001** | 类型错误 | 对象类型赋值不匹配 / 属性不存在 | 检查类型声明；如 `UIContext.getContext` 不存在（应用 `getContext(this)`） |
| **10605008** | any/unknown | 用了 `any` 或 `unknown` 类型 | 声明明确类型（如 `common.UIAbilityContext`） |
| **10605999** | undefined | `Object is possibly 'undefined'`（如 `getHostContext()` 返回值未判空，尤其作实参传递时） | 加判空 `if (ctx) { ... }` 或非空断言 `!`（组件生命周期内必然有值） |
| **10903329** | 资源错误 | 未知的资源名称 | 检查 `$r()` 引用路径；**多模块工程若批量出现 + 伴随 10505001/10905204，是单模块编译假错误** |
| **10905204** | UI 组件语法 | `'XxxPage();' does not meet UI component syntax` | 通常被 `Cannot find module` 连带触发——兄弟模块未解析导致组件类型未知。先解决模块依赖（全工程编译） |
| **10905324** | V2 状态管理 | @Local 属性被外部初始化 | 改用 @Param 或 @Param @Once（见 behavior-changes 迁移指南） |
| **10905213** | V2/V1 混用 | V2 组件内用了含 @Link 的 V1 系统组件 | 查 behavior-changes 迁移指南映射表：有 V2 替代的必须替换；无 V2 替代的该 struct 保留 @Component |
| **00308018** | 版本号格式 | `api version parameter is illegal` | 配置改纯 SemVer `"26.0.0"` |
| **00303168** | SDK 缺失 | `SDK component missing` | 两种原因：① DEVECO_SDK_HOME 指错了层级（应指 `Contents/sdk`，非 `Contents/sdk/default`）；② **本地 SDK 版本不足**（apiVersion 低于目标路径要求：路径A需≥23，路径B需≥26）——需升级 DevEco Studio 到最新版或在 SDK Manager 下载对应 API（见步骤1 前置门控） |
| **10505001** | 模块导入 | 找不到模块或接口 | 接口可能已废弃或改名，查 behavior-changes.md |
| **10200006** | Worker 序列化 | 数据无法序列化 | 用可序列化数据类型 |
| **10200020** | Method 不可调用 | async 方法无法被 callGlobalCallObjectMethod 调用 | 改用同步方法或调整 Worker 通信 |

> 错误码 10505001（模块/接口找不到）通常意味着 API 已废弃或改名，需查 `behavior-changes.md` 确认。

---

## 适配流程

### 新适配

**第 1 步：配置 hvigorw 编译环境**

**编译是升级流程的硬依赖**——步骤3废弃API检测靠编译告警，步骤6验证靠编译结果。**装了 DevEco Studio 就一定有 hvigorw**，配好环境变量即可。不允许以"找不到 hvigorw"为由跳过编译或中止流程。

hvigorw 不在系统 PATH 里，编译前必须定位 DevEco Studio 安装路径并配置环境变量。**不要写死路径**——安装位置因平台/用户/版本而异，用以下脚本动态探测：

```bash
# 第 1 步：检查是否已在 PATH
if command -v hvigorw >/dev/null 2>&1; then
  echo "hvigorw 已在 PATH"
else
  # 第 2 步：定位 DevEco Studio 安装路径（按平台）
  DEVECO_HOME=""
  # macOS：/Applications 或 ~/Applications
  [ -z "$DEVECO_HOME" ] && DEVECO_HOME=$(find /Applications ~/Applications -maxdepth 2 -name "DevEco*Studio*" -type d 2>/dev/null | head -1)
  # Windows (Git Bash/WSL)：/c/Program Files/Huawei/
  [ -z "$DEVECO_HOME" ] && DEVECO_HOME=$(ls -d /c/Program\ Files/Huawei/DevEco*Studio 2>/dev/null | head -1)
  # Linux：/opt 或 ~/
  [ -z "$DEVECO_HOME" ] && DEVECO_HOME=$(find /opt ~ -maxdepth 3 -name "DevEco*Studio*" -type d 2>/dev/null | head -1)
  # 用户自定义环境变量
  [ -z "$DEVECO_HOME" ] && [ -n "$DEVECO_STUDIO_HOME" ] && DEVECO_HOME="$DEVECO_STUDIO_HOME"

  if [ -n "$DEVECO_HOME" ]; then
    # 第 3 步：设置环境变量（macOS 可执行在 Contents/ 下，Win/Linux 直接在根下）
    TOOLS_DIR="$DEVECO_HOME/Contents/tools"
    SDK_DIR="$DEVECO_HOME/Contents/sdk"
    [ ! -d "$TOOLS_DIR" ] && TOOLS_DIR="$DEVECO_HOME/tools"      # Win/Linux 布局
    [ ! -d "$SDK_DIR" ]  && SDK_DIR="$DEVECO_HOME/sdk"
    export DEVECO_SDK_HOME="$SDK_DIR"
    export PATH="$TOOLS_DIR/hvigor/bin:$TOOLS_DIR/ohpm/bin:$PATH"
    echo "已配置 hvigorw（来自 $DEVECO_HOME）"
  else
    # 第 4 步：都没找到，向用户索取路径（不能放弃）
    echo "未找到 DevEco Studio。请提供安装路径："
    echo "   macOS: 拖 DevEco Studio 图标到终端获取路径，或 ls /Applications | grep DevEco"
    echo "   Win:   在 DevEco 里看 File > Settings > SDK Manager 的 SDK Location"
    echo "   或直接执行: find / -name hvigorw -type f 2>/dev/null"
  fi
fi

# 第 5 步：验证 hvigorw 真能跑
hvigorw --version
```

> **关键注意**：`DEVECO_SDK_HOME` 必须指向 `Contents/sdk`（macOS）或 `sdk`（Win/Linux），让 hvigor 自己找 default 子目录——**不要**指向 `Contents/sdk/default`。

如果自动探测失败，向用户索取路径后手动配置：
```bash
export DEVECO_SDK_HOME="<路径>/Contents/sdk"   # macOS；Win/Linux 去掉 Contents/
export PATH="<路径>/Contents/tools/hvigor/bin:$PATH"
export PATH="<路径>/Contents/tools/ohpm/bin:$PATH"
hvigorw --version  # 确认
```

确认 SDK 版本（前置门控，必须先过这关）：
```bash
# 按路径填 NEED_API：路径A(→6.1.0) 填 23；路径B(→26.0.0) 填 26
NEED_API=23   # 路径A；路径B 改成 26
SDK_VER=$(grep -o '"apiVersion"[[:space:]]*:[[:space:]]*"[0-9]*"' "$DEVECO_SDK_HOME/default/sdk-pkg.json" 2>/dev/null | grep -o '[0-9]*')
echo "本地 SDK apiVersion: $SDK_VER（目标路径需要 ≥ $NEED_API）"
```
- **apiVersion ≥ NEED_API**：SDK 够，继续后续编译。
- **apiVersion < NEED_API 或读不到**：**SDK 版本不足，停下来**。本地 DevEco Studio/SDK 版本太低——必须先升级 DevEco Studio 到最新版（DevEco 自带对应 SDK），或在 DevEco SDK Manager 下载目标 API。**不升级 SDK 就改配置，会报 `SDK component missing` / 找不到目标版本接口，编译必失败，整个升级流程无法进行。**

**第 2 步：编译**

```bash
cd <工程根目录>
hvigorw clean                          # 清缓存（增量编译可能不报 deprecated，要 clean 才全量）
```

hvigorw assembleHap --no-daemon 2>&1 | tail -40
```

**多模块工程（含 `file:` HAR 依赖）必须全工程编译**（上例默认命令）：兄弟模块不会被解析会报大量假错误（10505001/10905204/10903329）。**单模块工程**可加 `--mode module -p module=<模块名>@default -p product=default`（模块名从 `build-profile.json5` 的 `modules[].name` 读取，不一定是 `entry`）。

编译前先 `ohpm install`（工程根 + 每个含 oh-package.json5 的模块），否则 `file:` 本地依赖链接不上。判别假错误的方法：如果 10505001/10905204/10903329 成批出现且都指向 `entry` 模块、引用的模块名是兄弟 HAR（book_home/common 等），就是单模块编译的假错误，改全工程编译即消失。

**第 3 步：统计 deprecated 告警**

```bash
# clean 后全量编译才能看到完整告警（增量编译用缓存会跳过）
hvigorw clean --no-daemon
hvigorw assembleHap --no-daemon 2>&1 | sed 's/\x1b\[[0-9;]*m//g' > /tmp/build.txt

# 正确：先存日志到文件，再分步统计
TOTAL=$(grep -c "has been deprecated" /tmp/build.txt)
OH_MODULES=$(grep -B1 "has been deprecated" /tmp/build.txt | grep -c "oh_modules")
PROJECT=$((TOTAL - OH_MODULES))
echo "工程代码 deprecated: $PROJECT（目标 0）"
```

注意事项：
- 告警带 ANSI 颜色码，必须先 `sed 's/\x1b\[[0-9;]*m//g'` 去色再处理
- **不要用 paste 命令混合不同日志文件的行**——会把旧日志的 deprecated 行和新日志的 File 行错位拼接，造成假告警
- deprecated 告警是两行格式：`File: xxx.ets:行:列` + `'API名' has been deprecated.`，统计时用 `grep -B1` 配对
- 每次统计必须基于**当前最新一次** clean 编译的日志，不要复用旧日志文件

**第 4 步：修复循环**

```
编译报错 → 看错误码 → 查错误码分类矩阵定修复方向 → 改代码 → 重新编译 → 直到 0 ERROR
```

---

### 问题定位

**环境报错：**

| 报错 | 原因 | 修复 |
|-----|------|------|
| `hvigorw not found` | hvigorw 不在 PATH | 运行上方探测脚本定位 DevEco Studio，配 PATH |
| `Invalid value of 'DEVECO_SDK_HOME'` | 未设置 SDK 路径 | export DEVECO_SDK_HOME |
| `SDK component missing` | 两种：① DEVECO_SDK_HOME 指错层级（应指 `Contents/sdk` 非 `Contents/sdk/default`）；② **本地 SDK 版本不足**（apiVersion 低于目标路径要求：A<23 或 B<26） | 先查 `grep apiVersion .../sdk-pkg.json`：指错层级→修正路径；版本不足→**升级 DevEco Studio 到最新版或在 SDK Manager 下载对应 API**（用户说"升最新版"但 DevEco 还是老版本时的最常见原因） |
| `compatibleSdkVersion` 对应的 SDK 未安装 | 装的 SDK 版本低于工程配置（如配了 26.0.0 但只装了 API 20） | **升级 DevEco Studio 到最新版**（自带对应 SDK），或在 SDK Manager 下载目标 API level |
| Win 下 `hvigorw` 无扩展名报错 | Git Bash 不认 .bat | 用 `hvigorw.bat` 或在 cmd/PowerShell 执行 |

**编译报错**：查上方「错误码分类矩阵」。

---

## 关键 API

### hvigorw

**用途**：鸿蒙工程的编译构建工具，废弃 API 检测和最终验证的唯一可靠依据。

**注意**：不在系统 PATH，需通过 DevEco Studio 安装路径定位；多模块工程不能单模块编译。

### DEVECO_SDK_HOME

**用途**：指向 DevEco Studio SDK 目录的环境变量。

**注意**：必须指向 `Contents/sdk`（macOS）或 `sdk`（Win/Linux），让 hvigor 自己找 default 子目录——不要指向 `Contents/sdk/default`。

---

## 代码模式

### 模式 1：环境探测与配置

```bash
# ✅ 多平台动态探测 DevEco Studio，不写死路径
DEVECO_HOME=$(find /Applications ~/Applications -maxdepth 2 -name "DevEco*Studio*" -type d 2>/dev/null | head -1)
export DEVECO_SDK_HOME="$DEVECO_HOME/Contents/sdk"  # ✅ 指向 sdk，不是 sdk/default
export PATH="$DEVECO_HOME/Contents/tools/hvigor/bin:$PATH"
hvigorw --version  # 验证
```

### 模式 2：多模块工程全工程编译

```bash
# ✅ 多模块工程：先 ohpm install，再全工程编译（不带 --mode module）
ohpm install
hvigorw assembleHap --no-daemon

# ❌ 错误：单模块编译会报假错误（10505001/10905204/10903329 批量出现）
# hvigorw assembleHap --mode module -p module=entry@default
```

### 模式 3：deprecated 告警统计

```bash
# ✅ 正确：存日志到文件，去色，分步统计
hvigorw clean --no-daemon
hvigorw assembleHap --no-daemon 2>&1 | sed 's/\x1b\[[0-9;]*m//g' > /tmp/build.txt
TOTAL=$(grep -c "has been deprecated" /tmp/build.txt)
OH_MODULES=$(grep -B1 "has been deprecated" /tmp/build.txt | grep -c "oh_modules")
echo "工程代码 deprecated: $((TOTAL - OH_MODULES))（目标 0）"

# ❌ 错误：用 paste 混合不同日志文件——行错位拼接，造成假告警
```

---

## 验证清单

**基础验证（每次必做）：**
- [ ] `build-profile.json5` 中 `compatibleSdkVersion` 已改为目标版本（路径A=`"6.1.0(23)"`；路径B=`"26.0.0"`）
- [ ] 所有模块 `targetSdkVersion` 已统一为目标版本
- [ ] `hvigorw assembleHap` 无 ERROR
- [ ] 无 deprecated API 警告残留（工程代码 0）

**压力验证（发布前做）：**
- [ ] **版本隔离行为验证（版本隔离阈值按路径：路径B 为 targetSdk ≥ 26.0.0；路径A 为 targetSdk ≥ 6.1.0(23)）**
  - [ ] 触摸热区（Button/Toggle/Select/Chip）高度 32vp 是否影响布局
  - [ ] Dialog/Toast/AlphabetIndexer 沉浸式材质效果是否可接受
  - [ ] 内置文本孤字换行/小语种行高/音节换行效果
  - [ ] NodeAdapter onAttachToNode 回调时机是否正常
  - [ ] matchParent 布局是否符合预期
  - [ ] 阴影（radius=0）效果是否符合预期
- [ ] **无版本隔离行为验证（必须正常）**
  - [ ] Web 组件（ArkWeb/JSVM 内核 132→144）页面显示正常
  - [ ] async 函数相关逻辑（util.types().isAsyncFunction）无异常
  - [ ] XML 解析（fastConvertToJSObject）结果正确
  - [ ] 鼠标事件 rawDeltaX/rawDeltaY 数值合理
  - [ ] 媒体库权限（READ_IMAGEVIDEO）能读到本地资源
- [ ] **多设备**
  - [ ] 手机布局正常
  - [ ] 平板/折叠屏布局正常
  - [ ] 2in1/PC 布局正常（如适用）

---

## 常见问题

**Q：报 "hvigorw not found" 或 "SDK component missing"**
A：hvigorw 不在系统 PATH。运行环境探测脚本定位 DevEco Studio 安装路径，配置 `DEVECO_SDK_HOME` 指向 `Contents/sdk`（不是 `Contents/sdk/default`）和 PATH。装了 DevEco Studio 就一定有 hvigorw。

**Q：多模块工程编译报大量 "Cannot find module"（10505001）+ 组件语法（10905204）+ 资源（10903329）错误**
A：这是单模块编译的假错误。多模块工程（含 `file:` HAR 依赖）不能用 `--mode module -p module=entry` 单模块编译——兄弟模块不会被解析。正确做法是先 `ohpm install`，再不带 `--mode module` 全工程编译：`hvigorw assembleHap --no-daemon`。

**Q：deprecated 告警统计数量不准**
A：检查以下几点：是否先 `hvigorw clean` 再全量编译（增量编译用缓存会跳过告警）；是否用 `sed` 去掉了 ANSI 颜色码；是否用 `grep -B1` 配对了两行格式的告警；是否复用了旧日志文件（每次必须基于最新 clean 编译日志）；是否误用了 paste 混合不同日志文件（会行错位）。

**Q：报 10605999 "Object is possibly 'undefined'"**
A：`getHostContext()` 返回 `Context | undefined`。加判空 `if (ctx) { ... }` 或非空断言 `!`（组件生命周期内必然有值）。注意闭包内控制流分析会重置，仍认为可能 undefined，需用非空断言 `!`。

**Q：报 10905213 "V2 component cannot be used with @Link"**
A：V2 组件内用了含 @Link 的 V1 系统组件。查 behavior-changes 迁移指南映射表：有 V2 替代的（SegmentButton/TextReaderIcon/ProgressButton/SubHeader/ToolBar/Dialog 等）必须替换成 V2 版本；无 V2 替代的（ComposeTitleBar/Chip/TreeView 等，见白名单）该 struct 保留 @Component。

---

## 延伸阅读

- `behavior-changes.md`：行为变化适配——编译报错涉及 API 行为变化时查
- `upgrade-config.md`：配置文件升级——编译报错涉及版本号格式时查
- `deprecated-apis.md`：废弃 API 检查与替换——deprecated 告警迁移时查
