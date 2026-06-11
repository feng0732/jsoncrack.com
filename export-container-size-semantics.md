# 导出尺寸语义：Reaflow Canvas 容器与 SVG 包裹层关系分析

本文档梳理 JSONCrack 导出功能中，从目标元素选择、容器层级结构、尺寸决定来源到 foreignObject 缩放语义的完整链路。

---

## 1. 导出目标元素选择逻辑

导出入口位于 [DownloadModal/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx)。

```typescript
const getExportElement = () =>
  (document.querySelector(".jsoncrack-canvas") as HTMLElement | null) ??
  (document.querySelector("svg[id*='ref']") as HTMLElement | null);
```

**选择优先级：**
1. **首选**：`.jsoncrack-canvas` —— 这是 Reaflow `Canvas` 组件渲染出的外层 DOM 容器（一个 `<div>`），内部包裹着真正的 `<svg>` 元素。
2. **备选**：`svg[id*='ref']` —— 如果找不到 canvas div，则直接查找任意 id 含 "ref" 的 svg 元素作为兜底。

然后将该 HTMLElement 传入 `html-to-image` 库的 `toPng` / `toJpeg` / `toSvg` / `toBlob` 方法进行栅格化或序列化。

---

## 2. DOM / SVG 层级结构关系

从外到内的完整 DOM 树如下（仅列关键节点）：

```
编辑器页面最外层
└── <StyledEditorWrapper>          width:100%; height:100%  (styled-components, GraphView 内)
    └── <JSONCrack> 组件根
        └── <div.canvasWrapper>    position:relative; width:100%; height:100%
            └── <Space>            className="jsoncrack-space"   (react-zoomable-ui 提供视口缩放)
                └── <Canvas>       className="jsoncrack-canvas"  (Reaflow Canvas 组件)
                    └── <svg>      ← 真正的 SVG 根元素，宽高由 width/height/maxWidth/maxHeight props 决定
                        └── <g>    ← 内容组（Reaflow 渲染的节点和边的容器）
                            ├── <g id="..."> 节点1 (含 <foreignObject>)
                            ├── <g id="..."> 节点2 (含 <foreignObject>)
                            └── <path> 边
```

### 各层职责与代码定位

