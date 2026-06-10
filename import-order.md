# JSONCrack ImportModal 文件导入时序深度分析

> 本文精确到事件循环级别（宏任务 / 微任务 / 同步代码），分析 ImportModal 文件导入过程中 `setFormat()` 与 `file.text()` 的并发竞态、编辑区内容的最终来源、以及图谱解析文本最终取自哪次写入。

---

## 一、前置知识：事件循环与异步类型

所有时序都遵循 JavaScript 事件循环规则：

```
同步代码（当前栈）
    → 微任务队列（Promise.then / await / queueMicrotask）
        → 宏任务队列（setTimeout / fetch / I/O / rAF）
```

本文涉及的异步操作类型：

| 代码片段 | 异步类型 | 说明 |
|---------|---------|------|
| `await contentToJson()` | 微任务（首次含宏任务动态 import） | 内部 `await import("js-yaml")` 是宏任务+微任务 |
| `await jsonToContent()` | 微任务（首次含宏任务动态 import） | 同上 |
| `file.text()` | 宏任务（Blob I/O） | 浏览器读取文件，交给 I/O 线程 |
| `debouncedUpdateJson` (400ms) | 宏任务（setTimeout） | lodash debounce 底层是 setTimeout |
| React `setState` / `useEffect` | 微任务（调度批处理） | React 18 自动批处理，通常在微任务阶段 flush |

---

## 二、handleImportFile 同步代码的精确执行顺序

