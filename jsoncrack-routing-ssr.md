# JSON Crack — Next.js 路由与 SSR 边界代码梳理

## 1. 项目概览

| 项 | 值 |
|---|---|
| 框架 | Next.js 16.2.6（Pages Router） |
| 输出模式 | `output: "export"` — 静态导出（SSG），无运行时 Node 服务 |
| React 版本 | 19.2.4 |
| 状态管理 | Zustand 5（无 Redux/Context） |
| UI 框架 | Mantine 8 + styled-components 6 |
| 包管理 | pnpm monorepo（turborepo） |
| 主应用入口 | `apps/www/` |

**核心发现**：`next.config.js` 中 `output: "export"` 意味着整个站点在构建时生成纯静态 HTML/JS/CSS，**不存在真正的 SSR 运行时**。所有页面要么是 SSG（`getStaticProps`），要么是纯客户端渲染。这从根本上决定了路由与渲染边界的分工方式。

---

## 2. 路由表

项目使用 **Pages Router**，路由文件位于 `apps/www/src/pages/`。

| URL 路径 | 文件 | 渲染方式 | 是否有 getStaticProps / getServerSideProps |
|---|---|---|---|
| `/` | `pages/index.tsx` | SSG | ✅ `getStaticProps` — 构建时 fetch GitHub stars |
| `/editor` | `pages/editor.tsx` | SSG（骨架）+ 客户端动态 | ❌ 无，纯静态骨架 |
| `/editor?json=<url>` | 同上 | 客户端 fetch | 路由查询参数在 `useEffect` 中处理 |
| `/widget` | `pages/widget.tsx` | SSG（骨架）+ 客户端动态 | ❌ 无，纯静态骨架 |
| `/docs` | `pages/docs.tsx` | SSG | ❌ 无，纯静态内容 |
| `/legal/privacy` | `pages/legal/privacy.tsx` | SSG | ❌ 无，数据来自静态 JSON |
| `/legal/terms` | `pages/legal/terms.tsx` | SSG | ❌ 无，数据来自静态 JSON |
| `404` | `pages/404.tsx` | SSG | ❌ 无 |
| `500` | `pages/_error.tsx` | SSG | ❌ 无 |

---

## 3. 逐段代码分析

### 3.1 `_document.tsx` — 文档层（构建时执行一次）

文件：`apps/www/src/pages/_document.tsx`

```tsx
class MyDocument extends Document {
  static async getInitialProps(ctx: DocumentContext) {
    const sheet = new ServerStyleSheet();
    const originalRenderPage = ctx.renderPage;
    ctx.renderPage = () =>
      originalRenderPage({
        enhanceApp: App => props => sheet.collectStyles(<App {...props} />),
      });
    const initialProps = await Document.getInitialProps(ctx);
    return {
      ...initialProps,
      styles: <>{initialProps.styles}{sheet.getStyleElement()}</>,
    };
  }
}
```

**关键点**：
- 通过 `ServerStyleSheet` 收集 styled-components 的服务端样式，注入到静态 HTML 的 `<head>` 中
- `<ColorSchemeScript />` 注入 Mantine 的颜色方案脚本，防止首屏闪烁（FOUC）
- 此代码在 **构建时** 运行（因为 `output: "export"`），不是运行时 SSR

### 3.2 `_app.tsx` — 应用壳（全局共享层）

文件：`apps/www/src/pages/_app.tsx`

```tsx
function JSONCrackApp({ Component, pageProps }: AppProps) {
  const { pathname } = useRouter();
  const colorSchemeManager = smartColorSchemeManager({
    key: "editor-color-scheme",
    getPathname: () => pathname,
    dynamicPaths: ["/editor", "/widget"],
  });

  return (
    <>
      <Head>{generateDefaultSeo(SEO)}</Head>
      <SoftwareApplicationJsonLd ... />
      <MantineProvider colorSchemeManager={colorSchemeManager} ...>
        <CodeHighlightAdapterProvider adapter={shikiAdapter}>
          <ThemeProvider theme={lightTheme}>
            <Toaster ... />
            <GlobalStyle />
            <GoogleAnalytics trackPageViews />
            <Component {...pageProps} />
          </ThemeProvider>
        </CodeHighlightAdapterProvider>
      </MantineProvider>
    </>
  );
}
```

