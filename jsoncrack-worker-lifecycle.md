# JSON Crack Web Worker 生命周期分析报告

## 1. 概述

本文档深入分析 JSON Crack 项目中 Web Worker 的使用情况、任务分发机制、CSP 回退策略以及空闲与销毁边界。

### 1.1 核心结论

| 结论 | 说明 |
|------|------|
| **无自定义 JSON 解析 Worker** | JSON 解析在主线程同步执行，使用 `jsonc-parser` |
| **唯一 Worker 用于 ELK 布局** | 来自依赖库 `reaflow` 内部调用 `elkjs`，使用原生 `new Worker()` 创建 |
| **Worker 是模块级单例** | 整个页面共享一个 ELK Worker，多个 Canvas 组件复用同一实例 |
| **组件卸载不终止 Worker** | `useLayout` 清理函数只取消 Promise，**不调用** `elk.terminateWorker()` |
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
│  │             ELK Layout Worker（模块级单例）               │  │
│  │  (elkjs 原生 new Worker()，可选 — Chrome 扩展下被禁用)   │  │
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
elkjs v0.10.2
    ↓ (运行时，原生 Web Worker API)
new Worker(elk-worker.min.js)   ← 注意：不使用 web-worker npm 包
```

**依赖说明**：

| 包 | 角色 |
|----|------|
| `reaflow` | React 流程图库，提供 `Canvas` 组件和 `useLayout` Hook |
| `elkjs` | Eclipse Layout Kernel，布局算法核心，自带 elk-api.js + elk-worker.js |
| **原生 `new Worker()`** | elkjs 的 `workerFactory` 选项直接使用浏览器原生 Web Worker API，**不经过 `web-worker` npm 包** |

### 3.2 Worker 的用途

Web Worker 仅用于 **ELK 图形布局计算**，不涉及 JSON 解析。具体计算任务：

- 节点位置计算（分层布局算法 `layered`）
- 边路径计算（正交连线）
- 布局约束满足（间距、方向等）
- 标签位置优化

### 3.3 任务分发机制

#### 3.3.1 分发入口：JSONCrack 组件 → reaflow Canvas → useLayout → elkLayout

`JSONCrack` 组件本身不直接操作 Worker。完整调用链：

```
JSONCrack 组件
    ↓ (props: nodes, edges)
reaflow <Canvas>
    ↓ (内部调用)
useLayout Hook (reaflow/src/layout/useLayout.ts)
    ↓ (useEffect 中调用)
elkLayout() 函数 (reaflow/src/layout/elkLayout.ts)
    ↓ (调用单例 ELK 实例)
elk.layout(graph)  →  elk-api.js 通过 postMessage 发送给 Worker
    ↓
Worker 中 elk-worker.js 执行布局算法
```

**代码位置**（reaflow 源码片段）：

```typescript
// reaflow/src/layout/useLayout.ts — useEffect 分发布局
useEffect(() => {
    const promise = elkLayout(nodes, edges, {
      'elk.direction': direction,
      ...layoutOptions
    });
    promise
      .then((result) => {
        if (!isEqual(layout, result)) {
          setLayout(result);
          onLayoutChange(result);
        }
      })
      .catch((err) => {
        if (err.name !== 'CancelError') {
          console.error('Layout Error:', err);
        }
      });
    return () => promise.cancel();   // ⚠️ 注意：只 cancel Promise，不 terminate Worker
  }, [nodes, edges]);
```

```typescript
// reaflow/src/layout/elkLayout.ts — 模块级单例 ELK 实例
let elkInstance: ELK | null = null;    // 全局单例，所有 Canvas 共享

