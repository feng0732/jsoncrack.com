# JSON Crack — Next.js 路由与 SSR 边界代码梳理

> 重点梳理：`_app` 全局壳与页面在静态导出预渲染时的执行关系、`prefetch={false}` 的视口与 hover 行为、错误页与普通页面的路由差异、嵌入页 `push` 是否真的跳转，以及完整的导航方式对照表。

---

## 1. 基础前提：`output: "export"` 静态导出

**配置位置**：`apps/www/next.config.js:L9`

```js
const config = {
  output: "export",
  reactStrictMode: false,
  productionBrowserSourceMaps: true,
  compiler: { styledComponents: true },
  // ...
};
```

`output: "export"` 决定了：
- 没有运行时 Node 服务，`next build` 只产出纯静态文件（HTML/JS/CSS）
- **不存在运行时 SSR**，只有构建时的 SSG（静态生成）
- 所有"SSR 语义"的代码（`getInitialProps`、`ServerStyleSheet`）都**只在构建期**执行
- **按页面逐一**执行，页面间互不共享状态

---

## 2. `_app` 全局壳与页面在静态导出预渲染时的执行关系

### 2.1 相关代码定位

| 文件 | 关键行 | 作用 |
|---|---|---|
| `apps/www/src/pages/_document.tsx` | `L7-L31` | `getInitialProps` 劫持渲染入口 |
| `apps/www/src/pages/_document.tsx` | `L33-L45` | `render()` 输出 HTML 骨架 |
| `apps/www/src/pages/_app.tsx` | `L70-L121` | 全局壳：Provider 层 + 页面挂载点 |
| `apps/www/src/pages/index.tsx` | `L32-L49` | 唯一的 `getStaticProps` |

### 2.2 构建时单个页面的预渲染执行流程

以下流程对每一个页面（`/`、`/editor`、`/widget`、`/docs`、`/legal/*`、`/404`、`/_error`）**独立执行一次**：

```
next build 处理当前页面（例如 /editor）
  │
  ├─ 1. 如果页面有 getStaticProps → 先执行
  │     （仅首页有：fetch GitHub API，产出 { stars }）
  │
  ├─ 2. 调用 _document.getInitialProps(ctx)        ← 每页面一次
  │     │
  │     ├─ L8: new ServerStyleSheet()              ← 每页面新建，样式不共享
  │     ├─ L12-L15: ctx.renderPage = () => ...     ← 劫持渲染函数
  │     │       enhanceApp: App => props =>
  │     │         sheet.collectStyles(<App {...props} />)
  │     │                                          ← _app 被 collectStyles 包裹
  │     │
  │     ├─ L17: Document.getInitialProps(ctx)      ← 触发实际渲染
  │     │       │
  │     │       └─ 内部调用劫持后的 ctx.renderPage()
  │     │            │
  │     │            └─ React 渲染以下组件树：
  │     │                 sheet.collectStyles(
  │     │                   └─ _app (AppProps)        ← 全局壳渲染
  │     │                         │
  │     │                         ├─ <Head>（全局 SEO）
  │     │                         ├─ <SoftwareApplicationJsonLd>
  │     │                         ├─ <MantineProvider>
  │     │                         │    └─ <CodeHighlightAdapterProvider>
  │     │                         │         └─ <ThemeProvider>
  │     │                         │              ├─ <Toaster />
  │     │                         │              ├─ <GlobalStyle />
  │     │                         │              ├─ <GoogleAnalytics />
  │     │                         │              └─ <Component {...pageProps} />  ← 当前页面
  │     │                         │                   （如 EditorPage、HomePage 等）
  │     │                         │
  │     │                         └─ (styled-components 样式被 sheet 收集)
  │     │                 )
  │     │
  │     └─ L19-L27: 组装 initialProps.styles，注入 <style> 标签
  │
  ├─ 3. 调用 MyDocument.render()                  ← 生成完整 HTML
  │     └─ 输出：<Html><Head>...</Head><body><Main/><NextScript/></body></Html>
  │
  ├─ 4. 输出静态文件：out/editor.html
  │     包含：HTML + 内联样式 + NextScript 标签（页面 JS 入口）
  │
  └─ 5. L29: sheet.seal()                         ← 密封，一次性使用
```

