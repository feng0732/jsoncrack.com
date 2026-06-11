# 导出视口与边界机制深度分析

> 本文档深入分析：视口缩放（Zoom）、平移（Pan）、画布尺寸（Canvas Size）、节点超限（Node Limit）
> 四大状态如何影响 PNG / SVG 导出的最终结果。

---

## 一、核心 DOM 层级与分层职责

理解影响的前提是理清渲染树的**四层嵌套结构**，每层职责完全不同，缩放/平移/尺寸分别作用在不同层：

```
┌──────────────────────────────────────────────────────────────┐
│ Layer 1: .canvasWrapper (最外层)                             │
│  尺寸: width:100% height:100% (视口容器大小)                  │
│  定位: relative                                              │
│  作用: 提供背景色 + 背景网格 (CSS background-image)           │
│  代码: [JSONCrackStyles.module.css#L1-L24]                   │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Layer 2: <Space> (react-zoomable-ui)                  │  │
│  │  类名: .jsoncrack-space                                │  │
│  │  ⚡ 这一层才是应用 CSS transform(scale + translate) 的层 │  │
│  │  代码: [JSONCrackComponent.tsx#L570-L613]              │  │
│  │                                                        │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │ Layer 3: <Canvas className="jsoncrack-canvas">   │  │  │
│  │  │  来源: reaflow                                    │  │  │
│  │  │  ⚡ 这是 getExportElement() 直接抓取的目标元素     │  │  │
│  │  │  width/height = paneWidth / paneHeight           │  │  │
│  │  │  代码: [JSONCrackComponent.tsx#L585-L611]         │  │  │
│  │  │                                                  │  │  │
│  │  │  ┌────────────────────────────────────────────┐  │  │  │
│  │  │  │ Layer 4: <svg> (reaflow 内部渲染)           │  │  │  │
│  │  │  │   └── <g> (content group)                  │  │  │  │
│  │  │  │        ├── <g id="node-1">...</g>          │  │  │  │
│  │  │  │        ├── <g id="edge-1">...</g>          │  │  │  │
│  │  │  │        └── ...                              │  │  │  │
│  │  │  └────────────────────────────────────────────┘  │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

**代码定位**：
- Layer 1/2/3 组装：[JSONCrackComponent.tsx#L536-L614](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L536-L614)
- Layer 4 中的节点（foreignObject 内嵌 HTML）：[ObjectNode.tsx#L95-L113](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/packages/jsoncrack-react/src/components/ObjectNode.tsx#L95-L113)、[TextNode.tsx#L22-L40](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/packages/jsoncrack-react/src/components/TextNode.tsx#L22-L40)

---

## 二、视口缩放（Zoom）对导出的影响

### 2.1 缩放实现原理

缩放通过 `react-zoomable-ui` 的 `ViewPort.camera` 接口实现，关键 API：

```ts
// 单次步进 (±0.1)
adjustViewPortZoom(viewPort, ±0.1)   // [canvasHelpers.ts#L257-L259]

// 绝对设值
setViewPortZoom(viewPort, zoomFactor) // [canvasHelpers.ts#L251-L253]

// 内部实现 (camera.recenter)
viewPort.camera?.recenter(viewPort.centerX, viewPort.centerY, newZoomFactor)
```

**作用层级**：`react-zoomable-ui` 的 `<Space>` 会对它的**直接子元素**应用 `CSS transform: translate(tx, ty) scale(zf)`。
这个 transform 作用在 **Layer 2 和 Layer 3 之间的 wrapper** 上，**不会**写在 `.jsoncrack-canvas` 自身的 style 上。

### 2.2 缩放状态对导出结果的影响——**关键结论：无影响**

| 维度 | 用户所见 | html-to-image 导出时 |
|------|---------|---------------------|
| 抓取目标 | `.jsoncrack-canvas`（Layer 3） | 同左 |
| Zoom transform 所在元素 | Layer 2 的 Space 内部 wrapper | **不在被抓取元素上** |
| 应用 transform 的祖先是否会被克隆 | 视觉上缩放了整个画布 | **html-to-image 只克隆目标元素及其子树，不会向上追溯祖先的 transform** |
| 导出图像像素尺寸 | 屏幕上被缩小了 | 依然是 `paneWidth × paneHeight` 的完整分辨率 |

**原理证明**：`html-to-image` 的工作方式
```
toPng(element, options):
  1. 克隆 element（.jsoncrack-canvas）及其所有子节点
  2. 对每个克隆节点，内联 window.getComputedStyle(原节点) 的样式
  3. 把克隆树包裹进 <foreignObject> → <svg>
  4. SVG 绘制到 <canvas> → canvas.toDataURL("image/png")
