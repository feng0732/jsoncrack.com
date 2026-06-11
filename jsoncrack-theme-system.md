# JSON Crack 主题切换与样式系统协作分析

## 一、系统架构概览

JSON Crack 采用了**三层主题系统**协同工作的架构，同时通过**四类主题切换入口**（编辑器顶栏、画布偏好菜单、嵌入页 postMessage、VS Code / Chrome 扩展宿主环境）驱动主题切换。各入口的可用性按页面模式有所差异：

| 入口 | `/editor` 编辑器页 | `/widget` 嵌入页 | VS Code 扩展 | Chrome 扩展 |
|------|-------------------|-----------------|-------------|------------|
| 顶栏 ThemeToggle | ✅ 可用 | ❌ 不存在（无顶栏） | ❌ 不存在 | ❌ 不存在 |
| 画布偏好菜单（齿轮） | ✅ 可用 | ❌ `isWidget=true` 时被条件排除 | ❌ 不存在 | ❌ 不存在 |
| postMessage（需携带 json） | N/A（未监听） | ✅ 唯一控制方式 | N/A | N/A |
| 宿主环境主题（IDE/系统） | N/A | N/A | ✅ 只读 | ✅ 实时跟随 |

| 层级 | 技术方案 | 覆盖范围 | 主题来源 |
|------|----------|----------|----------|
| UI 组件层 | Mantine UI + ColorSchemeManager | 按钮、弹窗、输入框等 Mantine 组件 | `apps/www/src/lib/utils/mantineColorScheme.ts` |
| 业务样式层 | styled-components + ThemeProvider | 编辑器布局、工具栏、树视图等自定义组件 | `apps/www/src/constants/theme.ts` |
| 画布渲染层 | CSS Variables + inline style | JSON 图形画布（节点、连线、网格等） | `packages/jsoncrack-react/src/theme.ts` |

### 关键文件索引（仓库相对路径）

| 角色 | 仓库相对路径 |
|------|-------------|
| 全局入口 | `apps/www/src/pages/_app.tsx` |
| 编辑器页面 | `apps/www/src/pages/editor.tsx` |
| 嵌入页 | `apps/www/src/pages/widget.tsx` |
| 文档页（嵌入 API 文档） | `apps/www/src/pages/docs.tsx` |
| 状态存储 | `apps/www/src/store/useConfig.ts` |
| Mantine 颜色方案管理器 | `apps/www/src/lib/utils/mantineColorScheme.ts` |
| WWW 主题 Token 定义 | `apps/www/src/constants/theme.ts` |
| styled-components 类型声明 | `apps/www/src/types/styled.d.ts` |
| 全局基础样式 | `apps/www/src/constants/globalStyle.ts` |
| 编辑器顶栏 ThemeToggle | `apps/www/src/features/editor/Toolbar/ThemeToggle.tsx` |
| 编辑器顶栏主入口 | `apps/www/src/features/editor/Toolbar/index.tsx` |
| 顶栏样式模板 | `apps/www/src/features/editor/Toolbar/styles.ts` |
| 画布偏好菜单（含主题切换） | `apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx` |
| GraphView 主组件 | `apps/www/src/features/editor/views/GraphView/index.tsx` |
| Monaco 编辑器 | `apps/www/src/features/editor/TextEditor.tsx` |
| LiveEditor 容器 | `apps/www/src/features/editor/LiveEditor.tsx` |
| BottomBar | `apps/www/src/features/editor/BottomBar.tsx` |
| 画布组件入口 | `packages/jsoncrack-react/src/JSONCrackComponent.tsx` |
| 画布主题 Token | `packages/jsoncrack-react/src/theme.ts` |
| 画布 CSS Variables 构建器 | `packages/jsoncrack-react/src/canvasHelpers.ts` |
| 画布全局样式 | `packages/jsoncrack-react/src/JSONCrackStyles.module.css` |
| 画布节点样式模块 | `packages/jsoncrack-react/src/components/Node.module.css` |
| 节点颜色映射函数 | `packages/jsoncrack-react/src/components/nodeStyles.ts` |
| ObjectNode 组件 | `packages/jsoncrack-react/src/components/ObjectNode.tsx` |
| TextNode 组件 | `packages/jsoncrack-react/src/components/TextNode.tsx` |
| CustomNode 组件 | `packages/jsoncrack-react/src/components/CustomNode.tsx` |
| CustomEdge 组件 | `packages/jsoncrack-react/src/components/CustomEdge.tsx` |
| Controls 组件 | `packages/jsoncrack-react/src/components/Controls.tsx` |
| Controls 样式 | `packages/jsoncrack-react/src/components/Controls.module.css` |
| 类型定义（含 CanvasThemeMode） | `packages/jsoncrack-react/src/types.ts` |
| 包导出入口 | `packages/jsoncrack-react/src/index.ts` |
| Chrome 扩展 content-script | `apps/chrome-extension/src/content-script.tsx` |
| VS Code 扩展 App | `apps/vscode/src/App.tsx` |

---

## 二、主题状态管理

### 2.1 单一真实数据源：Zustand Store

主题状态的核心是 `darkmodeEnabled` 布尔值，存储在 Zustand 的 `useConfig` store 中：

**文件**：`apps/www/src/store/useConfig.ts`

```typescript
const initialStates = {
  darkmodeEnabled: true,   // 默认深色模式
  liveTransformEnabled: true,
  gesturesEnabled: false,
  rulersEnabled: true,
};

// 通过 zustand/persist 中间件自动持久化到 localStorage (key: "config")
const useConfig = create(
  persist<typeof initialStates & ConfigActions>(
    set => ({
      ...initialStates,
      toggleDarkMode: darkmodeEnabled => set({ darkmodeEnabled }),
      // ...其他 toggle 方法
    }),
    {
      name: "config",  // localStorage key
    }
  )
);
```

**状态持久化机制**：
- 使用 `zustand/middleware` 的 `persist` 中间件
- 自动将 store 状态序列化存储到 `localStorage["config"]`
- 页面刷新时自动从 localStorage 恢复

### 2.2 主题切换入口一：编辑器顶栏 ThemeToggle

**文件**：`apps/www/src/features/editor/Toolbar/ThemeToggle.tsx`

```tsx
export const ThemeToggle = () => {
  const darkmodeEnabled = useConfig(state => state.darkmodeEnabled);
  const toggleDarkMode = useConfig(state => state.toggleDarkMode);

  return (
    <StyledToolElement
      title={!darkmodeEnabled ? "Dark Mode" : "Light Mode"}
      onClick={() => toggleDarkMode(!darkmodeEnabled)}
    >
      {!darkmodeEnabled ? <FaMoon size="18" /> : <FaSun size="18" />}
    </StyledToolElement>
  );
};
```

该组件位于编辑器页面**顶部工具栏右侧**（`Toolbar/index.tsx` L87），直接修改 Zustand store。

### 2.3 主题切换入口二：画布偏好菜单（Preferences）

**文件**：`apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx`（L281-L327）

画布底部悬浮工具栏最右侧有一个齿轮图标（`LuSettings2`），点击展开 Preferences 菜单，其中包含主题切换项：

