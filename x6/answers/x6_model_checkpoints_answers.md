# X6 Model 层检查点答案

## 检查点问题解答

### 1. 你能解释 resetCells 为何分两次 prepareCell 吗？

**答案：** `resetCells` 分两次调用 `prepareCell` 是为了解决**边引用旧节点**的问题。

**源码分析：**
```typescript
resetCells(cells: Cell[], options: Collection.SetOptions = {}) {
  // 第一次：dryrun=true，不更新model，只准备cell
  cells.map((cell) => this.prepareCell(cell, { ...options, dryrun: true }))
  this.collection.reset(cells, options)
  // 第二次：更新model，触发边更新其引用
  cells.map((cell) => this.prepareCell(cell, { options }))
  return this
}
```

**原因分析：**

1. **第一次 prepareCell（dryrun=true）**：
   - 设置 `cell.model = this`，但不触发事件
   - 设置 `zIndex`，但不更新模型状态
   - 目的是让边能够找到新的节点引用

2. **Collection.reset()**：
   - 清空旧集合，添加新集合
   - 触发 `reseted` 事件
   - 此时边的 source/target 已经指向新节点

3. **第二次 prepareCell（正常模式）**：
   - 正式设置 `cell.model = this`
   - 触发边更新其引用关系
   - 确保所有引用关系正确建立

**为什么需要两次？**
- 如果只调用一次，边在更新引用时可能找不到目标节点
- 第一次确保节点已准备就绪，第二次确保引用关系正确建立
- 避免了"边引用旧节点"的问题

### 2. 你能用 getSubGraph/cloneSubGraph 实现子图复制吗？

**答案：** 可以，这是 X6 提供的标准子图复制功能。

**实现方式：**

```typescript
// 1. 获取子图（包含所有相关节点和边）
const subgraph = model.getSubGraph(selectedCells, { deep: true })

// 2. 克隆子图
const cloneMap = model.cloneSubGraph(selectedCells, { deep: true })

// 3. 将克隆的节点添加到图中
const clonedCells = Object.values(cloneMap)
model.addCells(clonedCells)
```

**getSubGraph 功能：**
- 收集指定节点及其所有相关边
- 如果 `deep: true`，包含嵌套的子节点
- 自动包含边的 source/target 节点
- 确保子图的完整性

**cloneSubGraph 功能：**
- 基于 `getSubGraph` 获取完整子图
- 使用 `Cell.cloneCells` 进行深度克隆
- 自动处理边的引用关系更新
- 返回 `{ [原ID]: [克隆对象] }` 的映射

**实际应用场景：**
```typescript
// 复制选中的节点及其连接
const selectedNodes = graph.getSelectedCells()
const cloneMap = graph.model.cloneSubGraph(selectedNodes, { deep: true })

// 计算偏移量，避免重叠
const offset = { x: 100, y: 100 }
Object.values(cloneMap).forEach(cell => {
  if (cell.isNode()) {
    cell.translate(offset.x, offset.y)
  }
})

// 添加到图中
graph.addCells(Object.values(cloneMap))
```

### 3. 删除节点时，何时应选择 disconnectConnectedEdges？

**答案：** 当需要**保留边但断开连接**时选择 `disconnectConnectedEdges`，当需要**完全删除边**时选择 `removeConnectedEdges`。

**两种方法的区别：**

**disconnectConnectedEdges（断开连接）：**
```typescript
disconnectConnectedEdges(cell: Cell | string, options: Edge.SetOptions = {}) {
  const cellId = typeof cell === 'string' ? cell : cell.id
  this.getConnectedEdges(cell).forEach((edge) => {
    const sourceCellId = edge.getSourceCellId()
    const targetCellId = edge.getTargetCellId()

    if (sourceCellId === cellId) {
      edge.setSource({ x: 0, y: 0 }, options)  // 断开源连接
    }

    if (targetCellId === cellId) {
      edge.setTarget({ x: 0, y: 0 }, options)  // 断开目标连接
    }
  })
}
```
- **保留边**：边仍然存在于图中
- **断开连接**：将边的 source/target 设置为坐标点
- **适用场景**：临时断开连接，后续可能重新连接

**removeConnectedEdges（删除边）：**
```typescript
removeConnectedEdges(cell: Cell | string, options: Cell.RemoveOptions = {}) {
  const edges = this.getConnectedEdges(cell)
  edges.forEach((edge) => {
    edge.remove(options)  // 完全删除边
  })
  return edges
}
```
- **删除边**：边从图中完全移除
- **清理引用**：自动清理所有相关引用
- **适用场景**：永久删除，不需要保留边

**使用场景对比：**

| 场景 | 选择方法 | 原因 |
|------|----------|------|
| 临时隐藏节点 | `disconnectConnectedEdges` | 保留边，便于恢复 |
| 永久删除节点 | `removeConnectedEdges` | 清理所有相关边 |
| 节点重构 | `disconnectConnectedEdges` | 保留边结构，重新连接 |
| 数据清理 | `removeConnectedEdges` | 彻底清理无用数据 |
| 撤销操作 | `disconnectConnectedEdges` | 便于撤销时恢复连接 |

**实际使用：**
```typescript
// 场景1：临时断开连接
model.removeCell(node, { disconnectEdges: true })
// 等价于：disconnectConnectedEdges

// 场景2：完全删除
model.removeCell(node, { disconnectEdges: false })
// 等价于：removeConnectedEdges（默认行为）

// 场景3：手动控制
model.disconnectConnectedEdges(node)  // 断开连接
model.removeConnectedEdges(node)      // 删除边
```

**总结：**
- `disconnectConnectedEdges`：保留边，断开连接，适用于临时操作
- `removeConnectedEdges`：删除边，清理引用，适用于永久删除
- 选择依据：是否需要保留边的结构和数据