```

第 2 步只会复制 `.jsoncrack-canvas` 自身的 computed style，而 Space 的 `transform: scale()` 在**祖先元素**上，不会被内联到目标元素的 style 中。

### 2.3 验证依据

`computeGraphClientRect()` 的实现间接证明了这一点。它在测量节点边界时使用了 `getBoundingClientRect()`（**已应用祖先 transform** 的屏幕坐标），然后必须通过 `translateClientRectToVirtualSpace(rect)` 把屏幕坐标反变换回**虚拟空间坐标**——这说明"实际渲染的节点尺寸"和"屏幕上呈现的像素尺寸"是两套坐标系：

```ts
// [canvasHelpers.ts#L233-L237]
const rect = computeGraphClientRect(container, layoutSize); // 屏幕坐标
const virtualRect = viewPort.translateClientRectToVirtualSpace(rect); // 反缩放/反平移
viewPort.camera?.centerFitAreaIntoView(virtualRect);
```

而导出时，html-to-image 拿到的是 **Layer 3 的 DOM + 自身 computed style**，与 Layer 2 的 transform 完全解耦。

---

## 三、视口平移（Pan）对导出的影响

### 3.1 平移实现原理

平移同样通过 `camera.recenter` / `camera.moveByInClientSpace` 等 API，同样写在 Layer 2 的 transform 上（`translate(tx, ty)` 部分）。

用户可以：
- 拖拽画布空白处平移（Space 默认行为）
- 节点折叠后相机跟随按钮定位：`viewPort?.camera?.moveByInClientSpace(dx, dy)` [JSONCrackComponent.tsx#L435](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L435)

### 3.2 平移状态对导出结果的影响——**关键结论：无影响**

平移和缩放一样，作用在 `.jsoncrack-canvas` 的**祖先元素**上。导出抓取 Layer 3 时：

```
用户视图：平移到右下角，屏幕上只看到节点 N 的一部分
        ↓
导出抓取：.jsoncrack-canvas 完整 DOM（所有节点 + 原始坐标）
        ↓
导出结果：整张图完整居中，所有节点都在，内容未被裁剪
```

**用户很容易产生的误解**：
- ❌ 以为"只导出当前视口可见部分"（像截图工具那样）
- ✅ 实际是"导出整个画布的完整内容"（相当于"另存为原图"）

---

## 四、画布尺寸（Canvas Size）对导出的影响

### 4.1 paneWidth / paneHeight 的来源

画布尺寸不是固定值，而是由 **ELK 布局引擎** 根据节点数量和大小在**每次布局时动态计算**：

代码链路：
```
parseJsonGraph(jsonText)                    // 构造 nodes/edges 数组
         ↓
reaflow <Canvas nodes edges onLayoutChange> // 交给 ELK 计算
         ↓
onLayoutChange(layout: ElkRoot) {           // [JSONCrackComponent.tsx#L401-L411]
    layoutSizeRef.current = { width, height };
    setPaneWidth(layout.width + 50);        // 每边 +25px 边距
    setPaneHeight(layout.height + 50);
}
```

ELK 返回的 `layout.width / layout.height` 是**刚好包裹所有节点**的那个矩形尺寸（不包含边距，边距由 +50 补上）。

### 4.2 尺寸传递到的层级

```
<Canvas
  maxWidth={paneWidth}   // 传给 reaflow，约束 SVG 的 width
  maxHeight={paneHeight} // 传给 reaflow，约束 SVG 的 height
  width={paneWidth}      // 同上
  height={paneHeight}    // 同上
  ...
