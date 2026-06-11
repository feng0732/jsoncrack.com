# 导出变换边界语义分析

> 版本：1.0
> 日期：2026-06-11

## 一、DOM 层级与 Transform 分布全景

### 1.1 完整 DOM 层级（从外到内）

```
<body>
  <StyledEditorWrapper>                    /* width: 100%, height: 100% */
    <div.containerRef div>
      <div.canvasWrapper>                /* position: relative; width: 100%; height: 100% */
        <div.jsoncrack-canvas>         /* ← 导出目标 */
          <Space.jsoncrack-space>        /* ← Space 组件外层 */
            <div.space-viewport>    /* ← Space 视口层 */
              <div.space-inner>     /* ← Space transform 应用层！ */
                <div>                /* ← 无变换 */
                  <Canvas.jsoncrack-canvas>  /* ← Reaflow Canvas div */
                    <svg>               /* ← Reaflow SVG */
                      <defs>...</defs>
                      <g>                  /* ← 内容组（motion.g） */
                        <g id="node-1" transform="translate(100, 50)">  /* ← 节点位移 */
                          <foreignObject width="200" height="30">...</foreignObject>
                        </g>
                        <g id="edge-1" transform="translate(0, 0)">
                          <path d="..." />
                        </g>
                      </g>
                    </svg>
                  </div>
                </div>
              </div>
            </Space>
          </div>
        </div>
      </div>
    </div>
  </StyledEditorWrapper>
```

### 1.2 关键 Transform 位置标注

| 层级 | 元素 | Transform 类型 | 应用位置 |
|------|------|----------------|----------|
| L1 | `.space-inner` | CSS `transform: matrix(sx, 0, 0, sy, tx, ty) | **Space 内部层，**目标元素外部** |
| L2 | `<g>` 内容组 | SVG `transform="translate(x, y)"` 或 CSS transform | **目标元素内部**，Reaflow 内部 |
| L3 | `<g id="node-x">` 节点 | SVG `transform="translate(x, y)" | **目标元素内部**，每个节点的位置 |
| L4 | 无 | 无 | 边和节点内部无 transform，位置由 path 坐标定义 |

---

## 二、导出目标选择

### 2.1 代码位置：[DownloadModal/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L65-L67)

```typescript
const getExportElement = () =>
  (document.querySelector(".jsoncrack-canvas") as HTMLElement | null) ??
  (document.querySelector("svg[id*='ref']") as HTMLElement | null);
```

**导出目标是 `.jsoncrack-canvas` div**，而非内部 `<svg>`。

### 2.2 导出目标的 DOM 边界：

```
.jsoncrack-canvas
└── <Space>
    └── .space-viewport
        └── .space-inner    /* ← transform 在这里！
            └── <div>
                └── <Canvas.jsoncrack-canvas>
                    └── <svg>
                        └── <g> 内容组
                            └── <g id="node-1" transform="translate(...)"></g>
```

**重要：`.space-inner` 的 transform **在导出目标的**外部**（因为它在 Space 内部，但是 `.jsoncrack-canvas` 的子级）。

---

## 三、各层 Transform 详细分析

### 3.1 Space 层的 Transform（外部）

**来源**：`react-zoomable-ui` 的 `<Space>` 组件

**应用位置**：`.space-inner` div（Space 内部的 inner 元素）

**变换类型**：CSS `transform: matrix(a, b, c, d, tx, ty)`

- `a, d` = 缩放分量（scaleX, scaleY）
- `tx, ty` = 平移分量

**代码证据**：[canvasHelpers.ts](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L236)

```typescript
const virtualRect = viewPort.translateClientRectToVirtualSpace(rect);
```

`translateClientRectToVirtualSpace` 会反向应用 Space 的 transform，说明 Space 的 transform 是通过 CSS `transform` 属性施加的。

**特点**：
- 纯视觉变换，不影响 `clientWidth` / `offsetWidth`
- 影响 `getBoundingClientRect()` 的返回值
- **在导出目标 `.jsoncrack-canvas` 的外部**

