# JSONCrack 边界路径深度分析

本文聚焦三个容易混淆的边界场景：
1. **入口边界**：从编辑器输入到解析文本的完整链路（含对象引用缓存）
2. **分流边界**：语法错误、节点超限、解析异常三种结果如何精细分流
3. **折叠边界**：行级路径到子节点隐藏的完整生命周期（含折叠按钮屏幕坐标固定）

---

## 一、入口边界：对象 → 解析文本的数据流

完整链路跨越 **7 层**：Monaco 编辑器 → useFile → useJson → JSONCrack 组件 → toJsonText → parseJsonGraph → parseGraph。

### 1.1 最上层：编辑器输入 → useFile

**代码位置：** [TextEditor.tsx#L82-L92](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/features/editor/TextEditor.tsx#L82-L92)

```tsx
<Editor
  value={contents}
  onChange={contents => setContents({ contents, skipUpdate: true })}
  onValidate={errors => setError(errors[0]?.message || "")}
/>
```

`onChange` 每一次按键都会调用 `useFile.setContents()`，但传入了 `skipUpdate: true` 标记。

### 1.2 useFile：400ms 防抖 + 格式适配

**代码位置：** [useFile.ts#L67-L69](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/store/useFile.ts#L67-L69)

```ts
const debouncedUpdateJson = debounce((value: unknown) => {
  useJson.getState().setJson(JSON.stringify(value, null, 2));
}, 400);
```

**代码位置：** [useFile.ts#L100-L126](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/store/useFile.ts#L100-L126)

`setContents()` 内部流程：
```
setContents({ contents, skipUpdate })
    ↓
① 更新 store.contents / error=null / hasChanges
    ↓
② contentToJson(contents, format)  ← 格式适配（JSON/YAML/XML/CSV → JS 对象）
    ↓
③ skipUpdate 开关检查：
   ├─ liveTransformEnabled=false 且 skipUpdate=true → return（不触发图更新）
   └─ 否则 → debouncedUpdateJson(json)
                    ↓
                  400ms 防抖 → useJson.setJson(JSON.stringify(...))
```

**关键边界：**
- `skipUpdate=true` + `liveTransformEnabled=true` → **仍然**更新图（`liveTransformEnabled` 优先级更高）
- 有修改且内容 < 80KB → 写入 sessionStorage 持久化
- `contentToJson` 抛异常 → 只写 `store.error`，**不**更新 useJson（图保持上次状态）

### 1.3 入口来源不止编辑器：5 条写入 useFile 的路径

| 路径 | 代码位置 | 说明 |
|-----|---------|------|
| 编辑器输入 | `TextEditor` `onChange` | 高频，防抖 400ms |
| URL 参数 `?json=...` | [useFile.ts#L142-L145](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/store/useFile.ts#L142-L145) | `checkEditorSession()` 识别 URL，调 `fetchUrl()` |
| 文件导入 | `ImportModal` → `setFile()` | 一次性写入，无防抖 |
| 格式切换 | `ViewMenu` → `setFormat()` | 走 `jsonToContent` 重新序列化 |
| 会话恢复 | `checkEditorSession()` 读 sessionStorage | 页面刷新后恢复 |

### 1.4 JSONCrack 组件内部：toJsonText + WeakMap 引用缓存

**代码位置：** [canvasHelpers.ts#L8-L26](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L8-L26)

```ts
const objectJsonCache = new WeakMap<object, string>();

export const toJsonText = (json: JsonInput): string => {
  if (typeof json === "string") return json;                // ① 字符串直接返回

  if (json && typeof json === "object") {
    const cached = objectJsonCache.get(json);               // ② 按对象引用查 WeakMap
    if (cached) return cached;
    const serialized = JSON.stringify(json, null, 2);
    objectJsonCache.set(json, serialized);                  // ③ 写入缓存
    return serialized;
  }

  return JSON.stringify(json, null, 2);                     // ④ null / undefined / 原始值
};
```

#### WeakMap 设计的三个边界含义

| 场景 | 行为 | 为什么这样设计 |
|-----|------|--------------|
| **同一对象引用重复传入** | 命中缓存，跳过 `JSON.stringify` | 避免父组件每 render 重建引用但实际值未变时的重复序列化 |
| **相同内容但不同引用** | 缓存失效，重新序列化 | WeakMap 是**引用等价**不是值等价——`{a:1} !== {a:1}`，这是有意为之，强迫调用方做 `useMemo` |
| **对象被 GC** | 缓存条目自动清除 | WeakMap 的 key 是弱引用，不阻止垃圾回收，防内存泄漏 |

### 1.5 useEffect 依赖链：为什么用 jsonText 而不是 json

**代码位置：** [JSONCrackComponent.tsx#L167-L205](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L167-L205)

```ts
// 第一步：useMemo 将引用类型归一化为字符串
const jsonText = useMemo(() => toJsonText(json), [json]);

// 第二步：useEffect 只依赖字符串（原始类型，稳定比较）
useEffect(() => {
  const result = parseJsonGraph(jsonText, maxRenderableNodes);
  // ...
}, [jsonText, maxRenderableNodes]);
```

**关键推理：**
- 如果直接依赖 `json` prop（`string | object`），当 `json` 是对象时，父组件每次 render 传新引用都会触发重解析
- `useMemo` + `toJsonText` 形成两层过滤：
  - **第一层**：`[json]` 引用变化才重新跑 `toJsonText`
  - **第二层**：`jsonText` 字符串内容相同 → useEffect 跳过（React 对原始类型做值比较）
- **副作用**：相同内容但不同引用的对象 → `toJsonText` 跑两次但 `parseGraph` 只跑一次（因为序列化后的字符串相同）

### 1.6 入口边界全景图

```
Monaco Editor (onChange 每按键)
    │  setContents({ contents, skipUpdate: true })
    ▼
useFile.setContents()
    │  ① contentToJson(contents, format) — 多格式转 JS 对象
    │  ② 可选 sessionStorage 持久化（<80KB）
    │  ③ debouncedUpdateJson() ← 400ms 防抖
    ▼
useJson.setJson(jsonString)  ← Zustand store
    │  组件订阅 useJson(state => state.json)
    ▼
<JSONCrack json={jsonString} />
    │  ┌────────────────────────────────────────┐
    │  │  jsonText = useMemo(() =>              │
    │  │    toJsonText(json), [json])           │
    │  │    └─ string: 直接返回                  │
    │  │    └─ object: WeakMap 缓存命中?         │
    │  │       └─ 是 → 返回缓存                  │
    │  │       └─ 否 → JSON.stringify + 缓存    │
    │  └────────────────────────────────────────┘
    ▼
useEffect([jsonText, maxRenderableNodes])
    │  parseJsonGraph(jsonText, maxNodes)
    ▼
parseGraph() → 节点 + 边
```

---

## 二、分流边界：语法错误 / 节点超限 / 解析异常

### 2.1 可辨识联合：parseJsonGraph 的三态返回

**代码位置：** [canvasHelpers.ts#L67-L90](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L67-L90)

```ts
export type ParseJsonGraphResult =
  | { kind: "ok";          graph: GraphData; syntaxErrorCount: number }  // 正常（可能含容错错误）
  | { kind: "above-limit"; total: number }                              // 节点超限
  | { kind: "error";       error: Error };                              // 运行时异常
```

#### 三种 kind 的触发条件

| kind | 触发源 | 判定条件 |
|-----|--------|---------|
| `"error"` | `try/catch` | `parseGraph()` 内部抛异常（通常是 `jsonc-parser` 的极端情况） |
| `"above-limit"` | `if (graph.nodes.length > maxRenderableNodes)` | 解析成功但节点数超过阈值（默认 1500） |
| `"ok"` | 默认分支 | 解析成功且节点数 ≤ 阈值 |

**注意：** `"ok"` 状态下 `syntaxErrorCount` 可能 > 0，这是 jsonc-parser **容错解析**的结果——有语法错误但仍生成了部分图。

### 2.2 三态在 React 中的精细状态转移

**代码位置：** [JSONCrackComponent.tsx#L170-L205](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L170-L205)

```
进入 useEffect([jsonText])
    │
    ├─ setLoading(true)
    ├─ setInitialFitDone(false)
    ▼
parseJsonGraph()
    │
    ├─────────────────────────────────────────────────────┐
    │ kind="error"                                        │
    │   setNodes([])                                      │
    │   setEdges([])                                      │
    │   setLoading(false)   ← 立即关 loading              │
    │   onParseError(error)                               │
    │   return                                            │
    │                                                     │
    ├─────────────────────────────────────────────────────┤
    │ kind="above-limit"                                  │
    │   setTotalNodes(total)                              │
    │   setAboveSupportedLimit(true)                      │
    │   setNodes([])                                      │
    │   setEdges([])                                      │
    │   setLoading(false)   ← 立即关 loading              │
    │   return                                            │
    │                                                     │
    └─────────────────────────────────────────────────────┘
      kind="ok"
        │  syntaxErrorCount > 0 ?
        │    └─ 是 → onParseError(new Error("N syntax error(s)"))
        │         （仍继续渲染！只是通知外部）
        ├─ setTotalNodes(graph.nodes.length)
        ├─ setAboveSupportedLimit(false)
        ├─ setNodes(graph.nodes)
        ├─ setEdges(graph.edges)
        ├─ onParse({ nodes, edges })
        └─ graph.nodes.length === 0 ?
             ├─ 是 → setLoading(false)   ← 空图立即关
             └─ 否 → loading 保持 true，等 ELK onLayoutChange 关
```

### 2.3 四个关键分流细节

#### 细节 1："有语法错误" ≠ "解析失败"

```ts
// kind="ok" 但 syntaxErrorCount > 0 时
if (syntaxErrorCount > 0) {
  callbacksRef.current.onParseError?.(
    new Error(`Failed to parse data (${syntaxErrorCount} syntax error(s)).`)
  );
}
// 继续 setNodes + setEdges，图照常渲染
```

这是与常见「错误就白屏」不同的设计——调用方收到 `onParseError` 回调，但同时也能拿到部分可渲染的图。UI 层可以选择显示警告但展示已有内容。

#### 细节 2：节点超限仍然完整解析

```ts
const graph = parseGraph(jsonText);              // ① 完整跑遍历
if (graph.nodes.length > maxRenderableNodes) {   // ② 然后才判断
  return { kind: "above-limit", total: graph.nodes.length };
}
```

**代价**：大 JSON 即使超限也会跑完整的 traverse + calculateNodeSize。  
**设计原因**：需要在「超限提示」里显示准确的节点总数（`totalNodes`），如果提前终止就不知道实际有多少。

#### 细节 3：loading 状态由两条路径关闭

| 场景 | 关闭时机 | 代码位置 |
|-----|---------|---------|
| 空图 / 出错 / 超限 | useEffect 内同步 `setLoading(false)` | [JSONCrackComponent.tsx#L179](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L179) 等 |
| 有节点的正常图 | ELK `onLayoutChange` 回调 | [JSONCrackComponent.tsx#L401-L411](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L401-L411) |

为什么正常情况不在 useEffect 里关？因为 `setNodes` 后 reaflow 需要跑 ELK 布局算法（几百 ms ~ 几秒），此时关 loading 会让用户看到空画布。必须等 ELK 回调证明布局有了尺寸才关。

#### 细节 4：callbacksRef 避免重跑解析

```ts
const callbacksRef = useRef({ onParse, onParseError });
useEffect(() => {
  callbacksRef.current = { onParse, onParseError };
}, [onParse, onParseError]);

// 解析 useEffect 只依赖 jsonText / maxRenderableNodes，不依赖 callbacks
useEffect(() => {
  // ...
  callbacksRef.current.onParse?.(...);  // 通过 ref 读最新
}, [jsonText, maxRenderableNodes]);
```

如果直接把 `onParse` 放进 useEffect deps，父组件每次 render 传新的箭头函数都会触发重新解析。用 `useRef` 镜像打破了这个依赖链。

---

## 三、折叠边界：行级路径 → 子节点隐藏的完整生命周期

### 3.1 折叠按钮在哪：ObjectNode 的 Row 组件

**代码位置：** [ObjectNode.tsx#L25-L90](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/components/ObjectNode.tsx#L25-L90)

```tsx
const Row = ({ row, x, y, index, parentPath }: RowProps) => {
  // 判断这一行是否是可折叠的容器（对象/数组且有子元素）
  const isContainer =
    (row.type === "object" || row.type === "array") && (row.childrenCount ?? 0) > 0;

  // 构造这一行的 JSON 路径 = 父节点路径 + 当前行的 key
  // 例：parentPath=["users"], row.key=0 → rowPath=["users", 0]
  const rowPath: JSONPath | null = React.useMemo(
    () => (isContainer && row.key != null ? [...parentPath, row.key] : null),
    [isContainer, parentPath, row.key]
  );

  // 查询当前是否已折叠
  const collapsed = rowPath != null && isPathCollapsed(collapsedSet, rowPath);
```

#### 路径构造的边界规则

| row.key | row.type | 有 childrenCount | rowPath | 显示折叠按钮？ |
|---------|----------|----------------|---------|--------------|
| `"name"` | `"string"` | - | `null` | 否 |
| `"address"` | `"object"` | 0 | `null` | 否（空对象不折叠） |
| `"address"` | `"object"` | 3 | `[...parentPath, "address"]` | 是 |
| `"items"` | `"array"` | 10 | `[...parentPath, "items"]` | 是 |

**为什么 `row.key == null` 时不构造 rowPath？**
数组的根节点（`[N items]` 那行）`row.key` 为 `null`，它的路径就是 `parentPath` 本身。如果它自己也有子节点，由数组元素对应的独立节点去处理折叠——不在这里。

### 3.2 点击折叠按钮：wrappedToggleCollapse 的三件事

**代码位置：** [JSONCrackComponent.tsx#L306-L332](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L306-L332)

```ts
const wrappedToggleCollapse = useCallback((path: JSONPath) => {
  const key = JSON.stringify(path);

  // ── 第一件事：记录按钮当前屏幕坐标 ──
  const btn = findCollapseButton(key);
  if (btn) {
    const rect = btn.getBoundingClientRect();
    pendingRecenterRef.current = {
      key,
      clientX: rect.left + rect.width / 2,   // 按钮中心点 X
      clientY: rect.top + rect.height / 2,   // 按钮中心点 Y
    };
    setIsRelayouting(true);  // 画布进入"重新布局中"→ visibility:hidden
  } else {
    pendingRecenterRef.current = null;
  }

  // ── 第二件事：切换 collapsed 状态 ──
  if (isControlled) {
    controlledOnToggle?.(path);      // 受控模式：交给外部
    return;
  }
  setInternalCollapsedPaths(prev => {
    const next = prev.includes(key)
      ? prev.filter(p => p !== key)  // 已折叠 → 展开
      : [...prev, key];              // 未折叠 → 折叠
    onCollapseChangeRef.current?.(next);
    return next;
  });

  // ── 第三件事（自动）：collapsedSet 变化 → useEffect 触发 setLoading(true) ──
}, [...]);
```

#### `isRelayouting` 的视觉效果

```tsx
<Space
  style={{
    opacity: initialFitDone && !isRelayouting ? 1 : 0,
    visibility: isRelayouting ? "hidden" : "visible",  // ← 关键
    transition: "opacity 120ms",
  }}
>
```

折叠后的 ELK 重新布局会让所有节点坐标大幅跳变，用户会看到「内容飞过去再回到按钮下」的闪动感。`visibility:hidden` 完全隐藏这个过程，等相机平移到位后再显示。

### 3.3 路径过滤：collapsedPrefixes → visibleNodes / visibleEdges

#### 预处理：Set<string> → JSONPath[]

**代码位置：** [JSONCrackComponent.tsx#L252-L267](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L252-L267)

```ts
// collapsedSet: Set<'["users"]' | '["config","theme"]'>
const collapsedPrefixes = useMemo<JSONPath[]>(() => {
  const out: JSONPath[] = [];
  for (const key of collapsedSet) {
    try {
      out.push(JSON.parse(key) as JSONPath);   // 反序列化为真实路径数组
    } catch { /* skip malformed */ }
  }
  return out;
}, [collapsedSet]);
```

**为什么不在每个节点里 `JSON.parse`？**  
如果有 1000 个节点 × 5 个折叠路径 = 5000 次 `JSON.parse`。这里一次性解析 5 次，后面每个节点只做数组比较（O(1) × 前缀长度）。

#### isNodeHidden：前缀匹配算法

**代码位置：** [CollapseContext.ts#L20-L37](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/components/CollapseContext.ts#L20-L37)

```ts
export const isNodeHidden = (
  collapsedPrefixes: readonly JSONPath[],  // 已折叠的路径列表
  nodePath: JSONPath | undefined           // 当前节点的路径
): boolean => {
  if (!nodePath || collapsedPrefixes.length === 0) return false;

  for (const prefix of collapsedPrefixes) {
    if (prefix.length > nodePath.length) continue;  // 前缀比路径长，不可能匹配

    let matches = true;
    for (let i = 0; i < prefix.length; i += 1) {
      if (prefix[i] !== nodePath[i]) {   // 严格 === 比较
        matches = false;
        break;
      }
    }
    if (matches) return true;  // 任意一个前缀匹配 → 隐藏
  }
  return false;
};
```

#### 前缀匹配的边界用例

| 折叠路径 prefix | 节点路径 nodePath | 结果 | 原因 |
|---------------|-----------------|------|------|
| `["users"]` | `["users", 0]` | ✅ 隐藏 | 前缀匹配 |
| `["users"]` | `["users", 0, "name"]` | ✅ 隐藏 | 前缀匹配（所有后代都隐藏） |
| `["users"]` | `["usersOther"]` | ❌ 不隐藏 | 严格字符串比较：`"users" !== "usersOther"` |
| `["users"]` | `[]`（根节点） | ❌ 不隐藏 | `prefix.length > nodePath.length` |
| `["users", 0, "address"]` | `["users", 0]` | ❌ 不隐藏 | 前缀比路径长 → continue |
| `[0]`（数字） | `["0"]`（字符串） | ❌ 不隐藏 | `0 !== "0"`，`===` 区分类型 |

测试验证参见：[CollapseContext.test.ts#L17-L60](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/__tests__/CollapseContext.test.ts#L17-L60)

#### visibleEdges：级联过滤

**代码位置：** [JSONCrackComponent.tsx#L269-L282](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L269-L282)

```ts
const { visibleNodes, visibleEdges } = useMemo(() => {
  if (collapsedPrefixes.length === 0) return { visibleNodes: nodes, visibleEdges: edges };

  const hiddenIds = new Set<string>();
  const keptNodes = [];
  for (const node of nodes) {
    if (isNodeHidden(collapsedPrefixes, node.path)) {
      hiddenIds.add(node.id);
    } else {
      keptNodes.push(node);
    }
  }

  // 边的任一端点被隐藏 → 整条边也隐藏
  const keptEdges = edges.filter(
    edge => !hiddenIds.has(edge.from) && !hiddenIds.has(edge.to)
  );
  return { visibleNodes: keptNodes, visibleEdges: keptEdges };
}, [nodes, edges, collapsedPrefixes]);
```

**不处理悬空边的情况**：因为折叠隐藏是「前缀 → 整个子树」，不会出现 `from` 可见而 `to` 隐藏的单边。任何边的两个节点要么都可见（在上层），要么都隐藏（在被折叠的子树里）。

### 3.4 prunePaths：JSON 变更后清理无效折叠路径

**代码位置：** [CollapseContext.ts#L44-L72](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/components/CollapseContext.ts#L44-L72)

**代码位置：** [JSONCrackComponent.tsx#L245-L250](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L245-L250)

```ts
useEffect(() => {
  if (isControlled) return;
  if (internalCollapsedPaths.length === 0) return;
  const kept = prunePaths(jsonText, internalCollapsedPaths);
  if (kept.length !== internalCollapsedPaths.length) setInternalCollapsedPaths(kept);
}, [jsonText, internalCollapsedPaths, isControlled]);
```

`prunePaths` 做什么？

```
场景：用户折叠了 ["users"]，然后编辑 JSON 删除了 users 字段
      → 折叠路径 ["users"] 现在指向不存在的值
      → 如果不清理，下次 JSON 加回 users 时会意外保持折叠

算法：
  for each 序列化的路径 key:
    ① JSON.parse(key) → 解析失败则丢弃
    ② 顺着 path 在 JSON 对象里逐段导航：
         for (const seg of path) { cur = cur[seg] }
       任何一步 cur 不是对象 → 丢弃
    ③ 最终到达的值必须是 object / array（标量值不能折叠）
    ④ 以上都通过 → 保留
```

**特殊边界**：当 JSON 本身解析失败时（用户正在输入中间态），`prunePaths` **原样返回**所有路径，避免「打错一个字就清空所有折叠状态」。参见测试 [CollapseContext.test.ts#L96-L99](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/__tests__/CollapseContext.test.ts#L96-L99)。

### 3.5 折叠后：rAF 轮询 + 相机平移固定按钮位置

这是最精妙的 UX 细节。用户点击折叠按钮时，按钮在屏幕坐标 (100, 200)。ELK 重新布局后整个图会平移，导致按钮飞到 (300, 500)。为了让用户感觉「按钮没动」，需要把相机也往相同方向偏移。

**代码位置：** [JSONCrackComponent.tsx#L418-L483](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L418-L483)

```
时序：
  T0: 用户点击 +/− 按钮
       wrappedToggleCollapse()
         ├─ 记录按钮当前中心坐标 (100, 200) → pendingRecenterRef
         ├─ setIsRelayouting(true)          ← 画布隐藏
         └─ setInternalCollapsedPaths([...]) ← 触发重渲染

  T1: collapsedSet 变化
       → visibleNodes/visibleEdges 重新计算
       → reaflow Canvas 检测到节点变化，开始跑 ELK

  T2: visibleNodes 变化触发 useEffect（本函数）
       ┌─ pendingRecenterRef 有值？
       ├─ 是：开始 requestAnimationFrame 轮询
       └─ 否：直接 return

  轮询循环 tick()：
    ① 通过 data-collapse-path 属性重新找到按钮 DOM
       （如果按钮还没渲染出来 → 继续下一帧，最多等 180 帧 ≈ 3 秒）
    ② 读取新的按钮中心坐标 (300, 500)
    ③ 判断稳定性：
         lastX !== null 时，对比与上一帧的差值
         ├─ 移动了 >0.5px → movementSeen = true，继续等待
         ├─ 没动 且 movementSeen 已为 true → 「稳定了！」执行平移
         └─ 没动 但还没见过移动 → 可能还没开始布局，继续等
    ④ 最多 180 次尝试（约 3 秒），超时则放弃（但如果见过至少一次移动就应用）

  T3: 稳定后 finish(x=300, y=500)：
       dx = 300 - 100 = +200
       dy = 500 - 200 = +300
       viewPort.camera.moveByInClientSpace(200, 300)  ← 相机跟着偏移
       setIsRelayouting(false)                        ← 显示画布
```

为什么需要「先看到移动 → 再等稳定」，而不是直接取第一次的值？

因为 reaflow 内部用 framer-motion 做节点动画，新布局不是一帧到位。如果直接取第一次测量，动画还在跑，相机偏移会不完整。必须等「按钮在两帧之间没动」才能确定 ELK + 动画都跑完了。

### 3.6 折叠边界全景图

```
用户点击 ObjectNode.Row 上的 [+/−] 按钮
           │
           ▼  ObjectNode.tsx handleToggle()
           │  event.stopPropagation()  ← 不触发节点点击
           │  onToggleCollapse(rowPath)
           ▼
   wrappedToggleCollapse(path)  ← JSONCrackComponent 层
     │
     ├─ ① findCollapseButton() → 读当前屏幕坐标
     │     pendingRecenterRef = { clientX, clientY }
     │     setIsRelayouting(true)  ← Space.visibility = "hidden"
     │
     └─ ② setInternalCollapsedPaths([...])
              │
              ▼  collapsedSet = new Set(collapsedPaths)
              ▼  collapsedPrefixes = JSON.parse() 预处理
              ▼  visibleNodes/visibleEdges = isNodeHidden() 过滤
              │
     ┌────────┘
     ▼
  visibleNodes/visibleEdges 变化 → reaflow Canvas 重新布局
     │
     ▼
  useEffect([visibleNodes, visibleEdges]) 触发：
     │  pendingRecenterRef 有值？
     ▼  是 → requestAnimationFrame(tick) 轮询
              每帧：
                findCollapseButton(key) 再找按钮
                读新坐标 → 与上一帧比较
                移动过且两帧稳定？
                  ├─ 是：moveByInClientSpace(dx, dy) 相机平移
                  │       setIsRelayouting(false)  显示画布
                  └─ 否：继续下一帧（最多 180 次）
     │
     ▼
  用户看到：按钮位置没变（实际是图动了 + 相机也动了，互相抵消）
           折叠/展开的内容在按钮下方出现/消失
```

---

## 四、三条边界的共同设计模式

### 4.1 渐进降级而非硬失败

| 边界 | 降级策略 |
|-----|---------|
| 入口 | WeakMap 失效 → 重序列化；格式适配失败 → 保留旧图不白屏 |
| 错误分流 | 语法错误 → 通知但仍渲染；解析异常 → 至少清空 loading；JSON 坏掉 → 不清理折叠路径 |
| 折叠 | 找不到按钮 → 不 recenter 但正常折叠；rAF 轮询超时 → 放弃平移但不卡死 |

### 4.2 引用 vs 值的双重缓存策略

```
入口层：WeakMap<object, string>  — 引用等价，防重复序列化
折叠层：Set<string> (pathKey)    — 值等价，防重复前缀匹配
尺寸层：Map<string, Size>        — 值等价，DOM 测量缓存 + TTL
```

### 4.3 useEffect 依赖瘦身

所有回调都用 `useRef` 镜像，确保解析 effect、折叠 effect 的 deps 尽可能少（只含真正改变数据的变量），避免父组件 render 传新箭头函数导致的意外重跑。
