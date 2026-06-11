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

### 4.1 现状：无跨标签页同步

经过代码分析，JSON Crack **没有实现**跨浏览器标签页的状态同步机制：

- ❌ 未使用 `BroadcastChannel` API
- ❌ 未监听 `storage` 事件（`window.addEventListener('storage', ...)`）
- ❌ 未使用 `SharedWorker`
- ❌ 未使用 IndexedDB

### 4.2 各标签页状态独立性

每个浏览器标签页拥有独立的状态：

| 存储类型 | 跨标签页共享 | 自动同步 | 说明 |
|---------|-------------|---------|------|
| localStorage | ✅ 是 | ❌ 否 | 数据共享，但需要刷新页面才能看到其他标签页的更改 |
| sessionStorage | ❌ 否 | ❌ 否 | 每个标签页独立，数据不共享 |

**localStorage 的隐式共享**:
- localStorage 本身是同源共享的
- 但由于没有监听 `storage` 事件，其他标签页的更改不会实时反映
- 需要手动刷新页面才能获取最新的 localStorage 数据

### 4.3 Widget 与父窗口的同步

Widget 嵌入模式通过 `postMessage` 实现与父窗口的单向通信（父→子）：

```typescript
// 父窗口发送消息
window.postMessage({
  json: "...",
  options: {
    theme: "dark",
    direction: "RIGHT"
  }
}, "*");

// Widget 接收并更新状态
// 参见 widget.tsx 中的 message 事件监听
```

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
4. **无跨标签页同步**: 依赖浏览器原生存储特性，不主动实现多标签页状态同步
5. **Widget 特殊处理**: 嵌入模式下禁用 sessionStorage 恢复，通过 postMessage 与父窗口通信