### 2.3 关键结论：`_app` 与页面的关系

1. **嵌套关系**：`_app` 是外层壳，页面是 `<Component {...pageProps} />` 子节点。构建时渲染就是一次 React 树的同步渲染。

2. **执行时机**：`_app` 和页面在**同一个 `ctx.renderPage()` 调用内**顺序渲染，先 `_app` 外层，再页面组件。构建时只走同步 render 阶段，**所有 `useEffect` 都不执行**。

3. **每页面独立**：`ServerStyleSheet` 在 `_document.getInitialProps` 中 `new`（`_document.tsx:L8`），最后 `seal()`（`_document.tsx:L29`）。说明每个页面都有**全新的样式表**，页面间样式不交叉。

4. **`getStaticProps` 与 `_document` 的先后**：`getStaticProps` 先执行（首页才有），产出的 `props` 注入到 `AppProps.pageProps`，再传入 `_app`，最终被 `<Component {...pageProps} />` 消费。

### 2.4 构建时 vs 客户端对比

| 阶段 | `_app` 是否执行 | 页面组件是否执行 | `useEffect` 是否执行 | 数据来源 |
|---|---|---|---|---|
| 构建时（SSG） | ✅ 是（render 部分） | ✅ 是（render 部分） | ❌ 否 | `getStaticProps`（仅首页） |
| 客户端 hydration | ✅ 是 | ✅ 是 | ✅ 是 | `useEffect` 中发起 |
| 客户端路由跳转 | ✅ 保留（只更新 pathname） | ❌ 卸载旧页，挂载新页 | ✅ 新页的 effects 执行 | 同上 |

**`_app` 的特殊地位**：路由跳转时 `_app` **不卸载**，只重新渲染；只有页面组件替换。`_app` 的 `useRouter().pathname` 会变化（`_app.tsx:L71`），触发颜色方案管理器重配置。

---

## 3. `_document` 在静态导出时的逐页渲染时机

### 3.1 完整构建流程（逐页渲染示意）

```
next build
  │
  ├─ 扫描 apps/www/src/pages/
  │    发现路由：
  │      index.tsx        → "/"
  │      editor.tsx       → "/editor"
  │      widget.tsx       → "/widget"
  │      docs.tsx         → "/docs"
  │      legal/privacy.tsx→ "/legal/privacy"
  │      legal/terms.tsx  → "/legal/terms"
  │      404.tsx          → "/404"
  │      _error.tsx       → "/_error"
  │
  │  ┌─ 页面 N 处理开始 ──────────────────────────────┐
  │  │                                                 │
  │  │  ① getStaticProps()  ← 仅首页执行               │
  │  │     fetch("https://api.github.com/...")         │
  │  │     → { stars: 28347 }                          │
  │  │                                                 │
  │  │  ② MyDocument.getInitialProps(ctx)              │
  │  │     └─ ctx.pathname = 当前页面路径               │
  │  │                                                 │
  │  │  ③ 劫持 ctx.renderPage                          │
  │  │     └─ sheet.collectStyles(<App {...props} />)  │
  │  │                                                 │
  │  │  ④ Document.getInitialProps(ctx)                │
  │  │     └─ 实际调用 renderPage，触发 React 渲染     │
  │  │        → 输出 HTML 字符串片段                    │
  │  │        → styled-components 样式收集到 sheet     │
  │  │                                                 │
  │  │  ⑤ 合并 styles: {initialProps.styles, sheet}   │
  │  │                                                 │
  │  │  ⑥ MyDocument.render()                          │
  │  │     → 组装完整 HTML 文档：                      │
  │  │       <Html lang="en">                          │
  │  │         <Head>                                  │
  │  │           <ColorSchemeScript />                 │
  │  │           + SEO meta + JSON-LD                  │
  │  │           + styled-components <style>           │
  │  │         </Head>                                 │
  │  │         <body><Main /><NextScript /></body>     │
  │  │       </Html>                                   │
  │  │                                                 │
  │  │  ⑦ sheet.seal()                                 │
  │  │                                                 │
  │  │  ⑧ 写入 out/<page>.html                         │
  │  │                                                 │
  │  └─ 页面 N 处理结束 ──────────────────────────────┘
  │
  │  （对下一个页面重复以上步骤，8 个页面 × 8 次）
  │
  └─ postbuild: next-sitemap 生成 sitemap.xml
```

