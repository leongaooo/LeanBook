## 08 形状系统（Shapes）

目标：掌握内置形状与自定义节点/边的开发方式。

### 1. 内置形状

- 目录：`src/shape/`
- 导出：`Rect/Circle/Ellipse/Path/Polygon/Polyline/Image/HTML/TextBlock/Edge`
- 典型属性：`attrs.body/*`、`attrs.label/*`，与 `markup` 的 selector 对应。

### 2. 自定义形状步骤

1) 定义 `markup` 与 `defaults`；
2) 可选定义 `attrHooks/propHooks` 做属性预处理；
3) `Graph.registerNode('your-node', spec)` 或 `Graph.registerEdge('your-edge', spec)`；
4) 通过 `shape: 'your-node'` 使用。

### 3. 与注册表的关系

- 形状内部可引用 `router/connector/marker/anchor/connectionPoint` 等注册项；
- 形状本身也通过注册方式暴露给 Graph。

### 4. 检查点

- 何时需要自定义 `markup` 而不是仅用 `attrs`？
- 文本/HTML 节点与 SVG 节点混用的取舍是什么？







