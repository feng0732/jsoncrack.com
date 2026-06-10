# JSONCrack 入口路径勘误与真实调用链分析

> 本文纠正了此前对文件导入、URL 导入、格式切换、手动转换按钮的几处理解偏差，逐入口对照代码给出精确调用链。

---

## 总览：所有入口的终点（下游公共链路）

无论从哪个入口进入，最终都会汇入以下公共链路：

```
                    ┌─────────────────────────────────────────┐
                    │           useFile.setContents()         │
                    │   ① contentToJson(contents, format)     │
                    │   ② debouncedUpdateJson() ← 400ms 防抖  │
                    └────────────────────┬────────────────────┘
                                         ▼
                              useJson.setJson(jsonStr)
                                         ▼
                              <JSONCrack json={jsonStr} />
                                         ▼
                         toJsonText() → parseJsonGraph() → parseGraph()
                                         ▼
                                  节点 + 边 → ELK 布局 → SVG
```

`setContents()` 是绝大多数入口的"漏斗"。只有一条路径例外：`fetchUrl()`（见下文）。

---

## 一、此前的理解偏差对照

| 之前的说法 | 实际代码 | 偏差说明 |
|-----------|---------|---------|
| "ViewMenu → setFormat() 格式切换" | `ViewMenu` 只切换 Graph/Tree 视图模式 | **完全错配**。文件格式切换在 BottomBar 右下角，ViewMenu 与文件格式无关 |
| "URL 导入走 fetchUrl()" | ImportModal 内的 URL 是原生 `fetch()` + `setContents()` | URL 导入有**两条独立路径**，ImportModal 不走 `fetchUrl` |
| "文件导入 → setFile()" | ImportModal 走 `setFormat()` + `setContents()`，FullscreenDropzone 走 `setContents({contents, format})` | 两条文件导入路径都不经过 `setFile()`，`setFile()` 仅用于云端已保存文件的加载 |
| "格式切换 → jsonToContent 重新序列化" | `setFormat()` 是 **contentToJson(旧格式) + jsonToContent(新格式)** 的双向转换，不是单向序列化 | 少说了"先用旧格式解析"这一步 |
| 未提及 "Click to Transform" 按钮 | BottomBar 在 `liveTransformEnabled=false` 时显示，调用 `setContents({})` | 此前完全遗漏了这条入口 |

---

## 二、入口 1：Monaco 编辑器手动输入