### 3.2 Reaflow 内容组的 Transform（内部）

**来源**：Reaflow Canvas 组件内部

**应用位置**：`<g>` 元素（内容组容器）

**变换类型**：
- 默认：SVG `transform="translate(x, y)"`（用于自动居中）
- 当前配置：**无 transform** 或 `translate(0, 0)`

**代码证据**：[JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L606-L610)

```typescript
// Disable reaflow's built-in auto-centering of content inside the
// pane. Passing a nullish `defaultPosition` makes reaflow leave
// the group at the svg origin so our fit-to-viewport rect math
// matches reality.
defaultPosition={null as unknown as undefined}
```

由于 `defaultPosition={null}`，Reaflow **不会**对内容组应用自动居中的 translate。

**补充**：内容组还可能被 framer-motion 施加 CSS transform（用于动画），但当前配置 `animated={false}` 禁用了动画。

### 3.3 节点级别的 Transform（内部）

**来源**：Reaflow 对每个节点/边的包装 `<g>`

**应用位置**：`<g id="node-xxx">` 或 `<g id="edge-xxx">`

**变换类型**：SVG `transform="translate(x, y)"

**代码证据**：[canvasHelpers.ts](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L112)

```typescript
// Each reaflow node/edge is wrapped in a `g` with an id
```

**作用**：定位每个节点和边的位置。`x, y` 来自 ELK 布局结果。

---

## 四、html-to-image 克隆边界分析

### 4.1 克隆起点

`html-to-image` 的 `toSvg` / `toPng` 等方法的克隆起点是**传入的目标节点本身**：

```typescript
const imageElement = getExportElement(); // .jsoncrack-canvas div
toBlob(imageElement, options);
```

### 4.2 克隆流程（6 步）

1. **深度克隆目标节点**：`node.cloneNode(true)`
   - 只克隆目标节点及其**后代**
   - **不克隆任何祖先节点**（包括 Space 的父级）

2. **内联计算样式**：对克隆树中的每个节点，调用 `getComputedStyle(originalNode)`，将计算后的样式写入 `clone.style.cssText`
   - 包括 `transform` 属性如果存在会被内联

3. **嵌入字体和图片**：将字体、图片转为 data URL

4. **构建 SVG 外壳**：创建新 `<svg viewBox="0 0 W H"> + `<foreignObject width="100%" height="100%">`，将克隆树塞入

5. **XML 序列化**：`XMLSerializer.serializeToString()`

6. **编码为 data URL**

### 4.3 关键代码模式（还原自 html-to-image v1.11.11）

```typescript
// 克隆节点
function cloneNode<T extends Node>(node: T): T {
  return node.cloneNode(true) as T;
}

// 内联样式（核心）
function cloneStyle(node: HTMLElement, clone: HTMLElement) {
  const style = window.getComputedStyle(node);
  clone.style.cssText = style.cssText;  // 包括 transform！
  
  // 递归处理子节点
}

// 创建 SVG 外壳
function createForeignObjectSVG(
  width: number,
  height: number,
  x: number,
  y: number,
  node: Node
): SVGElement {
  const xmlns = 'http://www.w3.org/2000/svg';
  const svg = document.createElementNS(xmlns, 'svg');
  const foreignObject = document.createElementNS(xmlns, 'foreignObject');
  
  svg.setAttributeNS(null, 'width', width.toString());
  svg.setAttributeNS(null, 'height', height.toString());
  svg.setAttributeNS(null, 'viewBox', `0 0 ${width} ${height}`);
  
  foreignObject.setAttributeNS(null, 'width', '100%');
  foreignObject.setAttributeNS(null, 'height', '100%');
  foreignObject.setAttributeNS(null, 'x', x.toString());
  foreignObject.setAttributeNS(null, 'y', y.toString());
  
  svg.appendChild(foreignObject);
  foreignObject.appendChild(node);
  
  return svg;
}
```

### 4.4 克隆边界图示