### 3.2 每页面独立渲染的代码证据

| 证据 | 位置 | 说明 |
|---|---|---|
| `const sheet = new ServerStyleSheet()` | `_document.tsx:L8` | 每次调用 `getInitialProps` 都新建样式表实例 |
| `sheet.seal()` | `_document.tsx:L29` | 密封后不可写入，是"一次性使用"的明确标志 |
| 没有页面级别的 `getInitialProps` | — | 只有 `_document` 有 `getInitialProps`，页面组件都不自己声明 |
| 没有 `getServerSideProps` | — | 无运行时渲染 |
| 只有首页有 `getStaticProps` | `index.tsx:L32-L49` | 构建时 fetch 一次，star 数内嵌到 HTML |

---

## 4. `prefetch={false}` 的视口预取与 hover 行为

### 4.1 Pages Router 中 Link 预取的两种机制

Next.js Pages Router 的 `next/link` 预取有两个触发时机：

| 触发时机 | 默认 `prefetch={true}` | `prefetch={false}` |
|---|---|---|
| **视口预取**（IntersectionObserver） | 链接进入视口时预取 | ❌ 不预取 |
| **鼠标 hover** | hover 时再次确认/加强预取 | ❌ 不预取 |
| **用户点击时** | 已预取，几乎无延迟 | 点击时才开始加载 JS chunk |

> **注意**：Pages Router 中 `prefetch` 的默认值在**生产环境**为 `true`，在开发环境为 `false`。静态导出后走生产构建，默认值为 `true`。

### 4.2 项目中所有 `next/link` 的使用盘点

| 位置 | 链接目标 | `prefetch` 值 | 文件 |
|---|---|---|---|
| Navbar "Embed" 按钮 | `/docs` | `{false}` | `layout/PageLayout/Navbar.tsx:L90-L98` |
| Footer FAQ 链接 | `/#faq` | `{false}` | `layout/PageLayout/Footer.tsx:L54` |
| Footer Docs 链接 | `/docs` | `{false}` | `layout/PageLayout/Footer.tsx:L57` |
| Footer Terms 链接 | `/legal/terms` | `{false}` | `layout/PageLayout/Footer.tsx:L109` |
| Footer Privacy 链接 | `/legal/privacy` | `{false}` | `layout/PageLayout/Footer.tsx:L114` |
| Logo 链接 | `/` | `{false}` | `layout/JSONCrackBrandLogo.tsx:L48` |
| 404 页"Go Home"按钮 | `/` | **未设置（默认 true）** | `pages/404.tsx:L22-L26` |
| HeroSection "GitHub" 按钮 | `https://github.com/...` | 不适用（外部链接，prefetch 无效） | `layout/Landing/HeroSection.tsx:L109` |
| Toolbar "Chrome 扩展" 图标 | `https://chromewebstore...` | 不适用（外部链接） | `features/editor/Toolbar/index.tsx:L88-L92` |

**结论**：
- 主站 6 处内链全部 `prefetch={false}`
- 404 页的返回首页链接**没有**设置 `prefetch`，使用默认值 `true` → 视口进入时会预取首页 JS
- 外部链接的 `prefetch` 属性无意义（Next.js 不会预取外部域）

### 4.3 `prefetch={false}` 下的完整跳转流程（静态导出场景）

