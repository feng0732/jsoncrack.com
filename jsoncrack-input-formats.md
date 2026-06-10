# JSON Crack 输入格式多样性与统一解析链路分析

> **代码定位约定**：本文档所有源码引用采用「仓库相对路径 + 行号」形式，仓库根为 `185-jsoncrack.com/`；对应绝对路径可拼接为 `<repoRoot>/<relativePath>`。
> 例如 `apps/www/src/store/useFile.ts#L100-L126` 在本机对应
> `d:/fz/0601/solo-dogfeeding/code/185-jsoncrack.com/apps/www/src/store/useFile.ts#L100-L126`。

本文档按代码调用顺序，追踪不同来源、不同格式的输入如何汇聚到同一条解析链路，最终被渲染为可视化图。

---

## 0. 仓库结构速览

```
185-jsoncrack.com/
├── apps/
│   ├── www/                     # Web 主程序 (Next.js)
│   │   └── src/
│   │       ├── enums/file.enum.ts              ← FileFormat 枚举
│   │       ├── store/                          ← zustand stores
│   │       │   ├── useFile.ts                  ← 文件/内容中枢
│   │       │   ├── useJson.ts                  ← 归一化 JSON store
│   │       │   └── useConfig.ts
│   │       ├── lib/utils/
│   │       │   └── jsonAdapter.ts              ← 格式转换双向管道
│   │       ├── features/
│   │       │   └── editor/
│   │       │       ├── TextEditor.tsx          ← Monaco 编辑器
│   │       │       ├── LiveEditor.tsx
│   │       │       ├── BottomBar.tsx           ← 格式切换/实时开关
│   │       │       ├── FullscreenDropzone.tsx  ← 拖拽上传
│   │       │       └── views/
│   │       │           ├── GraphView/index.tsx ← 图视图入口
│   │       │           ├── TreeView/index.tsx  ← 树视图入口
│   │       │           └── GraphView/stores/useGraph.ts
│   │       ├── pages/
│   │       │   ├── editor.tsx                  ← 编辑器页面
│   │       │   └── widget.tsx                  ← Widget 嵌入页
│   │       └── features/modals/ImportModal/index.tsx
│   ├── vscode/                  # VSCode 扩展
│   │   ├── ext-src/
│   │   │   ├── extension.ts     ← 扩展宿主（活动文档/选中文本/指定内容）
│   │   │   └── webview.ts       ← Webview 面板构建
│   │   └── src/App.tsx          ← Webview 端 React
│   └── chrome-extension/        # Chrome 扩展
│       ├── public/manifest.json
│       └── src/content-script.tsx ← 页面内容脚本
├── packages/
│   └── jsoncrack-react/         # 独立可视化 React 包
│       ├── src/
│       │   ├── index.ts
│       │   ├── parser.ts                 ← 核心 parseGraph
│       │   ├── canvasHelpers.ts          ← toJsonText / parseJsonGraph
│       │   ├── JSONCrackComponent.tsx    ← <JSONCrack> 组件
│       │   └── types.ts
│       └── src/__tests__/parser.test.ts  ← 解析器单元测试
└── package.json
```

---

## 1. 支持的输入格式枚举

所有支持的文件格式定义在 `apps/www/src/enums/file.enum.ts#L1-L6`：

```ts
export enum FileFormat {
  "JSON" = "json",
  "YAML" = "yaml",
  "XML"  = "xml",
  "CSV"  = "csv",
}
```

---

## 2. 输入入口全景（8 种来源 + 3 种扩展命令）

### 2.1 Web 主程序入口（6 种）

| # | 入口 | 触发方式 | 格式 | 所在文件 |
|---|------|---------|------|---------|
| 1 | Monaco 编辑器实时输入 | 键盘编辑 | JSON / YAML / XML / CSV | `apps/www/src/features/editor/TextEditor.tsx#L90` |
| 2 | 拖拽文件到全屏 Dropzone | 拖拽 `.json/.yaml/.xml/.csv` | 按文件扩展名推断 | `apps/www/src/features/editor/FullscreenDropzone.tsx#L17-L27` |
| 3 | ImportModal（URL 获取 / 文件上传） | 菜单 → Import | URL 返回 JSON；文件按扩展名推断 | `apps/www/src/features/modals/ImportModal/index.tsx#L18-L47` |
| 4 | URL 查询参数 `?json=URL` | 编辑器页面加载时 | 远程 JSON | `apps/www/src/store/useFile.ts#L129-L141` → `apps/www/src/pages/editor.tsx#L116-L117` |
| 5 | SessionStorage 恢复 | 页面刷新 | 之前保存的任意格式 | `apps/www/src/store/useFile.ts#L142-L154` |
| 6 | Widget iframe postMessage | 父页面 `postMessage({ json })` | JSON 字符串 | `apps/www/src/pages/widget.tsx#L57-L75` |

### 2.2 VSCode 扩展入口（3 条命令，三者发送策略完全不同）

扩展在 `apps/vscode/package.json#L29-L76` 注册了 3 个命令，全部在 `apps/vscode/ext-src/extension.ts#L12-L23` 的 `activate()` 中绑定。**三条命令的发送时机、握手、变更监听策略各不相同，不得混淆**：

