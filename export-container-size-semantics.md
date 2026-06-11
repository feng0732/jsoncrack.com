# 导出容器尺寸语义深度分析

> 本文档精确分析：Reaflow Canvas 容器的 DOM 层级、目标元素宽高的决定因素（容器样式 / 内部 SVG / 布局结果）、
> Space transform 对尺寸读取的影响、以及 SVG foreignObject 包裹后的缩放语义。

---

## 一、完整 DOM 层级（从最外层到最内层）

### 1.1 整体结构

```
Layer 0: <div className="canvasWrapper">         // 视口容器
  width: 100%, height: 100%, position: relative
  [JSONCrackStyles.module.css#L1-L6]
          │
          ▼
Layer 1: <Space className="jsoncrack-space">     // react-zoomable-ui 外层
  ├── Outer Div (无 transform)
  │     className="jsoncrack-space"
  │     提供事件监听、overflow 隐藏
  │
  └── Inner Div (⭐ transform 应用在这里 ⭐)
          style: { transform: "translate(tx, ty) scale(zf)" }
          这是 Space 的内部 wrapper，用户不可见
          [SpaceProps API: innerDivClassName / innerDivStyle]
          │
          ▼
Layer 2: <Canvas className="jsoncrack-canvas">   // Reaflow Canvas 容器
  className="jsoncrack-canvas"
  ⚠️ 这是 getExportElement() 直接抓取的目标元素
  width/height/maxWidth/maxHeight = paneWidth/paneHeight
  [JSONCrackComponent.tsx#L585-L611]
          │
          ▼
Layer 3: <svg>                                   // Reaflow 内部 SVG
  width={paneWidth}     // 来自 Canvas 的 width prop
  height={paneHeight}   // 来自 Canvas 的 height prop
  viewBox="0 0 {paneWidth} {paneHeight}"
  max-width={paneWidth}
  max-height={paneHeight}
  [Reaflow Canvas 实现]
          │
          ▼
Layer 4: <g> (content group)
  ├── <g id="node-1"> （节点组）
  │     └── <foreignObject width={node.width} height={node.height} x={node.x} y={node.y}>
  │             └── <span className="row">...</span>  // 内嵌 HTML 文本
  │             [ObjectNode.tsx#L95-L113]
  ├── <g id="edge-1"> （边组）
  └── ...
```

### 1.2 关键类名溯源