```tsx
<Menu trigger="click" position="top-end">
  <Menu.Target>
    <Tooltip label="Preferences" position="top" withArrow openDelay={750}>
      <ActionIcon aria-label="preferences" size="lg" radius="md" variant="subtle" color="gray">
        <LuSettings2 size={18} />
      </ActionIcon>
    </Tooltip>
  </Menu.Target>
  <Menu.Dropdown>
    <Menu.Item
      fz="sm"
      leftSection={darkmodeEnabled ? <FaSun /> : <FaMoon />}
      onClick={() => toggleDarkMode(!darkmodeEnabled)}  // 同样调用 useConfig.toggleDarkMode
      closeMenuOnClick={false}
    >
      {darkmodeEnabled ? "Light Mode" : "Dark Mode"}
    </Menu.Item>
    <Menu.Item fz="sm" rightSection={<BsCheck2 display={gesturesEnabled ? "initial" : "none"} />}
      onClick={() => toggleGestures(!gesturesEnabled)} closeMenuOnClick={false}>
      Zoom on Scroll
    </Menu.Item>
    <Menu.Item fz="sm" rightSection={<BsCheck2 display={rulersEnabled ? "initial" : "none"} />}
      onClick={() => toggleRulers(!rulersEnabled)} closeMenuOnClick={false}>
      Rulers
    </Menu.Item>
  </Menu.Dropdown>
</Menu>
```

**两个入口的统一性**：顶栏 `ThemeToggle` 和画布偏好菜单都调用同一个 `useConfig.toggleDarkMode()`，写入同一个 Zustand store，因此无论从哪个入口切换，三层主题系统的联动效果完全一致。

### 2.4 主题切换入口三：嵌入页 postMessage

**文件**：`apps/www/src/pages/widget.tsx`

嵌入页（`/widget`）作为 iframe 被第三方网站嵌入时，通过 `window.postMessage` 接收外部传入的主题参数：

```typescript
interface EmbedMessage {
  data: {
    json?: string;
    options?: {
      theme?: "light" | "dark";
      direction?: LayoutDirection;
    };
  };
}
```

**消息监听与主题同步**（`widget.tsx` L56-L75）：

```tsx
React.useEffect(() => {
  const handler = (event: EmbedMessage) => {
    try {
      if (!event.data?.json) return;
      // 接收外部主题参数
      if (event.data?.options?.theme === "light" || event.data?.options?.theme === "dark") {
        setTheme(event.data.options.theme);             // 更新组件本地 state
        toggleDarkMode(event.data.options.theme === "dark"); // 同步到 Zustand store
      }
      setContents({ contents: event.data.json, hasChanges: false });
      setDirection(event.data.options?.direction || "RIGHT");
    } catch (error) {
      console.error(error);
      toast.error("Invalid JSON!");
    }
  };

  window.addEventListener("message", handler);
  return () => window.removeEventListener("message", handler);
}, [setColorScheme, setContents, setDirection, toggleDarkMode, theme]);
```

**Mantine 层同步**（`widget.tsx` L77-L79）：

```tsx
React.useEffect(() => {
  setColorScheme(theme);  // 将本地 state 同步到 Mantine ColorScheme
}, [setColorScheme, theme]);
```

**styled-components 层同步**（`widget.tsx` L82-L88）：

```tsx
<ThemeProvider theme={theme === "dark" ? darkTheme : lightTheme}>
  <GraphView isWidget />
</ThemeProvider>
```

**嵌入页 vs 编辑器页的关键差异**：

| 对比项 | editor.tsx | widget.tsx |
|--------|-----------|------------|
| 主题来源 | Zustand store (`darkmodeEnabled`)，单一来源 | **双状态并存**：本地 `theme` state + Zustand `darkmodeEnabled` |
| 主题驱动源 | 用户 UI 交互（2 个入口）触发 `toggleDarkMode` | `postMessage` 外部传入（必须携带 json 字段） |
| 本地 state | 无（直接读 store） | 有 `const [theme, setTheme] = useState<"dark" \| "light">("dark")` |
| Mantine 同步 | `useEffect` 监听 `darkmodeEnabled` → `setColorScheme()` | `useEffect` 监听 **本地 `theme` state** → `setColorScheme()` |
| styled-components 同步 | `ThemeProvider theme={darkmodeEnabled ? darkTheme : lightTheme}` | `ThemeProvider theme={theme === "dark" ? darkTheme : lightTheme}` → 读取**本地 `theme` state** |
| 画布主题传递 | `<JSONCrack theme={darkmodeEnabled ? "dark" : "light"}>` | `<GraphView isWidget>` → 内部读 **`useConfig.darkmodeEnabled`**（不读本地 state） |
| 持久化 | Zustand persist → `localStorage["config"]` | 同样 Zustand persist（每次 postMessage 覆盖） |
| 画布工具栏（偏好菜单） | 渲染（`{!isWidget && <Toolbar />}`，L91） | **不渲染**（`isWidget=true` 时被 `{!isWidget}` 条件排除） |
| 主题切换入口数 | 2 个（顶栏 ThemeToggle + 画布偏好菜单） | **0 个 UI 入口**，仅通过 `postMessage` 接收外部指令 |

### 嵌入页双状态职责边界（核心澄清）

嵌入页（`widget.tsx`）中存在**两个独立但同步**的主题状态，各自驱动不同的子系统：

#### 1. 本地 `theme` state（`widget.tsx` L40）
```typescript
const [theme, setTheme] = React.useState<"dark" | "light">("dark");
```
**职责**：驱动 **Mantine 层** 和 **styled-components 层**
- **Mantine 消费者**（`widget.tsx` L77-L79）：
  ```tsx
  React.useEffect(() => {
    setColorScheme(theme);  // 读取本地 theme state
  }, [setColorScheme, theme]);
  ```
- **styled-components 消费者**（`widget.tsx` L82）：
  ```tsx
  <ThemeProvider theme={theme === "dark" ? darkTheme : lightTheme}>
  ```

#### 2. Zustand `darkmodeEnabled`（`useConfig` store）
**职责**：驱动 **画布层** + **持久化**
- **画布消费者**（`GraphView/index.tsx` L58, L101）：
  ```tsx
  const darkmodeEnabled = useConfig(state => state.darkmodeEnabled);  // 读 Zustand
  // ...
  <JSONCrack theme={darkmodeEnabled ? "dark" : "light"} />
  ```
- **持久化**：zustand/persist 自动写入 `localStorage["config"]`

#### 3. 同步机制（`widget.tsx` L60-L62）
两个状态在 postMessage handler 中**同步依次更新**，确保始终一致：
```tsx
if (event.data?.options?.theme === "light" || event.data?.options?.theme === "dark") {
  setTheme(event.data.options.theme);              // 1. 更新本地 state → Mantine + styled-components
  toggleDarkMode(event.data.options.theme === "dark");  // 2. 更新 Zustand → 画布 + 持久化
}
```

**为什么需要双状态？**
- 本地 `theme` state 是 `postMessage` 主题参数的**接收缓冲区**，直接驱动页面 UI 框架（Mantine/styled-components）
- Zustand `darkmodeEnabled` 是**全局配置源**，负责画布渲染和持久化，与编辑器页共享同一 store
- 两者同步写入保证三层系统的一致性

---

**嵌入页的同步链路（含 postMessage 前置守卫）**：

```
外部父页面 postMessage({json: "...", options: {theme: "light"}})
   │
   ├─ [Guard] if (!event.data?.json) return;   ← widget.tsx L59：必须携带 json 字段
   │    ↑ 若只传 theme 而不传 json，整条消息被直接丢弃
   │
   ├─→ [Conditional] if (event.data?.options?.theme === "light" || "dark")
   │     ├─→ setTheme("light")              → 本地 state 更新
   │     └─→ toggleDarkMode(false)          → Zustand store.darkmodeEnabled = false
   │           └─→ localStorage["config"] 持久化
   │           └─→ GraphView 内部 selector 触发重渲染
   │                 └─→ <JSONCrack theme="light" />
   │
   ├─→ setContents(...)                      → JSON 数据同步
   └─→ setDirection(...)                     → 布局方向同步

useEffect([theme]) 触发
   └─→ setColorScheme("light")               → Mantine 层更新
         └─→ localStorage["editor-color-scheme"] = "light"

useEffect([theme]) 也触发 ThemeProvider 重渲染
   └─→ <ThemeProvider theme={lightTheme}>    → styled-components 层更新
```