| 命令 ID | 菜单位置 | 含义 | 对应函数 | 核心差异 |
|---------|---------|------|---------|---------|
| `jsoncrack-vscode.start` | 编辑器标题栏（仅 `.json` / langId=json） | 可视化**整个活动文档** | `createWebviewForActiveEditor()` `apps/vscode/ext-src/extension.ts#L67-L93` | 仅 ready 握手推送 + 实时同步**完整文档**，无创建后即时发送 |
| `jsoncrack-vscode.start.selected` | 右键上下文菜单（`editorHasSelection`） | 可视化**选中的文本片段** | `createWebviewForSelectedText()` `apps/vscode/ext-src/extension.ts#L27-L65` | 创建后**立即**发 + ready 补发 + 实时同步**当前选区切片**，**有前置校验** |
| `jsoncrack-vscode.start.specific` | 仅命令面板（其他扩展编程调用） | 可视化**编程传入的指定字符串** | `createWebviewForContent()` `apps/vscode/ext-src/extension.ts#L100-L110` | **仅创建后发一次**，无 ready 握手，无变更监听，**纯一次性展示** |

### 2.3 Chrome 扩展入口

| 入口 | 触发方式 | 格式 | 所在文件 |
|------|---------|------|---------|
| content-script | 浏览任意 `contentType` 含 `json` 的页面，`document_idle` 时自动注入 | JSON 字符串（从 `document.body.innerText` / `textContent` 提取） | `apps/chrome-extension/public/manifest.json#L24-L29` → `apps/chrome-extension/src/content-script.tsx#L52-L84` |

### 2.4 外部集成概览表（Chrome / VSCode 直接调用 `jsoncrack-react`）

| # | 入口 | 触发方式 | 格式 | 所在文件 |
|---|------|---------|------|---------|
| 7 | Chrome 扩展 content-script | 浏览器打开 JSON 响应 | JSON（`document.body.innerText`） | `apps/chrome-extension/src/content-script.tsx#L71-L84` |
| 8 | VSCode 扩展 Webview | 宿主 `postMessage({ json })` | JSON 字符串 | `apps/vscode/src/App.tsx#L27-L42` |

---

## 3. 统一解析链路——按代码执行顺序

### 阶段 A：输入汇聚到 `useFile.setContents()`

除 Chrome 扩展和 VSCode 扩展外，所有 Web 端入口最终都调用 `apps/www/src/store/useFile.ts#L100-L126` 中的 `setContents()`：

```
入口 1 (TextEditor)    → onChange
  → setContents({ contents, skipUpdate: true })
入口 2 (Dropzone)      → onDrop
  → setContents({ contents, format, hasChanges: false })
入口 3 (ImportModal 文件)   → file.text().then(text)
  → setFormat(format as FileFormat) → setContents({ contents: text })
入口 3 (ImportModal URL)    → fetch().then(res.json())
  → setContents({ contents: JSON.stringify(json, null, 2) })
入口 4 (URL查询参数)   → fetchUrl()
  → setContents({ contents: JSON.stringify(json, null, 2) })
入口 5 (SessionStorage)→ checkEditorSession()
  → set({ format }) → setContents({ contents, hasChanges: false })
入口 6 (Widget iframe) → postMessage handler
  → setContents({ contents: event.data.json, hasChanges: false })
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

`apps/www/src/store/useFile.ts#L100-L126` 的核心流程：

```
setContents({ contents, hasChanges, skipUpdate, format })
  │
  ├─ 1. 更新 zustand state
  │     set({
  │       ...(contents && { contents }),
  │       error: null,
  │       hasChanges,
  │       format: format ?? get().format,
  │     })
  │
  ├─ 2. contentToJson(get().contents, get().format)   ← 格式归一化的关键
  │     │  (定义见 apps/www/src/lib/utils/jsonAdapter.ts#L4-L45)
  │     │
  │     ├─ format === "json" → jsonc-parser 的 parse()（容忍注释；若有错误回退 JSON.parse）
  │     ├─ format === "yaml" → js-yaml 的 load()
  │     ├─ format === "xml"  → fast-xml-parser 的 XMLParser
  │     │                        (attributeNamePrefix="$", ignoreAttributes=false)
  │     └─ format === "csv"  → json-2-csv 的 csv2json()
  │
  │     返回：统一的 JavaScript object / object[]
  │
  ├─ 3. 条件持久化 sessionStorage
  │     if (hasChanges && contents.length < 80_000 && !isIframe() && !isFetchURL)
  │       sessionStorage.setItem("content", contents)
  │       sessionStorage.setItem("format", get().format)
  │
  └─ 4. debouncedUpdateJson(json)   ← lodash.debounce 400ms
        → useJson.getState().setJson(JSON.stringify(json, null, 2))
```

#### `contentToJson()` 详细——四种格式的解析路径

定义在 `apps/www/src/lib/utils/jsonAdapter.ts#L4-L45`：

