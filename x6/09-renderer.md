## 09 渲染器（Renderer）与性能优化

目标：理解渲染调度、视图管理与性能策略。

### 1. 文件与职责

- `src/renderer/renderer.ts`：视图创建/查找/更新/销毁；
- `src/renderer/scheduler.ts`：调度与批处理；
- `src/renderer/queueJob.ts`：作业队列。

### 2. 工作流

1) Model 变化 → Graph 触发事件；
2) Renderer 计算受影响的 `CellView`；
3) 合并更新（微任务/宏任务）后批量刷新；
4) 仅更新差异节点，减少重绘。

### 3. 性能策略

- 批处理：`batchUpdate` + 渲染调度；
- 虚拟渲染：仅渲染可视区域；
- 命中优化：缓存坐标换算与包围盒；
- 结构化 DOM：通过 `markup + selectors` 精准更新。

### 4. 检查点

- 在大图场景下如何识别“可见区域”？
- 何时应当拆分更新为多个帧以避免掉帧？







