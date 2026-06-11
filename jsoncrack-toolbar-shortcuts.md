# JSON Crack 工具栏与快捷键体系代码分析

## 体系架构总览

JSON Crack 的工具栏与快捷键体系采用**分层架构**，由三个核心部分协作：

1. **动作注册层**：基于 Zustand store 管理的状态和动作
2. **快捷键触发层**：基于 Mantine Hooks 的快捷键绑定
3. **界面反馈层**：基于 Mantine UI 组件和 react-hot-toast 的用户反馈

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  工具栏按钮点击 │────▶│   动作执行      │────▶│   界面反馈      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                 ▲
                                 │
┌─────────────────┐              │
│  快捷键触发     │──────────────┘
└─────────────────┘
```

---

## 1. 动作注册机制

### 1.1 核心 Store 架构

项目使用 Zustand 作为状态管理库，将动作分散在多个专门的 store 中：

| Store | 位置 | 职责 |
|-------|------|------|
| `useConfig` | [useConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/store/useConfig.ts) | 主题切换、实时转换、手势、标尺开关 |
| `useModal` | [useModal.ts](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/store/useModal.ts) | 所有模态框的显示控制 |
| `useGraph` | [useGraph.ts](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/stores/useGraph.ts) | 图形操作（缩放、居中、展开/折叠） |
| `useFile` | [useFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/store/useFile.ts) | 文件操作（导入、导出、格式转换） |
| `useJson` | [useJson.ts](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/store/useJson.ts) | JSON 数据管理 |

### 1.2 动作注册模式

每个 store 都遵循相同的三段式模式：

1. **定义初始状态**：`initialStates` 对象
2. **定义动作接口**：`XxxActions` 接口
3. **创建 store**：使用 `create` 函数合并状态和动作

**代码示例**（[useConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/store/useConfig.ts#L18-L31)）：

```typescript
const useConfig = create(
  persist<typeof initialStates & ConfigActions>(
    set => ({
      ...initialStates,
      toggleRulers: rulersEnabled => set({ rulersEnabled }),
      toggleGestures: gesturesEnabled => set({ gesturesEnabled }),
      toggleLiveTransform: liveTransformEnabled => set({ liveTransformEnabled }),
      toggleDarkMode: darkmodeEnabled => set({ darkmodeEnabled }),
    }),
    { name: "config" }  // persist 中间件自动持久化到 localStorage
  )
);
```

### 1.3 模态框动作注册的特殊机制

模态框系统采用**动态注册**模式：

1. [modalTypes.ts](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/modals/modalTypes.ts) 从 `index.ts` 导出中动态提取所有模态框名称：
   ```typescript
   export const modals = Object.freeze(Object.keys(ModalComponents));
   ```

2. [useModal.ts](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/store/useModal.ts#L10-L13) 使用 reduce 动态初始化所有模态框状态：
   ```typescript
   const initialStates: ModalState = modals.reduce((acc, modal) => {
     acc[modal] = false;
     return acc;
   }, {} as ModalState);
   ```

### 1.4 工具栏动作绑定点

工具栏组件通过 `onClick` 事件绑定到 store 动作，主要有三个工具栏：

| 工具栏 | 位置 | 主要动作 |
|--------|------|----------|
| 顶部工具栏 | [Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/index.tsx) | 文件操作、视图切换、工具菜单、主题切换、全屏 |
| 图形浮动工具栏 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx) | 缩放、居中、导出、搜索、旋转、展开/折叠 |
| 底部状态栏 | [BottomBar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/BottomBar.tsx) | 格式切换、实时转换开关、编辑器全屏 |

---

## 2. 快捷键触发机制

### 2.1 技术栈

- **`@mantine/hooks`** 提供两个核心 hooks：
  - `useHotkeys`：注册全局快捷键
  - `getHotkeyHandler`：创建组件级快捷键处理器
- **跨平台兼容**：`mod` 修饰符自动适配
  - macOS：`⌘` (Command)
  - Windows/Linux：`Ctrl`

### 2.2 全局快捷键注册

在 [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L130-L141) 中注册所有全局快捷键：

```typescript
useHotkeys(
  [
    ["mod+[plus]", zoomIn, { usePhysicalKeys: true }],      // 放大
    ["mod+[minus]", zoomOut, { usePhysicalKeys: true }],    // 缩小
    ["shift+Digit1", focusFirstNode, { usePhysicalKeys: true }],  // 聚焦第一个节点
    ["shift+Digit2", centerView, { usePhysicalKeys: true }],       // 居中视图
    ["mod+f", handleSearchToggle],                          // 搜索
    ["mod+s", () => setVisible("DownloadModal", true)],     // 导出
    ["mod+shift+d", toggleDirection],                       // 旋转布局
  ],
  []  // 空依赖数组，组件挂载时注册一次
);
```

**关键点**：
- `usePhysicalKeys: true`：使用物理键码，不受键盘布局影响
- 快捷键与工具栏按钮**共享相同的动作函数**，但按钮点击时会额外调用 `gaEvent` 进行埋点统计

### 2.3 快捷键与按钮共用动作的真实链路

**部分共用模式**：快捷键直接调用 store 动作，按钮调用时额外增加 GA 埋点。

| 功能 | 快捷键调用 | 按钮调用 | 共用程度 |
|------|-----------|---------|---------|
| 放大 | `zoomIn` | `() => { zoomIn(); gaEvent("zoom_in"); }` | 动作函数共用，按钮多埋点 |
| 缩小 | `zoomOut` | `() => { zoomOut(); gaEvent("zoom_out"); }` | 动作函数共用，按钮多埋点 |
| 聚焦首节点 | `focusFirstNode` | `() => { focusFirstNode(); gaEvent("focus_first_node"); }` | 动作函数共用，按钮多埋点 |
| 居中视图 | `centerView` | `() => { centerView(); gaEvent("center_view"); }` | 动作函数共用，按钮多埋点 |
| 导出 | `() => setVisible("DownloadModal", true)` | `() => setVisible("DownloadModal", true)` | 完全共用 |
| 搜索 | `handleSearchToggle` | `handleSearchToggle` | 完全共用 |
| 旋转布局 | `toggleDirection` | `toggleDirection` | 完全共用 |

**代码对比**（[GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx)）：

```typescript
// 快捷键注册（第132行）- 纯动作调用
["mod+[plus]", zoomIn, { usePhysicalKeys: true }],