| 格式 | 解析库 | 解析代码位置 | 输出 |
|------|--------|-------------|------|
| JSON | `jsonc-parser` `parse()` | `apps/www/src/lib/utils/jsonAdapter.ts#L7-L13` | `object` |
| YAML | `js-yaml` `load()` | `apps/www/src/lib/utils/jsonAdapter.ts#L15-L18` | `object` |
| XML  | `fast-xml-parser` `XMLParser` | `apps/www/src/lib/utils/jsonAdapter.ts#L20-L31` | `object`（属性前缀 `$`） |
| CSV  | `json-2-csv` `csv2json()` | `apps/www/src/lib/utils/jsonAdapter.ts#L33-L42` | `object[]` |

所有格式最终输出一个标准的 JavaScript 对象（或对象数组），完成**格式归一化**。

### 阶段 C：JSON 字符串流入 `useJson` store

`apps/www/src/store/useJson.ts#L9-L25` 极其简单：

```ts
const initialStates = { json: "{}", loading: true };
// ...
setJson: json => {
  set({ json, loading: false });
}
```

它只存储格式化后的 JSON 字符串。通过 `debouncedUpdateJson`（400ms 防抖，`apps/www/src/store/useFile.ts#L67-L69`）写入，避免编辑器高频输入导致过度重渲染。

### 阶段 D：JSON 字符串流入 `<JSONCrack>` 组件

在 Web 端，`useJson.json` 被 GraphView 和 TreeView 消费：

#### GraphView 路径

`apps/www/src/features/editor/views/GraphView/index.tsx#L59` →
```tsx
const json = useJson(state => state.json);
// ...
<JSONCrack json={json} ... />
```

#### TreeView 路径

`apps/www/src/features/editor/views/TreeView/index.tsx#L10-L16` →
```tsx
const json = useJson(state => state.json);
// ...
<JSONTree data={JSON.parse(json)} ... />
```

### 阶段 E：`<JSONCrack>` 组件内部的统一处理

`packages/jsoncrack-react/src/JSONCrackComponent.tsx#L168-L205`

```
<JSONCrack json={...} />
  │
  ├─ 1. toJsonText(json)              ← 归一化输入为字符串
  │     (packages/jsoncrack-react/src/canvasHelpers.ts#L13-L26)
  │     ├─ json 是 string → 直接返回
  │     └─ json 是 object → JSON.stringify(json, null, 2)
  │                              使用 WeakMap<object, string> 缓存同引用结果
  │
  ├─ 2. parseJsonGraph(jsonText, maxRenderableNodes)
  │     (packages/jsoncrack-react/src/canvasHelpers.ts#L74-L90)
  │     │
  │     ├─ try { graph = parseGraph(jsonText) }
  │     │      packages/jsoncrack-react/src/parser.ts#L9-L219
  │     │
  │     ├─ graph.nodes.length > maxRenderableNodes
  │     │   → return { kind: "above-limit", total }
  │     │
  │     └─ return { kind: "ok", graph, syntaxErrorCount: graph.errors.length }
  │
  └─ 3. setNodes / setEdges → 触发 reaflow <Canvas> 渲染
        (packages/jsoncrack-react/src/JSONCrackComponent.tsx#L193-L205)
```

### 阶段 F：`parseGraph()`——JSON AST 到图数据的转换

`packages/jsoncrack-react/src/parser.ts#L9-L219`

```
parseGraph(json: string) → ParseGraphResult (extends GraphData + { errors })
  │
  ├─ 1. parseTree(json, parseErrors)     → jsonc-parser 构建 AST
  │     (packages/jsoncrack-react/src/parser.ts#L11)
  │     若 parseTree 返回 null → 返回 { nodes: [], edges: [], errors }
  │
  └─ 2. traverse(node: Node, parentId?: string)  递归遍历 AST
        (packages/jsoncrack-react/src/parser.ts#L26-L211)
        │
        ├─ 数组节点 + 根级/父为数组
        │   → 文本 `[N items]`，然后 forEach child → traverse(child, id)
        │
        ├─ 对象属性（child.children && child.children[1] 有值）
        │   ├─ key   = child.children[0].value
        │   ├─ value = child.children[1]  (valueNode)
        │   │
        │   ├─ valueNode.type === "array"
        │   │   → forEach arrayChild → traverse(arrayChild, undefined) 收集 targetIds
        │   │   → 边: { from: id, to: targetIds[i], text: key }
        │   │
        │   ├─ valueNode.type === "object"
        │   │   → objectNodeId = traverse(valueNode, id)
        │   │   → 边: { from: id, to: objectNodeId, text: key }
        │   │
        │   └─ valueNode.type === string/number/boolean/null
        │       → 行文本直接记录（无子节点、无连线）
        │
        ├─ 特例：数组中的空对象
        │   → 文本 `{0 keys}`
        │
        ├─ 叶子节点（text.length === 0 且 node.value !== undefined）
        │   → 单节点，value 即自身
        │
        └─ 每个节点附加
            - path: jsonc-parser.getNodePath(node)
            - parentKey / parentType: appendParentKey() 上溯推导
```

该解析器的行为可在单元测试 `packages/jsoncrack-react/src/__tests__/parser.test.ts` 中验证：
- 空字符串 → 空图（`L5-L9`）
- 基本类型 → 1 个节点（`L11-L16`）
- 扁平对象 → 1 节点多行文本（`L18-L30`）
- 嵌套对象 → 2 节点 + 1 条边，`edge.text === 属性名`（`L32-L43`）
- 数组 → 父节点 + N 个子节点 + N 条边（`L45-L86`）
- 错误容忍：语法错误仍返回部分树，`errors.length > 0`（`L95-L99`）

