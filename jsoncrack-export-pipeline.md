# JSON Crack 导出流程（PNG / SVG / PDF）梳理文档

## 一、概述

JSON Crack 的导出功能集中在**图像导出**（PNG、JPEG、SVG）三个格式，基于第三方库 `html-to-image` 实现 DOM → 图像的转换。**代码库中不存在 PDF 导出的原生实现**，如需导出 PDF 需用户在浏览器打印界面手动"另存为 PDF"。

---

## 二、核心文件索引

| 文件 | 作用 |
|------|------|
| [DownloadModal/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx) | 导出模态框 UI + 核心导出逻辑 |
| [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx) | 底部工具栏：导出按钮 + 快捷键绑定 |
| [FileMenu.tsx](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/editor/Toolbar/FileMenu.tsx) | 顶部 File 菜单（含 Import / Export 文本文件） |
| [JSONCrackComponent.tsx](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx) | 画布渲染组件（reaflow Canvas） |
| [useModal.ts](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/store/useModal.ts) | 模态框全局状态管理（zustand） |
| [ModalController.tsx](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/modals/ModalController.tsx) | 全局模态框统一渲染入口 |

---

## 三、触发入口

### 3.1 快捷键

在 [GraphView/Toolbar/index.tsx#L130-L141](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L130-L141) 通过 `@mantine/hooks` 的 `useHotkeys` 绑定：

```text
mod + S  →  setVisible("DownloadModal", true)   (mod = Ctrl 或 ⌘)
```

### 3.2 底部工具栏按钮

在 [GraphView/Toolbar/index.tsx#L214-L225](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L214-L225) 点击图像下载图标 `LuImageDown`：

```tsx
<Tooltip label={`Export (${coreKey}+S)`} ...>
  <ActionIcon
    onClick={() => setVisible("DownloadModal", true)}
    ...
  >
    <LuImageDown size={18} />
  </ActionIcon>
</Tooltip>
```

### 3.3 ⚠️ FileMenu 的 "Export" 不是图像导出

注意：顶部工具栏的 [FileMenu.tsx#L14-L23](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/editor/Toolbar/FileMenu.tsx#L14-L23) 中的 `Export` 菜单项是**下载原始 JSON/文本文件**（`.jsoncrack.json` 等），不是图像导出。

---

## 四、模态框状态管理（Zustand）

### 4.1 状态初始化

[useModal.ts#L10-L20](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/store/useModal.ts#L10-L20)：

```ts
const initialStates: ModalState = modals.reduce((acc, modal) => {
  acc[modal] = false;
  return acc;
}, {} as ModalState);

export const useModal = create<ModalState & ModalActions>()(set => ({
  ...initialStates,
  setVisible: (name, open) => { set({ [name]: open }); },
}));
```

### 4.2 模态框渲染

[ModalController.tsx#L14-L16](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/modals/ModalController.tsx#L14-L16) 遍历所有已注册模态框：

```tsx
const ModalController = () => {
  return modals.map(modal => <Modal key={modal} modalKey={modal} />);
};
```

每个模态框从 `useModal` 读取 `opened` 状态和 `onClose` 回调。

---

## 五、DownloadModal 核心实现

### 5.1 第三方依赖

使用 `html-to-image@1.11.11`（见 [package.json#L24](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/package.json#L24)），导入其核心转换函数：

```ts
import { toBlob, toJpeg, toPng, toSvg } from "html-to-image";
```

### 5.2 格式枚举与映射

[DownloadModal/index.tsx#L18-L33](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L18-L33)：

```ts
enum Extensions {
  SVG = "svg",
  PNG = "png",
  JPEG = "jpeg",
}

const getDownloadFormat = (format: Extensions) => {
  switch (format) {
    case Extensions.SVG:  return toSvg;   // → SVG 字符串 (data:image/svg+xml,...)
    case Extensions.PNG:  return toPng;   // → PNG  base64 (data:image/png;base64,...)
    case Extensions.JPEG: return toJpeg;  // → JPEG base64
  }
};
```

### 5.3 定位要导出的画布元素

[DownloadModal/index.tsx#L65-L67](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L65-L67)：

```ts
const getExportElement = () =>
  (document.querySelector(".jsoncrack-canvas") as HTMLElement | null) ??
  (document.querySelector("svg[id*='ref']") as HTMLElement | null);
```

**优先级**：
1. `.jsoncrack-canvas` — reaflow 渲染的根容器（包含整个 SVG 画布）
2. `svg[id*='ref']` — 兜底，任何带 ref 的 SVG 元素

### 5.4 画布来源（reaflow Canvas）

`.jsoncrack-canvas` 类名在 [JSONCrackComponent.tsx#L585-L611](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/packages/jsoncrack-react/src/JSONCrackComponent.tsx#L585-L611) 绑定到 reaflow 的 `Canvas` 组件：

```tsx
<Canvas
  className="jsoncrack-canvas"
  onLayoutChange={onLayoutChange}
  node={renderNode}
  edge={renderEdge}
  nodes={visibleNodes}
  edges={visibleEdges}
  // ... 宽高、方向、布局配置
/>
```

reaflow 内部会将节点、边渲染为 SVG，最终嵌套结构大致为：
```
div.jsoncrack-canvas
  └── svg
       └── g (content group)
            ├── g#node-1, g#node-2, ... (nodes)
            └── g#edge-1, g#edge-2, ... (edges)
```

---

## 六、导出执行流程

### 6.1 图像导出（download）

**流程代码**：[DownloadModal/index.tsx#L118-L143](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L118-L143)

```
用户点击 Download 按钮
    │
    ▼
exportAsImage()
    │
    ├─► toast.loading("Downloading...")
    │
    ├─► getExportElement()  → 获取 .jsoncrack-canvas
    │     └─► 找不到？ toast.error("Canvas not found.") 并 return
    │
    ├─► 构造 imageOptions = {
    │       quality: 1,
    │       backgroundColor: "#FFFFFF",  // 用户选的背景色
    │       skipFonts: true,             // 跳过字体嵌入（加速）
    │   }
    │
    ├─► getDownloadFormat(extension)(imageElement, imageOptions)
    │     ├─ PNG  → html-to-image 的 toPng()
    │     ├─ JPEG → html-to-image 的 toJpeg()
    │     └─ SVG  → html-to-image 的 toSvg()
    │     内部实现原理（html-to-image）：
    │       1. 克隆目标 DOM 及其子树
    │       2. 内联所有 computed style
    │       3. 转换为 SVG <foreignObject> 包装
    │       4. SVG → dataURL
    │       5. （PNG/JPEG 时）用 Canvas 2D 解码 SVG 再 toDataURL
    │
    ▼
  得到 dataURI (data:image/xxx;base64,...)
    │
    ▼
downloadURI(dataURI, `filename.ext`)
    │
    └─► 触发浏览器下载（动态 a 标签 + click）
    │
    ├─► gaEvent("download_img", { label })  // GA 埋点
    └─► toast.dismiss + onClose()
```

### 6.2 下载辅助函数

[DownloadModal/index.tsx#L55-L63](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L55-L63)：

```ts
function downloadURI(uri: string, name: string) {
  const link = document.createElement("a");
  link.download = name;
  link.href = uri;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}
```

### 6.3 复制到剪贴板（Clipboard）

[DownloadModal/index.tsx#L77-L116](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L77-L116)：

与导出流程类似，但使用 `html-to-image` 的 `toBlob()` 而非 `toPng/toSvg`：

```ts
const blob = await toBlob(imageElement, imageOptions);
await navigator.clipboard?.write([
  new ClipboardItem({ [blob.type]: blob }),
]);
```

---

## 七、用户可配置参数

模态框 UI 暴露以下可配置项 [DownloadModal/index.tsx#L148-L191](file:///d:/fz/0601/solo-dogfeeding/code/186-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L148-L191)：

| 参数 | 控件 | 默认值 | 说明 |
|------|------|--------|------|
| filename | `TextInput` | `"jsoncrack.com"` | 下载文件名（不含扩展名） |
| extension | `SegmentedControl` | `PNG` | 导出格式：PNG / JPEG / SVG |
| backgroundColor | `ColorInput` + `ColorPicker` | `#FFFFFF` | 背景色（支持 rgba 和 transparent） |
| quality | （硬编码） | `1` | 图像质量，0~1（仅 JPEG 生效） |
| skipFonts | （硬编码） | `true` | 是否跳过字体嵌入 |

---

## 八、关于 PDF 导出

### 8.1 当前状态：**无原生 PDF 导出**

经过全文检索（`jspdf` / `html2pdf` / `application/pdf` / `.pdf`），代码库中**不存在 PDF 导出相关实现**，`package.json` 也没有 `jsPDF`、`html2pdf.js` 等 PDF 生成库。

### 8.2 用户侧替代方案

- **浏览器打印**：`Ctrl + P` → 目标打印机选择"另存为 PDF"
- **先导出 SVG 后转换**：用模态框导出 SVG → 用 [Inkscape](https://inkscape.org/) / Adobe Illustrator / 在线工具转 PDF

### 8.3 若要实现原生 PDF 导出（建议方案）

可在 `DownloadModal` 中扩展 `Extensions.PDF`，采用：

```
方案 A：html-to-image → PNG → jsPDF.addImage()
  优点：实现简单、渲染一致
  缺点：PDF 是位图，不可搜索文本

方案 B：SVG → jsPDF（通过 canvg 或 svg2pdf）
  优点：矢量 PDF，可缩放、文件小
  缺点：CSS 样式映射复杂
```

---

## 九、完整调用链路图

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │                          用户触发层                                  │
 │  ┌─────────────┐  ┌─────────────────┐  ┌─────────────────────────┐  │
 │  │ Ctrl/⌘ + S  │  │ 底部工具栏按钮  │  │   未来扩展: FileMenu    │  │
 │  └──────┬──────┘  └────────┬────────┘  └─────────────────────────┘  │
 │         │                  │                                         │
 └─────────┼──────────────────┼─────────────────────────────────────────┘
           │                  │
           ▼                  ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │                    Zustand 状态层 (useModal)                         │
 │         setVisible("DownloadModal", true)                            │
 └────────────────────────────┬────────────────────────────────────────┘
                              │
                              ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │                   ModalController 渲染层                             │
 │         <DownloadModal opened={true} onClose={...} />               │
 └────────────────────────────┬────────────────────────────────────────┘
                              │
                              ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │                   DownloadModal 交互层                               │
 │  ┌──────────────────────────────────────────────────────────────┐   │
 │  │  用户配置：                                                   │   │
 │  │    • 文件名 (jsoncrack.com)                                  │   │
 │  │    • 格式   [PNG | JPEG | SVG]  ← 选 PDF 的话当前无实现       │   │
 │  │    • 背景色 (#FFFFFF / transparent)                          │   │
 │  └──────────────────────────┬───────────────────────────────────┘   │
 │                             │  点击 Download / Clipboard             │
 └─────────────────────────────┼───────────────────────────────────────┘
                               │
                               ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │                    html-to-image 转换层                              │
 │  ┌──────────────────────────────────────────────────────────────┐   │
 │  │  getExportElement()                                          │   │
 │  │    → .jsoncrack-canvas (reaflow Canvas)                      │   │
 │  │         └── <svg> 节点/边图形                                 │   │
 │  └──────────────────────────┬───────────────────────────────────┘   │
 │                             │                                        │
 │  ┌──────────────────────────▼───────────────────────────────────┐   │
 │  │  getDownloadFormat(format)(element, options)                  │   │
 │  │    • PNG   → toPng()   → data:image/png;base64,...           │   │
 │  │    • JPEG  → toJpeg()  → data:image/jpeg;base64,...          │   │
 │  │    • SVG   → toSvg()   → data:image/svg+xml,...              │   │
 │  │    • BLOB  → toBlob()  → Blob (给剪贴板用)                    │   │
 │  └──────────────────────────┬───────────────────────────────────┘   │
 └─────────────────────────────┼───────────────────────────────────────┘
                               │
              ┌────────────────┴───────────────┐
              ▼                                ▼
 ┌──────────────────────────┐   ┌──────────────────────────────┐
 │   downloadURI()          │   │  navigator.clipboard.write()  │
 │   <a download> + click() │   │  ClipboardItem { Blob }       │
 │   → 触发浏览器保存对话框  │   │  → 系统剪贴板                 │
 └──────────────────────────┘   └──────────────────────────────┘
```

---

## 十、关键注意事项 & 潜在优化点

1. **`skipFonts: true` 的权衡**：加速导出但如果画布使用了特殊字体，导出图像可能字体降级。如果追求渲染一致性，可考虑关闭并配合字体预加载。

2. **`backgroundColor` 的应用**：html-to-image 会在最外层包一层带背景色的容器，所以即使用户选 `transparent`，SVG 输出也能正常透明。

3. **硬编码 quality=1**：JPEG 的质量未暴露 UI，始终最高质量。如需更小文件可加 Slider 让用户调整。

4. **DOM 选择器脆弱性**：`getExportElement()` 依赖 `.jsoncrack-canvas` 类名，如果 reaflow 内部结构或类名改变，会导致导出失败（已用 `??` 做了一层兜底）。

5. **节点超限时的导出**：当节点数超过 `maxRenderableNodes` 时，Canvas 不会渲染，`getExportElement` 会报错"Canvas not found"。这是符合预期的行为。