```
用户浏览页面，页面上有一个 <Link href="/editor" prefetch={false}>
  │
  ├─ 链接进入视口
  │    └─ ❌ 不触发预取
  │
  ├─ 用户鼠标 hover 到链接上
  │    └─ ❌ 不触发预取
  │
  ├─ 用户点击链接
  │    │
  │    ├─ ① Next.js router 捕获 click 事件
  │    ├─ ② 发起 fetch 请求加载 /editor 页面的 JS chunk
  │    │     （静态导出下：fetch /_next/static/chunks/pages/editor.[hash].js）
  │    │
  │    ├─ ③ JS chunk 下载完成
  │    ├─ ④ React 卸载当前页面组件
  │    ├─ ⑤ React 挂载 /editor 页面组件
  │    ├─ ⑥ history.pushState 更新 URL（不触发整页刷新）
  │    ├─ ⑦ _app pathname 变化 → 重新渲染 → 颜色方案重新计算
  │    └─ ⑧ /editor 页面 useEffect 执行 → 读取 query → 加载数据
  │
  └─ 用户看到新页面内容
```

### 4.4 为什么项目中主站链接全部关闭预取

可能的原因（从代码推断）：
1. **减少带宽**：静态导出后每个页面都有独立 JS chunk，全部预取会浪费流量
2. **编辑器页面体积大**：`/editor` 包含 Monaco Editor + JSONCrack 图形库，chunk 体积大，预取代价高
3. **营销站流量少**：首页 → 编辑器是主要跳转路径，用户点击意向明确时才加载

---

## 5. 错误页保留路由与普通页面的差异

### 5.1 项目中的错误页面

| 页面 | 文件 | 用途 |
|---|---|---|
| 404 页 | `pages/404.tsx` | 路由不匹配时展示 |
| 500 页（`_error`） | `pages/_error.tsx` | 运行时异常时展示 |

### 5.2 与普通页面的相同点

- 都使用 `Layout` 包裹（Navbar + Footer + 内容区，见 `pages/404.tsx:L11` 和 `pages/_error.tsx:L13`）
- 都通过 `Head` 设置 SEO meta（`404.tsx:L12` 为 `noindex: true`）
- 构建时都生成独立的静态 HTML 文件（`out/404.html`、`out/500.html` 等）
- 都经过 `_document.tsx` 的样式收集流程

### 5.3 与普通页面的差异

| 差异点 | 普通页面（如 `/docs`） | 404 页 | 500 页（`_error`） |
|---|---|---|---|
| **路由来源** | pages 目录下有对应文件，明确路由 | 匹配不到任何路由时兜底 | 运行时出错时触发 |
| **SEO 配置** | 正常 index | `noindex: true`（`404.tsx:L12`） | 正常 index |
| **页面内跳转方式** | 多用 `<Link prefetch={false}>` | `<Link href="/">`（无 prefetch，默认 true）（`404.tsx:L22`） | `router.reload()` 整页刷新（`_error.tsx:L29`） |
| **构建时是否生成单文件** | 是 | 是（`404.html`） | 是（但主要用于开发/SSR 错误） |
| **是否保留当前路径** | 是（路径与页面一一对应） | 是（浏览器地址栏保持错误的 URL，页面显示 404 内容） | 是（地址栏不变，内容显示错误） |

### 5.4 关于"保留路由"的说明

在 Next.js Pages Router 中，错误页的**路由保留**行为：

- **404 页**：用户访问 `/nonexistent`，浏览器地址栏仍然显示 `/nonexistent`，但 React 树中渲染的是 `404.tsx` 组件。`useRouter().pathname` 返回 `/nonexistent`，而不是 `/404`。
  - 静态导出场景下：服务器需要配置将未知路径指向 `404.html`（Nginx try_files 等）。
  - 本地开发时：Next.js 自动处理，路径保留。

- **500 页（`_error`）**：运行时异常时渲染 `_error.tsx`，但 URL 保持不变。
  - 静态导出场景下：500 页主要是客户端运行时错误（JS 异常）的降级展示，不是服务端 500。

### 5.5 500 页 `router.reload()` 的特殊性

`pages/_error.tsx:L29`：
```tsx
<Button onClick={() => router.reload()}>Refresh the page</Button>
```

- `router.reload()` 等同于 `window.location.reload()`，是**整页刷新**
- 不是 SPA 导航，`_app` 会完全卸载重建
- 错误页选择 reload 而不是 Link 跳首页，是因为错误状态下客户端路由可能已经不可靠，整页刷新更安全