---

## 4. 编辑器扩展宿主到可视化页面的完整通信链路

### 4.1 VSCode 扩展：三条命令 → 活动文档 / 选中文本 / 指定内容

扩展分**宿主进程侧**（`ext-src/extension.ts` + `ext-src/webview.ts`）与**Webview 渲染侧**（`src/App.tsx`），两边通过 `postMessage` / `onDidReceiveMessage` 双向通信。

#### 4.1.1 命令注册与激活

`apps/vscode/package.json#L29-L76` 定义 activationEvents 和 3 个命令；
`apps/vscode/ext-src/extension.ts#L12-L23` 在 activate() 中注册：

```ts
vscode.commands.registerCommand("jsoncrack-vscode.start",            () => createWebviewForActiveEditor(context))
vscode.commands.registerCommand("jsoncrack-vscode.start.specific",   (content?: string) => createWebviewForContent(context, content))
vscode.commands.registerCommand("jsoncrack-vscode.start.selected",   () => createWebviewForSelectedText(context))
```

#### 4.1.2 Webview 面板构建

`apps/vscode/ext-src/webview.ts#L4-L51` 构造 `createWebviewPanel()`：

```
vscode.window.createWebviewPanel("liveHTMLPreviewer", title, ViewColumn.Beside, {
  enableScripts: true,
  retainContextWhenHidden: true,            ← 切走 Tab 后保留 React 状态
  localResourceRoots: [build/webview, assets]
})
  │
  ├─ asWebviewUri() 转换 build/webview/index.js 和 index.css
  ├─ 注入带 CSP 的 HTML（script-src 含 'unsafe-eval'，worker-src 支持 ELK）
  └─ 返回 panel 实例，调用方通过 panel.webview.postMessage 传数据
```

#### 4.1.3 命令 1：`jsoncrack-vscode.start` — 可视化**整个活动文档**
**代码位置**：`apps/vscode/ext-src/extension.ts#L67-L93`

> **独特策略**：**只做 ready 握手 + 实时同步完整文档**，不做面板创建后的即时推送。
>
> 这是三者中**唯一**在面板创建后不立即发消息的命令。它完全依赖 Webview 侧的 `ready` 回包来触发首屏数据。

```
createWebviewForActiveEditor(context)
  │
  ├─ 数据获取
  │    editor = vscode.window.activeTextEditor
  │    → 数据范围 = editor.document.getText()   ← 整个文档全文
  │
  ├─ panel = createWebviewPanel(context, title)
  │
  ├─ ① 发送时机：**仅 ready 握手触发**（L71-L77）
  │    panel.webview.onDidReceiveMessage(e)
  │      if e === "ready"
  │        → panel.webview.postMessage({
  │             json: editor?.document.getText()     ← 每次都重新拉取全文
  │           })
  │
  ├─ ② 变更监听：**有，完整文档同步**（L79-L85）
  │    vscode.workspace.onDidChangeTextDocument(changeEvent)
  │      if changeEvent.document === editor.document
  │        → panel.webview.postMessage({
  │             json: changeEvent.document.getText()  ← 变更后推送全文
  │           })
  │
  └─ ③ 监听器：onReceiveMessage + onTextChange，共 2 个（L87-L92）
      panel.onDidDispose → dispose 两个监听器
```

**无**：创建后即时发送、前置校验。

---

#### 4.1.4 命令 2：`jsoncrack-vscode.start.selected` — 可视化**选中的文本片段**
**代码位置**：`apps/vscode/ext-src/extension.ts#L27-L65`

> **独特策略**：**创建后立即发送 + ready 补发 + 变更时重取选区切片**，且**有前置校验**。
>
> 这是三者中**唯一**有前置校验的命令，也是**唯一**在面板创建后立刻发一次消息的命令。变更时也不是推送全文，而是**重新对当前选区切片**。

```
createWebviewForSelectedText(context)
  │
  ├─ 前置校验（L30-L33）—— 三者中【唯一】有
  │    if editor.selection.isEmpty
  │      → vscode.window.showInformationMessage("Please select some text first!")
  │      → return  （不创建面板）
  │
  ├─ 数据获取
  │    selectedText = editor.document.getText(editor.selection)
  │    → 数据范围 = 当前选中的文本片段
  │
  ├─ panel = createWebviewPanel(...)
  │
  ├─ ① 发送时机 1：**创建后立即发一次**（L39-L41）—— 三者中【唯一】
  │    panel.webview.postMessage({ json: selectedText })
  │
  ├─ ② 发送时机 2：**ready 握手补发**（L43-L49）
  │    panel.webview.onDidReceiveMessage(e)
  │      if e === "ready"
  │        → panel.webview.postMessage({ json: selectedText })
  │
  ├─ ③ 变更监听：**有，但重新对选区切片**（L51-L57）
  │    vscode.workspace.onDidChangeTextDocument(changeEvent)
  │      if changeEvent.document === editor?.document
  │        → panel.webview.postMessage({
  │             json: changeEvent.document.getText(editor?.selection)
  │                                                    ↑ 不是全文！是当前选区
  │           })
  │
  └─ ④ 监听器：onReceiveMessage + onTextChange，共 2 个（L59-L64）
      panel.onDidDispose → dispose 两个监听器
```

