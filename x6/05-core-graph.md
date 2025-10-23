## 05 控制器（Graph）与交互编排

目标：掌握 Graph 的组合式设计、配置体系与对外 API。

### 1. Graph 的组合式管理器

- `transform` 视口矩阵/缩放/平移/居中/自适应；
- `coord` 坐标系互转（page/client/local/graph）；
- `grid` 网格绘制与网格对齐；
- `background` 画布背景；
- `highlight` 交互高亮；
- `renderer` 视图查找/渲染调度；
- `panning` 平移；`mousewheel` 缩放；`virtualRender` 虚拟渲染；
- `size` 尺寸与自适应。

### 2. 配置体系（Options）

- `Options.Manual` → `Options.Definition`；
- 核心子配置：`grid/panning/mousewheel/connecting/translating/embedding/highlighting`；
- 连接策略（connecting）：`anchor/edgeAnchor/connectionPoint/router/connector` 与 `validateConnection/createEdge`。

### 3. 对外常用 API

- 数据：`addNode/addEdge/addCell/removeCell/fromJSON/toJSON/getNodes/getEdges`；
- 视图：`findView/findViewsFromPoint/findViewsInArea`；
- 交互：`scale/zoom/translate/center/zoomToRect/zoomToFit/positionCell`；
- 背景与网格：`drawBackground/drawGrid/setGridSize`；
- 插件：`use/enablePlugins/disablePlugins/isPluginEnabled/disposePlugins`。

### 4. 事件

- `model:*` 事件代理（`model:updated/model:reseted/...`）；
- 画布空白区：`blank:click/dblclick/contextmenu/mousedown/mousemove/mouseup`；
- 变换：`scale/resize/translate`。

### 5. 检查点

- Graph 为什么要持有 `Model` 与 `Renderer`？
- 如何通过配置关闭空白区域默认行为？
- 何时使用 `batchUpdate` 包裹你的数据更新？



