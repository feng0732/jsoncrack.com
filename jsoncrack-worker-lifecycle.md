# JSON Crack Web Worker 生命周期分析报告

## 1. 概述

本文档分析 JSON Crack 项目中 Web Worker 的使用情况、任务分发机制和回收流程。JSON Crack 是一个 JSON 数据可视化工具，将 JSON 数据转换为交互式的节点-边关系图。

### 1.1 核心发现

**项目本身没有实现自定义的 JSON 解析 Web Worker**。JSON 解析在主线程同步执行，而唯一的 Web Worker 使用来自依赖库 `reaflow` 中的 ELK 布局引擎，用于图形布局计算的卸载。

---

## 2. 架构概览

### 2.1 技术栈分层

```
┌─────────────────────────────────────────────────────┐
│                    应用层 (主线程)                    │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐ │
│  │  JSON 解析   │  │  状态管理    │  │  UI 渲染      │ │
│  │ (jsonc-     │  │ (zustand)   │  │  (React)      │ │
│  │  parser)    │  │             │  │              │ │
│  └─────────────┘  └─────────────┘  └──────────────┘ │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│                图形渲染层 (reaflow)                   │
│  ┌───────────────────────────────────────────────┐  │
│  │              Canvas 组件                        │  │
│  │  (节点/边渲染 + ELK 布局集成)                    │  │
│  └───────────────────┬───────────────────────────┘  │
│                      │ Worker 通信                    │
│  ┌───────────────────▼───────────────────────────┐  │
│  │            ELK Layout Worker                   │  │
│  │    (web-worker + elkjs)                        │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## 3. Web Worker 详细分析

### 3.1 Worker 来源与依赖

| 依赖包 | 版本 | 作用 |
|--------|------|------|
| `reaflow` | 5.4.1 | React 流程图库，提供 Canvas 组件 |
| `elkjs` | 0.10.2 | Eclipse Layout Kernel，图形布局算法 |
| `web-worker` | 1.5.0 | Worker 管理工具库 |

**依赖关系**：
- `jsoncrack-react` → `reaflow` → `elkjs` + `web-worker`

### 3.2 Worker 的用途

Web Worker 仅用于 **ELK 图形布局计算**，不涉及 JSON 解析。具体任务包括：
- 节点位置计算（分层布局算法）
- 边路径计算
- 布局约束满足
- 标签放置优化

### 3.3 Worker 创建时机

Worker 由 `reaflow` 内部的 `web-worker` 库管理，创建时机：
1. `Canvas` 组件首次挂载时
2. 首次接收到 nodes/edges 数据需要布局时
3. 由 `elkjs` 内部的 `web-worker` 包懒加载创建

---

## 4. 任务分发与回收流程

### 4.1 完整数据流

```
用户输入 JSON
     │
     ▼
┌──────────────────┐
│ 主线程: JSON 解析 │  使用 jsonc-parser 同步解析
│ parseGraph()     │  位置: parser.ts
└─────────┬────────┘
          │ nodes, edges
          ▼
┌──────────────────┐
│ 主线程: 状态更新  │  setNodes(), setEdges()
│ React useState   │  位置: JSONCrackComponent.tsx
└─────────┬────────┘
          │ props 传递
          ▼
┌──────────────────┐
│ reaflow Canvas   │
│ 组件             │
└─────────┬────────┘
          │
          ▼
┌──────────────────┐
│ Worker: ELK 布局  │  postMessage 分发任务
│ elkjs 算法        │
└─────────┬────────┘
          │ 布局结果
          ▼
┌──────────────────┐
│ 主线程: 布局回调  │  onLayoutChange 回调
│ 更新画布尺寸      │  位置: JSONCrackComponent.tsx
└─────────┬────────┘
          │
          ▼
    渲染图形到 SVG