**无**：文档级别全文同步（始终走选区切片）。

---

#### 4.1.5 命令 3：`jsoncrack-vscode.start.specific` — 可视化**编程传入的指定字符串**
**代码位置**：`apps/vscode/ext-src/extension.ts#L100-L110`

> **独特策略**：**只在创建后发一次**，无 ready 握手，无变更监听，**纯一次性展示**。
>
> 这是三者中**唯一**没有任何监听器的命令。它面向其他扩展的编程调用，假定调用方自己管理内容同步。

```ts
/**
 * Renders a readonly diagram from a string
 * @param context ExtensionContext
 * @param content JSON content as a string
 */
function createWebviewForContent(context?: ExtensionContext, content?: string): any {
  if (context && content) {
    const panel = createWebviewPanel(
      context,
      getPanelTitle(vscode.window.activeTextEditor?.document)
    );

    // ── ① 发送时机：创建后发一次，且仅此一次 ──
    panel.webview.postMessage({ json: content });

    // ── ② 无 ready 握手监听 ──
    // ── ③ 无 onDidChangeTextDocument 变更监听 ──
    // ── ④ 无 onDidDispose 清理（无监听器可 dispose）──
  }
}
```

**无**：ready 握手、变更监听、前置校验。

---

#### 4.1.6 Webview 侧（React App）收消息并接入统一链路

**代码位置**：`apps/vscode/src/App.tsx#L21-L56`

> 无论三条命令发送策略如何不同，Webview 侧的接收逻辑是**同一套**：只要收到 `event.data.json` 为字符串，就更新 state 并驱动 `<JSONCrack>` 渲染。

```
<App />
  │
  ├─ 初始 state: json = "{}"
  │
  ├─ useEffect: 建立通道（L26-L41）
  │   vscode = window.acquireVsCodeApi?.()
  │   vscode.postMessage("ready")                       ← 告诉宿主「我已就绪」
  │   window.addEventListener("message", onMessage)
  │     onMessage(event)
  │       if (typeof event.data?.json === "string")
  │         setJson(event.data.json)                      ← 写入 React state
  │
  └─ 渲染 <JSONCrack json={json} theme={getTheme()} ... />  （L56）
       │
       └─ 与 Web 主程序完全相同的统一链路：
          toJsonText → parseJsonGraph → parseGraph → 渲染
```

---

#### 4.1.7 三条命令发送策略对比总表

| 对比维度 | `createWebviewForActiveEditor` (`.start`) | `createWebviewForSelectedText` (`.start.selected`) | `createWebviewForContent` (`.start.specific`) |
|---------|-------------------------------------------|---------------------------------------------------|----------------------------------------------|
| **代码位置** | `apps/vscode/ext-src/extension.ts#L67-L93` | `apps/vscode/ext-src/extension.ts#L27-L65` | `apps/vscode/ext-src/extension.ts#L100-L110` |
| **触发入口** | 编辑器标题栏图标（仅 `.json` / langId=json） | 右键上下文菜单（`editorHasSelection`） | 仅命令面板，供其他扩展编程调用 |
| **数据范围** | `editor.document.getText()` — 完整文档 | `document.getText(editor.selection)` — 当前选区 | 函数参数 `content` — 调用方传入 |
| **前置校验** | ❌ 无 | ✅ 有（`selection.isEmpty` 则提示并 return） | ❌ 无（仅检查 `context && content`） |
| **面板创建后立即发送** | ❌ 无 | ✅ 有（`panel.postMessage({ json: selectedText })` L39-L41） | ✅ 有（且仅此一次，L106-L108） |
| **ready 握手** | ✅ 有（L71-L77） | ✅ 有（L43-L49） | ❌ 无 |
| **变更监听** | ✅ 有，同步**完整文档**（L79-L85） | ✅ 有，同步**当前选区切片**（L51-L57） | ❌ 无 |
| **监听器数量** | 2 个（onReceiveMessage + onTextChange） | 2 个（onReceiveMessage + onTextChange） | 0 个 |
| **发送时机数量** | 2 次（ready + 每次变更） | 3 次（创建后 + ready + 每次变更） | 1 次（仅创建后） |
| **适用场景** | 打开整个 JSON 文件实时联动 | 临时查看某段 JSON 片段 | 其他扩展一次性传入内容展示 |

### 4.2 Chrome 扩展：JSON 响应页面自动捕获 + Raw/Graph 切换

#### 4.2.1 注入时机与匹配

`apps/chrome-extension/public/manifest.json#L24-L29`：

```json
"content_scripts": [
  {
    "matches": ["<all_urls>"],
    "js": ["content-script.js"],
    "run_at": "document_idle"
  }
]
```

→ 所有页面在 `document_idle` 时注入，实际是否生效由 `getJsonSource()` 判断。

#### 4.2.2 JSON 源获取

`apps/chrome-extension/src/content-script.tsx#L52-L84`（IIFE 启动）：

