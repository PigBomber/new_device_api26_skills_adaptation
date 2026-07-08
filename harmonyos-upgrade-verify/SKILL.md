---
name: harmonyos-upgrade-verify
description: >
  HarmonyOS 项目升级的编译验证与多设备检查环节。当用户说「升级后编译报错」「hvigorw ERROR」「编译失败怎么修」「升级完要验证什么」「多设备测试」「错误码 xxx 什么意思」时触发。
  本 skill 负责 hvigorw 编译校验、错误码分类修复、多设备/UX 验证清单。属于升级流程的④验证环节，通常由 harmonyos-upgrade 总 skill 路由调用。
version: 1.0.0
---

# 编译验证与多设备检查

## 这个 Skill 做什么

升级改动完成后：
1. 跑 hvigorw 编译，捕获并分类错误
2. 按错误码指引修复
3. 多设备/UX 验证清单逐项确认

## 编译环境配置（关键，必须完成）

**编译是升级流程的硬依赖**：③废弃API检测靠编译告警，⑥验证靠编译结果。**装了 DevEco Studio 就一定有 hvigorw**——配好环境变量即可。不允许以"找不到 hvigorw"为由跳过编译或中止流程。

### hvigorw 定位与配置（多平台 robust 探测）

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

### 如果自动探测失败

向用户索取路径后，手动配置：
```bash
# 用户提供 DevEco Studio 安装路径后
export DEVECO_SDK_HOME="<路径>/Contents/sdk"   # macOS；Win/Linux 去掉 Contents/
export PATH="<路径>/Contents/tools/hvigor/bin:$PATH"
export PATH="<路径>/Contents/tools/ohpm/bin:$PATH"
hvigorw --version  # 确认
```

**常见环境报错与修复：**

| 报错 | 原因 | 修复 |
|-----|------|------|
| `hvigorw not found` | hvigorw 不在 PATH | 运行上方探测脚本定位 DevEco Studio，配 PATH |
| `Invalid value of 'DEVECO_SDK_HOME'` | 未设置 SDK 路径 | export DEVECO_SDK_HOME |
| `SDK component missing` | DEVECO_SDK_HOME 指错了层级 | 指向 `Contents/sdk`（让 hvigor 自己找 default 子目录），不是 `Contents/sdk/default` |
| `compatibleSdkVersion` 对应的 SDK 未安装 | 装的 SDK 版本低于工程配置 | 在 DevEco SDK Manager 下载对应版本 |
| Win 下 `hvigorw` 无扩展名报错 | Git Bash 不认 .bat | 用 `hvigorw.bat` 或在 cmd/PowerShell 执行 |

**确认 SDK 版本：**
```bash
grep apiVersion "$DEVECO_SDK_HOME/default/sdk-pkg.json"
# 应输出 "apiVersion": "26" 表示装了 API 26 SDK
```

## 编译命令

```bash
cd <工程根目录>
hvigorw clean                          # 清缓存（增量编译可能不报 deprecated，要 clean 才全量）
hvigorw assembleHap --mode module -p module=phone@default -p product=default --no-daemon 2>&1 | tail -40
```

**模块名注意**：`-p module=` 后面跟的是模块名，不一定是 `entry`。从工程 `build-profile.json5` 的 `modules[].name` 读取（如本工程是 `phone`）。

**捕获 deprecated 告警**：
```bash
# clean 后全量编译才能看到完整告警（增量编译用缓存会跳过）
hvigorw clean --no-daemon
hvigorw assembleHap --mode module -p module=phone@default -p product=default --no-daemon 2>&1 | sed 's/\x1b\[[0-9;]*m//g' > /tmp/build.txt
```

**统计 deprecated 数量（必须用可靠方法）**：
```bash
# 正确：先存日志到文件，再分步统计
TOTAL=$(grep -c "has been deprecated" /tmp/build.txt)
OH_MODULES=$(grep -B1 "has been deprecated" /tmp/build.txt | grep -c "oh_modules")
PROJECT=$((TOTAL - OH_MODULES))
echo "工程代码 deprecated: $PROJECT（目标 0）"
```

**注意事项**：
- 告警带 ANSI 颜色码，必须先 `sed 's/\x1b\[[0-9;]*m//g'` 去色再处理
- **不要用 paste 命令混合不同日志文件的行**——会把旧日志的 deprecated 行和新日志的 File 行错位拼接，造成假告警
- deprecated 告警是两行格式：`File: xxx.ets:行:列` + `'API名' has been deprecated.`，统计时用 `grep -B1` 配对
- 每次统计必须基于**当前最新一次** clean 编译的日志，不要复用旧日志文件