**嵌入页主题边界代码定位**：

| 边界条件 | 仓库相对路径 | 行号 |
|---------|-------------|-----|
| postMessage 必须携带 `json` 字段 | `apps/www/src/pages/widget.tsx` | L59 `if (!event.data?.json) return;` |
| theme 必须是 `"light"` 或 `"dark"` 才生效 | `apps/www/src/pages/widget.tsx` | L60 `if (event.data?.options?.theme === "light" \|\| ...)` |
| widget 模式不渲染画布偏好菜单（无 UI 切换入口） | `apps/www/src/features/editor/views/GraphView/index.tsx` | L91 `{!isWidget && <Toolbar />}` |

### 2.5 主题切换入口四：外部客户端（VS Code / Chrome 扩展）

这两个客户端不使用 Zustand store，而是**直接读取宿主环境主题**，传递给 `<JSONCrack theme={...} />`：

#### VS Code 扩展

**文件**：`apps/vscode/src/App.tsx`（L15-L19, L56）

```typescript
function getTheme() {
  const theme = document.body.getAttribute("data-vscode-theme-kind");
  if (theme?.includes("light")) return "light" as const;
  return "dark";
}

// 直接传给 JSONCrack，不经过任何 Zustand store
<JSONCrack json={json} theme={theme} showControls={false} onNodeClick={handleNodeClick} />

// Mantine 通过 forceColorScheme 锁定
<MantineProvider forceColorScheme={theme}>
```

VS Code 扩展的特点：
- 主题由 VS Code IDE 自身的颜色方案决定（`data-vscode-theme-kind` 属性）
- 使用 `forceColorScheme` 而非 `ColorSchemeManager`，因为不需要用户手动切换
- 没有提供用户可操作的主题切换 UI

#### Chrome 扩展

**文件**：`apps/chrome-extension/src/content-script.tsx`（L86-L103, L304-L363）

```typescript
function detectTheme(): ThemeMode {
  return window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light";
}

function useSystemTheme(): ThemeMode {
  const [theme, setTheme] = useState<ThemeMode>(detectTheme);

  useEffect(() => {
    const media = window.matchMedia("(prefers-color-scheme: dark)");
    const handler = (event: MediaQueryListEvent) => {
      setTheme(event.matches ? "dark" : "light");
    };
    media.addEventListener("change", handler);
    return () => media.removeEventListener("change", handler);
  }, []);

  return theme;
}

// 在 GraphView 中使用
function GraphView({ rawJson }: { rawJson: string }) {
  const theme = useSystemTheme();
  // ...
  return <JSONCrackComponent json={parsedJson} theme={theme} showControls showGrid centerOnLayout />;
}
```

Chrome 扩展的特点：
- 主题由**操作系统偏好**（`prefers-color-scheme`）决定
- 实时监听系统主题变更（`media.addEventListener("change", handler)`）
- 没有用户可操作的主题切换 UI（跟随系统）
- 不使用 styled-components ThemeProvider 或 Mantine，仅使用 `jsoncrack-react` 包的 CSS Variables 机制

---

### 2.6 四端主题状态来源与刷新时序对比

四个客户端的主题驱动机制完全不同，汇总如下：

| 对比维度 | 编辑器页 `/editor` | 嵌入页 `/widget` | VS Code 扩展 | Chrome 扩展 |
|---------|-------------------|-----------------|-------------|------------|
| **主题状态来源** | Zustand store `darkmodeEnabled`（单一来源） | 双状态：本地 `theme` state + Zustand `darkmodeEnabled` | `document.body.getAttribute("data-vscode-theme-kind")` | `window.matchMedia("(prefers-color-scheme: dark)")` |
| **驱动触发源** | 用户 UI 交互（2 个入口：ThemeToggle + 偏好菜单） | 父页面 `postMessage`（必须携带 json） | VS Code 宿主环境（页面加载时一次性读取） | 操作系统（实时监听 `MediaQueryList` 事件） |
| **本地 state** | 无（直接读 Zustand） | 有 `useState<"dark" \| "light">("dark")` | 无（直接读 DOM 属性） | 有 `useState`（仅作为系统主题的镜像） |
| **使用 Zustand** | ✅ 唯一真实数据源 | ✅ 双状态并存 | ❌ 无 | ❌ 无 |
| **使用 Mantine** | ✅ `ColorSchemeManager` | ✅ `ColorSchemeManager` | ✅ `forceColorScheme` | ❌ 无 |
| **使用 styled-components** | ✅ `ThemeProvider` | ✅ `ThemeProvider` | ❌ 无 | ❌ 无 |
| **Mantine 同步方式** | `useEffect([darkmodeEnabled]) → setColorScheme()` | `useEffect([本地 theme]) → setColorScheme()` | `forceColorScheme={theme}`（一次性锁定） | N/A |
| **styled-components 同步方式** | `ThemeProvider theme={darkmodeEnabled ? ...}` | `ThemeProvider theme={本地 theme ? ...}` | N/A | N/A |
| **画布主题传递** | `<JSONCrack theme={darkmodeEnabled ? ...}>` | `<JSONCrack theme={useConfig.darkmodeEnabled}>` | `<JSONCrack theme={theme}>` | `<JSONCrackComponent theme={theme}>` |
| **运行时切换能力** | ✅ 用户可随时切换 | ✅ 父页面可随时发 postMessage | ❌ 页面加载时一次性确定，无切换监听 | ✅ 实时跟随系统主题变化 |
| **持久化** | Zustand persist → `localStorage["config"]` + Mantine → `localStorage["editor-color-scheme"]` | 同上（双持久化） | ❌ 无（每次加载重新读取） | ❌ 无（每次挂载重新检测） |

### 2.6.1 四端状态驱动因果链

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                        编辑器页 (editor.tsx)                                   │
│  Zustand store.darkmodeEnabled                                                │
│    ↑ (toggleDarkMode)                                                         │
│    ├─ 用户点击顶栏 ThemeToggle  → 直接驱动所有三层                             │
│    └─ 用户点击画布偏好菜单   → 直接驱动所有三层                                │
└───────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                        嵌入页 (widget.tsx)                                    │
│  postMessage({json: "...", options: {theme: "light"}})                        │
│    │ [Guard L59] if (!json) return;                                           │
│    ▼                                                                          │
│  ┌─ 同步执行 ──────────────────────────────────────────────────────────────┐  │
│  │  ① setTheme("light")           → 本地 state                           │  │
│  │     ├─ 驱动 Mantine (useEffect([theme]) → setColorScheme)            │  │
│  │     └─ 驱动 styled-components (ThemeProvider theme=...)               │  │
│  │  ② toggleDarkMode(false)        → Zustand store                       │  │
│  │     ├─ 驱动 GraphView 画布 (<JSONCrack theme=...>)                    │  │
│  │     └─ 持久化到 localStorage["config"]                                │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────────┐
│                      VS Code 扩展 (apps/vscode/src/App.tsx)                   │
│  document.body.getAttribute("data-vscode-theme-kind")                        │
│    │ 页面加载时一次性读取                                                     │
│    ▼                                                                          │
│  <MantineProvider forceColorScheme={theme}>                                   │
│  <JSONCrack json={...} theme={theme} />                                       │
│  无 Zustand，无运行时切换监听                                                 │
└───────────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────────┐
│                    Chrome 扩展 (apps/chrome-extension/...)                    │
│  window.matchMedia("(prefers-color-scheme: dark)")                            │
│    │ 挂载时 + "change" 事件时触发                                             │
│    ▼                                                                          │
│  useSystemTheme() → 本地 state（仅作为镜像）                                  │
│    │                                                                          │
│    ▼                                                                          │
│  <JSONCrackComponent json={...} theme={theme} />                              │
│  无 Zustand，无 Mantine，无 styled-components，仅 CSS Variables               │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