**SSR/客户端边界分析**：

| 层级 | 组件 | 运行环境 | 说明 |
|---|---|---|---|
| 全局 | `MantineProvider` | SSR + 客户端 | 提供 UI 主题上下文 |
| 全局 | `ThemeProvider`（styled-components） | SSR + 客户端 | 提供样式主题 |
| 全局 | `CodeHighlightAdapterProvider` | SSR + 客户端 | Shiki 高亮器，异步加载 |
| 全局 | `GoogleAnalytics` | 仅客户端 | 条件渲染：`process.env.NEXT_PUBLIC_GA_MEASUREMENT_ID` |
| 全局 | `Toaster`（react-hot-toast） | 仅客户端 | toast 通知容器 |
| 全局 | `GlobalStyle` | SSR + 客户端 | styled-components 全局样式 |

**颜色方案策略**（`smartColorSchemeManager`）：
- 非编辑器页面（`/`、`/docs`、`/legal/*`）→ 强制 `light` 主题
- 编辑器页面（`/editor`、`/widget`）→ 读取 localStorage 中的 `editor-color-scheme`，支持 dark/light 切换
- 这是通过路径判断实现的，确保营销页面始终亮色，编辑器页面跟随用户偏好

### 3.3 `index.tsx` — 首页（SSG + 静态数据）

文件：`apps/www/src/pages/index.tsx`

```tsx
export const getStaticProps = async () => {
  const res = await fetch("https://api.github.com/repos/AykutSarac/jsoncrack.com");
  const data = await res.json();
  return { props: { stars: data?.stargazers_count || 0 } };
};
```

**渲染流程**：
1. **构建时**：`getStaticProps` fetch GitHub API 获取 star 数，生成静态 HTML
2. **构建时**：`_document.tsx` 的 `ServerStyleSheet` 收集所有 styled-components 样式
3. **客户端接管**：React hydration，`HeroSection` 显示 star 数（构建时已内嵌到 HTML）
4. **无运行时数据获取**：star 数在构建时固定，客户端不再请求

页面结构：`Layout > [HeroSection, HeroPreview, Section1, Section2, Section3, Features, FAQ]`

所有子组件（`HeroSection`、`Features` 等）都是纯展示组件，无 `"use client"` 标记（Pages Router 不需要），无 `useEffect` 副作用，完全可 SSR。

### 3.4 `editor.tsx` — 编辑器页面（SSG 骨架 + 大量客户端动态）

文件：`apps/www/src/pages/editor.tsx`

这是整个项目中 **SSR/客户端边界最复杂** 的页面。

**动态导入（`next/dynamic`）— 关键的 SSR 边界控制**：

```tsx
const ModalController = dynamic(() => import("../features/modals/ModalController"));
// ↑ 默认 SSR: true，会在构建时渲染占位

const EditorChoiceModal = dynamic(
  () => import("../features/modals/EditorChoiceModal").then(mod => ({ default: mod.EditorChoiceModal })),
  { ssr: false }
);
// ↑ ssr: false — 纯客户端渲染，构建时不生成 HTML

const ExternalMode = dynamic(() => import("../features/editor/ExternalMode"));
// ↑ 默认 SSR: true

const TextEditor = dynamic(() => import("../features/editor/TextEditor"), { ssr: false });
// ↑ ssr: false — Monaco Editor 无法在 Node 环境运行

const LiveEditor = dynamic(() => import("../features/editor/LiveEditor"), { ssr: false });
// ↑ ssr: false — 依赖 JSONCrack 画布组件，需浏览器 API
```