// 按钮点击（第198-210行）- 动作调用 + GA埋点
<ActionIcon
  onClick={() => {
    zoomIn();           // 与快捷键共用的动作
    gaEvent("zoom_in"); // 按钮独有的埋点
  }}
>
  <LuPlus size={16} />
</ActionIcon>
```

### 2.4 局部快捷键处理

在 [SearchInput.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/SearchInput.tsx#L68-L72) 中使用 `getHotkeyHandler` 处理组件级快捷键：

```typescript
onKeyDown={getHotkeyHandler([
  ["Enter", next],           // 下一个匹配项
  ["shift+Enter", prev],     // 上一个匹配项
  ["Escape", handleClose],   // 关闭搜索
])}
```

这种方式确保快捷键仅在组件获得焦点时生效，避免与全局快捷键冲突。

### 2.5 快捷键检测与提示

在 [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L115-L119) 中检测用户操作系统：

```typescript
React.useEffect(() => {
  if (typeof window !== "undefined") {
    setCoreKey(navigator.userAgent.indexOf("Mac OS X") !== -1 ? "⌘" : "CTRL");
  }
}, []);
```

然后通过 Mantine 的 `Tooltip` 组件在工具栏按钮上显示快捷键提示：

```typescript
<Tooltip label={`Search (${coreKey}+F)`} position="top" withArrow openDelay={750}>
  <ActionIcon ...>
    <LuSearch size={18} />
  </ActionIcon>
</Tooltip>
```

---

## 3. 界面反馈机制

### 3.1 模态框反馈

模态框系统采用**集中式管理**，由三个文件协作：

```
[modalTypes.ts] 定义模态框类型
       ↓
[useModal.ts] 管理显示状态（setVisible）
       ↓
