# JSONCrack 数据流向分析：从 JSON 文本到节点关系图

## 总览

整个可视化流程分为 **5 个主要阶段**：

```
JSON 文本输入
    ↓
1. 格式适配 (jsonAdapter.ts) — 多格式 → 统一 JSON
    ↓
2. 语法解析 (parser.ts)      — JSON 文本 → AST 语法树
    ↓
3. 图结构构建 (parser.ts)    — AST → NodeData[] + EdgeData[]
    ↓
4. 节点尺寸计算 (calculateNodeSize.ts) — 计算每个节点的宽高
    ↓
5. 画布布局与渲染 (JSONCrackComponent.tsx + reaflow ELK) — 自动布局 + SVG 绘制
```

---

## 阶段 1：格式适配层

**代码位置：** [jsonAdapter.ts](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/lib/utils/jsonAdapter.ts)

这是最上层的输入处理，支持将多种格式统一转换为 JSON 对象：

| 输入格式 | 转换依赖库 | 说明 |
|---------|-----------|------|
| JSON / JSONC | `jsonc-parser` | 容错解析，有语法错误时降级到原生 `JSON.parse` |
| YAML | `js-yaml` | `load()` 直接转为 JS 对象 |
| XML | `fast-xml-parser` | 支持属性前缀、布尔值解析 |
| CSV | `json-2-csv` | 支持 Excel BOM、字段裁剪 |

核心函数：
```ts
export const contentToJson = async (value: string, format: FileFormat): Promise<object>
```

---

## 阶段 2：语法解析 — JSON 文本 → AST