**SSR 边界决策表**：

| 组件 | SSR? | 原因 |
|---|---|---|
| `Toolbar` | ✅ 是 | 直接导入，纯 UI，无浏览器 API 依赖 |
| `BottomBar` | ✅ 是 | 直接导入，纯 UI |
| `ModalController` | ✅ 是（默认） | 弹窗容器，不影响首屏 |
| `EditorChoiceModal` | ❌ 否 | 弹窗内容，延迟加载 |
| `ExternalMode` | ✅ 是（默认） | 检测域名的弹窗 |
| `TextEditor`（Monaco） | ❌ 否 | Monaco Editor 依赖 DOM API |
| `LiveEditor`（GraphView） | ❌ 否 | 依赖 reaflow/SVG/Canvas 浏览器 API |
| `FullscreenDropzone` | ✅ 是 | 直接导入 |

**客户端副作用链**：

```
EditorPage 挂载
  ├─ useEffect [isReady, query]
  │   └─ checkEditorSession(query?.json)
  │       ├─ 如果 query.json 是 URL → fetchUrl() → 客户端 fetch 远程 JSON
  │       └─ 否则 → 从 sessionStorage 恢复上次编辑内容
  │
  ├─ useEffect [darkmodeEnabled]
  │   └─ setColorScheme(darkmodeEnabled ? "dark" : "light")
  │       └─ 读取 Zustand persist store (localStorage "config")
  │
  └─ Zustand store 链路:
      useFile.setContents → contentToJson → debouncedUpdateJson → useJson.setJson
      ↓
      GraphView 接收 json prop → JSONCrack 组件渲染图形
```

**渲染时序**：
1. 构建时生成静态骨架（Toolbar + BottomBar + 空白 Allotment 布局）
2. 客户端 hydration
3. `TextEditor` 和 `LiveEditor` 动态加载（显示 loading）
4. Monaco Editor 加载完成 → 显示代码编辑器
5. `checkEditorSession` 完成 → JSON 数据流入 → 图形渲染

### 3.5 `widget.tsx` — 嵌入小部件页面

文件：`apps/www/src/pages/widget.tsx`

```tsx
const ModalController = dynamic(() => import("../features/modals/ModalController"), { ssr: false });
const GraphView = dynamic(() => import("../features/editor/views/GraphView").then(c => c.GraphView), { ssr: false });
```

**特点**：
- 无 Layout 包裹（没有 Navbar/Footer），全屏显示图形
- 支持 `postMessage` API：父页面通过 `window.postMessage` 发送 JSON 数据
- 支持 URL 参数 `?json=<url>` 自动加载远程数据
- 客户端初始化时向父窗口发送 iframe id：`window.parent.postMessage(window.frameElement?.getAttribute("id"), "*")`
- 所有核心组件都 `ssr: false`，构建时只生成空壳 HTML

### 3.6 `docs.tsx` — 文档页面

文件：`apps/www/src/pages/docs.tsx`

- 纯静态内容，无 `getStaticProps`
- 使用 Mantine `CodeHighlight` 组件（Shiki 异步加载）
- 嵌入了 CodePen iframe 示例
- 完全 SSR 友好

### 3.7 `legal/privacy.tsx` & `legal/terms.tsx` — 法律页面

- 数据来自静态 JSON 文件（`data/privacy.json`、`data/terms.json`）
- 无动态数据获取
- 完全 SSR 友好

---

## 4. 全局布局层次

