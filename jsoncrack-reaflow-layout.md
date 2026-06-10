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

---

## 六、关键文件索引

| 文件 | 职责 |
|------|------|
| [JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx) | 主组件：布局触发、ViewPort 管理、折叠重定位、适配时机调度 |
| [canvasHelpers.ts](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts) | 缩放纯函数、四级图形测量、fit/center/focus 实现 |
| [parser.ts](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/parser.ts) | JSON AST → 图数据（nodes/edges）转换 |
| [calculateNodeSize.ts](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/utils/calculateNodeSize.ts) | 布局前的节点宽高预计算（DOM 测量 + 缓存） |
| [useGraph.ts](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/apps/www/src/features/editor/views/GraphView/stores/useGraph.ts) | 应用层状态：桥接 Toolbar/快捷键 与 组件 ref API |
| [Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx) | 工具栏按钮 + useHotkeys 快捷键绑定 |
| [GraphView/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/apps/www/src/features/editor/views/GraphView/index.tsx) | 应用层组装：direction/gestures/theme 等配置注入 |
| [CustomNode.tsx](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/CustomNode.tsx) | Reaflow Node 包装，点击/hover 行为 |
| [CustomEdge.tsx](file:///d:/fz/0601/solo-dogfeeding/code/182-jsoncrack.com/packages/jsoncrack-react/src/components/CustomEdge.tsx) | Reaflow Edge 包装，点击边跳转到目标节点 |
