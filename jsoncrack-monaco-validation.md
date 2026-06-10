# JSON Crack — 编辑器与校验集成协作分析

## 1. 整体架构概览

整个数据流围绕三个核心 Zustand Store 和一条单向数据链路展开：

```
Monaco Editor (输入)
    │
    ▼
useFile Store (中转枢纽：内容、错误、格式)
    │
    ├─→ BottomBar (校验反馈 UI)
    │
    └─→ useJson Store (JSON 字符串)
            │
            ▼
        JSONCrack Component (画布渲染)
```

关键文件定位：

| 角色 | 文件路径 |
|------|----------|
| 编辑器页面布局 | [editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/pages/editor.tsx) |
| Monaco 文本编辑器 | [TextEditor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/features/editor/TextEditor.tsx) |
| 画布视图容器 | [LiveEditor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/features/editor/LiveEditor.tsx) |
| Graph 画布组件 | [GraphView/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/features/editor/views/GraphView/index.tsx) |
| 底部校验状态栏 | [BottomBar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/features/editor/BottomBar.tsx) |
| 文件/内容 Store | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/store/useFile.ts) |
| JSON Store | [useJson.ts](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/store/useJson.ts) |
| 配置 Store | [useConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/store/useConfig.ts) |
| 格式适配器 | [jsonAdapter.ts](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/lib/utils/jsonAdapter.ts) |
| 图谱解析器 | [parser.ts](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/packages/jsoncrack-react/src/parser.ts) |
| 画布辅助工具 | [canvasHelpers.ts](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts) |
| JSONCrack 核心组件 | [JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx) |
| Graph Store | [useGraph.ts](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/features/editor/views/GraphView/stores/useGraph.ts) |

---

## 2. 编辑器输入流程

### 2.1 Monaco Editor 绑定

[TextEditor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/features/editor/TextEditor.tsx) 是 Monaco 编辑器的宿主组件，核心绑定逻辑：

```tsx
// TextEditor.tsx L25-L31
const contents = useFile(state => state.contents);
const setContents = useFile(state => state.setContents);
const setError = useFile(state => state.setError);
const jsonSchema = useFile(state => state.jsonSchema);
const fileType = useFile(state => state.format);

// TextEditor.tsx L82-L92
<Editor
  language={fileType}                          // 语言模式：json/yaml/xml/csv
  theme={theme}                                 // vs-dark 或 light
  value={contents}                              // 受控值，来自 useFile.contents
  onValidate={errors => setError(errors[0]?.message || "")}  // Monaco 校验回调
  onChange={contents => setContents({ contents, skipUpdate: true })}  // 输入变更回调
/>
```

关键点：
- `value` 绑定到 `useFile.contents`，Monaco 为受控模式
- `onChange` 每次输入触发，传入 `skipUpdate: true` 标记
- `language` 根据当前 `format` 动态切换（json/yaml/xml/csv）
- `onValidate` 由 Monaco 内置校验器驱动，将首个错误写入 `useFile.error`

### 2.2 Monaco JSON Schema 配置

```tsx
// TextEditor.tsx L36-L53
React.useEffect(() => {
  if (!jsonDefaults) return;
  jsonDefaults.setDiagnosticsOptions({
    validate: true,
    allowComments: true,
    enableSchemaRequest: true,
    ...(jsonSchema && {
      schemas: [{
        uri: "http://myserver/foo-schema.json",
        fileMatch: ["*"],
        schema: jsonSchema,
      }],
    }),
  });
}, [jsonDefaults, jsonSchema]);
```

当用户通过 SchemaModal 设置了 `jsonSchema`，Monaco 的 JSON 语言服务会启用基于 Schema 的深度校验，错误同样通过 `onValidate` 回调上报。

---

## 3. 校验(Validation)反馈机制

校验发生在**两个独立层面**，分别由不同系统驱动：

### 3.1 第一层：Monaco 内置校验（编辑器层）