**代码位置：** [ImportModal/index.tsx#L18-L46](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/features/modals/ImportModal/index.tsx#L18-L46)

```ts
const handleImportFile = () => {
  if (url) { ... }
  else if (file) {
    const lastIndex = file.name.lastIndexOf(".");
    const format = file.name.substring(lastIndex + 1);
    setFormat(format as FileFormat);              // ← ① 同步调用 async 函数

    file.text().then(text => {                    // ← ② 启动宏任务（文件 I/O）
      setContents({ contents: text });             // ← 将来某个宏任务阶段执行
      setFile(null);
      setURL("");
      onClose();
    });
  }
};
```

### ① setFormat() 同步部分：立刻执行到第一个 await

**代码位置：** [useFile.ts#L86-L99](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/store/useFile.ts#L86-L99)

```ts
setFormat: async format => {
  try {
    // ── 这些都是同步执行！──────────────────────────────────
    const prevFormat = get().format;               // (A) 取旧 format = "json"
    set({ format });                                 // (B) ✅ Zustand format 立刻改为 "yaml"
    // ──────────────────────────────────────────────────────
    const contentJson = await contentToJson(get().contents, prevFormat);  // ← 第一个 await：挂起点 1
    const jsonContent = await jsonToContent(JSON.stringify(contentJson, null, 2), format);  // 挂起点 2
    get().setContents({ contents: jsonContent });
  } catch { get().clear(); }
},
```

**关键事实：setFormat() 调用后立刻返回 Promise，但 (A)(B) 两行**同步完成**。

此时 Zustand store 的快照：

| 属性 | T0 同步代码完成时的值 |
|-----|----------------------|
| `format` | **已改为新格式**（`set({format})` 同步执行） |
| `contents` | 仍是示例 JSON 字符串（尚未修改） |
| `fileData` | 不变 |

### ② file.text() 同步部分：立刻返回 Promise，I/O 在后台线程

`file.text()` 同步返回一个 Pending 状态的 Promise。文件读取操作交给浏览器 I/O 线程，完成后将 `.then()` 回调排入宏任务队列。

---

## 三、事件循环展开：精确到每一个 tick

### 场景假设

- 初始状态：编辑器是示例 JSON（`format="json"`, `contents="{...示例...}"`）
- 用户导入一个 50KB 的 YAML 文件 `data.yaml`
- 用户还没有操作过 Live Transform，所以 `liveTransformEnabled=true`（默认）

### 时序总览图

```
T0: 同步阶段 (调用栈)
  ├─ handleImportFile 执行
  │    ├─ setFormat("yaml") 同步部分
  │    │    ├─ prevFormat = "json"
  │    │    └─ set({ format: "yaml" })   ✅ format 立刻变
  │    │    └─ await contentToJson(示例内容, "json") → 挂起
  │    └─ file.text() 启动 I/O，挂起
  └─ 调用栈清空

T1: 微任务阶段 (contentToJson resolve)
  ├─ contentToJson 返回 JS 对象（解析示例 JSON）
  ├─ JSON.stringify(contentJson, null, 2) → 美化后的示例 JSON 字符串
  ├─ await jsonToContent(美化后的JSON, "yaml") → 再次挂起

T2: 微任务阶段 (jsonToContent resolve)
  ├─ jsonToContent 返回 YAML 字符串（示例内容转 YAML）
  ├─ setContents({ contents: 示例转成的YAML })
  │    ├─ set({ contents: 示例YAML, ... })        ✅ 第 1 次写 contents
  │    ├─ contentToJson(示例YAML, "yaml")
  │    ├─ debouncedUpdateJson(JS对象)  ← 防抖 400ms 开始计时 T=0ms
  │    └─ sessionStorage 持久化

    ─── 此时 Zustand: format=yaml, contents=示例YAML ───

T3: 宏任务阶段 (file.text() I/O 完成, 约 T+Xms)
  ├─ file.text() resolve，回调执行:
  │    ├─ setContents({ contents: 真实文件内容 })  ✅ 第 2 次写 contents
  │    │    ├─ set({ contents: 真实YAML, ... })
  │    │    ├─ contentToJson(真实YAML, "yaml")
  │    │    ├─ debouncedUpdateJson(...)  ← 防抖重新计时! T=Xms
  │    │    └─ sessionStorage 持久化
  │    ├─ setFile(null), setURL(""), onClose()

    ─── 此时 Zustand: format=yaml, contents=真实YAML ───

T4: 宏任务阶段 (防抖 400ms 到, T3+400ms)
  ├─ debouncedUpdateJson 触发
  └─ useJson.setJson(真实文件的 JSON 序列化)
         └─ <JSONCrack> 订阅到变化
                └─ jsonText useMemo → toJsonText(string) → 直接返回
                       └─ useEffect([jsonText]) → parseJsonGraph(真实JSON)
```

---

## 四、两种竞态结果：谁先 resolve 决定内容覆盖顺序

T1-T2 与 T3 的先后顺序**不确定**，取决于：
- 动态 import 是否首次加载（jsonc-parser / js-yaml 等包）
- 示例内容大小和转换复杂度
- 文件 I/O 速度（磁盘 vs SSD，文件大小）

### 情况 A：setFormat 先完成（T2 < T3）— 这是最常见情况

```
T0→T1→T2 (setFormat 跑完，写了示例转 YAML)
  然后 T3 (file.text() 完成，真实文件覆盖示例)
```

| 时间点 | Zustand.contents | 防抖计时器状态 | React Editor 显示 |
|-------|-----------------|--------------|-----------------|
| T0 | `{...示例JSON...}` | 无 | 示例 JSON |
| T2 | `示例转YAML的字符串` | 400ms 计时启动 (T=0) | 短暂闪一下：YAML 格式的示例内容 |
| T3 | `真实文件的YAML字符串` | **防抖重置！** 重新计时 T=0 | 立即变成：真实 YAML 内容 |
| T3+400ms | 真实YAML | 触发 | 真实图出现 |

**重要细节**：第 1 次防抖启动在 T2，400ms 到应该是 T2+400。但 T3 在 T2+X 时重新调 `debouncedUpdateJson` → **lodash debounce 会重置计时**。所以最终第 1 次（也是唯一一次）解析发生在 T3+400ms，内容是真实文件。

**结论**：即使编辑区短暂闪了示例转 YAML，图谱解析文本永远来自真实文件——因为防抖被重置了。

---

### 情况 B：file.text() 先完成（T3 < T1）— 大文件 + 首次动态 import 时可能发生

```
T0→T3 (file.text() 先完成，写入真实文件)
  然后 T1→T2 (setFormat 完成，示例转YAML覆盖真实文件!)
```

| 时间点 | Zustand.contents | 防抖计时器状态 | React Editor 显示 |
|-------|-----------------|--------------|-----------------|
| T0 | `{...示例JSON...}` | 无 | 示例 JSON |
| T3 | `真实文件的YAML字符串` | 400ms 计时启动 (T=0) | 真实 YAML 内容 |
| T2 | `示例转YAML的字符串` ❌ | **防抖重置！** 重新计时 T=0 | ❌ 被覆盖：又变回 YAML 格式的示例内容 |
| T2+400ms | 示例YAML | 触发 | ❌ 渲染的是示例内容的图！ |

**这是一个真实的 bug！**  
当文件较大、I/O 较慢，但 js-yaml 的动态 import + 格式转换也慢，且 js-yaml 的微任务链刚好在 file.text() 的宏任务 resolve 之后才跑完，就会发生：

> **用户导入了 data.yaml，但编辑器和图谱最后显示的是示例内容转 YAML**

触发条件：
1. 首次使用 YAML/XML/CSV 格式（需要动态 import，T1-T2 时间变长）
2. 文件很小（file.text() 很快完成）
3. 或者反过来：内容转换的 CPU 时间 > 文件 I/O 时间

---

## 五、为什么会产生竞态：两个根本原因

### 原因 1：ImportModal 没有 await setFormat()

```ts
setFormat(format as FileFormat);      // ❌ 没 await！
file.text().then(text => { ... });
```

如果改成：
```ts
await setFormat(format as FileFormat);  // ✅ 先完成格式切换
const text = await file.text();         // ✅ 再读文件内容
setContents({ contents: text });        // ✅ 顺序确定
```

就不会有竞态。但当前代码两者并发。

### 原因 2：setFormat() 用「setFormat 被调用时刻的 contents」做转换

```ts
const prevFormat = get().format;    // (A) T0 时刻的旧 format
set({ format });                     // 改 format
const contentJson = await contentToJson(get().contents, prevFormat);  // ❌ T0 时刻的旧 contents
const jsonContent = await jsonToContent(..., format);
get().setContents({ contents: jsonContent });   // ❌ 写回转换后的旧内容
```

setFormat() 的设计假设是「用户只切格式，内容本身不变」——此时应该用当前 contents 做格式转换。  
但 ImportModal 的场景是「用户导入了新格式的新内容」——此时的 contents 应该是用户要导入的文件内容，而不是当前编辑器的旧内容。

**这是语义错位**：setFormat 被用于了它设计之外的场景。

---

## 六、防抖层的额外保护：为什么 bug 不一定每次都表现出来

即使发生了情况 B（示例覆盖了真实文件），图谱的最终显示也分两种情况：

### 子情况 B1：T3 和 T2 间隔 > 400ms（极罕见）

```
T3: setContents(真实) → 防抖 A 启动 (0ms)
        ... 400ms 过去了，防抖 A 到点！
T3+400ms: 防抖 A 触发 → useJson.setJson(真实JSON)  ✅ 真实写入了
T3+500ms: T2 到达 → setContents(示例) → 防抖 B 启动 (500ms)
T3+900ms: 防抖 B 触发 → useJson.setJson(示例JSON)  ❌ 然后示例覆盖真实

最终图谱：示例内容 ❌
编辑区内容：示例内容 ❌
```

### 子情况 B2：T3 和 T2 间隔 < 400ms（常见）

```
T3: setContents(真实) → 防抖 A 启动 (0ms)
T3+50ms:  T2 到达 → setContents(示例) → 防抖 A 被重置！重新从 0ms 计时
T3+450ms: 防抖 A 触发 → useJson.setJson(示例JSON) ❌

最终图谱：示例内容 ❌
编辑区内容：示例内容 ❌
```

**无论哪种子情况，情况 B 的结果都一样糟糕**——因为编辑区已经被 T2 的 setContents 改坏了，防抖只是把错误内容传递给图谱。

---

## 七、图谱解析文本的最终来源：完整链路追踪

从导入到图谱解析，文本在每一层是这样流动的：

### 正常情况（A：setFormat 先完成）的数据流

```
T2: setFormat 内部 setContents(示例转YAML)
    ↓
    contents = 示例转YAML
    debouncedUpdateJson 启动 → 400ms 计时
    ↓
T3: file.text() → setContents(真实YAML)
    ↓
    contents = 真实YAML   ← ✅ 编辑区最终内容
    debouncedUpdateJson 被重置 → 重新 400ms 计时
    ↓
T3+400ms: debouncedUpdateJson 触发
    ↓
    useJson.setJson(JSON.stringify(真实YAML解析结果, null, 2))
    ↓
    useJson.json = 真实文件的JSON字符串
    ↓
    <JSONCrack json={useJson.json} />
        ↓
        jsonText = useMemo(() => toJsonText(jsonProp), [jsonProp])
            ↓
            typeof jsonProp === "string" → 直接 return  ← ✅ 到这一步已是字符串，跳过WeakMap
        ↓
        useEffect([jsonText, maxRenderableNodes])
            ↓
            parseJsonGraph(jsonText, maxNodes) ← ✅ 解析真实文本
                ↓
                parseGraph(真实文本) → 节点 + 边 → ELK → SVG
```

### 图谱解析文本取自哪次写入：确定性结论

| 竞态情况 | Zustand.contents 最终值 | useJson.json 最终值 | 图谱解析文本 |
|---------|------------------------|---------------------|------------|
| A（T2 < T3）常见 | 真实文件 | 真实文件 → JSON | ✅ 真实文件 |
| B（T3 < T1）bug | ❌ 示例转 YAML | ❌ 示例 → JSON | ❌ 示例内容 |

**图谱解析文本 = useJson.json 的字符串值 = debouncedUpdateJson 最后一次被调用时传入的参数**

而 debouncedUpdateJson 最后一次被调用的参数 = `contentToJson(Zustand.contents, Zustand.format)` 的结果 = **最后一次 setContents 写的 contents**。

**所以「图谱解析文本的最终来源」=「最后一次 setContents 写入的 contents」**。

---

## 八、setContents 内部的 format 取值细节

**代码位置：** [useFile.ts#L100-L126](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/store/useFile.ts#L100-L126)

```ts
setContents: async ({ contents, hasChanges = true, skipUpdate = false, format }) => {
  set({
    ...(contents && { contents }),
    error: null,
    hasChanges,
    format: format ?? get().format,   // ← 关键：format 参数 undefined 时取 get().format
  });

  const json = await contentToJson(get().contents, get().format);
  // ...
  debouncedUpdateJson(json);
},
```

### file.text() 回调里的 setContents 调用分析

```ts
setContents({ contents: text });
```

传的参数里没有 `format`，所以 `format ?? get().format` 走右分支，取 `get().format`。

但 `get().format` 是什么？  
→ T0 同步阶段 setFormat() 的同步代码已经 `set({ format })` 了。  
→ **所以 format 一定是新格式。**

### setFormat 内部的 setContents 调用分析

```ts
get().setContents({ contents: jsonContent });
```

同样不传 format，所以 format 取 get().format = 新格式。  
→ contentToJson(jsonContent, 新格式) 正确。

**无论竞态如何，format 永远是新格式**——因为 setFormat 的 `set({format})` 在 T0 同步阶段就完成了，不会被任何异步操作覆盖。

所以不会出现「用旧 format 解析新内容」的错误，只会出现「contents 是旧的示例内容」的错误。

---

## 九、FullscreenDropzone：没有竞态的对比参考

**代码位置：** [FullscreenDropzone.tsx#L17-L27](file:///d:/fz/0601/solo-dogfeeding/code/181-jsoncrack.com/apps/www/src/features/editor/FullscreenDropzone.tsx#L17-L27)

```ts
onDrop={async e => {
  const fileContent = await e[0].text();
  let fileExtension = e[0].name.split(".").pop() as FileFormat | undefined;
  if (!fileExtension) fileExtension = FileFormat.JSON;

  setContents({ contents: fileContent, format: fileExtension, hasChanges: false });
}}
```

**为什么没有竞态**：
1. `await file.text()` 先完成（同步到第一个 await）
2. 再算 fileExtension
3. **一次 setContents() 同时传 contents + format**

一步到位，没有并发 Promise，没有 format 与 contents 分开设置的语义错位。

这是 ImportModal 文件导入路径应该参考的正确写法。

---

## 十、时序纠正总结

| 之前的说法 | 实际代码行为 | 纠正 |
|-----------|-------------|------|
| "T0: setFormat → 用示例转新格式；T1: file.text() resolve → setContents 真实内容" | 顺序是不确定的！取决于哪个 Promise 先 resolve | T0 同步阶段两者**同时启动**，T1/T2/T3 的先后取决于 I/O 速度 + 动态 import 时间 |
| "T0 的解析是冗余但无害的" | 当 T3 < T1 时，T2 的 setContents 会**覆盖真实文件内容**，造成 bug | 不是无害，是存在真实的竞态条件 bug |
| "图谱解析文本来自真实文件" → 永远正确 | 当发生情况 B 时，解析的是示例内容转 YAML | **最后一次 setContents 写的 contents 决定了图谱文本**，不是文件导入语义 |
| "format 可能取值错误" → format 与 contents 不一致 | setFormat() 同步阶段先 set({format})，所以所有 setContents 内 format 都是新格式 | format 永远正确，只有 contents 可能错乱 |
