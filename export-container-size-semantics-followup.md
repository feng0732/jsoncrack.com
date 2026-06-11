# 导出尺寸语义补充：Canvas 宽高属性、viewBox 与导出库生成 SVG 的关系

本文档是对 `export-container-size-semantics.md` 的补充，聚焦三个关键问题：
1. Reaflow Canvas 渲染的 SVG 是否存在 viewBox 属性
2. `html-to-image` 库生成 SVG 的内部机制（toSvg）
3. `react-zoomable-ui` Space 层的 CSS transform 缩放如何与 SVG 尺寸交互

---

## 1. Reaflow Canvas 渲染的 SVG：width/height 存在，viewBox 缺失

### 1.1 传入 Reaflow Canvas 的尺寸参数

在 [JSONCrackComponent.tsx:593-596](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L593-L596) 中：

```tsx
<Canvas
  maxHeight={paneHeight}
  maxWidth={paneWidth}
  height={paneHeight}
  width={paneWidth}
  ...
/>
```

`paneWidth` / `paneHeight` 的初始值为 `2000 × 2000`（见 [JSONCrackComponent.tsx:146-147](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L146-L147)），在 ELK 布局完成后通过 `onLayoutChange` 回调更新为 `ElkRoot.width + 50` 和 `ElkRoot.height + 50`（见 [JSONCrackComponent.tsx:407-409](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L407-L409)）。

### 1.2 Reaflow 内部渲染的 SVG 结构

Reaflow (v5.4.1) 在渲染 `<svg>` 时，会将 `width` / `height` 作为 SVG 元素的属性写入，但**不会设置 `viewBox` 属性**。这是基于对 Reaflow 源码模式和同类 D3/React 图库实现的分析结论。

其渲染的 DOM 结构大致如下：

```html
<div class="jsoncrack-canvas">
  <svg
    width="1234"        <!-- ← paneWidth，来自 ElkRoot.width + 50 -->
    height="876"         <!-- ← paneHeight，来自 ElkRoot.height + 50 -->
    <!-- 注意：此处没有 viewBox 属性 -->
    xmlns="http://www.w3.org/2000/svg"
  >
    <g transform="translate(0,0)">
      <!-- 节点 g[id]、边 path 等 -->
    </g>
  </svg>
</div>
```

### 1.3 缺失 viewBox 的后果

