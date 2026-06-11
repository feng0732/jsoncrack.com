# JSON Crack — Next.js 路由与 SSR 边界代码梳理

> 三个核心问题：`_app` 全局壳与页面在静态导出预渲染时的执行关系、嵌入页 `push` 路由对象是否真的触发跳转、以及关键代码的定位说明。

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

## 2. 问题一：`_app` 全局壳与页面在静态导出预渲染时的执行关系

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

## 3. 问题二：嵌入页路由对象 `push` 是否实际触发跳转

### 3.1 代码定位

**widget 页面对象解构**：`apps/www/src/pages/widget.tsx:L38`
```tsx
const { query, push, isReady } = useRouter();
```

**`push` 的使用位置**：`apps/www/src/pages/widget.tsx:L54`
```tsx
}, [checkEditorSession, clearJson, isReady, push, query.json, query.partner]);
```

### 3.2 结论：`push` 从未被调用

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

### 3.3 `widget` 页面中实际使用的 router API

| API | 位置 | 用途 |
|---|---|---|
| `query` | `widget.tsx:L49` | 读取 `?json=` 和 `?partner=` 查询参数 |
| `isReady` | `widget.tsx:L48` | 等待 router 解析完 URL（`query` 可用的分界） |
| `push` | `widget.tsx:L54`（deps 中） | **未使用**，可能是历史遗留或为未来扩展保留 |

### 3.4 `editor` 页面的 router 使用

`apps/www/src/pages/editor.tsx:L103`
```tsx
const { query, isReady } = useRouter();
```

- 只解构了 `query` 和 `isReady`，**没有解构 `push`**
- 用途与 widget 完全一致：等 `isReady` 后读取 `query?.json` 加载数据

### 3.5 整个项目的路由跳转方式

项目**完全没有程序化路由跳转**（`router.push/replace/back`），所有跳转方式：

| 方式 | 位置 | 说明 |
|---|---|---|
| `<Link href>`（`prefetch={false}`） | Navbar、Footer、Logo | 客户端 SPA 跳转，不整页刷新 |
| `<a href>`（`target="_blank"`） | Navbar 外部链接 | 打开新标签页，指向第三方站点 |
| `<a href="/editor">` | Navbar "Editor" 按钮 | 不使用 Link，直接用 Mantine `component="a"` 渲染 |
| `window.open("/", "_blank")` | Logo 在 widget 页面点击 | 主动在新标签页打开首页，不走 Next.js router |
| `router.reload()` | `_error.tsx:L29` | 500 错误页的"刷新"按钮（整页刷新） |

---

## 4. 问题一延伸：`_document` 在静态导出时的逐页渲染时机

### 4.1 完整构建流程（逐页渲染示意）

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

### 4.2 每页面独立渲染的代码证据

| 证据 | 位置 | 说明 |
|---|---|---|
| `const sheet = new ServerStyleSheet()` | `_document.tsx:L8` | 每次调用 `getInitialProps` 都新建样式表实例 |
| `sheet.seal()` | `_document.tsx:L29` | 密封后不可写入，是"一次性使用"的明确标志 |
| 没有页面级别的 `getInitialProps` | — | 只有 `_document` 有 `getInitialProps`，页面组件都不自己声明 |
| 没有 `getServerSideProps` | — | 无运行时渲染 |
| 只有首页有 `getStaticProps` | `index.tsx:L32-L49` | 构建时 fetch 一次，star 数内嵌到 HTML |

---

## 5. `prefetch={false}` 关闭预取后的跳转方式

### 5.1 所有关闭预取的位置

项目中 6 处 `next/link` **全部** `prefetch={false}`：

| 位置 | 链接目标 | 文件 |
|---|---|---|
| Navbar "Embed" 按钮 | `/docs` | `layout/PageLayout/Navbar.tsx:L90-L98` |
| Footer FAQ | `/#faq` | `layout/PageLayout/Footer.tsx:L54` |
| Footer Docs | `/docs` | `layout/PageLayout/Footer.tsx:L57` |
| Footer Terms | `/legal/terms` | `layout/PageLayout/Footer.tsx:L109` |
| Footer Privacy | `/legal/privacy` | `layout/PageLayout/Footer.tsx:L114` |
| Logo | `/` | `layout/JSONCrackBrandLogo.tsx:L48` |

