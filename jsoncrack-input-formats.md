# JSON Crack 输入格式多样性与统一解析链路分析

本文档按代码调用顺序，追踪不同来源、不同格式的输入如何汇聚到同一条解析链路，最终被渲染为可视化图。

---

## 1. 支持的输入格式枚举

所有支持的文件格式定义在 [file.enum.ts](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/enums/file.enum.ts#L1-L6)：

```ts
export enum FileFormat {
  "JSON" = "json",
  "YAML" = "yaml",
  "XML"  = "xml",
  "CSV"  = "csv",
}
```

---

## 2. 输入入口全景（6 种来源）

| # | 入口 | 触发方式 | 格式 | 所在文件 |
|---|------|---------|------|---------|
| 1 | Monaco 编辑器实时输入 | 键盘编辑 | JSON / YAML / XML / CSV | [TextEditor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/features/editor/TextEditor.tsx#L90) |
| 2 | 拖拽文件到全屏 Dropzone | 拖拽 `.json/.yaml/.xml/.csv` 文件 | 按文件扩展名推断 | [FullscreenDropzone.tsx](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/features/editor/FullscreenDropzone.tsx#L17-L27) |
| 3 | ImportModal（URL 获取 / 文件上传） | 菜单 → Import | URL 返回 JSON；文件按扩展名推断 | [ImportModal/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/features/modals/ImportModal/index.tsx#L18-L47) |
| 4 | URL 查询参数 `?json=URL` | 编辑器页面加载时 | 远程 JSON | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/store/useFile.ts#L129-L141) → [editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/pages/editor.tsx#L116-L117) |
| 5 | SessionStorage 恢复 | 页面刷新 | 之前保存的任意格式 | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/store/useFile.ts#L142-L154) |
| 6 | Widget iframe postMessage | 父页面 `postMessage({ json })` | JSON 字符串 | [widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/pages/widget.tsx#L57-L75) |

此外还有两个**外部集成**入口（直接调用 `jsoncrack-react` 包，绕过 `useFile`）：

| # | 入口 | 触发方式 | 格式 | 所在文件 |
|---|------|---------|------|---------|
| 7 | Chrome 扩展 content-script | 浏览器打开 JSON 响应 | JSON（`document.body.innerText`） | [content-script.tsx](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/chrome-extension/src/content-script.tsx#L71-L84) |
| 8 | VSCode 扩展 | VSCode 消息 `postMessage({ json })` | JSON 字符串 | [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/vscode/src/App.tsx#L27-L42) |

---

## 3. 统一解析链路——按代码执行顺序

### 阶段 A：输入汇聚到 `useFile.setContents()`

除 Chrome 扩展和 VSCode 扩展外，所有 Web 端入口最终都调用 `useFile` store 的 [`setContents()`](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/store/useFile.ts#L100-L126)：

```
入口 1 (TextEditor)    → onChange → setContents({ contents, skipUpdate: true })
入口 2 (Dropzone)      → onDrop   → setContents({ contents, format, hasChanges: false })
入口 3 (ImportModal)   → onDrop/URL → setContents({ contents }) 或 setContents + setFormat
入口 4 (URL查询参数)   → fetchUrl → setContents({ contents })
入口 5 (SessionStorage)→ checkEditorSession → setContents({ contents, hasChanges: false })
入口 6 (Widget iframe) → postMessage handler → setContents({ contents, hasChanges: false })
```

各入口调用 `setContents` 时携带的参数差异：

| 入口 | `contents` | `format` | `hasChanges` | `skipUpdate` |
|------|-----------|----------|-------------|-------------|
| TextEditor | 编辑器当前文本 | 不传（沿用当前） | 默认 `true` | `true`（关闭实时时可跳过） |
| Dropzone | 文件文本内容 | 文件扩展名 | `false` | 默认 `false` |
| ImportModal 文件 | 文件文本内容 | 先调 `setFormat` | 默认 `true` | 默认 `false` |
| ImportModal URL | `JSON.stringify(json, null, 2)` | 不传（默认 JSON） | 默认 `true` | 默认 `false` |
| URL查询参数 | `JSON.stringify(json, null, 2)` | 不传 | 默认 `true` | 默认 `false` |
| SessionStorage | 缓存内容 | 通过 `set({ format })` 设置 | `false` | 默认 `false` |
| Widget iframe | `event.data.json` | 不传 | `false` | 默认 `false` |

### 阶段 B：`setContents()` 内部——格式归一化到 JSON 对象

[useFile.ts#L100-L126](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/store/useFile.ts#L100-L126) 的核心流程：

```
setContents({ contents, hasChanges, skipUpdate, format })
  │
  ├─ 1. 更新 zustand state: { contents, error: null, hasChanges, format }
  │
  ├─ 2. contentToJson(contents, format)   ← 格式归一化的关键
  │     │
  │     ├─ format === "json" → jsonc-parser 的 parse()（容忍注释）
  │     ├─ format === "yaml" → js-yaml 的 load()
  │     ├─ format === "xml"  → fast-xml-parser 的 XMLParser
  │     └─ format === "csv"  → json-2-csv 的 csv2json()
  │
  │     返回：统一的 JavaScript object
  │
  ├─ 3. 可选：将 contents 存入 sessionStorage（条件：hasChanges && < 80KB && 非iframe && 非URL获取）
  │
  └─ 4. debouncedUpdateJson(json)  → useJson.setJson(JSON.stringify(json, null, 2))
```

#### `contentToJson()` 详细——四种格式的解析路径

定义在 [jsonAdapter.ts](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/lib/utils/jsonAdapter.ts#L4-L45)：

| 格式 | 解析库 | 解析代码 | 输出 |
|------|--------|---------|------|
| JSON | `jsonc-parser` `parse()` | `parse(value, errors)` | `object` |
| YAML | `js-yaml` `load()` | `load(value)` | `object` |
| XML  | `fast-xml-parser` `XMLParser` | `parser.parse(value)` | `object`（属性前缀 `$`） |
| CSV  | `json-2-csv` `csv2json()` | `csv2json(value, opts)` | `object[]` |

所有格式最终输出一个标准的 JavaScript 对象（或对象数组），完成**格式归一化**。

### 阶段 C：JSON 字符串流入 `useJson` store

[useJson.ts](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/store/useJson.ts#L16-L25) 极其简单：

```ts
setJson: json => {
  set({ json, loading: false });
}
```

它只存储格式化后的 JSON 字符串。通过 `debouncedUpdateJson`（400ms 防抖）写入，避免编辑器高频输入导致过度重渲染。

### 阶段 D：JSON 字符串流入 `<JSONCrack>` 组件

在 Web 端，`useJson.json` 被 GraphView 和 TreeView 消费：

#### GraphView 路径

[GraphView/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/features/editor/views/GraphView/index.tsx#L59) →
```tsx
const json = useJson(state => state.json);
// ...
<JSONCrack json={json} ... />
```

#### TreeView 路径

[TreeView/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/features/editor/views/TreeView/index.tsx#L10-L11) →
```tsx
const json = useJson(state => state.json);
// ...
<JSONTree data={JSON.parse(json)} ... />
```

### 阶段 E：`<JSONCrack>` 组件内部的统一处理

[JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L168-L205)

```
<JSONCrack json={...} />
  │
  ├─ 1. toJsonText(json)              ← 归一化输入为字符串
  │     ├─ json 是 string → 直接返回
  │     └─ json 是 object → JSON.stringify（WeakMap 缓存）
  │
  ├─ 2. parseJsonGraph(jsonText, maxRenderableNodes)
  │     │
  │     ├─ parseGraph(jsonText)        ← 核心解析（jsonc-parser 的 parseTree）
  │     │   │
  │     │   ├─ parseTree(json, errors) → AST
  │     │   └─ traverse(AST) → nodes[] + edges[]
  │     │
  │     ├─ 节点数 > maxRenderableNodes → 返回 { kind: "above-limit" }
  │     └─ 正常 → 返回 { kind: "ok", graph: { nodes, edges }, syntaxErrorCount }
  │
  └─ 3. setNodes / setEdges → 触发 reaflow Canvas 渲染
```

### 阶段 F：`parseGraph()`——JSON AST 到图数据的转换

[parser.ts](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/packages/jsoncrack-react/src/parser.ts#L9-L219)

```
parseGraph(json: string) → { nodes, edges, errors }
  │
  ├─ 1. parseTree(json, parseErrors)     → jsonc-parser 构建 AST
  │
  └─ 2. traverse(node, parentId?)        → 递归遍历 AST
        │
        ├─ 数组节点 → 创建 [N items] 文本，递归子元素
        ├─ 对象属性
        │   ├─ 值为数组 → 递归子元素，边标注 key
        │   ├─ 值为对象 → 递归子元素，边标注 key
        │   └─ 值为原始类型 → 直接记录 key-value
        ├─ 空对象（数组内）→ {0 keys}
        ├─ 原始值节点 → 叶子节点
        └─ 每个节点计算 parentKey / parentType / path
```

---

## 4. Chrome 扩展与 VSCode 扩展的特殊路径

这两个入口直接使用 `jsoncrack-react` 包，绕过了 `useFile` / `jsonAdapter` 层。

### Chrome 扩展

[content-script.tsx](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/chrome-extension/src/content-script.tsx#L71-L84)：

```
浏览器打开 JSON 响应页
  │
  ├─ getJsonSource()
  │   ├─ 检查 contentType 包含 "json"
  │   ├─ 读取 document.body.innerText
  │   └─ JSON.parse 校验有效性
  │
  └─ <JSONCrack json={parsedJson} />     ← 直接传 object（非 string）
       │
       └─ toJsonText(parsedJson)
            → WeakMap 缓存 → JSON.stringify → 进入 parseGraph
```

### VSCode 扩展

[App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/vscode/src/App.tsx#L27-L42)：

```
VSCode 扩展宿主发送 postMessage({ json: "..." })
  │
  ├─ window.addEventListener("message", onMessage)
  │   └─ setJson(jsonData)                 ← React state
  │
  └─ <JSONCrack json={json} />            ← 传入 string
       │
       └─ toJsonText(json) → 直接返回字符串 → 进入 parseGraph
```

---

## 5. 格式转换的双向管道

除了输入解析，[jsonAdapter.ts](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/lib/utils/jsonAdapter.ts) 还提供反向转换 `jsonToContent()`，用于格式切换：

```
setFormat(newFormat)                    [useFile.ts#L86-L98]
  │
  ├─ contentToJson(contents, oldFormat) → 中间 JSON 对象
  └─ jsonToContent(jsonStr, newFormat)  → 新格式字符串
       ├─ "json" → JSON.stringify(parsedJson, null, 2)
       ├─ "yaml" → js-yaml dump()
       ├─ "xml"  → fast-xml-parser XMLBuilder
       └─ "csv"  → json-2-csv json2csv()
```

这确保用户在 BottomBar 切换格式时，编辑器中的内容能在四种格式之间无损互转。

---

## 6. 完整数据流总图

```
┌─────────────────────────────────────────────────────────────┐
│                       输入入口层                             │
│                                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐  │
│  │ TextEditor│ │Dropzone  │ │ImportModal│ │URL Query Param│  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────┬────────┘  │
│       │            │            │               │           │
│  ┌────┴─────┐ ┌────┴──────────┐│          ┌────┴────────┐  │
│  │SessionStor│ │Widget postMsg ││          │ fetchUrl()  │  │
│  └────┬─────┘ └────┬──────────┘│          └──────┬──────┘  │
│       │            │           │                 │          │
└───────┼────────────┼───────────┼─────────────────┼──────────┘
        │            │           │                 │
        ▼            ▼           ▼                 ▼
┌───────────────────────────────────────────────────────────────┐
│                  useFile.setContents()                        │
│                  [useFile.ts#L100]                            │
│                                                               │
│   ① 更新 zustand state (contents, format, hasChanges)        │
│   ② contentToJson(contents, format)  ← 格式归一化            │
│      ├─ "json" → jsonc-parser parse()                        │
│      ├─ "yaml" → js-yaml load()                              │
│      ├─ "xml"  → fast-xml-parser XMLParser                   │
│      └─ "csv"  → json-2-csv csv2json()                       │
│   ③ 可选: sessionStorage 持久化                              │
│   ④ debouncedUpdateJson(obj) → useJson.setJson(str)          │
└───────────────────────────┬───────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│               useJson store (纯 JSON 字符串)                  │
│               [useJson.ts#L9-L25]                            │
└───────────┬─────────────────────────────────────┬─────────────┘
            │                                     │
            ▼                                     ▼
┌──────────────────────────┐      ┌───────────────────────────┐
│  GraphView (Graph 模式)   │      │  TreeView (Tree 模式)     │
│  <JSONCrack json={json}/>│      │  <JSONTree data=parse()/> │
└────────────┬─────────────┘      └───────────────────────────┘
             │
             ▼
┌───────────────────────────────────────────────────────────────┐
│           JSONCrack Component (jsoncrack-react)               │
│           [JSONCrackComponent.tsx#L168]                       │
│                                                               │
│   ① toJsonText(json) → 统一为 string                         │
│      ├─ string → 直接返回                                    │
│      └─ object → JSON.stringify (WeakMap 缓存)               │
│                                                               │
│   ② parseJsonGraph(jsonText, maxRenderableNodes)              │
│      └─ parseGraph(jsonText)                                  │
│          ├─ parseTree(json) → AST (jsonc-parser)             │
│          └─ traverse(AST) → nodes[] + edges[]                │
│                                                               │
│   ③ setNodes / setEdges → reaflow <Canvas> 渲染              │
└───────────────────────────────────────────────────────────────┘
```

---

## 7. 关键设计要点

1. **格式适配层前置**：`contentToJson()` 在 `setContents()` 中率先将任意格式转为 JS 对象，后续链路只需处理 JSON。
2. **双类型输入兼容**：`JSONCrack` 组件的 `json` prop 接受 `string | object`（[canvasHelpers.ts#L8](file:///d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L8)），`toJsonText()` 内部做归一化。
3. **容错 JSON 解析**：核心 `parseGraph()` 使用 `jsonc-parser` 的 `parseTree()`，容忍 JSON 中的注释和尾逗号。
4. **防抖更新**：编辑器输入经 400ms 防抖后写入 `useJson`，避免频繁重绘。
5. **会话恢复**：`sessionStorage` 同时保存 `contents`（原始格式文本）和 `format`，刷新后能正确恢复并重新走解析链路。
6. **外部集成直通**：Chrome 扩展和 VSCode 扩展跳过 `useFile`/`jsonAdapter`，直接将 JSON 传入 `<JSONCrack>`，由 `toJsonText()` + `parseGraph()` 处理。