```
真实 DOM（屏幕上）：

  <div.space-inner style="transform: matrix(0.8, 0, 0, 0.8, 100, 50)">
    ├─────────────────────────────────────────┐
    │  <div>                            │  克隆边界（外部，不被克隆）
    │    <div.jsoncrack-canvas> ← 目标  │
    │      <Space>                        │
    │        <svg>                      │
    │          <g transform="translate(100,50)">  │
    │            <foreignObject>...</foreignObject> │
    │          </g>                       │
    │        </svg>                      │
    │      </Space>                       │
    │    </div>                         │
    └─────────────────────────────────────────┘
  </div>

克隆后的 DOM（内存中）：

  <foreignObject width="100%" height="100%">
    <div>
      <div.jsoncrack-canvas style="...（所有计算样式被内联">
        <Space>
          <svg>
            <g transform="translate(100,50)">  ← 保留！
              <foreignObject>...</foreignObject>
            </g>
          </svg>
        </Space>
      </div>
    </div>
  </foreignObject>
```

---

## 五、Transform 穿透性分析

### 5.1 哪些 Transform 会进入导出 SVG

| Transform | 位置 | 是否被克隆 | 进入导出 | 说明 |
|-----------|------|----------|----------|------|
| **Space `.space-inner` CSS transform | **目标外部**（在 `.jsoncrack-canvas` 之外 | ❌ 否 | ❌ 否 | 在目标元素的父级，不被 clone |
| **内容组 `<g>` SVG transform | **目标内部**（`<svg>` 内部 | ✅ 是 | ✅ 是 | 目标内部元素，被 clone |
| **节点 `<g id="node-x">` SVG transform | **目标内部** | ✅ 是 | ✅ 是 | 目标内部元素，被 clone |
| **节点 `<g id="edge-x">` SVG transform | **目标内部** | ✅ 是 | ✅ 是 | 目标内部元素，被 clone |
| **`<foreignObject>` 的 `x/y 属性 | **目标内部** | ✅ 是 | ✅ 是 | 目标内部元素，被 clone |

### 5.2 Space Transform 的特殊说明

**Space 的 transform 不会进入导出的关键原因：

```
真实 DOM 结构：

  .jsoncrack-canvas  ← 导出目标（clone 起点）
    └── Space
        └── .space-viewport
            └── .space-inner  ← transform: matrix(...) 在这里！
                └── <div>
                    └── Canvas
                        └── svg
                            └── ...
