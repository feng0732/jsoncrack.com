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
- 快捷键与工具栏按钮共享相同的动作处理函数

### 2.3 局部快捷键处理

在 [SearchInput.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/SearchInput.tsx#L68-L72) 中使用 `getHotkeyHandler` 处理组件级快捷键：

```typescript
onKeyDown={getHotkeyHandler([
  ["Enter", next],           // 下一个匹配项
  ["shift+Enter", prev],     // 上一个匹配项
  ["Escape", handleClose],   // 关闭搜索
])}
```

这种方式确保快捷键仅在组件获得焦点时生效，避免与全局快捷键冲突。

### 2.4 快捷键检测与提示

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

使用 `react-hot-toast` 提供即时操作反馈，支持三种状态：

| 状态 | 示例位置 |
|------|----------|
| 加载中 | [DownloadModal/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L79) |
| 成功 | [DownloadModal/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L102) |
| 错误 | [DownloadModal/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/modals/DownloadModal/index.tsx#L110) |

**代码模式**：

```typescript
const exportAsImage = async () => {
  try {
    toast.loading("Downloading...", { id: "toastDownload" });
    // ... 执行操作
    toast.success("Download complete");
  } catch {
    toast.error("Failed to download image!");
  } finally {
    toast.dismiss("toastDownload");  // 确保 loading toast 被清除
    onClose();
  }
};
```

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

### 3.4 搜索反馈

[useFocusNode.ts](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/hooks/useFocusNode.ts) 提供多层次搜索反馈：

1. **计数反馈**：`{selectedNode + 1} / {nodeCount}` 显示当前位置
2. **无结果反馈**：红色显示 "No matches"
3. **视觉高亮**：`highlightMatchedNodes()` 高亮匹配节点
4. **自动滚动**：`centerFitElementIntoView()` 自动将匹配项滚入视图

---

## 4. 完整协作流程

### 4.1 导出图片流程

```
┌─────────────────────────────────────────────────────────────┐
│                        导出图片流程                          │
└─────────────────────────────────────────────────────────────┘

1. 触发点（二选一）
   ├─ 工具栏导出按钮点击
   └─ 快捷键 Ctrl+S / ⌘+S
      ↓
2. [GraphView/Toolbar/index.tsx]
   ├─ onClick 或 useHotkeys 触发
   └─ 调用 setVisible("DownloadModal", true)
      ↓
3. [useModal.ts]
   └─ Zustand store 更新 { DownloadModal: true }
      ↓
4. [ModalController.tsx]
   └─ 监听状态变化，渲染 DownloadModal 组件
      ↓
5. 用户在模态框中操作后点击 Download
   ↓
6. [DownloadModal/index.tsx] exportAsImage()
   ├─ toast.loading("Downloading...") 显示加载状态
   ├─ html-to-image 执行导出
   ├─ 成功：toast.success("Download complete")
   └─ 失败：toast.error("Failed to download")
      ↓
7. finally 块
   ├─ toast.dismiss() 清除 loading toast
   └─ onClose() 关闭模态框
```

### 4.2 搜索功能流程

```
┌─────────────────────────────────────────────────────────────┐
│                        搜索功能流程                          │
└─────────────────────────────────────────────────────────────┘

1. 触发点（二选一）
   ├─ 工具栏搜索按钮点击
   └─ 快捷键 Ctrl+F / ⌘+F
      ↓
2. handleSearchToggle() → setSearchOpen(true)
   ↓
3. 渲染 SearchInput 组件
   ├─ useEffect 自动聚焦输入框
   └─ getHotkeyHandler 注册 Enter/Shift+Enter/Esc
      ↓
4. 用户输入搜索词
   ↓
5. [useFocusNode.ts]
   ├─ useDebouncedValue 防抖 300ms
   ├─ searchQuery() 查找匹配节点
   ├─ highlightMatchedNodes() 高亮显示
   ├─ centerFitElementIntoView() 自动滚动居中
   └─ 更新 nodeCount 和 selectedNode 状态
      ↓
6. 界面反馈
   ├─ 显示匹配计数："1 / 5"
   └─ 无结果时显示红色 "No matches"
      ↓
7. 用户按 Enter/Shift+Enter 切换匹配项
   └─ next()/prev() 更新 selectedNode，重复步骤 5
```

### 4.3 主题切换流程

```
┌─────────────────────────────────────────────────────────────┐
│                        主题切换流程                          │
└─────────────────────────────────────────────────────────────┘

1. 触发点（三选一）
   ├─ 顶部工具栏 ThemeToggle 按钮
   ├─ 浮动工具栏 Preferences 菜单
   └─ 底部状态栏（无直接按钮，可扩展）
      ↓
2. 调用 toggleDarkMode(!darkmodeEnabled)
   ↓
3. [useConfig.ts]
   ├─ 更新 darkmodeEnabled 状态
   └─ persist 中间件自动保存到 localStorage
      ↓
4. [editor.tsx] useEffect 监听状态变化
   ├─ setColorScheme() 更新 Mantine 主题
   └─ styled-components ThemeProvider 响应变化
      ↓
5. 全应用主题切换
   ├─ 所有 styled-components 接收新 theme
   ├─ 按钮图标切换：月亮 ↔ 太阳
   └─ 编辑器主题同步更新（monaco-editor）
```

---

## 5. 关键代码路径汇总表

| 功能模块 | 入口文件 | 核心逻辑 | 状态管理 |
|---------|---------|---------|---------|
| 顶部工具栏 | [Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/index.tsx) | FileMenu, ViewMenu, ToolsMenu | useModal, useFile |
| 浮动工具栏 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx) | useHotkeys, Tooltip | useGraph, useConfig |
| 搜索功能 | [SearchInput.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/SearchInput.tsx) | getHotkeyHandler, useFocusNode | useGraph |
| 模态框系统 | [ModalController.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/modals/ModalController.tsx) | 动态渲染所有模态框 | useModal |
| 底部状态栏 | [BottomBar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/BottomBar.tsx) | 状态指示器、格式切换 | useFile, useConfig, useGraph |
| 全局快捷键 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L130-L141) | useHotkeys | 多个 store |

