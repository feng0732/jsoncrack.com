# 导出图像尺寸规则：代码级精确分析

> 本文档逐行追踪 html-to-image 源码 + JSON Crack 业务代码，精确还原 PNG / JPEG / SVG
> 三种格式导出时图像尺寸的完整计算链路，以及 Canvas 上限裁剪对超大画布的影响。

---

## 一、尺寸计算的完整代码调用链

### 1.1 入口：DownloadModal 传入的 options

[DownloadModal/index.tsx#L127-L131](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L127-L131)

```ts
const imageOptions = {
  quality: fileDetails.quality,        // = 1（硬编码初始值，无 UI 调节）
  backgroundColor: fileDetails.backgroundColor, // 用户选的背景色
  skipFonts: true,
  // ⚠️ 注意：没有传 width / height / pixelRatio / canvasWidth / canvasHeight / skipAutoScale
};
```

**所有未传的 options 字段**均走 html-to-image 的默认逻辑。下面逐个追踪。

### 1.2 第一站：getImageSize() —— 获取目标元素的 CSS 像素尺寸

html-to-image `util.js` 源码：

```js
function getNodeWidth(node) {
  const leftBorder  = px(node, 'border-left-width');
  const rightBorder = px(node, 'border-right-width');
  return node.clientWidth + leftBorder + rightBorder;
}
function getNodeHeight(node) {
  const topBorder    = px(node, 'border-top-width');
  const bottomBorder = px(node, 'border-bottom-width');
  return node.clientHeight + topBorder + bottomBorder;
}
export function getImageSize(targetNode, options = {}) {
  const width  = options.width  || getNodeWidth(targetNode);
  const height = options.height || getNodeHeight(targetNode);
  return { width, height };
}
```

**关键细节**：
- `options.width` 未传 → 走 `getNodeWidth(node)` → 读 `node.clientWidth` + border
- **目标 node** 是 `getExportElement()` 返回的 `.jsoncrack-canvas`（一个 `<div>`）
- `clientWidth` = 元素的 **CSS 像素宽度**（content + padding，不含 border 和 scrollbar）

### 1.3 `.jsoncrack-canvas` 的 clientWidth/clientHeight 从何而来

`getExportElement()` 返回的 `.jsoncrack-canvas` 是 reaflow `<Canvas>` 渲染的容器 `<div>`，
其内部包含一个 `<svg>` 元素。reaflow 设置 `<svg>` 的 `width`/`height` 属性等于传入的 props：

```tsx
// [JSONCrackComponent.tsx#L585-L596]
<Canvas
  className="jsoncrack-canvas"
  maxHeight={paneHeight}    // → svg 的 max-height
  maxWidth={paneWidth}      // → svg 的 max-width
  height={paneHeight}       // → svg 的 height attribute
  width={paneWidth}         // → svg 的 width attribute
  ...
/>
```

而 `paneWidth / paneHeight` 的值来自：

```ts
// [JSONCrackComponent.tsx#L146-L147] 初始值
const [paneWidth,  setPaneWidth]  = useState(2000);
const [paneHeight, setPaneHeight] = useState(2000);

// [JSONCrackComponent.tsx#L401-L410] ELK 布局完成后更新
const onLayoutChange = useCallback((layout: ElkRoot) => {
  if (!layout.width || !layout.height) { setLoading(false); return; }
  layoutSizeRef.current = { width: layout.width, height: layout.height };
  setPaneWidth(layout.width + 50);   // +50 = 每边 25px padding
  setPaneHeight(layout.height + 50);
  setLoading(false);
}, []);
```

所以 **`node.clientWidth` ≈ `paneWidth` ≈ ELK layout width + 50**。

**但有一个微妙差异**：`clientWidth` 取的是 **div** 的渲染宽度，而不是 svg 属性。
如果 `.jsoncrack-canvas` 这个 div 没有显式设 `width/height`（实际没有），那么它的尺寸由**内部 svg 的固有尺寸**撑开。
在 CSS 默认行为下，一个 block div 包含一个 `<svg width="5000" height="3000">`：
- div 的 `clientWidth` = `svg.width` 的 CSS 像素值 = `paneWidth`
- div 的 `clientHeight` = `svg.height` 的 CSS 像素值 = `paneHeight`

**但 Space 的 transform 可能限制可见范围**：
`react-zoomable-ui` 的 `<Space>` 会对子元素施加 `transform: translate(...) scale(...)`，
这会改变子元素在屏幕上的视觉大小，但 **不改变 `clientWidth/clientHeight`**。

> `clientWidth` 反映的是元素的布局宽度（layout width），受 CSS `width`、`box-sizing`、
> `padding` 等影响，**不受 `transform` 影响**。`transform` 是一种视觉叠加（visual effect），
> 在布局之后应用，不回流（reflow）。

**结论**：`getImageSize()` 返回的 `{ width, height }` = `{ paneWidth, paneHeight }`（大约值，不含 border）。

---

## 二、三条格式的尺寸生成路径

### 2.1 SVG 路径（toSvg）

```js
// html-to-image index.js
export async function toSvg(node, options = {}) {
    const { width, height } = getImageSize(node, options);  // ← paneWidth, paneHeight
    const clonedNode = await cloneNode(node, options, true);
    await embedWebFonts(clonedNode, options);
    await embedImages(clonedNode, options);
    applyStyle(clonedNode, options);
    const datauri = await nodeToDataURL(clonedNode, width, height);
    return datauri;
}
```

`nodeToDataURL` 的实现：

```js
export async function nodeToDataURL(node, width, height) {
    const xmlns = 'http://www.w3.org/2000/svg';
    const svg = document.createElementNS(xmlns, 'svg');
    const foreignObject = document.createElementNS(xmlns, 'foreignObject');
    svg.setAttribute('width', `${width}`);
    svg.setAttribute('height', `${height}`);
    svg.setAttribute('viewBox', `0 0 ${width} ${height}`);
    foreignObject.setAttribute('width', '100%');
    foreignObject.setAttribute('height', '100%');
    foreignObject.setAttribute('x', '0');
    foreignObject.setAttribute('y', '0');
    svg.appendChild(foreignObject);
    foreignObject.appendChild(node);
    return svgToDataURL(svg);
}
```

**SVG 导出尺寸公式**：

```
output SVG width  = getImageSize().width  = paneWidth   (CSS 像素)
output SVG height = getImageSize().height = paneHeight  (CSS 像素)
viewBox           = "0 0 paneWidth paneHeight"
```

**没有 pixelRatio 放大**！SVG 是矢量格式，不需要 DPR 缩放。
外层 SVG 的 `width/height` 直接用了 CSS 像素值，内部通过 `foreignObject 100%` 填满。

但注意：**这个"SVG"实际上不是纯矢量**。它的内部是 `<foreignObject>` 包裹的 **HTML 克隆**，
HTML 内容是位图语义（包括内部的 reaflow SVG 本身也可能含 `<foreignObject>` 节点）。
所以这个 SVG 文件的"矢量性"仅限于外层坐标缩放不变，**文字和节点渲染仍是位图式嵌入**。

### 2.2 PNG / JPEG 路径（toPng / toJpeg → toCanvas）

```js
export async function toCanvas(node, options = {}) {
    const { width, height } = getImageSize(node, options);  // ← paneWidth, paneHeight
    const svg = await toSvg(node, options);                 // ← 先走 SVG 路径
    const img = await createImage(svg);                     // ← SVG → <img>
    const canvas = document.createElement('canvas');
    const context = canvas.getContext('2d');
    const ratio = options.pixelRatio || getPixelRatio();    // ← 关键！
    const canvasWidth = options.canvasWidth || width;       // ← paneWidth
    const canvasHeight = options.canvasHeight || height;    // ← paneHeight
    canvas.width = canvasWidth * ratio;                     // ← 物理像素！
    canvas.height = canvasHeight * ratio;                   // ← 物理像素！
    if (!options.skipAutoScale) {
        checkCanvasDimensions(canvas);                      // ← 16384 上限裁剪
    }
    canvas.style.width = `${canvasWidth}`;
    canvas.style.height = `${canvasHeight}`;
    if (options.backgroundColor) {
        context.fillStyle = options.backgroundColor;
        context.fillRect(0, 0, canvas.width, canvas.height);
    }
    context.drawImage(img, 0, 0, canvas.width, canvas.height);
    return canvas;
}

export async function toPng(node, options = {}) {
    const canvas = await toCanvas(node, options);
    return canvas.toDataURL();                              // ← 输出 data:image/png;base64,...
}

export async function toJpeg(node, options = {}) {
    const canvas = await toCanvas(node, options);
    return canvas.toDataURL('image/jpeg', options.quality || 1);
}
```

**PNG / JPEG 导出尺寸公式**：

```
ratio          = options.pixelRatio ?? window.devicePixelRatio ?? 1
canvasWidth    = options.canvasWidth ?? paneWidth       (CSS 像素)
canvasHeight   = options.canvasHeight ?? paneHeight     (CSS 像素)

canvas.width   = canvasWidth  × ratio                   (物理像素，输出的实际分辨率)
canvas.height  = canvasHeight × ratio                   (物理像素)

↓ checkCanvasDimensions 裁剪（见第三节）

canvas.style.width  = canvasWidth   (仅 CSS 显示大小，不影响 dataURL 内容)
canvas.style.height = canvasHeight
```

### 2.3 getPixelRatio() 的默认值

```js
export function getPixelRatio() {
    let ratio;
    let FINAL_PROCESS;
    try { FINAL_PROCESS = process; } catch (e) { /* pass */ }
    const val = FINAL_PROCESS && FINAL_PROCESS.env
        ? FINAL_PROCESS.env.devicePixelRatio
        : null;
    if (val) {
        ratio = parseInt(val, 10);
        if (Number.isNaN(ratio)) { ratio = 1; }
    }
    return ratio || window.devicePixelRatio || 1;
}
```

在浏览器环境中 `process` 不存在，所以走 `window.devicePixelRatio || 1`。

| 设备 | `window.devicePixelRatio` | 导出放大倍率 |
|------|--------------------------|------------|
| 普通显示器 | 1 | 1× |
| Retina MacBook | 2 | 2× |
| 高 DPI 外接屏 | 1.25 / 1.5 | 1.25× / 1.5× |
| 4K 200% 缩放 | 2 | 2× |
| 手机 | 2 / 3 | 2× / 3× |

---

## 三、Canvas 自动缩放上限：checkCanvasDimensions()

### 3.1 源码

```js
const canvasDimensionLimit = 16384;

export function checkCanvasDimensions(canvas) {
    if (canvas.width > canvasDimensionLimit ||
        canvas.height > canvasDimensionLimit) {
        if (canvas.width > canvasDimensionLimit &&
            canvas.height > canvasDimensionLimit) {
            if (canvas.width > canvas.height) {
                canvas.height *= canvasDimensionLimit / canvas.width;
                canvas.width = canvasDimensionLimit;
            } else {
                canvas.width *= canvasDimensionLimit / canvas.height;
                canvas.height = canvasDimensionLimit;
            }
        } else if (canvas.width > canvasDimensionLimit) {
            canvas.height *= canvasDimensionLimit / canvas.width;
            canvas.width = canvasDimensionLimit;
        } else {
            canvas.width *= canvasDimensionLimit / canvas.height;
            canvas.height = canvasDimensionLimit;
        }
    }
}
```

### 3.2 逐行解读

该函数的目的是：**当 `canvas.width` 或 `canvas.height` 超过 16384 物理像素时，
等比缩放使最大边 = 16384**。

它不是简单地 clamp，而是**保持宽高比**缩放：

```
if (width > 16384 && height > 16384):
    把较大边缩到 16384，另一边等比缩小

if (仅 width > 16384):
    height *= 16384 / width
    width = 16384

if (仅 height > 16384):
    width *= 16384 / height
    height = 16384
```

### 3.3 什么时候会触发？

只有当 `canvasWidth × ratio > 16384` 或 `canvasHeight × ratio > 16384` 时。

在 JSON Crack 中：
- `canvasWidth = paneWidth = ELK layout.width + 50`
- `canvasHeight = paneHeight = ELK layout.height + 50`
- `ratio = window.devicePixelRatio`

**触发的条件**：ELK 布局宽度 + 50 > 16384 / ratio

| DPR | 触发阈值（ELK layout width） |
|-----|--------------------------|
| 1   | > 16,334 px |
| 1.5 | > 10,856 px |
| 2   | > 8,167 px  |

**实际场景**：一个拥有几百个节点的大 JSON，ELK 布局宽度可能达到 8000~12000px。
在 DPR=2 的 Retina 屏上，`paneWidth × 2 = 16000~24000`，**很可能触发裁剪**。

### 3.4 裁剪的具体效果

假设：`paneWidth = 10000`，`paneHeight = 5000`，`DPR = 2`

```
canvas.width  = 10000 × 2 = 20000
canvas.height = 5000  × 2 = 10000

↓ checkCanvasDimensions:

20000 > 16384 && 10000 ≤ 16384  → 走第三个分支:
canvas.height *= 16384 / 20000 = 10000 × 0.8192 = 8192
canvas.width = 16384
```

结果：
- **输出 PNG 分辨率**：16384 × 8192（而非原始的 20000 × 10000）
- **宽高比保持**：2:1 → 保持为 2:1 ✓
- **内容完整性**：`context.drawImage(img, 0, 0, canvas.width, canvas.height)` 把原始 SVG 图像
  绘制到更小的 canvas 上——**内容不会丢失，只是被缩小了**

### 3.5 skipAutoScale 的作用

DownloadModal 中**没有传 `skipAutoScale`**，所以走默认值 `undefined` → falsy → **不跳过**。

```js
if (!options.skipAutoScale) {
    checkCanvasDimensions(canvas);
}
```

如果传 `skipAutoScale: true`，则完全不做裁剪。但这很危险：
- Chrome 的 canvas 最大面积 ≈ 268,435,456 像素（16384 × 16384）
- 超过后 `canvas.toDataURL()` 返回空字符串或抛错
- 某些浏览器/设备上 canvas 会被强制清零

### 3.6 16384 这个值的来源

html-to-image 硬编码了 `const canvasDimensionLimit = 16384`，
依据是 MDN 文档中 Chrome/Safari 的 **最大面积限制**：
- Chrome: 32,767 × 32,767 单维度，但面积 ≤ 268,435,456 (≈ 16384 × 16384)
- Safari: 同上
- Firefox: 面积可达 472,907,776，但 16384 对所有浏览器安全

html-to-image 选择了**最保守的值 16384**，保证在所有浏览器上不出问题。

---

## 四、applyStyle 对尺寸的二次影响

```js
export function applyStyle(node, options) {
    const { style } = node;
    if (options.backgroundColor) { style.backgroundColor = options.backgroundColor; }
    if (options.width)  { style.width  = `${options.width}px`; }
    if (options.height) { style.height = `${options.height}px`; }
    const manual = options.style;
    if (manual != null) { Object.keys(manual).forEach((key) => { style[key] = manual[key]; }); }
    return node;
}
```

在 JSON Crack 中，`options.width` 和 `options.height` **未传**（undefined），所以不会进入这两个 if。
`applyStyle` 只做了 `backgroundColor` 的设置。

**如果有人传了 `options.width`**：
1. `getImageSize` 会优先使用 `options.width`（而非 `getNodeWidth`）
2. `applyStyle` 会把这个值写到克隆节点的 `style.width` 上
3. 两者一致，不会冲突

---

## 五、cloneCSSStyle 与 transform 的关系

```js
function cloneCSSStyle(nativeNode, clonedNode) {
    const targetStyle = clonedNode.style;
    const sourceStyle = window.getComputedStyle(nativeNode);
    if (sourceStyle.cssText) {
        targetStyle.cssText = sourceStyle.cssText;
        targetStyle.transformOrigin = sourceStyle.transformOrigin;
    } else {
        toArray(sourceStyle).forEach((name) => {
            let value = sourceStyle.getPropertyValue(name);
            // ... font-size 微调等 ...
            targetStyle.setProperty(name, value, sourceStyle.getPropertyPriority(name));
        });
    }
}
```

**关键**：`window.getComputedStyle(nativeNode)` 读取的是**元素自身**的 computed style。

对于 `.jsoncrack-canvas` 这个 div：
- 它自身没有 `transform` 属性（`transform: none`）
- `transform` 在它的**祖先** `.jsoncrack-space` 的内部 wrapper 上
- 所以克隆出来的节点**不会继承祖先的 transform**

`getComputedStyle` **不会合并祖先的 transform**。CSS transform 是视觉叠加层，
不是继承属性。一个元素的 `computed transform` 只反映它自己声明的值。

这再次确认：**视口缩放/平移不影响导出**。

---

## 六、完整尺寸计算公式（汇总）

### 6.1 SVG 导出

```
outputWidth   = paneWidth   = ELK.layout.width  + 50    (CSS 像素)
outputHeight  = paneHeight  = ELK.layout.height + 50    (CSS 像素)
viewBox       = "0 0 {outputWidth} {outputHeight}"

无 DPR 放大
无 16384 裁剪
输出格式: data:image/svg+xml;charset=utf-8,...
```

### 6.2 PNG 导出

```
// Step 1: 计算基础尺寸
baseWidth     = paneWidth   = ELK.layout.width  + 50
baseHeight    = paneHeight  = ELK.layout.height + 50

// Step 2: 计算 DPR
ratio         = window.devicePixelRatio || 1

// Step 3: canvas 物理像素尺寸
canvas.width  = baseWidth  × ratio
canvas.height = baseHeight × ratio

// Step 4: checkCanvasDimensions 裁剪
if (canvas.width > 16384 || canvas.height > 16384):
    等比缩放使 max(canvas.width, canvas.height) = 16384

// Step 5: 输出
canvas.toDataURL() → data:image/png;base64,...
输出图像的像素尺寸 = 裁剪后的 canvas.width × canvas.height
```

### 6.3 JPEG 导出

同 PNG，仅最后一步不同：
```
canvas.toDataURL('image/jpeg', quality)  // quality = 1（默认）
```
JPEG 的 `quality` 参数**不影响尺寸**，只影响压缩率和文件大小。

---

## 七、实战场景：超大画布导出的具体影响

### 场景 A：大型 JSON（300 节点），ELK layout = 12000 × 6000，DPR = 2

```
baseWidth     = 12050
baseHeight    = 6050

canvas.width  = 12050 × 2 = 24100
canvas.height = 6050 × 2  = 12100

↓ checkCanvasDimensions:
  24100 > 16384 → 进入裁剪
  canvas.height *= 16384 / 24100 = 12100 × 0.6799 = 8227
  canvas.width = 16384

最终 PNG: 16384 × 8227 像素（约 1.35 亿像素，~50MB PNG 文件）
原始未裁剪应该是: 24100 × 12100 = 2.92 亿像素 → 超过 Chrome 面积限制
```

**效果**：PNG 分辨率被缩小到原始的 67.99%，但**内容完整不丢失**（等比缩放 drawImage）。
在 Retina 屏上，这个 PNG 看起来会比屏幕上看到的模糊一些——因为原始需要 24100px 的内容
被压缩到了 16384px。

### 场景 B：小型 JSON（10 节点），ELK layout = 800 × 400，DPR = 2

```
baseWidth     = 850
baseHeight    = 450

canvas.width  = 850 × 2 = 1700
canvas.height = 450 × 2 = 900

↓ checkCanvasDimensions: 不触发（均 < 16384）

最终 PNG: 1700 × 900 像素（正常）
```

### 场景 C：节点超限（> 1500 节点），未触发 onLayoutChange

```
nodes = [], edges = []
onLayoutChange 未被调用 → paneWidth/paneHeight 保持初始值 2000

baseWidth     = 2000
baseHeight    = 2000

canvas.width  = 2000 × 2 = 4000   (DPR=2)
canvas.height = 2000 × 2 = 4000

↓ checkCanvasDimensions: 不触发

最终 PNG: 4000 × 4000 像素，内容为空（纯 backgroundColor 色块）
```

### 场景 D：SVG 导出同一大型 JSON

```
outputWidth   = 12050    (CSS 像素)
outputHeight  = 6050     (CSS 像素)

无 DPR 放大
无 16384 裁剪

输出 SVG: 12050 × 6050 viewBox，文件大小取决于内嵌 HTML 的复杂度
```

**SVG 没有 16384 限制**，因为 SVG 不经过 Canvas 绘制，直接序列化。
但 SVG 文件里嵌了完整的 HTML 克隆 + 内联样式，大型图导出时文件可能非常大。

---

## 八、16384 上限 vs 浏览器真实上限的差距

html-to-image 选择的 `16384` 是一个**保守值**，不是所有浏览器的真实极限：

| 浏览器 | 单维度上限 | 面积上限 | html-to-image 使用的值 |
|--------|----------|---------|---------------------|
| Chrome | 32,767 | 268,435,456 (16384²) | 16384 ✓ 刚好匹配面积上限 |
| Firefox | 32,767 | 472,907,776 (~21754²) | 16384 ✗ 远低于 Firefox 的能力 |
| Safari | 32,767 | 268,435,456 | 16384 ✓ |
| Safari iOS | 更低 | ~16M (4096²) | 16384 ✗ **可能超出** |
| 移动端 Chrome | 更低 | ~16M–64M | 16384 ✗ **可能超出** |

**潜在问题**：
- 在 iOS Safari 上，Canvas 面积上限可能只有 ~16M 像素（4096×4096）
- html-to-image 的 `checkCanvasDimensions` 允许 16384 × 16384 = 268M 像素
- 在 iOS 上导出大图可能会**静默失败**（canvas 被清空 → 输出空白 PNG）
- 这是 html-to-image 的已知问题，它没有做浏览器嗅探来适配不同设备

---

## 九、DownloadModal 中缺失的 options 及其影响

| Option | 当前值 | 默认行为 | 影响 |
|--------|--------|---------|------|
| `width` | 未传 | `getNodeWidth(node)` | 使用元素自身 clientWidth |
| `height` | 未传 | `getNodeHeight(node)` | 使用元素自身 clientHeight |
| `pixelRatio` | 未传 | `window.devicePixelRatio` | DPR=2 时 PNG 面积 ×4 |
| `canvasWidth` | 未传 | `getImageSize().width` | 等于 paneWidth |
| `canvasHeight` | 未传 | `getImageSize().height` | 等于 paneHeight |
| `skipAutoScale` | 未传 | `false` → 走裁剪 | 16384 上限生效 |
| `quality` | `1` | JPEG 压缩质量 | 仅 JPEG 生效，PNG 忽略 |
| `skipFonts` | `true` | 跳过字体嵌入 | 加速，字体可能降级 |
| `style` | 未传 | 不追加额外样式 | 无额外覆盖 |
| `filter` | 未传 | 不过滤任何节点 | 克隆所有子节点 |

**最关键的缺失**：`pixelRatio`。
用户在 DPR=2 的 Retina 屏上导出，PNG 像素是 CSS 像素的 2×2=4 倍。
这既是优点（更清晰），也是隐患（更容易触发 16384 裁剪 + 更大文件体积）。

---

## 十、优化建议

### 建议 1：暴露 pixelRatio 控制项（P1）

在 DownloadModal 中加一个 SegmentedControl：

```tsx
const imageOptions = {
  quality: fileDetails.quality,
  backgroundColor: fileDetails.backgroundColor,
  skipFonts: true,
  pixelRatio: fileDetails.pixelRatio,  // 1 | 2 | auto
};
```

- `1`：强制 1×，避免 DPR 放大（文件小，但在 Retina 上略模糊）
- `auto`：使用 `window.devicePixelRatio`（当前默认行为）
- `2`：强制 2×，任何设备都高清

### 建议 2：导出前检测是否会被裁剪并警告（P1）

```ts
const ratio = window.devicePixelRatio || 1;
const willBeClipped = paneWidth * ratio > 16384 || paneHeight * ratio > 16384;
if (willBeClipped) {
  toast.warning("画布过大，导出分辨率将被自动缩小至 16384px 上限");
}
```

### 建议 3：超大画布走 SVG 路径绕过 Canvas 限制（P2）

当检测到 `paneWidth * ratio > 16384` 时，自动降级为 SVG 导出，
因为 SVG 路径不经过 Canvas，不受 16384 限制。

### 建议 4：移动端适配（P2）

iOS Safari 的 Canvas 面积上限远低于 16384²。可以在导出前创建一个测试 canvas
探测实际上限，而非依赖硬编码值：

```ts
function detectMaxCanvasArea() {
  const test = document.createElement('canvas');
  let area = 16384 * 16384;
  while (area > 0) {
    const side = Math.floor(Math.sqrt(area));
    test.width = side; test.height = side;
    try {
      test.getContext('2d')?.fillRect(0, 0, 1, 1);
      const data = test.getContext('2d')?.getImageData(0, 0, 1, 1);
      if (data?.data[3] === 255) return area;
    } catch {}
    area = Math.floor(area / 2);
  }
  return 4096 * 4096; // fallback
}
```

---

## 十一、调试指南

### 验证实际 PNG 尺寸

在 `exportAsImage` 的 `dataURI` 返回后：

```ts
// 解码 base64 获取 PNG header 中的宽高
const binary = atob(dataURI.split(',')[1]);
const byteArr = new Uint8Array(binary.length);
for (let i = 0; i < binary.length; i++) byteArr[i] = binary.charCodeAt(i);
// PNG 宽度在 byte 16-19 (big-endian)
const pngWidth  = (byteArr[16] << 24) | (byteArr[17] << 16) | (byteArr[18] << 8) | byteArr[19];
const pngHeight = (byteArr[20] << 24) | (byteArr[21] << 16) | (byteArr[22] << 8) | byteArr[23];
console.log(`PNG actual size: ${pngWidth} × ${pngHeight}`);
```

### 验证 Canvas 裁剪是否发生

在浏览器控制台断点到 html-to-image 的 `checkCanvasDimensions` 内，
查看裁剪前后 `canvas.width / canvas.height` 的变化。
