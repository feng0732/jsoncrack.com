# JSONCrack Reaflow 图布局与缩放实现链路深度解析

## 一、整体架构顶层视图

整个可视化系统采用 **三层职责分离** 架构：

```
┌─────────────────────────────────────────────────────────────┐
│  应用层 (apps/www)                                           │
│  GraphView / Toolbar / useGraph (zustand store)              │
│  ── 负责：用户交互、快捷键、工具栏按钮、全局状态协调           │
├─────────────────────────────────────────────────────────────┤
│  组件层 (packages/jsoncrack-react)                           │
│  JSONCrack.tsx 主组件 / Space (react-zoomable-ui)            │
│  ── 负责：Viewport 生命周期、布局触发、缩放命令封装、折叠管理  │
├─────────────────────────────────────────────────────────────┤
│  渲染层                                                      │
│  Canvas (reaflow) + ELK 布局引擎                             │
│  CustomNode / CustomEdge / ObjectNode / TextNode             │
│  ── 负责：节点 SVG 渲染、ELK 分层布局计算、边绘制            │
└─────────────────────────────────────────────────────────────┘
```

关键依赖库：
- **`reaflow`**：基于 ELK 的 React 图库，提供 `<Canvas>` 组件封装自动布局
- **`react-zoomable-ui`**：独立的平移/缩放视口系统，通过 `<Space>` 创建 `ViewPort`
- **`jsonc-parser`**：容错 JSON 解析器，生成 AST 树遍历构图
- **`zustand`**：应用层全局状态（`useGraph` store）

---

## 二、布局计算完整链路

### 阶段 1：JSON 规范化与图数据构建

