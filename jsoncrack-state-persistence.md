# JSON Crack 全局状态与持久化实现分析

## 一、全局状态架构概览

JSON Crack 使用 **Zustand** 作为状态管理库，共定义了 5 个全局 store，分布在两个目录中：

### 1.1 核心 Stores（`apps/www/src/store/`）

| Store | 文件 | 用途 | 持久化方式 |
|-------|------|------|-----------|
| useConfig | [useConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/store/useConfig.ts) | 用户配置（暗色模式、实时转换、手势、标尺） | localStorage（zustand persist） |
| useFile | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/store/useFile.ts) | 文件内容与格式 | sessionStorage（手动管理） |
| useJson | [useJson.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/store/useJson.ts) | 解析后的 JSON 数据 | 无（内存状态） |
| useModal | [useModal.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/store/useModal.ts) | 模态框显示状态 | 无（内存状态） |

### 1.2 视图 Store（`apps/www/src/features/editor/views/GraphView/stores/`）

| Store | 文件 | 用途 | 持久化方式 |
|-------|------|------|-----------|
| useGraph | [useGraph.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/features/editor/views/GraphView/stores/useGraph.ts) | 图形视图状态（视口、方向、全屏等） | 无（内存状态） |

---

## 二、状态存储机制

### 2.1 localStorage 持久化

#### 2.1.1 useConfig - Zustand Persist 中间件