```
_document.tsx
  └─ <Html lang="en">
       ├─ <Head>
       │    └─ <ColorSchemeScript />  ← Mantine 防闪烁脚本（构建时注入）
       └─ <body>
            ├─ <Main />               ← 页面内容
            └─ <NextScript />         ← Next.js 运行时 + 页面 JS

_app.tsx
  ├─ <Head>{generateDefaultSeo(SEO)}</Head>  ← 全局 SEO meta
  ├─ <SoftwareApplicationJsonLd />           ← JSON-LD 结构化数据
  ├─ <MantineProvider>                       ← 全局 UI 主题
  │    └─ <CodeHighlightAdapterProvider>     ← 代码高亮
  │         └─ <ThemeProvider>               ← styled-components 主题
  │              ├─ <Toaster />              ← 全局 toast 通知
  │              ├─ <GlobalStyle />          ← 全局样式
  │              ├─ <GoogleAnalytics />       ← GA 追踪
  │              └─ <Component />            ← 当前页面
  └─ (end)

页面级布局：
  营销页面（/, /docs, /legal/*, 404, 500）:
    Layout > [Navbar, Content, Footer]

  编辑器页面（/editor, /widget）:
    无 Layout 包裹，全屏编辑器界面
    自带 ThemeProvider（dark/light 切换）
```

---

## 5. SSR 与客户端接管的分界线

### 5.1 整体渲染模型

```
┌─────────────────────────────────────────────────┐
│              构建时（next build）                  │
│                                                   │
│  getStaticProps (仅 index.tsx)                    │
│     └─ fetch GitHub API → stars 数值              │
│                                                   │
│  _document.tsx getInitialProps                    │
│     └─ ServerStyleSheet 收集 styled-components    │
│                                                   │
│  页面组件渲染为静态 HTML                           │
│     ├─ ssr: true 的组件 → HTML 内嵌               │
│     └─ ssr: false 的组件 → 空白占位               │
│                                                   │
│  输出 → 静态文件（HTML + JS + CSS）               │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│              客户端接管（浏览器）                  │
│                                                   │
│  1. React Hydration                               │
│     └─ 将静态 HTML 与 React 状态绑定              │
│                                                   │
│  2. 动态组件加载（next/dynamic ssr: false）       │
│     ├─ Monaco Editor 加载                         │
│     ├─ GraphView (JSONCrack) 加载                 │
│     └─ Modal 组件延迟加载                         │
│                                                   │
│  3. 客户端数据流                                  │
│     ├─ sessionStorage 恢复编辑内容                │
│     ├─ URL query 参数解析 (?json=...)             │
│     ├─ postMessage 监听（widget 页面）            │
│     └─ Zustand persist 从 localStorage 恢复配置  │
│                                                   │
│  4. 用户交互 → Zustand store 更新 → UI 重渲染     │
└─────────────────────────────────────────────────┘
```

### 5.2 关键分界标记

| 标记 | 含义 | 出现位置 |
|---|---|---|
| `output: "export"` | 整站静态导出，无 SSR 运行时 | `next.config.js` |
| `getStaticProps` | 构建时数据获取 | 仅 `index.tsx` |
| `dynamic(() => ..., { ssr: false })` | 客户端独占组件 | `editor.tsx`、`widget.tsx` |
| `useEffect` | 客户端副作用入口 | 所有页面的动态逻辑 |
| `typeof window !== "undefined"` | 运行环境守卫 | `mantineColorScheme.ts` |
| `sessionStorage` / `localStorage` | 客户端专属存储 | `useFile.ts`、`useConfig.ts` |
| Zustand `persist` middleware | 客户端持久化 | `useConfig.ts` |

### 5.3 为什么 `editor.tsx` 和 `widget.tsx` 大量使用 `ssr: false`

1. **Monaco Editor**：依赖 `document`、`window` 等 DOM API，Node 环境无法运行
2. **JSONCrack（GraphView）**：基于 reaflow，使用 SVG/Canvas 渲染，需要浏览器布局引擎
3. **Modal 弹窗**：延迟加载可减少首屏 JS 体积
4. **构建时 HTML**：这些组件在构建时只输出空占位，客户端加载后再填充

### 5.4 为什么其他页面可以完全 SSR

1. **首页组件**：纯 JSX + styled-components，无浏览器 API 依赖
2. **文档页面**：Mantine UI + 代码高亮，都支持 SSR
3. **法律页面**：静态 JSON 数据 + Mantine 布局
4. **这些页面构建时就能生成完整 HTML**，客户端 hydration 只是绑定交互

