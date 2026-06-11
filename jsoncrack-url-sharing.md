# JSON Crack URL 分享与状态压缩执行路径分析

## 概述

JSON Crack 采用 **URL 参数驱动 + 客户端存储** 的混合方案实现状态分享与恢复。经过深度代码分析，可以明确以下核心结论：

> **🔍 关键结论**
> 1. **一键生成分享链接功能：未实现** - 代码中不存在将当前 JSON 内容编码为可分享 URL 的逻辑
> 2. **压缩过程：完全不存在** - 未使用任何压缩算法（pako、lz-string 等）或 base64 编码
> 3. **远程加载与本地恢复：界限清晰** - 通过 URL 参数正则校验实现严格的分支处理

与传统的 URL 状态压缩方案不同，当前版本并未实现将 JSON 内容直接压缩编码到 URL Hash 或 Query 中，而是通过 **引用远程 URL** + **sessionStorage 本地缓存** 的方式工作。

---

## 一、核心结论验证：一键分享链接功能分析

### 1.1 功能实现状态：❌ 未实现

经过全面代码搜索，**不存在任何"一键生成分享链接"的功能实现**。证据如下：

| 搜索维度 | 搜索结果 | 结论 |
|----------|----------|------|
| 关键词搜索 | `shareLink`, `generateUrl`, `buildUrl`, `createLink`, `getShareUrl`, `shareable` | **无匹配** |
| URL 操作 | `window.open()`, `history.pushState()`, `history.replaceState()` | 仅用于页面跳转，无生成分享链接逻辑 |
| 剪贴板操作 | `navigator.clipboard.writeText()` | 仅用于 Chrome 扩展，无复制 URL 功能 |
| UI 按钮 | 工具栏 [Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/editor/Toolbar/index.tsx) | 仅包含 File/View/Tools 菜单，无 Share 按钮 |
| FileMenu | [FileMenu.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/editor/Toolbar/FileMenu.tsx) | 仅包含 Import/Export，无 Share 选项 |

### 1.2 工具栏 UI 结构验证

