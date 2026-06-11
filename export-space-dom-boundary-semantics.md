# Space DOM 边界与 Transform 源码校准分析

> 版本：1.0
> 日期：2026-06-11
> 数据来源：react-zoomable-ui v0.11.0 编译源码（unpkg.com）

---

## 一、react-zoomable-ui v0.11.0 源码校准

### 1.1 Space 组件 render 方法

**源码位置**：[Space.js#L254-L274](https://unpkg.com/react-zoomable-ui@0.11.0/dist/Space.js)

```javascript
render() {
  let transformedDivStyle = this.state.transformStyle;
  if (this.props.innerDivStyle) {
    transformedDivStyle = Object.assign(
      Object.assign(Object.assign({}, transformedDivStyle), this.props.innerDivStyle),
      { margin: 0 }
    );
  }
  
  return React.createElement(
    "div",
    {
      ref: this.setOuterDivRefAndCreateViewPort,
      id: this.props.id,
      className: `react-zoomable-ui-outer-div ${this.rootDivUniqueClassName} ${this.props.className || ''}`,
      style: this.props.style
    },
    React.createElement("style", null, this.constantStyles),
    this.state.contextValue && React.createElement(
      SpaceContext_1.SpaceContext.Provider,
      { value: this.state.contextValue },
      React.createElement(
        "div",
        {
          className: `react-zoomable-ui-inner-div ${this.props.innerDivClassName || ''}`,
          style: transformedDivStyle  // ← Transform 在这里！
        },
        this.props.children  // ← <Canvas> 作为 children 传入
      )
    )
  );
}
```

### 1.2 外层和内层 div 类名

| 层级 | 类名组成 | 说明 |
|------|---------|------|
| **外层 div** | `react-zoomable-ui-outer-div` + `{uniqueClassName}` + `props.className` | 固定前缀类名 + 随机唯一类名 + 用户传入类名 |
| **内层 div** | `react-zoomable-ui-inner-div` + `props.innerDivClassName` | 固定前缀类名 + 用户传入类名（已废弃） |

**在 JSONCrack 项目中**：

- [JSONCrackComponent.tsx:577](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L577) 传入 `className="jsoncrack-space"`
- 未传入 `innerDivClassName`

**最终渲染的类名**：
```
外层 div: "react-zoomable-ui-outer-div react-zoomable-ui-xxxxxx jsoncrack-space"
内层 div: "react-zoomable-ui-inner-div"
```

其中 `xxxxxx` 是 `generateRandomId()` 生成的随机字符串，每个 Space 实例唯一。

### 1.3 Transform 写法（关键！）

**源码位置**：[Space.js#L83-L91](https://unpkg.com/react-zoomable-ui@0.11.0/dist/Space.js)

```javascript
this.createTransformStyle = () => {
  if (this.viewPort) {
    return {
      transform: `scale(${this.viewPort.zoomFactor}) translate(${-1 * this.viewPort.left}px,${-1 * this.viewPort.top}px)`,
    };
  }
  return undefined;
};
```

**实际生成的 CSS transform**：
```css
transform: scale(0.8) translate(-100px, -50px);
```

**特点**：
- 使用 `scale(s)` + `translate(tx, ty)` 分开写法，**不是 `matrix()` 格式**
- `translate` 的值是**负的** `viewPort.left` 和 `viewPort.top`（因为视口向右下移动相当于内容向左上移动）
- `transform-origin: 0% 0%`（左上角），由内置样式设置

### 1.4 内置 constantStyles

**源码位置**：[Space.js#L54-L72](https://unpkg.com/react-zoomable-ui@0.11.0/dist/Space.js)

```javascript
this.constantStyles = `
.${this.rootDivUniqueClassName} {
  position: absolute;
  top: 0; bottom: 0; left: 0; right: 0;
  cursor: default;
}
.${this.rootDivUniqueClassName} > .react-zoomable-ui-inner-div {
  margin: 0; padding: 0; 
  transform-origin: 0% 0%;    /* ← 变换原点在左上角 */
  min-height: 100%;
  width: 100%;
}
`;
```

**关键**：`transform-origin: 0% 0%` 确保缩放和平移都以内容左上角为基准。

---

## 二、真实 DOM 层级（源码校准后）

### 2.1 完整 DOM 树（从外到内）

```
L1: <div ref={containerRef} className="canvasWrapper ...">      // 外层容器，不带 jsoncrack-canvas
  │
  └─ L2: <div className="react-zoomable-ui-outer-div react-zoomable-ui-xxxxxx jsoncrack-space">
        │  // Space 外层 div，位置: absolute; top:0; bottom:0; left:0; right:0;
        │
        └─ L3: <div className="react-zoomable-ui-inner-div"
                  style="transform: scale(0.8) translate(-100px, -50px); transform-origin: 0% 0%; ...">
              │  // Space 内层 div，transform 应用在这里！
              │  // 这是 Reaflow Canvas 的直接父元素
              │
              └─ **L4: <div className="jsoncrack-canvas">**                    ← 导出目标！
                    │  // Reaflow Canvas div，document.querySelector(".jsoncrack-canvas") 唯一命中
                    │
                    └─ L5: <svg width="2050" height="2050">                        // Reaflow SVG
                          │
                          ├─ <defs>...</defs>
                          │
                          └─ L6: <g transform="translate(0, 0)">                     // 内容组
                                │
                                ├─ <g id="node-1" transform="translate(100, 50)">      // 节点
                                │     └─ <foreignObject width="200" height="30">
                                │
                                └─ <g id="edge-1" transform="translate(0, 0)">         // 边
                                      └─ <path d="..." />
```

### 2.2 各层标识与类名

| 层级 | 元素 | 类名 / 标识 | 导出选择器是否命中 |
|------|------|-----------|-------------------|
| L1 | 外层 container div | `canvasWrapper`（CSS Modules 哈希） | ❌ 否 |
| L2 | Space 外层 div | `react-zoomable-ui-outer-div` + `jsoncrack-space` | ❌ 否 |
| **L3** | **Space 内层 div** | **`react-zoomable-ui-inner-div`** | **❌ 否**（Transform 在这里！） |
| **L4** | **Reaflow Canvas div** | **`jsoncrack-canvas`** | **✅ 是（唯一命中）** |
| L5 | Reaflow SVG | `svg`（无类名） | ❌ 否 |
| L6 | 内容组 `<g>` | （无类名） | ❌ 否 |

---

## 三、Transform 位置与 html-to-image 克隆边界

### 3.1 导出目标与克隆起点

**导出代码**：[DownloadModal/index.tsx:65-67](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L65-L67)

```typescript
const getExportElement = () =>
  (document.querySelector(".jsoncrack-canvas") as HTMLElement | null) ??
  (document.querySelector("svg[id*='ref']") as HTMLElement | null);
```

**克隆起点**：L4 `<div className="jsoncrack-canvas">`（Reaflow Canvas div）

### 3.2 html-to-image 克隆范围

```
cloneNode(true) 范围：
  ├─ ✅ L4: <div className="jsoncrack-canvas">          ← 起点
  │   ├─ ✅ L5: <svg width="2050" height="2050">
  │   │   ├─ ✅ <defs>
  │   │   └─ ✅ L6: <g transform="translate(0, 0)">
  │   │       ├─ ✅ <g id="node-1" transform="translate(100, 50)">
  │   │       │   └─ ✅ <foreignObject>
  │   │       └─ ✅ <g id="edge-1" transform="translate(0, 0)">
  │   │           └─ ✅ <path>
  │   └─ ✅ ...其他后代
  │
  └─ ❌ L3: <div className="react-zoomable-ui-inner-div">  ← 父级，不克隆
        style="transform: scale(0.8) translate(-100px, -50px)"
```

**关键结论**：
- L3（Space 内层 div）是 L4（导出目标）的**直接父元素**
- `cloneNode(true)` **只克隆节点及其后代，不克隆父级**
- 因此 L3 上的 `transform: scale(0.8) translate(-100px, -50px)` **不会进入导出**

### 3.3 边界图示

```
真实 DOM（屏幕上）：

  L1: <div.canvasWrapper>                           /* 不被克隆 */
    └─ L2: <div.react-zoomable-ui-outer-div>        /* 不被克隆 */
          └─ L3: <div.react-zoomable-ui-inner-div>  /* 不被克隆 — transform 在这里！ */
                style="transform: scale(0.8) translate(-100px, -50px)"
                ├─────────────────────────────────────────────┐
                │ L4: <div.jsoncrack-canvas>  ← 克隆边界起点 │
                │   └─ <svg>...                              │  全部被克隆
                └─────────────────────────────────────────────┘
```

---

## 四、哪些祖先层不会被克隆

### 4.1 完整祖先链（从 L4 向上）

```
导出目标 L4: <div.jsoncrack-canvas>
  ↑ 父级
L3: <div.react-zoomable-ui-inner-div>
    style="transform: scale(s) translate(tx, ty)"    ← ❌ 不克隆，带 Space transform
  ↑ 父级
L2: <div.react-zoomable-ui-outer-div.jsoncrack-space>
    position: absolute; top:0; bottom:0; left:0; right:0;  ← ❌ 不克隆
  ↑ 父级
L1: <div.canvasWrapper>
    position: relative; width: 100%; height: 100%  ← ❌ 不克隆
  ↑ 父级
... 更上层的应用容器（StyledEditorWrapper 等）  ← ❌ 不克隆
```

### 4.2 不被克隆的祖先层总结

| 层级 | 元素 | 不被克隆的原因 | 携带的关键信息 |
|------|------|---------------|---------------|
| L3 | `react-zoomable-ui-inner-div` | 是 L4 的父级 | **Space 的 transform**（缩放 + 平移） |
| L2 | `react-zoomable-ui-outer-div jsoncrack-space` | 是 L4 的祖先 | Space 外层定位样式（absolute 铺满） |
| L1 | `canvasWrapper` | 是 L4 的祖先 | 页面布局容器样式（背景、网格等） |
| 上层 | 应用容器 | 是 L4 的祖先 | 应用全局样式 |

**最重要的是 L3**：它携带了 Space 的 transform，这个 transform 完全不会进入导出结果。

---

## 五、Transform 类型与穿透性完整表

### 5.1 按位置分类

| Transform | 应用位置 | 相对导出目标 | 进入导出？ | 说明 |
|-----------|---------|------------|-----------|------|
| **❌ Space `scale(s) translate(tx, ty)`** | L3 `react-zoomable-ui-inner-div` | **父级（外部）** | **否** | 在 L4 的直接父元素上，不被克隆 |
| ✅ 内容组 `transform="translate(x,y)"` | L6 `<g>` | **后代（内部）** | **是** | 当前为 `translate(0,0)`（`defaultPosition={null}`） |
| ✅ 节点 `<g id="...">` `transform="translate(x,y)"` | 节点 `<g>` | **后代（内部）** | **是** | 来自 ELK 布局 |
| ✅ 边 `<g id="...">` `transform="translate(x,y)"` | 边 `<g>` | **后代（内部）** | **是** | 来自 ELK 布局 |
| ✅ `<foreignObject>` `x`/`y` 属性 | `foreignObject` | **后代（内部）** | **是** | 通常为 0,0 |

### 5.2 Space Transform 不进入导出的证据链

```
证据 1：Space 源码 render 结构
  → <div.outer-div>
       → <div.inner-div style={transformStyle}>  ← transform 在这里
           → {this.props.children}              ← 这是 <Canvas>，即 L4

证据 2：JSONCrack 中 Space 的使用
  <Space className="jsoncrack-space" ...>
    <Canvas className="jsoncrack-canvas" ... />  ← L4 是 Space 的 children
  </Space>

证据 3：DOM 层级关系
  L3.inner-div（带 transform）
    └─ L4.jsoncrack-canvas（导出目标）

证据 4：html-to-image 克隆机制
  cloneNode(true) 只克隆节点及其后代
  L4 的父级（L3）不在克隆范围内

结论：Space transform 在 L3，导出从 L4 开始 → 不进入导出 ✓
```

---

## 六、用户缩放操作对导出的实际影响

### 6.1 用户交互 → Space Transform 更新流程

```
用户鼠标滚轮 / 触摸手势
    ↓
react-zoomable-ui 内部事件处理
    ↓
更新 viewPort.zoomFactor、viewPort.left、viewPort.top
    ↓
调用 handleViewPortUpdated()
    ↓
createTransformStyle() 生成新的 transform
    ↓
this.setState({ transformStyle })
    ↓
React 重新渲染 L3.inner-div
    ↓
L3.style.transform = "scale(0.8) translate(-100px, -50px)"
```

### 6.2 对导出的影响

**屏幕视觉效果**：
- L3 应用 `scale(0.8) translate(-100px, -50px)`
- 内容看起来缩小到 80%，并偏移了 (100px, 50px)

**导出结果**：
- 克隆从 L4 开始，不包含 L3 的 transform
- L4.clientWidth = 2050, L4.clientHeight = 2050（不受 transform 影响）
- 导出 SVG viewBox = "0 0 2050 2050"
- 内容按原始尺寸 1:1 渲染，节点位置精确

**一句话总结**：**用户在界面上的缩放和平移操作，只改变屏幕视觉效果，完全不影响导出结果。**

---

## 七、源码校准后的完整决策链

```
用户点击导出
    ↓
getExportElement()
    ├─ document.querySelector(".jsoncrack-canvas")
    └─ → 命中 L4: <div className="jsoncrack-canvas">
    ↓
html-to-image toSvg/toPng
    ├─ 1. 克隆节点（从 L4 开始，cloneNode(true)）
    │   ├─ ✅ 克隆 L4 及其所有后代（svg、g、foreignObject 等）
    │   └─ ❌ 不克隆任何祖先（L3/L2/L1 全在边界外）
    │       └─ 关键：L3 的 transform: scale(s) translate(tx,ty) 不进入导出
    │
    ├─ 2. 内联计算样式
    │   └─ 对克隆树每个节点调用 getComputedStyle()
    │       注意：L3 的 transform 不在克隆树内，不会被内联
    │
    ├─ 3. 测量尺寸
    │   └─ L4.clientWidth × L4.clientHeight = paneWidth × paneHeight
    │       （Space transform 不影响 clientWidth）
    │
    ├─ 4. 构建 SVG 外壳
    │   └─ <svg viewBox="0 0 W H"> + <foreignObject width="100%" height="100%">
    │
    └─ 5. 序列化
    ↓
最终导出结果：
  - 尺寸：paneWidth × paneHeight（原始尺寸）
  - 内容：1:1 像素映射，无 Space 的缩放/偏移
  - 节点定位：完全按 ELK 布局结果
```

---

## 八、关键结论汇总

### 8.1 Space DOM 结构（源码确认）

1. **两层 div 结构**：
   - 外层：`react-zoomable-ui-outer-div` + 用户 className（`jsoncrack-space`）
   - 内层：`react-zoomable-ui-inner-div`（transform 应用在这里）

2. **Transform 写法**：
   ```css
   transform: scale(zoomFactor) translate(-left px, -top px);
   transform-origin: 0% 0%;
   ```
   不是 `matrix()` 格式，是 `scale()` + `translate()` 分开写。

3. **变换原点**：`transform-origin: 0% 0%`（左上角），由内置样式设置。

### 8.2 克隆边界（源码确认）

- **导出目标**：L4 `<div className="jsoncrack-canvas">`（Reaflow Canvas）
- **Space Transform 位置**：L3 `<div className="react-zoomable-ui-inner-div">`（L4 的直接父级）
- **克隆范围**：L4 及其后代，**不包括 L3**
- **结论**：**Space 的 transform 完全不会进入导出**

### 8.3 对导出的影响

| 项目 | 实际情况 |
|------|---------|
| 用户缩放影响导出尺寸 | ❌ 否，导出尺寸固定为 paneWidth × paneHeight |
| 用户平移影响导出内容位置 | ❌ 否，导出内容从原点开始 |
| 导出 SVG 包含 Space transform | ❌ 否，transform 在父级，不被克隆 |
| 导出内容是原始 1:1 尺寸 | ✅ 是，不受界面缩放影响 |

---

## 九、与之前分析的对比校准

| 项目 | 之前的推测 | 源码校准后 |
|------|----------|-----------|
| Space 渲染的 div 数量 | 推测两层（viewport + inner） | ✅ 确认两层（outer-div + inner-div） |
| 内层 div 类名 | 推测 `.space-inner` | ❌ 校准为 `react-zoomable-ui-inner-div` |
| Transform 格式 | 推测 `matrix(a,0,0,d,tx,ty)` | ❌ 校准为 `scale(s) translate(tx, ty)` |
| Transform 位置 | 确认在导出目标父级 | ✅ 确认在 L3（L4 的直接父级） |
| 是否进入导出 | 确认不进入 | ✅ 再次确认不进入 |
| 变换原点 | 未提及 | ✅ 确认 `transform-origin: 0% 0%` |