### 5.2 跳转行为

```
prefetch={false} 语义（静态导出场景）：

  hover / 进入视口 → 不预取目标页面 JS chunk
        ↓
  用户点击 <Link>
        ↓
  Next.js runtime 发起 fetch 请求加载目标页面 JS bundle
        ↓
  JS 加载完成 → React 卸载旧页面组件、挂载新页面组件
        ↓
  history.pushState 更新 URL（不触发整页刷新）
        ↓
  _app 的 useRouter().pathname 变化 → 重新渲染
        ↓
  新页面 useEffect 执行 → 查询参数加载（如果是 editor/widget）
```

### 5.3 特殊情况：widget 页面 Logo 点击

`layout/JSONCrackBrandLogo.tsx:L39-L45`
```tsx
const handleLogoClick = (event) => {
  if (typeof window === "undefined") return;          // SSR 守卫
  if (!window.location.href.includes("widget")) return;  // 只在 widget 页面触发

  event.preventDefault();                               // 阻止 Link 默认 SPA 跳转
  window.open("/", "_blank", "noopener,noreferrer");    // 新标签页打开首页
};
```

- 普通页面点击 Logo：正常 `<Link>` 跳转（SPA 方式）
- widget 页面点击 Logo：`preventDefault` + `window.open`，不经过 Next.js router

---

## 6. 查询参数 `?json=` 在浏览器接管后加载数据的链路

### 6.1 入口：必须等 `router.isReady`

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

### 6.2 为什么必须等 `isReady`

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

### 6.3 `checkEditorSession` 分发逻辑

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

### 6.4 分支 A：远程 URL 加载

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

### 6.5 分支 B：`setContents` 内部链路

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

### 6.6 完整时序图（editor 页面）

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

### 6.7 三个 Store 的分工

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

## 7. 三层边界总结

### 7.1 构建时 ↔ 客户端边界

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

### 7.2 页面间跳转的边界

| 机制 | 是否整页刷新 | `_app` 是否卸载 |
|---|---|---|
| `<Link>` SPA 跳转（`prefetch={false}`） | ❌ 否 | ❌ 保留，只更新 `pathname` |
| `<a href>` 站内链接（如 Navbar "Editor" 按钮） | ✅ 是（整页导航） | ✅ 卸载，下次再重建 |
| `window.open` 新标签页 | 新页面 | 完全独立实例 |
| `router.reload()`（错误页） | ✅ 是（整页刷新） | ✅ 卸载重建 |

### 7.3 查询参数驱动数据加载的边界

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

## 8. 关键文件索引（相对项目根目录）

| 文件路径 | 关键行 | 角色 |
|---|---|---|
| `apps/www/next.config.js` | `L9` | `output: "export"` 全局静态导出 |
| `apps/www/src/pages/_document.tsx` | `L7-L31`, `L33-L45` | 文档层：样式收集、HTML 骨架、每页面构建时执行 |
| `apps/www/src/pages/_app.tsx` | `L70-L121` | 应用壳：Providers、全局 SEO、页面挂载点 |
| `apps/www/src/pages/index.tsx` | `L32-L49` | 首页：唯一使用 `getStaticProps` 的页面 |
| `apps/www/src/pages/editor.tsx` | `L103`, `L115-L117` | 编辑器：`?json=` 入口、`isReady` 守卫 |
| `apps/www/src/pages/widget.tsx` | `L38`, `L47-L54` | 嵌入页：解构 `push` 但未调用，`postMessage` 握手 |
| `apps/www/src/store/useFile.ts` | `L100-L125`, `L129-L140`, `L142-L154` | `setContents`、`fetchUrl`、`checkEditorSession` |
| `apps/www/src/store/useJson.ts` | 全文 | 图形渲染数据源 store |
| `apps/www/src/lib/utils/mantineColorScheme.ts` | `L17-L76` | 颜色方案管理器：按路径决定 light/dark |
| `apps/www/src/layout/PageLayout/Navbar.tsx` | `L90-L98`, `L125-L134` | 导航：`prefetch={false}` 与 `<a>` 混合跳转 |
| `apps/www/src/layout/JSONCrackBrandLogo.tsx` | `L39-L45`, `L48` | Logo：widget 页面 `window.open` 特殊处理 |