[ModalController.tsx] 统一渲染所有模态框
```

**触发流程**：
1. 用户操作 → 调用 `setVisible("ModalName", true)`
2. Zustand store 更新状态
3. [ModalController.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/modals/ModalController.tsx) 遍历所有模态框，根据状态渲染

**代码示例**（[ModalController.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/modals/ModalController.tsx#L14-L16)）：

```typescript
const ModalController = () => {
  return modals.map(modal => <Modal key={modal} modalKey={modal} />);
};
```

### 3.2 Toast 通知反馈

使用 `react-hot-toast` 提供即时操作反馈，支持三种状态。

#### 3.2.1 导出图片的真实反馈链路

在 [DownloadModal/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx) 中有两个独立的导出函数：

**1. 复制到剪贴板（clipboardImage）**：

```typescript
const clipboardImage = async () => {
  try {
    toast.loading("Copying to clipboard...", { id: "toastClipboard" });  // 加载状态
    // ... 执行导出
    toast.success("Copied to clipboard");  // ✅ 成功提示（真实代码）
    gaEvent("clipboard_img");
  } catch (error) {
    if (error instanceof Error && error.name === "NotAllowedError") {
      toast.error("Clipboard write permission denied...");  // ❌ 权限错误
    } else {
      toast.error("Failed to copy to clipboard");  // ❌ 通用错误
    }
  } finally {
    toast.dismiss("toastClipboard");  // 清除 loading toast
    onClose();
  }
};
```

**2. 下载图片（exportAsImage）**：

```typescript
const exportAsImage = async () => {
  try {
    toast.loading("Downloading...", { id: "toastDownload" });  // 加载状态
    // ... 执行导出
    // ⚠️ 真实代码：下载成功没有 toast.success，直接触发浏览器下载
    downloadURI(dataURI, `${fileDetails.filename}.${extension}`);
    gaEvent("download_img", { label: extension });
  } catch {
    toast.error("Failed to download image!");  // ❌ 失败提示（真实代码）
  } finally {
    toast.dismiss("toastDownload");  // 清除 loading toast
    onClose();
  }
};
```

**重要修正**：下载图片成功时**没有** Toast 成功提示，只有浏览器原生的下载通知。

### 3.3 视觉状态反馈

#### 3.3.1 按钮高亮

搜索按钮激活时使用 `variant="light"` 提供视觉反馈（[GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L232)）：

```typescript
variant={searchOpen ? "light" : "subtle"}
```

#### 3.3.2 图标切换

根据状态动态切换图标，使用户能直观感知当前状态：

- **主题切换**：[ThemeToggle.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/ThemeToggle.tsx#L14)
  ```typescript
  {!darkmodeEnabled ? <FaMoon size="18" /> : <FaSun size="18" />}
  ```

- **展开/折叠**：[GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L275)
  ```typescript
  {collapsedCount > 0 ? <LuCopyPlus size={18} /> : <LuCopyMinus size={18} />}
  ```

- **实时转换**：[BottomBar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/BottomBar.tsx#L139)
  ```typescript
  {liveTransformEnabled ? <VscSync /> : <VscSyncIgnored />}
  ```

#### 3.3.3 状态指示器

[BottomBar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/BottomBar.tsx#L112-L131) 显示 JSON 验证状态：

```typescript
{error ? (
  <Flex align="center" gap={2}>
    <VscError color="red" />
    <Text c="red" fw={500} fz="xs">Invalid</Text>
  </Flex>
) : (
  <Flex align="center" gap={2}>
    <VscCheck />
    <Text size="xs">Valid</Text>
  </Flex>
)}
```

### 3.4 搜索反馈的真实链路

搜索反馈**没有 Toast 通知**，完全通过 DOM 操作和视觉变化实现：

1. **计数反馈**（[SearchInput.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/SearchInput.tsx#L77-L80)）：
   ```typescript
   <Counter $none={noResults}>
     {noResults ? "No matches" : `${selectedNode + 1} / ${nodeCount}`}
   </Counter>
   ```

2. **DOM 高亮**（[search.ts](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/lib/utils/search.ts#L12-L17)）：
   ```typescript
   export const highlightMatchedNodes = (nodes: NodeListOf<Element>, selectedNode: number) => {
     for (let i = 0; i < nodes.length; i++) {
       nodes[i].classList.add("searched");     // 所有匹配项添加 searched 类
     }
     nodes[selectedNode].classList.add("highlight");  // 当前选中项添加 highlight 类
   };
   ```

3. **自动滚动**（[useFocusNode.ts](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/hooks/useFocusNode.ts#L47-L49)）：
   ```typescript
   viewPort?.camera.centerFitElementIntoView(matchedNode, {
     elementExtraMarginForZoom: 200,
   });
   ```

4. **无结果反馈**：当 `hasValue && !hasMatches` 时，`Counter` 组件显示红色 "No matches"

---

## 4. 偏好开关影响链路（画布网格、手势、主题同步）

### 4.1 偏好开关的真实协作链路

偏好开关在 [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L281-L327) 的 Preferences 菜单中：

```typescript
<Menu.Dropdown>
  <Menu.Item onClick={() => toggleDarkMode(!darkmodeEnabled)}>
    {darkmodeEnabled ? "Light Mode" : "Dark Mode"}
  </Menu.Item>
  <Menu.Item onClick={() => toggleGestures(!gesturesEnabled)}>
    Zoom on Scroll
  </Menu.Item>
  <Menu.Item onClick={() => toggleRulers(!rulersEnabled)}>
    Rulers
  </Menu.Item>