```
(() => {
  if (window.top !== window) return;           // 不注入 iframe 内
  if (document.getElementById(TOGGLE_ID)) return; // 避免重复注入

  const source = getJsonSource();
  if (!source) return;
  injectStyles();
  injectToggle(source);                         // 同时启动 Raw/Graph 切换
})();
```

`getJsonSource()`（`L71-L84`）：

```ts
function getJsonSource() {
  const contentType = (document.contentType || "").toLowerCase();
  if (!contentType.includes("json")) return null;              // content-type 过滤

  const raw = (document.body.innerText || document.body.textContent || "").trim();
  if (!raw) return null;

  try { JSON.parse(raw); return raw; } catch { return null; }   // 解析合法性校验
}
```

→ 三层过滤：content-type 含 json → 取 body 全文 → `JSON.parse` 校验，最终返回合法 JSON 字符串。

#### 4.2.3 切换到 Graph 模式并接入统一链路

`apps/chrome-extension/src/content-script.tsx#L300-L363` 的 `GraphView` 组件：

```
GraphView({ rawJson })
  │
  ├─ ① 懒加载 JSONCrack 组件（动态 import 含 Worker 临时屏蔽）
  │    loadJsonCrackComponent() → 临时 shadow globalThis.Worker
  │      → 导入 jsoncrack-react → 恢复 Worker（应对严格 CSP）
  │
  ├─ ② 预先 parse 一次供错误提示
  │    useMemo(() => try { JSON.parse(rawJson) } catch { null })
  │
  └─ ③ 渲染
       <JSONCrackComponent
         json={parsedJson}     ← 传 object（非 string），WeakMap 缓存命中
         theme={useSystemTheme()}
         showControls showGrid centerOnLayout
         onNodeClick={...}
       />
       │
       └─ toJsonText(object) → JSON.stringify → parseJsonGraph → parseGraph → 渲染
```

---

## 5. Chrome 扩展与 VSCode 扩展链路汇总（含 VSCode 三命令差异）

### 5.1 VSCode 三条命令各自的独立链路

```
┌────────────────────────────────────────────────────────────────────────────┐
│  VSCode 扩展宿主 (apps/vscode/ext-src/extension.ts)                         │
│                                                                            │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ jsoncrack-vscode.start (L67-L93) — 整个活动文档                      │  │
│  │                                                                    │  │
│  │  • 数据: editor.document.getText()  [完整文档]                      │  │
│  │  • 发送时机: 仅 ready 握手触发 + 每次变更全文推送                    │  │
│  │  • ✅ ready 握手    ✅ 变更监听(全文)    ❌ 创建后立即发  ❌ 前置校验 │  │
│  └──────────────────────────────┬───────────────────────────────────────┘  │
│                                 │ postMessage({ json: fullText })         │
│  ┌──────────────────────────────────┼──────────────────────────────────────┐  │
│  │ jsoncrack-vscode.start.selected (L27-L65) — 选中文本                 │  │
│  │                                    │                                 │  │
│  │  • 前置校验: selection.isEmpty → 提示 return  [三者唯一]              │  │
│  │  • 数据: document.getText(editor.selection)  [选区切片]               │  │
│  │  • 发送时机: 创建后立即发 + ready 补发 + 每次变更重取切片 [三者唯一]   │  │
│  │  • ✅ ready 握手    ✅ 变更监听(选区)    ✅ 创建后立即发  ✅ 前置校验   │  │
│  └──────────────────────────────┬───────────────────────────────────────┘  │
│                                 │ postMessage({ json: selectedText })     │
│  ┌──────────────────────────────────┼──────────────────────────────────────┐  │
│  │ jsoncrack-vscode.start.specific (L100-L110) — 指定内容               │  │
│  │                                    │                                 │  │
│  │  • 数据: 函数参数 content  [调用方传入]                               │  │
│  │  • 发送时机: 仅创建后发一次  [三者唯一]                               │  │
│  │  • ❌ ready 握手    ❌ 变更监听        ✅ 创建后立即发  ❌ 前置校验   │  │
│  └──────────────────────────────┬───────────────────────────────────────┘  │
│                                 │ postMessage({ json: content })          │
└─────────────────────────────────┼──────────────────────────────────────────┘
                                  │
                                  ▼
                       createWebviewPanel(webview.ts)
                         • 构造带 CSP 的 HTML
                         • 加载 index.js (React App)
                                  │
                                  ▼
                  ┌──────────────────────────────────────┐
                  │  VSCode Webview 端 (src/App.tsx)     │
                  │  【接收逻辑三者共用同一套】          │
                  │                                      │
                  │  useEffect:                          │
                  │    vscode.postMessage("ready")       │
                  │    window.onmessage →                │
                  │      if (data.json is string)        │
                  │        setJson(data.json)            │
                  │                                      │
                  │  <JSONCrack json={json} />           │
                  └──────────────────┬───────────────────┘
                                     │
                                     ▼
                          统一链路 (packages/jsoncrack-react)
                            toJsonText → parseJsonGraph → parseGraph → reaflow Canvas
```

### 5.2 Chrome 扩展链路

