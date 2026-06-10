# JSON Crack Web Worker 生命周期分析报告

## 1. 概述

本文档深入分析 JSON Crack 项目中 Web Worker 的使用情况、任务分发机制、CSP 回退策略以及空闲与销毁边界。

### 1.1 核心结论

| 结论 | 说明 |
|------|------|
| **无自定义 JSON 解析 Worker** | JSON 解析在主线程同步执行，使用 `jsonc-parser` |
| **唯一 Worker 用于 ELK 布局** | 来自依赖库 `reaflow`，用于图形布局计算卸载 |
| **四个平台都用 JSONCrack 组件** | Web 编辑器、Widget 嵌入页、VS Code webview、Chrome 扩展 |
| **内部 Worker ≠ 跨上下文通信** | Widget/VS Code 的 postMessage 是跨上下文通信，与内部布局 Worker 是两回事 |
| **Chrome 扩展需禁用 Worker** | 因 CSP 限制，通过两层防护机制回退到同步模式 |

---

## 2. 架构总览

### 2.1 四层架构

```
┌───────────────────────────────────────────────────────────────┐
│                      宿主环境层                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Web 浏览器│  │ iframe   │  │ VS Code  │  │ Chrome 扩展    │  │
│  │ (编辑器)  │  │ (widget) │  │ webview  │  │ content script│  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬───────┘  │
│       │             │             │                │          │
└───────┼─────────────┼─────────────┼────────────────┼──────────┘
        │             │             │                │
        ▼             ▼             ▼                ▼
┌───────────────────────────────────────────────────────────────┐
│                    应用状态层 (主线程)                         │
│  Zustand stores / React state / JSON 解析                      │
└──────────────────────────┬────────────────────────────────────┘
                           │ nodes, edges
                           ▼
┌───────────────────────────────────────────────────────────────┐
│                   JSONCrack 组件层                             │
│  节点渲染 / 边渲染 / 视口控制 / 折叠管理                        │
└──────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
┌───────────────────────────────────────────────────────────────┐
│                   reaflow Canvas 层                            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                   ELK Layout Worker                     │  │
│  │  (elkjs + web-worker，可选 — Chrome 扩展下被禁用)        │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

### 2.2 各平台的 Worker 状态对比

| 平台 | 内部 ELK 布局 Worker | 跨上下文通信方式 | CSP 限制 |
|------|---------------------|-----------------|---------|
| Web 编辑器 | ✅ 正常运行 | 无 | 无 |
| Widget 嵌入页 | ✅ 正常运行 | `window.postMessage`（与父窗口） | 取决于宿主页面 |
| VS Code webview | ✅ 正常运行 | VS Code API（与扩展主机） | 显式配置 `worker-src` |
| Chrome 扩展 | ❌ 禁用（同步回退） | 无 | 宿主页面 CSP 可能严格 |

> **重要区分**：Widget 页面的 `postMessage` 和 VS Code 的 `postMessage` 是**跨上下文通信机制**，用于不同执行上下文之间传递数据。它们不是 Web Worker，但都使用了消息传递模式。而 **ELK 布局 Worker** 是在同一个渲染上下文中的后台线程，用于计算卸载。

---

## 3. ELK 布局 Worker 详解

### 3.1 Worker 来源与依赖链

```
jsoncrack-react
    ↓ (依赖)
reaflow v5.4.1
    ↓ (依赖)
elkjs v0.10.2 + web-worker v1.5.0
    ↓ (运行时)