| 层级 | 类名 / 标签 | 定义位置 | 职责 |
|------|------------|----------|------|
| 样式包装层 | `StyledEditorWrapper` | [GraphView/index.tsx:15-44](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/apps/www/src/features/editor/views/GraphView/index.tsx#L15-L44) | 提供 100% 宽高、自定义鼠标光标、节点圆角阴影等全局样式覆盖 |
| Canvas 包装层 | `div.canvasWrapper` | [JSONCrackComponent.tsx:537-614](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L537-L614) 配合 [JSONCrackStyles.module.css:1-6](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackStyles.module.css#L1-L6) | 相对定位容器，控制 100% 占比，承载主题 CSS 变量（背景色、网格等） |
| 缩放视口层 | `Space.jsoncrack-space` | [JSONCrackComponent.tsx:570-583](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L570-L583) | `react-zoomable-ui` 的 `Space` 组件，提供可平移缩放的虚拟坐标系 |
| Reaflow Canvas 容器 | `div.jsoncrack-canvas` | [JSONCrackComponent.tsx:585-611](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L585-L611) | Reaflow 库的 `Canvas` 组件渲染出的外层 div，内部包含 `<svg>` |
| SVG 根元素 | `<svg>` | Reaflow 内部渲染 | 实际承载图形的 SVG 根元素，其 viewport 尺寸由传入 Canvas 的 props 决定 |
| 内容组 | `<g>` (svg > g) | Reaflow 内部渲染 | 所有节点 `<g>` 和边 `<path>` 的父容器，受 transform 控制位置 |

---

## 3. 宽高尺寸的决定来源

目标元素 `.jsoncrack-canvas` 及其内部 `<svg>` 的尺寸并非由单一来源决定，而是一条清晰的"布局 → 状态 → props → DOM"数据流。

### 3.1 数据流全景

```
parseGraph (计算每个节点的 width/height)
        │
        ▼
   ELK 布局引擎 (Reaflow 内部)
        │  输出 ElkRoot { width, height, ... }
        ▼
onLayoutChange 回调
        │  layoutSizeRef.current = { width, height }
        │  setPaneWidth(layout.width + 50)
        │  setPaneHeight(layout.height + 50)
        ▼
<Canvas width={paneWidth} height={paneHeight}
        maxWidth={paneWidth} maxHeight={paneHeight} />
        │
        ▼
Reaflow 渲染 <svg width="..." height="...">
```

### 3.2 各阶段详解

**阶段 1：节点预计算（parseGraph 阶段）**

在 [parser.ts](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/parser.ts) 中调用 [calculateNodeSize](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/utils/calculateNodeSize.ts) 为每个节点预先计算像素尺寸：

- 通过创建一个 `visibility:hidden` 的临时 `<div>`，设置与真实节点相同的字体（12px monospace）、padding（0 10px）、whiteSpace 等样式
- 调用 `getBoundingClientRect()` 测量文本宽度
- 高度：单行节点固定 36px (`PARENT_HEIGHT`)，多行节点 = 行数 × 30px (`ROW_HEIGHT`)
- 宽度上限 700px，下限 45px
- 父节点（含子对象/数组的节点）额外加宽 80px

这些 `width` / `height` 写入 `NodeData`，成为 ELK 布局的输入约束。

**阶段 2：ELK 布局计算整体画布尺寸**

Reaflow 内部使用 ELK (Eclipse Layout Kernel) 算法。布局完成后触发 `onLayoutChange` 回调，传入 `ElkRoot` 对象：

```typescript
// JSONCrackComponent.tsx:401-411
const onLayoutChange = useCallback((layout: ElkRoot) => {
  if (!layout.width || !layout.height) {
    setLoading(false);
    return;
  }

  layoutSizeRef.current = { width: layout.width, height: layout.height };
  setPaneWidth(layout.width + 50);   // 加 50px 留白
  setPaneHeight(layout.height + 50); // 加 50px 留白
  setLoading(false);
}, []);
```

**关键**：`ElkRoot.width` / `ElkRoot.height` 是 ELK 布局器根据所有节点位置和尺寸计算出的**整体包围盒**，即布局坐标系中所有节点能完整容纳的最小矩形。

**阶段 3：状态驱动 Canvas props**

`paneWidth` / `paneHeight` 作为 React state，同时传给 Reaflow `Canvas` 的四个属性：

```tsx
// JSONCrackComponent.tsx:593-596
maxHeight={paneHeight}
maxWidth={paneWidth}
height={paneHeight}
width={paneWidth}
```

初始值为 `2000 × 2000`（见 [JSONCrackComponent.tsx:146-147](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L146-L147)），首次布局完成后被 ELK 结果覆盖。

**阶段 4：外层容器样式的影响**

| 层级 | 尺寸规则 | 是否影响导出尺寸 |
|------|---------|----------------|
| `StyledEditorWrapper` (GraphView) | `width:100%; height:100%` | **间接影响**：决定了可视视口大小，但不影响 svg 的内在 width/height 属性 |
| `div.canvasWrapper` | `position:relative; width:100%; height:100%` | **间接影响**：同上 |
| `Space` (react-zoomable-ui) | 自适应容器大小，内部维护虚拟坐标 | **不影响**：Space 只做视口缩放变换，svg 本身的 width/height 不变 |
| `div.jsoncrack-canvas` (Reaflow Canvas div) | 由 Reaflow 内部样式控制，通常是 `position:relative; overflow:hidden` | **不影响**：svg 内在尺寸由 props 决定 |
| `<svg>` (实际 SVG 根) | `width={paneWidth} height={paneHeight}` | **直接决定**：这就是 html-to-image 测量和导出的基础尺寸 |

**结论**：导出图片的**内在像素尺寸**完全由 ELK 布局结果 + 50px 留白决定（即 `paneWidth × paneHeight`），外层容器的 100% 宽高只控制屏幕可视区域，不影响导出文件的分辨率。

### 3.3 辅助验证：computeGraphClientRect 的测量降级链

[canvasHelpers.ts:163-221](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L163-L221) 中 `computeGraphClientRect` 函数展示了四层测量策略，从侧面印证了尺寸来源的优先级：

1. **最高优先级**：union 所有 `g[id]` 叶子节点（节点和边）的 `getBoundingClientRect()` —— 真实渲染结果
2. **次优先**：内容组 `<g>` 自身的 `getBoundingClientRect()`
3. **再次**：`getBBox()` + `getScreenCTM()` 投影 —— 布局几何
4. **最后兜底**：`layoutSize` (ELK 结果) + svg `getScreenCTM()` 投影

---

## 4. foreignObject 包裹后的 SVG 缩放语义

### 4.1 foreignObject 的使用方式

两种节点类型都使用 `<foreignObject>` 将 HTML 内容嵌入 SVG：

**ObjectNode** ([ObjectNode.tsx:95-113](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/components/ObjectNode.tsx#L95-L113))：
```tsx
<foreignObject
  className={`${styles.foreignObject} ${styles.objectForeignObject}`}
  data-id={`node-${node.id}`}
  width={node.width}
  height={node.height}
  x={0}
  y={0}
>
  {node.text.map((row, index) => <Row ... />)}
</foreignObject>
```

**TextNode** ([TextNode.tsx:22-40](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/components/TextNode.tsx#L22-L40))：
```tsx
<foreignObject
  className={styles.foreignObject}
  data-id={`node-${node.id}`}
  width={width}
  height={height}
  x={0}
  y={0}
>
  <span className={styles.textNodeWrapper} ...>
    <span className={styles.key}><TextRenderer>{value}</TextRenderer></span>
  </span>
</foreignObject>
```

### 4.2 尺寸映射关系

```
┌─────────────────────────────────────────────────────┐
│  <svg width={paneWidth} height={paneHeight}>        │  SVG 视口（由 ELK 决定）
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │  <g transform="translate(x,y)">               │  │  节点组（Reaflow 定位）
│  │                                               │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │  <foreignObject                        │  │  │
│  │  │    width={node.width}   ← 固定像素值   │  │  │
│  │  │    height={node.height} ← 固定像素值   │  │  │
│  │  │    x=0, y=0                            │  │  │
│  │  │                                         │  │  │
│  │  │  ┌─────────────────────────────────┐    │  │  │
│  │  │  │  HTML 内容 (span 等)            │    │  │  │
│  │  │  │  - 12px monospace               │    │  │  │
│  │  │  │  - 行高 30px                    │    │  │  │
│  │  │  │  - padding 由 CSS 控制           │    │  │  │
│  │  │  └─────────────────────────────────┘    │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 4.3 缩放语义详解

#### 情况 A：用户在界面上缩放（react-zoomable-ui Space 层）

`Space` 组件通过 CSS `transform: scale()` 作用于内部内容。这个变换发生在 `div.jsoncrack-canvas` 这一层之上（Space 是 Canvas 的父容器）。

- `<svg>` 的 `width` / `height` 属性值**不改变**（仍是 paneWidth × paneHeight）
- `<foreignObject>` 的 `width` / `height` 属性值**不改变**（仍是 node.width × node.height）
- 屏幕上看到的视觉大小变化来自外层 CSS transform，属于**显示缩放**
- **导出时**：html-to-image 直接抓取 DOM，获取的是 svg 原始 width/height（即 paneWidth × paneHeight），不受当前界面缩放级别影响

#### 情况 B：SVG 本身作为矢量缩放（例如导出 SVG 后在浏览器中打开）

`<foreignObject>` 的 width/height 是 SVG 用户坐标系中的长度值（无单位时视为像素）。当 SVG 被整体缩放时（例如通过 CSS 设置 `svg { width: 100% }`）：

- foreignObject 的**逻辑尺寸**随 SVG viewBox 一起缩放
- 但 foreignObject **内部的 HTML 内容**遵循 HTML 渲染规则：
  - 字体大小（12px）是 CSS 像素，需要乘以 SVG 的缩放比例
  - 边框、padding 等也是如此
  - 如果 SVG 被缩放到很大，HTML 文字会出现模糊（因为 foreignObject 内部光栅化）
  - 如果 SVG 被缩小，HTML 元素可能出现截断或布局错乱

#### 情况 C：html-to-image 的 toSvg() 序列化

`html-to-image` 的 `toSvg()` 会将目标元素（`.jsoncrack-canvas` div）**整体**序列化为一个新的 SVG：
1. 先通过测量获取目标元素的布局尺寸（getBoundingClientRect 或 offsetWidth/Height）
2. 将所有子元素（包括原 `<svg>`、原 `<foreignObject>` 内的 HTML）重新绘制到一个新的 `<svg>` 中
3. 原 `<foreignObject>` 内的 HTML 会被**光栅化**或转换为 SVG 文本/形状（取决于 html-to-image 的实现）

这意味着：导出 SVG 时，原有的 foreignObject 语义可能已经丢失，变成了纯 SVG 元素或内联图像。

### 4.4 关键样式对 foreignObject 内容的约束

[Node.module.css:1-12](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/components/Node.module.css#L1-L12)：

```css
.foreignObject {
  text-align: center;
  color: var(--node-text);
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
  font-size: 12px;
  font-weight: 500;
  overflow: hidden;       /* ← 超出 foreignObject 边界的内容被裁剪 */
  pointer-events: none;
  border-radius: 5px;
}
```

**`overflow: hidden` 的重要性**：确保即便 HTML 内容实际尺寸超过 foreignObject 的 width/height（例如字体加载前后差异），也不会溢出节点矩形边界，从而保证导出图片的节点尺寸与布局计算一致。

---

## 5. 总结：导出尺寸决策链

```
用户点击导出
    │
    ▼
getExportElement()  →  获取 div.jsoncrack-canvas
    │
    ▼
html-to-image 读取目标元素尺寸
    │
    │  div.jsoncrack-canvas 的 offsetWidth/Height
    │  = 其内部 <svg> 的 width/height 属性值
    │  = paneWidth × paneHeight
    │  = (ElkRoot.width + 50) × (ElkRoot.height + 50)
    │
    ▼
栅格化 / 序列化
    │
    ├─ PNG/JPEG：按 svg width/height 像素渲染（1:1），无额外缩放
    │            foreignObject 内 HTML 以 12px 字号光栅化
    │
    └─ SVG：html-to-image 将整体重新序列化为新 SVG
             原有 foreignObject 可能被转换为纯 SVG 元素
```

**最终结论**：
- 导出图片的**画布尺寸**由 ELK 布局结果（所有节点包围盒）+ 50px 留白决定，存储在 `paneWidth` / `paneHeight` state 中
- 外层所有 100% 宽高的容器（StyledEditorWrapper、canvasWrapper、Space）只控制可视视口，**不影响导出分辨率**
- `<foreignObject>` 的 width/height 与节点在 parse 阶段预计算的尺寸一致，且内部 HTML 使用 `overflow:hidden` 保证不溢出
- 用户在界面上的缩放操作（Space 层的 CSS transform）**不影响**导出尺寸
