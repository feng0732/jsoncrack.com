# JSON Crack URL 分享与状态压缩执行路径分析

## 概述

JSON Crack 采用 **URL 参数驱动 + 客户端存储** 的混合方案实现状态分享与恢复。与传统的 URL 状态压缩方案不同，当前版本并未实现将 JSON 内容直接压缩编码到 URL Hash 或 Query 中，而是通过 **引用远程 URL** + **sessionStorage 本地缓存** 的方式工作。

---

## 一、URL 分享入口与参数解析

### 1.1 支持的页面路由

| 页面 | 路由 | 核心文件 |
|------|------|----------|
| 编辑器主页面 | `/editor` | [editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/editor.tsx) |
| Widget 嵌入页 | `/widget` | [widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/widget.tsx) |

### 1.2 URL 参数格式

```
https://jsoncrack.com/editor?json=<URL_ENCODED_REMOTE_JSON_URL>
```

**示例**：
```
https://jsoncrack.com/editor?json=https://catfact.ninja/fact
```

### 1.3 参数解析入口

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

---

## 二、核心状态处理逻辑：`checkEditorSession`

### 2.1 完整执行路径

**位置**：[useFile.ts#L142-L154](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L142-L154)

```
URL 参数解析 → URL 有效性校验 → 分支处理
     │
     ├─ 是有效远程 URL → fetchUrl() 加载远程数据
     │
     └─ 非 URL → 检查 sessionStorage → 有缓存则恢复
                        │
                        └─ 无缓存 → 使用默认示例 JSON
```

### 2.2 URL 有效性校验

**位置**：[useFile.ts#L61-L65](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L61-L65)

```typescript
const isURL = (value: string) => {
  return /(https?:\/\/(?:www\.|(?!www))[a-zA-Z0-9][a-zA-Z0-9-]+[a-zA-Z0-9]\.[^\s]{2,}|www\.[a-zA-Z0-9][a-zA-Z0-9-]+[a-zA-Z0-9]\.[^\s]{2,}|https?:\/\/(?:www\.|(?!www))[a-zA-Z0-9]+\.[^\s]{2,}|www\.[a-zA-Z0-9]+\.[^\s]{2,})/gi.test(value);
};
```

### 2.3 远程数据加载：`fetchUrl`

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

---

## 三、状态序列化与持久化机制

### 3.1 序列化方案（无压缩）

当前版本 **未使用任何压缩算法**（如 pako、lz-string、gzip 等），也未使用 base64 编码。状态持久化直接使用原始字符串。

### 3.2 sessionStorage 本地缓存

**位置**：[useFile.ts#L114-L118](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L114-L118)

```typescript
if (get().hasChanges && contents && contents.length < 80_000 && !isIframe() && !isFetchURL) {
  sessionStorage.setItem("content", contents);
  sessionStorage.setItem("format", get().format);
  set({ hasChanges: true });
}
```

**缓存条件**：
- 内容有变更（`hasChanges = true`）
- 内容非空且长度 < 80,000 字符
- 非 iframe 嵌入模式
- URL 不包含查询参数（非远程加载模式）

### 3.3 用户配置持久化（localStorage）

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

## 四、状态恢复流程（反序列化）

### 4.1 完整恢复链

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
     │    ├─ 存在 content → 恢复内容和格式
     │    │
     │    └─ 不存在 → 使用默认示例 JSON
     │
     └─ 3. 检查 localStorage（zustand persist 自动恢复）
          └─ 恢复用户配置（主题、视图等）
```

### 4.2 Widget 页面的特殊恢复逻辑

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

### 4.3 postMessage 动态数据注入

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

## 五、JSON 内容格式转换链

### 5.1 多格式支持架构

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

### 5.2 序列化细节

**位置**：[useFile.ts#L67-L69](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts#L67-L69)

```typescript
const debouncedUpdateJson = debounce((value: unknown) => {
  useJson.getState().setJson(JSON.stringify(value, null, 2));
}, 400);
```

- 使用 400ms 防抖避免频繁重渲染
- 序列化时使用 2 空格缩进格式化输出

---

## 六、关键数据结构与 Store 设计

### 6.1 useFile Store（核心状态）

**位置**：[useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts)

| 状态 | 类型 | 说明 |
|------|------|------|
| `contents` | `string` | 原始编辑器内容（可能是 JSON/YAML/XML/CSV） |
| `format` | `FileFormat` | 当前文件格式 |
| `hasChanges` | `boolean` | 是否有未保存变更 |
| `error` | `string \| null` | 解析错误信息 |
| `fileData` | `File \| null` | 关联的后端文件数据 |

### 6.2 useJson Store（渲染状态）

**位置**：[useJson.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useJson.ts)

| 状态 | 类型 | 说明 |
|------|------|------|
| `json` | `string` | 标准化的 JSON 字符串（供渲染） |
| `loading` | `boolean` | 解析加载状态 |

### 6.3 useGraph Store（视图状态）

**位置**：[useGraph.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/features/editor/views/GraphView/stores/useGraph.ts)

| 状态 | 类型 | 说明 |
|------|------|------|
| `direction` | `LayoutDirection` | 图布局方向（RIGHT/DOWN/LEFT/UP） |
| `fullscreen` | `boolean` | 是否全屏模式 |
| `viewPort` | `ViewPort \| null` | 视口对象（缩放/平移） |
| `collapsedCount` | `number` | 已折叠节点数 |

**注意**：视图状态（缩放、平移、折叠）**不会被序列化到 URL**，仅在内存中维护。

---

## 七、当前方案的技术特点

### 7.1 优点

1. **URL 简洁**：仅传递远程 URL 引用，避免超长 URL 问题
2. **无需服务端**：纯客户端实现，无状态压缩/解压服务端依赖
3. **多格式透明**：支持 JSON/YAML/XML/CSV 多种输入格式
4. **性能可控**：80KB 大小限制防止 sessionStorage 溢出

### 7.2 局限

1. **无内容压缩**：无法直接分享 JSON 内容，必须通过远程 URL
2. **视图状态丢失**：刷新页面后缩放、平移、折叠状态会丢失
3. **依赖 CORS**：远程 URL 必须支持跨域访问
4. **会话级存储**：sessionStorage 在关闭标签后即清除

### 7.3 潜在的优化方向

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

## 八、关键代码引用速查表

| 功能模块 | 文件路径 | 核心行号 |
|----------|----------|----------|
| URL 参数解析入口 | [editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/editor.tsx) | L103-L117 |
| Widget URL 解析 | [widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/widget.tsx) | L37-L54 |
| 会话检查主逻辑 | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L142-L154 |
| 远程 URL 加载 | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L129-L141 |
| sessionStorage 缓存 | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L114-L118 |
| URL 正则校验 | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L61-L65 |
| 防抖序列化 | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useFile.ts) | L67-L69 |
| 多格式适配器 | [jsonAdapter.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/lib/utils/jsonAdapter.ts) | L4-L93 |
| 配置持久化 | [useConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/store/useConfig.ts) | L18-L31 |
| postMessage API | [widget.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/widget.tsx) | L56-L75 |
| 嵌入文档示例 | [docs.tsx](file:///d:/fz/0601/solo-dogfeeding/code/188-jsoncrack.com/apps/www/src/pages/docs.tsx) | L28-L58 |
