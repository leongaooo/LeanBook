## 04 视图层（View）与渲染协作

目标：理解 View 如何承接 Model，并与 Renderer 协作完成高效渲染与交互。

### 1. 文件与职责

- `src/view/view/index.ts`：`View` 基类（DOM 操作、事件委托、数据缓存）。
- `src/view/cell/**`：`CellView` 及其派生（`NodeView/EdgeView`）。
- `src/view/markup/**`：视图标记（markup）与选择器（selectors）体系。

### 2. View 基类能力

- DOM 操作：`setClass/addClass/removeClass/setStyle/setAttrs/empty/remove`；
- 事件委托：`delegateEvents/undelegateEvents` 与文档级事件；
- 工具方法：`find/findOne/findByAttr/getSelector/prefixClassName`；
- 事件数据：`getEventData/setEventData/eventData/normalizeEvent`；
- 生命周期：`dispose()` 自动解绑事件与 DOM。

### 3. CellView 协作点

- 绑定 Model 变化事件（`change:*`） → 局部更新对应 DOM；
- 使用 `markup + selectors` 进行结构化渲染与命中；
- 与 `Highlighter`、`Tools`（工具组件）协同响应交互状态。

### 4. 交互与命中

- 命中策略由 `Options.interacting` 与高亮配置 `highlighting` 控制；
- 事件坐标通过 `Coord/Transform` 统一换算；
- 常见交互：选中/拖拽/连接/编辑标签/手柄工具等由插件注入。

### 5. 检查点

- 你能解释 `delegateEvents` 的好处与命名空间策略吗？
- 如何在 `CellView` 中仅更新改变的局部 DOM？
- `findByAttr('magnet')` 在连线交互中的作用是什么？



