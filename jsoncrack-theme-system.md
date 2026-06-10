# JSON Crack 主题切换与样式系统协作分析

## 一、系统架构概览

JSON Crack 采用了**三层主题系统**协同工作的架构：

| 层级 | 技术方案 | 覆盖范围 | 主题来源 |
|------|----------|----------|----------|
| UI 组件层 | Mantine UI + ColorSchemeManager | 按钮、弹窗、输入框等 Mantine 组件 | `mantineColorScheme.ts` |
| 业务样式层 | styled-components + ThemeProvider | 编辑器布局、工具栏、树视图等自定义组件 | `constants/theme.ts` |
| 画布渲染层 | CSS Variables + inline style | JSON 图形画布（节点、连线、网格等） | `packages/jsoncrack-react/src/theme.ts` |

关键文件定位：
- 全局入口：[_app.tsx](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/apps/www/src/pages/_app.tsx#L1-L123)
- 编辑器页面：[editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/apps/www/src/pages/editor.tsx#L1-L179)
- 状态存储：[useConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/apps/www/src/store/useConfig.ts#L1-L33)

---

## 二、主题状态管理

### 2.1 单一真实数据源：Zustand Store

主题状态的核心是 `darkmodeEnabled` 布尔值，存储在 Zustand 的 `useConfig` store 中：

**文件**：[useConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/apps/www/src/store/useConfig.ts#L1-L33)

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

### 2.2 主题切换触发点：ThemeToggle 组件

**文件**：[ThemeToggle.tsx](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/apps/www/src/features/editor/Toolbar/ThemeToggle.tsx#L1-L17)

```tsx
export const ThemeToggle = () => {
  // 读取状态（通过 selector 订阅，仅当 darkmodeEnabled 变化时重渲染）
  const darkmodeEnabled = useConfig(state => state.darkmodeEnabled);
  const toggleDarkMode = useConfig(state => state.toggleDarkMode);

  return (
    <StyledToolElement
      title={!darkmodeEnabled ? "Dark Mode" : "Light Mode"}
      onClick={() => toggleDarkMode(!darkmodeEnabled)}  // 触发状态变更
    >
      {!darkmodeEnabled ? <FaMoon size="18" /> : <FaSun size="18" />}
    </StyledToolElement>
  );
};
```

### 2.3 Mantine 专用：智能颜色方案管理器

为了实现**路径感知**的主题行为（编辑器页面用动态主题，营销页面强制浅色），项目自定义了 `smartColorSchemeManager`：

**文件**：[mantineColorScheme.ts](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/apps/www/src/lib/utils/mantineColorScheme.ts#L1-L76)

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
    subscribe: () => {},  // 空实现，无需订阅
    unsubscribe: () => {},
    clear: () => { /* 清除内存和 localStorage */ },
  };
}
```

**动态路径配置**（[_app.tsx](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/apps/www/src/pages/_app.tsx#L74-L78)）：
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

**文件**：[_app.tsx](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/apps/www/src/pages/_app.tsx#L91-L118)

```tsx
// Provider 嵌套层次（由外到内）：
<MantineProvider
  colorSchemeManager={colorSchemeManager}  // 第1层：Mantine 主题
  defaultColorScheme="light"
  theme={mantineTheme}
>
  <CodeHighlightAdapterProvider adapter={shikiAdapter}>
    <ThemeProvider theme={lightTheme}>       {/* 第2层：styled-components 主题（营销页面默认浅色）*/}
      <GlobalStyle />                        {/* 全局基础样式 */}
      <Component {...pageProps} />           {/* 子页面可覆盖 ThemeProvider */}
    </ThemeProvider>
  </CodeHighlightAdapterProvider>
</MantineProvider>
```

**注意**：`_app.tsx` 中的 `ThemeProvider` 只设置了 `lightTheme`，这是营销页面的默认值。真正的动态切换在 `editor.tsx` 中重新覆盖。

### 3.2 编辑器页面的关键联动

**文件**：[editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/apps/www/src/pages/editor.tsx#L102-L175)

这是串联三层系统的**核心节点**：

```tsx
const EditorPage = () => {
  // 1. 读取 store 中的主题状态
  const darkmodeEnabled = useConfig(state => state.darkmodeEnabled);

  // 2. 获取 Mantine 的 setColorScheme API
  const { setColorScheme } = useMantineColorScheme();

  // 3. 联动 Mantine 主题层
  useEffect(() => {
    setColorScheme(darkmodeEnabled ? "dark" : "light");
  }, [darkmodeEnabled, setColorScheme]);

  return (
    // 4. 重新提供 styled-components ThemeProvider（覆盖 _app.tsx 中的 lightTheme）
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
用户点击 ThemeToggle
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

**主题定义文件**：[theme.ts](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/apps/www/src/constants/theme.ts#L1-L114)

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

**消费方式**（以 [styles.ts](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/apps/www/src/features/editor/Toolbar/styles.ts#L1-L25) 为例）：
```tsx
export const StyledToolElement = styled.button<{ $hide?: boolean; $highlight?: boolean }>`
  color: ${({ theme }) => theme.INTERACTIVE_NORMAL};    // 从 theme 对象读取
  
  &:hover {
    color: ${({ theme }) => theme.INTERACTIVE_HOVER};   // 不同状态读取不同 Token
  }
`;
```

**TypeScript 类型支持**：[styled.d.ts](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/apps/www/src/types/styled.d.ts#L1-L9)
```typescript
import "styled-components";
import type theme from "../constants/theme";

type CustomTheme = typeof theme;

declare module "styled-components" {
  export interface DefaultTheme extends CustomTheme {}
}
```
这使得在 `styled.*` 中 `theme` 对象具有完整的类型提示。

### 4.2 JSONCrack 画布：CSS 自定义属性（CSS Variables）

画布组件（`jsoncrack-react` 包）采用独立的主题系统，通过**内联 style 注入 CSS Variables**。

**步骤 1：主题定义** - [packages/jsoncrack-react/src/theme.ts](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/packages/jsoncrack-react/src/theme.ts#L1-L71)
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

**步骤 2：构建 CSS Variables** - [canvasHelpers.ts](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/packages/jsoncrack-react/src/canvasHelpers.ts#L32-L65)
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

**步骤 3：注入到组件** - [JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L160-L165)
```tsx
// 使用 useMemo 缓存，避免不必要的重算
const canvasStyle = useMemo(() => buildCanvasStyle(theme, style), [theme, style]);

// 通过 inline style 设置在容器 div 上
return (
  <div ref={containerRef} style={canvasStyle} ... >
    {/* 内部子元素通过 CSS 变量读取颜色 */}
  </div>
);
```

**步骤 4：CSS 消费** - [JSONCrackStyles.module.css](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackStyles.module.css#L1-L75)
```css
.canvasWrapper {
  background-color: var(--bg-color);
}

.showGrid {
  background-image:
    linear-gradient(var(--line-color-1) 1.5px, transparent 1.5px),
    linear-gradient(90deg, var(--line-color-1) 1.5px, transparent 1.5px),
    /* ... */;
}

.canvasWrapper :global(text) {
  fill: var(--interactive-normal) !important;
}

.canvasWrapper :global(rect) {
  fill: var(--node-fill);
}
```

**步骤 5：DOM 元素行内消费** - [nodeStyles.ts](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/packages/jsoncrack-react/src/components/nodeStyles.ts#L1-L13)
```typescript
export const getTextColor = ({ type, value }: TextColorOptions) => {
  if (value === null) return "var(--node-null)";
  if (type === "object") return "var(--node-key)";
  if (type === "number") return "var(--node-integer)";
  if (value === true) return "var(--node-bool-true)";
  if (value === false) return "var(--node-bool-false)";
  return "var(--node-value)";
};

// 在 ObjectNode.tsx 中使用：
// <span style={{ color: getTextColor({ value: row.value, type: typeof row.value }) }}>
```

### 4.3 Monaco 编辑器：专有主题映射

**文件**：[TextEditor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/187-jsoncrack.com/apps/www/src/features/editor/TextEditor.tsx#L23-L96)

Monaco 有自己的主题系统，通过简单的映射实现联动：
```tsx
const theme = useConfig(state => (state.darkmodeEnabled ? "vs-dark" : "light"));

<Editor
  theme={theme}   // 直接传 Monaco 内置主题名
  language={fileType}
  // ...
/>
```

---

## 五、完整响应链路时序分析

### 场景：用户在编辑器页面点击主题切换按钮

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
│   │   │       ├─ StyledEditor.background 重新取值 → 新的 BACKGROUND_SECONDARY
│   │   │       ├─ StyledToolElement.color 重新取值 → 新的 INTERACTIVE_NORMAL
│   │   │       └─ ... 所有使用 ${({theme}) => ...} 的样式全部重新计算
│   │   │
│   │   └─ a2. useEffect 依赖 darkmodeEnabled 触发
│   │       └─ setColorScheme(dark ? "dark" : "light")  → Mantine ColorSchemeManager
│   │           ├─ smartColorSchemeManager.set()
│   │           │   ├─ 更新内存中的 currentColorScheme
│   │           │   └─ localStorage["editor-color-scheme"] = "dark"/"light"
│   │           └─ MantineProvider 更新内部 colorScheme
│   │               └─ 所有 Mantine 组件（Button/Modal/Tooltip...）应用新主题
│   │
│   ├─ 3b. [GraphView 重渲染] - views/GraphView/index.tsx
│   │   └─ <JSONCrack theme={darkmodeEnabled ? "dark" : "light"} /> props 更新
│   │       └─ JSONCrack 组件内部：
│   │           ├─ useMemo(() => buildCanvasStyle(theme, style), [theme, style]) 重新执行
│   │           │   └─ 生成包含 20+ 个 CSS Variables 的新 style 对象
│   │           ├─ div[style=...] 重新应用 inline style
│   │           │   └─ DOM 元素的 --bg-color / --node-fill / --node-key 等变量变更
│   │           └─ 浏览器自动根据 CSS 变量重绘
│   │               ├─ .canvasWrapper 的 background-color 随 --bg-color 更新
│   │               ├─ :global(rect) 的 fill 随 --node-fill 更新
│   │               ├─ 行内 span color 随 var(--node-key) 等更新
│   │               └─ 网格线颜色、连线颜色、spinner 颜色等全部更新
│   │
│   └─ 3c. [TextEditor 重渲染] - TextEditor.tsx
│       └─ theme 变量从 "vs-dark" 切换到 "light"（或反向）
│           └─ Monaco Editor 内部切换主题（字体颜色、背景等）
│
└─ 4. [渲染完成] 浏览器完成所有重绘，用户看到新主题

备注：使用 useMemo 的地方：
  - buildCanvasStyle(theme, style) → 避免非 theme 变更时重算
  - useConfig(state => state.darkmodeEnabled) → 仅当该字段变化才触发组件重渲染
  - styled-components 内部 diffing → 仅 theme 相关样式重新注入 class
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

### 6.2 画布 CSS Variables 对应表

| CSS Variable | 来源 Token | 消费位置 |
|--------------|------------|----------|
| `--bg-color` | `GRID_BG_COLOR` | `.canvasWrapper` 背景 |
| `--line-color-1` | `GRID_COLOR_PRIMARY` | 100px 大网格线 |
| `--line-color-2` | `GRID_COLOR_SECONDARY` | 20px 小网格线 |
| `--node-fill` | 动态计算 | SVG `<rect>` 填充 |
| `--node-stroke` | 动态计算 | SVG `<rect>` 边框 |
| `--node-text` | `NODE_COLORS.TEXT` | 普通文本 |
| `--node-key` | `NODE_COLORS.NODE_KEY` | 对象 Key 文字 |
| `--node-integer` | `NODE_COLORS.INTEGER` | 数字值 |
| `--node-null` | `NODE_COLORS.NULL` | null 值 |
| `--node-bool-true` | `NODE_COLORS.BOOL.TRUE` | true 值 |
| `--node-bool-false` | `NODE_COLORS.BOOL.FALSE` | false 值 |
| `--node-child-count` | `NODE_COLORS.CHILD_COUNT` | 子元素计数文字 |
| `--node-divider` | `NODE_COLORS.DIVIDER` | 节点内分隔线 |
| `--spinner-track` / `--spinner-head` | 动态计算 | 加载 spinner |

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
// JSONCrackComponent.tsx L161-165
const canvasStyle = useMemo(
  () => buildCanvasStyle(theme, style),
  [theme, style]  // 仅 theme 或 style 引用变化才重算
);
```

### 7.3 CSS Variables 优势

相比 inline style 为每个节点单独设置颜色：
- 使用 CSS Variables 只需修改容器 div 的 style
- 浏览器自动层叠传播到所有后代
- 无需逐个遍历节点触发 React 重渲染

### 7.4 路径感知避免无关更新

`smartColorSchemeManager` 确保营销页面（首页、文档）始终为 light，不受编辑器主题切换影响，避免这些页面不必要的重渲染。

---

## 八、双持久化机制说明

项目存在两个独立的主题持久化存储：

| localStorage Key | 写入者 | 用途 | 格式 |
|------------------|--------|------|------|
| `config` | zustand/persist（useConfig） | 应用配置（含 darkmodeEnabled、rulersEnabled 等） | `{"darkmodeEnabled":true,...}` JSON 字符串 |
| `editor-color-scheme` | smartColorSchemeManager | Mantine 颜色方案 | `"dark"` 或 `"light"` 字符串 |

**为什么需要两个？**
1. `config` 是 zustand store 的整体持久化，包含所有配置项
2. `editor-color-scheme` 是 Mantine ColorSchemeManager 协议要求的独立存储
3. 在 `editor.tsx` 中通过 `useEffect` 将两者**同步**：
   ```tsx
   useEffect(() => {
     setColorScheme(darkmodeEnabled ? "dark" : "light");
   }, [darkmodeEnabled, setColorScheme]);
   ```
   即：**zustand store 是主数据源，Mantine 的 localStorage 是其派生镜像**。

---

## 九、架构图

```
                        ┌──────────────────────────┐
                        │  用户点击 ThemeToggle    │
                        └────────────┬─────────────┘
                                     │
                                     ▼
                        ┌──────────────────────────┐
                        │   Zustand useConfig      │
                        │   darkmodeEnabled: bool  │◄─────── localStorage["config"]
                        │   (通过 persist 中间件)   │
                        └──────────┬────┬──────────┘
                                   │    │
                    ┌──────────────┘    └─────────────────┐
                    │                                       │
                    ▼                                       ▼
     ┌──────────────────────────────┐          ┌──────────────────────────┐
     │    EditorPage 重渲染          │          │   GraphView 重渲染       │
     │                              │          │                          │
     │  ┌────────────────────────┐  │          │  JSONCrack theme prop    │
     │  │ styled-components      │  │          │       │                  │
     │  │ ThemeProvider 切换     │  │          │       ▼                  │
     │  │ theme={darkTheme/light}│  │          │  buildCanvasStyle()      │
     │  └───────────┬────────────┘  │          │  生成 CSS Variables     │
     │              │               │          │       │                  │
     │              ▼               │          │       ▼                  │
     │  styled.* 组件样式重计算     │          │  容器 div inline style  │
     │  (BACKGROUND_SECONDARY ...) │          │  (--bg-color 等 20+ 变量)│
     └──────────────────────────────┘          └───────┬──────────────────┘
                    │                                   │
                    │  useEffect 触发                    └─── 浏览器 CSS 层叠生效
                    ▼                                       画布重绘无需 React 遍历
     ┌──────────────────────────────┐
     │  Mantine setColorScheme()    │
     │                              │
     │  localStorage["editor-color- │
     │  scheme"] = "dark"/"light"   │
     │                              │
     │  Mantine 组件主题切换         │
     │  (Button/Modal/Loading...)   │
     └──────────────────────────────┘
```