ELK Layout Worker
```

**依赖说明**：

| 包 | 角色 |
|----|------|
| `reaflow` | React 流程图库，提供 `Canvas` 组件 |
| `elkjs` | Eclipse Layout Kernel，布局算法核心 |
| `web-worker` | Worker 池管理工具，封装 Worker 通信 |

### 3.2 Worker 的用途

Web Worker 仅用于 **ELK 图形布局计算**，不涉及 JSON 解析。具体计算任务：

- 节点位置计算（分层布局算法 `layered`）
- 边路径计算（正交连线）
- 布局约束满足（间距、方向等）
- 标签位置优化

### 3.3 任务分发机制

#### 3.3.1 分发入口：JSONCrack 组件

`JSONCrack` 组件本身不直接操作 Worker，而是通过 `reaflow` 的 `Canvas` 组件间接使用。

**代码位置**：[JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L585-L611)

```tsx
<Canvas
  className="jsoncrack-canvas"
  onLayoutChange={onLayoutChange}      // 布局完成回调
  node={renderNode}
  edge={renderEdge}
  nodes={visibleNodes}                 // 输入：可见节点
  edges={visibleEdges}                 // 输入：可见边
  direction={layoutDirection}
  layoutOptions={layoutOptions}
  // ...
/>
```

#### 3.3.2 分发流程

```
主线程 (React)                         Worker (ELK)
    │                                     │
    │ 1. nodes/edges 作为 props 传入       │
    │    Canvas 组件                       │
    │                                     │
    │ 2. reaflow 内部调用 elkjs            │
    │    ── postMessage(layoutTask) ─────►│
    │                                     │
    │                                     │ 3. ELK 算法计算布局
    │                                     │    (分层布局算法)
    │                                     │
    │ 4. 布局结果返回                      │
    │ ◄── postMessage(layoutResult) ──────│
    │                                     │
    │ 5. onLayoutChange 回调               │
    │    更新 paneWidth/paneHeight         │
    │    setLoading(false)                │
    │                                     │
```

#### 3.3.3 布局回调：结果回收

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
  setLoading(false);  // 布局完成，取消加载状态
}, []);
```

---

## 4. Worker 生命周期边界

### 4.1 创建时机

Worker 由 `reaflow` 内部的 `web-worker` 库管理，创建发生在：

1. **`Canvas` 组件首次挂载**：组件挂载后首次需要布局时
2. **懒加载创建**：`web-worker` 库在首次需要布局计算时才创建 Worker 实例
3. **模块初始化时检测**：`elkjs` 在模块加载时检测 Worker 可用性，决定使用 Worker 模式还是同步模式

### 4.2 空闲状态

布局计算完成后，Worker 进入空闲状态：

- **状态**：Worker 保持运行，等待新的布局任务
- **触发新任务**：当 `nodes` 或 `edges` 变化时，`reaflow` 重新布局，新任务被发送到 Worker
- **触发场景**：
  - JSON 数据变化 → 重新解析 → 新的 nodes/edges → 重新布局
  - 折叠/展开节点 → visibleNodes 变化 → 重新布局
  - 布局方向变化 → 重新布局

### 4.3 销毁时机

Worker 的销毁由 `reaflow` 内部管理，销毁时机：

1. **`Canvas` 组件卸载**：组件 `unmount` 时，`reaflow` 清理 Worker
2. **页面关闭**：浏览器标签关闭时，所有 Worker 自动终止
3. **无显式 terminate**：JSON Crack 代码中没有直接调用 `worker.terminate()`

### 4.4 生命周期状态机

```
            组件挂载
               │
               ▼
         ┌───────────┐
         │  未创建    │  Worker 尚未创建
         └─────┬─────┘
               │ 首次布局
               ▼
         ┌───────────┐
         │  创建中    │  web-worker 初始化 Worker
         └─────┬─────┘
               │
               ▼
         ┌───────────┐
   ┌────►│  运行中    │  处理布局任务
   │     └─────┬─────┘
   │           │
   │ 布局完成   │ 新布局任务
   │           ▼
   │     ┌───────────┐
   │     │  空闲中    │  等待新任务
   │     └───────────┘
   │
   └──── 组件卸载
         │
         ▼
   ┌───────────┐
   │  已销毁    │  Worker 被终止
   └───────────┘
```

---