## 错误码速查

| 错误码 | 类别 | 原因 | 修复 |
|--------|------|------|------|
| **10605080** | ArkTS 语法 | `for..in` / `any` / 类型断言等不支持 | 改用 `Object.keys().forEach()` 或 `for..of`；声明明确类型 |
| **10605029** | ArkTS 类型 | 索引访问 `obj[key]` 不支持 | 用 `Record<string, T>` 类型声明 |
| **10605001** | 类型错误 | 对象类型赋值不匹配 / 属性不存在 | 检查类型声明；如 `UIContext.getContext` 不存在（应用 `getContext(this)`） |
| **10605008** | any/unknown | 用了 `any` 或 `unknown` 类型 | 声明明确类型（如 `common.UIAbilityContext`） |
| **10605999** | undefined | Object is possibly 'undefined'（如 `getHostContext()` 返回值未判空） | 加判空 `if (ctx) { ... }` 或非空断言 `!` |
| **10903329** | 资源错误 | 未知的资源名称 | 检查 `$r()` 引用路径 |
| **10905324** | V2 状态管理 | @Local 属性被外部初始化 | 改用 @Param 或 @Param @Once（见 behavior-changes 迁移指南） |
| **10905213** | V2/V1 混用 | V2 组件内用了含 @Link 的 V1 系统组件 | 查 behavior-changes 迁移指南第5节映射表：**有 V2 替代的必须替换**（SegmentButton→SegmentButtonV2、TextReaderIcon→TextReaderIconV2、ProgressButton/SubHeader/ToolBar→V2）；**无 V2 替代的（ComposeTitleBar/Chip/TreeView 等，见白名单）该 struct 保留 @Component** |
| **00308018** | 版本号格式 | `api version parameter is illegal` | 配置改纯 SemVer `"26.0.0"` |
| **00303168** | SDK 缺失 | `SDK component missing` | DEVECO_SDK_HOME 指向 `Contents/sdk`（非 `Contents/sdk/default`） |
| **10505001** | 模块导入 | 找不到模块或接口 | 接口可能已废弃或改名，查 harmonyos-behavior-changes |
| **10200006** | Worker 序列化 | 数据无法序列化 | 用可序列化数据类型 |
| **10200020** | Method 不可调用 | async 方法无法被 callGlobalCallObjectMethod 调用 | 改用同步方法或调整 Worker 通信 |

## 修复循环

```
编译报错 → 看错误码 → 查上表定修复方向 → 改代码 → 重新编译 → 直到 0 ERROR
```

**注意**：错误码 10505001（模块/接口找不到）通常意味着 API 已废弃或改名，需查 `harmonyos-behavior-changes` skill 的对应版本文件确认。

## 多设备/UX 验证清单

升级到 26.0.0 后逐项确认：

### 配置验证
- [ ] `build-profile.json5` 中 `compatibleSdkVersion` = `"26.0.0"`
- [ ] 所有模块 `targetSdkVersion` = `"26.0.0"`
- [ ] `hvigorw assembleHap` 无 ERROR
- [ ] 无 deprecated API 警告残留

### 版本隔离行为验证（targetSdk ≥ 26 才生效）
- [ ] 触摸热区（Button/Toggle/Select/Chip）高度 32vp 是否影响布局
- [ ] Dialog/Toast/AlphabetIndexer 沉浸式材质效果是否可接受
- [ ] 内置文本孤字换行/小语种行高/音节换行效果
- [ ] NodeAdapter onAttachToNode 回调时机是否正常
- [ ] matchParent 布局是否符合预期
- [ ] 阴影（radius=0）效果是否符合预期

### 无版本隔离行为验证（必须正常）
- [ ] Web 组件（ArkWeb/JSVM 内核 132→144）页面显示正常
- [ ] async 函数相关逻辑（util.types().isAsyncFunction）无异常
- [ ] XML 解析（fastConvertToJSObject）结果正确
- [ ] 鼠标事件 rawDeltaX/rawDeltaY 数值合理
- [ ] 媒体库权限（READ_IMAGEVIDEO）能读到本地资源

### 多设备
- [ ] 手机布局正常
- [ ] 平板/折叠屏布局正常
- [ ] 2in1/PC 布局正常（如适用）

## 与其他子 skill 的衔接

- 编译报错涉及 API 行为变化 → 查 `harmonyos-behavior-changes`
- 编译报错涉及版本号格式 → 查 `harmonyos-upgrade-config`
- 全部验证通过 → 升级完成
