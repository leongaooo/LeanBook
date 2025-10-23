## 10 通用能力（Common）与工具库

目标：了解支撑全局的基础设施，便于复用与贡献。

### 1. 目录与要点

- `src/common/algorithm`：Dijkstra 等；
- `src/common/event`：事件系统与类型；
- `src/common/dom`：DOM 操作与事件封装；
- `src/common/object/array/function/string/number/util`：常用工具；
- `src/common/base`：`Basecoat` 混入 `Disposable`，统一 `dispose`；
- `src/common/animation`：动画时序与过渡辅助。

### 2. 对上层的价值

- 降低实现复杂度与代码重复；
- 统一工程风格（事件、可释放资源、工具方法）；
- 强类型与明确定义减少歧义。

### 3. 检查点

- 你能解释 `Basecoat` 的职责与 `dispose` 的重要性吗？
- 何时应下沉通用逻辑到 `common` 以便复用？







