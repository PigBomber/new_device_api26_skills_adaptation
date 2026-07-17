# 叠加 UI 布局 API 速查

> 本文件为 `SKILL.md` 的补充，记录后置摄像头**叠加 UI 多形态布局适配**用到的 API：安全区避让、形态/窗口监听、响应式布局属性。SKILL.md 讲流程与模式，本文件讲 API 细节与枚举速查。
>
> **不在本文件**：预览 Surface 几何（`setXComponentSurfaceRect` 宽高比数学）、旋转角度（`getPreviewRotation`/`capture(rotation)`）、stride 防花屏、相机枚举与会话重建——这些属硬件适配范畴，不在本 skill。

---

## 安全区避让

叠加在相机预览上的控件（快门、模式条、滑块）常会被系统状态栏、导航指示条、挖孔遮挡。避让的标准做法是**背景撑满 + 内容内缩**。

### 核心方法

| 方法/属性 | 归属 | 用途 |
|---------|------|------|
| `expandSafeArea(edges?, enable?)` | 通用属性 | 让组件**背景**延伸到安全区外（沉浸式），常用于 XComponent 预览层 |
| `window.getLastWindow(ctx)` → `win.getWindowAvoidArea(type)` | window | 取指定类型的避让矩形（px），内容层据此 `padding` 内缩 |
| `win.on('avoidAreaChange')` | window | 安全区变化时回调（键盘弹出、横竖屏系统栏高度变），重算内缩 |

### AvoidAreaType 枚举

| 枚举值 | 含义 | 叠加控件是否常避让 |
|--------|------|---------|
| `TYPE_SYSTEM` | 状态栏等系统区域 | 是 |
| `TYPE_NAVIGATION_INDICATOR` | 底部导航指示条 | 是 |
| `TYPE_CUTOUT` | 挖孔/刘海 | 是 |
| `TYPE_KEYBOARD` | 软键盘 | 按需 |

### getWindowAvoidArea 返回结构

`AvoidArea` 含 `topRect` / `bottomRect` / `leftRect` / `rightRect`，每个 `{ left, top, width, height }`（px）。内容内缩时用 `px2vp` 转 vp 再设 `padding`。

> **常见组合**：顶部避让取 `TYPE_SYSTEM` 与 `TYPE_CUTOUT` 的最大值；底部避让 `TYPE_NAVIGATION_INDICATOR`；横屏左右常需避让 `TYPE_CUTOUT`。

### expandSafeArea 参数

- `edges: Edge` 或 `Edges[]`：要延伸的边（`Edge.TOP`/`Edge.BOTTOM`/`Edge.LEFT`/`Edge.RIGHT`/`Edge.ALL`）。
- `enable: boolean` 或 `boolean[]`：对应边是否开启。
- 预览层常用 `.expandSafeArea(true, true)`（宽高都延伸）做沉浸式背景。

---

## 形态 / 窗口监听

叠加控件重排的触发源。预览区那个矩形的尺寸/比例/位置变化时，监听这些事件触发 ArkUI 重渲染。

### display（屏幕/折叠）

| API | 用途 | 注意 |
|------|------|------|
| `display.isFoldable()` | 当前设备是否折叠屏 | **必须用这个**，不要用 `deviceInfo.deviceType`（折叠设备返回 `'phone'`） |
| `display.getDefaultDisplaySync()` | 取默认 Display 对象 | 读 `.rotation`/`.width`/`.height` |
| `display.on('change', callback)` | 屏幕属性变化（含旋转） | **补 `windowSizeChange` 的 180° 旋转盲区** |
| `display.off('change')` | 取消监听 | aboutToDisappear 时调用 |

### FoldStatus 枚举（折叠状态）

| 枚举值 | 含义 |
|--------|------|
| `UNKNOWN` | 未知 |
| `FOLDED` | 折叠态（外屏） |
| `EXPANDED` | 展开态 |
| `HALF_FOLDED` | 半折叠/悬停态（过渡态，常可豁免重排） |

相关事件/属性：`foldStatusChange`（监听折叠状态变化）、`foldDisplayMode` / `foldDisplayModeChange`（折叠显示模式）、`display.getFoldStatus()`（取当前状态）、`display.getCurrentFoldCreaseRegion()`（折痕区域，避让折痕）。

> **豁免原则**：`HALF_FOLDED`↔`EXPANDED` 间若预览区几何不变（可用镜头集没变），叠加控件重排可跳过。只有真正改变预览区尺寸/比例的形态切换才触发重排。

### 窗口（window）

| API | 用途 | 注意 |
|------|------|------|
| `window.getLastWindow(ctx)` | 取当前窗口 | 返回 `Window` 实例，挂监听 |
| `win.on('windowSizeChange', cb)` | 窗口尺寸变化（分屏、部分旋转） | **180° 旋转不触发**（宽高没变），须配 `display.on('change')` |
| `win.on('avoidAreaChange', cb)` | 安全区变化 | 键盘弹出/横竖屏系统栏变时重算内缩 |
| `win.off(...)` | 取消监听 | aboutToDisappear 时清理 |