```
Chrome 扩展宿主 (apps/chrome-extension/src/content-script.tsx)
│
├─ manifest (public/manifest.json):
│     matches: "<all_urls>"
│     run_at: "document_idle"
│
├─ IIFE 启动 (L52-L64)
│     if (window.top !== window) return          // 不注入 iframe
│     if (document.getElementById(TOGGLE_ID)) return  // 不重复注入
│
├─ getJsonSource() (L71-L84) — 三层过滤
│     ① contentType 含 "json"?
│     ② body.innerText 非空?
│     ③ JSON.parse() 解析合法?
│     → 返回合法 JSON 字符串 rawJson
│
├─ injectStyles()  injectToggle(rawJson)
│     → Raw / Graph 切换按钮注入页面
│
└─ 用户点击 "Graph" → GraphView({ rawJson }) (L300-L363)
       │
       ├─ loadJsonCrackComponent() (L28-L50)
       │    → 临时 shadow Worker → 导入 jsoncrack-react → 恢复 Worker
       │       [规避严格 CSP]
       │
       ├─ parsedJson = JSON.parse(rawJson)
       │
       └─ <JSONCrackComponent json={parsedJson} />   ← object 类型
            │
            └─ toJsonText(parsedJson)
                 → WeakMap 缓存命中 → JSON.stringify
                 → parseJsonGraph → parseGraph → 渲染
```

---

## 6. 格式转换的双向管道

除了输入解析，`apps/www/src/lib/utils/jsonAdapter.ts` 还提供反向转换 `jsonToContent()`（`L47-L93`），用于格式切换（BottomBar 中的格式菜单，见 `apps/www/src/features/editor/BottomBar.tsx#L151-L171`）：

```
setFormat(newFormat)                    apps/www/src/store/useFile.ts#L86-L98
  │
  ├─ prevFormat = get().format
  ├─ set({ format: newFormat })
  ├─ contentJson = contentToJson(contents, prevFormat)    ← 旧格式 → 中间 JSON 对象
  └─ jsonContent = jsonToContent(JSON.stringify(contentJson, null, 2), newFormat)
       │
       ├─ "json" → JSON.stringify(JSON.parse(json), null, 2)
       ├─ "yaml" → js-yaml dump(parse(json))
       ├─ "xml"  → fast-xml-parser XMLBuilder
       └─ "csv"  → json-2-csv json2csv(确保数组，expand 嵌套/数组)
  │
  └─ setContents({ contents: jsonContent })  ← 新格式文本重新走解析链路
```

这确保用户在 BottomBar 切换格式时，编辑器中的内容能在四种格式之间无损互转。

---

## 7. 完整数据流总图（Web 主程序 + 扩展）