### 2.7 Mantine 专用：智能颜色方案管理器

为了实现**路径感知**的主题行为（编辑器页面用动态主题，营销页面强制浅色），项目自定义了 `smartColorSchemeManager`：

**文件**：`apps/www/src/lib/utils/mantineColorScheme.ts`

核心逻辑：
```typescript
export function smartColorSchemeManager({
  key, getPathname, dynamicPaths = [],
}: SmartColorSchemeManagerOptions): MantineColorSchemeManager {
  let currentColorScheme: MantineColorScheme | null = null;

  const shouldUseDynamicBehavior = () => {
    const pathname = getPathname();
    return dynamicPaths.some(path => pathname === path || pathname.startsWith(`${path}/`));
  };

  return {
    get: defaultValue => {
      // 非动态路径（如首页、文档页）→ 强制返回 light
      if (!shouldUseDynamicBehavior()) return "light";
      // 动态路径（/editor, /widget）→ 从内存或 localStorage 读取
      if (currentColorScheme) return currentColorScheme;
      // ...从 localStorage["editor-color-scheme"] 读取
    },
    set: value => {
      // 仅动态路径才保存
      if (!shouldUseDynamicBehavior()) return;
      currentColorScheme = value;
      window.localStorage.setItem(key, value);
    },
    subscribe: () => {},
    unsubscribe: () => {},
    clear: () => { /* 清除内存和 localStorage */ },
  };
}
```

**动态路径配置**（`apps/www/src/pages/_app.tsx` L74-L78）：
```typescript
const colorSchemeManager = smartColorSchemeManager({
  key: "editor-color-scheme",
  getPathname: () => pathname,
  dynamicPaths: ["/editor", "/widget"],
});
```

---

## 三、三层主题系统的串联点

### 3.1 全局入口初始化

**文件**：`apps/www/src/pages/_app.tsx`

```tsx
// Provider 嵌套层次（由外到内）：
<MantineProvider
  colorSchemeManager={colorSchemeManager}  // 第1层：Mantine 主题
  defaultColorScheme="light"
  theme={mantineTheme}
>
  <CodeHighlightAdapterProvider adapter={shikiAdapter}>
    <ThemeProvider theme={lightTheme}>       {/* 第2层：styled-components 主题（营销页面默认浅色）*/}
      <GlobalStyle />
      <Component {...pageProps} />           {/* 子页面可覆盖 ThemeProvider */}
    </ThemeProvider>
  </CodeHighlightAdapterProvider>
</MantineProvider>
```

**注意**：`_app.tsx` 中的 `ThemeProvider` 只设置了 `lightTheme`，这是营销页面的默认值。真正的动态切换在 `editor.tsx` / `widget.tsx` 中重新覆盖。

### 3.2 编辑器页面的关键联动

**文件**：`apps/www/src/pages/editor.tsx`

这是串联三层系统的**核心节点**：

```tsx
const EditorPage = () => {
  const darkmodeEnabled = useConfig(state => state.darkmodeEnabled);
  const { setColorScheme } = useMantineColorScheme();

  useEffect(() => {
    setColorScheme(darkmodeEnabled ? "dark" : "light");
  }, [darkmodeEnabled, setColorScheme]);

  return (
    <ThemeProvider theme={darkmodeEnabled ? darkTheme : lightTheme}>
      <StyledPageWrapper>
        <Toolbar />
        <StyledEditorWrapper>
          <StyledEditor proportionalLayout={false}>
            {/* ...TextEditor 和 LiveEditor 放在内部 */}
          </StyledEditor>
        </StyledEditorWrapper>
      </StyledPageWrapper>
    </ThemeProvider>
  );
};
```

**联动关系**：

```
用户点击 ThemeToggle / 偏好菜单主题项
   ↓
useConfig.toggleDarkMode() 更新 Zustand store
   ↓
├─→ localStorage["config"] 自动持久化（zustand/persist）
├─→ EditorPage 的 darkmodeEnabled selector 触发重渲染
│     ├─→ ThemeProvider 切换 theme 对象（darkTheme / lightTheme）
│     └─→ useEffect 触发 setColorScheme() 更新 Mantine 层
│           └─→ Mantine Provider 更新 + localStorage["editor-color-scheme"] 持久化
└─→ GraphView 的 darkmodeEnabled selector 触发重渲染
      └─→ <JSONCrack theme={darkmodeEnabled ? "dark" : "light"} />
            └─→ buildCanvasStyle() 重新计算 CSS Variables
```

---

## 四、样式计算机制

### 4.1 styled-components：主题对象 + CSS-in-JS

**主题定义文件**：`apps/www/src/constants/theme.ts`

主题对象结构：
```typescript
const fixedColors = { /* 不随主题变化的颜色：CRIMSON, BLURPLE 等 */ };

const nodeColors = {
  dark: { NODE_COLORS: { TEXT, NODE_KEY, NODE_VALUE, ... } },
  light: { NODE_COLORS: { /* ... */ } },
};

export const darkTheme = {
  ...fixedColors,
  ...nodeColors.dark,
  BACKGROUND_PRIMARY: "#36393f",
  BACKGROUND_SECONDARY: "#2f3136",
  TEXT_NORMAL: "#dcddde",
  // ... 共 20+ 个语义化颜色 Token
};

export const lightTheme = {
  ...fixedColors,
  ...nodeColors.light,
  BACKGROUND_PRIMARY: "#FFFFFF",
  BACKGROUND_SECONDARY: "#f2f3f5",
  TEXT_NORMAL: "#2e3338",
  // ...
};
```

**消费方式**（以 `apps/www/src/features/editor/Toolbar/styles.ts` 为例）：
```tsx
export const StyledToolElement = styled.button<{ $hide?: boolean; $highlight?: boolean }>`
  color: ${({ theme }) => theme.INTERACTIVE_NORMAL};
  
  &:hover {
    color: ${({ theme }) => theme.INTERACTIVE_HOVER};
  }
`;
```

**间接消费 theme 判断亮暗**：部分组件通过 `theme.BACKGROUND_SECONDARY` 的值来判断当前是否浅色模式，用于条件样式：

- `apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx` L26-L51（`glassSurface` 毛玻璃样式）
- `apps/www/src/features/editor/views/GraphView/index.tsx` L31-L37（节点阴影颜色）
- `apps/www/src/features/editor/BottomBar.tsx` L71-L73（hover 背景色）

```tsx
// GraphView/Toolbar/index.tsx 中的 glassSurface
background: ${({ theme }) =>
  theme.BACKGROUND_SECONDARY === "#f2f3f5"  // 用具体值判断是否浅色
    ? "rgba(255, 255, 255, 0.72)"
    : "rgba(28, 28, 30, 0.72)"};
```

**TypeScript 类型支持**：`apps/www/src/types/styled.d.ts`
```typescript
import "styled-components";
import type theme from "../constants/theme";

type CustomTheme = typeof theme;

declare module "styled-components" {
  export interface DefaultTheme extends CustomTheme {}
}
```

### 4.2 JSONCrack 画布：CSS 自定义属性（CSS Variables）

画布组件（`jsoncrack-react` 包）采用独立的主题系统，通过**内联 style 注入 CSS Variables**。

