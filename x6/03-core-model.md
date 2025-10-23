## 03 核心模型层（Model）深度解析

目标：掌握 Model/Cell/Node/Edge/Collection 的职责、协作与关键算法。

### 1. 文件与职责

- `src/model/cell.ts`：Cell 基类（属性存储、事件、层级、动画钩子）。
- `src/model/node.ts`：Node（几何、嵌套/分组、端口）。
- `src/model/edge.ts`：Edge（端点、路由、连线状态）。
- `src/model/model.ts`：图数据容器（增删改查、查询/邻接、最短路径、批处理）。
- `src/model/collection.ts`：Cell 集合（排序、索引、事件）。
- `src/model/store.ts`：属性存储与变更事件分发。

### 2. Cell：数据原子与事件源

- 静态配置：`markup/defaults/attrHooks/propHooks` —— 形状与属性加工入口；
- 生命周期：`preprocess → setup → init → postprocess`；
- 事件：`change:*`（任意属性）、`change:key`（特定属性）、`changed`（批量完成）；
- 结构：`parent/children` 管理嵌套节点；
- 动画：`Animation` 注入支持过渡效果（视图层消费）。

重点 API：
- `setProp/getProp/prop(path, value)`：精细化更新；
- `translate/resize/scale/rotate`：几何变换写入模型；
- `getBBox()`：统一获取包围盒（与几何层协作）。

### 3. Model：图数据容器与图算法入口

- 结构缓存：`nodes/edges/outgoings/incomings`，支持 O(1) 查询出/入边；
- 变更批处理：`startBatch/stopBatch/batchUpdate`，降低事件风暴；
- 结构操作：`addCell/removeCell/resetCells/clear`；
- 图查询：`getNeighbors/getSuccessors/getPredecessors/getSubGraph/cloneSubGraph`；
- 路径：`getShortestPath` 基于 Dijkstra；
- 边/节点边界：`getRoots/getLeafs/isRoot/isLeaf`。

关键点：
- `fromJSON/toJSON` 支持全量与 diff 更新；
- `updateCellId` 保证连边端点引用一致性；
- `disconnectConnectedEdges/removeConnectedEdges` 安全删除节点。

### 4. Node/Edge：特化能力

- Node：尺寸/位置/角度、端口系统（`PortManager`）、嵌套与布局协作；
- Edge：`source/target` 终端、路由（router）、连接器（connector）、标记（marker）、锚点（anchor）。

### 5. 事件流与性能

- Model 产生的事件由 Graph 聚合转发（`model:*`）；
- 大量更新使用 `batchUpdate` 包裹，视图侧做一次性刷新；
- 索引缓存避免 N² 查询。

### 6. 检查点

- 你能解释 `resetCells` 为何分两次 `prepareCell` 吗？
- 你能用 `getSubGraph/cloneSubGraph` 实现子图复制吗？
- 删除节点时，何时应选择 `disconnectConnectedEdges`？