> **180° 旋转盲区**：旋转 180° 时窗口宽高不变，`windowSizeChange` 不触发。只监听它会漏掉 180° 后的控件重排。必须配 `display.on('change')` 读 `display.rotation`（0/90/180/270）补盲。这是叠加控件跨旋转适配最常踩的坑。

### display.rotation 枚举值

| 值 | 含义 |
|----|------|
| `0` | 竖屏（默认） |
| `1` | 左横屏（90°） |
| `2` | 反向竖屏（180°） |
| `3` | 右横屏（270°） |

> 判断横竖屏：`rotation === 1 || rotation === 3` → landscape，否则 portrait。

### 旋转策略与断点

- **sm 断点**：不旋转（手机竖屏固定）。
- **md/lg 断点**：支持旋转（窗口最小维度 ≥ 600vp）。
- 影响：md/lg 设备上叠加控件要同时考虑横竖屏两套排列；sm 只需竖屏一套。

> 断点（`sm`/`md`/`lg`）的获取与 `GridRow`/`GridCol` 布局属通用一多范畴，不在本 skill。本 skill 聚焦相机叠加控件的响应式重排与避让；需要按断点切换控件排列结构本身时，属通用一多范畴。

### onAreaChange（组件尺寸感知）

| API | 用途 |
|------|------|
| `.onAreaChange(oldValue, newValue)` | 组件自身尺寸/位置变化时回调 |

预览区域比例变化（前后摄 profile 不同、`onSurfaceChanged`）也可由此感知，作为触发控件重排的信号之一。预览区几何本身（Surface 宽高比怎么算）属硬件适配范畴。

---

## 响应式布局属性速查（相机叠加控件）

叠加控件跨形态重排用的 ArkUI 响应式属性。核心思想：控件不写死坐标，随容器自动重排。

| 属性/组件 | 用途 | 相机叠加场景 |
|---------|------|---------|
| `Stack` | 层叠容器，后入元素在上 | XComponent（底层）+ 叠加控件层（上层）同 Stack |
| `Blank()` | 弹性空白占位 | 顶部模式条与底部快门之间撑开，随容器高度自适应 |
| `layoutWeight(number)` | 按权重分配父容器剩余空间 | 多个控件区按比例分高度/宽度 |
| `width('100%')` / `height('100%')` | 百分比尺寸 | 控件层铺满 Stack，随预览区缩放 |
| `aspectRatio(ratio)` | 固定宽高比 | 快门按钮保持圆形不受容器形变影响 |
| `justifyContent(FlexAlign)` | 主轴对齐 | `Center`/`SpaceEvenly`/`SpaceBetween` 排列模式按钮 |
| `alignItems(HorizontalAlign/VerticalAlign)` | 交叉轴对齐 | 控件在交叉方向居中 |
| `padding({top,bottom,left,right})` | 内边距 | 内容内缩安全区（值来自 getWindowAvoidArea） |

### FlexAlign 常用值

| 值 | 含义 |
|----|------|
| `Start` / `Center` / `End` | 起始/居中/末尾对齐 |
| `SpaceBetween` | 两端贴边，中间均分间距 |
| `SpaceAround` | 每个元素两侧等距 |
| `SpaceEvenly` | 所有间距（含两端）均等 |

### 典型叠加布局结构

```typescript
Stack() {
  XComponent(...)                         // 预览层（背景撑满安全区）
    .expandSafeArea(true, true)

  Column() {                              // 控件层（内容内缩安全区）
    Row() { /* 模式切换条 */ }
      .width('100%').justifyContent(FlexAlign.Center)

    Blank()                               // 弹性占位：随容器高度把底部控件推下去

    Row() {                               // 底部控件
        Image($r('app.media.shutter'))
          .width(72).height(72).aspectRatio(1)   // 圆形快门不受形变影响
      }
      .width('100%').justifyContent(FlexAlign.SpaceEvenly)
  }
  .width('100%').height('100%')
  .padding({ top: safeInsetTop, bottom: safeInsetBottom })   // 内缩安全区
}
```

> 这个结构在折叠/旋转/分屏/切相机下都不需要手算坐标：`Blank` 自适应高度、`width('100%')` 自适应宽度、`padding` 内缩动态安全区。预览区比例变化时控件层自动重排。

---

## 生命周期清理

所有监听必须在 `aboutToDisappear` 取消，避免泄漏：

```typescript
aboutToDisappear() {
  display.off('change');
  this.win?.off('windowSizeChange');
  this.win?.off('avoidAreaChange');
  if (this.resizeTimer !== -1) clearTimeout(this.resizeTimer);
}
```

> 形态/窗口监听不清理会导致页面销毁后仍回调，引发对已释放组件的操作崩溃。