**步骤 1：主题定义** - `packages/jsoncrack-react/src/theme.ts`
```typescript
export const themes: Record<CanvasThemeMode, JSONCrackTheme> = {
  dark: {
    NODE_COLORS: { TEXT: "#DCE5E7", NODE_KEY: "#59b8ff", ... },
    GRID_BG_COLOR: "#141414",
    // ...
  },
  light: {
    NODE_COLORS: { TEXT: "#000000", NODE_KEY: "#761CEA", ... },
    GRID_BG_COLOR: "#f7f7f7",
    // ...
  },
};
```

**步骤 2：构建 CSS Variables** - `packages/jsoncrack-react/src/canvasHelpers.ts` L32-L65
```typescript
export const buildCanvasStyle = (
  theme: CanvasThemeMode,
  userStyle?: CSSProperties
): CSSProperties => {
  const themeTokens = themes[theme];
  const isDark = theme === "dark";

  return {
    "--bg-color": themeTokens.GRID_BG_COLOR,
    "--line-color-1": themeTokens.GRID_COLOR_PRIMARY,
    "--node-fill": isDark ? "#292929" : "#ffffff",
    "--node-stroke": isDark ? "#424242" : "#BCBEC0",
    "--node-text": themeTokens.NODE_COLORS.TEXT,
    "--node-key": themeTokens.NODE_COLORS.NODE_KEY,
    "--node-integer": themeTokens.NODE_COLORS.INTEGER,
    "--node-null": themeTokens.NODE_COLORS.NULL,
    "--node-bool-true": themeTokens.NODE_COLORS.BOOL.TRUE,
    "--node-bool-false": themeTokens.NODE_COLORS.BOOL.FALSE,
    // ... 共约 20 个 CSS Variables
    ...userStyle,
  } as CSSProperties;
};
```

**步骤 3：注入到组件** - `packages/jsoncrack-react/src/JSONCrackComponent.tsx` L161-L165
```tsx
const canvasStyle = useMemo(() => buildCanvasStyle(theme, style), [theme, style]);

return (
  <div ref={containerRef} style={canvasStyle} ... >
    {/* 内部子元素通过 CSS 变量读取颜色 */}
  </div>
);
```

**步骤 4：CSS 消费** - `packages/jsoncrack-react/src/JSONCrackStyles.module.css`

| CSS 选择器 | 消费的变量 | 效果 |
|-----------|-----------|------|
| `.canvasWrapper` | `var(--bg-color)` | 画布背景色 |
| `.showGrid` | `var(--line-color-1)`, `var(--line-color-2)` | 网格线颜色 |
| `:global(text)` | `var(--interactive-normal)` | SVG 文字填充色 |
| `:global(rect)` | `var(--node-fill)` | SVG 矩形填充色 |
| `.spinner` | `var(--spinner-track)`, `var(--spinner-head)` | 加载指示器 |

**步骤 5：DOM 元素行内消费** - `packages/jsoncrack-react/src/components/nodeStyles.ts`

```typescript
export const getTextColor = ({ type, value }: TextColorOptions) => {
  if (value === null) return "var(--node-null)";
  if (type === "object") return "var(--node-key)";
  if (type === "number") return "var(--node-integer)";
  if (value === true) return "var(--node-bool-true)";
  if (value === false) return "var(--node-bool-false)";
  return "var(--node-value)";
};
```

**步骤 6：SVG 元素样式消费** - `packages/jsoncrack-react/src/components/CustomNode.tsx` & `CustomEdge.tsx`

```tsx
// CustomNode.tsx - 节点容器使用 CSS Variables
<Node
  style={{
    fill: "var(--node-fill)",
    stroke: "var(--node-stroke)",
    strokeWidth: 1,
  }}
  onEnter={event => { event.currentTarget.style.stroke = "#3B82F6"; }}
  onLeave={event => { event.currentTarget.style.stroke = "var(--node-stroke)"; }}
/>

// CustomEdge.tsx - 连线使用 CSS Variables
<Edge
  style={{
    stroke: hovered ? "#3B82F6" : "var(--edge-stroke)",
    strokeWidth: 1.5,
  }}
/>
```

**步骤 7：节点内行样式消费** - `packages/jsoncrack-react/src/components/Node.module.css`

| CSS 类 | 消费的变量 | 效果 |
|--------|-----------|------|
| `.foreignObject` | `var(--node-text)` | 节点基础文字色 |
| `.row` | `var(--node-divider)` | 行间分隔线 |
| `.collapseButton` | `var(--interactive-normal)`, `var(--node-divider)` | 折叠按钮边框/文字 |

### 4.3 Monaco 编辑器：专有主题映射

**文件**：`apps/www/src/features/editor/TextEditor.tsx`

Monaco 有自己的主题系统，通过简单的映射实现联动：
```tsx
const theme = useConfig(state => (state.darkmodeEnabled ? "vs-dark" : "light"));

<Editor
  theme={theme}   // 直接传 Monaco 内置主题名
  language={fileType}
/>
```

---

## 五、完整响应链路时序分析

### 场景 A：编辑器页面 — 点击顶栏 ThemeToggle

```
时间轴（从上到下）
│
├─ 1. [用户交互] 点击 <ThemeToggle> 中的月亮/太阳图标
│
├─ 2. [状态更新] useConfig.toggleDarkMode(!darkmodeEnabled)
│   ├─ Zustand 更新内部 state.darkmodeEnabled
│   └─ persist 中间件：localStorage["config"] = {...darkmodeEnabled: true/false}
│
├─ 3. [订阅通知] 所有使用 selector 读取 darkmodeEnabled 的组件触发重渲染：
│   │
│   ├─ 3a. [EditorPage 重渲染]
│   │   ├─ a1. <ThemeProvider theme={darkTheme/lightTheme}> 切换 theme 对象
│   │   │   └─ styled-components 通知所有后代 styled.* 组件重新计算样式
│   │   │       ├─ StyledEditor.background → BACKGROUND_SECONDARY
│   │   │       ├─ StyledToolElement.color → INTERACTIVE_NORMAL
│   │   │       ├─ StyledBottomBar.border-bottom → BACKGROUND_MODIFIER_ACCENT
│   │   │       ├─ StyledBottomBar.background → TOOLBAR_BG
│   │   │       ├─ glassSurface (画布 Toolbar) → 条件判断 BACKGROUND_SECONDARY
│   │   │       └─ ... 所有使用 ${({theme}) => ...} 的样式
│   │   │
│   │   └─ a2. useEffect 依赖 darkmodeEnabled 触发
│   │       └─ setColorScheme(dark ? "dark" : "light") → Mantine ColorSchemeManager
│   │           ├─ smartColorSchemeManager.set()
│   │           │   ├─ 更新内存中的 currentColorScheme
│   │           │   └─ localStorage["editor-color-scheme"] = "dark"/"light"
│   │           └─ MantineProvider 更新内部 colorScheme
│   │               └─ 所有 Mantine 组件（Button/Modal/Tooltip/Menu/ActionIcon...）应用新主题
│   │
│   ├─ 3b. [GraphView 重渲染]
│   │   └─ <JSONCrack theme={darkmodeEnabled ? "dark" : "light"} /> props 更新
│   │       └─ useMemo(() => buildCanvasStyle(theme, style), [theme, style]) 重新执行
│   │           └─ 生成包含 20+ 个 CSS Variables 的新 style 对象
│   │               └─ div[style=...] 更新 inline style
│   │                   └─ 浏览器根据 CSS 变量层叠重绘（无需 React 遍历节点）
│   │
│   └─ 3c. [TextEditor 重渲染]
│       └─ Monaco Editor theme="vs-dark"/"light" 切换
│
└─ 4. [渲染完成]
```