- **驱动者**：Monaco Editor 的 JSON/YAML 等语言服务
- **触发时机**：编辑器内容变化后自动异步执行
- **回调**：`onValidate` → `setError(errors[0]?.message || "")`
- **覆盖范围**：语法错误、Schema 验证错误
- **结果存储**：`useFile.error`

### 3.2 第二层：内容解析校验（数据转换层）

- **驱动者**：`useFile.setContents` 内部的 `contentToJson` 调用
- **触发时机**：`onChange` → `setContents({ contents, skipUpdate: true })`
- **实现位置**：[useFile.ts L100-L126](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/store/useFile.ts#L100-L126)
- **覆盖范围**：YAML/XML/CSV 的解析错误（这些格式 Monaco 不原生校验）

```ts
// useFile.ts L100-L126
setContents: async ({ contents, hasChanges = true, skipUpdate = false, format }) => {
  try {
    set({ ...(contents && { contents }), error: null, hasChanges, format: format ?? get().format });

    const json = await contentToJson(get().contents, get().format);

    if (!useConfig.getState().liveTransformEnabled && skipUpdate) return;

    // ...session 持久化...
    debouncedUpdateJson(json);
  } catch (error: any) {
    if (error?.mark?.snippet) return set({ error: error.mark.snippet });
    if (error?.message) set({ error: error.message });
    useJson.setState({ loading: false });
  }
},
```

`contentToJson` 根据当前格式动态导入解析器（jsonc-parser / js-yaml / fast-xml-parser / json-2-csv），解析失败时 catch 块写入 `error`。

### 3.3 第三层：图谱解析校验（画布层）

- **驱动者**：`parseGraph` in [parser.ts](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/packages/jsoncrack-react/src/parser.ts)
- **实现**：使用 `jsonc-parser` 的 `parseTree` 构建 AST，返回 `ParseError[]`
- **特点**：即使有语法错误，`parseTree` 仍会尽力返回部分树结构，画布会尝试渲染可解析的部分

### 3.4 BottomBar 校验反馈 UI

[BottomBar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/features/editor/BottomBar.tsx#L83-L132) 消费 `useFile.error` 展示校验状态：

```
error 存在 → 红色 VscError 图标 + "Invalid" 文字 + Popover 展示错误详情
error 为空 → 绿色 VscCheck 图标 + "Valid" 文字
```

同时 BottomBar 还控制两个关键交互：
- **Live Transform 开关**：决定编辑器输入是否实时同步到画布
- **Click to Transform 按钮**：Live Transform 关闭时，手动触发 `setContents({})` 强制同步

---

## 4. 画布(Canvas)更新流程

### 4.1 Live Transform 模式（默认开启）

数据流全链路：

```
Monaco onChange
  → setContents({ contents, skipUpdate: true })
    → contentToJson(contents, format)   // 解析为 JSON 对象
    → debouncedUpdateJson(json)         // 400ms 防抖
      → useJson.setJson(JSON.stringify(value, null, 2))
        → useJson.json 变更
          → GraphView 读取 useJson(state => state.json)
            → <JSONCrack json={json} />
              → toJsonText(json)          // 归一化为字符串
              → parseJsonGraph(jsonText)   // 解析为 nodes + edges
                → setNodes / setEdges      // React 状态更新
                  → reaflow <Canvas> 重渲染
```

### 4.2 手动 Transform 模式（Live Transform 关闭）

当 `liveTransformEnabled === false` 时，`setContents` 在检测到 `skipUpdate === true` 时提前 return，不会调用 `debouncedUpdateJson`。用户需要点击 BottomBar 的 "Click to Transform" 按钮，触发 `setContents({})`（无 skipUpdate），强制走完整个链路。

```ts
// useFile.ts L112
if (!useConfig.getState().liveTransformEnabled && skipUpdate) return;
```

### 4.3 JSONCrack 组件内部解析

[JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L167-L205) 内部通过 `useEffect` 监听 `jsonText` 变化：

```tsx
const jsonText = useMemo(() => toJsonText(json), [json]);

useEffect(() => {
  setLoading(true);
  setInitialFitDone(false);
  const result = parseJsonGraph(jsonText, maxRenderableNodes);

  if (result.kind === "error") {
    setNodes([]); setEdges([]); setLoading(false);
    callbacksRef.current.onParseError?.(result.error);
    return;
  }

  if (result.kind === "above-limit") {
    setTotalNodes(result.total); setAboveSupportedLimit(true);
    setNodes([]); setEdges([]); setLoading(false);
    return;
  }

  const { graph, syntaxErrorCount } = result;
  setTotalNodes(graph.nodes.length);
  setAboveSupportedLimit(false);
  setNodes(graph.nodes);
  setEdges(graph.edges);
  // ...
}, [jsonText, maxRenderableNodes]);
```

三层结果处理：
1. **error**：完全解析失败 → 清空画布
2. **above-limit**：节点超过 `maxRenderableNodes` → 显示超限提示
3. **ok**：正常渲染，`syntaxErrorCount > 0` 时仍会渲染部分图并上报错误

### 4.4 画布布局与适配

`reaflow` 的 `<Canvas>` 组件接收 `nodes` 和 `edges`，通过 ELK 算法自动布局。布局完成后 `onLayoutChange` 回调更新画布尺寸，触发 `fitGraphToViewPort` 自适应缩放。

---

## 5. 编辑器→校验→画布 完整关联链路

### 5.1 数据流时序图

```
用户输入 JSON
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│ Monaco Editor                                           │
│  • onDidChangeModelContent 触发                          │
│  • Monaco 语言服务异步校验                                 │
│  • 触发 onChange 回调                                     │
└───────────┬─────────────────────────┬───────────────────┘
            │                         │
     onValidate(errors)          onChange(contents)
            │                         │
            ▼                         ▼
    useFile.setError()      useFile.setContents({
      (错误消息)               contents,
                               skipUpdate: true
                             })
            │                         │
            │                         ▼
            │               ┌─────────────────────┐
            │               │ contentToJson()      │
            │               │  解析内容为 JSON 对象  │
            │               └────────┬────────────┘
            │                        │
            │                  ┌─────┴──────┐
            │                  │ 解析成功?    │
            │                  └─────┬──────┘
            │                   是 │      │ 否
            │                     ▼      ▼
            │            Live模式检查   useFile.set({ error })
            │               │
            │        ┌──────┴───────┐
            │        │ Live开启?     │
            │        └──────┬───────┘
            │          是 │      │ 否 + skipUpdate
            │             ▼      ▼
            │    debouncedUpdateJson  return (不更新画布)
            │    (400ms 防抖)
            │             │
            ▼             ▼
     BottomBar 读取   useJson.setJson()
     useFile.error    (JSON 字符串)
     显示 Valid/Invalid      │
                             ▼
                    GraphView 订阅
                    useJson(state => state.json)
                             │
                             ▼
                    <JSONCrack json={json} />
                             │
                             ▼
                    parseJsonGraph(jsonText)
                    (jsonc-parser parseTree)
                             │
                    ┌────────┴────────┐
                    │ 结果类型          │
                    ├─────────────────┤
                    │ ok → 渲染图       │
                    │ error → 清空画布  │
                    │ above-limit → 提示│
                    └─────────────────┘
```

### 5.2 关键状态变量说明

| 状态 | 所属 Store | 类型 | 作用 |
|------|-----------|------|------|
| `contents` | useFile | `string` | 编辑器原始文本内容 |
| `error` | useFile | `string \| null` | 当前校验错误消息 |
| `format` | useFile | `FileFormat` | 当前文件格式(json/yaml/xml/csv) |
| `jsonSchema` | useFile | `object \| null` | 用户设置的 JSON Schema |
| `hasChanges` | useFile | `boolean` | 是否有未保存的变更 |
| `json` | useJson | `string` | 已解析的 JSON 字符串，供画布消费 |
| `loading` | useJson | `boolean` | 画布加载状态 |
| `liveTransformEnabled` | useConfig | `boolean` | 是否实时同步编辑器到画布 |

### 5.3 两个防抖/节流点

1. **`debouncedUpdateJson`**（[useFile.ts L67-L69](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/store/useFile.ts#L67-L69)）：400ms 防抖，避免每次按键都触发画布重解析
2. **Monaco `onValidate`**：Monaco 内部已有防抖机制，不会在每个字符输入时立即回调

### 5.4 错误状态的两种来源与覆盖关系

| 来源 | 触发条件 | 写入方式 | 覆盖关系 |
|------|---------|---------|---------|
| Monaco onValidate | 任何内容变化 | `setError(errors[0]?.message \|\| "")` | 直接 set |
| contentToJson catch | YAML/XML/CSV 解析失败 | `set({ error: ... })` | 在 setContents 中 set |
| setContents try 成功 | 解析成功 | `set({ error: null })` | 清除错误 |

时序上：`setContents` 先将 `error` 设为 `null`（成功路径），然后 `contentToJson` 失败时重新设置 `error`。Monaco 的 `onValidate` 是异步的，可能在 `setContents` 之后才到达，因此可能覆盖 `setContents` 设置的错误状态。这意味着：
- **JSON 格式**：Monaco 校验结果具有最终话语权
- **YAML/XML/CSV 格式**：`contentToJson` 的错误占主导（Monaco 不原生支持这些格式的深度校验）

---

## 6. 协作不够直观的问题分析

### 6.1 双重校验源导致状态冲突

Monaco 的 `onValidate` 和 `setContents` 内部的 `contentToJson` 都会写入同一个 `useFile.error`，但它们的执行时序不确定。可能出现：
- `contentToJson` 解析成功（error=null），但 Monaco 随后报告语法错误
- `contentToJson` 解析失败设置了 error，Monaco 校验通过后用空字符串覆盖

### 6.2 skipUpdate 语义模糊

`setContents({ contents, skipUpdate: true })` 中的 `skipUpdate` 名字暗示"跳过更新"，但实际含义是"在 Live Transform 关闭时跳过画布同步"。它并不影响校验流程（校验总是执行），但这个命名容易让人误解。

### 6.3 校验与画布更新的耦合

在 `setContents` 中，校验（`contentToJson`）和画布更新（`debouncedUpdateJson`）是串行耦合的：
- 校验失败 → 不更新画布（正确）
- Live Transform 关闭 → 不更新画布（正确）
- 但校验成功 + Live Transform 关闭时，`contentToJson` 仍然被执行了，只是结果被丢弃

### 6.4 错误归一化不统一

- Monaco 错误：字符串消息（`errors[0]?.message`）
- YAML 错误：`error.mark.snippet`（代码片段）
- 通用错误：`error.message`
- 画布层错误：通过 `onParseError` 回调，但 GraphView 并未监听此回调

---

## 7. Store 依赖关系图

```
useConfig (持久化配置)
  ├── liveTransformEnabled  ──→  useFile.setContents 判断是否更新画布
  ├── darkmodeEnabled      ──→  TextEditor 主题 + GraphView 主题
  ├── gesturesEnabled      ──→  GraphView 手势
  └── rulersEnabled        ──→  GraphView 网格

useFile (内容枢纽)
  ├── contents             ──→  TextEditor value
  ├── error                ──→  BottomBar 校验状态
  ├── format               ──→  TextEditor language
  ├── jsonSchema           ──→  Monaco diagnostics options
  └── setContents()        ──→  useJson.setJson() (间接)

useJson (画布数据源)
  ├── json                 ──→  GraphView → JSONCrack json prop
  └── loading              ──→  (未被 www 直接消费)

useGraph (画布交互状态)
  ├── direction            ──→  JSONCrack layoutDirection
  ├── fullscreen           ──→  editor.tsx 面板可见性
  ├── selectedNode         ──→  NodeModal
  └── collapsedCount       ──→  (工具栏显示)
```

---

## 8. 无效输入时：内容解析失败、底部校验提示、画布保留旧图 三者关系

这是理解"无效输入时画布是否刷新"的核心。三者的行为由 `useFile.setContents` 的 try/catch 结构、`useJson.json` 的更新条件、以及 `JSONCrack` 组件的解析策略共同决定。

### 8.1 核心决策点：`useFile.setContents` 的 catch 分支

[useFile.ts L100-L126](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/store/useFile.ts#L100-L126) 的 try/catch 结构是整个联动关系的枢纽：

```ts
setContents: async ({ contents, hasChanges = true, skipUpdate = false, format }) => {
  try {
    // ① 先清除错误状态 + 更新 contents
    set({
      ...(contents && { contents }),
      error: null,              // ← 暂时标记为无错误
      hasChanges,
      format: format ?? get().format,
    });

    // ② 尝试解析内容
    const json = await contentToJson(get().contents, get().format);

    // ③ Live Transform 关闭且 skipUpdate → 不更新画布但也不报错
    if (!useConfig.getState().liveTransformEnabled && skipUpdate) return;

    // ...session 持久化...

    // ④ 解析成功 → 更新画布数据源
    debouncedUpdateJson(json);
  } catch (error: any) {
    // ⑤ 解析失败 → 只设置 error，不更新 useJson.json
    if (error?.mark?.snippet) return set({ error: error.mark.snippet });
    if (error?.message) set({ error: error.message });
    useJson.setState({ loading: false });
    // ⚠️ 注意：这里没有调用 debouncedUpdateJson！
  }
},
```

关键结论：
- **catch 分支只设置 `error`，绝不会调用 `debouncedUpdateJson`**
- **`useJson.json` 在解析失败时保持上一次成功解析的值不变**
- 这就是"无效输入时画布保留旧图"的根本原因

### 8.2 `contentToJson` 的容错策略（JSON 格式的特殊行为）

[jsonAdapter.ts L4-L13](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/lib/utils/jsonAdapter.ts#L4-L13) 对 JSON 格式有一个特殊的两步解析策略：

```ts
if (format === FileFormat.JSON) {
  const { parse } = await import("jsonc-parser");
  const errors: ParseError[] = [];
  const result = parse(value, errors);       // 第一步：jsonc-parser 容错解析
  if (errors.length > 0) JSON.parse(value);  // 第二步：有错误则用标准 JSON.parse 抛异常
  return result;
}
```

这意味着：
- `jsonc-parser.parse()` 本身是**容错的**，即使有语法错误也会尽力返回部分解析结果（不会抛异常）
- 只有当 `errors.length > 0` 时，才会调用原生 `JSON.parse()` 刻意触发异常
- 异常抛出后进入 `setContents` 的 catch 分支 → 设置 error + **不更新 useJson.json**

### 8.3 画布保留旧图 vs 清空画布的完整条件矩阵

画布内容由 `JSONCrack` 组件内部的 `nodes` 和 `edges` 两个 React state 决定。画布是否刷新取决于两层屏障：

| 屏障层 | 位置 | 条件 | 画布行为 |
|--------|------|------|---------|
| **第一层：useJson.json 是否更新** | [useFile.ts catch 分支](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/store/useFile.ts#L121-L125) | `contentToJson` 解析失败 → catch 分支 | `useJson.json` **不变** → GraphView 不触发重新渲染 → **画布保留旧图** |
| **第一层（续）** | [useFile.ts L120](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/store/useFile.ts#L120) | `contentToJson` 解析成功 + Live Transform 开启 → `debouncedUpdateJson` 被调用 | `useJson.json` **更新** → 进入第二层判断 |
| **第一层（续）** | [useFile.ts L112](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/store/useFile.ts#L112) | `contentToJson` 解析成功 + Live Transform 关闭 + `skipUpdate=true` | 提前 return → `useJson.json` **不变** → **画布保留旧图** |
| **第二层：JSONCrack 内部解析** | [JSONCrackComponent.tsx L170-L205](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L170-L205) | `parseJsonGraph` 返回 `kind: "error"` | `setNodes([]); setEdges([])` → **清空画布** |
| **第二层（续）** | 同上 | `parseJsonGraph` 返回 `kind: "ok"` + `syntaxErrorCount > 0` | 仍然 `setNodes(graph.nodes); setEdges(graph.edges)` → **渲染部分图** |
| **第二层（续）** | 同上 | `parseJsonGraph` 返回 `kind: "ok"` + `syntaxErrorCount === 0` | **完整渲染** |

> **重要区分**：
> - "保留旧图" = `useJson.json` 没变 → JSONCrack 组件的 `json` prop 没变 → 内部 `useEffect` 不触发 → nodes/edges 保持旧值
> - "清空画布" = `useJson.json` 变了 → JSONCrack 重新解析 → 解析彻底失败 → setNodes([]), setEdges([])

### 8.4 BottomBar 校验提示的错误来源与时序

[BottomBar.tsx L112-L131](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/features/editor/BottomBar.tsx#L112-L131) 的显示逻辑很简单：

```tsx
{error ? (
  // 红色 VscError + "Invalid" + Popover 显示 error 内容
) : (
  // 绿色 VscCheck + "Valid"
)}
```

但 `useFile.error` 的写入有三个独立来源，时序互相穿插：

| 写入来源 | 代码位置 | 写入值 | 触发时机 | 异步/同步 |
|----------|---------|--------|---------|-----------|
| **A: setContents try 块入口** | [useFile.ts L102-L107](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/store/useFile.ts#L102-L107) | `error: null` | 每次 `setContents` 被调用时（每次按键） | 同步 |
| **B: setContents catch 块** | [useFile.ts L121-L125](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/store/useFile.ts#L121-L125) | `error.mark.snippet` 或 `error.message` | `contentToJson` 解析失败时 | 异步（await contentToJson 后） |
| **C: Monaco onValidate 回调** | [TextEditor.tsx L89](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/features/editor/TextEditor.tsx#L89) | `errors[0]?.message \|\| ""` | Monaco 语言服务校验完成时 | 异步（Monaco 内部调度，延迟不确定） |

典型时序（用户输入无效 JSON `{bad`）：

```
T0: 用户按下按键
     ├─ Monaco onChange 同步触发 → setContents({ contents: "{bad", skipUpdate: true })
     │    ├─ A: 同步 set({ error: null, contents: "{bad" })
     │    │      → BottomBar 瞬时显示 "Valid"（闪烁）
     │    └─ B: 异步 await contentToJson("{bad}", "json")
     │          → jsonc-parser 容错返回部分结果，但 errors.length > 0
     │          → JSON.parse("{bad}") 抛 SyntaxError
     │          → catch: set({ error: "Unexpected token..." })
     │          → BottomBar 显示 "Invalid"
     │          → ⚠️ useJson.json 保持不变 → 画布保留旧图
     │
     └─ C: Monaco onValidate 稍后异步回调
          → setError("Expected ':'...")  ← 可能覆盖 B 设置的错误消息
          → BottomBar 仍显示 "Invalid"（消息内容可能变化）
```

典型时序（用户修正为有效 JSON `{"a":1}`）：

```
T0: 用户按下按键
     ├─ setContents({ contents: '{"a":1}', skipUpdate: true })
     │    ├─ A: 同步 set({ error: null, ... })
     │    │      → BottomBar 显示 "Valid"
     │    └─ B: contentToJson 成功 → { a: 1 }
     │          → debouncedUpdateJson({ a: 1 })  (400ms 防抖)
     │          → 400ms 后 useJson.json = '{\n  "a": 1\n}'
     │          → JSONCrack 重新解析 → 画布刷新
     │
     └─ C: Monaco onValidate 回调 → setError("")  ← 空字符串也是 falsy
          → BottomBar 仍显示 "Valid"
```

### 8.5 四种典型场景下三者的联动表

| 场景 | contentToJson 结果 | useFile.error | useJson.json | BottomBar 显示 | 画布行为 |
|------|-------------------|---------------|--------------|---------------|---------|
| **有效 JSON + Live 开启** | 成功 → `{...}` | `null`（try 块设置），随后 Monaco 返回 `""` | 更新（400ms 后） | Valid | 刷新为新图 |
| **无效 JSON** | 失败（catch） | 先 `null`（闪烁），后被 catch 设为错误消息，再可能被 Monaco 覆盖 | **不变** | Invalid（瞬时 Valid 闪烁） | **保留旧图** |
| **有效 JSON + Live 关闭 + skipUpdate** | 成功 → `{...}` | `null` | **不变**（被 L112 return 拦截） | Valid | **保留旧图**（直到点 Click to Transform） |
| **有效 YAML + Monaco 无 YAML 校验** | 成功 → `{...}` | `null`（Monaco 可能也返回 `""`） | 更新（400ms 后） | Valid | 刷新为新图 |
| **无效 YAML** | 失败（catch） | catch 设置 `error.mark.snippet`（YAML js-yaml 的错误格式） | **不变** | Invalid | **保留旧图** |
| **空内容 `""`** | contentToJson 返回 `{}`（L5：`if (!value) return {}`） | `null` | 更新为 `"{}"` | Valid | 刷新为空对象图 |

### 8.6 一个容易混淆的边界：空字符串

[jsonAdapter.ts L5](file:///d:/fz/0601/solo-dogfeeding/code/183-jsoncrack.com/apps/www/src/lib/utils/jsonAdapter.ts#L5) 的特殊处理：

```ts
export const contentToJson = async (value: string, format = FileFormat.JSON): Promise<object> => {
  if (!value) return {};  // ← 空字符串不抛异常，返回空对象
  // ...
};
```

当编辑器内容被清空时：
- `contentToJson("")` 返回 `{}`，不会进 catch
- `useFile.error` 被设为 `null` → BottomBar 显示 Valid
- `debouncedUpdateJson({})` 被调用 → `useJson.json = "{}"` → 画布刷新为一个空对象节点

这解释了为什么"删除所有内容"会让画布变成单个空对象节点，而不是保留旧图。

### 8.7 另一个边界：第二层清空画布的触发条件

第一层屏障（`useJson.json` 不变）通常已经阻止了无效输入到达画布。但在以下特殊情况下，第二层屏障也会触发：

1. **手动通过 API 直接设置 `useJson.setJson("invalid")`**（绕过了 useFile 的校验）
2. **`contentToJson` 成功但 `jsonc-parser.parseTree` 完全无法构建 AST**：
   - `parseTree` 返回 `null` → `parseGraph` 返回 `{ nodes: [], edges: [], errors: [...] }`
   - `parseJsonGraph` 返回 `{ kind: "ok", graph: {nodes:[], edges:[]}, syntaxErrorCount: N }`
   - 由于 `kind` 不是 "error"，JSONCrack 会执行 `setNodes([])` + `setEdges([])` → **画布清空（但显示空白，不是保留旧图）**

注意 `parseGraph` 返回空数组时，JSONCrack 走的是 `kind: "ok"` 分支（不是 `kind: "error"`），所以不会调用 `onParseError`，只会把 nodes/edges 设为空数组。

---

## 9. 总结：三者关系的本质

```
                    ┌─────────────────────────────┐
                    │   用户输入无效内容            │
                    └─────────────┬───────────────┘
                                  │
                                  ▼
              ┌───────────────────────────────────────┐
              │  setContents try {                     │
              │    ① error = null   (BottomBar 瞬时Valid)│
              │    ② contentToJson → 抛异常            │
              │  } catch {                              │
              │    ③ error = "错误消息" (BottomBar Invalid)│
              │    ④ ⚠️ 不调用 debouncedUpdateJson       │
              │  }                                      │
              └───────────────┬────────────────────────┘
                              │
              ┌───────────────┴────────────────────────┐
              │                                        │
              ▼                                        ▼
    useFile.error 被设置                      useJson.json 保持不变
    → BottomBar 显示 Invalid                  → GraphView 不重渲染
                                              → JSONCrack 的 json prop 不变
                                              → 内部 useEffect 不触发
                                              → nodes/edges 保留旧值
                                              → ✅ 画布保留旧图
```

一句话概括：**内容解析失败时，catch 分支只更新 error（影响 BottomBar），但绝不更新 useJson.json（影响画布），这就是底部显示 Invalid 但画布仍显示旧图的设计原理。**