/>
```

reaflow 内部会将这些数值设置为 `<svg>` 元素的 `width` / `height` / `viewBox` 属性。

### 4.3 尺寸对导出的直接影响

| 导出格式 | html-to-image 函数 | 导出图像尺寸来源 |
|---------|-------------------|----------------|
| **PNG** | `toPng()` | = `.jsoncrack-canvas` 的 `offsetWidth × offsetHeight` × devicePixelRatio |
| **JPEG** | `toJpeg()` | 同 PNG |
| **SVG** | `toSvg()` | `<foreignObject>` 包裹尺寸 = `.jsoncrack-canvas` 的 `getBoundingClientRect().width/height`（不经过 DPR 放大，是 CSS 像素值） |

**注意**：由于 Zoom 不影响导出（见第二节），导出图像的尺寸只与 `paneWidth × paneHeight` 成正比，与当前缩放系数 zf 无关。

| 场景 | 当前屏幕观感 | 导出 PNG 尺寸 |
|-----|------------|-------------|
| 大图、缩放到 20% 刚好装下 | 看起来很小 | 仍为 ~(paneWidth×DPR) × (paneHeight×DPR) —— 高清大图 |
| 小图、放大到 250% | 屏幕上一个节点占一半 | 仍为 ~(paneWidth×DPR) × (paneHeight×DPR) —— 不会变成 2.5 倍 |

### 4.4 背景网格的缺失问题

**容易踩坑**：Layer 1 的 `.canvasWrapper` 上通过 CSS 画了网格背景（`background-image: linear-gradient(...)`），但 `getExportElement()` 抓的是 **Layer 3**（`.jsoncrack-canvas`），**所以导出的 PNG/SVG 默认没有网格**。

这不是 bug，是设计如此。用户通过模态框里的 `backgroundColor` 选项只能填充纯色背景，**无法恢复网格**。

如果需要带网格的导出，有两种思路：
1. 改为抓取 Layer 1 `.canvasWrapper`（但会包含 Controls、overlay、侧边栏等，需要额外过滤）
2. 导出前给 svg 追加一个 `<rect>` 用 pattern 模拟网格

---

## 五、节点超限状态对导出的影响

### 5.1 超限判定链路

```
用户上传 JSON
    ↓
parseGraph(jsonText) → 得到 graph.nodes.length
    ↓
parseJsonGraph(jsonText, maxRenderableNodes)
    │
    ├─ if graph.nodes.length > maxRenderableNodes:
    │     → { kind: "above-limit", total: graph.nodes.length }
    │           ↓
    │     React state:
    │       setNodes([])      // 清空节点
    │       setEdges([])      // 清空边
    │       setAboveSupportedLimit(true)
    │       setTotalNodes(graph.nodes.length)
    │
    └─ else:
          → { kind: "ok", graph: graph }
          → 正常渲染
```

代码：[canvasHelpers.ts#L74-L90](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L74-L90) +
[JSONCrackComponent.tsx#L171-L205](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L171-L205)

### 5.2 maxRenderableNodes 的数值

来源链：
```
process.env.NEXT_PUBLIC_NODE_LIMIT            // 环境变量
         ↓
SUPPORTED_LIMIT                               // [graph.ts#L6]
         ↓
Number.isFinite(SUPPORTED_LIMIT) ? SUPPORTED_LIMIT : 1500  // 默认 1500
         ↓