---

## 6. 导航方式全对照表

### 6.1 项目中实际使用的 6 种导航方式

| 方式 | 典型代码位置 | 视口预取 | hover 预取 | 整页刷新 | `_app` 保留 | 客户端接管 | 浏览器历史 | 场景 |
|---|---|---|---|---|---|---|---|---|
| **`<Link prefetch={false}>`** | Navbar "Embed"、Footer 各链接、Logo | ❌ 否 | ❌ 否 | ❌ 否 | ✅ 是 | ✅ 是（SPA 方式） | pushState | 主站内跳转（6 处） |
| **`<Link>`（默认 prefetch=true）** | 404 页"Go Home"按钮 | ✅ 是 | ✅ 是 | ❌ 否 | ✅ 是 | ✅ 是（SPA 方式） | pushState | 错误页返回首页（1 处） |
| **普通 `<a href>` 站内** | Navbar "Editor" 按钮、HeroSection "Go to Editor" | ❌ 否（原生 a 标签无预取） | ❌ 否 | ✅ 是 | ❌ 否（全量加载） | ❌ 否（整个页面重新初始化） | 整页导航 | 编辑器入口（5 处） |
| **普通 `<a href>` 站外** | Navbar "VS Code"、"Chrome"、"Open Source" 等 | ❌ 不适用 | ❌ 不适用 | 跳转至外部站点 | 不适用 | 不适用 | 整页跳转 | 外部链接（20+ 处） |
| **`window.open(url, "_blank")`** | Logo 在 widget 页点击 | ❌ 否 | ❌ 否 | 新标签页 | 完全独立实例 | 全新实例 | 新标签页 | widget 页 Logo 点击（1 处） |
| **`router.reload()`** | 500 页"Refresh"按钮 | ❌ 否 | ❌ 否 | ✅ 是（整页刷新） | ❌ 否（完全重建） | ❌ 否（重新来过） | replace 当前历史 | 错误恢复（1 处） |

### 6.2 对"客户端接管"的影响说明

| 导航方式 | 接管过程 | 说明 |
|---|---|---|
| `<Link>` SPA 跳转 | 增量接管 | 只替换页面组件，`_app` 及其 Provider 全部保留，store 状态不丢，`useEffect` 按页面增删执行 |
| 普通 `<a>` 整页跳转 | 完全重新接管 | 相当于重新打开页面，`_app`、所有 Provider、Zustand store 全部重建从头来 |
| `window.open` 新标签页 | 全新实例 | 完全独立的 Next.js runtime，store 互不影响 |
| `router.reload()` | 完全重新接管 | 同整页跳转，所有状态清零重建 |

### 6.3 为什么 Editor 入口用普通 `<a>` 而不是 `<Link>`

位置：`layout/PageLayout/Navbar.tsx:L125-L134` 和 `layout/Landing/HeroSection.tsx:L148-L157`

```tsx
<Button component="a" href="/editor">Editor</Button>
```

可能原因（代码推断）：
1. **编辑器与营销页是两套体验**：编辑器是重型应用，整页加载比 SPA 切换更干净
2. **颜色方案差异**：营销页强制 light，编辑器跟随用户偏好。整页刷新可以避免主题切换动画
3. **`_app` 内的 `pathname` 判断**：如果用 SPA 切换，`pathname` 变化会触发 `smartColorSchemeManager` 重新计算主题，可能产生闪烁

---

## 7. 嵌入页 `push` 路由对象是否实际触发跳转

### 7.1 代码定位

**widget 页面对象解构**：`pages/widget.tsx:L38`
```tsx
const { query, push, isReady } = useRouter();
```

**`push` 的使用位置**：`pages/widget.tsx:L54`
```tsx
}, [checkEditorSession, clearJson, isReady, push, query.json, query.partner]);
```

### 7.2 结论：`push` 从未被调用

全项目搜索结果：
- 没有任何 `router.push(...)`、`router.replace(...)`、`router.back(...)` 调用
- `widget.tsx` 中的 `push` 只出现在 `useEffect` 的依赖数组里
- 依赖数组里的变量不会被副作用函数引用执行

