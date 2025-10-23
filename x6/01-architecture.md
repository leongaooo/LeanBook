## 01 架构总览与模块关系

本章目标：
- 建立对 X6 的整体认知（模块/职责/协作）。
- 明确「数据 → 视图 → 交互」在框架内的流转路径。
- 掌握阅读源码的导航图。

### 1. 总体架构：MVC + 插件化 + 注册表

- **Model（数据层）**：`src/model/` 管理节点 Node、边 Edge、集合 Collection、图 Model；提供查询、增删改、拓扑分析、序列化等能力。
- **View（视图层）**：`src/view/` 负责 Cell 的可视化呈现（DOM/SVG/HTML），事件绑定、命中检测、局部更新。
- **Graph（控制器）**：`src/graph/graph.ts` 协调 Model 与 View，提供画布级 API、交互编排、坐标变换与布局适配。
- **Renderer（渲染）**：`src/renderer/` 组织调度视图的创建/更新/销毁，保障渲染效率（任务调度、批量更新）。
- **Registry（注册表）**：`src/registry/` 统一可插拔要素（router/connector/anchor/attr/filter/grid/...）的注册与查找。
- **Plugins（插件）**：`src/plugin/` 以插件形式扩展交互与编辑功能，如 selection、transform、keyboard、minimap、scroller、history、export 等。
- **Shapes（形状）**：`src/shape/` 内置节点/边/文本/HTML 等形状，结合注册表形成开放的形状体系。
- **Common（通用能力）**：`src/common/` 包含事件系统、数据结构、DOM 封装、算法、动画、工具集等底座能力。
- **Geometry（几何）**：`src/geometry/` 点/线/矩形/曲线/路径等几何算法。

数据流简述：
1) 开发者通过 Graph API 改变数据（Model）；2) Model 触发事件 → Graph 转发；3) Renderer 驱动对应 CellView 局部更新；4) 交互事件由 View 捕获 → 通过 Graph/Plugin 反馈到 Model。

### 2. 关键协作关系

- `Graph` 持有 `Model`、`Renderer` 与诸多 Manager（transform/grid/background/coord/...）。
- `Model` 管理 `Cell`（`Node`/`Edge`），并维护出入度缓存、子父层级关系。
- `CellView` 与 `Cell` 一一对应，`Renderer` 负责查找/创建/更新/销毁 `CellView`。
- `Registry` 为 `Graph.registerXxx()` 提供背书，支撑运行时扩展。
- `Plugin` 通过 `graph.use(plugin)` 接入生命周期与能力面板。

### 3. 目录导航与源码入口

- 入口导出：`src/index.ts`（聚合导出 Shape、Registry、Model、View、Graph、Plugin、Common、Geometry）。
- Graph 核心：`src/graph/graph.ts`、`src/graph/options.ts`、`src/graph/events.ts`。
- Model 核心：`src/model/model.ts`、`src/model/cell.ts`、`src/model/node.ts`、`src/model/edge.ts`、`src/model/collection.ts`。
- View/Renderer：`src/view/**`、`src/renderer/**`。
- Registry/Shapes：`src/registry/**`、`src/shape/**`。
- 示例入口：`examples/src/pages/graph/index.tsx` 与 `render.ts`。

### 4. 架构优势

- **强扩展性**：注册表 + 插件化，运行时插拔自定义能力；
- **职责清晰**：Model/Graph/View 分责，便于测试与维护；
- **性能可控**：Renderer 调度、局部视图更新、虚拟渲染（VirtualRender）；
- **数据驱动**：Model 为真相源，事件驱动同步视图；
- **生态完备**：内置丰富插件与形状；示例与站点完善。

### 5. 学习检查点

- 你能否画出「Graph - Model - Renderer - View - Plugin - Registry」关系图？
- 你能指出 Graph 的构造顺序以及为何这样安排？
- 你能说明一次 `graph.addNode()` 到视图出现的完整链路吗？

### 6. 思考题

- 若要增加“对齐参考线”的新高阶功能，你会做成「插件」还是「注册项」？为什么？
- 如果要支持 WebGL 渲染，应在哪一层进行替换与抽象收敛？