</Menu.Dropdown>
```

### 4.2 画布网格和手势的影响链路

**核心机制**：通过 React `key` 属性变化触发组件完全重建。

在 [GraphView/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/index.tsx#L97-L112) 中：

```typescript
<JSONCrack
  ref={jsonCrackRef}
  // ⚠️ 关键：当 direction、gesturesEnabled、rulersEnabled 任一变化时，
  // key 变化导致整个 JSONCrack 组件卸载并重新挂载
  key={[direction, gesturesEnabled, rulersEnabled].join("-")}
  json={json}
  theme={darkmodeEnabled ? "dark" : "light"}
  layoutDirection={direction}
  showControls={false}
  showGrid={rulersEnabled}      // 控制网格显示
  trackpadZoom={gesturesEnabled}  // 控制手势缩放
  maxRenderableNodes={maxVisibleNodes}
  centerOnLayout
  onViewportCreate={setViewPort}
  onNodeClick={handleNodeClick}
  onCollapseChange={handleCollapseChange}
/>
```

**完整链路**：

```
用户点击 "Rulers" 菜单项
    ↓
toggleRulers(!rulersEnabled)  [GraphView/Toolbar/index.tsx]
    ↓
useConfig store 更新 rulersEnabled 状态  [useConfig.ts]
    ↓
GraphView 组件重新渲染，读取新的 rulersEnabled  [GraphView/index.tsx]
    ↓
JSONCrack 组件的 key 属性变化（"RIGHT-false-true" → "RIGHT-false-false"）
    ↓
React 卸载旧的 JSONCrack 组件，挂载新的 JSONCrack 组件
    ↓
新组件使用新的 showGrid={rulersEnabled} prop 渲染
    ↓
画布网格显示/隐藏
```

### 4.3 主题同步的完整链路

主题切换涉及三层同步：Mantine 主题、styled-components 主题、JSONCrack 组件主题。

```
用户点击 "Dark Mode" 菜单项
    ↓
toggleDarkMode(!darkmodeEnabled)  [GraphView/Toolbar/index.tsx]
    ↓
useConfig store 更新 darkmodeEnabled 状态，persist 到 localStorage  [useConfig.ts]
    ↓