**证据链**：

```
widget.tsx:L38  解构了 push
        ↓
widget.tsx:L54  出现在 useEffect deps 中
        ↓
useEffect 内部（L47-L53）:
  - 只调用了 checkEditorSession / clearJson / window.parent.postMessage
  - 完全没有引用 push 变量
        ↓
结论：push 是未使用的依赖，永远不会触发路由跳转
```

### 7.3 `widget` 页面中实际使用的 router API

| API | 位置 | 用途 |
|---|---|---|
| `query` | `widget.tsx:L49` | 读取 `?json=` 和 `?partner=` 查询参数 |
| `isReady` | `widget.tsx:L48` | 等待 router 解析完 URL（`query` 可用的分界） |
| `push` | `widget.tsx:L54`（deps 中） | **未使用**，可能是历史遗留或为未来扩展保留 |

### 7.4 `editor` 页面的 router 使用

`pages/editor.tsx:L103`
```tsx
const { query, isReady } = useRouter();
```

- 只解构了 `query` 和 `isReady`，**没有解构 `push`**
- 用途与 widget 完全一致：等 `isReady` 后读取 `query?.json` 加载数据

---

## 8. 查询参数 `?json=` 在浏览器接管后加载数据的链路

### 8.1 入口：必须等 `router.isReady`

查询参数在静态导出下**不能在构建时/首屏渲染时拿到**。必须等客户端 router 解析 URL 后才可用。

**editor 页面入口**：`pages/editor.tsx:L115-L117`
```tsx
useEffect(() => {
  if (isReady) checkEditorSession(query?.json);
}, [checkEditorSession, isReady, query]);
```

**widget 页面入口**：`pages/widget.tsx:L47-L54`
```tsx
React.useEffect(() => {
  if (isReady) {
    if (typeof query?.json === "string") checkEditorSession(query.json, true);
    else clearJson();

    window.parent.postMessage(window.frameElement?.getAttribute("id"), "*");
  }
}, [checkEditorSession, clearJson, isReady, push, query.json, query.partner]);
```

### 8.2 为什么必须等 `isReady`

静态导出的 Pages Router 中：
```
第一次渲染（hydration）
  router.query = {}  ← 空对象，因为静态 HTML 不含路由参数信息
  query.json = undefined

(isReady === true 后)
  Next.js 在客户端解析 window.location.search
  router.query = { json: "https://api.example.com/data" }
  query.json = "https://api.example.com/data"
```

**不等 `isReady` 的后果**：首次渲染执行 `checkEditorSession(undefined)`，错误地进入 sessionStorage/示例 JSON 分支，等 URL 解析后再切换，造成内容闪烁。

### 8.3 `checkEditorSession` 分发逻辑

`store/useFile.ts:L142-L154`

```tsx
checkEditorSession: (url, widget) => {
  // 分支 A：如果 ?json= 是合法 URL → 远程 fetch
  if (url && typeof url === "string" && isURL(url)) {
    return get().fetchUrl(url);
  }

  // 分支 B：恢复本地内容
  let contents = defaultJson;                                 // 默认示例 JSON
  const sessionContent = sessionStorage.getItem("content");   // 读取 sessionStorage
  const format = sessionStorage.getItem("format");
  if (sessionContent && !widget) contents = sessionContent;  // widget 模式不使用 sessionStorage

  if (format) set({ format });
  get().setContents({ contents, hasChanges: false });
}
```

### 8.4 分支 A：远程 URL 加载

`store/useFile.ts:L129-L140`

```tsx
fetchUrl: async url => {
  try {
    const res = await fetch(url);                          // 纯客户端 fetch
    const json = await res.json();
    const jsonStr = JSON.stringify(json, null, 2);         // 格式化

    get().setContents({ contents: jsonStr });              // 更新编辑器内容
    return useJson.setState({ json: jsonStr, loading: false }); // 立即更新图形 store
  } catch {
    get().clear();
    toast.error("Failed to fetch document from URL!");
  }
},
```

### 8.5 分支 B：`setContents` 内部链路

`store/useFile.ts:L100-L125`