### 场景 B：编辑器页面 — 点击画布偏好菜单的主题项

```
时间轴（从上到下）
│
├─ 1. [用户交互] 点击画布底部工具栏齿轮图标 → 点击 "Dark Mode"/"Light Mode" 菜单项
│
├─ 2. [状态更新] useConfig.toggleDarkMode(!darkmodeEnabled)
│
└─ 3. 与场景 A 完全一致（同一个 Zustand store，同一个 toggleDarkMode）
```

### 场景 C：嵌入页 — 外部 postMessage 传入主题（必须携带 json，双状态分路驱动）

```
时间轴（从上到下）
│
├─ 1. [外部消息] 父页面发送 postMessage({json: "...", options: {theme: "light"}})
│   ↑ 注：若未携带 json 字段，L59 guard 直接 return，整条消息被丢弃
│
├─ 2. [widget.tsx handler] 同步执行两个状态更新（L60-L62）
│   │
│   ├─ 2a. setTheme("light")                       → 本地 state 更新
│   │   │  （驱动 Mantine + styled-components 层）
│   │   │
│   │   └─→ 3. [Mantine 同步] useEffect([theme]) 触发（L77-L79）
│   │       └─ setColorScheme("light")
│   │           └─ localStorage["editor-color-scheme"] = "light"
│   │               └─ Mantine 组件重绘
│   │
│   └─ 2b. toggleDarkMode(false)                   → Zustand store 更新
│        │  （驱动画布层 + 持久化）
│        ├─ persist 中间件：localStorage["config"] = {...darkmodeEnabled: false}
│        │
│        └─→ 4. [styled-components 同步] WidgetPage 重渲染
│        │   └─ <ThemeProvider theme={lightTheme}>  → styled.* 组件重算样式
│        │
│        └─→ 5. [画布同步] GraphView 内部 selector 触发重渲染（L58）
│            └─ <JSONCrack theme="light" />  props 更新（L101）
│                └─ buildCanvasStyle("light") → CSS Variables 更新
│                    └─ 浏览器层叠重绘（无需 React 遍历节点）
│
└─ 6. [渲染完成] 三层系统全部更新完毕
```

**双状态分路说明**：
- 本地 `theme` state → 驱动 Mantine（L77-79）和 styled-components（L82）
- Zustand `darkmodeEnabled` → 驱动 GraphView 画布（GraphView L58, L101）和持久化
- 两者在 handler 中**同步依次执行**，保证三层系统一致

### 场景 D：Chrome 扩展 — 跟随系统主题（仅 CSS Variables，无 UI 框架）

```
时间轴（从上到下）
│
├─ 1. [系统事件] 操作系统切换明暗模式（如 macOS → 系统设置 → 外观 → 浅色）
│
├─ 2. [useSystemTheme] MediaQueryList "change" 事件触发（content-script.tsx L93-L99）
│   └─ setTheme(event.matches ? "dark" : "light")  → 更新本地 state
│
├─ 3. [GraphView 重渲染]（content-script.tsx L305, L351-L358）
│   └─ <JSONCrackComponent theme={theme} />  props 更新
│       └─ buildCanvasStyle(theme, style) 重新执行（JSONCrackComponent.tsx L161）
│           └─ 生成 20+ 个 CSS Variables 的新 style 对象
│               └─ div[style=...] 更新 inline style
│                   └─ 浏览器根据 CSS 变量层叠重绘
│
└─ 4. [渲染完成]
    注意：Chrome 扩展**不使用** Zustand、Mantine、styled-components
          仅依赖 `jsoncrack-react` 包的 CSS Variables 机制
```

### 场景 E：VS Code 扩展 — 跟随 IDE 主题（一次性读取，无运行时切换）

```
时间轴（从上到下）
│
├─ 1. [页面加载] VS Code 注入主题属性到 document.body
│   └─ getTheme() 读取 data-vscode-theme-kind 属性（App.tsx L15-L19）
│       → "vscode-light" 或 "vscode-dark" 或 "vscode-high-contrast" 等
│
├─ 2. [Mantine 锁定] <MantineProvider forceColorScheme={theme}>（App.tsx L53）
│   └─ Mantine 主题在组件挂载时一次性锁定，无运行时监听
│
├─ 3. [画布同步] <JSONCrack json={json} theme={theme} />（App.tsx L56）
│   └─ buildCanvasStyle(theme, style)  → CSS Variables 初始化
│       └─ 画布首次渲染时应用主题
│
└─ 4. [渲染完成]
    注意：VS Code 扩展**不使用** Zustand 和 styled-components
          主题在页面加载时一次性确定，**不监听** IDE 主题变化
          用户需刷新 Webview 才能应用新主题
```

---

## 六、主题 Token 对照表

### 6.1 styled-components 主题 Token 示例

| Token 名 | Dark 值 | Light 值 | 用途 |
|----------|---------|----------|------|
| `BACKGROUND_PRIMARY` | `#36393f` | `#FFFFFF` | 主背景色 |
| `BACKGROUND_SECONDARY` | `#2f3136` | `#f2f3f5` | 次背景色（编辑器面板） |
| `BACKGROUND_NODE` | `#2B2C3E` | `#F6F8FA` | 节点背景 |
| `TOOLBAR_BG` | `#262626` | `#ECECEC` | 工具栏背景 |
| `TEXT_NORMAL` | `#dcddde` | `#2e3338` | 普通文本 |
| `INTERACTIVE_NORMAL` | `#b9bbbe` | `#4f5660` | 可交互元素默认色 |
| `INTERACTIVE_HOVER` | `#dcddde` | `#2e3338` | 可交互元素悬停色 |
| `NODE_COLORS.NODE_KEY` | `#59b8ff` | `#761CEA` | JSON Key 颜色（节点内） |
| `NODE_COLORS.INTEGER` | `#e8c479` | `#FD0079` | 数字类型颜色 |
| `NODE_COLORS.BOOL.TRUE` | `#00DC7D` | `#748700` | true 颜色 |
| `NODE_COLORS.BOOL.FALSE` | `#F85C50` | `#FF0000` | false 颜色 |
| `GRID_BG_COLOR` | `#141414` | `#f7f7f7` | 网格背景色 |
| `GRID_COLOR_PRIMARY` | `#1c1b1b` | `#ebe8e8` | 主轴网格线 |
| `MODAL_BACKGROUND` | `#36393E` | `#FFFFFF` | 弹窗背景 |
| `SILVER_DARK` | `#4D4D4D` | `#CCCCCC` | 工具栏底部分隔线 |

### 6.2 画布 CSS Variables 对应表

