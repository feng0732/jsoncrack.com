# JSON Crack — Next.js 路由与 SSR 边界代码梳理

> 重点梳理三个问题：静态导出时 `_document` 按页面渲染的时机、`prefetch={false}` 关闭预取后的跳转方式、查询参数 `?json=` 在浏览器接管后加载数据的完整链路。

---

## 1. 基础前提：`output: "export"` 静态导出

文件：[next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/next.config.js#L9-L9)

```js
const config = {
  output: "export",
  reactStrictMode: false,
  productionBrowserSourceMaps: true,
  compiler: { styledComponents: true },
  // ...
};
```

`output: "export"` 意味着 `next build` 会产出纯静态文件（HTML/JS/CSS），**没有运行时 Node 服务**。
这直接决定了：不存在传统意义的 SSR（服务端渲染），只有 SSG（构建时静态生成）。
所有"SSR"相关代码（`getInitialProps`、`ServerStyleSheet`）都只在**构建期**执行，并且是**按页面逐一执行**。

---

## 2. 问题一：`_document` 在静态导出时按页面渲染的时机

### 2.1 `_document` 的代码结构

文件：[`_document.tsx`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/pages/_document.tsx)

```tsx
class MyDocument extends Document {
  static async getInitialProps(ctx: DocumentContext): Promise<DocumentInitialProps> {
    const sheet = new ServerStyleSheet();              // ① 新建一个 styled-components 样式表
    const originalRenderPage = ctx.renderPage;

    try {
      ctx.renderPage = () =>                           // ② 劫持 renderPage
        originalRenderPage({
          enhanceApp: App => props => sheet.collectStyles(<App {...props} />),
        });                                             // 用 collectStyles 包裹 <App />，收集样式

      const initialProps = await Document.getInitialProps(ctx);  // ③ 触发实际渲染

      return {
        ...initialProps,
        styles: (
          <>
            {initialProps.styles}
            {sheet.getStyleElement()}                  // ④ 把收集到的样式注入 <head>
          </>
        ),
      };
    } finally {
      sheet.seal();
    }
  }

  render() {
    return (
      <Html lang="en">
        <Head>
          <ColorSchemeScript />                        // ⑤ Mantine 防闪烁脚本
        </Head>
        <body>
          <Main />                                     // ⑥ 页面内容占位
          <NextScript />                               // ⑦ Next.js 运行时脚本
        </body>
      </Html>
    );
  }
}
```

### 2.2 静态导出下的逐页渲染时机（构建流程）

```
next build 开始
  │
  ├─ 扫描 pages 目录，确定所有路由
  │    /, /editor, /widget, /docs, /legal/privacy, /legal/terms, /404, /_error
  │
  │  对每一个页面，独立执行：
  │
  │  ┌───────────────────────────────────────────────────────────────┐
  │  │  页面 N 的渲染周期                                             │
  │  │                                                               │
  │  │  1. 执行页面的 getStaticProps（如果有）                         │
  │  │     → 仅 index.tsx 有，fetch GitHub API 获取 stars             │
  │  │                                                               │
  │  │  2. 调用 _document.getInitialProps(ctx)                       │
  │  │     → ctx 中包含当前页面 pathname、query（构建时为空）         │
  │  │                                                               │
  │  │  3. 内部调用 ctx.renderPage() → 被劫持后的版本                 │
  │  │     → render _app.tsx + 当前 page 组件                        │
  │  │     → sheet.collectStyles(...) 一路收样式                     │
  │  │                                                               │
  │  │  4. Document.getInitialProps 拿到渲染后的 HTML 片段           │
  │  │                                                               │
  │  │  5. 拼接 <style> 标签到 styles 字段                           │
  │  │                                                               │
  │  │  6. 调用 MyDocument.render() 生成完整 HTML 文档               │
  │  │     → <Html> / <Head> / <Main> / <NextScript> 全部组装        │
  │  │                                                               │
  │  │  7. 输出 .html 文件到 out 目录                                │
  │  │     （每个页面对应一个独立 .html）                             │
  │  └───────────────────────────────────────────────────────────────┘
  │
  └─ 构建完成 → out/ 目录下所有静态文件
```

### 2.3 逐页渲染的关键证据

| 证据 | 所在文件 | 说明 |
|---|---|---|
| `ServerStyleSheet` 在 `getInitialProps` 内**每次新建** | [`_document.tsx:8`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/pages/_document.tsx#L8-L8) | 每次调用都 `new ServerStyleSheet()`，说明每个页面独立收集样式，样式不共享 |
| `finally { sheet.seal() }` | [`_document.tsx:29`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/pages/_document.tsx#L29-L29) | 密封样式表，防止后续写入，是一次性使用的标志 |
| `ColorSchemeScript` 在 `<Head>` 内 | [`_document.tsx:37`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/pages/_document.tsx#L37-L37) | 每个 HTML 文件都会被注入这段脚本，防止主题闪烁 |
| 没有 `getServerSideProps` | — | 全项目无 `getServerSideProps`，只有 `getStaticProps` 出现在首页 |

### 2.4 `_app.tsx` 与 `_document.tsx` 的分工

- **`_document.tsx`**：只在构建时运行，负责生成 HTML 骨架（`<html>`、`<head>`、`<body>`、样式收集）。
- **`_app.tsx`**：构建时运行一次（用于收集样式和生成 HTML），客户端也运行（用于 hydration 和后续渲染）。负责全局 Provider、全局 SEO、全局样式。

文件：[`_app.tsx`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/pages/_app.tsx)

```
构建时:
  _document.getInitialProps
    → ctx.renderPage()
      → _app.tsx 作为 App 组件被渲染
        → 当前页面组件被渲染
      → 收集 styled-components 样式
    → 组装完整 HTML 文档
    → 输出 .html

客户端:
  浏览器加载 .html
    → Next.js runtime 启动
    → hydrate _app.tsx
    → hydrate 当前页面组件
    → useEffect 等副作用开始执行
```

---

## 3. 问题二：`prefetch={false}` 关闭预取后的跳转方式

### 3.1 所有 `prefetch={false}` 的位置

项目中 `next/link` 的使用**全部**关闭了预取（共 6 处）：

| 位置 | 链接目标 | 所在文件 |
|---|---|---|
| Navbar 中间 "Embed" 按钮 | `/docs` | [`Navbar.tsx:90-98`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/layout/PageLayout/Navbar.tsx#L90-L98) |
| Footer FAQ 链接 | `/#faq` | [`Footer.tsx:54`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/layout/PageLayout/Footer.tsx#L54-L54) |
| Footer Docs 链接 | `/docs` | [`Footer.tsx:57`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/layout/PageLayout/Footer.tsx#L57-L57) |
| Footer Terms 链接 | `/legal/terms` | [`Footer.tsx:109`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/layout/PageLayout/Footer.tsx#L109-L109) |
| Footer Privacy 链接 | `/legal/privacy` | [`Footer.tsx:114`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/layout/PageLayout/Footer.tsx#L114-L114) |
| Logo 链接 | `/` | [`JSONCrackBrandLogo.tsx:48`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/layout/JSONCrackBrandLogo.tsx#L48-L48) |

**Navbar 中部分按钮直接用 `<a>` 而非 Link**：

- "VS Code"、"Chrome"、"Open Source"、"Upgrade" 都是外部链接，直接用 `component="a"` 搭配 `target="_blank"`
- "Editor" 按钮使用 `component="a"` + `href="/editor"`（未使用 `next/link`），见 [`Navbar.tsx:125-134`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/layout/PageLayout/Navbar.tsx#L125-L134)

### 3.2 关闭预取后的跳转行为

Next.js Pages Router 中 `prefetch={false}` 的语义：

```
默认（prefetch=true）:
  鼠标 hover / 链接进入视口时，预取目标页面的 JS chunk
  用户点击时 → 立即执行客户端路由切换（SPA 方式），几乎无延迟

prefetch={false}:
  链接进入视口 / hover 时，不预取任何资源
  用户点击时 → 先发起网络请求加载目标页面 JS chunk → 再执行客户端路由切换
  （有明显延迟，取决于网络和 chunk 大小）
```

在静态导出（`output: "export"`）场景下：

1. **每个页面是独立的 HTML + 独立的 JS chunk**
2. `prefetch={false}` 意味着首屏只加载当前页面的 JS
3. 点击跳转时，Next.js 运行时通过 `fetch` / `XMLHttpRequest` 拉取目标页面的 JS bundle
4. 拉取完成后，React 卸载旧页面、挂载新页面（`_app` 保留，页面组件替换）
5. URL 通过 `history.pushState` 更新，浏览器**不发生整页刷新**

### 3.3 一个特殊例外：Logo 点击在 widget 页面的行为

文件：[`JSONCrackBrandLogo.tsx:39-45`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/layout/JSONCrackBrandLogo.tsx#L39-L45)

```tsx
const handleLogoClick = React.useCallback((event: React.MouseEvent<HTMLAnchorElement>) => {
  if (typeof window === "undefined") return;
  if (!window.location.href.includes("widget")) return;

  event.preventDefault();
  window.open("/", "_blank", "noopener,noreferrer");
}, []);
```

- widget 页面内点击 Logo → 不使用 Next.js 路由，而是 `window.open` 打开新标签页
- 原因：widget 设计为 iframe 嵌入，在 iframe 内跳转没有意义

### 3.4 `useRouter` 的使用情况

| 页面 | 用到的 router API | 用途 |
|---|---|---|
| `_app.tsx` | `pathname` | 判断路径以切换颜色方案管理器模式 |
| `editor.tsx` | `query`, `isReady` | 读取 `?json=` 查询参数加载数据 |
| `widget.tsx` | `query`, `push`, `isReady` | 读取查询参数 + 程序化跳转 |
| `_error.tsx` | `router.reload()` | 500 错误页刷新按钮 |

`_app.tsx` 中 `pathname` 的使用见 [`_app.tsx:71-78`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/pages/_app.tsx#L71-L78)：

```tsx
const { pathname } = useRouter();
const colorSchemeManager = smartColorSchemeManager({
  key: "editor-color-scheme",
  getPathname: () => pathname,
  dynamicPaths: ["/editor", "/widget"],
});
```

这里 `pathname` 是**同步**可读的（因为静态导出时路径在构建时就确定了），所以不会有 `isReady` 问题。

---

## 4. 问题三：查询参数 `?json=` 在浏览器接管后加载数据的链路

### 4.1 入口：`useEffect` + `router.isReady`

查询参数**不能**在 SSR/构建时获取（因为静态导出没有动态路由参数），必须等客户端路由准备就绪。

#### Editor 页面入口

文件：[`editor.tsx:115-117`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/pages/editor.tsx#L115-L117)

```tsx
useEffect(() => {
  if (isReady) checkEditorSession(query?.json);
}, [checkEditorSession, isReady, query]);
```

#### Widget 页面入口

文件：[`widget.tsx:47-54`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/pages/widget.tsx#L47-L54)

```tsx
React.useEffect(() => {
  if (isReady) {
    if (typeof query?.json === "string") checkEditorSession(query.json, true);
    else clearJson();

    window.parent.postMessage(window.frameElement?.getAttribute("id"), "*");
  }
}, [checkEditorSession, clearJson, isReady, push, query.json, query.partner]);
```

关键点：
- **`isReady` 守卫**：确保 router 已初始化、query 已解析
- **widget 多一个 `clearJson()` 分支**：没有 `?json=` 时清空（widget 默认空，editor 默认有示例）
- **widget 多一个 `postMessage`**：通知父窗口 "widget 已就绪"

### 4.2 `checkEditorSession` — 入口分发

文件：[`useFile.ts:142-154`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/store/useFile.ts#L142-L154)

```tsx
checkEditorSession: (url, widget) => {
  // 分支 1：url 是合法 URL → 远程 fetch
  if (url && typeof url === "string" && isURL(url)) {
    return get().fetchUrl(url);
  }

  // 分支 2：从 sessionStorage 恢复（仅非 widget 模式）
  let contents = defaultJson;
  const sessionContent = sessionStorage.getItem("content") as string | null;
  const format = sessionStorage.getItem("format") as FileFormat | null;
  if (sessionContent && !widget) contents = sessionContent;

  if (format) set({ format });
  get().setContents({ contents, hasChanges: false });
}
```

两个分支：
1. **`?json=` 是 URL** → `fetchUrl(url)` 远程拉取
2. **`?json=` 不存在或不是 URL** → 用 `sessionStorage` 里的内容兜底（widget 模式下不用 sessionStorage）

### 4.3 分支 A：URL 远程拉取 `fetchUrl`

文件：[`useFile.ts:129-140`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/store/useFile.ts#L129-L140)

```tsx
fetchUrl: async url => {
  try {
    const res = await fetch(url);
    const json = await res.json();
    const jsonStr = JSON.stringify(json, null, 2);

    get().setContents({ contents: jsonStr });
    return useJson.setState({ json: jsonStr, loading: false });
  } catch {
    get().clear();
    toast.error("Failed to fetch document from URL!");
  }
},
```

特点：
- 纯客户端 `fetch`，跨域依赖目标服务器的 CORS
- 拿到数据后格式化为带缩进的 JSON 字符串
- 同时更新 `useFile.contents`（编辑器内容）和 `useJson.json`（图形渲染数据）

### 4.4 分支 B：`setContents` 内部数据处理链路

文件：[`useFile.ts:100-125`](file:///d:/fz/0601/solo-dogfeeding/code/191-jsoncrack.com/apps/www/src/store/useFile.ts#L100-L125)

```tsx
setContents: async ({ contents, hasChanges = true, skipUpdate = false, format }) => {
  try {
    set({
      ...(contents && { contents }),
      error: null,
      hasChanges,
      format: format ?? get().format,
    });

    const isFetchURL = window.location.href.includes("?");
    const json = await contentToJson(get().contents, get().format);

    if (!useConfig.getState().liveTransformEnabled && skipUpdate) return;

    // sessionStorage 持久化（仅内容 < 80KB 且非 iframe 且非 URL 加载场景）
    if (get().hasChanges && contents && contents.length < 80_000 && !isIframe() && !isFetchURL) {
      sessionStorage.setItem("content", contents);
      sessionStorage.setItem("format", get().format);
      set({ hasChanges: true });
    }

    debouncedUpdateJson(json);  // 400ms 防抖后更新 useJson store
  } catch (error: any) {
    if (error?.mark?.snippet) return set({ error: error.mark.snippet });
    if (error?.message) set({ error: error.message });
    useJson.setState({ loading: false });
  }
},
```

### 4.5 完整链路时序图

```
浏览器加载 /editor?json=https://api.example.com/data

  │
  ├─ [构建时生成的静态 HTML] 展示骨架
  │    Toolbar + BottomBar + 空白布局
  │
  ├─ React hydration 完成
  │
  ├─ useEffect 第一波执行
  │    ├─ router.isReady = false → 不做任何事
  │    └─ (TextEditor / LiveEditor 动态加载中)
  │
  ├─ Router 准备完毕（isReady = true）
  │    └─ query.json = "https://api.example.com/data"
  │
  ├─ useEffect 重新执行
  │    └─ checkEditorSession(query.json)
  │         │
  │         ├─ isURL(url) → true
  │         └─ fetchUrl(url)
  │              │
  │              ├─ await fetch(url)  ← 网络请求
  │              ├─ await res.json()
  │              ├─ JSON.stringify(json, null, 2)
  │              │
  │              ├─ setContents({ contents: jsonStr })
  │              │    ├─ 更新 useFile.contents
  │              │    ├─ contentToJson() 解析
  │              │    └─ debouncedUpdateJson()
  │              │
  │              └─ useJson.setState({ json, loading: false })
  │
  ├─ 400ms 防抖后
  │    └─ debouncedUpdateJson 触发 useJson.setJson
  │
  ├─ GraphView 检测到 useJson.json 变化
  │    └─ JSONCrack 组件重绘图形
  │
  └─ 完成：编辑器代码 + 可视化图形 同步展示
```

### 4.6 为什么必须等 `isReady`

在静态导出的 Pages Router 中：

1. **第一次渲染**（hydration）：`router.query` 是空对象 `{}`，因为静态 HTML 里没有动态参数信息
2. **`isReady` 变为 true 后**：Next.js 在客户端解析 URL，填充 `query` 对象

如果去掉 `isReady` 判断，首次渲染时 `query.json` 为 `undefined`，会错误地进入 "无查询参数" 分支，用示例 JSON 或 sessionStorage 内容初始化，等 URL 解析后再切换，造成闪烁。

### 4.7 两个 Store 的分工

```
useFile store
  ├─ contents: string     ← 编辑器里的原始文本（可能是 JSON/YAML/CSV/XML）
  ├─ format: FileFormat   ← 当前文件格式
  ├─ error: string | null ← 解析错误信息
  └─ setContents()        ← 入口：设置内容 → 解析 → 防抖更新 useJson

useJson store
  ├─ json: string         ← 标准化后的 JSON 字符串（供图形渲染用）
  └─ loading: boolean     ← 是否正在加载

useGraph store
  ├─ direction            ← 布局方向
  ├─ fullscreen           ← 是否全屏
  └─ viewport / jsonCrackRef ← 视图状态
```

数据流向：`URL 或用户输入 → useFile → (contentToJson 转换) → useJson → GraphView 渲染`

---

## 5. 总结：三层边界

### 5.1 构建时 ↔ 客户端 的边界

| 侧 | 执行时机 | 关键代码 | 能访问的资源 |
|---|---|---|---|
| 构建时（SSG） | `next build` | `_document.getInitialProps`、`getStaticProps`、`_app` 渲染、页面组件渲染 | Node.js API、`fs` 模块（shim 了）、网络（`fetch`） |
| 客户端 | 浏览器加载后 | `useEffect`、事件处理、动态 import | `window`、`document`、`localStorage`、`sessionStorage`、DOM API |

**分界标记**：
- `output: "export"` → 一切 SSR 语义的代码都在构建期执行
- `dynamic({ ssr: false })` → 组件只在客户端加载，构建时输出空占位
- `typeof window !== "undefined"` → 运行时环境守卫
- `useEffect` → 客户端副作用入口
- `router.isReady` → 查询参数可用的分界点

### 5.2 页面间跳转的边界

- `prefetch={false}` → 不预取，点击时才加载目标页面 JS
- 静态导出下仍是 SPA 跳转（`history.pushState`），不是整页刷新
- `_app` 保持挂载，页面组件卸载/替换

### 5.3 查询参数驱动数据加载的边界

- URL 中有 `?json=...`，但构建时/首屏渲染时拿不到 → `router.isReady` 是分界
- 数据加载完全走客户端：`fetch` → `useFile` → `useJson` → 图形重渲染
- 不经过任何服务端（因为没有服务端）