**文件**: [useConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/store/useConfig.ts#L1-L33)

使用 `zustand/middleware` 的 `persist` 中间件实现自动持久化：

```typescript
const useConfig = create(
  persist<typeof initialStates & ConfigActions>(
    set => ({
      ...initialStates,
      toggleRulers: rulersEnabled => set({ rulersEnabled }),
      toggleGestures: gesturesEnabled => set({ gesturesEnabled }),
      toggleLiveTransform: liveTransformEnabled => set({ liveTransformEnabled }),
      toggleDarkMode: darkmodeEnabled => set({ darkmodeEnabled }),
    }),
    {
      name: "config",  // localStorage key
    }
  )
);
```

**存储的状态**:
- `darkmodeEnabled`: 暗色模式开关（默认 `true`）
- `liveTransformEnabled`: 实时转换开关（默认 `true`）
- `gesturesEnabled`: 手势开关（默认 `false`）
- `rulersEnabled`: 标尺开关（默认 `true`）

**特点**:
- Zustand persist 自动处理读写
- 状态变更时自动同步到 localStorage
- 页面加载时自动从 localStorage 恢复

#### 2.1.2 Mantine 颜色方案 - 自定义管理器

**文件**: [mantineColorScheme.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/lib/utils/mantineColorScheme.ts#L1-L76)

自定义 `smartColorSchemeManager` 实现 Mantine 主题的持久化：

```typescript
export function smartColorSchemeManager({
  key,           // localStorage key: "editor-color-scheme"
  getPathname,   // 获取当前路径的函数
  dynamicPaths,  // 需要动态主题的路径列表
}: SmartColorSchemeManagerOptions): MantineColorSchemeManager
```

**存储逻辑**:
- 仅在 `/editor` 和 `/widget` 路径下使用动态主题（持久化）
- 其他路径强制使用 `light` 主题（不持久化）
- 使用内存变量 `currentColorScheme` 作为缓存，优先从内存读取

**相关代码**:
- `get()`: 读取主题，动态路径下优先内存，再读 localStorage
- `set(value)`: 设置主题，动态路径下同时更新内存和 localStorage
- `subscribe/unsubscribe`: 空实现（不支持订阅）
- `clear()`: 清除内存和 localStorage 中的值

**使用位置**: [_app.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/pages/_app.tsx#L73-L78)

```typescript
const colorSchemeManager = smartColorSchemeManager({
  key: "editor-color-scheme",
  getPathname: () => pathname,
  dynamicPaths: ["/editor", "/widget"],
});
```

---

### 2.2 sessionStorage 持久化

#### 2.2.1 文件内容 - useFile 手动管理

**文件**: [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/store/useFile.ts#L100-L126)

在 `setContents` 方法中手动写入 sessionStorage：

```typescript
setContents: async ({ contents, hasChanges = true, skipUpdate = false, format }) => {
  // ...
  if (get().hasChanges && contents && contents.length < 80_000 && !isIframe() && !isFetchURL) {
    sessionStorage.setItem("content", contents);
    sessionStorage.setItem("format", get().format);
    set({ hasChanges: true });
  }
  // ...
}
```

**存储的状态**:
- `content`: 文件内容字符串
- `format`: 文件格式（JSON/YAML/XML/CSV）

**写入条件**（需同时满足）:
1. `hasChanges` 为 `true`（有未保存的更改）
2. `contents` 存在且长度 < 80,000 字符
3. 不在 iframe 中（`!isIframe()`）
4. URL 不包含查询参数（`!isFetchURL`）

#### 2.2.2 视图模式 - Mantine Hook

**文件**: 
- [LiveEditor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/features/editor/LiveEditor.tsx#L30-L39)
- [ViewMenu.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/features/editor/Toolbar/ViewMenu.tsx#L9-L12)

使用 `@mantine/hooks` 的 `useSessionStorage` hook：

```typescript
const [viewMode, setViewMode] = useSessionStorage({
  key: "viewMode",
  defaultValue: ViewMode.Graph,
});
```

**存储的状态**:
- `viewMode`: 视图模式（`graph` 或 `tree`）

---

### 2.3 内存状态（无持久化）

以下 store 仅存在于内存中，页面刷新后重置为初始值：

| Store | 状态 | 说明 |
|-------|------|------|
| useJson | `json`, `loading` | 解析后的 JSON 字符串和加载状态 |
| useModal | 各模态框显示状态 | 所有模态框的可见性 |
| useGraph | `viewPort`, `direction`, `fullscreen`, `selectedNode`, `collapsedCount`, `jsonCrackRef` | 图形视图的运行时状态 |

---

## 三、状态恢复流程

### 3.1 Editor 页面初始化流程

**入口文件**: [editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/pages/editor.tsx#L115-L121)

```typescript
useEffect(() => {
  if (isReady) checkEditorSession(query?.json);
}, [checkEditorSession, isReady, query]);

useEffect(() => {
  setColorScheme(darkmodeEnabled ? "dark" : "light");
}, [darkmodeEnabled, setColorScheme]);
```

### 3.2 checkEditorSession 详细流程

**文件**: [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/store/useFile.ts#L142-L154)

```typescript
checkEditorSession: (url, widget) => {
  // 步骤1: 优先从 URL 参数加载
  if (url && typeof url === "string" && isURL(url)) {
    return get().fetchUrl(url);
  }

  // 步骤2: 从 sessionStorage 恢复
  let contents = defaultJson;
  const sessionContent = sessionStorage.getItem("content") as string | null;
  const format = sessionStorage.getItem("format") as FileFormat | null;
  if (sessionContent && !widget) contents = sessionContent;

  // 步骤3: 设置到 store
  if (format) set({ format });
  get().setContents({ contents, hasChanges: false });
}
```

**恢复优先级**:
1. **URL 参数** (`?json=...`): 如果 URL 中包含有效的 JSON URL，优先从远程获取
2. **sessionStorage**: 如果存在会话存储的内容，从 sessionStorage 恢复（widget 模式下不恢复）
3. **默认示例**: 使用内置的 example.json 作为默认内容

### 3.3 Widget 页面初始化流程

**入口文件**: [widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/pages/widget.tsx#L47-L75)

```typescript
// 步骤1: 从 URL 参数或 postMessage 加载内容
useEffect(() => {
  if (isReady) {
    if (typeof query?.json === "string") checkEditorSession(query.json, true);
    else clearJson();
    window.parent.postMessage(window.frameElement?.getAttribute("id"), "*");
  }
}, [...]);

// 步骤2: 监听父窗口的 postMessage 动态更新
useEffect(() => {
  const handler = (event: EmbedMessage) => {
    if (!event.data?.json) return;
    // 更新主题
    if (event.data?.options?.theme === "light" || event.data?.options?.theme === "dark") {
      setTheme(event.data.options.theme);
      toggleDarkMode(event.data.options.theme === "dark");
    }
    // 更新内容和布局方向
    setContents({ contents: event.data.json, hasChanges: false });
    setDirection(event.data.options?.direction || "RIGHT");
  };
  window.addEventListener("message", handler);
  return () => window.removeEventListener("message", handler);
}, [...]);
```

**Widget 模式特点**:
- 不使用 sessionStorage 恢复内容（`widget=true` 参数）
- 通过 `postMessage` API 与父窗口通信
- 支持动态接收 JSON 数据、主题和布局方向配置

### 3.4 主题恢复的双重机制

主题状态存在两套独立的持久化机制，可能存在同步问题：

1. **useConfig.darkmodeEnabled** (localStorage: "config")
   - 由 Zustand persist 管理
   - 通过 `toggleDarkMode` 更新

2. **Mantine colorScheme** (localStorage: "editor-color-scheme")
   - 由 `smartColorSchemeManager` 管理
   - 通过 `setColorScheme` 更新

**同步逻辑**: 在 editor 页面的 useEffect 中，以 useConfig 为准同步到 Mantine：

```typescript
useEffect(() => {
  setColorScheme(darkmodeEnabled ? "dark" : "light");
}, [darkmodeEnabled, setColorScheme]);
```

---

## 四、跨页面/标签页同步

### 4.1 现状：无跨标签页实时同步

经过代码分析，JSON Crack **没有实现**跨浏览器标签页的实时状态同步机制：

- ❌ 未使用 `BroadcastChannel` API
- ❌ 未监听 `storage` 事件（`window.addEventListener('storage', ...)`）
- ❌ 未使用 `SharedWorker`
- ❌ 未使用 IndexedDB

### 4.2 多标签页无实时同步的边界条件

#### 4.2.1 存储层面的边界

| 存储类型 | 跨标签页共享 | 自动同步 | 同步边界 |
|---------|-------------|---------|---------|
| localStorage | ✅ 是 | ❌ 否 | 数据在磁盘层面共享，但无事件通知机制 |
| sessionStorage | ❌ 否 | ❌ 否 | 每个标签页完全独立，数据不共享 |

**localStorage 的隐式共享但不同步**:
- localStorage 本身是同源共享的，多个标签页读取同一个物理存储
- 但由于没有监听 `storage` 事件，标签页 A 的更改不会实时推送到标签页 B
- 标签页 B 只有在**重新读取 localStorage 时**（如页面刷新、store 重新初始化）才能看到最新值
- 示例：标签页 A 切换暗色模式 → localStorage 更新 → 标签页 B 仍显示亮色 → 刷新标签页 B 后才变为暗色

#### 4.2.2 Zustand persist 的同步边界

**文件**: [useConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/store/useConfig.ts#L18-L31)

Zustand 的 `persist` 中间件默认不支持跨标签页同步：
- 仅在 store 创建时从 localStorage 读取一次
- 状态变更时写入 localStorage，但不会通知其他标签页
- 其他标签页的 store 内存状态保持不变，直到重新初始化

#### 4.2.3 Mantine 主题管理器的同步边界

**文件**: [mantineColorScheme.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/lib/utils/mantineColorScheme.ts#L66-L68)

`smartColorSchemeManager` 的 `subscribe` 和 `unsubscribe` 是空实现：

```typescript
return {
  // ...
  // These do nothing regardless of path
  subscribe: () => {},
  unsubscribe: () => {},
  // ...
};
```

这意味着：
- Mantine 无法感知其他标签页的主题变更
- 内存缓存 `currentColorScheme` 会保持旧值，即使 localStorage 已被其他标签页修改
- 只有重新调用 `get()` 时才会读取最新的 localStorage 值

#### 4.2.4 sessionStorage 的天然隔离

**文件**: [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/store/useFile.ts#L114-L117)

sessionStorage 是浏览器级别的标签页隔离机制：
- 每个标签页拥有独立的 sessionStorage 存储空间
- 即使是同一个 URL 打开的多个标签页，sessionStorage 也不共享
- 标签页关闭后 sessionStorage 自动清除
- 这意味着：在标签页 A 编辑的内容，标签页 B 完全看不到

### 4.3 Widget 嵌入页面与父页面的消息往返

#### 4.3.1 完整消息时序图

```
父页面（包含 iframe）                     Widget 页面（iframe 内）
     |                                          |
     |  1. 创建 iframe，src=/widget             |
     |----------------------------------------->|
     |                                          |  2. Widget 初始化完成
     |                                          |  3. 发送 ready 信号
     |<-----------------------------------------|
     |      postMessage(iframe.id, "*")         |
     |                                          |
     |  4. 父页面收到 ready 信号                |
     |  5. 下发 JSON 数据和配置                 |
     |----------------------------------------->|
     |      postMessage({json, options}, "*")   |
     |                                          |  6. Widget 接收并更新状态
     |                                          |  7. 渲染图形
```

#### 4.3.2 Widget 初始化与 ready 信号

**文件**: [widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/pages/widget.tsx#L47-L54)

```typescript
React.useEffect(() => {
  if (isReady) {
    if (typeof query?.json === "string") checkEditorSession(query.json, true);
    else clearJson();

    // 发送 ready 信号给父窗口
    window.parent.postMessage(window.frameElement?.getAttribute("id"), "*");
  }
}, [checkEditorSession, clearJson, isReady, push, query.json, query.partner]);
```

**关键细节**:
- Widget 加载完成后，将 iframe 的 `id` 属性作为消息内容发送给父窗口
- 父页面通过匹配此 id 来识别哪个 Widget 已就绪
- 这是**子→父**的唯一消息（其他消息都是**父→子**）

#### 4.3.3 父页面监听与消息下发

**文档示例**: [docs.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/pages/docs.tsx#L44-L58)

```typescript
// 父页面代码
const iframe = document.getElementById("json-crack-embed");

// 等待 Widget 发出 ready 信号
window.addEventListener("message", (event) => {
  if (event.data === "json-crack-embed") {
    // Widget 已就绪，发送数据
    iframe.contentWindow.postMessage({
      json: JSON.stringify({ hello: "world" }),
      options: {
        theme: "light",
        direction: "DOWN"
      }
    }, "*");
  }
});
```

**消息格式定义**:

```typescript
interface EmbedMessage {
  data: {
    json?: string;           // JSON 字符串
    options?: {
      theme?: "light" | "dark";      // 主题
      direction?: LayoutDirection;   // 布局方向: "RIGHT" | "DOWN" | "LEFT" | "UP"
    };
  };
}
```

### 4.4 内容下发的完整链路

**文件**: [widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/pages/widget.tsx#L56-L75)

```typescript
React.useEffect(() => {
  const handler = (event: EmbedMessage) => {
    try {
      if (!event.data?.json) return;
      
      // 1. 内容更新
      setContents({ contents: event.data.json, hasChanges: false });
      
      // 2. 布局方向更新
      setDirection(event.data.options?.direction || "RIGHT");
      
      // 3. 主题更新（见下文）
      if (event.data?.options?.theme === "light" || event.data?.options?.theme === "dark") {
        setTheme(event.data.options.theme);
        toggleDarkMode(event.data.options.theme === "dark");
      }
    } catch (error) {
      console.error(error);
      toast.error("Invalid JSON!");
    }
  };

  window.addEventListener("message", handler);
  return () => window.removeEventListener("message", handler);
}, [setColorScheme, setContents, setDirection, toggleDarkMode, theme]);
```

**内容下发链路**:
```
父窗口 postMessage({ json, options })
    ↓
Widget message 事件触发
    ↓
┌─ handler 函数 ─────────────────────────────┐
│  1. 校验 event.data.json 存在               │
│  2. 调用 useFile.setContents()             │
│     ├─ 更新内存状态 contents                │
│     ├─ hasChanges 设为 false（不持久化）    │
│     └─ 防抖调用 useJson.setJson()           │
│  3. 调用 useGraph.setDirection()            │
│  4. 主题更新（见下文）                      │
└─────────────────────────────────────────────┘
    ↓
GraphView 重新渲染
```

**关键点**: 
- 父页面下发的内容 `hasChanges` 被强制设为 `false`，**不会写入 sessionStorage**
- Widget 模式下禁用了 sessionStorage 恢复，确保内容完全由父页面控制

### 4.5 主题下发的完整链路

主题下发涉及**三层状态同步**，是最复杂的部分：

#### 4.5.1 三层主题状态

| 层级 | 状态 | 存储位置 | 管理方 |
|------|------|---------|-------|
| 1 | `theme` (local state) | 内存 | Widget 组件 useState |
| 2 | `darkmodeEnabled` | localStorage "config" | useConfig (Zustand) |
| 3 | `colorScheme` | localStorage "editor-color-scheme" | Mantine 管理器 |

#### 4.5.2 主题下发完整流程

```
父窗口 postMessage({ options: { theme: "dark" } })
    ↓
Widget message 事件触发
    ↓
┌─ handler 函数 ─────────────────────────────────┐
│  1. setTheme("dark")                            │
│     └─ 更新本地 useState → 触发 useEffect       │
│  2. toggleDarkMode(true)                        │
│     └─ useConfig 更新                           │
│        ├─ 更新内存 darkmodeEnabled = true       │
│        └─ Zustand persist 写入 localStorage    │
│           key: "config"                         │
└─────────────────────────────────────────────────┘
    ↓
useEffect 触发（theme 依赖）
    ↓
┌─ [widget.tsx L77-L79] ───────────────────────┐
│  setColorScheme(theme)                        │
│  └─ smartColorSchemeManager.set("dark")       │
│     ├─ 更新内存 currentColorScheme = "dark"   │
│     └─ 写入 localStorage                      │
│        key: "editor-color-scheme"             │
└───────────────────────────────────────────────┘
    ↓
MantineProvider 主题更新
    ↓
ThemeProvider (styled-components) 主题更新
    ↓
UI 重新渲染
```

**代码实现**:

```typescript
// 步骤1: message handler 中更新
if (event.data?.options?.theme === "light" || event.data?.options?.theme === "dark") {
  setTheme(event.data.options.theme);              // 更新本地 state
  toggleDarkMode(event.data.options.theme === "dark");  // 更新 useConfig
}

// 步骤2: useEffect 同步到 Mantine
React.useEffect(() => {
  setColorScheme(theme);
}, [setColorScheme, theme]);

// 步骤3: ThemeProvider 消费
<ThemeProvider theme={theme === "dark" ? darkTheme : lightTheme}>
```

#### 4.5.3 主题同步的潜在问题

1. **三层状态可能不一致**:
   - 如果 `setColorScheme` 失败，本地 `theme` state 和 `darkmodeEnabled` 已更新，但 Mantine 主题未变
   - 如果 Widget 在非动态路径（虽然实际只在 /widget 和 /editor 下运行），`setColorScheme` 会静默失败

2. **localStorage 写入但无反向同步**:
   - `toggleDarkMode` 会写入 localStorage "config"
   - `setColorScheme` 会写入 localStorage "editor-color-scheme"
   - 但父页面无法感知这些变化（postMessage 是单向的）

3. **内存缓存导致的过期值**:
   - `smartColorSchemeManager` 的 `currentColorScheme` 内存缓存
   - 如果其他代码直接修改 localStorage，Mantine 不会感知
   - 必须通过 `setColorScheme` API 更新才能保证一致性

---

## 五、数据流与状态更新流程

### 5.1 文件内容更新流程

```
用户编辑文本
    ↓
TextEditor 调用 setContents()
    ↓
┌─ useFile.setContents() ──────────────────┐
│  1. 更新内存状态 (contents, error, format) │
│  2. 满足条件时写入 sessionStorage          │
│  3. 防抖调用 useJson.setJson()             │
└───────────────────────────────────────────┘
    ↓
useJson 更新 (json, loading=false)
    ↓
GraphView/TreeView 重新渲染
```

**防抖处理**: 使用 `lodash.debounce` 延迟 400ms 更新 JSON 解析结果，避免频繁重渲染。

### 5.2 配置更新流程

```
用户切换开关（如暗色模式）
    ↓
useConfig.toggleDarkMode(value)
    ↓
┌─ Zustand persist 中间件 ────────┐
│  1. 更新内存状态                 │
│  2. 自动同步到 localStorage      │
└──────────────────────────────────┘
    ↓
相关组件重新渲染（如 ThemeProvider）
```

---

## 六、关键代码索引

### 6.1 Store 文件

- [useConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/store/useConfig.ts) - 配置状态（localStorage 持久化）
- [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/store/useFile.ts) - 文件状态（sessionStorage 持久化）
- [useJson.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/store/useJson.ts) - JSON 数据状态
- [useModal.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/store/useModal.ts) - 模态框状态
- [useGraph.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/features/editor/views/GraphView/stores/useGraph.ts) - 图形视图状态

### 6.2 持久化工具

- [mantineColorScheme.ts](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/lib/utils/mantineColorScheme.ts) - Mantine 主题持久化管理器

### 6.3 页面入口

- [editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/pages/editor.tsx) - 编辑器页面
- [widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/pages/widget.tsx) - Widget 嵌入页面
- [docs.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/pages/docs.tsx) - 嵌入文档（含 postMessage 示例）
- [_app.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/pages/_app.tsx) - 应用根组件

### 6.4 UI 组件

- [ThemeToggle.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/features/editor/Toolbar/ThemeToggle.tsx) - 主题切换按钮
- [ViewMenu.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/features/editor/Toolbar/ViewMenu.tsx) - 视图模式切换
- [LiveEditor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/190-jsoncrack.com/apps/www/src/features/editor/LiveEditor.tsx) - 实时编辑器视图

---

## 七、总结与特点

### 7.1 持久化策略总结

| 状态类别 | 存储介质 | 管理方式 | 生命周期 |
|---------|---------|---------|---------|
| 用户偏好配置 | localStorage | Zustand persist | 永久（手动清除） |
| Mantine 主题 | localStorage | 自定义管理器 | 永久（手动清除） |
| 编辑内容 | sessionStorage | 手动管理 | 会话级（标签页关闭即失） |
| 视图模式 | sessionStorage | Mantine hook | 会话级 |
| 运行时状态 | 内存 | Zustand | 页面级 |

### 7.2 设计特点

1. **分层持久化**: 用户偏好使用 localStorage（长期保存），编辑内容使用 sessionStorage（会话级）
2. **条件持久化**: 文件内容仅在满足大小限制、非 iframe、非 URL 加载等条件下才持久化
3. **双重主题管理**: Zustand store 和 Mantine 管理器两套主题系统，需手动同步
4. **无跨标签页实时同步**: 依赖浏览器原生存储特性，不主动实现多标签页状态同步
5. **Widget 特殊处理**: 嵌入模式下禁用 sessionStorage 恢复，通过 postMessage 与父窗口通信
6. **postMessage 单向通信**: Widget 与父页面的通信是单向的（父→子），仅在初始化时子→父发送 ready 信号
7. **三层主题同步**: Widget 模式下主题需要同步本地 state、useConfig store 和 Mantine 管理器三层状态
8. **会话隔离设计**: sessionStorage 的天然隔离特性确保了多标签页编辑内容互不干扰

### 7.3 跨页面同步边界总结

| 场景 | 同步机制 | 实时性 | 说明 |
|------|---------|--------|------|
| 多标签页 Editor | 无 | ❌ | localStorage 隐式共享但不同步，sessionStorage 完全隔离 |
| Widget ↔ 父页面 | postMessage | ✅ | 父→子单向实时同步，子→父仅初始化时发送 ready 信号 |
| 多标签页 Widget | 无 | ❌ | 每个 iframe 独立，完全由各自父页面控制 |
| Editor ↔ Widget | 无 | ❌ | 即使同源打开，也无任何同步机制 |