**入口文件**：[JSONCrackComponent.tsx#L168-L205](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L168-L205)

#### Step 1.1：JSON → 字符串归一化

`toJsonText()` 在 [canvasHelpers.ts#L13-L26](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L13-L26) 中定义：

- 字符串输入：直接返回
- 对象输入：`JSON.stringify(json, null, 2)` + **WeakMap 按引用缓存**，避免同一对象反复序列化

```ts
const objectJsonCache = new WeakMap<object, string>();
```

#### Step 1.2：字符串 → 图结构（nodes + edges）

`parseJsonGraph()` → `parseGraph()` 在 [parser.ts#L9-L220](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/parser.ts#L9-L220) 中实现：

1. **AST 遍历**：使用 `jsonc-parser` 的 `parseTree()` 生成容错 AST（即使语法错误也尽量解析）
2. **递归 traverse()**：
   - 为每个对象/数组/基本值节点分配自增 ID（`nodeId++`）
   - **节点尺寸预计算**：调用 `calculateNodeSize(text)` 在渲染前就确定宽高（后续 ELK 布局需要）
   - **边生成**：从父节点 `property → value` 关系、数组 `parent → item` 关系生成 `EdgeData`

#### Step 1.3：节点尺寸预计算（渲染前就定好）

**为什么重要**：ELK 布局算法必须提前知道每个节点的精确 `width/height`，否则布局结果会与实际渲染不符。

在 [calculateNodeSize.ts#L67-L83](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/utils/calculateNodeSize.ts#L67-L83) 中：

```
策略：DOM 测量优先 → 估算降级
1. 创建 visibility:hidden 的 <div> 挂载到 body
2. 写入文本内容，设置 12px monospace 字体
3. getBoundingClientRect() 精确测量
4. 无 DOM 环境（SSR）时用 fallbackSize：最长行 × 8px + 24px
5. 结果存入 Map 缓存（key = 文本内容 + isParent），TTL 120 秒
```

节点输出的典型结构（[types.ts#L11-L31](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/types.ts#L11-L31)）：

```ts
interface NodeData {
  id: "1",                              // 自增ID字符串
  text: [{ key: "name", value: "Alice", type: "string" }, ...],
  width: 180,                           // ← 预计算
  height: 90,                           // ← 预计算
  path: ["user", 0, "name"],            // JSONPath，用于折叠
  parentKey: "user",
  parentType: "array"
}
```

### 阶段 2：折叠路径过滤（可选）

在 [JSONCrackComponent.tsx#L269-L282](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L269-L282)：

```
collapsedPaths (序列化的 JSONPath 数组)
        ↓ 解析为 JSONPath[]
collapsedPrefixes
        ↓ 遍历所有 nodes
        ↓   isNodeHidden(prefixes, node.path) = true?
        ↓     是 → 加入 hiddenIds Set
visibleNodes / visibleEdges（过滤掉被隐藏节点的连接边）
```

**折叠状态支持两种模式**：
- **受控**：父组件传入 `collapsedPaths` + `onToggleCollapse`
- **非受控**：组件内部 `useState<string[]>` 自管理

### 阶段 3：Reaflow Canvas + ELK 自动布局

#### Step 3.1：Canvas 组件配置

在 [JSONCrackComponent.tsx#L585-L611](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L585-L611)：

```tsx
<Canvas
  nodes={visibleNodes}       // 已折叠过滤
  edges={visibleEdges}
  direction={layoutDirection}  // "RIGHT" | "LEFT" | "DOWN" | "UP"
  layoutOptions={layoutOptions} // ← ELK 细粒度配置
  maxHeight={paneHeight}       // ← 阶段4回写的值
  maxWidth={paneWidth}
  height={paneHeight}
  width={paneWidth}
  pannable={false}             // ⚠️ 关键：禁用 Reaflow 内置平移
  zoomable={false}             // ⚠️ 关键：禁用 Reaflow 内置缩放
  animated={false}             // 禁用 framer-motion 动画
  defaultPosition={null}       // ⚠️ 关键：禁用内容自动居中
  onLayoutChange={onLayoutChange}  // ← 布局完成回调
/>
```

**关键设计决策**：**平移/缩放完全外包给 `react-zoomable-ui` 的 Space/ViewPort 系统**，Reaflow 只负责 SVG 内容的"纯布局渲染"。这避免了两套平移系统互相干扰。

#### Step 3.2：ELK 布局选项

在 [JSONCrackComponent.tsx#L40-L44](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L40-L44)：

```ts
const layoutOptions = {
  "elk.layered.compaction.postCompaction.strategy": "EDGE_LENGTH",
  "elk.layered.nodePlacement.strategy": "NETWORK_SIMPLEX",
  "elk.spacing.edgeLabel": "15",
};
```

- **`NETWORK_SIMPLEX` 节点放置**：基于网络单纯形的长路径最小化算法，适合层次数据（JSON 就是典型树结构）
- **`EDGE_LENGTH` 后压缩策略**：压缩时以边长为约束，保持视觉平衡

#### Step 3.3：布局结果回写 → 画布尺寸确定

`onLayoutChange` 回调在 [JSONCrackComponent.tsx#L401-L411](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L401-L411)：

```ts
const onLayoutChange = useCallback((layout: ElkRoot) => {
  if (!layout.width || !layout.height) {
    setLoading(false);
    return;
  }
  layoutSizeRef.current = { width: layout.width, height: layout.height };
  setPaneWidth(layout.width + 50);   // 四边各留25px边距
  setPaneHeight(layout.height + 50);
  setLoading(false);
}, []);
```

**形成闭环**：
```
初始 paneWidth=2000, paneHeight=2000
    ↓
Canvas 用此尺寸创建 SVG + 执行 ELK
    ↓
ELK 输出实际需要的 bounding box
    ↓
onLayoutChange 回写真实尺寸
    ↓
Canvas 以真实尺寸重新渲染
```

### 阶段 4：节点/边渲染

- **CustomNode**：[CustomNode.tsx#L12-L47](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/CustomNode.tsx#L12-L47) 包装 Reaflow 的 `<Node>`，区分根节点（`TextNode`）和对象节点（`ObjectNode`），支持 hover 描边高亮
- **CustomEdge**：[CustomEdge.tsx#L26-L65](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/CustomEdge.tsx#L26-L65) 包装 Reaflow 的 `<Edge>`，**点击边会自动缩放到目标节点**

---

## 三、缩放交互机制

### 3.1 三层视图：Space → ViewPort → Camera

`react-zoomable-ui` 库的核心抽象：

```
<Space> 组件
  ├── onCreate(viewPort) 回调暴露 ViewPort 对象
  │     ├── viewPort.camera          ← 核心控制对象
  │     │     ├── recenter(x, y, zoom)       // 设置绝对缩放+中心
  │     │     ├── moveByInClientSpace(dx, dy) // 平移
  │     │     ├── centerFitAreaIntoView(rect) // 适配矩形区域
  │     │     └── centerFitElementIntoView(el)
  │     ├── viewPort.zoomFactor       // 当前缩放比
  │     ├── viewPort.centerX/Y        // 视口中心（虚拟坐标）
  │     └── viewPort.translateClientRectToVirtualSpace() // 坐标转换
  └── 内部处理滚轮/触摸手势 → 驱动 camera
```

### 3.2 ViewPort 生命周期

在 [JSONCrackComponent.tsx#L570-L577](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L570-L577)：

```tsx
<Space
  onCreate={nextViewPort => {
    setViewPort(nextViewPort);                     // 存入 state
    onViewportCreateRef.current?.(nextViewPort);    // 通知应用层
  }}
  treatTwoFingerTrackPadGesturesLikeTouch={trackpadZoom}  // 用户可配置的触控板手势
/>
```

**容器尺寸同步**：`react-zoomable-ui` 只在创建时快照容器尺寸，需要手动监听 resize。

[JSONCrackComponent.tsx#L208-L220](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L208-L220)：
```ts
useEffect(() => {
  const observer = new ResizeObserver(() => {
    viewPort.updateContainerSize();  // 手动刷新容器尺寸
  });
  observer.observe(container);
}, [viewPort]);
```

### 3.3 缩放 API 的三层封装

```
┌────────────────────────────────────────────────────────────┐
│ L1: 底层 Camera 方法                                        │
│ canvasHelpers.ts 中的纯函数                                  │
│   setViewPortZoom(vp, 0.5)       → camera.recenter(,,0.5)  │
│   adjustViewPortZoom(vp, +0.1)    → camera.recenter(,,+0.1)│
│   fitGraphToViewPort(...)         → centerFitAreaIntoView()│
│   focusRootNode(...)              → centerFitElement..()   │
├────────────────────────────────────────────────────────────┤
│ L2: 组件级 viewPortApi                                       │
│ JSONCrackComponent.tsx#L222-L231 用 useMemo 包装            │
│   zoomIn / zoomOut / setZoom / centerView / focusFirstNode  │
│   通过 useImperativeHandle 暴露为 ref API（JSONCrackRef）    │
├────────────────────────────────────────────────────────────┤
│ L3: 应用层 useGraph store                                    │
│ apps/www/.../stores/useGraph.ts#L39-L71                     │
│   从 store 获取 jsonCrackRef.current → 调用 L2 API          │
│   Toolbar 按钮 / useHotkeys 快捷键 → dispatch → L3 调用    │
└────────────────────────────────────────────────────────────┘
```

**具体示例：缩放按钮的完整调用链**：

Toolbar [Toolbar/index.tsx#L198-L210](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L198-L210) 用户点击 `+` 按钮
→ `useGraph.getState().zoomIn()` [useGraph.ts#L49-L51](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/apps/www/src/features/editor/views/GraphView/stores/useGraph.ts#L49-L51)
→ `jsonCrackRef.current?.zoomIn()`（L2 imperative handle）
→ `adjustViewPortZoom(viewPort, +0.1)` [canvasHelpers.ts#L257-L260](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L257-L260)
→ `viewPort.camera?.recenter(centerX, centerY, zoomFactor + 0.1)`（L1）

---

## 三、附加：拖拽平移与指针事件隔离机制

这是整个交互系统中最容易被忽视但至关重要的部分：**如何在同一画布上同时支持"单击选中/折叠"、"拖拽平移"、"双指缩放"三种指针操作而不互相干扰**。

### 证据边界说明

本节分析严格区分三类证据：

| 标记 | 含义 | 验证方式 |
|------|------|---------|
| ✅ **仓库源码** | 本仓库内可直接验证的代码事实 | `Read` 对应文件即可确认 |
| 🌐 **外部依赖源码** | 通过 unpkg 获取的 npm 包发布版本源码 | 访问对应 URL 可验证 |
| ⚠️ **推断结论** | 基于源码合理推断、但未在运行时验证的行为 | 需要实际运行测试才能 100% 确认 |

理解本节的前提：浏览器中 **pointer 事件和 mouse 事件是两套独立的事件流**。`pointerdown` ≠ `mousedown`，它们各自独立冒泡，互不影响。对其中一种事件调用 `stopPropagation()` 不会影响另一种事件的传播。这是本节分析的关键事实基础（✅ 仓库源码 + 🌐 外部依赖源码 双重确认）。

---

### 3.4 两套事件系统在同一画布上的协作

画布上存在两套独立的"按下→拖拽→松开"检测系统，它们监听的是**不同类型的事件**：

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 系统 A：Space 组件的拖拽平移（🌐 react-zoomable-ui@0.11.0 ViewPort.js）   │
│   ├── hammerjs 处理：pan/pinch 手势                                       │
│   │   ├── pan threshold 显式设为 0（L408，不是默认 10px）                 │
│   │   └── 输入类型自动检测：PointerEvent > Touch > Mouse（🌐 hammerjs）     │
│   ├── 自行监听：mousedown/mousemove/mouseup（L379-L381）                  │
│   │           + wheel（L394）+ touch 事件（L383-L385）                    │
│   └── 光标样式：grab / grabbing (✅ JSONCrackStyles.module.css L26-L32)    │
│                                                                        │
│ 系统 B：useLongPress 长按检测（🌐 use-long-press@3.3.0 默认 detect: pointer）│
│   ├── 监听 pointerdown → pointermove → pointerup → pointerleave         │
│   ├── 按住 ≥150ms 不松开 → 回调 setCanvasDragging(true)（✅ L529-L532）   │
│   └── 松开时 → onFinish → setCanvasDragging(false)（✅ L529-L532）        │
│                                                                        │
│ ⚠️ 两套系统监听的事件类型不同，互不干扰                                    │
│ ⚠️ 系统 B 的真正目的不是"接管拖拽"，而是"抑制拖拽中的交互副作用"           │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 系统 B 的详细实现（全部 ✅ 仓库源码）

**长按绑定**：在 [JSONCrackComponent.tsx#L529-L532](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L529-L532)：

```tsx
const bindLongPress = useLongPress(
  () => setCanvasDragging(containerRef.current, true),
  {
    threshold: 150,
    onFinish: () => setCanvasDragging(containerRef.current, false),
  }
);
```

**事件绑定位置**：[JSONCrackComponent.tsx#L544](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L544)，`{...bindLongPress()}` 展开后实际注册的是 `onPointerDown / onPointerMove / onPointerUp / onPointerLeave`，绑定到 `containerRef` div（🌐 use-long-press@3.3.0 源码确认：默认 `detect: "pointer"`，监听 `pointerdown/pointermove/pointerup/pointerleave/pointerout`）。

**setCanvasDragging 实现**：在 [canvasHelpers.ts#L102-L107](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L102-L107)：

```ts
export const setCanvasDragging = (container: HTMLElement | null, dragging: boolean): void => {
  const canvas = container?.querySelector(".jsoncrack-canvas") as HTMLElement | null;
  if (!canvas) return;
  canvas.classList.toggle("dragging", dragging);
};
```

`.dragging` 类的 CSS 效果（✅ [JSONCrackStyles.module.css#L34-L37](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackStyles.module.css#L34-L37)）：

```css
.canvasWrapper :global(.dragging),
.canvasWrapper :global(.dragging *) {
  pointer-events: none !important;
}
```

**系统 B 的真正目的**：`.dragging` 类的作用 **不是** "让事件穿透到 Space 以启用拖拽"——Space 本身已经能检测拖拽（🌐 react-zoomable-ui@0.11.0 ViewPort.js 确认：hammerjs + 自行监听都在工作）。它的真正目的是**消除拖拽过程中的交互副作用**：
1. **抑制 hover 闪烁**：拖拽时指针经过节点，节点的 `onEnter/onLeave` 会反复改变描边色，视觉上产生蓝色闪烁。`.dragging` 使所有 canvas 子元素 `pointer-events: none`，hover 不再触发
2. **统一光标**：折叠按钮有 `cursor: pointer`，拖拽经过按钮时光标会从 `grabbing` 跳到 `pointer`。`.dragging` 使按钮不可见给指针，光标保持 `grabbing`
3. **防止误触**：拖拽结束时指针恰好在链接上，松手会打开 URL。`.dragging` 在按住期间使链接不可点击

---

### 3.5 三类指针区域的事件路径分述

画布上的指针区域按事件行为可分为三类，它们的 pointer-events 配置和事件冒泡路径截然不同。

#### 区域 1：节点 SVG rect + foreignObject 文本区（无交互子元素处）

```
DOM 结构（✅ 仓库源码）：
  <g> (Node wrapper)
  ├── <rect> (SVG 矩形, onClick/onEnter/onLeave)
  └── <foreignObject> (pointer-events: none)
       └── HTML 文本内容 (继承 pointer-events: none)
```

**pointerdown 事件路径**：
```
rect → g(node wrapper) → g(contentGroup) → svg → div(.jsoncrack-canvas) → div(.jsoncrack-space) → div(containerRef)
                                                                                                  ↑
                                                                                    useLongPress 的 onPointerDown
```

**mousedown 事件路径**：
```
同上路径，Space 的监听器在 .jsoncrack-space 上接收
```

**交互行为**：

| 用户操作 | 结果 | 证据类型 |
|---------|------|---------|
| 快速点击（<150ms） | Node 的 `onClick` 触发 → 打开节点详情 | ✅ 仓库源码 |
| 立即移动 | Space 识别 pan → 画布平移；Node 的 `onClick` 不触发（移动超过 click 判定阈值） | 🌐 hammerjs pan threshold = 0 + ⚠️ 浏览器 click 判定 |
| 按住 ≥150ms 后移动 | useLongPress 触发 → `.dragging` 类 → 抑制 hover/光标跳变 → Space 继续处理平移 | ✅ 仓库源码 + 🌐 use-long-press |
| 按住 ≥150ms 后松开 | useLongPress 触发 → `.dragging` 类 → 松开时 `onFinish` 移除类 → Node 的 `onClick` 触发情况取决于浏览器 | ⚠️ 推断（pointer-events: none 对 click 的影响） |

**关键点**（✅ 仓库源码）：SVG `<rect>` 没有 `onMouseDown stopPropagation`，也没有 `onPointerDown stopPropagation`，所以 pointer 和 mouse 事件都会正常冒泡到 Space 和 containerRef。这是最常见的交互区域，行为最简单。

#### 区域 2：折叠按钮

```
DOM 结构（✅ 仓库源码）：
  <g> (Node wrapper)
  ├── <rect> (SVG 矩形)
  └── <foreignObject> (pointer-events: none)
       └── <span> (pointer-events: none, .row)
            └── <span.collapseButton> (pointer-events: all ← 覆盖父级 none)
                 ├── onClick={handleToggle} (stopPropagation + preventDefault)
                 └── onMouseDown={e => e.stopPropagation()}  ← ⚠️ 只阻断 mousedown
```

**pointerdown 事件路径**（useLongPress 检测用）：
```
span.collapseButton → span.row → foreignObject → g(node wrapper) → ... → div(containerRef)
                                                                               ↑
                                                                 useLongPress 的 onPointerDown ✅ 能到达
```

**mousedown 事件路径**（Space hammerjs 检测用）：
```
span.collapseButton → ❌ onMouseDown stopPropagation! → 冒泡在此中断
```

**mousedown 被 stopPropagation 阻断后的影响**（🌐 react-zoomable-ui@0.11.0 ViewPort.js）：
- Space 自己的 `handleMouseDown` 监听器收不到 mousedown → **无法从按钮区域发起鼠标拖拽**
- 但 Space/hammerjs 的 `PointerEventInput` 可能仍然能收到 pointerdown（取决于浏览器支持）
- 🌐 hammerjs 源码确认：优先使用 PointerEvent（如果浏览器支持），否则 fallback 到 Mouse/Touch

**交互行为**：

| 用户操作 | 结果 | 证据类型 |
|---------|------|---------|
| 快速点击（<150ms） | 按钮的 `onClick` 触发 → `handleToggle` → 折叠/展开；`stopPropagation` 阻止 Node 的 `onClick` | ✅ 仓库源码 |
| 立即移动（鼠标） | mousedown 被 stopPropagation 阻断 → **Space 不识别 pan** → 不平移 | ✅ 仓库源码 + 🌐 react-zoomable-ui |
| 立即移动（触摸/支持 PointerEvent） | pointerdown 未被阻断 → Space 正常识别 pan → 平移 | 🌐 hammerjs PointerEventInput |
| 按住 ≥150ms | useLongPress 通过 **pointerdown** 检测到长按 → `.dragging` 类 → 按钮变为 `pointer-events: none !important` | ✅ 仓库源码 + 🌐 use-long-press |
| 按住 ≥150ms 后松开 | `onFinish` 移除 `.dragging` → 但此时按钮的 `click` 事件可能已被 `.dragging` 屏蔽 | ⚠️ 推断 |

**折叠按钮 `onMouseDown stopPropagation` 的设计意图**（✅ 仓库源码）：防止用户在按钮上 mousedown 时被 Space 误识别为 pan 起点。这是一种**保守的事件隔离**——宁可从按钮区域无法用鼠标拖拽，也不要让点击按钮时意外触发平移。

**但 `onMouseDown stopPropagation` 不影响 `pointerdown`**（✅ 仓库源码 + 🌐 use-long-press）：因为这是两种独立事件。`useLongPress` 默认监听 pointer 事件，所以即使 mousedown 被阻断，长按检测仍然正常工作。

#### 区域 3：超链接

```
DOM 结构（✅ 仓库源码）：
  <g> (Node wrapper)
  ├── <rect> (SVG 矩形)
  └── <foreignObject> (pointer-events: none)
       └── <a.link> (pointer-events: all ← 覆盖父级 none)
            └── onClick={e => e.stopPropagation()}  ← 只阻断 click，不阻断 mousedown/pointerdown
```

**pointerdown 事件路径**：
```
a.link → foreignObject → g(node wrapper) → ... → div(containerRef)
                                                    ↑
                                      useLongPress 的 onPointerDown ✅ 能到达
```

**mousedown 事件路径**：
```
a.link → foreignObject → g(node wrapper) → ... → div(.jsoncrack-space)
                                                    ↑
                                      Space 监听器 ✅ 能到达（无 stopPropagation）
```

**交互行为**：

| 用户操作 | 结果 | 证据类型 |
|---------|------|---------|
| 快速点击（<150ms） | 链接 `onClick` 触发 → `stopPropagation` 阻止 Node 的 `onClick` → 浏览器打开 URL | ✅ 仓库源码 |
| 立即移动 | Space 识别 pan → 画布平移；链接 `onClick` 不触发（移动超过 click 判定阈值） | 🌐 hammerjs pan threshold = 0 + ⚠️ 浏览器 click 判定 |
| 按住 ≥150ms | useLongPress 触发 → `.dragging` 类 → 链接变为 `pointer-events: none` → **防止拖拽结束时误触链接** | ✅ 仓库源码 |
| 按住 ≥150ms 后松开 | `onFinish` 移除 `.dragging` → 链接恢复正常可点击状态 | ✅ 仓库源码 |

**关键区别**（✅ 仓库源码）：链接只阻断 `click` 冒泡，不阻断 `mousedown/pointerdown`。这意味着从链接区域可以正常发起拖拽平移，同时 `onClick stopPropagation` 防止点击链接时触发 Node 的 `onClick`。

---

### 3.6 pointer-events 层级配置总览

整个系统通过 **CSS pointer-events 的分层控制**，实现"默认可交互、拖拽时全屏蔽、特定元素穿透"的精确行为。

#### 层级结构

```
containerRef (最外层 div)
│  绑定 useLongPress() → onPointerDown/Move/Up/Leave
│
├── <Space> (react-zoomable-ui)
│   │  className="jsoncrack-space"
│   │  cursor: grab / grabbing (✅ JSONCrackStyles L26-L32)
│   │  hammerjs pan(priority: PointerEvent) + 自行监听 wheel/mouse/touch
│   │
│   └── <Canvas> (reaflow) ← .dragging 类加在此元素上
│        className="jsoncrack-canvas"
│        pannable={false}, zoomable={false}
│        │
│        └── <svg>
│             └── <g> (contentGroup)
│                  │
│                  ├── <g> (每个 Node)
│                  │    ├── <rect> (SVG 矩形)
│                  │    │    onClick / onEnter / onLeave (✅ CustomNode L26-L31)
│                  │    │    无 mousedown/pointerdown stopPropagation
│                  │    │
│                  │    └── <foreignObject>
│                  │         pointer-events: none  ← ⭐ L1：屏蔽 HTML 内容
│                  │         │
│                  │         ├── .row (文本行，继承 none)
│                  │         │
│                  │         ├── .collapseButton
│                  │         │    pointer-events: all  ← ⭐ L2：按钮单独启用
│                  │         │    onMouseDown stopPropagation  ← 只阻断 mousedown
│                  │         │    onClick stopPropagation + preventDefault
│                  │         │
│                  │         └── a.link
│                  │              pointer-events: all  ← ⭐ L3：链接单独启用
│                  │              onClick stopPropagation  ← 只阻断 click
│                  │
│                  └── <path> (每个 Edge)
│                       onClick / onEnter / onLeave
│
├── .overlay (加载遮罩)
│    pointer-events: all  ← ⭐ L4：加载时屏蔽底层交互
│    z-index: 30
│
└── .tooLarge (超量提示)
     z-index: 40
```

**`.dragging` 激活时的覆盖规则**（✅ [JSONCrackStyles.module.css#L34-L37](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackStyles.module.css#L34-L37)）：

```css
.canvasWrapper :global(.dragging),
.canvasWrapper :global(.dragging *) {
  pointer-events: none !important;  /* ⭐ L5：拖拽时全局强制屏蔽，覆盖所有子元素 */
}
```

`!important` 意味着它会覆盖 L2/L3 中 `.collapseButton` 和 `.link` 的 `pointer-events: all`。这是关键——拖拽时连按钮和链接都不可交互。

#### 各层职责

| 层级 | 配置位置 | 值 | 作用 | 证据 |
|------|---------|----|------|------|
| **L1** | `foreignObject` | `pointer-events: none` | 让 SVG `<rect>` 接收点击/hover，而非内部 HTML | ✅ Node.module.css L10 |
| **L2** | `.collapseButton` | `pointer-events: all` | 按钮穿透 foreignObject 屏蔽，可被点击 | ✅ Node.module.css L58 |
| **L3** | `a.link` | `pointer-events: all` | 链接穿透 foreignObject 屏蔽，可被点击 | ✅ TextRenderer.module.css L19 |
| **L4** | `.overlay` | `pointer-events: all` | 加载时拦截所有底层交互 | ✅ JSONCrackStyles.module.css L56 |
| **L5** | `.dragging, .dragging *` | `pointer-events: none !important` | 长按拖拽时，覆盖 L1-L3，使所有 canvas 子元素不可交互 | ✅ JSONCrackStyles.module.css L34-L37 |

#### 为什么 foreignObject 默认 pointer-events: none？

Reaflow 的 `<Node>` 是 SVG `<g>` + `<rect>` 结构，节点的 `onClick`/`onEnter`/`onLeave` 绑定在 SVG 元素上。如果 foreignObject 内的 HTML 元素接收了 pointer events，会导致：
1. 点击节点文字时，SVG `<rect>` 的 `onClick` 不会触发（被 HTML 元素吞了）
2. hover 效果时有时无（取决于指针精确落在文字上还是空白上）

通过 `foreignObject { pointer-events: none }`，让所有指针事件直接穿透到 SVG 层，保证节点交互行为一致。需要交互的子元素（折叠按钮、超链接）再单独 `pointer-events: all` 启用。（✅ 仓库源码确认配置，⚠️ 行为推断）

---

### 3.7 事件传播控制：stopPropagation 的三处使用

除了 pointer-events 的静态配置，代码中有三处 `stopPropagation` 实现了精确的事件隔离：

#### 3.7.1 折叠按钮 — 双重 stopPropagation

在 [ObjectNode.tsx#L49-L54, L76](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/ObjectNode.tsx#L49-L54)（✅ 仓库源码）：

```tsx
const handleToggle = (event: React.MouseEvent) => {
  event.stopPropagation();   // 阻止 click 冒泡到 Node
  event.preventDefault();
  onToggleCollapse(rowPath);
};

<span
  className={styles.collapseButton}
  onClick={handleToggle}
  onMouseDown={event => event.stopPropagation()}  // 阻止 mousedown 冒泡
>
```

| stopPropagation 位置 | 阻断的事件 | 目的 | 证据 |
|----------------------|-----------|------|------|
| `onMouseDown` | `mousedown` | 防止 Space 的 `handleMouseDown` 从按钮区域识别 pan 手势起点 | ✅ 仓库源码（ObjectNode.tsx L76） + 🌐 react-zoomable-ui ViewPort.js L379-L381 |
| `onClick` | `click` | 防止触发 Node 外层的 `onNodeClick`（打开详情模态框） | ✅ 仓库源码（ObjectNode.tsx L49-L54） |

**注意**：`onMouseDown stopPropagation` 只阻断 `mousedown`，**不阻断 `pointerdown`**。因此 `useLongPress`（监听 pointer 事件）仍然能检测到在按钮上的长按（🌐 use-long-press@3.3.0 确认默认 detect: "pointer"）。

#### 3.7.2 超链接 — 只阻断 click

在 [TextRenderer.tsx#L28-L30](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/TextRenderer.tsx#L28-L30)（✅ 仓库源码）：

```tsx
<a
  className={styles.link}
  onClick={event => event.stopPropagation()}  // 只阻断 click
  href={href} target="_blank" rel="noopener noreferrer"
>
```

只阻断 `click` 冒泡，不阻断 `mousedown/pointerdown`。这意味着：
- 点击链接不会触发 Node 的 `onClick`（被 stopPropagation 阻断）
- 从链接区域可以正常发起拖拽平移（mousedown/pointerdown 正常冒泡到 Space）
- 浏览器默认行为（打开 URL）不受影响（未调用 `preventDefault`）

#### 3.7.3 右键菜单 — 全局禁用

- [JSONCrackComponent.tsx#L543](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L543)：`onContextMenu={event => event.preventDefault()}`
- [JSONCrackComponent.tsx#L575](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L575)：Space 层同样阻止

---

### 3.8 Space 内部阈值与步长证据汇总

在深入分析冲突化解之前，先集中展示 `<Space>` 内部所有关键阈值和步长的完整证据链，严格区分三类证据：

| 参数 | 值 | 证据类型 | 源码位置 |
|-----|----|---------|---------|
| **pan 拖拽阈值** | **0**（不是 hammerjs 默认 10px） | 🌐 外部依赖源码 | react-zoomable-ui@0.11.0 ViewPort.js L408 |
| **滚轮缩放步长** | 动态 `dZoom = ((-1 * dy) / containerHeight) * zoomFactor` | 🌐 外部依赖源码 | react-zoomable-ui@0.11.0 ViewPort.js L341-L342 |
| **工具栏按钮缩放步长** | 固定 ±0.1 | ✅ 仓库源码 | canvasHelpers.ts L257-L260 |
| **hammerjs 输入优先级** | PointerEvent > Touch > Mouse | 🌐 外部依赖源码 | hammerjs@2.0.8 输入类型检测逻辑 |
| **useLongPress 事件类型** | 默认 `detect: "pointer"` | 🌐 外部依赖源码 | use-long-press@3.3.0 源码事件数组 |
| **useLongPress 时间阈值** | 150ms（可配置） | ✅ 仓库源码 | JSONCrackComponent.tsx L529-L532 |

---

### 3.8.1 Space 手势识别分层

`react-zoomable-ui` 的 `<Space>` 组件手势识别分为两部分（🌐 react-zoomable-ui@0.11.0 ViewPort.js）：
1. **hammerjs**：用于 pan 和 pinch 手势，pan threshold 显式设置为 **0**（不是默认的 10px）
2. **自行监听**：wheel 滚轮缩放、mousedown/mousemove/mouseup 处理右键平移、touchstart/touchend 处理触摸点击

#### 手势类型

```
<Space> 手势识别层（🌐 react-zoomable-ui@0.11.0 ViewPort.js）
├── hammerjs pan → camera.moveByInClientSpace()     （平移，🌐 L408 threshold = 0）
├── hammerjs pinch → camera.recenter(,, newZoom)     （双指缩放，🌐 L407）
├── 自行监听 wheel → handleWheel() 计算 dZoom       （滚轮缩放，🌐 L341-L342 动态步长）
├── 自行监听 mousedown/up → 处理点击/右键平移        （鼠标，🌐 L379-L381）
└── 自行监听 touchstart/end → 处理触摸点击           （触摸，🌐 L383-L385）
```

**滚轮缩放步长**（🌐 react-zoomable-ui@0.11.0 ViewPort.js L317-L349 `handleWheel`）：
```js
// L341-L342
const dy = e.deltaY * scale;
const dZoom = ((-1 * dy) / this.containerHeight) * this.zoomFactor;
```
🌐 外部依赖源码确认：不是固定的 ±0.1，而是与滚轮滚动距离、容器高度、当前缩放比相关的动态值。scale 系数根据 deltaMode 调整（L330-L336）。

**工具栏按钮缩放步长**（✅ [canvasHelpers.ts#L257-L260](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L257-L260)）：
```ts
// ✅ 仓库源码确认：delta 固定为 0.1
viewPort.camera?.recenter(viewPort.centerX, viewPort.centerY, viewPort.zoomFactor + delta);
```
✅ 仓库源码确认：delta 固定为 0.1。这是工具栏按钮调用 `adjustViewPortZoom(viewPort, +0.1)` 时传入的固定值，与滚轮缩放的动态步长完全不同。

#### 触控板手势可配置性

`trackpadZoom` prop 在 [JSONCrackComponent.tsx#L576](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L576)（✅ 仓库源码）：

```tsx
<Space treatTwoFingerTrackPadGesturesLikeTouch={trackpadZoom} />
```

用户可在工具栏偏好设置中切换（见 [Toolbar/index.tsx#L307-L314](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L307-L314)）。

#### 按区域分析冲突化解策略

**核心矛盾**：pointerdown 时，系统不知道用户是想"单击"还是"拖拽"。

**区域 1（节点 rect + 文本区）的化解**——最简单，所有事件都正常冒泡：

```
用户按下指针 → pointerdown + mousedown 都冒泡到 Space 和 containerRef
      │
      ├─→ 快速松开 (<150ms)
      │    └── Node onClick 触发 → 打开节点详情
      │
      ├─→ 立即移动（任何距离，threshold = 0）
      │    └── Space 识别 pan → 画布平移
      │        Node onClick 不触发（移动超过浏览器 click 判定阈值）
      │
      └─→ 按住不动 ≥150ms
           ├── useLongPress 触发 → .dragging 类
           ├── 后续 hover/光标跳变被抑制（pointer-events: none）
           └── Space 继续处理后续移动 → 平移画布
```

**区域 2（折叠按钮）的化解**——最保守，mousedown 被阻断：

```
用户按下指针 → pointerdown 冒泡到 containerRef (useLongPress ✅)
             → mousedown 被 stopPropagation ❌ 不冒泡到 Space
      │
      ├─→ 快速松开 (<150ms)
      │    └── 按钮 onClick → handleToggle → 折叠/展开
      │        Node onClick 不触发（被 stopPropagation 阻断）
      │
      ├─→ 立即移动（鼠标）
      │    └── mousedown 未到达 Space → 不识别 pan → 不平移
      │
      ├─→ 立即移动（触摸/PointerEvent）
      │    └── pointerdown 到达 Space → 正常识别 pan → 平移
      │
      └─→ 按住不动 ≥150ms
           ├── useLongPress 通过 pointerdown 检测到 → .dragging 类
           ├── 按钮变为 pointer-events: none → 不再响应 hover/click
           └── 后续行为取决于 Space 能否通过 pointerdown 识别 pan
```

**区域 3（超链接）的化解**——最开放，只阻断 click：

```
用户按下指针 → pointerdown + mousedown 都冒泡到 Space 和 containerRef
      │
      ├─→ 快速松开 (<150ms)
      │    └── 链接 onClick → stopPropagation（阻止 Node onClick）+ 浏览器打开 URL
      │
      ├─→ 立即移动（任何距离，threshold = 0）
      │    └── Space 识别 pan → 画布平移
      │        链接 onClick 不触发（移动超过浏览器 click 判定阈值）
      │
      └─→ 按住不动 ≥150ms
           ├── useLongPress 触发 → .dragging 类
           ├── 链接变为 pointer-events: none → 拖拽中不会误触打开 URL
           └── Space 继续处理平移
```

---

### 3.9 状态机视角：交互模式切换

整个交互系统可以看作一个状态机，但不同区域的转移条件不同：

```
                    区域 1 (节点rect/文本)        区域 2 (折叠按钮)           区域 3 (超链接)
                    ─────────────────────        ─────────────────           ─────────────────
  mousedown 到达    ✅ 是                         ❌ 否 (stopPropagation)    ✅ 是
  pointerdown 到达  ✅ 是                         ✅ 是                      ✅ 是
  Space 可 pan      ✅ 是（鼠标+触摸）             ⚠️ 仅触摸/PointerEvent      ✅ 是
  useLongPress      ✅ 是                         ✅ 是 (通过 pointerdown)   ✅ 是

                    ┌─────────────────────────────────────────────────────────────┐
                    │                                                             │
                    │   IDLE (空闲)                                               │
                    │   - 可点击、可折叠、可跳转链接                                │
                    │   - cursor: grab                                             │
                    │                                                             │
                    └──────┬──────────────────────┬──────────────────────┬────────┘
                           │                      │                      │
              区域1/3:     │         所有区域:      │         区域1/3:     │
              pointerdown  │         pointerdown    │         pointerdown  │
              + 移动 > 0   │         + 静止 ≥150ms  │         + 快速松开   │
              (threshold=0)│                      │                      │
                           ▼                      ▼                      ▼
                    ┌──────────────┐    ┌──────────────────┐      ┌──────────────┐
                    │ DRAGGING     │    │ LONG_PRESS       │      │ CLICK        │
                    │ (平移中)     │    │ (长按抑制中)      │      │ (单击触发)   │
                    │ - Space pan  │    │ - .dragging 类   │      │ - 区域1:     │
                    │ - 无额外抑制 │    │ - pointer-events │      │   Node详情   │
                    │              │    │   全部 none      │      │ - 区域2:     │
                    │              │    │ - hover/光标/误触│      │   折叠/展开  │
                    │              │    │   全部被抑制     │      │ - 区域3:     │
                    └──────────────┘    └──────────────────┘      │   打开URL    │
                           │                      │               └──────────────┘
                           │                      │                      │
                           └──── pointerup ───────┴──────────────────────┘
                                        ↓
                                    回到 IDLE
                                    (onFinish 移除 .dragging)
```

**DRAGGING 和 LONG_PRESS 的区别**：
- **DRAGGING**：用户开始移动后，Space 立即识别 pan（threshold = 0），此时 `.dragging` 类**可能还没加**（因为还没到 150ms），hover 副作用仍然存在
- **LONG_PRESS**：用户按住不动 150ms 后，`.dragging` 类被加上，后续任何移动都是在"抑制状态"下进行的
- 实际上两种模式可能**重叠**：先移动触发 DRAGGING，持续按住超 150ms 又触发 LONG_PRESS，此时 `.dragging` 类补上，消除之前未抑制的 hover 副作用

---

### 已删除的不准确断言

在本次修正中，以下之前的不准确断言被删除或修正，因为通过外部依赖源码验证发现与事实不符：

| 已删除的不准确断言 | 正确结论 | 证据来源 |
|-------------------|---------|---------|
| ❌ "hammerjs 默认 ~5px 拖拽阈值" | 🌐 react-zoomable-ui@0.11.0 显式设置 `pan threshold = 0`（ViewPort.js L408），覆盖了 hammerjs 默认的 10px | 外部依赖源码 |
| ❌ "滚轮缩放 zoom±0.1" | 🌐 滚轮缩放使用动态步长 `dZoom = ((-1 * dy) / containerHeight) * zoomFactor`（ViewPort.js L341-L342）；±0.1 仅适用于工具栏按钮（✅ canvasHelpers.ts L257-L260） | 外部依赖源码 + 仓库源码 |
| ❌ "useLongPress 监听 mouse 事件" | 🌐 use-long-press@3.3.0 默认 `detect: "pointer"`，监听 pointerdown/pointermove/pointerup/pointerleave | 外部依赖源码 |
| ❌ "移动超过 click 判定阈值" | ⚠️ 浏览器 click 判定阈值未经验证，仅作推断标注，不做确定性断言 | 推断结论（待运行时验证） |
| ❌ "Space 无法从按钮区域发起拖拽" | ⚠️ 鼠标拖拽因 `onMouseDown stopPropagation` 被阻断，但触摸/PointerEvent 可能仍然可用（取决于浏览器支持） | 仓库源码 + 推断结论 |

> **证据边界说明**：所有被删除的断言都属于"无明确证据的推断"或"与外部依赖源码冲突"。修正后的结论严格按照 ✅ 仓库源码 / 🌐 外部依赖源码 / ⚠️ 推断结论 三类标记，确保读者能清楚区分哪些是已验证的事实，哪些是合理推断。

---

## 四、布局与缩放的配合链路（核心难点）

这是整个系统最精妙的部分：**布局变化后如何与缩放/视口定位无缝衔接**。

### 场景 1：初次加载 / JSON 变更 → 自动居中适配

**触发源**：`jsonText` 变化 → 解析 effect → setNodes/setEdges → Reaflow 重布局 → `onLayoutChange` → `paneWidth/paneHeight` 更新

**适配 effect**：[JSONCrackComponent.tsx#L491-L510](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L491-L510)

```
依赖数组：[viewPort, visibleNodes, loading, centerOnLayout, initialFitDone, paneWidth, paneHeight]

当所有条件满足（viewPort存在、有节点、加载完成、pane尺寸更新、未适配过）：
  ↓
requestAnimationFrame（等待一帧让 SVG 渲染完成，测量才准确）
  ↓
fitGraphToViewPort(viewPort, container, layoutSizeRef.current)
  ↓
标记 initialFitDone = true（后续 JSON 变化会重置它，见 parse effect L173）
```

### 场景 2：fitGraphToViewPort 的四级测量降级

核心函数在 [canvasHelpers.ts#L224-L238](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L224-L238)：

```
fitGraphToViewPort
  ├── 1. viewPort.updateContainerSize()  // 先确保容器是最新的
  ├── 2. computeGraphClientRect(container, layoutSize)  // 获取图形的屏幕矩形
  │     │
  │     ├── Level 1 ⭐：unionContentGroupLeafRects()
  │     │     遍历 SVG 内所有 g[id] 叶子节点（每个 node/edge 一个）
  │     │     取所有 getBoundingClientRect 的并集
  │     │     解决：framer-motion transform 导致外层 g 测量不准、
  │     │           曲线边 bulge 超出直线端点包围盒的问题
  │     │
  │     ├── Level 2：contentGroup.getBoundingClientRect()
  │     │     外层 g 直接测量，大部分形状准确
  │     │
  │     ├── Level 3：getBBox() + getScreenCTM() 投影
  │     │     SVG 原生几何边界 + 屏幕坐标变换矩阵
  │     │     即使 visibility:hidden 也能工作
  │     │
  │     └── Level 4：ELK 返回的 layoutSize + svg.getScreenCTM()
  │           布局完成但渲染未开始时的最后兜底
  │
  │     最后：四周膨胀 2%（GRAPH_FIT_PADDING_RATIO = 0.02）
  │           ↑ 用比例而非固定像素，保证不同缩放下结果一致
  │
  ├── 3. translateClientRectToVirtualSpace(rect)
  │     将"屏幕像素矩形"转换为"虚拟空间坐标矩形"（考虑当前缩放/平移）
  │
  └── 4. camera.centerFitAreaIntoView(virtualRect)
        计算合适的缩放比 + 平移量，让目标矩形居中填满视口
```

### 场景 3：折叠/展开 → 重布局后"钉住"点击按钮

**用户痛点**：点击折叠按钮后，ELK 重布局导致整个图移动，原来的按钮跑到屏幕其他地方，用户需要重新寻找。

**解决方案**：[JSONCrackComponent.tsx#L286-L483](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L286-L483) 实现了"点击位置记忆 + 等待稳定 + 补偿平移"机制。

```
点击折叠按钮时 wrappedToggleCollapse：
  ① 记录按钮当前的屏幕中心坐标 (clientX, clientY) → pendingRecenterRef
  ② setIsRelayouting(true)  // 隐藏画布（visibility:hidden）避免用户看到跳动

布局完成后 visibleNodes/visibleEdges 变化触发 effect：
  ① rAF 轮询（最多 180 帧 = 3 秒）
  ② 每帧重新找到按钮并测量屏幕坐标
  ③ 观察连续两帧：先检测到"移动过"（movementSeen），然后连续稳定
  ④ 稳定后：计算位移差 dx = 新位置.x - 原位置.x
  ⑤ viewPort.camera.moveByInClientSpace(dx, dy)  // 反向补偿平移
  ⑥ setIsRelayouting(false)  // 重新显示画布
```

**这是一个精巧的"后布局相机校正"模式**：先让 Reaflow 完成布局+渲染，再通过 react-zoomable-ui 的平移把按钮"拉回"用户点击的位置。用户感知是"折叠没有导致图乱跑"。

### 场景 4：布局方向切换 → 延迟居中

在 [useGraph.ts#L42-L45](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/apps/www/src/features/editor/views/GraphView/stores/useGraph.ts#L42-L45)：

```ts
setDirection: (direction = "RIGHT") => {
  set({ direction });
  setTimeout(() => get().centerView(), 200);  // 等待 200ms 让重布局完成
},
```

`direction` 作为 `<JSONCrack>` 的 `key` 一部分 [GraphView/index.tsx#L99](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/apps/www/src/features/editor/views/GraphView/index.tsx#L99)，会导致组件**完全卸载重建**（包括 ViewPort 重建）。因此用 setTimeout 等待 React 提交 + Reaflow 布局完成后再调 centerView。

### 场景 5：点击边 → 缩放到目标节点

在 [CustomEdge.tsx#L30-L50](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/CustomEdge.tsx#L30-L50)：

```
点击 Edge
  → edgeId 查 Map 得 targetNodeId
  → querySelector(`[data-id$="node-${targetNodeId}"]`) 找到 DOM 元素
  → camera.centerFitElementIntoView(target, { elementExtraMarginForZoom: 150 })
```

---

## 五、关键设计要点总结

### 5.1 为什么禁用 Reaflow 自带的 pannable/zoomable？

因为 Reaflow 的平移缩放是基于"调整 `<g>` 元素 transform"的，而 `react-zoomable-ui` 的 Space 是基于"调整 CSS perspective + camera 矩阵"的。两套系统混用会导致：
- 测量值（getBoundingClientRect）的坐标系混乱
- Space 的滚轮手势与 Canvas 的拖动事件冲突

本项目的选择：**让 Reaflow 只管"画什么"（布局+渲染），让 Space 只管"怎么看"（平移+缩放）**，通过 `defaultPosition={null}` 禁用 Reaflow 的内置居中，使内容精确落在 SVG 原点，配合 Space 的测量体系。

### 5.2 节点尺寸为什么在布局前"离线预计算"？

ELK 布局算法的核心输入就是节点宽高。如果：
1. 先让 Reaflow 布局（用默认尺寸）
2. 再渲染测实际尺寸
3. 再回调高宽重新布局

会导致**布局闪烁**（一次布局不对，两次布局用户看到跳动）。预计算的 `calculateNodeSize` 用 DOM 测量保证了传给 ELK 的宽高与渲染结果 1:1 匹配。

### 5.3 折叠"等待稳定"的轮询策略为什么用 rAF？

Reaflow 在大型图上：
1. ELK Web Worker 异步计算布局（~几到几十毫秒）
2. framer-motion 动画提交（即使 `animated={false}`，内部仍可能有多帧）
3. 浏览器布局/绘制流水线

直接在 setState 后读取 DOM，读到的可能是"布局未完成的中间态"。用 **180 帧 rAF 轮询 + 连续两帧稳定检测** 保证了：
- 不会提前读取导致校正失败
- 不会因动画还在进行而校正到错误位置
- 有超时兜底避免无限循环

### 5.4 `.dragging` 类的真正目的：消除拖拽副作用，而非"启用拖拽"

之前的分析曾误认为 useLongPress + `.dragging` 的作用是"让事件穿透到 Space 以启用拖拽"。实际上 **Space 本身已经能检测拖拽**（hammerjs 直接接收冒泡上来的 pointer/mouse 事件）。`.dragging` 类的三个真正目的是：
1. **抑制 hover 闪烁**：拖拽时经过节点，`onEnter/onLeave` 不再反复触发描边变色
2. **统一光标**：按钮的 `cursor: pointer` 不再覆盖 Space 的 `cursor: grabbing`
3. **防止误触**：拖拽结束松手时不会误击链接打开 URL

### 5.5 pointer 事件与 mouse 事件的独立性是理解本系统的关键

`pointerdown` 和 `mousedown` 是两套独立的事件流，各自冒泡，互不影响。代码中的 `stopPropagation` 只阻断对应类型的事件：
- 折叠按钮的 `onMouseDown stopPropagation` 只阻断 `mousedown`，不阻断 `pointerdown`
- 因此 `useLongPress`（监听 pointer 事件）仍然能检测到折叠按钮上的长按
- 但 Space/hammerjs 如果依赖 `mousedown` 来识别 pan 手势，则从按钮区域无法发起拖拽

### 5.6 三类区域的 stopPropagation 策略对比

| 区域 | 阻断 mousedown | 阻断 pointerdown | 阻断 click | 设计意图 |
|------|---------------|-----------------|------------|---------|
| 节点 rect + 文本 | ❌ | ❌ | ❌ | 完全开放，点击/拖拽/长按都正常工作 |
| 折叠按钮 | ✅ `onMouseDown stopPropagation` | ❌ | ✅ `onClick stopPropagation` | 保守隔离：确保点击不误触，但从按钮拖拽可能受限 |
| 超链接 | ❌ | ❌ | ✅ `onClick stopPropagation` | 开放策略：点击不误触 Node，但拖拽/长按正常工作 |

---

## 六、关键文件索引

| 文件 | 职责 |
|------|------|
| [JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx) | 主组件：布局触发、ViewPort 管理、折叠重定位、适配时机调度、长按平移绑定 |
| [canvasHelpers.ts](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts) | 缩放纯函数、四级图形测量、fit/center/focus、setCanvasDragging 拖拽状态切换 |
| [parser.ts](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/parser.ts) | JSON AST → 图数据（nodes/edges）转换 |
| [calculateNodeSize.ts](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/utils/calculateNodeSize.ts) | 布局前的节点宽高预计算（DOM 测量 + 缓存） |
| [useGraph.ts](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/apps/www/src/features/editor/views/GraphView/stores/useGraph.ts) | 应用层状态：桥接 Toolbar/快捷键 与 组件 ref API |
| [Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx) | 工具栏按钮 + useHotkeys 快捷键绑定 + 手势偏好设置 |
| [GraphView/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/apps/www/src/features/editor/views/GraphView/index.tsx) | 应用层组装：direction/gestures/theme 等配置注入 |
| [CustomNode.tsx](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/CustomNode.tsx) | Reaflow Node 包装，点击/hover 行为 |
| [CustomEdge.tsx](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/CustomEdge.tsx) | Reaflow Edge 包装，点击边跳转到目标节点 |
| [ObjectNode.tsx](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/ObjectNode.tsx) | 对象节点渲染、折叠按钮、stopPropagation 事件隔离 |
| [TextNode.tsx](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/TextNode.tsx) | 根节点/叶子节点渲染 |
| [TextRenderer.tsx](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/TextRenderer.tsx) | 文本渲染、URL 链接化、颜色预览、事件隔离 |
| [CollapseContext.ts](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/CollapseContext.ts) | 折叠状态 Context、路径匹配、隐藏节点过滤 |
| [JSONCrackStyles.module.css](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackStyles.module.css) | 全局样式、.dragging 类 pointer-events 屏蔽、光标样式 |
| [Node.module.css](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/Node.module.css) | 节点样式、foreignObject pointer-events 屏蔽、collapseButton 穿透启用 |
| [TextRenderer.module.css](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/TextRenderer.module.css) | 超链接 pointer-events 启用 |