```tsx
setContents: async ({ contents, hasChanges = true, skipUpdate = false, format }) => {
  // ① 更新本地 store
  set({ ...(contents && { contents }), error: null, hasChanges,
       format: format ?? get().format });

  // ② 解析为标准化 JSON
  const json = await contentToJson(get().contents, get().format);

  // ③ 手动模式下如果是"跳过更新"就不渲染图形
  if (!useConfig.getState().liveTransformEnabled && skipUpdate) return;

  // ④ sessionStorage 持久化
  //    条件：有变更 + 内容 < 80KB + 非 iframe + 非 URL 加载场景
  if (get().hasChanges && contents && contents.length < 80_000 && !isIframe() && !isFetchURL) {
    sessionStorage.setItem("content", contents);
    sessionStorage.setItem("format", get().format);
  }

  // ⑤ 400ms 防抖后更新 useJson store，触发图形重绘
  debouncedUpdateJson(json);
},
```

### 8.6 完整时序图（editor 页面）

```
浏览器请求 /editor?json=https://api.example.com/data
  │
  ├─ [构建时生成的静态 editor.html 展示]
  │    Toolbar + BottomBar + 空白布局（骨架）
  │    TextEditor / LiveEditor 区域 = 动态加载中占位
  │
  ├─ React hydration 完成
  │
  ├─ useEffect 首次执行
  │    ├─ isReady === false → checkEditorSession 不执行
  │    └─ TextEditor + LiveEditor 的 dynamic import 开始下载
  │
  ├─ Router 解析完成 → isReady = true → query 被填充
  │
  ├─ useEffect 重新执行
  │    └─ checkEditorSession("https://api.example.com/data")
  │         │
  │         ├─ isURL(url) → true
  │         └─ fetchUrl(url)
  │              ├─ fetch(url)         ← 网络请求
  │              ├─ res.json()         ← 解析响应
  │              ├─ JSON.stringify(.., null, 2)
  │              ├─ setContents({ contents: jsonStr })
  │              │    ├─ useFile 更新
  │              │    ├─ contentToJson 转换
  │              │    └─ debouncedUpdateJson() ← 400ms 计时器启动
  │              └─ useJson.setState({ json, loading: false })
  │
  ├─ Monaco Editor 加载完成 → 显示编辑器代码
  │
  ├─ 400ms 防抖结束 → debouncedUpdateJson 真正写入 useJson
  │
  ├─ GraphView 订阅 useJson.json → JSONCrack 组件重新渲染 SVG 图形
  │
  └─ 最终状态：编辑器代码 + 可视化图形 同步显示
```

### 8.7 三个 Store 的分工

```
useFile store  (store/useFile.ts)
  ├─ contents    编辑器原始文本（支持 JSON/YAML/CSV/XML）
  ├─ format      当前文件格式
  ├─ error       解析错误消息
  ├─ hasChanges  是否有未保存变更
  ├─ setContents 入口：写入内容 → 解析 → 防抖写入 useJson
  ├─ fetchUrl    远程 URL 加载
  └─ checkEditorSession  初始化分发（URL / sessionStorage / 默认值）

useJson store  (store/useJson.ts)
  ├─ json        标准化 JSON 字符串（GraphView 直接读取）
  └─ loading     是否加载中

useGraph store  (features/editor/views/GraphView/stores/useGraph.ts)
  ├─ direction   布局方向（RIGHT/DOWN/LEFT/UP）
  ├─ fullscreen  是否全屏模式
  └─ viewport    视图坐标

数据流：用户输入/URL fetch → useFile → contentToJson → debouncedUpdateJson → useJson → GraphView
```

---

## 9. 三层边界总结

### 9.1 构建时 ↔ 客户端边界

| 侧 | 执行时机 | 可访问资源 | 关键标记 |
|---|---|---|---|
| 构建时（SSG） | `next build` 期间 | Node API、网络 fetch（`getStaticProps`） | `output: "export"`、`getStaticProps`、`ServerStyleSheet` |
| 客户端 | 浏览器加载后 | `window`、`document`、Storage API、DOM | `useEffect`、`typeof window`、`dynamic({ ssr: false })`、`router.isReady` |