| CSS Variable | 来源 Token | 消费位置（仓库相对路径） |
|--------------|------------|--------------------------|
| `--bg-color` | `GRID_BG_COLOR` | `packages/.../JSONCrackStyles.module.css` → `.canvasWrapper` 背景 |
| `--line-color-1` | `GRID_COLOR_PRIMARY` | 同上 → `.showGrid` 大网格线 |
| `--line-color-2` | `GRID_COLOR_SECONDARY` | 同上 → `.showGrid` 小网格线 |
| `--edge-stroke` | 动态计算 | `packages/.../CustomEdge.tsx` → `<Edge style>` |
| `--node-fill` | 动态计算 | `packages/.../CustomNode.tsx` → `<Node style>` + `JSONCrackStyles.module.css` → `:global(rect)` |
| `--node-stroke` | 动态计算 | `packages/.../CustomNode.tsx` → `<Node style>` + `onLeave` 回调 |
| `--interactive-normal` | `INTERACTIVE_NORMAL` | `JSONCrackStyles.module.css` → `:global(text)` + `Node.module.css` → `.collapseButton` |
| `--background-node` | `BACKGROUND_NODE` | （预留，当前未直接消费） |
| `--node-text` | `NODE_COLORS.TEXT` | `Node.module.css` → `.foreignObject` color |
| `--node-key` | `NODE_COLORS.NODE_KEY` | `nodeStyles.ts` → `getTextColor` type=object |
| `--node-value` | `NODE_COLORS.NODE_VALUE` | `nodeStyles.ts` → `getTextColor` 默认返回 |
| `--node-integer` | `NODE_COLORS.INTEGER` | `nodeStyles.ts` → `getTextColor` type=number |
| `--node-null` | `NODE_COLORS.NULL` | `nodeStyles.ts` → `getTextColor` value=null |
| `--node-bool-true` | `NODE_COLORS.BOOL.TRUE` | `nodeStyles.ts` → `getTextColor` value=true |
| `--node-bool-false` | `NODE_COLORS.BOOL.FALSE` | `nodeStyles.ts` → `getTextColor` value=false |
| `--node-child-count` | `NODE_COLORS.CHILD_COUNT` | （预留，预留用于子元素计数） |
| `--node-divider` | `NODE_COLORS.DIVIDER` | `Node.module.css` → `.row` border-bottom + `.collapseButton` border |
| `--text-positive` | `TEXT_POSITIVE` | （预留） |
| `--background-modifier-accent` | `BACKGROUND_MODIFIER_ACCENT` | （预留） |
| `--spinner-track` / `--spinner-head` | 动态计算 | `JSONCrackStyles.module.css` → `.spinner` |
| `--overlay-bg` | 动态计算 | （预留用于加载遮罩） |

### 6.3 WWW 主题 vs 画布主题的 Token 差异

`apps/www/src/constants/theme.ts` 和 `packages/jsoncrack-react/src/theme.ts` 定义了**两套独立的主题数据**，虽然部分颜色值重复，但用途不同：

| 维度 | WWW 主题 (`constants/theme.ts`) | 画布主题 (`packages/.../theme.ts`) |
|------|------|------|
| 消费方式 | styled-components `theme` 对象 | CSS Variables `buildCanvasStyle()` |
| 作用域 | 编辑器布局、工具栏、弹窗 | 画布内节点、连线、网格 |
| 额外包含 | `TOOLBAR_BG`, `SILVER_DARK`, `MODAL_BACKGROUND` 等 UI Token | 无 UI Token，仅画布相关 |
| 节点颜色 | `NODE_COLORS` 嵌套在 dark/light 中 | `NODE_COLORS` 作为顶层字段 |
| 联动方式 | 由 `ThemeProvider` 切换 | 由 `theme` prop 传入 JSONCrack 后映射为 CSS Variables |

---

## 七、性能优化要点

### 7.1 细粒度订阅：Zustand Selector

所有组件**不**使用 `const config = useConfig()` 整体读取，而是精确 selector：

```tsx
// ✅ 正确：仅 darkmodeEnabled 变化时才重渲染
const darkmodeEnabled = useConfig(state => state.darkmodeEnabled);

// ❌ 错误：任何 config 字段变化（如 rulersEnabled）都会触发重渲染
const { darkmodeEnabled } = useConfig();
```

### 7.2 useMemo 缓存计算

```tsx
// packages/jsoncrack-react/src/JSONCrackComponent.tsx L161-165
const canvasStyle = useMemo(
  () => buildCanvasStyle(theme, style),
  [theme, style]
);
```

### 7.3 CSS Variables 优势

相比 inline style 为每个节点单独设置颜色：
- 使用 CSS Variables 只需修改容器 div 的 style
- 浏览器自动层叠传播到所有后代
- 无需逐个遍历节点触发 React 重渲染

### 7.4 React.memo 避免不必要重渲染

`ObjectNode` 和 `TextNode` 都使用 `React.memo` 做浅比较（`packages/jsoncrack-react/src/components/ObjectNode.tsx` L161-L168），当主题切换导致父组件重渲染时，如果节点数据未变则跳过重渲染。节点颜色完全通过 CSS Variables 层叠生效，不依赖 React props 传递。

### 7.5 路径感知避免无关更新

`smartColorSchemeManager` 确保营销页面（首页、文档）始终为 light，不受编辑器主题切换影响，避免这些页面不必要的重渲染。

---

## 八、双持久化机制说明（仅限 `apps/www`）

项目的主题持久化机制**仅存在于 `apps/www`** 下的编辑器页和嵌入页。VS Code 扩展和 Chrome 扩展均不使用任何持久化，主题在每次加载时重新检测。

### 8.1 `apps/www` 的双持久化存储

| localStorage Key | 写入者 | 适用页面 | 用途 | 格式 |
|------------------|--------|---------|------|------|
| `config` | zustand/persist（useConfig） | `/editor`, `/widget` | 应用配置（含 darkmodeEnabled、rulersEnabled 等） | `{"darkmodeEnabled":true,...}` JSON 字符串 |
| `editor-color-scheme` | smartColorSchemeManager | `/editor`, `/widget` | Mantine 颜色方案 | `"dark"` 或 `"light"` 字符串 |

### 8.2 各端持久化对比

| 客户端 | 是否使用 `localStorage["config"]` | 是否使用 `localStorage["editor-color-scheme"]` | 说明 |
|--------|----------------------------------|----------------------------------------------|------|
| 编辑器页 | ✅ | ✅ | Zustand persist 自动写入；Mantine 通过 setColorScheme 写入 |
| 嵌入页 | ✅ | ✅ | 双状态同步写入：toggleDarkMode 写 Zustand；setColorScheme 写 Mantine |
| VS Code 扩展 | ❌ | ❌ | 无 Zustand，每次加载重新读取 `data-vscode-theme-kind` |
| Chrome 扩展 | ❌ | ❌ | 无 Zustand 无 Mantine，每次挂载重新检测 `prefers-color-scheme` |

### 8.3 同步机制

**编辑器页**：`editor.tsx` 通过 `useEffect` 将 Zustand store 同步到 Mantine：
```tsx
useEffect(() => {
  setColorScheme(darkmodeEnabled ? "dark" : "light");
}, [darkmodeEnabled, setColorScheme]);
```
即：**zustand store 是主数据源，Mantine 的 localStorage 是其派生镜像**。

**嵌入页**：`widget.tsx` 中同时维护一个本地 `theme` state，它通过 `postMessage` 写入：
```tsx
// 同步依次执行
setTheme(event.data.options.theme);              // → Mantine + styled-components
toggleDarkMode(event.data.options.theme === "dark");  // → Zustand + 画布 + 持久化
```
本地 `theme` state 通过 `useEffect([theme])` 同步到 Mantine；Zustand `darkmodeEnabled` 自动持久化。两者同步写入保证一致性。

---

## 九、四入口统一架构图（四端差异标注）