maxRenderableNodes prop 传入 <JSONCrack>      // [GraphView/index.tsx#L86-L106]
```

### 5.3 超限时的 DOM 状态（对导出非常关键）

当 `aboveSupportedLimit = true` 时：

```tsx
// [JSONCrackComponent.tsx#L555-L562]
{aboveSupportedLimit &&
  (tooLargeContent ? (            // ← 用户传了 renderNodeLimitExceeded
     tooLargeContent              //    → 渲染 <NotSupported />（见下文）
  ) : (
     <div className={styles.tooLarge}>...</div>
  ))
}
```

实际渲染效果（www 应用里）：
```
Layer 1: .canvasWrapper
  ├── Overlay + NotSupported 卡片（绝对定位、居中、z-index: 10）
  │     内容: "Your diagram is too large / Upgrade Required" 宣传卡片
  │     代码: [NotSupported.tsx#L101-L163]
  │
  └── Layer 2/3/4: 依然存在，但 nodes = [], edges = []
                    所以 SVG 里只有空的 <g>，没有任何节点图形
```

### 5.4 超限时点击"导出"的行为

走到 [DownloadModal/index.tsx#L122-L126](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L122-L126)：

```ts
const imageElement = getExportElement();
//     ↓
// 虽然 .jsoncrack-canvas 存在，但它的 <svg> 是空的（0 nodes）
// imageElement 本身不是 null
if (!imageElement) { toast.error("Canvas not found."); return; }
//     ↓ 不会触发上面的分支，继续往下走...
const dataURI = await getDownloadFormat(extension)(imageElement, ...);
//     ↓
// 导出结果：一张纯色背景（用户所选 backgroundColor）的空白图
// 尺寸：paneWidth/paneHeight 默认 2000×2000（初始值未被 onLayoutChange 更新）
```

**⚠️ 潜在的 Bug / UX 问题**：超限时没有在 DownloadModal 里做拦截，用户仍然可以操作导出按钮，得到一张 2000×2000 的纯色空白图，没有任何解释。建议在 `exportAsImage()` 入口加一个 `useGraph(state => state.totalNodes)` 判断，超限就 toast 提示"节点超限无法导出"。

---

## 六、完整的状态 → 导出结果映射表

| 场景 | Zoom 200% | Zoom 20% | 平移到角落 | paneWidth=8000 | 超限 1500+节点 |
|-----|-----------|----------|-----------|----------------|---------------|
| **导出 PNG 尺寸** | `paneW × DPR` | 同左 | 同左 | `8000×DPR` | `2000×DPR` (初始值) |
| **导出内容完整性** | 完整全图 | 完整全图 | 完整全图 | 完整全图 | **空白图** |
| **相对比例 / 布局** | 1:1 原始比例 | 同左 | 同左 | 按 ELK 结果 | N/A |
| **背景网格** | ❌ 不含 | ❌ 不含 | ❌ 不含 | ❌ 不含 | ❌ 不含 |
| **包含 Loading 遮罩** | ❌ | ❌ | ❌ | ❌ | 视时机而定 |
| **包含 NotSupported 卡片** | ❌ 在 Layer 1 之上，不会被抓 | ❌ | ❌ | ❌ | ✅ 有可能，取决于 DOM 顺序（NotSupported 目前在 Overlay 中，不在 .jsoncrack-canvas 内 → **不会被抓**） |

---

## 七、深层原理：html-to-image 与 SVG foreignObject 的交互

### 7.1 为什么节点内的文本能被正确导出？

节点内部不是纯 SVG `<text>`，而是 **SVG `<foreignObject>` 嵌套 HTML**：

```tsx
// [ObjectNode.tsx#L95-L113]
<foreignObject width={node.width} height={node.height} x={0} y={0}>
  {node.text.map((row, index) => (
    <span className={styles.row} ...>  ← 这是 HTML 元素
      ...
    </span>
  ))}
</foreignObject>
```

html-to-image 的 `toSvg()` 本来是把**普通 HTML 元素**用 `<foreignObject>` 包起来转 SVG。但这里我们传给它的 `.jsoncrack-canvas` 里面**本身就是 SVG + foreignObject**。

最终结构为：
```
toPng() 生成的中间 SVG:
  <foreignObject>                        ← html-to-image 加的
    <div class="jsoncrack-canvas">       ← 克隆的目标根元素
      <svg>                              ← reaflow 的 SVG
        <g><foreignObject>HTML 文本</foreignObject></g>
      </svg>
    </div>
  </foreignObject>
```

两层 foreignObject 的嵌套在大多数浏览器中都能正常工作，但这也是**最容易出兼容性 Bug 的地方**（特别是某些旧版浏览器或使用 CSP 限制 SVG 内联 HTML 的场景）。

### 7.2 skipFonts: true 的具体影响

[DownloadModal/index.tsx#L89-L90](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L89-L90) 传入的 `skipFonts: true`：

html-to-image 默认会扫描 DOM 中用到的所有 webfont → fetch 字体文件 → 内联为 base64 dataURL 嵌入 SVG/Canvas。`skipFonts` 跳过这一步，带来两个结果：

- ✅ **加速**：省去所有字体的网络请求和 base64 编码
- ❌ **字体降级风险**：如果 Mona-Sans.woff2（项目里的字体）未在浏览器缓存里，导出时节点文本会回退到系统默认字体（`-apple-system`, `Segoe UI`, `Roboto` 等...见 [Node.module.css#L4-L6](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/packages/jsoncrack-react/src/components/Node.module.css#L4-L6)），与页面看到的有细微差别。

因为当前节点 CSS 中写了很长的 `font-family` fallback 链，所以大部分情况下差别可忽略。

---

## 八、潜在风险与优化建议

### 建议 1：超限时拦截导出（P0）

在 `DownloadModal` 中加判断：

```ts
const totalNodes = useGraph(s => s.totalNodes);
const aboveLimit = useGraph(s => s.aboveSupportedLimit);

const exportAsImage = async () => {
  if (aboveLimit) {
    toast.error(`节点超限（共 ${totalNodes} 个），无法导出图像。`);
    return;
  }
  // ... 原有逻辑
};
```

### 建议 2：支持导出网格（P1）

让用户可选"带网格导出"，方式：
- 在调用 `getDownloadFormat()` 前，给目标 svg 注入一个 `<rect>` + `<pattern>` 的网格背景
- 或者改为抓取 Layer 1 `.canvasWrapper` 但先把 Overlay/Controls `display:none`

### 建议 3：导出前强制 zoom = 1 + center（P2）

虽然已经论证过 Zoom 不影响导出，但在某些边缘情况下（比如 reaflow 内部使用了 transform 缓存），**导出前先 reset 到 fit-to-view** 可以消除意外。不过这会打断用户当前视图，建议加一个"是否恢复视图"的逻辑。

### 建议 4：导出时使用 pixelRatio 参数（P2）

html-to-image 有一个 `pixelRatio` 选项，可以覆盖 devicePixelRatio 强制输出更高/更低 DPI 的 PNG/JPEG。当前未显式设置，使用默认值（= DPR）。对于超大图场景，用户可能想控制文件大小。

### 建议 5：scrollbars 溢出（P3）

`.jsoncrack-canvas` 本身没有 `overflow: hidden`。某些情况下（比如 SVG 内的元素超出 viewBox）可能渲染出滚动条或 1px 外溢区域影响导出边界。建议在导出时给目标元素临时加 `overflow: hidden` / `transform: none` 重置样式。

---

## 九、调试指南：如何验证导出行为

在浏览器 DevTools 中验证上述结论：

1. **验证 Zoom 不影响导出**：
   - 缩放到 300%，导出 PNG → 看尺寸 = paneWidth × DPR（不是 3×）
   - 在 DevTools 里选 `.jsoncrack-canvas` → Computed → 看 `transform`，应该是 `none`（transform 在祖先上）

2. **验证超限时导出空白**：
   - F12 → Application → Local Storage → 设 mock 数据让节点数超过 limit
   - 导出 → 应该是纯色背景 + 空内容

3. **验证 Canvas 尺寸**：
   - DevTools 选中 `<svg>` 看属性：`width`, `height`, `viewBox` 应该与 `paneWidth + 50 / paneHeight + 50` 一致
   - 展开布局回调 `onLayoutChange` 可看到 ELK 返回的尺寸

4. **理解 html-to-image 输出**：
   - 在 `exportAsImage` 里 `console.log(dataURI)`，把 base64 解码即可看到中间 SVG/Blob 结构