```

### 4.2 任务分发机制

#### 4.2.1 JSON 解析（主线程同步）

JSON 解析在主线程通过 `jsonc-parser` 库同步执行，不使用 Worker：

**核心代码位置**：[parser.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/parser.ts#L9-L219)

```typescript
// 解析入口函数
export const parseGraph = (json: string): ParseGraphResult => {
  const parseErrors: ParseError[] = [];
  const jsonTree = parseTree(json, parseErrors); // 同步解析
  
  // 遍历构建节点和边
  // ...
  return { nodes, edges, errors: parseErrors };
};
```

**触发时机**：
- `json` prop 变化时
- 在 `JSONCrackComponent` 的 `useEffect` 中触发

**代码位置**：[JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L171-L205)

```typescript
useEffect(() => {
  setLoading(true);
  setInitialFitDone(false);
  const result = parseJsonGraph(jsonText, maxRenderableNodes); // 同步调用
  
  if (result.kind === "ok") {
    setNodes(graph.nodes);
    setEdges(graph.edges);
    // ...
  }
}, [jsonText, maxRenderableNodes]);
```

#### 4.2.2 布局计算（Worker 异步）

布局计算由 `reaflow` 的 `Canvas` 组件内部管理，通过 Worker 异步执行。

**任务分发流程**：

1. **数据输入**：`nodes` 和 `edges` 作为 props 传递给 `Canvas` 组件
2. **Worker 通信**：`reaflow` 内部通过 `web-worker` 包将布局任务发送给 Worker
3. **ELK 计算**：Worker 中运行 `elkjs` 布局算法
4. **结果回调**：布局完成后通过 `onLayoutChange` 回调返回结果

**代码位置**：[JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L401-L411)

```typescript
const onLayoutChange = useCallback((layout: ElkRoot) => {
  if (!layout.width || !layout.height) {
    setLoading(false);
    return;
  }

  layoutSizeRef.current = { width: layout.width, height: layout.height };
  setPaneWidth(layout.width + 50);
  setPaneHeight(layout.height + 50);
  setLoading(false); // 布局完成，取消加载状态
}, []);
```

### 4.3 Worker 回收机制

Worker 的生命周期由 `reaflow` 和 `web-worker` 内部管理：

1. **组件卸载时**：`Canvas` 组件卸载时会清理 Worker
2. **Worker 池管理**：`web-worker` 库可能会复用或销毁 Worker
3. **内存管理**：布局任务完成后 Worker 保持空闲，等待下一个任务

> **注意**：由于 Worker 管理在 `reaflow` 内部，JSON Crack 项目代码中没有显式的 `worker.terminate()` 调用。

---

## 5. 各平台的 Worker 处理策略

### 5.1 Web 应用（www）

**完整的 Worker 支持**：
- 标准浏览器环境，ELK Worker 正常工作
- 布局计算在后台线程执行，不阻塞 UI
- 加载状态通过 `loading` state 管理

**相关文件**：
- [editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/www/src/pages/editor.tsx)
- [GraphView/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/www/src/features/editor/views/GraphView/index.tsx)

### 5.2 Chrome 扩展（chrome-extension）

**Worker 禁用与回退机制**：

由于 Chrome 扩展的内容脚本（Content Script）可能面临严格的 CSP（Content Security Policy），导致 Worker 创建失败。项目采用了**临时禁用 Worker → 同步回退 → 恢复 Worker** 的策略。

#### 5.2.1 实现机制

**代码位置**：[content-script.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/chrome-extension/src/content-script.tsx#L28-L50)

```typescript
const loadJsonCrackComponent = async (): Promise<JSONCrackComponentType> => {
  if (jsonCrackComponentPromise) {
    return jsonCrackComponentPromise;
  }

  jsonCrackComponentPromise = (async () => {
    // 保存原始 Worker 构造函数
    const originalWorker = (globalThis as { Worker?: typeof Worker }).Worker;
    try {
      // 临时禁用 Worker
      (globalThis as { Worker?: typeof Worker }).Worker = undefined;
      // 动态导入组件（此时 ELK 会检测到 Worker 不可用）
      const mod = await import("jsoncrack-react");
      return mod.JSONCrack as JSONCrackComponentType;
    } finally {
      // 恢复 Worker，确保宿主页面的其他 Worker 正常工作
      (globalThis as { Worker?: typeof Worker }).Worker = originalWorker;
    }
  })();

  return jsonCrackComponentPromise;
};
```

#### 5.2.2 构建级防护

**代码位置**：[vite.config.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/chrome-extension/vite.config.ts#L23-L28)

```typescript
rollupOptions: {
  output: {
    banner:
      "var Worker = undefined; var process = globalThis.process || (globalThis.process = { env: { NODE_ENV: 'production' } });",
  },
},
```

在构建产物的开头注入代码，确保 Worker 在脚本开始时就被禁用。

#### 5.2.3 回退效果

- ELK 布局引擎检测到 Worker 不可用，自动回退到**同步布局模式**
- 布局计算在主线程执行，可能会短暂阻塞 UI
- 功能完整，只是性能有所下降

### 5.3 VS Code 扩展（vscode）

**非 Worker 通信：Webview postMessage**

VS Code 扩展使用的是 VS Code 内置的 Webview 消息传递机制，不是 Web Worker。

#### 5.3.1 通信架构

```
┌──────────────────────┐        ┌──────────────────────┐
│  Extension Host      │        │  Webview (渲染线程)   │
│  (Node.js 进程)      │        │  (浏览器环境)         │
└─────────┬────────────┘        └─────────┬────────────┘
          │                                │
          │   panel.webview.postMessage()  │
          │ ─────────────────────────────> │
          │                                │
          │   onDidReceiveMessage          │
          │ <───────────────────────────── │
          │                                │
```

#### 5.3.2 消息类型

**扩展 → Webview**：
- `json`：JSON 内容字符串
- 初始加载时发送
- 文档内容变化时发送

**代码位置**：[extension.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/ext-src/extension.ts#L39-L64)

```typescript
panel.webview.postMessage({
  json: selectedText,
});