```
              ┌────────────────────────────────────────────────────────────────────────┐
              │                    主题切换入口（按客户端可用性）                         │
              ├──────────────┬───────────────────┬──────────────┬───────────────────┤
              │  入口 A       │  入口 B            │  入口 C       │  入口 D           │
              │  顶栏Toggle  │  画布偏好菜单       │  postMessage │  外部客户端       │
              │  ThemeToggle │  Preferences Menu  │  (嵌入页)     │  VS Code / Chrome │
              │ 仅/editor    │ 仅/editor         │ 仅/widget    │  独立客户端        │
              └──────┬───────┴────────┬──────────┴──────┬───────┴──────────┬────────┘
                     │                │                  │                   │
                     ▼                ▼                  │                   │
              ┌────────────┐  ┌────────────┐            │                   │
              │ useConfig  │  │ useConfig  │            │                   │
              │ .toggle    │  │ .toggle    │            │                   │
              │ DarkMode() │  │ DarkMode() │            │                   │
              └─────┬──────┘  └─────┬──────┘            │                   │
                    │               │                   │                   │
                    └───────┬───────┘                   │                   │
                            │                           │                   │
                            ▼                           ▼                   ▼
                   ┌────────────────┐          ┌────────────────┐  ┌─────────────────────┐
                   │ Zustand Store  │          │ widget.tsx      │  │ 环境原生主题          │
                   │ darkmodeEnabled│          │ 双状态同步写入   │  │                     │
                   │ + persist →    │          │ ① setTheme()    │  │ VS Code:            │
                   │ localStorage   │          │   → Mantine +    │  │ data-vscode-theme-   │
                   │ ["config"]     │          │   styled-comp    │  │ kind                │
                   └───┬────┬───┬───┘          │ ② toggleDarkMode│  │ Chrome:             │
                       │    │   │              │   → Zustand +    │  │ prefers-color-      │
                       │    │   │              │   画布 + 持久化  │  │ scheme              │
          ┌────────────┘    │   └───────┐      └──────┬──────────┘  └──────────┬──────────┘
          │                 │           │             │                        │
          ▼                 ▼           ▼             ▼                        ▼
 ┌─────────────────┐ ┌──────────────┐ ┌───────────────┐ ┌─────────────────┐ ┌───────────────────┐
 │ ThemeProvider   │ │ GraphView    │ │ TextEditor    │ │ Mantine +        │ │ <JSONCrack        │
 │ (styled-comp)   │ │ <JSONCrack   │ │ Monaco theme  │ │ styled-comp      │ │   theme={env}     │
 │ darkTheme/      │ │  theme=      │ │ "vs-dark"/    │ │ (从本地 theme    │ │ />                │
 │ lightTheme      │ │  dark/light  │ │ "light"       │ │  state 读取)     │ │ (无 Zustand,      │
 └────────┬────────┘ └──────┬───────┘ └───────────────┘ └────────┬────────┘ │  直接环境主题)     │
          │                 │                                      │          └────────┬──────────┘
          ▼                 ▼                                      ▼                   │
 ┌─────────────────┐ ┌───────────────┐                     ┌───────────────┐           │
 │ styled.* 组件   │ │ CSS Variables │                     │ Mantine 组件   │           │
 │ 重计算样式      │ │ 20+ 变量更新  │                     │ 应用新主题     │           │
 └─────────────────┘ └───────┬───────┘                     └───────────────┘           │
                             │                                                         │
                             └─────────────────────────────────────────────────────────┘
                                       浏览器 CSS 层叠自动生效

┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  四端差异说明：                                                                         │
│  • 编辑器页：经过 Zustand，入口 A+B 可用，完整三层联动 + 双持久化                         │
│  • 嵌入页：入口 C 可用，双状态分路驱动（本地 state → UI 框架；Zustand → 画布+持久化）   │
│  • VS Code 扩展：不经过 Zustand，仅入口 D（环境主题），一次性读取无运行时切换           │
│  • Chrome 扩展：不经过 Zustand/Mantine/styled-comp，仅入口 D + CSS Variables，实时跟随  │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 十、嵌入 API 的主题参数规范

**文件**：`apps/www/src/pages/docs.tsx`（L28-L58）

文档页向外部用户展示了 postMessage API 的使用方式，其中 `options.theme` 是主题控制参数：

```javascript
// 父页面发送主题
iframe.contentWindow.postMessage({
  json: JSON.stringify({ hello: "world" }),
  options: {
    theme: "light",      // "light" | "dark"
    direction: "DOWN"    // "RIGHT" | "DOWN" | "LEFT" | "UP"
  }
}, "*");
```

**就绪信号**：widget 页面加载后会向父页面回传自己的 iframe id：
```typescript
// widget.tsx L52
window.parent.postMessage(window.frameElement?.getAttribute("id"), "*");
```

父页面需要先监听此信号再发送数据：
```javascript
window.addEventListener("message", (event) => {
  if (event.data === "json-crack-embed") {
    // Widget 就绪，可安全发送数据
    iframe.contentWindow.postMessage({ ... }, "*");
  }
});
```

**主题参数的传递边界（基于代码事实）**：

| 边界条件 | 代码事实（仓库相对路径 + 行号） | 说明 |
|---------|-------------------------------|------|
| 必须携带 `json` 字段 | `apps/www/src/pages/widget.tsx` L59：`if (!event.data?.json) return;` | **仅传 theme 不传 json 的消息会被直接丢弃**。主题参数的处理位于该 guard 之后（L60-L63），因此无法单独生效。 |
| theme 取值校验 | `apps/www/src/pages/widget.tsx` L60：`=== "light" \|\| === "dark"` | 非这两个值的 theme 参数将被忽略，不会触发任何主题变更。 |
| 无内嵌 UI 切换入口 | `apps/www/src/features/editor/views/GraphView/index.tsx` L91：`{!isWidget && <Toolbar />}` | Widget 模式下画布底部悬浮工具栏（含 Preferences 菜单中的主题切换项）不渲染，终端用户无法在 iframe 内手动切换主题，只能由父页面通过 postMessage 控制。 |
| 无顶栏切换入口 | `apps/www/src/pages/widget.tsx` 中未引入 `Toolbar` | Widget 页面也没有编辑器顶栏的 ThemeToggle 按钮。 |
| 主题与 JSON 同步更新 | `apps/www/src/pages/widget.tsx` L60-L66 | theme 参数与 json 数据在同一次 handler 调用中依次处理，随后由各自的 useEffect / selector 触发三层主题系统同步。 |
| 持久化会被覆盖 | Zustand persist 机制 | 每次接收到合法的 postMessage（含 theme）都会覆盖 localStorage 中此前保存的主题值。 |

**错误用法示例（被丢弃）**：
```javascript
// ❌ 只有 theme，没有 json → L59 直接 return，theme 参数不生效
iframe.contentWindow.postMessage({
  options: { theme: "light" }
}, "*");
```

**正确用法示例**：
```javascript
// ✅ 同时携带 json 和 theme，两个参数一起生效
iframe.contentWindow.postMessage({
  json: JSON.stringify({ hello: "world" }),
  options: { theme: "light", direction: "RIGHT" }
}, "*");
```

### 10.1 主题参数的内部处理链路（嵌入页）

当父页面发送合法的 postMessage（携带 json + theme）时，widget 内部的处理链路如下：

```
postMessage({json, options: {theme}})
   │
   ├─ [Guard L59] if (!json) return;  ✅ 通过
   │
   ├─ [L60] theme === "light" || "dark"  ✅ 通过
   │   │
   │   ├─→ [L61] setTheme(theme)         → 本地 state 更新
   │   │    │
   │   │    ├─→ [L77-79] useEffect([theme])
   │   │    │   └─ setColorScheme(theme) → Mantine 层更新
   │   │    │
   │   │    └─→ [L82] WidgetPage 重渲染
   │   │         └─ <ThemeProvider theme={...}> → styled-components 层更新
   │   │
   │   └─→ [L62] toggleDarkMode(theme === "dark")  → Zustand store 更新
   │        │
   │        ├─→ persist 中间件 → localStorage["config"] 持久化
   │        │
   │        └─→ GraphView 内部 selector（L58）触发重渲染
   │             └─→ <JSONCrack theme={...} />  → 画布层 CSS Variables 更新
   │
   └─→ [L65-66] JSON 数据 + 布局方向同步
```

**关键结论**：嵌入页的主题参数通过「双状态分路驱动」保证三层系统一致性，而非由单一状态驱动所有层级。