├─ 第一层：Mantine 主题同步
│   [editor.tsx#L119-L121] useEffect 监听 darkmodeEnabled 变化
│   setColorScheme(darkmodeEnabled ? "dark" : "light")
│   ↓
│   Mantine 组件库应用新主题
│
├─ 第二层：styled-components 主题同步
│   [editor.tsx#L134] ThemeProvider 接收新 theme
│   <ThemeProvider theme={darkmodeEnabled ? darkTheme : lightTheme}>
│   ↓
│   所有 styled-components 接收新主题变量
│
└─ 第三层：JSONCrack 组件主题同步
    [GraphView/index.tsx#L101] JSONCrack 接收新 theme prop
    theme={darkmodeEnabled ? "dark" : "light"}
    ↓
    JSONCrack 内部应用新的节点颜色、连线颜色等
```

**关键代码位置**：
- Mantine 同步：[editor.tsx#L119-L121](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/pages/editor.tsx#L119-L121)
- styled-components 同步：[editor.tsx#L134](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/pages/editor.tsx#L134)
- JSONCrack 同步：[GraphView/index.tsx#L101](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/index.tsx#L101)

---

## 5. 完整协作流程

### 5.1 导出图片流程（真实代码对齐）

```
┌─────────────────────────────────────────────────────────────┐
│                   导出图片流程（真实代码）                   │
└─────────────────────────────────────────────────────────────┘

1. 触发点（二选一）
   ├─ 工具栏导出按钮点击：onClick={() => setVisible("DownloadModal", true)}
   └─ 快捷键 Ctrl+S / ⌘+S：["mod+s", () => setVisible("DownloadModal", true)]
      ↓
2. [useModal.ts]
   └─ Zustand store 更新 { DownloadModal: true }
      ↓
3. [ModalController.tsx]
   └─ 监听状态变化，渲染 DownloadModal 组件
      ↓
4. 用户在模态框中操作后点击 "Download" 或 "Clipboard"
      ↓
   ├─ 分支 A：点击 Clipboard（clipboardImage 函数）
   │   ├─ toast.loading("Copying to clipboard...")
   │   ├─ html-to-image 执行 toBlob
   │   ├─ navigator.clipboard.write()
   │   ├─ ✅ 成功：toast.success("Copied to clipboard")
   │   └─ ❌ 失败：toast.error("Failed to copy to clipboard")
   │
   └─ 分支 B：点击 Download（exportAsImage 函数）
       ├─ toast.loading("Downloading...")
       ├─ html-to-image 执行 toPng/toJpeg/toSvg
       ├─ downloadURI() 触发浏览器下载
       ├─ ⚠️ 成功：无 Toast，只有浏览器下载通知
       └─ ❌ 失败：toast.error("Failed to download image!")
      ↓
5. finally 块
   ├─ toast.dismiss() 清除 loading toast
   └─ onClose() 关闭模态框
```

### 5.2 搜索功能流程（真实代码对齐）

```
┌─────────────────────────────────────────────────────────────┐
│                   搜索功能流程（真实代码）                   │
└─────────────────────────────────────────────────────────────┘

1. 触发点（二选一）
   ├─ 工具栏搜索按钮点击：onClick={handleSearchToggle}
   └─ 快捷键 Ctrl+F / ⌘+F：["mod+f", handleSearchToggle]
      ↓
2. handleSearchToggle() → setSearchOpen(true)
   ↓
3. 渲染 SearchInput 组件
   ├─ useEffect 自动聚焦输入框
   └─ getHotkeyHandler 注册 Enter/Shift+Enter/Esc
      ↓
4. 用户输入搜索词 → setValue(e.currentTarget.value)
   ↓
5. [useFocusNode.ts]
   ├─ useDebouncedValue 防抖 300ms
   ├─ searchQuery(`span[data-key*='...' i]`) 查找匹配节点
   ├─ highlightMatchedNodes() → 添加 .searched 和 .highlight CSS 类
   ├─ centerFitElementIntoView() 自动滚动居中
   └─ 更新 nodeCount 和 selectedNode 状态
      ↓
6. 界面反馈（无 Toast）
   ├─ 显示匹配计数："1 / 5"
   └─ 无结果时显示红色 "No matches"
      ↓
7. 用户按 Enter/Shift+Enter 切换匹配项
   └─ next()/prev() 更新 selectedNode，重复步骤 5
```

### 5.3 主题切换流程（真实代码对齐）

```
┌─────────────────────────────────────────────────────────────┐
│                   主题切换流程（真实代码）                   │
└─────────────────────────────────────────────────────────────┘

1. 触发点（三选一）
   ├─ 顶部工具栏 ThemeToggle 按钮
   ├─ 浮动工具栏 Preferences → Dark Mode / Light Mode
   └─ 底部状态栏（无直接按钮）
      ↓
2. 调用 toggleDarkMode(!darkmodeEnabled)
   ↓
3. [useConfig.ts]
   ├─ 更新 darkmodeEnabled 状态
   └─ persist 中间件自动保存到 localStorage
      ↓
4. 三层主题同步
   ├─ Mantine 主题：[editor.tsx#L119-L121] useEffect
   │   setColorScheme(darkmodeEnabled ? "dark" : "light")
   │
   ├─ styled-components 主题：[editor.tsx#L134]
   │   <ThemeProvider theme={darkmodeEnabled ? darkTheme : lightTheme}>
   │   ↓
   │   所有 styled-components 应用新主题变量
   │
   └─ JSONCrack 组件主题：[GraphView/index.tsx#L99-L101]
       key={[direction, gesturesEnabled, rulersEnabled].join("-")}
       theme={darkmodeEnabled ? "dark" : "light"}
       ↓
       由于 darkmodeEnabled 不在 key 中，组件不会重建，
       仅通过 prop 更新内部主题
      ↓
5. 图标同步更新
   ├─ ThemeToggle 按钮：月亮 ↔ 太阳
   └─ Preferences 菜单项文字：Dark Mode ↔ Light Mode
```

---

## 6. 关键代码路径汇总表

| 功能模块 | 入口文件 | 核心逻辑 | 状态管理 |
|---------|---------|---------|---------|
| 顶部工具栏 | [Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/index.tsx) | FileMenu, ViewMenu, ToolsMenu | useModal, useFile |
| 浮动工具栏 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx) | useHotkeys, Tooltip, Preferences 菜单 | useGraph, useConfig, useModal |
| 搜索功能 | [SearchInput.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/SearchInput.tsx) | getHotkeyHandler, useFocusNode | useGraph |
| 搜索高亮 | [search.ts](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/lib/utils/search.ts) | DOM classList 操作 | 无（直接操作 DOM） |
| 模态框系统 | [ModalController.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/modals/ModalController.tsx) | 动态渲染所有模态框 | useModal |
| 导出反馈 | [DownloadModal/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx) | toast loading/success/error | 无（直接调用 toast） |
| 底部状态栏 | [BottomBar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/BottomBar.tsx) | 状态指示器、格式切换 | useFile, useConfig, useGraph |
| 全局快捷键 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L130-L141) | useHotkeys | 多个 store |
| 画布重建 | [GraphView/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/index.tsx#L99) | React key 变化触发重建 | 无（React 机制） |
| 主题同步 | [editor.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/pages/editor.tsx#L119-L134) | Mantine + styled-components 双重主题 | useConfig |

---

## 7. 设计特点总结

### 7.1 架构优点

1. **关注点分离**：每个 store 只负责特定领域的状态和动作，职责清晰
2. **声明式快捷键**：使用 Mantine Hooks 以声明式方式定义快捷键，代码可读性高
3. **响应式反馈**：基于 Zustand 的订阅机制，UI 自动响应状态变化
4. **跨平台兼容**：`mod` 修饰符自动适配不同操作系统
5. **渐进式反馈**：Tooltip 提示 → 视觉状态变化 → Toast 通知 → 模态框，形成完整的反馈链路
6. **动态可扩展**：模态框系统无需修改核心代码即可添加新模态框
7. **强制重建机制**：通过 React key 变化确保画布参数变更时完全重建，避免状态污染

### 7.2 数据流方向

```
用户输入
    ↓
快捷键/按钮 → 动作函数 (+GA埋点) → Zustand Store → UI 组件 → 用户反馈
    ↑                                                    │
    └──────────────────── 状态订阅 ──────────────────────┘
```

### 7.3 快捷键清单（对齐真实代码）

| 快捷键 | 功能 | 注册位置 | 与按钮共用动作 |
|--------|------|----------|---------------|
| `Ctrl/⌘ + +` | 放大 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L132) | 共用 zoomIn，按钮多 gaEvent |
| `Ctrl/⌘ + -` | 缩小 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L133) | 共用 zoomOut，按钮多 gaEvent |
| `Shift + 1` | 聚焦第一个节点 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L134) | 共用 focusFirstNode，按钮多 gaEvent |
| `Shift + 2` | 居中视图 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L135) | 共用 centerView，按钮多 gaEvent |
| `Ctrl/⌘ + F` | 搜索节点 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L136) | 完全共用 handleSearchToggle |
| `Ctrl/⌘ + S` | 导出图片 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L137) | 完全共用 setVisible |
| `Ctrl/⌘ + Shift + D` | 旋转布局 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L138) | 完全共用 toggleDirection |
| `Enter` | 下一个匹配项（搜索时） | [SearchInput.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/SearchInput.tsx#L69) | 完全共用 next |
| `Shift + Enter` | 上一个匹配项（搜索时） | [SearchInput.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/SearchInput.tsx#L70) | 完全共用 prev |
| `Escape` | 关闭搜索 | [SearchInput.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/SearchInput.tsx#L71) | 完全共用 handleClose |