**位置**：[Toolbar/index.tsx#L64-L108](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/editor/Toolbar/index.tsx#L64-L108)

```typescript
export const Toolbar = () => {
  return (
    <StyledTools>
      <Group gap="xs" justify="left" w="100%" style={{ flexWrap: "nowrap" }}>
        <JSONCrackLogo />
        <FileMenu />      {/* Import / Export */}
        <ViewMenu />      {/* Graph / Tree 视图切换 */}
        <ToolsMenu />     {/* JQ / JPath / Schema / Generate Type */}
      </Group>
      <Group gap="xs" justify="right" w="100%" style={{ flexWrap: "nowrap" }}>
        <UpgradeToPro />  {/* 专业版升级链接 */}
        <ThemeToggle />
        <ChromeExtensionLink />
        <GitHubLink />
        <FullscreenButton />
      </Group>
    </StyledTools>
  );
};
```

**明确结论**：工具栏没有任何"分享"或"生成链接"的按钮或菜单项。

### 1.3 当前可用的分享方式

用户当前只能通过以下方式"分享"：

1. **手动构造 URL**：用户自行将远程 JSON URL 作为参数拼接
   ```
   https://jsoncrack.com/editor?json=https://example.com/data.json
   ```

2. **导出文件**：通过 FileMenu 的 Export 功能下载本地文件，然后手动分享

3. **Widget 嵌入**：通过 iframe 或 postMessage API 嵌入到其他网站

---

## 二、远程 URL 加载与本地恢复的界限分析

### 2.1 分支判断的核心逻辑

**位置**：[useFile.ts#L142-L154](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L142-L154)

```typescript
checkEditorSession: (url, widget) => {
  // ─── 界限 1：URL 有效性判断 ───
  if (url && typeof url === "string" && isURL(url)) {
    // ▶️ 远程加载分支
    return get().fetchUrl(url);
  }

  // ▶️ 本地恢复分支
  let contents = defaultJson;
  const sessionContent = sessionStorage.getItem("content") as string | null;
  const format = sessionStorage.getItem("format") as FileFormat | null;
  
  // ─── 界限 2：Widget 模式判断 ───
  if (sessionContent && !widget) contents = sessionContent;

  if (format) set({ format });
  get().setContents({ contents, hasChanges: false });
},
```

### 2.2 完整的界限划分表

| 判断条件 | 远程加载分支 | 本地恢复分支 |
|----------|-------------|-------------|
| **URL 参数存在** | `query.json` 非空且为字符串 | `query.json` 为空或非字符串 |
| **URL 格式验证** | 通过 `isURL()` 正则校验 | 未通过正则校验 |
| **数据来源** | 网络 fetch 请求 | sessionStorage 或默认值 |
| **hasChanges** | `true`（默认） | `false`（初始化时） |
| **是否写入 sessionStorage** | ❌ 禁用（见 `isFetchURL` 判断） | ✅ 启用（满足条件时） |
| **Widget 模式** | 不受影响 | Widget 模式下禁用 sessionStorage 恢复 |

### 2.3 第一界限：URL 有效性正则校验

**位置**：[useFile.ts#L61-L65](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L61-L65)

```typescript
const isURL = (value: string) => {
  return /(https?:\/\/(?:www\.|(?!www))[a-zA-Z0-9][a-zA-Z0-9-]+[a-zA-Z0-9]\.[^\s]{2,}|www\.[a-zA-Z0-9][a-zA-Z0-9-]+[a-zA-Z0-9]\.[^\s]{2,}|https?:\/\/(?:www\.|(?!www))[a-zA-Z0-9]+\.[^\s]{2,}|www\.[a-zA-Z0-9]+\.[^\s]{2,})/gi.test(value);
};
```

**校验规则**：
- 必须是 `http://` 或 `https://` 开头，或 `www.` 开头
- 必须包含域名（如 `example.com`）
- 不能包含空白字符

### 2.4 第二界限：isFetchURL 持久化控制

**位置**：[useFile.ts#L109-L118](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L109-L118)

```typescript
const isFetchURL = window.location.href.includes("?");

if (get().hasChanges && contents && contents.length < 80_000 && !isIframe() && !isFetchURL) {
  sessionStorage.setItem("content", contents);
  sessionStorage.setItem("format", get().format);
  set({ hasChanges: true });
}
```

**关键界限逻辑**：
- 当 URL 包含 `?`（即通过远程 URL 加载时），`isFetchURL = true`
- 此时 **不会将内容写入 sessionStorage**，避免远程内容污染本地缓存
- 只有在直接访问 `/editor`（无查询参数）时，才会启用本地持久化

### 2.5 第三界限：Widget 模式隔离

**位置**：[widget.tsx#L47-L54](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/widget.tsx#L47-L54)

```typescript
React.useEffect(() => {
  if (isReady) {
    if (typeof query?.json === "string") checkEditorSession(query.json, true);
    else clearJson();
    // ...
  }
}, [checkEditorSession, clearJson, isReady, push, query.json, query.partner]);
```

**Widget 模式的特殊规则**：
- 调用 `checkEditorSession` 时传入 `widget = true`
- 在 `checkEditorSession` 中，`if (sessionContent && !widget)` 会跳过 sessionStorage 恢复
- 确保嵌入页面不会意外加载主站的本地编辑内容

### 2.6 执行路径流程图

```
用户访问页面
     │
     ├─ 访问 /editor?json=<远程URL>
     │    │
     │    ├─ query.json 存在且为字符串 → isURL() 校验
     │    │    ├─ ✅ 通过 → fetchUrl() 远程加载 → 不写入 sessionStorage
     │    │    └─ ❌ 失败 → 进入本地恢复，但因 isFetchURL=true 仍不写入
     │    │
     │    └─ isFetchURL = true → 禁用 sessionStorage 写入
     │
     └─ 直接访问 /editor（无参数）
          │
          ├─ query.json 为空 → 跳过远程加载
          ├─ 检查 sessionStorage
          │    ├─ 有缓存且非 Widget → 恢复本地内容
          │    └─ 无缓存或 Widget → 使用默认示例
          └─ isFetchURL = false → 启用 sessionStorage 写入（编辑时）
```

---

## 三、压缩过程分析：完全不存在

### 3.1 压缩技术栈验证

经过全面搜索，**代码中不存在任何压缩或编码过程**：

| 压缩/编码技术 | 搜索关键词 | 结果 |
|--------------|-----------|------|
| DEFLATE 压缩 | `pako`, `zlib`, `deflate`, `inflate` | ❌ 无匹配 |
| LZ 系列压缩 | `lz`, `lz-string`, `lz77`, `lz78` | ❌ 无匹配 |
| GZIP 压缩 | `gzip`, `ungzip` | ❌ 无匹配 |
| Brotli 压缩 | `brotli` | ❌ 无匹配 |
| Base64 编码 | `btoa`, `atob`, `base64` | ❌ 无匹配 |
| URL 编码 | `encodeURIComponent`, `decodeURIComponent` | ❌ 无匹配（仅浏览器自动处理 URL 参数） |

### 3.2 状态序列化的实际过程

**位置**：[useFile.ts#L67-L69](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L67-L69)

```typescript
const debouncedUpdateJson = debounce((value: unknown) => {
  useJson.getState().setJson(JSON.stringify(value, null, 2));
}, 400);
```

**序列化过程（无压缩）**：
```
JavaScript 对象
     │
     ▼
JSON.stringify(value, null, 2)  ←─ 标准序列化，2 空格缩进
     │
     ▼
原始字符串（直接存储）
     │
     ├─ sessionStorage.setItem("content", contents)  ←─ 无压缩
     └─ useJson store  ←─ 无压缩
```

### 3.3 sessionStorage 存储的原始内容

**位置**：[useFile.ts#L114-L116](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L114-L116)

```typescript
sessionStorage.setItem("content", contents);  // contents 是原始字符串
sessionStorage.setItem("format", get().format);  // format 是枚举值字符串
```

**存储内容分析**：
- `content`：直接存储编辑器的原始内容（可能是 JSON、YAML、XML、CSV）
- `format`：直接存储文件格式枚举值（如 `"json"`, `"yaml"`）
- **无任何编码或压缩**，用户可在浏览器 DevTools 中直接查看明文内容

### 3.4 反序列化过程（无解压）

**位置**：[useFile.ts#L147-L153](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L147-L153)

```typescript
const sessionContent = sessionStorage.getItem("content") as string | null;
const format = sessionStorage.getItem("format") as FileFormat | null;
if (sessionContent && !widget) contents = sessionContent;
if (format) set({ format });
get().setContents({ contents, hasChanges: false });
```

**反序列化过程（无解压）**：
```
sessionStorage 读取字符串
     │
     ▼
直接赋值给 contents 变量  ←─ 无解码、无解压
     │
     ▼
setContents() → contentToJson() 解析
     │
     ▼
JavaScript 对象
```

---

## 四、URL 分享入口与参数解析

### 4.1 支持的页面路由

| 页面 | 路由 | 核心文件 |
|------|------|----------|
| 编辑器主页面 | `/editor` | [editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/editor.tsx) |
| Widget 嵌入页 | `/widget` | [widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/widget.tsx) |

### 4.2 URL 参数格式

```
https://jsoncrack.com/editor?json=<URL_ENCODED_REMOTE_JSON_URL>
```

**示例**：
```
https://jsoncrack.com/editor?json=https://catfact.ninja/fact
```

**注意**：`json` 参数值必须是一个可访问的远程 URL，不能是直接的 JSON 内容。

### 4.3 参数解析入口

**位置**：[editor.tsx#L103-L117](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/editor.tsx#L103-L117)

```typescript
const EditorPage = () => {
  const { query, isReady } = useRouter();
  const checkEditorSession = useFile(state => state.checkEditorSession);

  useEffect(() => {
    if (isReady) checkEditorSession(query?.json);
  }, [checkEditorSession, isReady, query]);
};
```

### 4.4 远程数据加载：`fetchUrl`

**位置**：[useFile.ts#L129-L141](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L129-L141)

```typescript
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

**远程加载流程**：
1. `fetch(url)` 发送网络请求
2. `res.json()` 解析响应为 JavaScript 对象
3. `JSON.stringify(json, null, 2)` 重新序列化为格式化的 JSON 字符串
4. 调用 `setContents()` 更新状态

---

## 五、状态序列化与持久化机制

### 5.1 序列化方案（无压缩）

当前版本 **未使用任何压缩算法**，也未使用 base64 编码。状态持久化直接使用原始字符串。

### 5.2 sessionStorage 本地缓存

**位置**：[useFile.ts#L114-L118](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L114-L118)

```typescript
if (get().hasChanges && contents && contents.length < 80_000 && !isIframe() && !isFetchURL) {
  sessionStorage.setItem("content", contents);
  sessionStorage.setItem("format", get().format);
  set({ hasChanges: true });
}
```

**缓存条件（全部满足）**：
- ✅ 内容有变更（`hasChanges = true`）
- ✅ 内容非空且长度 < 80,000 字符
- ✅ 非 iframe 嵌入模式
- ✅ URL 不包含查询参数（`isFetchURL = false`，即非远程加载模式）

### 5.3 用户配置持久化（localStorage）

**位置**：[useConfig.ts#L18-L31](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useConfig.ts#L18-L31)

使用 `zustand/persist` 中间件自动序列化到 localStorage：

```typescript
const useConfig = create(
  persist<typeof initialStates & ConfigActions>(
    set => ({ ... }),
    { name: "config" }  // localStorage key
  )
);
```

**持久化配置项**：
- `darkmodeEnabled` - 深色模式开关
- `liveTransformEnabled` - 实时转换开关
- `gesturesEnabled` - 触摸手势开关
- `rulersEnabled` - 标尺显示开关

---

## 六、状态恢复流程（反序列化）

### 6.1 完整恢复链

```
页面加载 → useEffect 触发 checkEditorSession()
     │
     ├─ 1. 检查 URL 参数 query.json
     │    │
     │    ├─ 是有效 URL → fetch 远程 JSON → JSON.parse() → setContents()
     │    │
     │    └─ 非 URL → 进入本地恢复
     │
     ├─ 2. 检查 sessionStorage
     │    │
     │    ├─ 存在 content 且非 Widget → 恢复内容和格式
     │    │
     │    └─ 不存在或 Widget → 使用默认示例 JSON
     │
     └─ 3. 检查 localStorage（zustand persist 自动恢复）
          └─ 恢复用户配置（主题、视图等）
```

### 6.2 Widget 页面的特殊恢复逻辑

**位置**：[widget.tsx#L47-L54](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/widget.tsx#L47-L54)

```typescript
React.useEffect(() => {
  if (isReady) {
    if (typeof query?.json === "string") checkEditorSession(query.json, true);
    else clearJson();
    window.parent.postMessage(window.frameElement?.getAttribute("id"), "*");
  }
}, [checkEditorSession, clearJson, isReady, push, query.json, query.partner]);
```

### 6.3 postMessage 动态数据注入

**位置**：[widget.tsx#L56-L75](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/widget.tsx#L56-L75)

Widget 页面支持通过 postMessage API 动态注入数据（绕过 URL 参数）：

```typescript
window.addEventListener("message", (event) => {
  if (event.data?.json) {
    setContents({ contents: event.data.json, hasChanges: false });
    // 还可设置主题、布局方向等
  }
});
```

---

## 七、JSON 内容格式转换链

### 7.1 多格式支持架构

**位置**：[jsonAdapter.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/lib/utils/jsonAdapter.ts)

```
输入内容（JSON/YAML/XML/CSV）
     │
     ▼
contentToJson() → 解析为统一的 JavaScript 对象
     │
     ▼
debouncedUpdateJson → JSON.stringify(value, null, 2)
     │
     ▼
useJson store → 供 GraphView 渲染
```

### 7.2 序列化细节

**位置**：[useFile.ts#L67-L69](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L67-L69)

```typescript
const debouncedUpdateJson = debounce((value: unknown) => {
  useJson.getState().setJson(JSON.stringify(value, null, 2));
}, 400);
```

- 使用 400ms 防抖避免频繁重渲染
- 序列化时使用 2 空格缩进格式化输出

---

## 八、关键数据结构与 Store 设计

### 8.1 useFile Store（核心状态）

**位置**：[useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts)

| 状态 | 类型 | 说明 |
|------|------|------|
| `contents` | `string` | 原始编辑器内容（可能是 JSON/YAML/XML/CSV） |
| `format` | `FileFormat` | 当前文件格式 |
| `hasChanges` | `boolean` | 是否有未保存变更 |
| `error` | `string \| null` | 解析错误信息 |
| `fileData` | `File \| null` | 关联的后端文件数据 |

### 8.2 useJson Store（渲染状态）

**位置**：[useJson.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useJson.ts)

| 状态 | 类型 | 说明 |
|------|------|------|
| `json` | `string` | 标准化的 JSON 字符串（供渲染） |
| `loading` | `boolean` | 解析加载状态 |

### 8.3 useGraph Store（视图状态）

**位置**：[useGraph.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/editor/views/GraphView/stores/useGraph.ts)

| 状态 | 类型 | 说明 |
|------|------|------|
| `direction` | `LayoutDirection` | 图布局方向（RIGHT/DOWN/LEFT/UP） |
| `fullscreen` | `boolean` | 是否全屏模式 |
| `viewPort` | `ViewPort \| null` | 视口对象（缩放/平移） |
| `collapsedCount` | `number` | 已折叠节点数 |

**注意**：视图状态（缩放、平移、折叠）**不会被序列化到 URL**，仅在内存中维护。

---

## 九、当前方案的技术特点

### 9.1 优点

1. **URL 简洁**：仅传递远程 URL 引用，避免超长 URL 问题
2. **无需服务端**：纯客户端实现，无状态压缩/解压服务端依赖
3. **多格式透明**：支持 JSON/YAML/XML/CSV 多种输入格式
4. **性能可控**：80KB 大小限制防止 sessionStorage 溢出
5. **界限清晰**：远程加载与本地恢复严格隔离，避免缓存污染

### 9.2 局限

1. **无一键分享**：用户无法直接分享当前编辑的 JSON 内容
2. **无内容压缩**：必须通过远程 URL 才能分享，本地内容无法编码到 URL
3. **视图状态丢失**：刷新页面后缩放、平移、折叠状态会丢失
4. **依赖 CORS**：远程 URL 必须支持跨域访问
5. **会话级存储**：sessionStorage 在关闭标签后即清除

### 9.3 潜在的优化方向

如需要实现"一键分享 JSON 内容"功能，可引入：

| 技术 | 库 | 压缩率 | URL 编码 |
|------|----|--------|----------|
| LZ-String | `lz-string` | ~60% | `encodeURIComponent` |
| DEFLATE | `pako` | ~70% | base64 + URL 安全编码 |
| Brotli | `brotli-wasm` | ~75% | base64 + URL 安全编码 |

典型实现模式：
```javascript
// 压缩 → 编码 → URL Hash
const compressed = pako.deflateRaw(JSON.stringify(data), { level: 9 });
const encoded = btoa(String.fromCharCode(...compressed));
window.location.hash = `d=${encodeURIComponent(encoded)}`;

// 恢复 → 解码 → 解压
const hashData = new URLSearchParams(window.location.hash.slice(1)).get('d');
const compressed = Uint8Array.from(atob(decodeURIComponent(hashData)), c => c.charCodeAt(0));
const data = JSON.parse(pako.inflateRaw(compressed, { to: 'string' }));
```

---

## 十、关键代码引用速查表

| 功能模块 | 文件路径 | 核心行号 |
|----------|----------|----------|
| URL 参数解析入口 | [editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/editor.tsx) | L103-L117 |
| Widget URL 解析 | [widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/widget.tsx) | L37-L54 |
| 会话检查主逻辑（分支界限） | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L142-L154 |
| 远程 URL 加载 | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L129-L141 |
| sessionStorage 缓存（isFetchURL 界限） | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L109-L118 |
| URL 正则校验 | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L61-L65 |
| 防抖序列化（无压缩） | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L67-L69 |
| 多格式适配器 | [jsonAdapter.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/lib/utils/jsonAdapter.ts) | L4-L93 |
| 配置持久化 | [useConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useConfig.ts) | L18-L31 |
| postMessage API | [widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/widget.tsx) | L56-L75 |
| 工具栏 UI（无分享按钮） | [Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/editor/Toolbar/index.tsx) | L64-L108 |
| FileMenu（无分享选项） | [FileMenu.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/editor/Toolbar/FileMenu.tsx) | L9-L41 |
| 嵌入文档示例 | [docs.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/docs.tsx) | L28-L58 |
| ImportModal URL 导入 | [ImportModal/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/modals/ImportModal/index.tsx) | L19-L32 |
| ImportModal 文件导入 | [ImportModal/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/modals/ImportModal/index.tsx) | L33-L46 |
| format 默认值（initialStates） | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L50-L57 |
| setContents format 回退逻辑 | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L100-L107 |
| BottomBar 格式切换器 | [BottomBar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/editor/BottomBar.tsx) | L150-L171 |
| setFile（服务端文件加载） | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L78-L82 |
| setFormat 格式转换 | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L86-L99 |

---

## 十一、URL 导入入口 vs 地址参数入口：代码实现差异分析

### 11.1 两条远程加载路径概述

JSON Crack 中存在**两条独立的远程数据加载路径**，它们的触发时机、代码位置和对本地缓存的影响完全不同：

| 路径名称 | 触发方式 | 核心文件 | 调用入口 |
|----------|---------|----------|---------|
| **地址参数入口** | 用户访问带参数的 URL：`/editor?json=<URL>` | [editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/editor.tsx) | `useEffect` → `checkEditorSession(query.json)` |
| **URL 导入入口** | 用户在编辑器内通过 ImportModal 手动输入 URL | [ImportModal/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/modals/ImportModal/index.tsx) | `handleImportFile()` → `fetch(url)` |

---

### 11.2 地址参数入口的代码实现

**入口位置**：[editor.tsx#L103-L117](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/editor.tsx#L103-L117)

```typescript
const EditorPage = () => {
  const { query, isReady } = useRouter();
  const checkEditorSession = useFile(state => state.checkEditorSession);

  // 页面初始化时自动触发
  useEffect(() => {
    if (isReady) checkEditorSession(query?.json);
  }, [checkEditorSession, isReady, query]);
};
```

**调用链路**：
```
页面加载 → useEffect 自动触发
     │
     ▼
checkEditorSession(url, widget=false)
     │
     ▼
isURL(url) 正则校验通过
     │
     ▼
fetchUrl(url)   ← useFile store 内部方法
     │
     ▼
get().setContents({ contents: jsonStr })
```

**关键代码**：[useFile.ts#L142-L145](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L142-L145)
```typescript
checkEditorSession: (url, widget) => {
  if (url && typeof url === "string" && isURL(url)) {
    return get().fetchUrl(url);  // 通过 store 内部的 fetchUrl 方法
  }
  // ...
},
```

---

### 11.3 URL 导入入口的代码实现

**入口位置**：[ImportModal/index.tsx#L11-L47](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/modals/ImportModal/index.tsx#L11-L47)

```typescript
export const ImportModal = ({ opened, onClose }: ModalProps) => {
  const [url, setURL] = React.useState("");
  const setContents = useFile(state => state.setContents);
  const setFormat = useFile(state => state.setFormat);

  const handleImportFile = () => {
    if (url) {
      toast.loading("Loading...", { id: "toastFetch" });
      gaEvent("fetch_url");

      // ⚠️ 关键：直接在组件内调用 fetch，绕过了 store 的 fetchUrl 方法
      return fetch(url)
        .then(res => res.json())
        .then(json => {
          setContents({ contents: JSON.stringify(json, null, 2) });
          onClose();
        })
        .catch(() => toast.error("Failed to fetch JSON!"))
        .finally(() => toast.dismiss("toastFetch"));
    } else if (file) {
      // 文件导入逻辑...
    }
  };
  // ...
};
```

**调用链路**：
```
用户点击 FileMenu → Import → 打开 ImportModal
     │
     ▼
用户在 TextInput 中输入 URL → 点击 Import 按钮
     │
     ▼
handleImportFile()  ← 组件内部方法
     │
     ▼
fetch(url)   ← 直接在组件中调用原生 fetch
     │
     ▼
res.json() → JSON.stringify(json, null, 2)
     │
     ▼
setContents({ contents: jsonStr })
```

---

### 11.4 两条路径的代码实现差异对比

| 对比维度 | 地址参数入口 | URL 导入入口 |
|----------|-------------|-------------|
| **触发时机** | 页面加载时自动触发（useEffect） | 用户手动点击 Import 按钮触发 |
| **代码位置** | store 层：[useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | 组件层：[ImportModal/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/modals/ImportModal/index.tsx) |
| **fetch 调用位置** | store 内部 `fetchUrl()` 方法 | ImportModal 组件内直接调用原生 `fetch()` |
| **URL 有效性校验** | ✅ `isURL()` 严格正则校验 | ❌ **无校验**，直接将用户输入传给 fetch |
| **hasChanges 默认值** | `true`（setContents 未显式指定） | `true`（setContents 未显式指定，使用默认值） |
| **错误提示文本** | `"Failed to fetch document from URL!"` | `"Failed to fetch JSON!"` |
| **Google Analytics** | ❌ 无埋点 | ✅ `gaEvent("fetch_url")` 埋点 |
| **Loading 状态** | 依赖 useJson store 的 `loading` 状态 | 使用 `toast.loading("Loading...")` 独立提示 |
| **错误时行为** | `get().clear()` 清空内容 | 仅显示错误 toast，不清空当前内容 |
| **Widget 模式支持** | ✅ 支持（通过 widget 参数控制） | ❌ Widget 页面无 ImportModal |

---

### 11.5 对本地缓存与恢复界限的影响：关键发现

两条路径最核心的差异在于 **`isFetchURL` 标志的计算结果不同**，这直接决定了远程加载的内容是否会被写入本地 sessionStorage。

#### `isFetchURL` 判断逻辑

**位置**：[useFile.ts#L109](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L109)

```typescript
const isFetchURL = window.location.href.includes("?");
```

> **⚠️ 关键漏洞**：`isFetchURL` 的判断依据是**当前浏览器地址栏 URL 是否包含 `?`**，而不是数据的实际来源。

---

#### 路径一：地址参数入口 → `isFetchURL = true`

**场景**：用户访问 `https://jsoncrack.com/editor?json=https://example.com/data.json`

```
window.location.href = "https://jsoncrack.com/editor?json=https://example.com/data.json"
                              │
                              └─ 包含 "?" → isFetchURL = true
                                               │
                                               ▼
setContents() 中缓存条件检查：
  get().hasChanges → true
  contents 存在且 < 80KB → true
  !isIframe() → true
  !isFetchURL → false  ← ⛔ 条件不满足
                                           │
                                           ▼
                              ❌ sessionStorage.setItem 不会执行
                              （远程内容不会污染本地缓存）
```

**结果**：✅ **远程内容被隔离**，不会写入 sessionStorage

---

#### 路径二：URL 导入入口 → `isFetchURL = false`

**场景**：用户直接访问 `https://jsoncrack.com/editor`，然后通过 ImportModal 输入 URL 导入

```
window.location.href = "https://jsoncrack.com/editor"
                              │
                              └─ 不包含 "?" → isFetchURL = false
                                               │
                                               ▼
setContents() 中缓存条件检查：
  get().hasChanges → true (默认值)
  contents 存在且 < 80KB → true
  !isIframe() → true
  !isFetchURL → true  ← ✅ 所有条件满足
                                           │
                                           ▼
                              ✅ sessionStorage.setItem 被执行
                              （远程内容被写入本地缓存！）
```

**结果**：⚠️ **远程内容泄漏到本地缓存**，用户后续刷新页面时会恢复这份远程数据

---

### 11.6 缓存界限穿透的完整演示

| 步骤 | 用户操作 | 浏览器 URL | `isFetchURL` | 数据来源 | sessionStorage 状态 |
|------|---------|-----------|-------------|---------|-------------------|
| 1 | 直接访问编辑器 | `/editor` | `false` | 默认示例 | 空（或历史内容） |
| 2 | 打开 ImportModal，输入 `https://a.com/data.json`，点击 Import | `/editor` | `false` | 远程 fetch | ✅ **被写入**：`data.json` 的内容 |
| 3 | 刷新页面 | `/editor` | `false` | sessionStorage 恢复 | 恢复 `data.json` 的内容 |

**对比地址参数入口**：

| 步骤 | 用户操作 | 浏览器 URL | `isFetchURL` | 数据来源 | sessionStorage 状态 |
|------|---------|-----------|-------------|---------|-------------------|
| 1 | 访问 `/editor?json=https://a.com/data.json` | `/editor?json=...` | `true` | 远程 fetch | ❌ **不写入** |
| 2 | 刷新页面 | `/editor?json=...` | `true` | 再次远程 fetch | 保持为空或历史内容 |

---

### 11.7 代码设计问题总结

#### 问题 1：`isFetchURL` 判断维度错误

**当前实现**：[useFile.ts#L109](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L109)
```typescript
const isFetchURL = window.location.href.includes("?");  // 以浏览器地址栏为判断依据
```

**问题**：判断的是"页面 URL 是否有查询参数"，而不是"当前内容是否来自远程加载"。URL 导入入口虽然也是远程加载，但因页面 URL 无 `?` 导致被误判为"本地编辑"。

**建议修复方向**：在 `setContents` 的参数中增加 `source` 字段（如 `"remote" | "local" | "default"`），或在 store 中维护 `dataSource` 状态，而不是依赖地址栏判断。

```typescript
// 建议的修复方案
setContents: async ({ contents, hasChanges = true, source = "local", format }) => {
  // ...
  if (source === "local" && ...) {
    sessionStorage.setItem("content", contents);
  }
  // ...
};
```

#### 问题 2：URL 导入入口缺少 URL 校验

**当前实现**：[ImportModal/index.tsx#L25-L32](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/modals/ImportModal/index.tsx#L25-L32)
```typescript
return fetch(url)  // 直接使用用户输入，无任何校验
  .then(res => res.json())
```

**问题**：用户可能输入非 URL 字符串、恶意 URL 或不支持的协议，缺少与 `checkEditorSession` 中一致的 `isURL()` 校验。

#### 问题 3：两条路径 fetch 逻辑重复

- 地址参数入口：store 层 `fetchUrl()` 方法
- URL 导入入口：组件层直接调用 `fetch()`

两处实现了几乎相同的 fetch → parse → stringify → setContents 逻辑，违反 DRY 原则，且导致行为不一致（错误提示、缓存策略等）。

**建议**：URL 导入入口应统一调用 `useFile.getState().fetchUrl(url)`，而非在组件内重新实现。

---

## 十二、逐段代码分析：URL 导入后 format 为什么不会自动重置为 JSON

### 12.1 问题来源

很多人会有这个疑问：**通过 URL 加载远程 JSON 后，当前格式（format）为什么不会自动重置为 JSON？**

答案在于代码实现的一个细节设计——**URL 导入路径没有显式设置 format**，format 始终保留调用前的状态。下面逐段追踪代码，完整还原 format 状态的变化路径。

---

### 12.2 初始状态：format 的默认值

**位置**：[useFile.ts#L50-L57](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L50-L57)

```typescript
const initialStates = {
  fileData: null as File | null,
  format: FileFormat.JSON,  // ← 初始值：json
  contents: defaultJson,
  error: null as any,
  hasChanges: false,
  jsonSchema: null as object | null,
};
```

**结论**：页面首次加载时，format 默认值为 `FileFormat.JSON`（即 `"json"`）。

---

### 12.3 地址参数入口：checkEditorSession → fetchUrl 路径

让我们逐段追踪用户访问 `/editor?json=<URL>` 时，format 经历了什么。

#### 第一步：checkEditorSession 入口

**位置**：[useFile.ts#L142-L154](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L142-L154)

```typescript
checkEditorSession: (url, widget) => {
  // 分支一：URL 有效 → 调用 fetchUrl
  if (url && typeof url === "string" && isURL(url)) {
    return get().fetchUrl(url);  // ← 注意：只传了 url，没有传 format
  }

  // 分支二：本地恢复
  let contents = defaultJson;
  const sessionContent = sessionStorage.getItem("content") as string | null;
  const format = sessionStorage.getItem("format") as FileFormat | null;
  if (sessionContent && !widget) contents = sessionContent;

  if (format) set({ format });  // ← 本地恢复时会设置 format
  get().setContents({ contents, hasChanges: false });
},
```

**关键观察**：
- ✅ **本地恢复分支**：会从 sessionStorage 读取 format 并调用 `set({ format })`
- ❌ **远程加载分支**：直接调用 `get().fetchUrl(url)`，**完全没有涉及 format 的设置**

#### 第二步：fetchUrl 内部

**位置**：[useFile.ts#L129-L141](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L129-L141)

```typescript
fetchUrl: async url => {
  try {
    const res = await fetch(url);
    const json = await res.json();
    const jsonStr = JSON.stringify(json, null, 2);

    get().setContents({ contents: jsonStr });  // ← 只传了 contents，没有传 format！
    return useJson.setState({ json: jsonStr, loading: false });
  } catch {
    get().clear();
    toast.error("Failed to fetch document from URL!");
  }
},
```

**关键观察**：
- `fetchUrl` 的职责是"获取 URL 内容并设置到编辑器"
- 它只设置了 `contents`，**完全没有修改 format**
- 既然没有修改，format 就保持调用 `fetchUrl` 之前的值

#### 第三步：setContents 中 format 的处理

**位置**：[useFile.ts#L100-L107](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L100-L107)

```typescript
setContents: async ({ contents, hasChanges = true, skipUpdate = false, format }) => {
  try {
    set({
      ...(contents && { contents }),
      error: null,
      hasChanges,
      format: format ?? get().format,  // ← format 未指定时，使用当前值！
    });
```

**关键逻辑**：`format: format ?? get().format`
- `??` 是空值合并运算符
- 如果传入的 `format` 是 `undefined`（即没传），就使用 `get().format`（当前值）
- 由于 `fetchUrl` 调用 `setContents` 时**没有传 format 参数**，所以 format 保持原样

#### 第四步：保存到 sessionStorage

**位置**：[useFile.ts#L114-L117](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L114-L117)

```typescript
if (get().hasChanges && contents && contents.length < 80_000 && !isIframe() && !isFetchURL) {
  sessionStorage.setItem("content", contents);
  sessionStorage.setItem("format", get().format);  // ← 保存的是当前 format
  set({ hasChanges: true });
}
```

> **⚠️ 但注意**：地址参数入口时 `isFetchURL = true`（因为 URL 含 `?`），所以**不会写入 sessionStorage**。这部分内容我们在第十一章已经讨论过。

---

### 12.4 场景演示：先切 YAML 再用 URL 导入

让我们用一个具体场景来演示 format 不会自动重置的问题。

| 步骤 | 用户操作 | format 值 | contents | 说明 |
|------|---------|----------|----------|------|
| 1 | 打开编辑器 | `"json"` | 默认示例 | 初始状态 |
| 2 | 在 BottomBar 点击切换到 YAML | `"yaml"` | 转换为 YAML 格式的内容 | 用户主动切换格式 |
| 3 | 打开 ImportModal，输入远程 JSON URL，点击 Import | **`"yaml"`** | **JSON 字符串** | ⚠️ 问题出现了！ |
| 4 | contentToJson 用 yaml 解析器解析 JSON 内容 | `"yaml"` | JSON 字符串 | ❌ 解析失败，显示错误 |

**第三步的详细执行路径**：

```
用户在 ImportModal 输入 URL → 点击 Import
    │
    ▼
handleImportFile()  ← [ImportModal/index.tsx#L18-L32]
    │
    ├─ toast.loading("Loading...")
    ├─ gaEvent("fetch_url")
    │
    ▼
fetch(url)
    │
    ▼
res.json()  ← 服务器返回的 JSON 被解析为对象
    │
    ▼
JSON.stringify(json, null, 2)  ← 重新序列化为 JSON 字符串
    │
    ▼
setContents({ contents: jsonStr })  ← 只传了 contents！没有传 format！
    │
    ├─ set({ contents: jsonStr, format: undefined ?? get().format })
    │                      └─ format = undefined → 用当前值 "yaml"
    │
    └─ format 保持为 "yaml" 不变 ← ⚠️ 这就是问题所在
```

---

### 12.5 ImportModal 中：URL 导入 vs 文件导入的对比

**ImportModal 有两个导入分支**，它们对 format 的处理方式完全不同：

| 对比项 | URL 导入分支 | 文件导入分支 |
|--------|------------|-----------|
| **代码位置** | [ImportModal/index.tsx#L19-L32](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/modals/ImportModal/index.tsx#L19-L32) | [ImportModal/index.tsx#L33-L46](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/modals/ImportModal/index.tsx#L33-L46) |
| **format 处理** | ❌ 不设置 format，保持当前值 | ✅ `setFormat(format as FileFormat)` |
| **format 来源** | 无（保持原样） | 从文件名后缀提取（`file.name`） |
| **内容与格式一致性** | ⚠️ 可能不一致（内容是 JSON，但 format 可能不是） | ✅ 一致（文件名后缀决定格式） |
| **解析结果** | 可能失败（format 不匹配时） | 通常成功 |

**文件导入分支的代码**：
```typescript
} else if (file) {
  const lastIndex = file.name.lastIndexOf(".");
  const format = file.name.substring(lastIndex + 1);
  setFormat(format as FileFormat);  // ← ✅ 显式设置 format

  file.text().then(text => {
    setContents({ contents: text });  // ← 后设置内容
    // ...
  });
  // ...
}
```

> **对比结论**：文件导入时**先 setFormat 再 setContents**，确保 format 与内容一致；但 URL 导入时**只 setContents，不设置 format**，导致 format 保持为调用前的任意值。

---

### 12.6 三条远程/外部加载路径的 format 处理对比

JSON Crack 中至少有三条外部数据加载路径，它们对 format 的处理各不相同：

| 加载路径 | 触发方式 | 是否设置 format | format 来源 | 内容与格式一致性 |
|---------|---------|---------------|------------|---------------|
| **地址参数入口** | `/editor?json=<URL>` | ❌ 不设置 | 保持当前值（默认 json） | ⚠️ 多数情况下一致（默认 json），但用户切过格式就不一致 |
| **ImportModal URL 导入** | File → Import → 输入 URL | ❌ 不设置 | 保持当前值 | ⚠️ 可能不一致 |
| **ImportModal 文件导入** | File → Import → 上传文件 | ✅ 设置 | 文件名后缀 | ✅ 一致 |
| **setFile（服务端文件）** | 从云端加载文件 | ✅ 设置 | `fileData.format` | ✅ 一致 |

**setFile 的实现**（格式处理最严谨）：
```typescript
setFile: fileData => {
  set({ fileData, format: fileData.format || FileFormat.JSON });  // ← 显式设置 format
  get().setContents({ contents: fileData.content, hasChanges: false });
  gaEvent("set_content", { label: fileData.format });
},
```

---

### 12.7 对 sessionStorage 保存与恢复的连锁影响

现在把 format 问题和 sessionStorage 串联起来分析。

#### 场景 A：地址参数入口（isFetchURL = true）

```
用户访问 /editor?json=<URL>
    │
    ├─ format 保持为 json（初始值）
    ├─ contents 设置为远程 JSON 内容
    ├─ isFetchURL = true（URL 含 ?）
    │
    ▼
    不写入 sessionStorage
    （因为 !isFetchURL 条件不满足）
```

**影响**：
- sessionStorage 中保留的是上一次本地编辑的内容和 format
- 远程加载的内容**不会污染** sessionStorage
- 但如果用户之前切过格式、存了 YAML，刷新后从 sessionStorage 恢复时还是 YAML，不会因为 URL 加载了 JSON 就自动变回去

#### 场景 B：ImportModal URL 导入（isFetchURL = false）

```
用户直接访问 /editor → 切到 YAML → ImportModal 导入远程 URL
    │
    ├─ format 保持为 yaml（用户之前切的）
    ├─ contents 设置为远程 JSON 内容（注意：内容是 JSON 字符串！）
    ├─ isFetchURL = false（URL 不含 ?）
    │
    ▼
    满足所有保存条件 → 写入 sessionStorage
    ├─ content = JSON 字符串
    └─ format = "yaml"  ← ⚠️ 错误！内容是 JSON，但 format 标成了 yaml
```

**影响——保存时**：
- sessionStorage 中存储了 **content 是 JSON 但 format 是 yaml** 的"错误配对"
- 下次恢复时，会用 YAML 解析器去解析 JSON 内容，**必然失败**

**影响——恢复时**：
```
用户刷新页面
    │
    ├─ checkEditorSession() 调用
    │    ├─ query.json 为空 → 走本地恢复分支
    │    ├─ 从 sessionStorage 读 format = "yaml"
    │    ├─ 先 set({ format: "yaml" })
    │    └─ 再 setContents({ contents: <JSON 字符串> })
    │
    ├─ setContents 内部调用 contentToJson(contents, "yaml")
    │    └─ YAML 解析器解析 JSON 字符串 → 报错！
    │
    └─ BottomBar 显示 "Invalid" + 错误信息
```

> **⚠️ 这是一个隐藏 Bug**：用户先切到非 JSON 格式，再通过 ImportModal 导入远程 URL，会导致 sessionStorage 中 content 与 format 不匹配，刷新页面后解析失败。

---

### 12.8 根因总结与修复建议

#### 根本原因

URL 导入路径（包括地址参数入口和 ImportModal URL 导入）**没有显式将 format 设置为 JSON**，而 URL fetch 得到的内容一定是 JSON（因为 `res.json()` 解析后再 `JSON.stringify`），导致 content 与 format 可能不一致。

#### 为什么会这样设计？

可能的原因：
1. **默认就是 JSON**：多数情况下用户不会切换格式，所以不设置也没问题
2. **职责分离**：`fetchUrl` 的职责是"加载 URL 内容"，不负责改格式
3. **历史遗留**：早期只有 JSON 格式，后来加了多格式支持但 URL 导入路径没同步更新

#### 修复建议

**方案 1：在 fetchUrl 中显式设置 format 为 JSON**

```typescript
fetchUrl: async url => {
  try {
    const res = await fetch(url);
    const json = await res.json();
    const jsonStr = JSON.stringify(json, null, 2);

    get().setContents({ contents: jsonStr, format: FileFormat.JSON });  // ← 加上 format
    return useJson.setState({ json: jsonStr, loading: false });
  } catch {
    get().clear();
    toast.error("Failed to fetch document from URL!");
  }
},
```

**方案 2：ImportModal URL 导入分支先调用 setFormat**

```typescript
if (url) {
  setFile(null);
  setFormat(FileFormat.JSON);  // ← 先设置格式
  toast.loading("Loading...", { id: "toastFetch" });
  // ...
}
```

**方案 3：在 setContents 中添加智能判断（不推荐）**

不建议，因为会增加复杂度且破坏单一职责。

---

## 十三、URL 导入 JSON 格式对 sessionStorage 保存与恢复的影响

### 13.1 sessionStorage 保存的两个核心字段

**位置**：[useFile.ts#L114-L117](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L114-L117)

```typescript
if (get().hasChanges && contents && contents.length < 80_000 && !isIframe() && !isFetchURL) {
  sessionStorage.setItem("content", contents);   // 原始内容字符串
  sessionStorage.setItem("format", get().format); // 格式标识
  set({ hasChanges: true });
}
```

sessionStorage 中保存的是**一对关联数据**：
- `content`：编辑器原始内容（可以是 JSON/YAML/XML/CSV 任意格式的字符串）
- `format`：内容对应的格式枚举值（`"json" | "yaml" | "xml" | "csv"`）

---

### 12.2 多格式（JSON/YAML/XML/CSV）对保存的影响

#### URL 导入入口的格式默认值

**位置**：[ImportModal/index.tsx#L25-L28](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/modals/ImportModal/index.tsx#L25-L28)

```typescript
return fetch(url)
  .then(res => res.json())
  .then(json => {
    setContents({ contents: JSON.stringify(json, null, 2) });
    // ⚠️ 注意：未显式指定 format 参数
    onClose();
  })
```

**关键发现**：通过 URL 导入（无论是地址参数入口还是 ImportModal 入口）加载远程 JSON 时，**`setContents` 未传入 `format` 参数**，因此 format 会使用 store 的**当前值**（默认为 `FileFormat.JSON`）。

**位置**：[useFile.ts#L100-L107](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L100-L107)

```typescript
setContents: async ({ contents, hasChanges = true, skipUpdate = false, format }) => {
  try {
    set({
      ...(contents && { contents }),
      error: null,
      hasChanges,
      format: format ?? get().format,  // ← format 未指定时，使用当前值
    });
```

#### 各格式在保存时的表现

| 格式 | sessionStorage 存储内容（示例） | 字符串长度对比 | 80KB 限制影响 |
|------|------------------------------|-------------|-------------|
| **JSON** | `{"name": "test", "value": 123}` | 基准长度 | 标准限制 |
| **YAML** | `name: test\nvalue: 123` | 比 JSON 短 ~15-30%（省略引号、括号） | 可存储更多有效数据 |
| **XML** | `<root><name>test</name><value>123</value></root>` | 比 JSON 长 ~50-100%（标签冗余） | 更容易触发 80KB 限制 |
| **CSV** | `name,value\ntest,123` | 比 JSON 短 ~40-60%（无键名重复） | 可存储最多数据 |

**格式对保存的直接影响**：
1. **XML 格式最容易触发 80KB 限制**：相同数据量，XML 字符串长度最长
2. **CSV 格式最节省空间**：扁平化结构 + 无键名重复，相同数据量占用空间最小
3. **URL 导入始终保存为 JSON 格式**：因为 `fetch().json()` → `JSON.stringify()` 产生的是 JSON 字符串，且 format 使用默认值 `"json"`

---

### 12.3 format 字段在恢复流程中的决定性作用

#### 恢复时的 format 读取逻辑

**位置**：[useFile.ts#L147-L153](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L147-L153)

```typescript
const sessionContent = sessionStorage.getItem("content") as string | null;
const format = sessionStorage.getItem("format") as FileFormat | null;
if (sessionContent && !widget) contents = sessionContent;

if (format) set({ format });  // ← 先恢复 format
get().setContents({ contents, hasChanges: false });
```

**恢复顺序至关重要**：
1. **先设置 format**：`if (format) set({ format })`
2. **再调用 setContents**：此时 `get().format` 已经是恢复后的正确值
3. **contentToJson 使用恢复后的 format 解析**

#### contentToJson 的格式分支解析

**位置**：[jsonAdapter.ts#L4-L45](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/lib/utils/jsonAdapter.ts#L4-L45)

```typescript
export const contentToJson = async (value: string, format = FileFormat.JSON): Promise<object> => {
  if (!value) return {};

  if (format === FileFormat.JSON) {
    const { parse } = await import("jsonc-parser");
    // JSON 解析（支持注释和尾随逗号）
  }

  if (format === FileFormat.YAML) {
    const { load } = await import("js-yaml");
    // YAML 解析
  }

  if (format === FileFormat.XML) {
    const { XMLParser } = await import("fast-xml-parser");
    // XML 解析
  }

  if (format === FileFormat.CSV) {
    const { csv2json } = await import("json-2-csv");
    // CSV 解析
  }

  return {};
};
```

> **⚠️ 关键结论**：如果 format 字段丢失或与 content 不匹配，**contentToJson 会使用错误的解析器**，导致解析失败或数据错误。

#### format-content 不匹配的后果演示

| sessionStorage 状态 | content 实际格式 | format 值 | 解析器选择 | 结果 |
|-------------------|---------------|----------|----------|------|
| 正常保存 | JSON | `"json"` | jsonc-parser | ✅ 解析成功 |
| format 丢失 | JSON | `undefined`（默认 JSON） | jsonc-parser | ✅ 恰好成功（因默认是 JSON） |
| format 丢失 | YAML | `undefined`（默认 JSON） | jsonc-parser | ❌ 解析失败，显示错误 |
| format 丢失 | XML | `undefined`（默认 JSON） | jsonc-parser | ❌ 解析失败，显示错误 |
| format 丢失 | CSV | `undefined`（默认 JSON） | jsonc-parser | ❌ 解析失败，显示错误 |
| 格式不匹配 | YAML | `"xml"` | fast-xml-parser | ❌ 解析失败或产生错误结构 |

---

### 12.4 JSON 内容特征对保存与恢复的影响

#### 1. 内容大小限制：80KB 硬阈值

**位置**：[useFile.ts#L114](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L114)

```typescript
if (get().hasChanges && contents && contents.length < 80_000 && !isIframe() && !isFetchURL)
```

**影响分析**：
- 以**字符数**（`contents.length`）而非字节数判断，UTF-8 多字节字符可能导致实际占用超出预期
- 超过 80,000 字符时，**静默跳过保存**，不报错不提示
- 用户编辑大文件时刷新页面会丢失内容（因从未写入 sessionStorage）

| 内容规模 | JSON 字符数 | 保存结果 | 刷新后 |
|---------|------------|---------|--------|
| 小型 | < 80,000 | ✅ 保存到 sessionStorage | 内容恢复 |
| 大型 | ≥ 80,000 | ❌ 不保存，无提示 | 恢复默认示例 JSON，内容丢失 |

#### 2. JSON 嵌套深度：递归解析风险

**保存层面**：嵌套深度不影响 `sessionStorage.setItem`，因为保存的是已字符串化的内容。

**恢复层面**：嵌套深度会影响 `contentToJson()` 中的 `jsonc-parser` 和 `JSON.parse()`：
- 现代浏览器 `JSON.parse` 通常支持数千层嵌套
- 极端深度（>10,000 层）可能触发栈溢出或解析超时
- 解析失败时会在 BottomBar 显示错误提示，但 sessionStorage 中的内容不会被清除

**位置**：[useFile.ts#L121-L125](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L121-L125)

```typescript
} catch (error: any) {
  if (error?.mark?.snippet) return set({ error: error.mark.snippet });
  if (error?.message) set({ error: error.message });
  useJson.setState({ loading: false });
}
```

#### 3. 特殊字符与转义序列

sessionStorage 使用 UTF-16 字符串存储，对 JSON 中的特殊字符有以下影响：

| 特殊字符类型 | 示例 | 保存影响 | 恢复影响 |
|------------|------|---------|---------|
| **Unicode 字符** | `{"name": "测试 🎉"}` | ✅ 正常保存（UTF-16） | ✅ `JSON.parse` 正常恢复 |
| **控制字符** | `{"text": "line1\u000Aline2"}` | ✅ 转义序列被当作普通字符存储 | ✅ `JSON.parse` 解释转义序列 |
| **原始控制字符** | 字符串中包含真实的 `\n`（0x0A） | ⚠️ sessionStorage 可存储，但 `JSON.parse` 会失败 | ❌ 解析错误："Unexpected token" |
| **代理对字符** | `{"emoji": "😀"}`（U+1F600） | ✅ 以 UTF-16 代理对存储 | ✅ 正常恢复 |
| **反斜杠本身** | `{"path": "C:\\\\Users"}` | ✅ 字符串中的 `\\` 被正确存储 | ✅ 恢复为单个 `\` |
| **双引号** | `{"quote": "\"Hello\""}` | ✅ `\"` 转义正常存储 | ✅ 恢复为 `"` |

**关键风险点**：如果 JSON 字符串中包含**未转义的原始控制字符**（如真实换行符、制表符），`sessionStorage.setItem` 可正常保存，但 `JSON.parse` / `jsonc-parser` 在恢复时会抛出解析错误。

#### 4. JSONC（带注释 JSON）的特殊处理

**位置**：[jsonAdapter.ts#L7-L13](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/lib/utils/jsonAdapter.ts#L7-L13)

```typescript
if (format === FileFormat.JSON) {
  const { parse } = await import("jsonc-parser");
  const errors: ParseError[] = [];
  const result = parse(value, errors);
  if (errors.length > 0) JSON.parse(value);  // JSONC 解析失败时回退到标准 JSON.parse
  return result;
}
```

**保存与恢复行为**：
- **保存**：JSONC 内容（含 `//` 注释或 `/* */` 注释）作为原始字符串直接存入 sessionStorage
- **恢复**：优先使用 `jsonc-parser` 解析（支持注释），若解析出错则回退到标准 `JSON.parse`
- **边界情况**：如果 JSONC 注释中包含看起来像 JSON 语法的内容，`jsonc-parser` 的容错机制可能产生非预期结果

---

### 12.5 格式转换（setFormat）对 sessionStorage 的连锁影响

#### 用户切换格式的触发点

**位置**：[BottomBar.tsx#L150-L171](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/editor/BottomBar.tsx#L150-L171)

```typescript
{formats.map(format => (
  <Menu.Item
    key={format.value}
    onClick={() => setFormat(format.value)}  // 用户点击切换格式
    rightSection={currentFormat === format.value && <IoMdCheckmark />}
  >
    {format.label}
  </Menu.Item>
))}
```

#### setFormat 的完整执行链

**位置**：[useFile.ts#L86-L99](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L86-L99)

```typescript
setFormat: async format => {
  try {
    const prevFormat = get().format;

    set({ format });  // ① 先更新 store 中的 format
    // ② 将当前内容从旧格式解析为对象
    const contentJson = await contentToJson(get().contents, prevFormat);
    // ③ 将对象转换为新格式的字符串
    const jsonContent = await jsonToContent(JSON.stringify(contentJson, null, 2), format);

    get().setContents({ contents: jsonContent });  // ④ 保存新格式内容
  } catch {
    get().clear();
    console.warn("The content was unable to be converted, so it was cleared instead.");
  }
},
```

#### 格式转换对 sessionStorage 的影响

```
用户在 BottomBar 点击切换格式（JSON → YAML）
     │
     ▼
setFormat("yaml")
     │
     ├─ ① 更新 format = "yaml"
     ├─ ② contentToJson(contents, "json") → 解析旧格式为对象
     ├─ ③ jsonToContent(obj, "yaml") → 转换为 YAML 字符串
     │
     ▼
setContents({ contents: yamlString })  ← hasChanges 默认为 true
     │
     ├─ 满足条件：hasChanges=true, contents存在, 长度<80KB, 非iframe, 非isFetchURL
     │
     ▼
sessionStorage.setItem("content", yamlString)   ← ✅ 新格式内容被保存
sessionStorage.setItem("format", "yaml")        ← ✅ format 同步更新
```

**格式转换失败的后果**：
- 若转换过程中抛出异常（如 XML→CSV 结构不兼容），会执行 `get().clear()` 清空内容
- sessionStorage 中的旧内容**不会被同步清除**，下次刷新可能恢复旧内容

---

### 12.6 综合影响矩阵

| 影响维度 | JSON 格式 | YAML 格式 | XML 格式 | CSV 格式 |
|---------|----------|----------|----------|----------|
| **保存体积** | 基准 | 小 15-30% | 大 50-100% | 小 40-60% |
| **80KB 限制** | 标准 | 不易触发 | 最易触发 | 最难触发 |
| **解析复杂度** | 低（jsonc-parser） | 中（js-yaml） | 高（fast-xml-parser） | 中（json-2-csv） |
| **format 丢失后果** | ✅ 恰好默认匹配 | ❌ 解析失败 | ❌ 解析失败 | ❌ 解析失败 |
| **嵌套深度影响** | 中等 | 中等 | 高（标签递归） | 无（扁平化） |
| **特殊字符敏感度** | 高（严格转义） | 低 | 中 | 低 |
| **URL 导入入口可用** | ✅ 默认 | ❌ 需手动切换 | ❌ 需手动切换 | ❌ 需手动切换 |
| **格式可逆性** | ✅ 无损 | ✅ 无损 | ⚠️ 可能丢失属性顺序 | ⚠️ 嵌套结构扁平化 |

---

## 总结

| 问题 | 结论 | 证据 |
|------|------|------|
| **一键生成分享链接是否实现？** | ❌ 未实现 | 无相关函数、无 UI 按钮、无剪贴板复制 URL 逻辑 |
| **远程加载与本地恢复是否有明确界限？** | ⚠️ 名义上有，但存在漏洞 | 地址参数入口能正确隔离，但 URL 导入入口会将远程内容写入本地缓存 |
| **压缩过程是否包含在流程中？** | ❌ 完全不存在 | 无压缩库依赖、无 base64 编码、所有持久化使用原始字符串 |
| **两条远程加载路径是否一致？** | ❌ 存在显著差异 | 代码位置、URL 校验、缓存策略、错误处理均不同 |
| **JSON 格式是否影响 sessionStorage？** | ✅ 多维度影响 | 大小限制、format 匹配、特殊字符、格式转换均影响保存恢复 |
| **URL 导入后 format 会自动重置为 JSON 吗？** | ❌ 不会自动重置 | fetchUrl 和 ImportModal URL 导入均不设置 format，保持调用前的值 |

### 核心发现

1. **两条独立远程加载路径**：
   - **地址参数入口**：页面加载时自动触发，走 store 层 `checkEditorSession()` → `fetchUrl()`，**能正确隔离远程内容**
   - **URL 导入入口**：用户手动触发 ImportModal，组件内直接调用 `fetch()`，**会将远程内容写入 sessionStorage**

2. **缓存界限穿透漏洞**：
   - 根本原因是 `isFetchURL` 以 `window.location.href.includes("?")` 作为判断标准
   - 当用户通过 ImportModal 导入远程 URL 时，页面地址栏无 `?`，导致 `isFetchURL = false`
   - 远程数据被误写入 sessionStorage，下次刷新时被当作"本地内容"恢复

3. **URL 导入后 format 不会自动重置为 JSON**：
   - `fetchUrl` 调用 `setContents` 时只传 `contents`，不传 `format`
   - `setContents` 中 `format: format ?? get().format`，没传就用当前值
   - 若用户之前切换过格式（如 YAML），URL 导入后 content 是 JSON 但 format 仍是 YAML
   - 会导致 sessionStorage 中 content 与 format 不匹配，刷新后解析失败

4. **多路径 format 处理不一致**：
   - `setFile`（服务端文件）：显式设置 format，最严谨
   - ImportModal 文件导入：从文件名提取 format 并设置
   - ImportModal URL 导入：不设置 format，保持当前值
   - 地址参数入口（fetchUrl）：不设置 format，保持当前值

5. **代码层面的其他差异**：
   - 地址参数入口有 `isURL()` 严格正则校验，URL 导入入口无校验
   - 地址参数入口错误时清空内容，URL 导入入口仅显示错误提示
   - URL 导入入口有 GA 埋点，地址参数入口无埋点

当前 JSON Crack 的 URL 分享机制本质上是 **"远程 URL 引用"** 而非 **"内容编码分享"**，用户无法直接将本地编辑的 JSON 内容通过 URL 分享给他人，只能分享一个指向远程 JSON 数据的 URL。