```
                         ┌─────────────────────────────────────────────┐
                         │           外部扩展宿主环境                   │
                         │                                             │
                         │  ┌──────────────────────────────────────┐  │
                         │  │ VSCode Extension Host (ext-src/)      │  │
                         │  │  .start   → active editor full text  │  │
                         │  │             [ready 握手 + 全文同步]   │  │
                         │  │  .selected→ selection slice          │  │
                         │  │             [前置校验 + 创建后立即发 + │  │
                         │  │              ready 补发 + 选区重切片] │  │
                         │  │  .specific→ caller-provided content  │  │
                         │  │             [仅创建后发一次，无监听] │  │
                         │  └──────────┬───────────────────────────┘  │
                         │             │ postMessage({ json })        │
                         │  ┌──────────┴───────────────────────────┐  │
                         │  │ Chrome content-script                │  │
                         │  │  contentType + JSON.parse 校验       │  │
                         │  │  getJsonSource() → rawJson string    │  │
                         │  └──────────┬───────────────────────────┘  │
                         └─────────────┼──────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       输入入口层（Web 主程序 6 种 + 扩展 Webview 2 种）              │
│                                                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐ ┌──────────────┐        │
│  │TextEditor│ │Dropzone  │ │ImportModal│ │URL Query Param│ │SessionStorage│        │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────┬────────┘ └──────┬───────┘        │
│       │            │            │               │                 │                │
│  ┌────┴────────────┴────────────┴───────────────┴─────────────────┴───────────┐    │
│  │ Widget iframe postMessage (pages/widget.tsx)                               │    │
│  └─────────────────────────────────────┬──────────────────────────────────────┘    │
│                                        │                                           │
│  ┌─────────────────────────────────────┴──────────────────────────────────────┐    │
│  │ VSCode Webview App (src/App.tsx)                                           │    │
│  │   window.onmessage → setJson → <JSONCrack json={str}/>                     │    │
│  │                                                                             │    │
│  │ Chrome GraphView (content-script.tsx)                                      │    │
│  │   parsedJson = JSON.parse(rawJson) → <JSONCrack json={obj}/>               │    │
│  └─────────────────────────────────────┬──────────────────────────────────────┘    │
└────────────────────────────────────────┼──────────────────────────────────────────┘
                                         │
          ┌──────────────────────────────┴─────────────────────────────────────┐
          │   Web 主程序统一入口：useFile.setContents()  (store/useFile.ts)     │
          │                                                                     │
          │  ① zustand state = { contents, error: null, hasChanges, format }  │
          │  ② contentToJson(contents, format)  ← 格式归一化                   │
          │      ├─ "json" → jsonc-parser parse()  (L7-13 jsonAdapter.ts)     │
          │      ├─ "yaml" → js-yaml load()            (L15-18)               │
          │      ├─ "xml"  → fast-xml-parser XMLParser  (L20-31)              │
          │      └─ "csv"  → json-2-csv csv2json()     (L33-42)               │
          │  ③ 可选 sessionStorage (条件: hasChanges && <80KB && !iframe)     │
          │  ④ debouncedUpdateJson(obj) → useJson.setJson(str)  400ms 防抖     │
          └──────────────────────────────┬─────────────────────────────────────┘
                                         │
          ┌──────────────────────────────┴─────────────────────────────────────┐
          │          useJson store (纯 JSON 字符串)  store/useJson.ts          │
          └──────────┬─────────────────────────────────────────────────────────┘
                     │
          ┌──────────┴───────────────────┐
          ▼                              ▼
  GraphView (Graph 模式)          TreeView (Tree 模式)
  <JSONCrack json={json}/>        <JSONTree data=JSON.parse(json)/>
          │
          ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│          JSONCrack Component  (packages/jsoncrack-react/src/)                         │
│  JSONCrackComponent.tsx#L168-L205                                                     │
│                                                                                       │
│  ① toJsonText(json) → string                                                          │
│     canvasHelpers.ts#L13-L26                                                          │
│     ├─ typeof json === "string" → 直接返回                                            │
│     └─ object → JSON.stringify(json, null, 2)                                         │
│             使用 WeakMap<object, string> 缓存，避免同引用重复序列化                    │
│                                                                                       │
│  ② parseJsonGraph(jsonText, maxRenderableNodes)                                       │
│     canvasHelpers.ts#L74-L90                                                          │
│     └─ parseGraph(jsonText)  parser.ts#L9-L219                                        │
│         ├─ jsonc-parser.parseTree(json, errors) → AST                                │
│         └─ traverse(AST) → nodes[] + edges[] + errors[]                              │
│                                                                                       │
│  ③ 阈值保护: graph.nodes.length > maxRenderableNodes → { kind: "above-limit" }       │
│                                                                                       │
│  ④ setNodes / setEdges → reaflow <Canvas> ELK 布局 + framer-motion 动画渲染          │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. 关键设计要点

1. **格式适配层前置**：`contentToJson()` 在 `useFile.setContents()` 中率先将任意格式转为 JS 对象，后续链路只需处理 JSON（`apps/www/src/lib/utils/jsonAdapter.ts#L4-L45`）。
2. **双类型输入兼容**：`JSONCrack` 组件的 `json` prop 接受 `string | object`（`packages/jsoncrack-react/src/canvasHelpers.ts#L8`），`toJsonText()` 内部做归一化，并使用 WeakMap 缓存对象序列化结果。
3. **容错 JSON 解析**：核心 `parseGraph()` 使用 `jsonc-parser` 的 `parseTree()`，容忍 JSON 中的注释和尾逗号；单元测试 `packages/jsoncrack-react/src/__tests__/parser.test.ts#L95-L99` 验证了此特性。
4. **防抖更新**：编辑器输入经 400ms 防抖后写入 `useJson`（`apps/www/src/store/useFile.ts#L67-L69`），避免频繁重绘。
5. **会话恢复**：`sessionStorage` 同时保存 `contents`（原始格式文本）和 `format`，刷新后 `checkEditorSession()` 正确恢复并重新走完整链路（`apps/www/src/store/useFile.ts#L142-L154`）。
6. **VSCode 三命令策略各异，不得混淆**：
   - `createWebviewForActiveEditor`（`.start`）：**仅 ready 握手 + 变更时全文同步**，无创建后即时发送，无前置校验（`apps/vscode/ext-src/extension.ts#L67-L93`）
   - `createWebviewForSelectedText`（`.start.selected`）：**唯一**有前置校验（`selection.isEmpty` 则提示并 return），**唯一**创建后立即发送，**唯一**变更时重取选区切片而非全文（`apps/vscode/ext-src/extension.ts#L27-L65`）
   - `createWebviewForContent`（`.start.specific`）：**唯一**无任何监听器（无 ready 握手、无变更监听），仅创建后发一次，纯一次性展示（`apps/vscode/ext-src/extension.ts#L100-L110`）
   - 三条命令 Webview 侧接收逻辑共用同一套：只要收到 `event.data.json` 为字符串就更新 state，驱动 `<JSONCrack>` 渲染（`apps/vscode/src/App.tsx#L26-L41`）
7. **Chrome Worker 规避**：导入 `jsoncrack-react` 前临时 shadow 全局 `Worker`（`apps/chrome-extension/src/content-script.tsx#L38-L47`），使 ELK 走同步路径，应对 JSON 响应页的严格 CSP。
8. **外部集成直通**：VSCode 扩展和 Chrome 扩展跳过 `useFile`/`jsonAdapter`（它们本身已确定是 JSON），直接将 JSON 传入 `<JSONCrack>`，由 `toJsonText()` + `parseGraph()` 处理，复用同一核心解析器。
9. **双向格式桥**：`contentToJson()` + `jsonToContent()` 构成完整双向转换（`apps/www/src/lib/utils/jsonAdapter.ts`），BottomBar 切换格式时能无损互转（`apps/www/src/store/useFile.ts#L86-L98`）。