**边界标记速查**：
- `output: "export"` — 全局静态化，无运行时 SSR
- `getStaticProps` — 构建时数据（仅首页使用）
- `_document.getInitialProps` — 每页面构建时执行一次
- `dynamic({ ssr: false })` — 组件仅客户端加载（Monaco、GraphView 等）
- `useEffect` — 纯客户端副作用入口
- `router.isReady` — 查询参数从不可用 → 可用的分界点
- `typeof window === "undefined"` — 运行环境守卫

### 9.2 页面间跳转的边界（6 种方式对比）

| 机制 | 整页刷新 | `_app` 卸载 | Store 状态保留 | 预取 | 典型位置 |
|---|---|---|---|---|---|
| `<Link prefetch={false}>` SPA | ❌ 否 | ❌ 否 | ✅ 是 | ❌ 否 | Navbar "Embed"、Footer |
| `<Link>` 默认 prefetch SPA | ❌ 否 | ❌ 否 | ✅ 是 | ✅ 视口预取 | 404 页返回首页 |
| `<a href>` 站内 | ✅ 是 | ✅ 是 | ❌ 否 | ❌ 否 | Navbar "Editor" 按钮 |
| `<a href>` 站外 | 跳转外部 | 不适用 | 不适用 | 不适用 | VS Code、Chrome 等外链 |
| `window.open` 新标签页 | 新标签页 | 全新实例 | 独立 | ❌ 否 | widget 页 Logo |
| `router.reload()` | ✅ 是 | ✅ 是 | ❌ 否 | ❌ 否 | 500 错误页刷新按钮 |

### 9.3 查询参数驱动数据加载的边界

```
构建时 / hydration：       query = {}           无 ?json= 数据
                                  │
                                  ▼
router.isReady = true：    query = { json: "..." }  数据可用
                                  │
                                  ▼
checkEditorSession(url)：  分支判断（URL fetch 或 本地恢复）
                                  │
                                  ▼
setContents()：            解析 → 防抖 → useJson 更新
                                  │
                                  ▼
GraphView 重绘：           JSON → 交互式图形
```

---

## 10. 关键文件索引（相对项目根目录）

| 文件路径 | 关键行 | 角色 |
|---|---|---|
| `apps/www/next.config.js` | `L9` | `output: "export"` 全局静态导出 |
| `apps/www/src/pages/_document.tsx` | `L7-L31`, `L33-L45` | 文档层：样式收集、HTML 骨架、每页面构建时执行 |
| `apps/www/src/pages/_app.tsx` | `L70-L121` | 应用壳：Providers、全局 SEO、页面挂载点 |
| `apps/www/src/pages/index.tsx` | `L32-L49` | 首页：唯一使用 `getStaticProps` 的页面 |
| `apps/www/src/pages/editor.tsx` | `L103`, `L115-L117` | 编辑器：`?json=` 入口、`isReady` 守卫 |
| `apps/www/src/pages/widget.tsx` | `L38`, `L47-L54` | 嵌入页：解构 `push` 但未调用、`postMessage` 握手 |
| `apps/www/src/pages/404.tsx` | `L22-L26` | 404 页：默认 prefetch 的返回首页 Link |
| `apps/www/src/pages/_error.tsx` | `L29` | 500 页：`router.reload()` 整页刷新 |
| `apps/www/src/store/useFile.ts` | `L100-L125`, `L129-L140`, `L142-L154` | `setContents`、`fetchUrl`、`checkEditorSession` |
| `apps/www/src/store/useJson.ts` | 全文 | 图形渲染数据源 store |
| `apps/www/src/lib/utils/mantineColorScheme.ts` | `L17-L76` | 颜色方案管理器：按路径决定 light/dark |
| `apps/www/src/layout/PageLayout/Navbar.tsx` | `L90-L98`, `L125-L134` | 导航：`prefetch={false}` Link 与 `<a>` 混合 |
| `apps/www/src/layout/JSONCrackBrandLogo.tsx` | `L39-L45`, `L48` | Logo：widget 页面 `window.open` 特殊处理 |
