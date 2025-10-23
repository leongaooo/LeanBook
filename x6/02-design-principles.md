## 02 设计原则与工程实践

本章聚焦 X6 的设计取舍与工程化实践，帮助你在阅读/贡献代码时对齐设计意图。

### 1. 核心设计原则

- **组合优于继承**：Graph 通过组合多个 Manager（transform/grid/highlight/...）组织能力，保持组件职责单一；
- **数据驱动**：Model 作为单一数据真相源，事件驱动同步视图；
- **可插拔**：Registry + Plugins 形成“注册-查找-使用”的扩展通道；
- **最小 API 面**：Graph 对外提供稳定简洁 API，内部通过 Manager 演进；
- **强类型**：TypeScript 全面建模，保证扩展点的类型安全；
- **渐进增强**：默认开箱即用，可按需开启插件或替换注册项。

### 2. 配置体系：友好入门与强大定制

- `Options.Manual`（用户输入） → `Options.Definition`（内部一体化）
- `Options.get()` 合并默认值与布尔/数字快捷写法，降低使用复杂度；
- 对交互（connecting/translating/embedding/highlighting）提供细粒度策略函数。

### 3. 事件与生命周期

- 事件分层：`model:*`、`blank:*`、`cell:*`、`view:*`、`renderer:*`；
- Graph 统一转发 Model 事件，便于业务只依赖 Graph；
- 插件生命周期：`init` → `enable/disable` → `dispose`，支持动态开关与资源回收。

### 4. 性能策略

- Renderer 任务调度与批量更新，避免抖动；
- 虚拟渲染（`VirtualRender`）减少大图渲染压力；
- 变换/坐标换算下沉到 `Transform/Coord`，复用矩阵与计算结果。

### 5. 可测试性

- Model/Graph/View/Plugin 均有独立职责与边界，可单测覆盖；
- `__tests__/` 提供丰富用例，示例作为集成回归用；
- API 设计倾向纯函数/幂等行为，降低副作用。

### 6. 对贡献者的重要约束

- 不破坏 Graph 对外 API 的稳定性；
- 新特性尽量以插件或注册项形式提供；
- 代码需具备可测试性与文档/示例；
- 遵循现有事件命名与类型建模方式；
- 性能敏感路径（渲染、坐标、拓扑）需有数据对比。