**代码位置：** [parser.ts#L9-L19](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/parser.ts#L9-L19)

使用第三方库 `jsonc-parser` 的 `parseTree()` 函数将 JSON 文本解析为带父子关系的 AST 节点树。

```ts
const parseErrors: ParseError[] = [];
const jsonTree = parseTree(json, parseErrors);  // 容错解析，即使有错误也尽量生成树
```

`jsonc-parser` 提供的 AST 节点类型（`Node`）：
- `type`: `"object" | "array" | "string" | "number" | "boolean" | "null" | "property"`
- `value`: 叶子节点的实际值
- `children`: 子节点数组
- `parent`: 父节点引用

**关键特性：** 这是容错解析——即使 JSON 有语法错误，也会返回部分 AST 树，同时收集错误到 `parseErrors`。

---

## 阶段 3：图结构构建 — AST → 节点 + 边

**代码位置：** [parser.ts#L21-L219](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/parser.ts#L21-L219)

这是整个流程的**核心**，通过递归遍历 AST 生成两类数据：`NodeData[]`（节点）和 `EdgeData[]`（边）。

### 数据结构

**代码位置：** [types.ts](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/types.ts)

```ts
// 节点内的一行键值对
interface NodeRow {
  key: string | null;           // 对象的属性名，数组元素为 null
  value: string | number | null | boolean;
  type: Node["type"];           // 值的类型
  childrenCount?: number;       // 子对象/子数组的元素个数
  to?: string[];                // 指向的子节点 ID 数组
}

// 一个图节点（对应一个对象或数组或标量值）
interface NodeData {
  id: string;                   // 自增 ID（"1", "2", "3"...）
  text: NodeRow[];              // 节点内展示的多行键值对
  width: number;                // 节点像素宽度（阶段4计算）
  height: number;               // 节点像素高度（阶段4计算）
  path?: JSONPath;              // 该节点在 JSON 中的路径，如 ["user", 0, "name"]
  parentKey?: string;           // 父节点的键名
  parentType?: string;          // 父节点类型 "object" | "array"
}

// 一条连接线
interface EdgeData {
  id: string;                   // 自增 ID
  from: string;                 // 起始节点 ID
  to: string;                   // 目标节点 ID
  text: string | null;          // 边上的标签（属性名）
}
```

### 核心算法：`traverse()` 递归遍历

`traverse(node, parentId)` 接收一个 AST 节点，返回该节点在图中的 ID。

#### 节点聚合规则

**不是每个 JSON 值都成为独立节点**，解析器采用了**聚合策略**：

| JSON 结构 | 生成方式 |
|----------|---------|
| 标量值（string/number/boolean/null） | 单独一个节点 |
| 扁平对象（所有值都是标量） | 聚合为**一个节点**，每个属性一行 `NodeRow` |
| 对象中有嵌套对象/数组 | 嵌套部分拆出去成为独立子节点，当前节点只保留占位行 `{N keys}` / `[N items]` |
| 数组 | 每个数组元素是独立子节点，根数组额外有一个父节点显示 `[N items]` |

#### 三种典型分支

**分支 A：根级数组** ([parser.ts#L42-L65](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/parser.ts#L42-L65))

```
输入: [1, 2, 3]
生成:
  节点1: [3 items]        ← 数组父节点
    ├─→ 节点2: 1
    ├─→ 节点3: 2
    └─→ 节点4: 3
  边: 1→2, 1→3, 1→4 (text: "")
```

**分支 B：对象中嵌套数组** ([parser.ts#L77-L100](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/parser.ts#L77-L100))

```
输入: {"fruits": ["apple", "banana"]}
生成:
  节点1: fruits: [2 items]   ← 聚合在父节点的一行
    ├─→ 节点2: "apple"
    └─→ 节点3: "banana"
  边: 1→2 (text: "fruits"), 1→3 (text: "fruits")
```

**分支 C：对象中嵌套对象** ([parser.ts#L101-L119](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/parser.ts#L101-L119))

```
输入: {"user": {"name": "Ada", "age": 36}}
生成:
  节点1: user: {2 keys}
    └─→ 节点2: name: "Ada"
             age: 36
  边: 1→2 (text: "user")
```

#### 边（Edge）的生成时机

边只在**跨节点引用**时生成：
1. **数组 → 元素**：父节点是数组时，每个元素创建一条边（`text: ""`）
2. **对象属性 → 嵌套数组元素**：属性值是数组时，每个数组元素创建一条边（`text: 属性名`）
3. **对象属性 → 嵌套对象**：属性值是对象时，创建一条边（`text: 属性名`）

扁平标量属性不会产生边，它们直接聚合在同一个节点的多行中。

#### 路径信息附加

每个节点创建时附加 `path`（JSON 路径）和父子信息：

```ts
const appendParentKey = () => {
  // 使用 jsonc-parser 的 getNodePath() 获取路径如 ["user", 0, "name"]
  const path = getNodePath(targetNode);
  // 根据 parent.type 判断是 "object" 还是 "array"
  // 返回 { parentKey, parentType }
};
```

这些路径信息用于**折叠/展开功能**（CollapseContext）。

---

## 阶段 4：节点尺寸计算

**代码位置：** [calculateNodeSize.ts](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/utils/calculateNodeSize.ts)

在节点被创建之前，必须计算其像素宽高，这样 ELK 布局引擎才能正确排布。

### 计算策略

1. **优先使用 DOM 测量**（浏览器环境）：
   - 创建一个隐藏的 `<div>`，设置与实际节点相同的字体（12px monospace）、padding、white-space
   - 将文本写入后读取 `getBoundingClientRect()`
   - 单行高度：36px，多行高度：行数 × 30px

2. **SSR 降级方案**：
   - 按字符数估算：`最长行 × 8px + 24px padding`
   - 最大宽度限制 700px，最小 45px

3. **性能优化**：
   - `sizeCache` (Map) 缓存计算结果，TTL 120 秒
   - 缓存 key：`JSON.stringify(text) + isParent`

---

## 阶段 5：画布布局与渲染

### 5.1 解析结果进入 React 状态

**代码位置：** [JSONCrackComponent.tsx#L170-L205](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L170-L205)

```ts
useEffect(() => {
  const result = parseJsonGraph(jsonText, maxRenderableNodes);
  // 三种结果：
  //   kind: "error"       → 清空节点，回调 onParseError
  //   kind: "above-limit" → 显示"节点超限"提示
  //   kind: "ok"          → setNodes + setEdges，回调 onParse
}, [jsonText, maxRenderableNodes]);
```

`parseJsonGraph()` 是 `canvasHelpers.ts` 中的包装函数，将 `parseGraph()` 的结果封装为**可辨识联合类型**（discriminated union），避免在 React 组件里处理异常。

### 5.2 折叠/展开过滤（CollapseContext）

**代码位置：** [CollapseContext.ts](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/components/CollapseContext.ts)

在节点和边送入 Canvas 之前，会根据折叠状态过滤：

```ts
const { visibleNodes, visibleEdges } = useMemo(() => {
  // 1. 收集所有被隐藏的节点 ID（其 path 以任意 collapsedPrefix 为前缀）
  // 2. 过滤掉 from 或 to 指向隐藏节点的边
}, [nodes, edges, collapsedPrefixes]);
```

`collapsedPrefixes` 是用户点击折叠按钮时记录的 JSON 路径集合，例如 `["user", "address"]` 序列化后存为字符串。

### 5.3 ELK 自动布局

**代码位置：** [JSONCrackComponent.tsx#L585-L611](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L585-L611)

使用 `reaflow` 库的 `<Canvas>` 组件，底层基于 **Eclipse Layout Kernel (ELK)** 的分层布局算法。

关键配置：
```ts
const layoutOptions = {
  "elk.layered.compaction.postCompaction.strategy": "EDGE_LENGTH",  // 紧凑布局
  "elk.layered.nodePlacement.strategy": "NETWORK_SIMPLEX",          // 节点放置策略
  "elk.spacing.edgeLabel": "15",                                    // 边标签间距
};

<Canvas
  nodes={visibleNodes}
  edges={visibleEdges}
  direction={layoutDirection}   // "LEFT" | "RIGHT" | "DOWN" | "UP"
  layoutOptions={layoutOptions}
  onLayoutChange={layout => {    // ELK 计算完成后回调
    // 更新 paneWidth / paneHeight（画布尺寸）
    // 设置 loading = false
  }}
/>
```

布局完成后，`onLayoutChange` 回调给出 ELK 计算的画布尺寸，同时触发自动居中效果。

### 5.4 节点渲染

**代码位置：** [CustomNode.tsx](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/components/CustomNode.tsx)

根据节点首行的 `key` 是否为 `null` 分派两种渲染：

| 条件 | 渲染组件 | 场景 |
|-----|---------|------|
| `text[0].key == null` | `TextNode` | 标量值节点、数组父节点（单行） |
| 否则 | `ObjectNode` | 对象节点（多行键值对表格） |

两者都是基于 `<foreignObject>` 的 HTML-in-SVG 渲染，保证复杂排版（多行、颜色、折叠按钮）的灵活性。

### 5.5 边渲染

**代码位置：** [CustomEdge.tsx](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/components/CustomEdge.tsx)

`reaflow` 默认生成贝塞尔曲线路径，`CustomEdge` 额外：
- 在边的中点渲染属性名标签
- 支持点击边定位目标节点
- 使用 CSS 变量 `--edge-stroke` 控制颜色

### 5.6 视口控制与自动居中

**代码位置：** [canvasHelpers.ts#L224-L260](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L224-L260)

画布外层使用 `react-zoomable-ui` 的 `<Space>` 组件提供平移/缩放。

首次布局完成后执行自动居中：
1. `computeGraphClientRect()` — 4 级降级策略计算图的实际包围盒：
   - ① 所有叶子 `<g[id]>` 的 getBoundingClientRect() 求并集（最精确）
   - ② 内容组 getBoundingClientRect()
   - ③ getBBox() × getScreenCTM() 矩阵变换
   - ④ ELK 返回的 layoutSize × SVG CTM
2. 按 2% padding 扩展包围盒
3. `viewPort.camera.centerFitAreaIntoView()` 平移 + 缩放适配视口

---

## 应用层集成

**代码位置：** [GraphView/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/features/editor/views/GraphView/index.tsx)

Next.js 应用层通过 Zustand store 将各模块串联：

```
useJson (json 字符串)
    ↓
<JSONCrack json={json} />
    ↓
useGraph.store ← 存储 viewPort/ref/direction/selectedNode
    ↓
Toolbar/ViewMenu 等 UI 组件调用 store 的 zoomIn/centerView/collapseAll
```

关键数据流：
- **输入**：`useJson` store 中的 `json` 字符串（可能来自编辑器、文件导入、URL 参数等）
- **输出**：用户点击节点 → `useGraph.setSelectedNode()` → 打开 `NodeModal` 显示详情
- **控制**：`useGraph` 通过 `jsonCrackRef` 调用 `JSONCrack` 暴露的 imperative API（zoomIn/out/centerView/collapseAll/expandAll）

---

## 关键测试用例参考

**代码位置：** [parser.test.ts](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/__tests__/parser.test.ts)

| 测试场景 | 预期节点数 | 预期边数 |
|---------|-----------|---------|
| 空字符串 | 0 | 0 |
| 单个字符串 `"hello"` | 1 | 0 |
| 扁平对象 4 个标量属性 | 1 | 0 |
| 嵌套对象 `{"user":{"name":"Ada"}}` | 2 | 1 |
| 对象 + 数组 3 元素 | 4 | 3 |
| 根级数组 `[1,2,3]` | 4（含数组父节点） | 3 |
| 深度嵌套 4 层对象 | 4 | 3 |
| 语法错误 `{"broken": }` | ≥ 0（容错） | errors.length > 0 |

---

## 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    apps/www (Next.js 应用层)                 │
│                                                              │
│  TextEditor / FileMenu / ImportModal                         │
│         ↓ json 字符串                                         │
│  useJson (Zustand store)                                     │
│         ↓                                                    │
│  GraphView ──→ useGraph (store viewPort/ref/direction)       │
│         ↓                                                    │
│  Toolbar 调用 ref.zoomIn() / ref.centerView() 等            │
└─────────────────────────────────────────────────────────────┘
                              ↓ json prop
┌─────────────────────────────────────────────────────────────┐
│              packages/jsoncrack-react (核心库)               │
│                                                              │
│  ┌─────────────────┐    ┌─────────────────┐                  │
│  │ canvasHelpers   │    │ CollapseContext │                  │
│  │  toJsonText()   │    │  collapsedSet   │                  │
│  │  parseJsonGraph │    │  isNodeHidden() │                  │
│  └────────┬────────┘    └────────┬────────┘                  │
│           ↓                      ↓ visibleNodes/Edges        │
│  ┌──────────────────────────────────────────┐                │
│  │           parser.ts : parseGraph()       │                │
│  │  jsonc-parser.parseTree → AST traverse   │                │
│  │         → NodeData[] + EdgeData[]        │                │
│  └──────────────────────┬───────────────────┘                │
│                         ↓                                    │
│  ┌──────────────────────────────────────────┐                │
│  │     calculateNodeSize.ts                 │                │
│  │     DOM 测量 / SSR 降级 + 缓存            │                │
│  └──────────────────────┬───────────────────┘                │
│                         ↓ nodes w/ width/height              │
│  ┌──────────────────────────────────────────┐                │
│  │        JSONCrackComponent.tsx            │                │
│  │  Space (react-zoomable-ui)               │                │
│  │    └─ Canvas (reaflow → ELK 布局)        │                │
│  │         ├─ CustomNode → ObjectNode/TextNode │             │
│  │         └─ CustomEdge                    │                │
│  └──────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```