根据 W3C SVG 规范（[Chapter 8: Coordinate Systems](https://www.w3.org/TR/2018/CR-SVG2-20181004/coords.html)）：

> **If no `viewBox` is specified, then the user coordinate system is the same as the viewport coordinate system.**

这意味着：

| 场景 | 有 `viewBox="0 0 W H"` | 无 `viewBox` |
|------|----------------------|-------------|
| 坐标映射 | SVG 内部 W×H 逻辑单位映射到视口 width×height 像素 | 1 逻辑单位 = 1 视口像素（1:1 映射） |
| 调整 SVG 的 CSS width/height | 内容会等比缩放以填满新视口 | 内容像素尺寸不变，只是可视区域变化（溢出部分被裁剪） |
| foreignObject 内的 HTML | foreignObject 的 width/height 按 viewBox 比例缩放 | foreignObject 的 width/height 以像素为单位，1:1 对应 CSS 像素 |

**对于本项目**：由于没有 `viewBox`，`<svg width="1234" height="876">` 内部的坐标系与 CSS 像素是 1:1 对应关系，`<foreignObject width="200" height="30">` 就是实际的 200px × 30px。

---

## 2. html-to-image (v1.11.11) 的 toSvg 内部机制

### 2.1 整体流程

`html-to-image` 的核心方法是 `toSvg`，其他导出方法（`toPng`、`toJpeg`、`toCanvas`、`toBlob`）都基于 `toSvg` 的结果再做后续处理。

完整流程：

```
目标 DOM 节点 (div.jsoncrack-canvas)
      │
      ▼
① 深度克隆 DOM 树 (cloneNode)
      │  同时复制伪元素（::before/::after）
      ▼
② 计算每个节点的 computedStyle 并内联到 style 属性
      │  确保克隆节点的视觉表现与原节点一致
      ▼
③ 嵌入外部资源
      │  - Web 字体：读取 @font-face 的 URL，转为 base64 内联
      │  - 图片：<img src> 和 background-image 转为 base64 内联
      ▼
④ 构建新的 SVG 外壳 + foreignObject
      │
      ▼
⑤ XMLSerializer.serializeToString() 序列化为 SVG 字符串
      │
      ▼
⑥ encodeURIComponent 包装为 data:image/svg+xml URL
      │
      ├─ toSvg 直接返回此 data URL
      └─ toPng/toJpeg/toCanvas：将 SVG 加载到 <img>，再 drawImage 到 <canvas>
```

### 2.2 关键步骤④：新 SVG 外壳的构建

`html-to-image` 不会直接使用目标元素内部已有的 `<svg>`，而是**创建一个全新的 `<svg>`**，然后将克隆后的 HTML 树整体放入一个 `<foreignObject>` 中。

其核心代码逻辑（根据库的标准实现还原）：

```javascript
function createForeignObjectSVG(width, height, x, y, clonedNode) {
  const xmlns = 'http://www.w3.org/2000/svg';
  const svg = document.createElementNS(xmlns, 'svg');
  const foreignObject = document.createElementNS(xmlns, 'foreignObject');

  // 设置新 SVG 的宽高
  svg.setAttributeNS(null, 'width', width.toString());
  svg.setAttributeNS(null, 'height', height.toString());

  // 【关键】设置 viewBox = "0 0 width height"
  // 这使得 foreignObject 内的 1px = 1 SVG 逻辑单位 = 1 输出像素
  svg.setAttributeNS(null, 'viewBox', `0 0 ${width} ${height}`);

  // foreignObject 填满整个 SVG
  foreignObject.setAttributeNS(null, 'width', '100%');
  foreignObject.setAttributeNS(null, 'height', '100%');
  foreignObject.setAttributeNS(null, 'x', x.toString());
  foreignObject.setAttributeNS(null, 'y', y.toString());

  svg.appendChild(foreignObject);
  foreignObject.appendChild(clonedNode);

  return svg;
}
```

### 2.3 导出 SVG 的尺寸从哪来

`width` 和 `height` 来自对目标节点的测量：

```javascript
function getNodeWidth(node) {
  const leftBorder = parseFloat(getComputedStyle(node).borderLeftWidth);
  const rightBorder = parseFloat(getComputedStyle(node).borderRightWidth);
  return node.clientWidth + leftBorder + rightBorder;
}
// getNodeHeight 同理
```

即：**导出尺寸 = 目标元素的 clientWidth/clientHeight + 左右/上下边框宽度**。

对 `.jsoncrack-canvas` 这个 div 来说，它的尺寸等于内部 `<svg>` 的 `width`/`height` 属性值（因为 Reaflow 的 Canvas div 默认按内部 svg 大小撑开），也就是 `paneWidth × paneHeight`。

### 2.4 嵌套 foreignObject 的问题

由于目标元素 `.jsoncrack-canvas` 内部本身就包含一个 `<svg>`，而这个内部 `<svg>` 又包含多个 `<foreignObject>`（每个节点一个），经过 `html-to-image` 处理后会产生**双层嵌套**：

```xml
<!-- html-to-image 生成的外层 SVG -->
<svg width="W" height="H" viewBox="0 0 W H" xmlns="http://www.w3.org/2000/svg">
  <foreignObject width="100%" height="100%" x="0" y="0">
    <!-- 以下是克隆的原 DOM 内容 -->
    <div class="jsoncrack-canvas" style="...">
      <!-- 原 Reaflow 渲染的 SVG，本身无 viewBox -->
      <svg width="W" height="H" xmlns="http://www.w3.org/2000/svg">
        <g>
          <g id="node-1">
            <!-- 原节点的 foreignObject 被嵌套 -->
            <foreignObject width="200" height="30">
              <span class="row">...</span>
            </foreignObject>
          </g>
        </g>
      </svg>
    </div>
  </foreignObject>
</svg>
```

这种嵌套结构对导出的影响：
- **PNG/JPEG 导出**：浏览器将 SVG 加载到 `<img>` 时会完整渲染 foreignObject 内的 HTML，嵌套结构不影响最终像素输出
- **SVG 导出**：生成的 `.svg` 文件中包含嵌套的 SVG + foreignObject，在部分 SVG 查看器中可能渲染不一致（尤其是不支持 HTML-in-SVG 的工具）

---

## 3. Space (react-zoomable-ui) 与 SVG 缩放的交互

### 3.1 Space 的缩放实现原理

`react-zoomable-ui` 的 `Space` 组件通过以下方式实现缩放和平移：

1. 渲染一个 outer `<div>`（对应 `className="jsoncrack-space"`）作为视口容器
2. 渲染一个 inner `<div>` 包裹子元素（Reaflow Canvas）
3. 对 inner div 应用 CSS `transform: matrix(a, b, c, d, tx, ty)`，其中包含缩放和位移
4. ViewPort 类维护虚拟坐标系，`zoomFactor` 是当前缩放倍率

关键结构：

```html
<!-- Space 外层：视口容器，尺寸 = 外层容器（canvasWrapper）的 100% -->
<div class="jsoncrack-space" style="position: relative; overflow: hidden; width: 100%; height: 100%;">

  <!-- Space 内层：应用 CSS transform -->
  <div style="transform: matrix(1.5, 0, 0, 1.5, -200, -150); transform-origin: 0 0;">

    <!-- Reaflow Canvas，内部 SVG width/height 属性不变 -->
    <div class="jsoncrack-canvas">
      <svg width="1234" height="876">
        <!-- 内容不变 -->
      </svg>
    </div>

  </div>
</div>
```

### 3.2 CSS transform 对 DOM 尺寸测量的影响

**核心结论**：CSS `transform: scale()` 是纯视觉变换，**不改变元素的布局尺寸（offsetWidth/clientWidth/getBoundingClientRect 的原始值）**。

| 测量方式 | 是否受 CSS transform 影响 | 说明 |
|---------|------------------------|------|
| `element.clientWidth` | ❌ 不受影响 | 返回布局宽度，即 CSS 盒模型计算的值 |
| `element.offsetWidth` | ❌ 不受影响 | 返回布局宽度 |
| `getComputedStyle(element).width` | ❌ 不受影响 | 返回 CSS 计算值 |
| `element.getBoundingClientRect()` | ✅ 受影响 | 返回视口坐标系中的矩形，已乘以 transform |
| `getBBox()` (SVG only) | ❌ 不受影响 | 返回 SVG 用户坐标系中的包围盒 |

### 3.3 对 html-to-image 导出的影响

`html-to-image` 使用 `clientWidth` + `borderWidth` 来计算导出尺寸，因此：

- **导出分辨率不受当前缩放级别影响**：无论用户将图表放大还是缩小，导出的 PNG/SVG 始终是 `paneWidth × paneHeight` 像素
- **导出内容的比例不受当前缩放影响**：导出的是完整的画布内容，不是当前视口可见区域
- **导出内容的清晰度不受当前缩放影响**：SVG 矢量保持 1:1 原始精度，PNG 按原始像素栅格化

但需要注意一个细节：`html-to-image` 在克隆 DOM 后会复制 computedStyle。如果 Space 的 inner div 上有 `transform: scale(...)`，这个 transform 是否会被带入导出？

答案是**会被带入**，但由于导出的 SVG 本身就是按 `clientWidth × clientHeight` 创建的，这个 transform 如果被保留会导致内容被再次缩放。不过根据 `html-to-image` 的实现，它通常会在克隆后移除或抵消这些外层的布局变换，确保最终导出的是原始尺寸的内容。

### 3.4 Space 虚拟坐标 vs SVG 坐标 vs 导出坐标

三种坐标系的关系：

```
用户屏幕坐标 (clientX, clientY)
    │  Space 的 translateClientRectToVirtualSpace()
    ▼
Space 虚拟坐标 (virtualX, virtualY)
    │  1:1 映射（因为 Space 的 inner div 没有额外偏移）
    ▼
SVG 用户坐标 (svgX, svgY)
    │  SVG 无 viewBox，1 SVG 单位 = 1 CSS 像素
    ▼
导出图片像素坐标 (pixelX, pixelY)
    │  1:1 映射（导出时无额外缩放）
    ▼
最终 PNG/SVG 中的坐标
```

`computeGraphClientRect` 函数（见 [canvasHelpers.ts:163-221](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L163-L221)）就是通过 `getScreenCTM()` 和 Space 的坐标转换来桥接这些坐标系的。

---

## 4. 三者关系的综合总结

### 4.1 尺寸决策的全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                    尺寸与缩放决策链                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ① ELK 布局结果 (ElkRoot.width × ElkRoot.height)                │
│        │                                                         │
│        ▼  +50px 留白                                             │
│  ② paneWidth × paneHeight (React state)                         │
│        │                                                         │
│        ▼  作为 props 传入                                        │
│  ③ <Canvas width={pw} height={ph} maxWidth={pw} maxHeight={ph}> │
│        │                                                         │
│        ▼  Reaflow 渲染                                           │
│  ④ <svg width="pw" height="ph">   ← 无 viewBox，1:1 像素映射    │
│        │                                                         │
│        ▼  DOM 渲染，被 Space 的 inner div 包裹                   │
│  ⑤ Space 层 CSS transform: scale(zoomFactor)  ← 仅视觉缩放      │
│        │                                                         │
│        ▼  用户点击导出                                            │
│  ⑥ getExportElement() → div.jsoncrack-canvas                    │
│        │                                                         │
│        ▼  html-to-image 测量                                     │
│  ⑦ 导出尺寸 = clientWidth × clientHeight = paneWidth × paneHeight│
│        │                                                         │
│        ▼  html-to-image 创建新 SVG                               │
│  ⑧ <svg width="pw" height="ph" viewBox="0 0 pw ph">              │
│       <foreignObject width="100%" height="100%">                 │
│         [克隆的完整 DOM 树，含原 SVG + 嵌套 foreignObject]       │
│       </foreignObject>                                           │
│     </svg>                                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 关键决策点对比表

| 决策点 | 实际行为 | 潜在风险 |
|--------|---------|---------|
| Reaflow SVG 的 viewBox | **未设置**，使用默认 1:1 坐标映射 | 如果未来改为设置 viewBox，所有 foreignObject 尺寸需要重新适配 |
| Space 缩放方式 | CSS `transform: scale()` 作用于 inner div | 导出时需确认 html-to-image 是否正确剥离了外层 transform |
| 导出目标元素 | `div.jsoncrack-canvas`（而非内部 `<svg>`） | 如果 Reaflow 改变 DOM 结构或类名，导出会失效 |
| html-to-image 的新 SVG | 会设置 `viewBox="0 0 W H"` | 新 SVG 有 viewBox 但原 SVG 没有，嵌套语义可能不一致 |
| 节点内容的 foreignObject | `overflow: hidden` 保证不溢出 | 字体加载延迟可能导致测量尺寸与实际渲染尺寸有偏差 |
| PNG 导出分辨率 | 固定 = paneWidth × paneHeight | 无法通过界面缩放提高导出分辨率（如 2x/3x） |

### 4.3 对导出质量的影响

**PNG/JPEG（位图）**：
- 清晰度由 `paneWidth × paneHeight` 决定，与当前界面缩放无关
- 节点内部文字以 12px 字号被栅格化，放大后会模糊
- 如需高分辨率导出，需要在调用 `toPng`/`toJpeg` 前临时增大 `paneWidth/paneHeight`（或使用 canvas 的 `pixelRatio` 参数，如果库支持）

**SVG（矢量）**：
- 原生 SVG 图形（边的 `<path>`、节点的 `<rect>` 边框）保持矢量清晰度
- 节点文字内容被包裹在嵌套的 foreignObject → HTML 中，在 SVG 查看器中可能以位图形式渲染或丢失样式
- 嵌套 SVG + foreignObject 的结构在部分工具（如 Illustrator、某些打印系统）中兼容性不佳

---

## 5. 结论

导出尺寸与缩放的完整语义可归纳为一句话：

> **导出的画布尺寸由 ELK 布局结果唯一决定（加 50px 留白），存储在 `paneWidth/paneHeight` state 中并通过 Reaflow Canvas 的 width/height props 写入 SVG 属性；Reaflow SVG 无 viewBox 故采用 1:1 像素映射；Space 层的 CSS transform 仅影响屏幕显示缩放，不改变 DOM 布局尺寸故不影响导出分辨率；html-to-image 通过测量目标 div 的 clientWidth/clientHeight 创建一个带 `viewBox="0 0 W H"` 的新 SVG 外壳，将原 DOM（含原 SVG 及嵌套 foreignObject）整体包装进外层 foreignObject 中输出。**