---

## 6. 状态管理与数据流

```
┌────────────────────────────────────────────────────────┐
│                    Zustand Stores                       │
│                                                        │
│  useConfig (persist: localStorage "config")            │
│    ├─ darkmodeEnabled: boolean                         │
│    ├─ liveTransformEnabled: boolean                    │
│    ├─ gesturesEnabled: boolean                         │
│    └─ rulersEnabled: boolean                           │
│                                                        │
│  useFile (无 persist，使用 sessionStorage 手动存储)     │
│    ├─ contents: string (编辑器内容)                     │
│    ├─ format: FileFormat (JSON/YAML/CSV/XML)           │
│    ├─ fileData: File | null                            │
│    ├─ error: string | null                             │
│    └─ jsonSchema: object | null                        │
│                                                        │
│  useJson (无 persist)                                  │
│    ├─ json: string (解析后的 JSON 字符串)               │
│    └─ loading: boolean                                 │
│                                                        │
│  useModal (无 persist)                                 │
│    └─ [modalName]: boolean (各弹窗开关)                 │
│                                                        │
│  useGraph (视图状态)                                   │
│    ├─ direction: LayoutDirection                       │
│    ├─ fullscreen: boolean                              │
│    ├─ selectedNode: NodeData                           │
│    └─ viewport / jsonCrackRef                          │
└────────────────────────────────────────────────────────┘

数据流:
  用户输入 → useFile.setContents()
    → contentToJson() 转换格式
    → debouncedUpdateJson() (400ms 防抖)
    → useJson.setJson()
    → GraphView 读取 useJson.json
    → JSONCrack 组件重渲染图形
```

**SSR 影响**：
- `useConfig` 使用 `persist` 中间件，在 SSR/构建时无法访问 localStorage，会使用默认值
- `useFile` 在 `setContents` 中手动操作 `sessionStorage`，有 `typeof window` 隐式检查（通过只在 `useEffect` 中调用）
- `useJson` 的 `loading: true` 初始值意味着构建时 HTML 不会显示图形，客户端接管后变为 `false`

---

## 7. SEO 与元数据策略

| 页面 | SEO 方式 | 关键配置 |
|---|---|---|
| 全局 | `generateDefaultSeo(SEO)` 在 `_app.tsx` | 默认标题、描述、OG 图片 |
| `/` | `generateNextSeo({ canonical: "https://jsoncrack.com" })` | 覆盖 canonical |
| `/editor` | 自定义 title/description | `"Editor \| JSON Crack"` |
| `/widget` | `noindex: true, nofollow: true` | 不被搜索引擎索引 |
| `/docs` | 自定义 title/description | `"Documentation - JSON Crack"` |
| `/legal/*` | 自定义 title/description | 各法律页面标题 |
| `404` | `noindex: true` | 不索引错误页 |

`next-seo` 库在构建时将所有 meta 标签内嵌到静态 HTML，客户端不需要额外请求。

---

## 8. 总结：SSR 边界分工原则

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   营销页面（/, /docs, /legal/*）                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  构建时：完整 HTML 生成（SSG）                       │   │
│   │  客户端：Hydration → 绑定交互                        │   │
│   │  特点：首屏即完整内容，SEO 友好                       │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   编辑器页面（/editor, /widget）                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  构建时：骨架 HTML（Toolbar/BottomBar/布局）         │   │
│   │  客户端：动态加载 Monaco + GraphView                  │   │
│   │         读取 sessionStorage/localStorage 恢复状态     │   │
│   │         解析 URL 参数加载远程数据                     │   │
│   │  特点：首屏快速骨架，核心功能客户端异步加载           │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   分界线：next/dynamic({ ssr: false })                      │
│          + useEffect 执行时机                               │
│          + output: "export" 全局静态化                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