---

## 6. 设计特点总结

### 6.1 架构优点

1. **关注点分离**：每个 store 只负责特定领域的状态和动作，职责清晰
2. **声明式快捷键**：使用 Mantine Hooks 以声明式方式定义快捷键，代码可读性高
3. **响应式反馈**：基于 Zustand 的订阅机制，UI 自动响应状态变化
4. **跨平台兼容**：`mod` 修饰符自动适配不同操作系统
5. **渐进式反馈**：Tooltip 提示 → 视觉状态变化 → Toast 通知 → 模态框，形成完整的反馈链路
6. **动态可扩展**：模态框系统无需修改核心代码即可添加新模态框

### 6.2 数据流方向

```
用户输入
    ↓
快捷键/按钮 → 动作函数 → Zustand Store → UI 组件 → 用户反馈
    ↑                                                    │
    └──────────────────── 状态订阅 ──────────────────────┘
```

### 6.3 快捷键清单

| 快捷键 | 功能 | 注册位置 |
|--------|------|----------|
| `Ctrl/⌘ + +` | 放大 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L132) |
| `Ctrl/⌘ + -` | 缩小 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L133) |
| `Shift + 1` | 聚焦第一个节点 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L134) |
| `Shift + 2` | 居中视图 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L135) |
| `Ctrl/⌘ + F` | 搜索节点 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L136) |
| `Ctrl/⌘ + S` | 导出图片 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L137) |
| `Ctrl/⌘ + Shift + D` | 旋转布局 | [GraphView/Toolbar/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/views/GraphView/Toolbar/index.tsx#L138) |
| `Enter` | 下一个匹配项（搜索时） | [SearchInput.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/SearchInput.tsx#L69) |
| `Shift + Enter` | 上一个匹配项（搜索时） | [SearchInput.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/SearchInput.tsx#L70) |
| `Escape` | 关闭搜索 | [SearchInput.tsx](file:///d:/fz/0601/solo-dogfeeding/code/192-jsoncrack.com/apps/www/src/features/editor/Toolbar/SearchInput.tsx#L71) |