const onTextChange = vscode.workspace.onDidChangeTextDocument(changeEvent => {
  if (changeEvent.document === editor?.document) {
    panel.webview.postMessage({
      json: changeEvent.document.getText(editor?.selection),
    });
  }
});
```

**Webview → 扩展**：
- `"ready"`：Webview 就绪信号

**代码位置**：[App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/src/App.tsx#L26-L42)

```typescript
useEffect(() => {
  const vscode = window?.acquireVsCodeApi?.();
  vscode?.postMessage("ready");

  const onMessage = (event: MessageEvent<{ json?: string }>) => {
    const jsonData = event.data?.json;
    if (typeof jsonData === "string") {
      setJson(jsonData);
    }
  };

  window.addEventListener("message", onMessage);
  return () => {
    window.removeEventListener("message", onMessage);
  };
}, []);
```

#### 5.3.3 资源清理

**代码位置**：[extension.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/ext-src/extension.ts#L59-L64)

```typescript
const disposer = () => {
  onTextChange.dispose();
  onReceiveMessage.dispose();
};

panel.onDidDispose(disposer, null, context.subscriptions);
```

### 5.4 Widget 嵌入页面

**非 Worker 通信：iframe postMessage**

Widget 页面使用 `window.postMessage` 进行跨 iframe 通信，不是 Web Worker。

**代码位置**：[widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/www/src/pages/widget.tsx#L47-L75)

```typescript
// 发送就绪信号给父页面
window.parent.postMessage(window.frameElement?.getAttribute("id"), "*");

// 监听父页面的消息
const handler = (event: EmbedMessage) => {
  if (event.data?.json) {
    setContents({ contents: event.data.json, hasChanges: false });
    // ...
  }
};

window.addEventListener("message", handler);
```

---

## 6. 加载状态管理

### 6.1 加载状态流转

```
初始状态: loading = true
     │
     ▼
JSON 解析开始
     │
     ├─ 解析成功 → 等待布局
     │      │
     │      ▼
     │   布局计算中 (Worker 中)
     │      │
     │      ▼
     │   布局完成 → setLoading(false)
     │
     └─ 解析失败 → setLoading(false) + onParseError
```

### 6.2 加载状态触发点

| 触发场景 | 代码位置 | 说明 |
|---------|---------|------|
| JSON 数据变化 | [JSONCrackComponent.tsx#L172](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L172) | 重新解析和布局 |
| 折叠/展开节点 | [JSONCrackComponent.tsx#L398](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L398) | 触发重新布局 |
| 布局完成回调 | [JSONCrackComponent.tsx#L410](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L410) | 取消加载状态 |

---

## 7. 性能考虑

### 7.1 为什么 JSON 解析不在 Worker 中？

1. **jsonc-parser 效率高**：使用增量解析和错误容忍，解析速度很快
2. **数据依赖性**：解析结果直接用于 React 渲染，需要频繁访问
3. **实现复杂度**：增加 Worker 会增加代码复杂度和通信开销
4. **主要瓶颈在布局**：大型 JSON 的性能瓶颈主要在 ELK 布局计算，而非解析

### 7.2 优化措施

1. **防抖更新**：使用 `lodash.debounce` 减少解析频率
   - [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/www/src/store/useFile.ts#L67-L69)

2. **节点数量限制**：超过 `maxRenderableNodes` 时不渲染
   - 默认限制 1500 个节点

3. **折叠功能**：用户可以折叠部分节点，减少布局计算量

---

## 8. 总结

### 8.1 Worker 使用清单

| 模块 | Worker 类型 | 用途 | 管理方式 |
|------|------------|------|---------|
| ELK 布局 | Web Worker | 图形布局计算 | reaflow 内部管理 |
| JSON 解析 | 无（主线程） | JSON → 图数据结构 | 同步执行 |
| Chrome 扩展 | 禁用状态 | CSP 兼容 | 临时禁用 + 同步回退 |
| VS Code 扩展 | 无（Webview 消息） | 扩展与视图通信 | VS Code API |
| Widget 页面 | 无（iframe 消息） | 父子页面通信 | postMessage API |

### 8.2 关键文件索引

| 文件路径 | 作用 |
|---------|------|
| [parser.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/parser.ts) | JSON 解析核心逻辑 |
| [JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx) | 主组件，布局回调处理 |
| [canvasHelpers.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts) | 解析辅助函数 |
| [content-script.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/chrome-extension/src/content-script.tsx) | Chrome 扩展 Worker 禁用逻辑 |
| [extension.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/ext-src/extension.ts) | VS Code 扩展消息发送 |
| [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/src/App.tsx) | VS Code Webview 消息接收 |
| [widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/www/src/pages/widget.tsx) | Widget 页面 postMessage |

### 8.3 核心结论

1. **JSON 解析不在 Web Worker 中执行**，而是在主线程同步进行
2. **唯一的 Web Worker 用于 ELK 图形布局计算**，由 `reaflow` 库内部管理
3. **Chrome 扩展需要特殊处理**，通过临时禁用 Worker 来兼容 CSP 限制
4. **VS Code 扩展和 Widget 使用 postMessage**，但这是跨上下文通信，不是 Web Worker
5. **Worker 的创建和回收由第三方库管理**，项目代码不直接控制