const getElk = async () => {
  if (elkInstance) return elkInstance;  // 复用已有实例

  if (!isBrowser) {
    // Node.js 环境：使用同步 bundled 版本
    const ELKModule = await import('elkjs/lib/elk.bundled.js');
    elkInstance = new ELKModule.default({ algorithms: ['layered'] });
    return elkInstance;
  } else {
    // 浏览器环境：使用 elk-api + 原生 Worker
    const ELKModule = await import('elkjs/lib/elk-api');
    elkInstance = new ELKModule.default({
      algorithms: ['layered'],
      workerFactory: () => {
        // ⚠️ 直接使用原生 new Worker()，不依赖 web-worker 包
        const workerUrl = new URL('elkjs/lib/elk-worker.min.js', import.meta.url).href;
        return new Worker(workerUrl);
      }
    });
    return elkInstance;
  }
};
```

#### 3.3.2 Worker 单例复用策略

`elkInstance` 是 `elkLayout.ts` 模块作用域内的变量（闭包单例），具有以下特性：

| 特性 | 说明 |
|------|------|
| **首次创建** | 页面中任何一个 Canvas 组件首次调用 `elkLayout()` 时，通过 `getElk()` 创建 ELK 实例和 Worker |
| **全局共享** | 同一页面中所有 `<Canvas>` 组件共享同一个 ELK 实例和同一个 Worker |
| **跨组件复用** | 如果有多个 JSONCrack 组件（例如多标签页场景），它们会共用同一个 Worker |
| **Node.js 环境** | 使用 `elk.bundled.js`（无 Worker），但同样是单例 |

**重要**：Chrome 扩展禁用 Worker 的机制之所以有效，是因为它在 `import("jsoncrack-react")` 之前就将 `Worker` 置为 `undefined`，导致 `workerFactory` 中 `new Worker()` 检测失败，ELK 自动回退到同步模式。一旦 ELK 实例创建完毕（模式已确定），后续即使恢复 `globalThis.Worker` 也不会改变。

---

## 4. Worker 生命周期边界

> **核心事实（与此前文档错误的修正）**：
> reaflow 源码中的 `useLayout` useEffect 清理函数只调用 `promise.cancel()`，**从未调用** `elk.terminateWorker()`。
> 组件卸载 **不会** 终止 ELK Worker。Worker 作为模块级单例，其生命周期与**整个页面**绑定，而非单个组件。

### 4.1 创建时机

Worker 的创建由 `getElk()` 函数控制，关键触发条件：

1. **首次布局调用**：页面中**任何一个** `<Canvas>` 组件的 `useLayout` useEffect 首次执行时，调用 `elkLayout()` → `getElk()` 创建
2. **懒加载创建**：通过 `await import('elkjs/lib/elk-api')` 动态导入 elk-api.js，随后立即在 `workerFactory` 中 `new Worker(...)` 创建 Worker
3. **单例模式**：创建后写入模块级变量 `elkInstance`，后续所有布局复用此实例

**实际创建条件**（必须同时满足）：
- 浏览器环境：`typeof window !== 'undefined' && typeof Worker !== 'undefined'`
  - 若 `Worker === undefined`（如 Chrome 扩展的 CSP 场景），`isBrowser` 判断为 `false`，回退到同步 bundled 模式
  - 若浏览器原生不支持 Worker，同样回退

### 4.2 空闲状态

布局计算完成后，Worker 进入空闲状态：

- **状态**：Worker 持续存活，等待接收新的 `layout` 消息
- **触发新任务**：
  - 任一 `<Canvas>` 的 `nodes`/`edges` props 变化 → useEffect 重新执行 → 发送新布局任务到**同一 Worker**
  - 折叠/展开节点 → visibleNodes 变化 → 重新布局
  - 布局方向变化（Canvas key 变化触发组件卸载+重新挂载）→ 仍复用同一 Worker
- **任务排队**：若前一布局未完成时发起新布局，`promise.cancel()` 会取消前一个 Promise（仅在主线程层面停止处理结果），但 Worker 中正在执行的计算**不会被中断**。新布局任务会在 Worker 空闲后继续处理。

### 4.3 销毁时机

**Worker 的真实销毁条件**：

| 场景 | Worker 是否被终止 | 原因 |
|------|------------------|------|
| `<Canvas>` 组件卸载 | ❌ **不会** | `useLayout` 清理函数只 `promise.cancel()`，不调用 `elk.terminateWorker()` |
| 多个 Canvas 逐个卸载 | ❌ **不会** | `elkInstance` 是模块变量，组件卸载不影响它 |
| 同一页面内路由切换（React SPA） | ❌ **不一定** | 取决于是否卸载了所有 Canvas 组件 + 是否有代码显式调用 `terminateWorker`（reaflow 没有） |
| **浏览器标签页关闭/刷新/跳转** | ✅ **会** | 浏览器回收整个 JS 上下文，Worker 随之终止 |
| **热更新（HMR）重新加载模块** | ✅ **旧实例会** | 模块重新执行，`elkInstance` 变量重建，但旧 Worker 可能泄漏（取决于 bundler 实现） |

**为什么不终止？** 可能的设计考量：
1. **Worker 创建代价高**：elk-worker.js 是 GWT 编译的 Java→JS 产物，体积大（数百 KB），初始化耗时较长
2. **避免频繁创建销毁**：布局任务是高频操作（节点折叠、数据变化都会触发），复用 Worker 更好
3. **单例全局复用**：如果同一页面有多个 Canvas（多标签编辑器等），共用同一个 Worker 更高效

**但这也带来了潜在问题**：
- 长期运行的 SPA 中，如果用户频繁进入/离开编辑器页面，Worker 会一直占用内存
- Worker 内部可能存在的状态无法被重置
- 没有提供暴露给外部的 Worker 清理接口

### 4.4 各平台的销毁行为差异

#### Web 编辑器（editor 页面）
- 用户切换到其他页面（如 docs、converter、widget）：如果使用了客户端路由且 React 树没有卸载根节点，**Worker 仍存活**
- 用户关闭标签页：Worker 销毁

#### Widget 嵌入页
- Widget 是独立 iframe，其 Worker 只存在于该 iframe 中
- 父页面删除 `<iframe>` 元素 → iframe 上下文销毁 → Worker 销毁

#### VS Code webview
- webview panel 关闭 → VS Code 销毁 webview 整个渲染上下文 → Worker 销毁
- 关闭 panel 时用户侧的清理（`onDidDispose`）仅清理 Node.js 侧监听器，不直接影响 Worker，但 webview 上下文整体销毁时 Worker 必然终止

#### Chrome 扩展（同步模式，无 Worker）
- 用户切换回 "Raw" 模式：`graphRoot.unmount()` → React 组件卸载 → 同步 ELK 实例被 GC 回收
- 但 `elkInstance` 单例变量仍在 content script 模块作用域中，只有整页刷新时才真正重置

### 4.5 生命周期状态机（修正版）

```
        页面加载 + 首次 Canvas 执行布局
                       │
                       ▼
                  ┌───────────┐
                  │  未创建    │  elkInstance = null
                  └─────┬─────┘
                        │ 首次 getElk()
                        ▼
                  ┌───────────┐
                  │  创建中    │  动态 import elk-api
                  │           │  workerFactory → new Worker()
                  └─────┬─────┘
                        │ elkInstance = <ELK实例>
                        ▼
                  ┌───────────┐
            ┌────►│  运行中    │  处理布局任务（postMessage）
            │     └─────┬─────┘
            │           │
  新布局任务 │ 布局完成   │ nodes/edges 变化
            │           ▼
            │     ┌───────────┐
            │     │  空闲中    │  等待消息，持续存活
            │     └───────────┘
            │
            └─── ✅ 仍有其他 Canvas，或同一 Canvas 继续使用

