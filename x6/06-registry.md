## 06 注册表（Registry）—— 可插拔扩展的基座

目标：理解注册/查找/卸载机制，掌握可插拔要素的扩展方式。

### 1. 注册表总览

- 位置：`src/registry/**`
- 能力：统一注册/反注册/查询能力；
- 覆盖项：`attr/background/connection-point/connector/edge-anchor/filter/grid/highlighter/marker/node-anchor/port-layout/port-label-layout/router/tool`。

### 2. Graph 的注册 API 透出

```ts
Graph.registerNode
Graph.registerEdge
Graph.registerView
Graph.registerAttr
Graph.registerGrid
Graph.registerFilter
Graph.registerNodeTool
Graph.registerEdgeTool
Graph.registerBackground
Graph.registerHighlighter
Graph.registerPortLayout
Graph.registerPortLabelLayout
Graph.registerMarker
Graph.registerRouter
Graph.registerConnector
Graph.registerAnchor
Graph.registerEdgeAnchor
Graph.registerConnectionPoint
```

对应也有 `unregisterXxx` 系列。

### 3. 设计要点

- 运行时扩展：无需改动核心代码；
- 命名约束：避免与内置重复，建议带前缀；
- 类型安全：注册项的 `args/options` 有完整类型定义；
- 与 Shape/Plugin 协同：形状引用注册项；插件可在 `init` 时注册能力。

### 4. 检查点

- 你能实现一个自定义 `router` 并注册到 Graph 吗？
- 若要在插件启用时注册、禁用时卸载，该把注册逻辑放在哪里？