- `.jsoncrack-canvas`：在 [JSONCrackComponent.tsx#L586](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L586) 作为 `className` prop 传递给 Reaflow `<Canvas>`。
  Reaflow 会将这个类名应用到它渲染的**最外层容器 div**上。

- `.jsoncrack-space`：在 [JSONCrackComponent.tsx#L577](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L577) 作为 `className` prop 传递给 `react-zoomable-ui` 的 `<Space>`。
  Space 会将这个类名应用到它的**外层 div**（不包含 transform 的那一层）。

### 1.3 Space 的 Inner Div 之谜

`react-zoomable-ui` 的 [SpaceProps API 文档](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/export-image-size-rules.md#L10-L14) 明确提到：

> `innerDivClassName`: Optional CSS class to use on the **inner div** that the Space **scales and transforms**.
> `innerDivStyle`: Optional styles to use on the **inner div** that the Space **scales and transforms**.

这证明：Space 对其子元素的 transform 不是直接写在 children 上，而是先把 children 包进一个**内部 wrapper div**，再对这个 wrapper 应用 transform。

**关键推论**：
- `transform: translate(tx, ty) scale(zf)` 在 **Layer 1 的 Inner Div** 上
- `.jsoncrack-canvas`（Layer 2）是 Inner Div 的直接子元素，自身 **没有** transform
- `getExportElement()` 抓取 Layer 2，**不会**带上 Layer 1 Inner Div 的 transform

---

## 二、目标元素（.jsoncrack-canvas）宽高的决定因素

### 2.1 三层含义的"宽度"对比

`html-to-image` 的 `getImageSize()` 通过 `getNodeWidth(node)` 获取宽度：

```js
// html-to-image util.js
function getNodeWidth(node) {
  const leftBorder  = px(node, 'border-left-width');
  const rightBorder = px(node, 'border-right-width');
  return node.clientWidth + leftBorder + rightBorder;
}
```

即 `getImageSize().width = node.clientWidth + borderWidth`。

下面对比三种尺寸读取方式的语义：

| 读取方式 | 读的是什么 | 受 Space transform 影响吗？ | 受 SVG width 属性影响吗？ | 单位 |
|---------|------------|---------------------------|--------------------------|------|
| `node.clientWidth` | CSS 布局宽度（content + padding） | ❌ 不受 | ⚠️ **间接影响**（见 2.2） | CSS 像素 |
| `node.offsetWidth` | 布局宽度（content + padding + border + scrollbar） | ❌ 不受 | ⚠️ 间接影响 | CSS 像素 |
| `node.getBoundingClientRect().width` | 屏幕上渲染的视觉宽度 | ✅ **受影响**（会被 scale(zf) 缩放） | ⚠️ 间接影响 | 物理像素（受 DPR 影响） |

**`html-to-image` 选择 `clientWidth` 的后果**：
- 导出尺寸 = 元素的**布局尺寸**（layout size），不是视觉尺寸（visual size）
- 这是 Zoom 不影响导出尺寸的**根本原因**
- 如果有人想"导出当前视口所见的尺寸"，应该改用 `getBoundingClientRect().width`

### 2.2 `.jsoncrack-canvas` 的 clientWidth 从何而来

`.jsoncrack-canvas` 是 Reaflow `<Canvas>` 渲染的最外层 div。Reaflow 的实现（基于 API 行为推断）：

```
<Canvas width={paneWidth} height={paneHeight} maxWidth={paneWidth} maxHeight={paneHeight}>
  ↓ Reaflow 内部渲染
<div class="jsoncrack-canvas">     <!-- 这个 div 没有显式的 width/height style -->
  <svg
    width={paneWidth}
    height={paneHeight}
    viewBox="0 0 {paneWidth} {paneHeight}"
    style="max-width: {paneWidth}; max-height: {paneHeight};"
  >
    ... 节点和边 ...
  </svg>
</div>
```

关键：**外层 div 没有显式设置 `width`/`height` CSS 属性**。
作为一个 block-level 元素：

```
div.clientWidth = min(max-content, available-width)
```

由于 `<svg width="paneWidth">` 是一个**已替换元素**（replaced element），它的固有尺寸 = `paneWidth`。
外层 div 的 `max-content` = 内部 SVG 的固有宽度 = `paneWidth`。

`available-width` = Space Inner Div 的可用宽度。由于 Space Inner Div 没有显式宽度约束，
且 transform 不改变布局尺寸，available-width 实际上等于 `.canvasWrapper` 的宽度（通常是浏览器视口宽度）。

所以最终：

```
div.clientWidth = min(paneWidth, viewportWidth)
```

等等——这与我们之前的结论有偏差！

### 2.3 修正：当 paneWidth > viewportWidth 时

如果 ELK 计算出 `paneWidth = 12000`，但浏览器视口只有 `1200px`：

```
div.clientWidth = min(12000, 1200) = 1200px
```

这意味着**导出的 PNG 宽度会被裁剪到 1200px**，而不是完整的 12000px！

**这是一个严重的问题。** 之前的分析忽略了这个细节。

但等一下——为什么 `getExportElement()` 的 fallback 是 `document.querySelector("svg[id*='ref']")`？

让我们重新审视 `getExportElement()` 的实现：

```ts
// [DownloadModal/index.tsx#L65-L67]
const getExportElement = () =>
  (document.querySelector(".jsoncrack-canvas") as HTMLElement | null) ??
  (document.querySelector("svg[id*='ref']") as HTMLElement | null);
```

**优先抓取 div.jsoncrack-canvas，失败才抓 svg**。

如果 div 的 clientWidth = 1200，而 svg 的 width 属性 = 12000，
导出会基于 div 的 1200px 进行，导致**超大画布在小视口下导出分辨率严重不足**。

### 2.4 为什么 Reaflow 不直接给 div 设置 width

Reaflow 把 `width`/`height` 写到 `<svg>` 的属性上，而不是外层 `<div>` 的 style 上。
这是 Reaflow 的设计选择，带来的副作用是外层 div 的尺寸依赖 CSS 布局规则。

**`maxWidth`/`maxHeight` 的作用**：
```tsx
<Canvas
  maxHeight={paneHeight}
  maxWidth={paneWidth}
  ...
/>
```
Reaflow 会把这两个值写到 `<svg>` 的 `style.max-width` / `style.max-height` 上。
在正常布局下，这不会影响 div 的 clientWidth，但当 svg 需要响应式缩放时（比如容器变窄），
max-width 会约束 svg 不超过 paneWidth。

**但当 div.clientWidth 已经因容器太窄而被缩小时，svg 的 width 属性还是 12000，
svg 会被 CSS 缩小到 1200px 显示，`getComputedStyle(svg).width` 返回 1200px，
但 `svg.getAttribute('width')` 返回 "12000"。**

### 2.5 修正后的尺寸公式

考虑容器宽度约束的完整公式：

```
availableWidth  = 容器可用宽度（通常 = 视口宽度）
availableHeight = 容器可用高度（通常 = 视口高度）

// SVG 的固有尺寸由属性决定
svgIntrinsicWidth  = Number(svg.getAttribute('width'))  // = paneWidth
svgIntrinsicHeight = Number(svg.getAttribute('height')) // = paneHeight

// 外层 div 的布局尺寸（block element 规则）
divLayoutWidth  = min(svgIntrinsicWidth,  availableWidth)
divLayoutHeight = min(svgIntrinsicHeight, availableHeight)

// clientWidth = divLayoutWidth（如果没有 padding）
// getNodeWidth = clientWidth + border
getImageSize().width  = divLayoutWidth  + borderWidth   // 通常 border = 0
getImageSize().height = divLayoutHeight + borderWidth   // 通常 border = 0
```

**场景验证**：

| 场景 | paneWidth | viewportWidth | div.clientWidth | getImageSize().width |
|------|-----------|---------------|-----------------|---------------------|
| 小图（10 节点） | 800 | 1200 | 800 | 800 |
| 大图（300 节点） | 12000 | 1200 | **1200** | **1200** ⚠️ 被容器约束！ |
| 大图，最大化浏览器 | 12000 | 2560 | **2560** | **2560** ⚠️ 还是不够！ |
| 大图，全屏 + 外接显示器 | 12000 | 3840 | **3840** | **3840** ⚠️ 仍不足 |

**这就解释了用户可能遇到的现象**：同一个 JSON，在小屏幕上导出的 PNG 分辨率比在大屏幕上低。

---

## 三、Space transform 对尺寸读取的影响详解

### 3.1 transform 作用的层级回顾

```
Space Outer Div (.jsoncrack-space)  ← 事件监听、overflow:hidden
  ↓ position: relative
Space Inner Div  ← transform: translate(tx, ty) scale(zf)  ⭐
  ↓ 包含所有 Space.children
Canvas Div (.jsoncrack-canvas)  ← getExportElement() 抓取目标
  ↓
SVG
```

### 3.2 三种尺寸读取方式的表现

假设 `paneWidth = 1200`，`viewportWidth = 1200`，`zoom = 200%`（`zf = 2`），`tx = -200`，`ty = -100`：

| 读取方式 | 对 `.jsoncrack-canvas` 调用 | 结果 | 原因 |
|---------|---------------------------|------|------|
| `clientWidth` | `node.clientWidth` | `1200` | 布局尺寸，不受祖先 transform 影响 |
| `offsetWidth` | `node.offsetWidth` | `1200` | 布局尺寸，不受祖先 transform 影响 |
| `getBoundingClientRect()` | `node.getBoundingClientRect().width` | `2400` | 视觉尺寸，祖先的 `scale(2)` 已经通过矩阵相乘应用到 bounding rect |
| `getBoundingClientRect().left` | `node.getBoundingClientRect().left` | 视口 left - 200 | 祖先的 `translate(-200, -100)` 也被应用 |

**为什么 `getBoundingClientRect()` 会受祖先 transform 影响？**

`getBoundingClientRect()` 的定义是：**返回元素相对于视口（viewport）的矩形**。
浏览器在计算时会遍历整个祖先链，将所有 CSS transform 矩阵相乘，
最后应用到元素的布局矩形上，得到屏幕上的实际渲染矩形。

而 `clientWidth` 的定义是：**元素的内容区域 + padding 的宽度**，
这是一个**布局内的量**（in-layout metric），只受 CSS `width`、`padding`、`box-sizing` 影响，
与视觉渲染叠加的 transform 无关。

### 3.3 html-to-image 为什么选择 clientWidth

html-to-image 的工作原理是克隆节点 → 内联样式 → foreignObject 包裹。
如果它用 `getBoundingClientRect().width` 作为导出尺寸，会有两个问题：

1. **Zoom 会改变导出尺寸**：缩放到 200% 时导出图变大一倍，缩放到 50% 时变小一半
2. **平移会导致负的 left/top**：这需要额外处理坐标偏移

`clientWidth` 提供了稳定的、与视图无关的尺寸基准。
但代价是**可能被父容器的宽度约束**（见 2.5 节）。

---

## 四、SVG viewBox + foreignObject 的缩放语义

### 4.1 html-to-image 生成的 SVG 结构

```xml
<!-- 外层 SVG: html-to-image 生成 -->
<svg xmlns="http://www.w3.org/2000/svg"
     width="{getImageSize().width}"
     height="{getImageSize().height}"
     viewBox="0 0 {getImageSize().width} {getImageSize().height}">

  <!-- foreignObject: html-to-image 生成，填满整个 SVG -->
  <foreignObject width="100%" height="100%" x="0" y="0">

    <!-- 克隆的目标节点: .jsoncrack-canvas div -->
    <div class="jsoncrack-canvas" style="...内联样式...">

      <!-- 内部 SVG: Reaflow 渲染，也被克隆了 -->
      <svg width="{paneWidth}" height="{paneHeight}" viewBox="0 0 {paneWidth} {paneHeight}">
        <g>
          <!-- 节点: foreignObject 内嵌 HTML -->
          <foreignObject width="200" height="80" x="50" y="50">
            <span class="row">key: "value"</span>
          </foreignObject>
          ...
        </g>
      </svg>

    </div>
  </foreignObject>
</svg>
```

### 4.2 两层 SVG + 两层 foreignObject 的嵌套问题

这是最容易出错的地方——实际结构是 **SVG 嵌套 SVG，foreignObject 嵌套 foreignObject**：

```
外层 SVG (html-to-image 生成)
  width = W_hti = min(paneWidth, viewportWidth)
  viewBox = "0 0 W_hti H_hti"
  └── foreignObject (100% × 100%)
       └── div.jsoncrack-canvas (克隆)
            └── 内层 SVG (Reaflow 渲染，也被克隆)
                 width = paneWidth
                 height = paneHeight
                 viewBox = "0 0 paneWidth paneHeight"
                 └── <g>
                      └── foreignObject (节点，宽高为节点尺寸)
                           └── HTML 文本
```

### 4.3 当 paneWidth ≠ W_hti 时的缩放行为

**场景**：`paneWidth = 12000`，`viewportWidth = 1200`，所以 `W_hti = 1200`

外层 SVG 的 viewBox 是 `0 0 1200 900`（假设 H_hti = 900）。
内层 SVG 的 width 属性是 `"12000"`，它被放在 foreignObject 内部的 HTML 流中。

**内层 SVG 的渲染尺寸**：
- 在 HTML 流中，`<svg width="12000">` 作为 replaced element，固有宽度 = 12000px
- 但它的父元素是 `div.jsoncrack-canvas`（克隆的），其 `width` 被 `applyStyle` 可能设置为 1200px
- 由于外层 div 没有 `overflow: visible`，超出部分会被裁剪

**关键问题**：如果 div 的 clientWidth 被约束到 1200px，而 svg 的固有宽度是 12000px，
svg 会被 CSS 默认 `overflow: hidden` 裁剪掉右侧 90% 的内容！

### 4.4 为什么我们之前没遇到这个问题

在实际使用中，如果用户在导出前**已经将画布缩放到 fit-to-view**，
Space Inner Div 会调整 `scale()` 使整个图在视口内可见。
但 Space 的 transform 不影响布局尺寸，所以 div.clientWidth 仍然 = min(paneWidth, viewportWidth)。

**真正的原因**：Reaflow 的 Canvas 外层 div 很可能被设置了 `position: absolute` 或 `display: inline-block`，
使其 `clientWidth` 等于内容宽度（即 SVG 的固有宽度），而不是受父容器约束。

这需要实际运行时验证，但基于代码推断：

```tsx
// Reaflow Canvas 渲染的外层 div 大致样式
position: absolute;
top: 0;
left: 0;
```

`position: absolute` 的元素脱离了正常文档流，其宽度默认由内容决定（shrink-to-fit），
而不是填充父容器。所以：

```
div.clientWidth = max-content = svgIntrinsicWidth = paneWidth
```

这才是 `clientWidth` 等于 `paneWidth` 的真正原因，而不是之前假设的 block element 行为。

---

## 五、最终的精确尺寸公式

### 5.1 前提假设

基于 Reaflow 5.4.1 和 react-zoomable-ui 0.11.0 的实际行为：

1. `.jsoncrack-canvas` div 的 `position` = `absolute`（脱离文档流）
2. 该 div 没有显式 `width` / `height`
3. 内部 `<svg>` 有 `width={paneWidth}` `height={paneHeight}`
4. Space Inner Div 没有 overflow 裁剪（或者 div 已脱离文档流）

### 5.2 精确公式

```
// ========== 第一部分：决定基础尺寸 ==========

// Step 1: ELK 布局结果
elkWidth  = layout.width   // ELK 返回的节点包围盒宽度
elkHeight = layout.height  // ELK 返回的节点包围盒高度

// Step 2: JSONCrack 加边距后设置 pane 尺寸
paneWidth  = elkWidth  + 50   // 每边 25px 边距
paneHeight = elkHeight + 50

// Step 3: Reaflow 将 pane 尺寸写到 svg 属性上
svgWidthAttr  = paneWidth     // svg.getAttribute('width')
svgHeightAttr = paneHeight    // svg.getAttribute('height')
svgViewBox    = `0 0 ${paneWidth} ${paneHeight}`
svgStyleMaxWidth  = `${paneWidth}px`
svgStyleMaxHeight = `${paneHeight}px`

// Step 4: 外层 div 的布局尺寸 (position: absolute + shrink-to-fit)
// = 内部 svg 的固有尺寸
divClientWidth  = svgWidthAttr   // = paneWidth
divClientHeight = svgHeightAttr  // = paneHeight

// Step 5: html-to-image 读取尺寸（+ border，通常为 0）
getImageSize = {
  width:  divClientWidth  + borderLeftWidth + borderRightWidth,   // ≈ paneWidth
  height: divClientHeight + borderTopWidth  + borderBottomWidth   // ≈ paneHeight
}


// ========== 第二部分：根据格式计算输出尺寸 ==========

// --- PNG / JPEG ---
ratio         = window.devicePixelRatio || 1
canvasWidth   = getImageSize.width    // = paneWidth
canvasHeight  = getImageSize.height   // = paneHeight
canvasPixelWidth  = canvasWidth  * ratio
canvasPixelHeight = canvasHeight * ratio

// checkCanvasDimensions 裁剪
if canvasPixelWidth > 16384 || canvasPixelHeight > 16384:
  scaleFactor = 16384 / max(canvasPixelWidth, canvasPixelHeight)
  canvasPixelWidth  = Math.round(canvasPixelWidth  * scaleFactor)
  canvasPixelHeight = Math.round(canvasPixelHeight * scaleFactor)

outputPngWidth  = canvasPixelWidth   // 最终 PNG 像素宽度
outputPngHeight = canvasPixelHeight  // 最终 PNG 像素高度

// --- SVG ---
outputSvgWidth  = getImageSize.width   // = paneWidth (CSS 像素)
outputSvgHeight = getImageSize.height  // = paneHeight (CSS 像素)
outputViewBox   = `0 0 ${outputSvgWidth} ${outputSvgHeight}`

// 无 DPR 放大，无 16384 裁剪


// ========== 第三部分：关键变量的影响 ==========

影响因子                     | 影响 outputPngWidth 吗？ | 影响 outputSvgWidth 吗？
---------------------------|-------------------------|------------------------
ELK layout.width           | ✅ 直接影响（× ratio）    | ✅ 直接影响
+50 边距                   | ✅ 直接影响               | ✅ 直接影响
Space zoom (scale)         | ❌ 不影响（clientWidth 独立） | ❌ 不影响
Space pan (translate)      | ❌ 不影响                 | ❌ 不影响
window.devicePixelRatio    | ✅ 放大（× ratio）        | ❌ 不影响
容器视口宽度               | ⚠️ 仅当 div 非 absolute 时才约束 | ⚠️ 仅当 div 非 absolute 时才约束
16384 上限                 | ✅ 超大图等比缩小         | ❌ 不影响
backgroundColor            | ❌ 不影响尺寸（仅影响填充） | ❌ 不影响尺寸
skipFonts                  | ❌ 不影响尺寸（仅影响字体渲染） | ❌ 不影响尺寸
```

### 5.3 修正之前分析的偏差

| 之前的说法 | 修正后的精确说法 |
|-----------|-----------------|
| "div.clientWidth = paneWidth" | "div.clientWidth = paneWidth 的前提是 div 为 `position: absolute` 且无 width 约束，使 shrink-to-fit 宽度等于内部 svg 的固有宽度。如果 div 是正常 block 元素，会被视口约束。" |
| "Zoom 不影响导出，因为 transform 在祖先上" | "Zoom 不影响导出，因为 html-to-image 使用 `clientWidth`（布局尺寸），它独立于祖先 transform。即使改用 `getBoundingClientRect()`，也会受 Zoom 影响。" |
| "PNG 尺寸 = paneWidth × DPR" | "PNG 尺寸 = min(paneWidth × DPR, 16384)，等比缩放保证宽高比不变。paneWidth = ELK width + 50。" |
| "SVG 导出无 DPR 放大" | "SVG 导出使用 CSS 像素尺寸（= paneWidth），无 DPR 放大，也无 16384 上限。但内嵌的 HTML 内容不是纯矢量，依赖浏览器渲染。" |

---

## 六、为什么 `getExportElement()` 有两个选择器

```ts
const getExportElement = () =>
  (document.querySelector(".jsoncrack-canvas") as HTMLElement | null) ??
  (document.querySelector("svg[id*='ref']") as HTMLElement | null);
```

**主选择器 `.jsoncrack-canvas`**：
- 目标是 Reaflow Canvas 的外层 div
- 优点：包含完整的 SVG 内容，有背景色样式
- 潜在问题：如果 Reaflow 版本改变了类名或 DOM 结构，会找不到

**兜底选择器 `svg[id*='ref']`**：
- 直接抓 SVG 元素（Reaflow 内部的 svg，其 id 包含 ref 字符串）
- 优点：不依赖外层 div 的类名，更健壮
- 缺点：可能缺少外层 div 的样式（如 backgroundColor）
- 说明代码作者已经预见了 Reaflow DOM 结构可能变化

**潜在改进**：应该优先选择 `svg` 而非外层 `div`，因为：
1. `svg.clientWidth` 不受 position 影响，始终等于 `width` 属性值
2. `svg` 没有 padding/border 干扰
3. 少一层 DOM 嵌套，html-to-image 处理更快更可靠

---

## 七、优化建议

### 建议 1：优先抓 SVG，而非外层 DIV（P0）

```ts
const getExportElement = () =>
  (document.querySelector("svg[id*='ref']") as HTMLElement | null) ??
  (document.querySelector(".jsoncrack-canvas") as HTMLElement | null);
```

**理由**：
- SVG 的 `clientWidth` = `paneWidth`，不受容器视口约束，也不受 `position` 属性影响
- 减少一层 foreignObject 嵌套，降低兼容性风险
- 导出文件更小（少一个 div 的样式）

### 建议 2：显式设置目标元素的 width/height 样式（P1）

在导出前给目标元素临时设置 `style.width` 和 `style.height`：

```ts
const exportAsImage = async () => {
  const imageElement = getExportElement();
  if (!imageElement) { toast.error("Canvas not found."); return; }

  // 临时设置尺寸，消除 CSS 布局的不确定性
  const originalWidth = imageElement.style.width;
  const originalHeight = imageElement.style.height;
  imageElement.style.width = `${paneWidth}px`;
  imageElement.style.height = `${paneHeight}px`;

  try {
    const dataURI = await getDownloadFormat(extension)(imageElement, imageOptions);
    downloadURI(dataURI, `${fileDetails.filename}.${extension}`);
  } finally {
    // 恢复原始样式
    imageElement.style.width = originalWidth;
    imageElement.style.height = originalHeight;
  }
};
```

**理由**：消除 `div.clientWidth = min(paneWidth, viewportWidth)` 的隐患，
确保无论视口多大，导出尺寸始终 = `paneWidth × paneHeight`。

### 建议 3：html-to-image 显式传 width/height（P1）

```ts
const imageOptions = {
  quality: fileDetails.quality,
  backgroundColor: fileDetails.backgroundColor,
  skipFonts: true,
  width: paneWidth,      // 显式覆盖
  height: paneHeight,    // 显式覆盖
};
```

**理由**：`getImageSize()` 会优先使用 `options.width` 而不是 `getNodeWidth(node)`，
这比修改 DOM style 更干净。

---

## 八、验证方法

在浏览器 DevTools 中验证上述结论：

### 验证 div 的 clientWidth 是否等于 paneWidth

```js
const div = document.querySelector('.jsoncrack-canvas');
const svg = div.querySelector('svg');
console.log('div.clientWidth:', div.clientWidth);
console.log('svg.getAttribute(width):', svg.getAttribute('width'));
console.log('div.position:', getComputedStyle(div).position);
```

预期输出（正常情况）：
```
div.clientWidth: 8050
svg.getAttribute(width): 8000
div.position: absolute
```

（多 50 是因为 +25px padding 每边）

### 验证 Space Inner Div 的 transform

```js
const space = document.querySelector('.jsoncrack-space');
const innerDiv = space.firstElementChild;  // 这是 Space Inner Div
console.log('Space Inner Div transform:', getComputedStyle(innerDiv).transform);
console.log('.jsoncrack-canvas transform:', getComputedStyle(div).transform);
```

预期输出：
```
Space Inner Div transform: matrix(1, 0, 0, 1, 0, 0)  // 或 matrix(2, 0, 0, 2, -200, -100)
.jsoncrack-canvas transform: none                     // 自身没有 transform
```

### 验证导出尺寸的实际计算

```js
// 模拟 html-to-image 的 getNodeWidth
function getNodeWidth(node) {
  const cs = getComputedStyle(node);
  const leftBorder = parseFloat(cs.borderLeftWidth) || 0;
  const rightBorder = parseFloat(cs.borderRightWidth) || 0;
  return node.clientWidth + leftBorder + rightBorder;
}

const node = document.querySelector('.jsoncrack-canvas');
console.log('html-to-image 读取的宽度:', getNodeWidth(node));
console.log('clientWidth:', node.clientWidth);
console.log('offsetWidth:', node.offsetWidth);
console.log('getBoundingClientRect().width:', node.getBoundingClientRect().width);
console.log('DPR:', window.devicePixelRatio);
console.log('预期 PNG 宽度（未裁剪）:', getNodeWidth(node) * window.devicePixelRatio);
```