╔══════════════════════════════════════════════════════════════╗
║  ❌ 组件卸载：只 cancel Promise，Worker 不终止                 ║
║      （仍处于空闲状态，elkInstance 仍在模块作用域）            ║
╠══════════════════════════════════════════════════════════════╣
║  ✅ 页面关闭 / 刷新 / iframe 销毁 / webview 销毁：              ║
║      整个 JS 上下文销毁 → Worker 随之终止                      ║
╚══════════════════════════════════════════════════════════════╝
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
- `blob:`：Blob URL 创建的 Worker（elkjs workerFactory 的一种创建方式）
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
1. VS Code 销毁 webview 实例（**整个渲染上下文销毁**）
2. webview 中的 React 组件卸载（此时 `useLayout` 清理函数只 `promise.cancel()`，不 terminate Worker）
3. **webview 的 JS 上下文整体销毁** → `elkInstance` 变量和 Worker 都被浏览器回收（不是显式 terminate，而是上下文销毁）
4. 扩展端清理事件监听器（Node.js 侧）

### 5.4 Chrome 扩展

#### 5.4.1 问题背景：为什么要禁用 Worker

Chrome 扩展的内容脚本（Content Script）注入到第三方 JSON 页面中运行。许多 JSON 查看页面带有严格的 CSP（如 GitHub Raw 的 `default-src 'none'`），如果布局引擎尝试创建 Web Worker，会因 CSP 拦截而抛错导致渲染失败。