## 5. 各平台的 Worker 实现细节

### 5.1 Web 编辑器（www/editor）

**完整 Worker 支持**：

- 标准浏览器环境，ELK Worker 正常工作
- 布局计算在后台线程执行，不阻塞 UI
- 加载状态通过 `loading` state 管理

**组件链路**：
[editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/www/src/pages/editor.tsx)
→ [LiveEditor](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/www/src/features/editor/LiveEditor.tsx)
→ [GraphView](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/www/src/features/editor/views/GraphView/index.tsx)
→ `JSONCrack` 组件
→ `Canvas` 组件
→ ELK Worker

### 5.2 Widget 嵌入页（www/widget）

#### 5.2.1 内部布局 Worker

Widget 页面内部运行完整的 `JSONCrack` 组件，因此 **有 ELK 布局 Worker**。

**组件链路**：
[widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/www/src/pages/widget.tsx)
→ `GraphView` 组件 (`isWidget` 模式)
→ `JSONCrack` 组件
→ `Canvas` 组件
→ ELK Worker

#### 5.2.2 外部通信：postMessage

Widget 页面通过 `window.postMessage` 与**父窗口**（嵌入它的页面）通信。这是跨 iframe 的消息传递，不是 Web Worker。

**代码位置**：[widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/www/src/pages/widget.tsx#L47-L75)

```typescript
// 发送就绪信号给父窗口
window.parent.postMessage(window.frameElement?.getAttribute("id"), "*");

// 监听父窗口的消息
const handler = (event: EmbedMessage) => {
  if (event.data?.json) {
    setContents({ contents: event.data.json, hasChanges: false });
    setDirection(event.data.options?.direction || "RIGHT");
  }
};

window.addEventListener("message", handler);
```

#### 5.2.3 双层通信模型

```
┌─────────────────────────────────────────────────────────┐
│                   父窗口 (宿主页面)                       │
└──────────────────────┬──────────────────────────────────┘
                       │ window.postMessage
                       │ (跨 iframe 通信)
                       ▼
┌─────────────────────────────────────────────────────────┐
│              Widget iframe (渲染主线程)                   │
│  ┌───────────────────────────────────────────────────┐  │
│  │  JSONCrack 组件 (React 渲染)                       │  │
│  └───────────────────────┬───────────────────────────┘  │
│                          │ postMessage                   │
│                          │ (内部 Worker 通信)            │
│                          ▼                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │           ELK 布局 Worker                          │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 5.3 VS Code 扩展

#### 5.3.1 内部布局 Worker

VS Code 的 webview 内部运行完整的 React 应用和 `JSONCrack` 组件，因此 **有 ELK 布局 Worker**。

**组件链路**：
[extension.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/ext-src/extension.ts)
→ 创建 webview
→ [index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/src/index.tsx)
→ [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/src/App.tsx)
→ `JSONCrack` 组件
→ `Canvas` 组件
→ ELK Worker

#### 5.3.2 CSP 配置

Webview 的 CSP 显式配置了 `worker-src`，允许 Worker 运行。

**代码位置**：[webview.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/ext-src/webview.ts#L28-L34)

```typescript
const csp = [
  `default-src 'self' ${panel.webview.cspSource} blob:`,
  `connect-src ${panel.webview.cspSource} blob:`,
  `script-src 'unsafe-eval' 'unsafe-inline' ${panel.webview.cspSource}`,
  `style-src ${panel.webview.cspSource} 'unsafe-inline'`,
  `worker-src ${panel.webview.cspSource} blob: data:`,   // 允许 Worker
].join("; ");
```

`worker-src` 指令允许从以下来源创建 Worker：
- `${panel.webview.cspSource}`：webview 自身的源
- `blob:`：Blob URL 创建的 Worker（`web-worker` 库常用方式）
- `data:`：Data URL 创建的 Worker

#### 5.3.3 外部通信：VS Code Webview API

扩展主机（Node.js 进程）与 webview（渲染进程）之间通过 VS Code API 通信。这是跨进程通信，不是 Web Worker。

**扩展 → Webview**（发送 JSON 数据）：
[extension.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/ext-src/extension.ts#L39-L42)

```typescript
panel.webview.postMessage({
  json: selectedText,
});
```

**Webview → 扩展**（发送就绪信号）：
[App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/src/App.tsx#L26-L28)

```typescript
const vscode = window?.acquireVsCodeApi?.();
vscode?.postMessage("ready");
```

#### 5.3.4 三层架构

```
┌─────────────────────────────────────────────────────────┐
│          Extension Host (Node.js 进程)                   │
│            extension.ts                                  │
└──────────────────────┬──────────────────────────────────┘
                       │ VS Code Webview API
                       │ (跨进程通信)
                       ▼
┌─────────────────────────────────────────────────────────┐
│           Webview (浏览器渲染进程)                        │
│  ┌───────────────────────────────────────────────────┐  │
│  │  JSONCrack 组件 (React 渲染)                       │  │
│  └───────────────────────┬───────────────────────────┘  │
│                          │ postMessage                   │
│                          │ (内部 Worker 通信)            │
│                          ▼                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │           ELK 布局 Worker                          │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

#### 5.3.5 资源销毁

**代码位置**：[extension.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/ext-src/extension.ts#L59-L64)

```typescript
const disposer = () => {
  onTextChange.dispose();    // 清理文档变更监听
  onReceiveMessage.dispose(); // 清理消息监听
};

panel.onDidDispose(disposer, null, context.subscriptions);
```

当 webview panel 关闭时：
1. VS Code 销毁 webview 实例
2. webview 中的 React 组件卸载
3. `Canvas` 组件卸载 → 内部 ELK Worker 被清理
4. 扩展端清理事件监听器

### 5.4 Chrome 扩展

#### 5.4.1 Worker 禁用：CSP 回退

Chrome 扩展的内容脚本（Content Script）注入到第三方页面中，这些页面可能有严格的 CSP（如 `default-src 'none'`），导致 Worker 创建失败。

项目采用**两层防护机制**确保 ELK 回退到同步布局模式。

##### 第一层：构建时 Banner 注入

**代码位置**：[vite.config.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/chrome-extension/vite.config.ts#L23-L28)

```typescript
rollupOptions: {
  output: {
    banner:
      "var Worker = undefined; var process = globalThis.process || (globalThis.process = { env: { NODE_ENV: 'production' } });",
  },
},
```

**作用**：
- 在 IIFE 打包产物的函数体开头注入 `var Worker = undefined;`
- 由于 `var` 声明在函数作用域内，它会**遮蔽**（shadow）全局的 `Worker` 构造函数
- 整个 content script 代码执行期间，直接引用 `Worker` 都会得到 `undefined`

##### 第二层：运行时全局禁用

**代码位置**：[content-script.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/chrome-extension/src/content-script.tsx#L28-L50)

```typescript
const loadJsonCrackComponent = async (): Promise<JSONCrackComponentType> => {
  if (jsonCrackComponentPromise) {
    return jsonCrackComponentPromise;
  }

  jsonCrackComponentPromise = (async () => {
    // ELK (the layout engine used by reaflow) tries to spawn a Web Worker
    // on first use. On JSON pages with a strict `default-src 'none'` CSP,
    // worker creation throws. Temporarily shadow `Worker` for the duration
    // of the import + initial layout so ELK falls back to its sync path,
    // then restore it so the host page's own workers keep working.
    const originalWorker = (globalThis as { Worker?: typeof Worker }).Worker;
    try {
      (globalThis as { Worker?: typeof Worker }).Worker = undefined;
      const mod = await import("jsoncrack-react");
      return mod.JSONCrack as JSONCrackComponentType;
    } finally {
      (globalThis as { Worker?: typeof Worker }).Worker = originalWorker;
    }
  })();

  return jsonCrackComponentPromise;
};
```

**关键逻辑**：
1. **保存**：保存原始的 `globalThis.Worker`
2. **禁用**：将 `globalThis.Worker` 设为 `undefined`
3. **导入**：动态导入 `jsoncrack-react` 模块
4. **恢复**：在 `finally` 块中恢复 `globalThis.Worker`

#### 5.4.2 为什么两层防护都需要？

| 层级 | 作用域 | 影响范围 | 目的 |
|------|--------|---------|------|
| Banner `var Worker = undefined` | IIFE 函数作用域 | content script 自身代码 | 遮蔽直接的 `Worker` 引用 |
| `globalThis.Worker = undefined` | 全局作用域 | 动态导入的模块 | 确保导入的模块也看不到 Worker |

动态导入的模块运行在自己的模块作用域中，它们访问 `Worker` 时会查找全局作用域，因此需要修改 `globalThis.Worker`。

#### 5.4.3 禁用时机与恢复边界

```
时间轴 →
  │
  ├─ content script 开始执行
  │   (banner: var Worker = undefined)
  │
  ├─ loadJsonCrackComponent() 调用
  │   │
  │   ├─ 保存 originalWorker
  │   ├─ globalThis.Worker = undefined  ──┐
  │   │                                    │ 禁用期
  │   ├─ await import("jsoncrack-react")  │
  │   │    (ELK 模块初始化)                │
  │   │    (检测到 Worker 不可用)           │
  │   │    (决定使用同步模式)              │
  │   │                                    │
  │   └─ globalThis.Worker = originalWorker ──┘
  │       (恢复全局 Worker)
  │
  ├─ 用户点击 Graph 按钮
  │   └─ 组件挂载 → 首次布局（同步模式）
  │
  ├─ 后续布局计算
  │   └─ 始终使用同步模式（ELK 已确定模式）
  │
  ▼
```

**关键点**：
- **禁用窗口**：只在 `import` 执行期间禁用全局 Worker
- **模式固化**：ELK 在模块初始化时检测 Worker 可用性，一旦决定使用同步模式，后续即使恢复了 `globalThis.Worker` 也不会再尝试创建 Worker
- **宿主页面不受影响**：恢复 `globalThis.Worker` 确保宿主页面的其他 Worker 正常工作

#### 5.4.4 同步模式的表现

- 布局计算在**主线程**执行
- 大型 JSON 文件可能会短暂阻塞 UI
- 功能完整，所有布局特性都支持
- 加载状态管理逻辑不变

#### 5.4.5 组件卸载与资源清理

**代码位置**：[content-script.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/chrome-extension/src/content-script.tsx#L149-L154)

```typescript
const unmountGraphView = () => {
  if (!graphRoot) return;
  graphRoot.unmount();
  graphRoot = null;
  reactRootContainer.textContent = "";
};
```

当用户切换回 "Raw" 模式时：
1. React 组件卸载
2. `Canvas` 组件卸载
3. ELK 同步实例被清理
4. DOM 容器清空

（由于 Worker 被禁用，不存在 Worker 销毁的问题）

---

## 6. JSON 解析：主线程同步执行

### 6.1 为什么 JSON 解析不在 Worker 中？

1. **jsonc-parser 效率高**：使用增量解析和错误容忍，解析速度很快
2. **数据依赖性强**：解析结果直接用于 React 渲染，需要频繁访问
3. **主要瓶颈在布局**：大型 JSON 的性能瓶颈主要在 ELK 布局计算，而非解析
4. **实现复杂度**：增加 Worker 会增加代码复杂度和通信开销

### 6.2 解析流程

**代码位置**：[parser.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/parser.ts#L9-L219)

```typescript
export const parseGraph = (json: string): ParseGraphResult => {
  const parseErrors: ParseError[] = [];
  const jsonTree = parseTree(json, parseErrors); // 同步解析

  // 遍历构建节点和边（同步）
  // ...

  return { nodes, edges, errors: parseErrors };
};
```

**触发位置**：[JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L171-L205)

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

---

## 7. 加载状态与任务流转

### 7.1 加载状态流转

```
初始 loading = true
     │
     ▼
JSON 解析（同步）
     │
     ├─ 解析失败 → setLoading(false) + onParseError
     │
     └─ 解析成功 → nodes/edges 更新 → Canvas 重新布局
                           │
                           ▼
                    布局计算中 (Worker 中)
                           │
                           ▼
              onLayoutChange 回调 → setLoading(false)
```

### 7.2 加载状态触发点

| 触发场景 | 代码位置 | 说明 |
|---------|---------|------|
| JSON 数据变化 | [JSONCrackComponent.tsx#L172](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L172) | 重新解析 + 重新布局 |
| 折叠/展开节点 | [JSONCrackComponent.tsx#L398](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L398) | visibleNodes 变化 → 重新布局 |
| 布局完成 | [JSONCrackComponent.tsx#L410](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L410) | onLayoutChange 回调 |

### 7.3 防抖优化

为了减少频繁解析和布局，输入层使用了防抖。

**代码位置**：[useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/www/src/store/useFile.ts#L67-L69)

```typescript
const debouncedUpdateJson = debounce((value: unknown) => {
  useJson.getState().setJson(JSON.stringify(value, null, 2));
}, 400);
```

---

## 8. 关键文件索引

| 文件 | 作用 |
|------|------|
| [parser.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/parser.ts) | JSON 解析核心逻辑（主线程同步） |
| [JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx) | 主组件，布局回调与状态管理 |
| [canvasHelpers.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts) | 解析辅助函数 |
| [widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/www/src/pages/widget.tsx) | Widget 嵌入页，postMessage 通信 + 内部 Worker |
| [webview.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/ext-src/webview.ts) | VS Code webview 创建与 CSP 配置 |
| [extension.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/ext-src/extension.ts) | VS Code 扩展主逻辑，消息发送 |
| [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/vscode/src/App.tsx) | VS Code webview React 入口 |
| [content-script.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/chrome-extension/src/content-script.tsx) | Chrome 扩展内容脚本，Worker 禁用逻辑 |
| [vite.config.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/chrome-extension/vite.config.ts) | Chrome 扩展构建配置，banner 注入 |

---

## 9. 核心结论

1. **JSON 解析在主线程同步执行**，使用 `jsonc-parser`，未使用 Worker 卸载

2. **唯一的 Web Worker 是 ELK 布局 Worker**，来自 `reaflow` 依赖库，用于图形布局计算卸载

3. **四个平台的 JSONCrack 组件内部都有布局计算**，但 Chrome 扩展因 CSP 限制禁用 Worker 回退到同步模式

4. **Widget 和 VS Code 的 postMessage 是跨上下文通信**，与内部布局 Worker 是两个独立的概念：
   - Widget：`window.postMessage` 用于 iframe ↔ 父窗口通信
   - VS Code：`webview.postMessage` 用于扩展主机 ↔ webview 通信
   - 两者内部都各自运行着 ELK 布局 Worker

5. **Chrome 扩展采用两层防护**禁用 Worker：
   - 构建时 banner 注入：`var Worker = undefined` 作用于 IIFE 作用域
   - 运行时全局禁用：`globalThis.Worker = undefined` 作用于动态导入模块
   - 导入完成后立即恢复全局 Worker，不影响宿主页面

6. **Worker 生命周期**：
   - 创建：组件首次布局时由 `reaflow` 内部懒加载创建
   - 空闲：布局完成后保持空闲，等待新任务
   - 销毁：组件卸载时由 `reaflow` 内部清理
   - 项目代码中无显式 `worker.terminate()` 调用
