# 导出目标边界校准分析

> 版本：1.0
> 日期：2026-06-11
> 说明：本文校准之前对导出目标 DOM 节点的判断，重新梳理 Space / Reaflow Canvas / SVG 的真实层级关系

---

## 一、代码证据：类名分配的真实情况

### 1.1 外层容器（containerRef div）

**代码位置**：[JSONCrackComponent.tsx:537-545](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L537-L545)

```tsx
<div
  ref={containerRef}
  className={canvasClassName}    // ← 关键！不是 "jsoncrack-canvas"
  style={canvasStyle}
  role="img"
  ...
>
```

**canvasClassName 的定义**：[canvasHelpers.ts:29-30](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L29-L30)

```typescript
export const buildCanvasClassName = (showGrid: boolean, userClassName?: string): string =>
  [styles.canvasWrapper, showGrid ? styles.showGrid : "", userClassName].filter(Boolean).join(" ");
```

**实际类名**：
- `styles.canvasWrapper` → CSS Modules 编译后的哈希类名（如 `_canvasWrapper_abc123`）
- 可选：`styles.showGrid`
- 可选：用户传入的 `className`

**结论**：外层 containerRef div 的类名**不包含** `"jsoncrack-canvas"`。

### 1.2 Space 组件

**代码位置**：[JSONCrackComponent.tsx:570-583](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L570-L583)

```tsx
<Space
  ...
  className="jsoncrack-space"   // ← 类名是 jsoncrack-space，不是 jsoncrack-canvas
  ...
>
```

**结论**：Space 的类名是 `"jsoncrack-space"`，**不是** `"jsoncrack-canvas"`。

### 1.3 Reaflow Canvas 组件

**代码位置**：[JSONCrackComponent.tsx:585-611](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L585-L611)

```tsx
<Canvas
  className="jsoncrack-canvas"   // ← 这里！只有这个地方用了 "jsoncrack-canvas"
  onLayoutChange={onLayoutChange}
  node={renderNode}
  ...
  width={paneWidth}
  height={paneHeight}
  ...
/>
```

**结论**：`"jsoncrack-canvas"` 类名**唯一**出现在 Reaflow `<Canvas>` 组件上。

### 1.4 canvasHelpers.ts 的旁证

**代码位置**：[canvasHelpers.ts:169](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L169)

```typescript
const svg = container.querySelector(".jsoncrack-canvas svg") as SVGSVGElement | null;
```

这里 `container` 就是 `containerRef.current`（外层 wrapper div），它通过 `querySelector(".jsoncrack-canvas svg")` 在**内部**查找 svg。这直接证明：
- `.jsoncrack-canvas` 在外层 container 的**内部**
- Space 夹在外层 container 和 `.jsoncrack-canvas` 之间

---

## 二、真实 DOM 层级（校准后）

### 2.1 完整 DOM 树（从外到内）

```
L1: <div ref={containerRef} className="canvasWrapper ...">        // 外层容器，不带 jsoncrack-canvas
  │
  ├─ <Controls />                    // 控制按钮（可选）
  ├─ <div className="tooLarge">      // 节点数超限提示（可选）
  ├─ <div className="overlay">       // loading 遮罩（可选）
  │
  └─ L2: <Space className="jsoncrack-space">                       // Space 组件，CSS transform 在这里的子层
        │
        └─ <.space-viewport>            // react-zoomable-ui 内部 viewport
              │
              └─ <.space-inner>         // ← CSS transform: matrix(...) 在这里！
                    │
                    └─ L3: <div className="jsoncrack-canvas">      // Reaflow Canvas div ← 导出目标！
                          │
                          └─ L4: <svg width="paneWidth" height="paneHeight">    // Reaflow SVG（无 viewBox）
                                │
                                ├─ <defs>...</defs>
                                │
                                └─ L5: <g>                                    // 内容组（motion.g）
                                      │
                                      ├─ <g id="node-xxx" transform="translate(x, y)">   // 节点位移
                                      │     └─ <foreignObject width="w" height="h">
                                      │
                                      └─ <g id="edge-xxx" transform="translate(x, y)">   // 边位移
                                            └─ <path d="..." />
```

### 2.2 类名与层级对应表