因此项目设计了**两层 Worker 禁用机制**，强制 ELK 回退到同步布局路径。

---

#### 5.4.2 第一层：构建时 Banner 注入（IIFE 作用域遮蔽）

**代码位置**：[vite.config.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/chrome-extension/vite.config.ts#L23-L28)

```typescript
rollupOptions: {
  output: {
    banner:
      "var Worker = undefined; var process = globalThis.process || (globalThis.process = { env: { NODE_ENV: 'production' } });",
  },
},
```

**构建产物结构**：
```javascript
// vite 打包为 IIFE 格式，banner 被注入函数体最开头
(function() {
  var Worker = undefined;  // ← banner 注入，IIFE 函数作用域内的局部变量
  var process = ...;
  // ... content-script.tsx 的所有代码 ...
})();
```

**作用域分析**：
- `var Worker = undefined` 声明在 **IIFE 函数作用域** 内
- 它只遮蔽 content script **自身代码** 中直接出现的 `Worker` 标识符
- 对于**动态导入**的模块（`jsoncrack-react`、`reaflow`、`elkjs`）无效——这些模块运行在自己的模块作用域，查找 `Worker` 时走全局作用域链，不会经过 IIFE 的局部变量
- 因此这一层是**兜底**，真正决定分支的是第二层

---

#### 5.4.3 第二层：运行时全局禁用（决定实际分支的关键）

**代码位置**：[content-script.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/chrome-extension/src/content-script.tsx#L28-L50)

```typescript
const loadJsonCrackComponent = async (): Promise<JSONCrackComponentType> => {
  if (jsonCrackComponentPromise) {
    return jsonCrackComponentPromise;
  }

  jsonCrackComponentPromise = (async () => {
    // 保存原始全局 Worker
    const originalWorker = (globalThis as { Worker?: typeof Worker }).Worker;
    try {
      // 关键：将全局 Worker 置为 undefined
      (globalThis as { Worker?: typeof Worker }).Worker = undefined;
      const mod = await import("jsoncrack-react");  // 动态导入期间 Worker 被禁用
      return mod.JSONCrack as JSONCrackComponentType;
    } finally {
      // 导入完成后立即恢复全局 Worker，避免影响宿主页面
      (globalThis as { Worker?: typeof Worker }).Worker = originalWorker;
    }
  })();

  return jsonCrackComponentPromise;
};
```

---

#### 5.4.4 实际分支选择：isBrowser 判断精确时机与结果

**reaflow 中的分支判断代码**（[elkLayout.ts](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/node_modules/reaflow/src/layout/elkLayout.ts) 顶层作用域）：

```typescript
// 模块顶层作用域，模块加载时立即求值
const isBrowser = typeof window !== 'undefined' && typeof Worker !== 'undefined';
```

**精确时间轴与求值结果**：

```
时间轴 →
  │
  ├─ content script 开始执行
  │   (banner 注入 var Worker = undefined 在 IIFE 作用域)
  │
  ├─ 用户首次点击 "Graph" 按钮
  │   └─ GraphView 组件挂载 → useEffect 调用 loadJsonCrackComponent()
  │       │
  │       ├─ 保存 originalWorker = window.Worker （真实的 Worker 构造函数）
  │       ├─ globalThis.Worker = undefined   ──┐
  │       │                                      │ 【禁用窗口】
  │       ├─ await import("jsoncrack-react")    │
  │       │   → 递归导入 reaflow                │
  │       │     → 递归导入 elkjs 相关模块        │
  │       │       → elkLayout.ts 模块加载        │
  │       │         → 顶层 isBrowser 立即求值     │
  │       │             typeof window → 'object' ✅
  │       │             typeof Worker → 'undefined' ❌  ← 关键！此时全局 Worker 被遮蔽
  │       │             isBrowser = false        │
  │       │         → getElk() 走同步分支       │
  │       │             import('elkjs/lib/elk.bundled.js')
  │       │                                      │
  │       └─ globalThis.Worker = originalWorker ──┘ （导入完成，恢复全局 Worker）
  │           （此时 isBrowser 早已求值完毕，恢复 Worker 不影响已固化的模式）
  │
  ├─ 首次布局
  │   └─ getElk() 返回同步 ELK 实例（elk.bundled.js）
  │       layout() 在主线程同步计算
  │
  ├─ 用户切换 Raw → Graph
  │   └─ elkInstance 已存在，直接复用同步实例
  │
  ▼
```

**最终分支选择结果**：

| 分支条件 | 值 | 原因 |
|---------|----|------|
| `typeof window` | `'object'` | 始终为 true，content script 运行在页面上下文中 |
| `typeof Worker` | `'undefined'` | 在禁用窗口内读取全局作用域，`globalThis.Worker` 被置为 `undefined` |
| `isBrowser` | `false` | 两个条件需同时满足，Worker 不满足 |
| **实际分支** | **`elkjs/lib/elk.bundled.js`（同步模式）** | 不走 Worker 路径 |

---

#### 5.4.5 elk.bundled.js 同步模式的真实行为

`elk.bundled.js` 与 `elk-api.js` 的区别：

| 特性 | `elk.bundled.js`（同步分支） | `elk-api.js`（Worker 分支） |
|------|---------------------------|------------------------|
| Worker | 不使用，算法直接嵌入主线程 | 通过 `workerFactory` 创建 Worker |
| layout() 返回类型 | Promise（但同步完成后立即 resolve） | Promise（Worker 计算完成后 resolve） |
| 布局计算位置 | **主线程**，阻塞 UI | Worker 线程，不阻塞 UI |
| 初始化体积 | GWT 编译的完整算法代码，较大 | 只是 API 包装，算法在 Worker 脚本中 |

**注意**：即使是同步模式，`elk.layout()` 的返回类型仍为 Promise——这是为了与 Worker 模式保持 API 兼容。布局计算在 `.then()` 回调前已经同步完成，但仍通过微任务队列 resolve Promise。

---

#### 5.4.6 两层防护的作用域对比

| 层级 | 代码 | 作用域 | 影响范围 | 是否决定分支 |
|------|------|--------|---------|------------|
| 构建时 banner | `var Worker = undefined` | IIFE 函数作用域 | content script 自身代码中直接的 `Worker` 引用 | ❌ 不是，动态导入模块不受影响 |
| 运行时全局 | `globalThis.Worker = undefined` | 全局作用域（隔离世界） | 动态导入的所有模块（jsoncrack-react、reaflow、elkjs） | ✅ **是**，决定 isBrowser = false |

**为什么两层都需要？**
- 第一层 banner 是**兜底防御**：防止 content script 自身代码某处直接写了 `new Worker()`
- 第二层全局禁用是**核心机制**：确保动态导入的依赖模块在初始化时检测不到 Worker

---

#### 5.4.7 模式固化：恢复 Worker 不会影响已确定的分支

`isBrowser` 是在 `elkLayout.ts` 模块顶层的 `const` 常量，**模块加载时求值一次，之后永不改变**。即使 `finally` 块立即恢复了 `globalThis.Worker`，后续所有布局调用仍然使用已确定的同步模式。

这是设计有意为之：
1. 导入完成后尽快恢复全局 Worker，避免干扰宿主页面的 Worker
2. ELK 模式只需在模块加载时确定一次，后续无需重新判断

---

#### 5.4.8 同步 ELK 单例的保留与释放

##### 单例变量位置

```typescript
// elkLayout.ts 模块作用域
let elkInstance: ELK | null = null;  // 同步模式下存储的是 elk.bundled.js 实例
```

Content script 运行在 Chrome 的**隔离世界（Isolated World）**中，拥有独立的 JS 执行上下文（独立的全局对象、独立的模块注册表），与宿主页面的 JS 上下文隔离。

---

##### 组件卸载（切换 Raw 模式）

**代码位置**：[content-script.tsx](file:///d:/fz/0601/solo-dogfeeding/code/184-jsoncrack.com/apps/chrome-extension/src/content-script.tsx#L149-L154)

```typescript
const unmountGraphView = () => {
  if (!graphRoot) return;
  graphRoot.unmount();      // React 卸载组件树
  graphRoot = null;
  reactRootContainer.textContent = "";
};
```

**切换 Raw 模式时发生什么**：

| 步骤 | 对象 | 是否被清理 | 说明 |
|------|------|----------|------|
| 1 | React 组件树 | ✅ 是 | `graphRoot.unmount()` 卸载 `<GraphView>` → `<JSONCrack>` → `<Canvas>` |
| 2 | useLayout 的 Promise | ✅ 是 | useEffect 清理函数执行 `promise.cancel()` |
| 3 | **`elkInstance` 单例变量** | ❌ **否** | 它属于 `elkLayout.ts` 模块作用域，模块仍在内存中 |
| 4 | 同步 ELK 实例内部的算法对象 | ❌ 否 | 被 `elkInstance` 引用，GC 不会回收 |
| 5 | DOM 容器 | ✅ 是 | `textContent = ""` 清空 |
| 6 | `jsonCrackComponentPromise` | ❌ **否** | content script 模块级变量，保留已加载的组件 |

**关键结论**：组件卸载只是卸载 React UI，**不会**释放同步 ELK 单例。用户可以在 Raw / Graph 之间反复切换，每次重新挂载时都会复用已有的 `elkInstance`，无需重新初始化 ELK 引擎。

---

##### 单例真正释放的时机

**只有以下情况会释放同步 ELK 单例**：

| 触发场景 | 发生了什么 | elkInstance 是否释放 |
|---------|----------|-------------------|
| **宿主页面刷新**（F5） | 页面重新加载，content script 被 Chrome 重新注入到新的隔离世界中，模块重新执行 | ✅ 是（旧隔离世界销毁） |
| **宿主页面导航**（点击链接跳转） | 同上，新页面的 content script 是全新的隔离世界 | ✅ 是（旧隔离世界销毁） |
| **Chrome 扩展被禁用/重新启用** | content script 上下文被销毁并重建 | ✅ 是 |
| **仅关闭 Graph 视图（切回 Raw）** | React 卸载，但 content script 仍在页面中运行 | ❌ 否 |
| **切换标签页再切回来** | content script 不受影响 | ❌ 否 |

---

##### jsonCrackComponentPromise 的保留策略

与 `elkInstance` 类似，`jsonCrackComponentPromise` 也是 content script 模块级变量：

```typescript
// content-script.tsx 顶层
let jsonCrackComponentPromise: Promise<JSONCrackComponentType> | null = null;
```

这意味着：
- 首次加载时动态 import 的 `jsoncrack-react` 模块会被缓存
- 切换 Raw → Graph 多次，不会重复执行动态 import，也不会重新创建 ELK 实例
- 整个单例链条（`jsonCrackComponentPromise` → `jsoncrack-react` 模块 → `reaflow` 模块 → `elkInstance`）在隔离世界生命周期内持续存在

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

2. **唯一的 Web Worker 是 ELK 布局 Worker**：
   - 由 `reaflow` 内部的 `elkLayout()` 函数通过 `elkjs` 创建
   - 使用原生 `new Worker()` API，**不依赖 `web-worker` npm 包**
   - Worker 脚本为 `elkjs/lib/elk-worker.min.js`（GWT 编译的 Java→JS 产物）

3. **Worker 是模块级单例**：
   - `let elkInstance: ELK | null = null` 定义在 `elkLayout.ts` 模块作用域中
   - 同一页面中所有 `<Canvas>` 组件共享同一个 ELK 实例和同一个 Worker
   - 跨组件复用，即使卸载了所有 Canvas 组件，Worker 仍存活

4. **组件卸载 ❌ 不会终止 Worker**（此前文档的关键错误）：
   - `useLayout` 的 useEffect 清理函数只调用 `promise.cancel()`
   - **从未调用** `elk.terminateWorker()`（elkjs 提供了此 API 但 reaflow 未使用）
   - Worker 只有在**整个 JS 上下文销毁**时才会终止（标签页关闭/刷新、iframe 移除、webview 销毁）

5. **四个平台的 JSONCrack 组件内部都有布局计算**，但 Chrome 扩展因 CSP 限制禁用 Worker 回退到同步模式

6. **Widget 和 VS Code 的 postMessage 是跨上下文通信**，与内部布局 Worker 是两个独立的概念：
   - Widget：`window.postMessage` 用于 iframe ↔ 父窗口通信
   - VS Code：`webview.postMessage` 用于扩展主机 ↔ webview 通信
   - 两者内部都各自运行着 ELK 布局 Worker

7. **Chrome 扩展两层禁用机制的精确分工**：
   - **第一层（构建时 banner）**：`var Worker = undefined` 在 IIFE 函数作用域内，只遮蔽 content script 自身代码的直接 `Worker` 引用 → **兜底，不决定实际分支**
   - **第二层（运行时全局）**：`globalThis.Worker = undefined` 在隔离世界的全局作用域，动态导入的 `jsoncrack-react` → `reaflow` → `elkjs` 都从全局读取 `Worker` → **核心，决定 isBrowser = false**
   - **判断时机**：`const isBrowser = typeof Worker !== 'undefined'` 在 `elkLayout.ts` 模块顶层，**import 执行期间立即求值**，此时正好处于禁用窗口
   - **实际分支**：`import('elkjs/lib/elk.bundled.js')`，同步模式，算法在主线程执行
   - **模式固化**：`isBrowser` 是 `const`，求值一次后永不改变；`finally` 恢复全局 Worker 不影响已确定的同步模式

8. **Chrome 扩展同步 ELK 单例的保留与释放**：
   - `elkInstance` 变量位于 `elkLayout.ts` 模块作用域，属于 content script 的隔离世界（Isolated World）
   - **切换 Raw/Graph（组件卸载/挂载）**：只 React unmount，`elkInstance` **不释放**，反复切换都复用同一实例
   - **真正释放时机**：只有宿主页面**刷新**（F5）、**导航跳转**、或扩展被**禁用/重新启用**，导致 content script 的隔离世界被销毁并重建时，模块才重新执行，单例才释放
   - `jsonCrackComponentPromise` 同理，也是 content script 模块级变量，在隔离世界生命周期内持续缓存

9. **Worker 生命周期（修正后）**：
   - ✅ **创建**：首次调用 `elkLayout()` 时通过 `getElk()` 懒加载创建
   - ✅ **复用**：模块级单例，所有布局任务都发往同一个 Worker
   - ✅ **空闲**：布局完成后持续存活，等待新的 `postMessage`
   - ❌ **组件卸载**：只 `promise.cancel()`，Worker **不终止**，同步 ELK 实例也**不释放**
   - ✅ **销毁**：页面关闭 / 上下文销毁（不是显式 terminate）
   - 项目代码 + reaflow 源码中 **均无** 显式 `worker.terminate()` 调用