```

**导出目标是 `.jsoncrack-canvas`，它是 Space 的直接父元素。Space 的 transform 应用在 `.space-inner`，它在导出目标的**内部**，在 Space 组件内部。

**修正**：重新看 DOM 结构：

```
.jsoncrack-canvas  (div)
└── <Space.jsoncrack-space>  (Space 组件外层 div
    └── .space-viewport  (Space viewport)
        └── .space-inner  (Space inner，transform 在这里)
            └── <div>  (无 class)
                └── <div.jsoncrack-canvas>  (Reaflow Canvas div)
                    └── <svg>
```

**哦，原来如此！** `.space-inner` 的 transform **在导出目标 `.jsoncrack-canvas` 的**内部**！因为：

1. 导出目标是**外层** `.jsoncrack-canvas`（L15-outer）
2. Space 是它的子元素
3. `.space-inner` 是 Space 的内部层
4. Reaflow Canvas 是 `.space-inner` 的子元素，也叫 `.jsoncrack-canvas`（L15-inner）

**结论**：`.space-inner` 的 transform **在导出目标内部**，理论上应该被克隆！

### 5.3 修正后的 Transform 穿透性

| Transform | 相对导出目标 | 是否被克隆 | 进入导出 | 说明 |
|-----------|------------|----------|----------|------|
| **Space `.space-inner` CSS transform | **内部**（Space 内部） | ✅ 是 | ✅ 是 | 目标内部元素 |
| **内容组 `<g>` SVG transform | **内部** | ✅ 是 | ✅ 是 | 目标内部元素 |
| **节点 `<g>` SVG transform | **内部** | ✅ 是 | ✅ 是 | 目标内部元素 |

**但等等**，这和之前的结论矛盾。让我再仔细看一下：

```typescript
// JSONCrackComponent.tsx:

<div className={canvasClassName}  // ← 外层 .jsoncrack-canvas
  ref={containerRef}
>
  <Space ...>              // ← Space 在这里，子元素
    <Canvas ... />        // ← Reaflow Canvas，也带 .jsoncrack-canvas class
  </Space>
</div>
```

```typescript
// DownloadModal:
const getExportElement = () =>
  (document.querySelector(".jsoncrack-canvas") as HTMLElement | null) ??
  ...
```

`document.querySelector(".jsoncrack-canvas") 会匹配**第一个**匹配的元素，也就是**外层**的那个 div（L15-outer）。

所以导出目标包含：

```
外层 .jsoncrack-canvas (div)
└── Space
    └── .space-viewport
        └── .space-inner  ← CSS transform: matrix(...) 在这里！
            └── div
                └── 内层 .jsoncrack-canvas (div，Reaflow Canvas)
                    └── svg
```

**结论**：`.space-inner` 的 transform **在导出目标内部**，会被 html-to-image 克隆！

---

## 六、getComputedStyle 对 Transform 的处理

### 6.1 getComputedStyle 返回的 transform 值

当对 `.space-inner` 有 `transform: matrix(0.8, 0, 0, 0.8, 100, 50)` 时：

```javascript
const style = window.getComputedStyle(spaceInnerDiv);
console.log(style.transform);  // "matrix(0.8, 0, 0, 0.8, 100, 50)"
```

### 6.2 html-to-image 会将这个值内联到克隆节点：

```javascript
clone.style.cssText = style.cssText;
// 结果：clone.style.transform = "matrix(0.8, 0, 0, 0.8, 100, 50)"
```

### 6.3 这意味着什么？

导出的 SVG 中会包含 Space 的 transform！

```xml
<svg viewBox="0 0 W H" xmlns="http://www.w3.org/2000/svg">
  <foreignObject width="100%" height="100%">
    <div xmlns="http://www.w3.org/1999/xhtml">
      <!-- 克隆的 .jsoncrack-canvas -->
      <div class="jsoncrack-canvas" style="...">
        <div class="jsoncrack-space" style="...">
          <div class="space-viewport" style="...">
            <!-- Space 的 transform 在这里！ -->
            <div class="space-inner" 
                 style="...; transform: matrix(0.8, 0, 0, 0.8, 100, 50); ...">
              <div>
                <div class="jsoncrack-canvas" style="...">
                  <svg ...>
                    <!-- Reaflow 内容 -->
                  </svg>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </foreignObject>
</svg>
```

### 6.4 这会导致什么问题？

**问题 1：双重缩放

```
真实屏幕视图：
  Space transform(scale(0.8)) → 视觉缩小到 80%

导出 SVG：
  外层 foreignObject 100% → 内层 space-inner transform(scale(0.8)) → 内容也缩小到 80%
```

结果：导出的 SVG 中内容只占 80% × 80% 的区域，周围有空白边距。

**问题 2**：平移分量 `translate(100, 50)` 会导致内容在 SVG 中偏移，产生空白。

---

## 七、导出尺寸测量

### 7.1 html-to-image 如何测量导出尺寸？

```typescript
function getNodeWidth(node: HTMLElement) {
  const leftBorder = parseFloat(getComputedStyle(node).borderLeftWidth);
  const rightBorder = parseFloat(getComputedStyle(node).borderRightWidth);
  return node.clientWidth + leftBorder + rightBorder;
}

function getNodeHeight(node: HTMLElement) {
  const topBorder = parseFloat(getComputedStyle(node).borderTopWidth);
  const bottomBorder = parseFloat(getComputedStyle(node).borderBottomWidth);
  return node.clientHeight + topBorder + bottomBorder;
}
```

**关键**：使用 `clientWidth`，**不受 CSS transform 影响**。

所以导出的 SVG `viewBox` 尺寸 = `clientWidth × clientHeight` = `paneWidth × paneHeight`。

### 7.2 但内容被 Space transform 缩放了

```
导出 SVG 尺寸：2050 × 2050（paneWidth=2000 + 50）

但内部 .space-inner 有 transform: scale(0.8)
所以实际内容渲染尺寸：2050 × 0.8 = 1640 × 1640

结果：SVG 中有 2050 × 2050 的画布，但内容只占 1640 × 1640，周围有空白。
```

---

## 八、实际测试验证

### 8.1 验证方法

在浏览器控制台运行：

```javascript
// 1. 找到导出目标
const target = document.querySelector(".jsoncrack-canvas");

// 2. 找到 space-inner
const spaceInner = document.querySelector(".space-inner");

// 3. 查看 space-inner 的 transform
console.log(getComputedStyle(spaceInner).transform;
// 预期："matrix(0.8, 0, 0, 0.8, 100, 50)"

// 4. 查看 target 的 clientWidth
console.log(target.clientWidth);
// 预期：2050（paneWidth + 50）

// 5. 导出后检查 SVG 内容
// 查看是否包含 space-inner 的 transform
```

### 8.2 预期结果

导出的 SVG 中：
- `viewBox="0 0 2050 2050"`（正确的画布尺寸）
- 但内部 `.space-inner` 有 `transform: matrix(0.8, 0, 0, 0.8, 100, 50)`
- 导致内容只占 80% 的区域，周围有空白

---

## 九、边界总结表

| 层级 | 元素 | Transform | 相对导出目标 | 是否进入导出 | 影响范围 |
|------|------|---------|-------------|-------------|----------|
| 1 | `.space-inner` | CSS `transform: matrix(...)` | 内部 | ✅ 是 | 整个内容的缩放和平移 |
| 2 | 内容组 `<g>` | SVG `transform="translate(x,y)"` | 内部 | ✅ 是 | 所有节点的整体偏移（当前为 0,0） |
| 3 | 节点 `<g id="...">` | SVG `transform="translate(x,y)"` | 内部 | ✅ 是 | 单个节点的位置 |
| 4 | 边 `<g id="...">` | SVG `transform="translate(x,y)"` | 内部 | ✅ 是 | 单条边的位置 |
| 5 | `<foreignObject>` | `x`/`y` 属性 | 内部 | ✅ 是 | foreignObject 内部 HTML 内容的位置 |

### 9.1 关键结论

1. **Space 的 transform 会进入导出**！因为 `.space-inner` 在导出目标内部
2. **这是一个潜在 Bug**：用户缩放界面后导出，SVG 中内容会缩小/偏移，周围有空白
3. **只有内容组和节点的 transform 正确**：它们是内容布局的一部分，应该被正确导出
4. **导出尺寸固定为 `paneWidth × paneHeight**：不受 Space transform 影响

### 9.2 修复建议（如果需要修复）

在调用 html-to-image 之前，临时移除 Space 的 transform：

```typescript
const exportElement = getExportElement();
const spaceInner = exportElement.querySelector('.space-inner');

// 保存原始 transform
const originalTransform = spaceInner.style.transform;

// 临时移除
spaceInner.style.transform = 'none';

// 导出
const blob = await toBlob(exportElement, options);

// 恢复
spaceInner.style.transform = originalTransform;
```

---

## 十、完整导出变换决策链

```
用户点击导出
    ↓
getExportElement() → 外层 .jsoncrack-canvas div
    ↓
html-to-image toSvg/toPng
    ├─ 克隆目标节点（深度克隆）
    │   ├─ 克隆 .space-inner（带 transform）
    │   ├─ 克隆内容组 <g>（带 translate）
    │   └─ 克隆节点 <g>（带 translate）
    ├─ getComputedStyle 内联所有样式
    │   └─ .space-inner 的 transform 被内联 ✅
    ├─ 测量 clientWidth × clientHeight = paneWidth × paneHeight
    ├─ 创建 <svg viewBox="0 0 W H">
    └─ 包装进 <foreignObject width="100%" height="100%">
    ↓
最终 SVG 包含 Space transform！
```