| 层级 | 元素 | 类名 / 标识 | 选择器 `.jsoncrack-canvas` 是否命中 |
|------|------|-----------|-------------------------------------|
| L1 | 外层 container div | `canvasWrapper`（CSS Modules 哈希） | ❌ 否 |
| L2 | Space | `jsoncrack-space` | ❌ 否 |
| L2-inner | `.space-inner` | （无公开类名，react-zoomable-ui 内部） | ❌ 否 |
| **L3** | **Reaflow Canvas div** | **`jsoncrack-canvas`** | **✅ 是**（唯一命中） |
| L4 | Reaflow SVG | `svg`（无类名） | ❌ 否 |
| L5 | 内容组 `<g>` | （无类名） | ❌ 否 |

---

## 三、导出选择器实际命中的节点

### 3.1 导出代码

**代码位置**：[DownloadModal/index.tsx:65-67](file:///d:/fz/0601/solo-dogfeeding/code/189-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L65-L67)

```typescript
const getExportElement = () =>
  (document.querySelector(".jsoncrack-canvas") as HTMLElement | null) ??
  (document.querySelector("svg[id*='ref']") as HTMLElement | null);
```

### 3.2 命中结果

**`document.querySelector(".jsoncrack-canvas")`** → **L3 Reaflow Canvas div**

- 唯一匹配 `.jsoncrack-canvas` 的元素
- 这是 Reaflow `<Canvas>` 组件渲染出的外层 div
- 它的内部包裹着真正的 `<svg>` 元素

**兜底选择器** `svg[id*='ref']`：如果 Reaflow Canvas div 找不到，尝试找一个 id 含 "ref" 的 svg。正常情况下不会触发。

---

## 四、html-to-image 克隆边界

### 4.1 克隆起点

```typescript
const imageElement = getExportElement();  // → L3 Reaflow Canvas div
toBlob(imageElement, options);
```

### 4.2 克隆范围（html-to-image 机制）

html-to-image 的克隆遵循 `node.cloneNode(true)` 的语义：

```
cloneNode(true) 克隆范围：
  ├─ 目标节点本身（L3 Reaflow Canvas div）
  └─ 目标节点的所有后代
        ├─ L4 <svg>
        │   ├─ <defs>
        │   └─ L5 <g> 内容组
        │       ├─ <g id="node-xxx" transform="translate(x,y)">
        │       │   └─ <foreignObject>
        │       └─ <g id="edge-xxx" transform="translate(x,y)">
        │           └─ <path>
        └─ ...其他后代节点
```

**不克隆的部分**（目标节点的祖先）：
- ❌ L1 外层 container div
- ❌ L2 Space 组件
- ❌ L2 Space 内部的 `.space-inner`（CSS transform 在这里！）

### 4.3 边界图示

```
真实 DOM（屏幕上）：

  L1: <div.canvasWrapper>                           /* 不被克隆 */
    └─ L2: <Space.jsoncrack-space>                 /* 不被克隆 */
          └─ .space-viewport                        /* 不被克隆 */
                └─ .space-inner                     /* 不被克隆 — transform: matrix(...) 在这里 */
                      ├─────────────────────────────────┐
                      │  L3: <div.jsoncrack-canvas>  │  ← 克隆边界起点
                      │    └─ L4: <svg>            │
                      │        └─ L5: <g>           │
                      │            └─ ...          │  全部被克隆
                      └─────────────────────────────────┘
```

---

## 五、Transform 穿透性校准

### 5.1 正确的 Transform 穿透表

| Transform | 应用位置 | 相对导出目标（L3） | 是否被克隆 | 是否进入导出 SVG | 说明 |
|-----------|---------|------------------|----------|-----------------|------|
| ❌ **Space `.space-inner` CSS `matrix(...)`** | `.space-inner` div | **祖先（外部）** | **否** | **否** | 在 L3 的父级方向，cloneNode(true) 不向上克隆 |
| ✅ **内容组 `<g>` SVG `translate(x,y)`** | `<svg>` 内的 `<g>` | **后代（内部）** | **是** | **是** | 当前为 `translate(0,0)`（`defaultPosition={null}`） |
| ✅ **节点 `<g id="...">` SVG `translate(x,y)`** | 每个节点的 `<g>` | **后代（内部）** | **是** | **是** | 来自 ELK 布局的节点定位 |
| ✅ **边 `<g id="...">` SVG `translate(x,y)`** | 每条边的 `<g>` | **后代（内部）** | **是** | **是** | 来自 ELK 布局的边定位 |
| ✅ **`<foreignObject>` `x/y` 属性** | 每个节点内的 `<foreignObject>` | **后代（内部）** | **是** | **是** | 一般为 `x=0, y=0` |

### 5.2 关键校准结论

**Space 的 transform 不会进入导出！**

原因链：
1. 导出目标是 **L3 Reaflow Canvas div**（`.jsoncrack-canvas`）
2. Space 的 CSS transform 应用在 **`.space-inner`**，它是 L3 的**祖先**（父级方向：Space → space-viewport → space-inner → L3）
3. `html-to-image` 只克隆目标节点及其**后代**，**不克隆祖先**
4. 因此 `.space-inner` 上的 `transform: matrix(...)` 完全不会出现在导出结果中

### 5.3 与之前结论的对比

| 项目 | 之前的错误结论 | 校准后的正确结论 |
|------|-------------|---------------|
| 导出目标 | 外层 canvasWrapper div（误将两个 `.jsoncrack-canvas` 混淆） | **L3 Reaflow Canvas div**（唯一命中 `.jsoncrack-canvas`） |
| Space 相对导出目标 | 在导出目标内部 | **在导出目标外部（祖先方向）** |
| `.space-inner` transform 是否进入导出 | 是（潜在 Bug） | **否（完全不进入）** |
| 导出结果是否含缩放偏移空白 | 是（因 Space transform） | **否（导出内容是原始 1:1 尺寸）** |

---

## 六、导出 SVG 的实际内容结构

### 6.1 导出 SVG 内部的真实层级

```xml
<svg xmlns="http://www.w3.org/2000/svg"
     width="2050" height="2050"
     viewBox="0 0 2050 2050">                   <!-- html-to-image 生成的外层 SVG -->
  <foreignObject width="100%" height="100%">        <!-- html-to-image 生成的外层 foreignObject -->
    <div xmlns="http://www.w3.org/1999/xhtml">

      <!-- L3: 克隆的 Reaflow Canvas div（计算样式已内联） -->
      <div class="jsoncrack-canvas" style="position: relative; width: 2050px; height: 2050px; ...">

        <!-- L4: 克隆的 Reaflow SVG -->
        <svg width="2050" height="2050" xmlns="http://www.w3.org/2000/svg">
          <!-- 注意：Reaflow SVG 没有 viewBox，坐标系 1:1 像素映射 -->

          <defs>...</defs>

          <!-- L5: 内容组，当前 translate(0,0) -->
          <g transform="translate(0, 0)">

            <!-- 节点 -->
            <g id="node-1" transform="translate(100, 50)">
              <foreignObject width="200" height="30" x="0" y="0">
                <!-- XHTML 节点内容 -->
                <div xmlns="http://www.w3.org/1999/xhtml">...</div>
              </foreignObject>
            </g>

            <!-- 边 -->
            <g id="edge-1" transform="translate(0, 0)">
              <path d="M150,65 C200,65 200,120 300,120" />
            </g>

          </g>
        </svg>

      </div>
    </div>
  </foreignObject>
</svg>
```

### 6.2 尺寸关系

```
html-to-image 外层 SVG:
  viewBox = "0 0 2050 2050"
  来源：L3 Reaflow Canvas div 的 clientWidth × clientHeight

  └─ foreignObject width="100%" height="100%"

      └─ L3 div（内联样式后宽 2050px，高 2050px）

          └─ L4 Reaflow SVG（width=2050, height=2050，无 viewBox）
              坐标系 1:1 映射，1 SVG 单位 = 1 CSS 像素

              └─ 内容组 translate(0,0)
                  └─ 节点 translate(x, y) —— 正确定位
```

---

## 七、用户缩放操作对导出的真实影响

### 7.1 用户在界面上缩放/平移

用户通过鼠标滚轮或手势操作时，react-zoomable-ui 的 Space 组件会修改 `.space-inner` 的 CSS transform：

```css
.space-inner {
  transform: matrix(0.8, 0, 0, 0.8, 50, 30);  /* scale(0.8) + translate(50px, 30px) */
}
```

**这只改变屏幕视觉效果**，对 L3 及以下的 DOM 没有任何修改。

### 7.2 导出时的真实情况

```
屏幕视图：                    导出结果：
  .space-inner transform(0.8)    L3 div 宽 2050, 高 2050
    → 视觉缩小到 80%                （原始尺寸，无缩放）
  内容有偏移                        内容从原点开始（translate 0,0）
                                    节点定位完全按 ELK 布局
```

**结论**：用户在界面上的缩放和平移操作**完全不影响导出结果**。导出始终是原始尺寸 `paneWidth × paneHeight` 的 1:1 内容。

---

## 八、校准后的完整决策链

```
用户点击导出
    ↓
getExportElement()
    ├─ document.querySelector(".jsoncrack-canvas")
    └─ → 命中 L3 Reaflow Canvas div（Space 的后代，非 Space 本身）
    ↓
html-to-image toSvg/toPng/toBlob
    ├─ 1. 克隆目标节点（L3 div）及其后代
    │   ├─ ✅ 克隆 L4 <svg>（含 width/height 属性，无 viewBox）
    │   ├─ ✅ 克隆 L5 内容组 <g transform="translate(0,0)">
    │   ├─ ✅ 克隆所有节点 <g transform="translate(x,y)">
    │   ├─ ✅ 克隆所有 foreignObject 及其 XHTML 内容
    │   └─ ❌ 不克隆任何祖先（L1/L2/Space/.space-inner 全在边界外）
    │
    ├─ 2. 内联计算样式
    │   └─ 对克隆树每个节点调用 getComputedStyle()，写入 style.cssText
    │       注意：.space-inner 的 transform 不在克隆树内，不会被内联
    │
    ├─ 3. 测量导出尺寸
    │   └─ L3.clientWidth × L3.clientHeight = paneWidth × paneHeight
    │       （Space transform 不影响 clientWidth）
    │
    ├─ 4. 构建 SVG 外壳
    │   └─ <svg viewBox="0 0 W H"> + <foreignObject width="100%" height="100%">
    │
    └─ 5. 序列化 → data URL
    ↓
最终导出结果：
  - 尺寸：paneWidth × paneHeight（原始尺寸，与用户缩放无关）
  - 内容：1:1 像素映射，节点按 ELK 布局精确定位
  - 不含 Space 的任何缩放或偏移
```

---

## 九、边界总结

### 9.1 导出边界一句话总结

**导出目标是 Reaflow `<Canvas>` 渲染的 div（`class="jsoncrack-canvas"`），Space 组件在它的外部（祖先层级），所以 Space 的 CSS transform 完全不会被克隆到导出结果中。**

### 9.2 三层容器的职责划分

| 容器 | 职责 | 对导出的影响 |
|------|------|-------------|
| **L1 canvasWrapper div** | 页面布局容器、背景色、网格 | 不直接影响（在边界外） |
| **L2 Space** | 界面缩放、平移交互 | **不影响导出**（在边界外，transform 不被克隆） |
| **L3 Reaflow Canvas div** | 画布尺寸容器（`paneWidth × paneHeight`） | **决定导出尺寸**（边界起点，clientWidth = paneWidth） |
| **L4 Reaflow SVG** | 图形渲染（无 viewBox，1:1 映射） | **决定内容尺寸和坐标系**（内部节点定位基准） |

### 9.3 Transform 类型与导出的关系

| Transform 类型 | 来源 | 应用位置 | 进入导出？ | 作用 |
|---------------|------|---------|-----------|------|
| CSS `transform: matrix()` | react-zoomable-ui Space | `.space-inner`（L3 外部） | ❌ 否 | 仅屏幕交互缩放平移 |
| SVG `transform="translate(x,y)"` | Reaflow 内容组 | `<g>`（L5，L3 内部） | ✅ 是 | 内容组整体偏移（当前为 0,0） |
| SVG `transform="translate(x,y)"` | Reaflow 节点包装 | 每个 `<g id="...">`（L3 内部） | ✅ 是 | 单个节点/边的精确定位 |
| SVG `x` / `y` 属性 | Reaflow foreignObject | `<foreignObject>`（L3 内部） | ✅ 是 | foreignObject 内 HTML 的偏移（通常为 0,0） |