**代码位置：** [TextEditor.tsx#L82-L92](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/features/editor/TextEditor.tsx#L82-L92)

```tsx
<Editor
  onChange={contents => setContents({ contents, skipUpdate: true })}
/>
```

### 真实调用链

```
onChange(contents)
    ↓
useFile.setContents({ contents, skipUpdate: true })
    ↓
① set({ contents, error: null, hasChanges: true, format: 当前format })
    ↓
② contentToJson(contents, format)  ← 格式适配 → JS 对象
    ↓
③ skipUpdate 开关判断：
   if (!liveTransformEnabled && skipUpdate) return;   ← 关了实时转换 + 传了 skipUpdate → 不更图
   else → 继续
    ↓
④ 可选 sessionStorage 持久化（hasChanges && contents<80KB && !iframe && !URL带参数）
    ↓
⑤ debouncedUpdateJson(json)  ← 400ms 防抖
    ↓
useJson.setJson(JSON.stringify(json, null, 2))
```

### 关键边界

- **`skipUpdate=true` 不是绝对跳过**：如果 `liveTransformEnabled=true`（默认开启），即使 `skipUpdate=true` 也会继续更新图。`skipUpdate` 只有在 liveTransform 关闭时才生效。
- 每按键都跑 `contentToJson`，但只有通过了 skipUpdate 判断的才会触发图更新。

---

## 三、入口 2：URL 导入（两条独立路径，容易混淆）

### 路径 A：页面加载时的 URL Query 参数

**代码位置：**
- 入口：[editor.tsx#L115-L117](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/pages/editor.tsx#L115-L117)
- 处理：[useFile.ts#L142-L154](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/store/useFile.ts#L142-L154)

```ts
// editor.tsx
useEffect(() => {
  if (isReady) checkEditorSession(query?.json);
}, [isReady, query]);

// useFile.ts checkEditorSession()
const isURL = (value: string) => /https?:\/\/.../.test(value);
if (url && typeof url === "string" && isURL(url)) {
  return get().fetchUrl(url);   // ← 识别为 URL 才走 fetchUrl
}
```

#### fetchUrl() 的特殊双通道更新

**代码位置：** [useFile.ts#L129-L141](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/store/useFile.ts#L129-L141)

```ts
fetchUrl: async url => {
  try {
    const res = await fetch(url);
    const json = await res.json();
    const jsonStr = JSON.stringify(json, null, 2);

    get().setContents({ contents: jsonStr });          // 通道 1：走完整 setContents 链路
    return useJson.setState({ json: jsonStr, loading: false });  // 通道 2：直接写入 useJson（跳过防抖！）
  } catch { ... }
},
```

**双通道含义**：
- `setContents()` 走完整链路（格式适配 + 防抖 + sessionStorage）
- `useJson.setState()` 直达图渲染，跳过 400ms 防抖
- 两次写入最终 `useJson` 的值相同，第二次 setState 是 no-op 但比防抖先到

---

### 路径 B：ImportModal 手动输入 URL

**代码位置：** [ImportModal/index.tsx#L18-L32](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/features/modals/ImportModal/index.tsx#L18-L32)

```ts
const handleImportFile = () => {
  if (url) {
    setFile(null);
    toast.loading("Loading...", { id: "toastFetch" });

    return fetch(url)                    // ← 原生 fetch，不走 fetchUrl()
      .then(res => res.json())
      .then(json => {
        setContents({ contents: JSON.stringify(json, null, 2) });
        onClose();
      })
      .catch(() => toast.error("Failed to fetch JSON!"))
      .finally(() => toast.dismiss("toastFetch"));
  }
  // ...
};
```

#### 两条 URL 路径的差异对照

| 维度 | 路径 A（query 参数） | 路径 B（ImportModal 手动输入） |
|-----|---------------------|------------------------------|
| 触发时机 | 页面加载 | 用户在弹窗里点 Import |
| URL 正则校验 | `isURL()` 严格匹配 | 无校验，任何字符串都会去 fetch |
| fetch 封装 | `useFile.fetchUrl()` | 原生 `fetch()` |
| 更新 useJson | 双通道（setContents + 直接 setState） | 单通道（只走 setContents） |
| 格式识别 | 固定为 JSON（`res.json()`） | 固定为 JSON |
| hasChanges 标记 | 由 setContents 默认 true | 由 setContents 默认 true |
| sessionStorage | setContents 内自动处理 | setContents 内自动处理 |

---

## 四、入口 3：文件导入（两条独立路径）

### 路径 A：ImportModal 弹窗选择文件

**代码位置：** [ImportModal/index.tsx#L33-L46](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/features/modals/ImportModal/index.tsx#L33-L46)

```ts
} else if (file) {
  const lastIndex = file.name.lastIndexOf(".");
  const format = file.name.substring(lastIndex + 1);
  setFormat(format as FileFormat);              // ① 先设置格式

  file.text().then(text => {
    setContents({ contents: text });             // ② 再设置内容
    setFile(null);
    setURL("");
    onClose();
  });
}
```

#### 重要时序细节：两步调用导致两次解析

`setFormat()` 是 `async` 函数（见入口 6），内部会：
1. 用**旧格式**解析当前 `store.contents`（可能是示例数据）
2. 转换成新格式
3. 调 `setContents({ contents: 转换后的内容 })`

然后 `file.text()` 异步完成后，再用真实文件内容调第二次 `setContents()`。

**实际效果**：
- T0: `setFormat(newFormat)` → 用示例 JSON 转成新格式 → setContents(转换后的示例)
- T1: `file.text()` resolve → setContents(真实文件内容)

T0 的解析是**冗余但无害**的，因为 T1 会覆盖 contents 和 useJson。

---

### 路径 B：FullscreenDropzone 拖拽上传

**代码位置：** [FullscreenDropzone.tsx#L17-L27](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/features/editor/FullscreenDropzone.tsx#L17-L27)

```ts
onDrop={async e => {
  try {
    const fileContent = await e[0].text();
    let fileExtension = e[0].name.split(".").pop() as FileFormat | undefined;
    if (!fileExtension) fileExtension = FileFormat.JSON;

    setContents({ contents: fileContent, format: fileExtension, hasChanges: false });
  } catch (err) { ... }
}}
```

#### 与 ImportModal 路径的关键差异

| 维度 | ImportModal 选文件 | FullscreenDropzone 拖拽 |
|-----|-------------------|----------------------|
| 调用次数 | 两次（setFormat + setContents） | 一次（setContents 同时传 format） |
| format 更新方式 | setFormat() 内部先 set({format}) | setContents 内 `format: format ?? get().format` 直接覆盖 |
| hasChanges | setContents 默认 true | 显式传 false |
| 冗余解析 | 有一次用示例数据的格式转换 | 无，一步到位 |
| sessionStorage 持久化 | hasChanges=true → 会存 | hasChanges=false → 不存 |

---

## 五、入口 4：JQ / JPath 查询（处理后替换 JSON）

### JQ Modal（jq 查询）

**代码位置：** [JQModal/index.tsx#L34](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/features/modals/JQModal/index.tsx#L34) + [useJsonQuery.ts#L14-L25](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/hooks/useJsonQuery.ts#L14-L25)

```ts
const updateJson = async (query: string, cb?: () => void) => {
  const jq = await import("jq-web");
  const res = await jq.promised.json(JSON.parse(getJson()), query);  // ← 从已解析的 useJson 取数据

  setContents({ contents: JSON.stringify(res, null, 2) });
  cb?.();
};
```

### JPath Modal（JSONPath 查询）

**代码位置：** [JPathModal/index.tsx#L16-L27](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/features/modals/JPathModal/index.tsx#L16-L27)

```ts
const evaluteJsonPath = () => {
  const json = getJson();
  const result = JSONPath({ path: query, json: JSON.parse(json) });

  setContents({ contents: JSON.stringify(result, null, 2) });
  onClose();
};
```

#### 两条查询路径的共同特征

- **数据来源**：都是从 `useJson.getJson()`（已解析好的 JSON 字符串）取输入，而不是从 `useFile.contents`（原始格式文本）取
- **输出格式**：查询结果固定序列化为 JSON，格式不会变
- **下游链路**：统一走 `setContents({ contents })`，跟编辑器输入完全相同
- **hasChanges**：默认 true，会写入 sessionStorage

---

## 六、入口 5：Click to Transform（手动转换按钮）

**代码位置：** [BottomBar.tsx#L142-L147](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/features/editor/BottomBar.tsx#L142-L147)

```tsx
{!liveTransformEnabled && (
  <StyledBottomBarItem onClick={() => setContents({})} disabled={!!error}>
    <VscRunAll />
    Click to Transform
  </StyledBottomBarItem>
)}
```

### 调用链：传 `{}` 的精妙之处

`setContents({})` 接收的参数全部走默认值：
- `contents` = `undefined` → `contents && { contents }` 为假，**不更新编辑器内容**
- `hasChanges` = `true`（默认值）
- `skipUpdate` = `false`（默认值）
- `format` = `undefined` → `format ?? get().format` → 维持原格式

进入 setContents 内部：

```ts
setContents({})
    ↓
① set({ error: null, hasChanges: true, format: 原format })
    ↓
② contentToJson(get().contents, get().format)   ← 用"当前编辑器内容"重新解析
    ↓
③ skipUpdate=false，liveTransform 不管开不开都继续
    ↓
④ debouncedUpdateJson(json)   ← 400ms 防抖后更新图
```

**本质**：Live Transform 关闭时，用户手动点击"用当前编辑器内容重新跑一次解析并更新图"。

---

## 七、入口 6：格式切换（BottomBar 右下角下拉菜单）

**代码位置：**
- UI：[BottomBar.tsx#L150-L171](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/features/editor/BottomBar.tsx#L150-L171)
- 逻辑：[useFile.ts#L86-L99](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/store/useFile.ts#L86-L99)

### 真实调用链：双向转换，不是单向序列化

```ts
setFormat: async format => {
  try {
    const prevFormat = get().format;          // ① 记住旧格式

    set({ format });                           // ② 先把 store.format 改成新格式
    const contentJson = await contentToJson(get().contents, prevFormat);  // ③ 用旧格式解析为 JS 对象
    const jsonContent = await jsonToContent(JSON.stringify(contentJson, null, 2), format);  // ④ 序列化为新格式字符串

    get().setContents({ contents: jsonContent });  // ⑤ 写回转换后的编辑器内容
  } catch {
    get().clear();                            // ⑥ 转换失败就清空
  }
},
```

#### 流程示意

```
场景：编辑器里是 YAML，用户切到 JSON

当前 store:
  format = "yaml"
  contents = "name: Alice\nage: 30"

步骤：
  ① prevFormat = "yaml"
  ② store.format = "json"
  ③ contentToJson("name: Alice\nage: 30", "yaml") → { name: "Alice", age: 30 }
  ④ jsonToContent('{"name":"Alice","age":30}', "json") → '{\n  "name": "Alice",\n  "age": 30\n}'
  ⑤ setContents({ contents: '{\n  "name": "Alice",\n  "age": 30\n}' })
     → 编辑器文本更新
     → contentToJson 再次解析（这次 format="json"）
     → debouncedUpdateJson → 图更新
```

#### 关键细节

- **不是只改 state.format**：而是真的把编辑器里的文本转成了新格式
- 内部第 ⑤ 步调了 `setContents()`，所以整条下游链路都会跑
- 转换异常会**清空全部内容**（`get().clear()` → `store.contents = ""` + `useJson.clear()`）

---

## 八、入口 7：会话恢复（sessionStorage + 默认示例）

**代码位置：** [useFile.ts#L142-L154](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/store/useFile.ts#L142-L154)

```ts
checkEditorSession: (url, widget) => {
  if (url && typeof url === "string" && isURL(url)) return get().fetchUrl(url);  // URL 模式，见入口 3A

  let contents = defaultJson;                                    // ① 默认是打包的示例 JSON
  const sessionContent = sessionStorage.getItem("content");      // ② 尝试读 sessionStorage
  const format = sessionStorage.getItem("format");
  if (sessionContent && !widget) contents = sessionContent;

  if (format) set({ format });                                   // ③ 先恢复 format（只 set，不走 setFormat！）
  get().setContents({ contents, hasChanges: false });            // ④ 恢复内容，hasChanges=false 表示不是用户编辑
},
```

#### 关键边界

- `set({ format })` 而**不是** `setFormat(format)`：只改 state，不触发双向转换，因为恢复的内容本来就是用该格式保存的
- URL 参数优先于 sessionStorage，sessionStorage 优先于默认示例
- Widget 模式（iframe 嵌入）**忽略** sessionStorage，始终用默认示例

---

## 九、入口 8：云端保存文件加载（setFile）

**代码位置：** [useFile.ts#L78-L82](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/store/useFile.ts#L78-L82)

```ts
setFile: fileData => {
  set({ fileData, format: fileData.format || FileFormat.JSON });  // ① 保存元信息 + format
  get().setContents({ contents: fileData.content, hasChanges: false });  // ② 写内容
  gaEvent("set_content", { label: fileData.format });
},
```

和会话恢复几乎相同，只是 `fileData` 来自云端 API 而不是 sessionStorage。`setFile` 是 8 个入口里唯一会设置 `store.fileData` 的。

---

## 十、入口全景图

```
                                 ┌──────────────────┐
                                 │  Monaco 编辑器    │
                                 │  onChange(每按键) │
                                 └────────┬─────────┘
                                          │ setContents({contents, skipUpdate:true})
                                          ▼
┌──────────────────┐            ┌─────────────────────┐            ┌──────────────────┐
│ ImportModal URL  │──fetch────▶│                     │            │  BottomBar        │
└──────────────────┘            │                     │            │  Click to Transform│
                                 │                     │            └────────┬─────────┘
┌──────────────────┐            │                     │                     │ setContents({})
│ ImportModal 文件 │─setFormat─▶│   useFile           │◀────────────────────┘
└──────────────────┘  └setContents▶│   .setContents()  │
                                 │                     │◀────────────────────┐
┌──────────────────┐            │                     │                     │
│ FullscreenDropzone ─setContents({contents,format})  │  ┌──────────────────┴──────────┐
└──────────────────┘            │                     │  │  BottomBar 格式切换          │
                                 │                     │  │  setFormat(format)           │
┌──────────────────┐            │                     │  │  (contentToJson 旧→新格式)   │
│  JQ / JPath Modal ─setContents▶│                     │  └──────────────────┬──────────┘
└──────────────────┘            │                     │                     │
                                 │                     │                     ▼
┌──────────────────┐            │                     │            ┌─────────────────────┐
│  query.json (URL) ─fetchUrl──▶│                     │            │  setContents 内部     │
└──────────────────┘            │  + 双通道 useJson    │            │  ① contentToJson()   │
                                 └──────────┬──────────┘            │  ② sessionStorage   │
                                            │                       │  ③ debouncedUpdateJson│
┌──────────────────┐                      │                       └──────────┬──────────┘
│  会话恢复         ─setContents({...})────┘                                  │
│  (sessionStorage) │                                                        ▼ 400ms 后
└──────────────────┘                                               ┌─────────────────────┐
                                                                   │  useJson.setJson()   │
┌──────────────────┐                                               └──────────┬──────────┘
│  云端文件 setFile ─setContents({...})───────────────────────────────────────┘
└──────────────────┘                                                              │
                                                                                 ▼
                                                                   ┌─────────────────────────────┐
                                                                   │  <JSONCrack json={jsonStr}> │
                                                                   │  toJsonText → parseGraph    │
                                                                   │  ELK 布局 → SVG 渲染        │
                                                                   └─────────────────────────────┘
```

---

## 十一、各入口对 `setContents()` 参数的实际传值对照

| 入口 | contents | hasChanges | skipUpdate | format |
|-----|----------|-----------|-----------|--------|
| 编辑器输入 | 用户输入的文本 | `true`（默认） | `true`（显式传） | 不传（用当前） |
| ImportModal URL | fetch 结果 JSON.stringify | `true`（默认） | `false`（默认） | 不传（用当前） |
| ImportModal 文件 | 先 setFormat → 再 file.text() 内容 | `true`（默认） | `false`（默认） | setFormat 单独设置 |
| FullscreenDropzone | 文件文本 | `false`（显式传） | `false`（默认） | 文件扩展名（显式传） |
| JQ/JPath Modal | 查询结果 JSON.stringify | `true`（默认） | `false`（默认） | 不传（用当前，固定 JSON） |
| Click to Transform | `undefined`（不传） | `true`（默认） | `false`（默认） | 不传（用当前） |
| 会话恢复 | sessionStorage / 默认示例 | `false`（显式传） | `false`（默认） | set({format}) 单独设置 |
| 云端文件 setFile | fileData.content | `false`（显式传） | `false`（默认） | set({format}) 单独设置 |
