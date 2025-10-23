# X6 Renderer 层检查点答案

## 检查点问题解答

### 1. 在大图场景下如何识别"可见区域"？

**答案：** 通过**虚拟渲染机制**识别可见区域，结合**视口计算**、**滚动位置**和**变换矩阵**来确定需要渲染的图形区域。

**可见区域识别机制：**

#### 1. 基础可见区域计算
```typescript
// src/graph/virtual-render.ts
export class VirtualRenderManager extends Base {
  resetRenderArea() {
    if (this.options.virtual) {
      // 获取当前图形区域
      const renderArea = this.graph.getGraphArea()
      if (renderArea) {
        // 为可视区域增加固定缓冲边距
        const VIRTUAL_RENDER_MARGIN = 120
        const eff = renderArea.clone()
        eff.inflate(VIRTUAL_RENDER_MARGIN, VIRTUAL_RENDER_MARGIN)
        this.graph.renderer.setRenderArea(eff)
        return
      }
    }
  }
}
```

#### 2. 视口变换计算
```typescript
// 获取当前视口区域
getGraphArea() {
  const scroller = this.getPlugin<any>('scroller')
  if (scroller) {
    // 从 Scroller 获取可见区域
    const area = scroller.getVisibleArea?.()
    if (area) return area
  }
  // 从 Transform 获取图形区域
  return this.transform.getGraphArea()
}
```

#### 3. 滚动位置监听
```typescript
// 监听滚动事件更新可见区域
private bindScrollerEvents(scroller: any) {
  this.scrollerRef = scroller
  if (typeof scroller.on === 'function') {
    scroller.on('pan:start', this.resetRenderArea, this)
    scroller.on('panning', this.resetRenderArea, this)
    scroller.on('pan:stop', this.resetRenderArea, this)
  }

  const container = scroller.container
  if (container) {
    this.scrollerScrollHandler = (_e) => {
      this.resetRenderArea()
    }
    Dom.Event.on(container, 'scroll', this.scrollerScrollHandler)
  }
}
```

#### 4. 变换事件监听
```typescript
// 监听图形变换事件
protected startListening() {
  this.graph.on('translate', this.resetRenderArea, this)
  this.graph.on('scale', this.resetRenderArea, this)
  this.graph.on('resize', this.resetRenderArea, this)
}
```

#### 5. 可见性判断
```typescript
// src/renderer/scheduler.ts
protected isUpdatable(view: CellView) {
  if (view.isNodeView()) {
    if (this.renderArea) {
      // 检查节点是否与可见区域相交
      return this.renderArea.isIntersectWithRect(view.cell.getBBox())
    }
    return true
  }

  if (view.isEdgeView()) {
    const edge = view.cell
    const intersects = this.renderArea
      ? this.renderArea.isIntersectWithRect(edge.getBBox())
      : true

    if (this.graph.options.virtual) {
      return intersects
    }

    // 检查边的源节点和目标节点是否在可见区域内
    const sourceCell = edge.getSourceCell()
    const targetCell = edge.getTargetCell()
    if (this.renderArea && sourceCell && targetCell) {
      return (
        this.renderArea.isIntersectWithRect(sourceCell.getBBox()) ||
        this.renderArea.isIntersectWithRect(targetCell.getBBox())
      )
    }
  }

  return true
}
```

#### 6. 完整的大图可见区域识别实现
```typescript
export class LargeGraphRenderer {
  private renderArea?: Rectangle
  private virtualMargin = 120

  // 启用虚拟渲染
  enableVirtualRender() {
    this.options.virtual = true
    this.resetRenderArea()
  }

  // 重置渲染区域
  resetRenderArea() {
    if (this.options.virtual) {
      // 1. 获取当前视口区域
      const viewport = this.getViewportArea()

      // 2. 应用变换矩阵
      const transformedArea = this.applyTransform(viewport)

      // 3. 添加缓冲边距
      const renderArea = transformedArea.clone()
      renderArea.inflate(this.virtualMargin, this.virtualMargin)

      // 4. 设置渲染区域
      this.setRenderArea(renderArea)
    }
  }

  // 获取视口区域
  private getViewportArea(): Rectangle {
    const scroller = this.graph.getPlugin<any>('scroller')
    if (scroller) {
      // 从 Scroller 获取可见区域
      return scroller.getVisibleArea() || this.getDefaultViewport()
    }

    // 从 Transform 获取图形区域
    return this.graph.transform.getGraphArea()
  }

  // 应用变换矩阵
  private applyTransform(area: Rectangle): Rectangle {
    const transform = this.graph.transform.getMatrix()
    return area.transform(transform)
  }

  // 检查元素是否在可见区域内
  isElementVisible(element: Cell): boolean {
    if (!this.renderArea) return true

    const bbox = element.getBBox()
    return this.renderArea.isIntersectWithRect(bbox)
  }

  // 获取可见元素列表
  getVisibleElements(): Cell[] {
    if (!this.renderArea) return this.graph.model.getCells()

    return this.graph.model.getCells().filter(cell =>
      this.isElementVisible(cell)
    )
  }
}
```

#### 7. 性能优化策略
```typescript
// 节流处理，避免频繁计算
protected init() {
  this.resetRenderArea = FunctionExt.throttle(this.resetRenderArea, 200, {
    leading: true,
  })
  this.resetRenderArea()
  this.startListening()
}

// 批量更新可见元素
updateVisibleElements() {
  const visibleElements = this.getVisibleElements()
  const hiddenElements = this.graph.model.getCells().filter(
    cell => !visibleElements.includes(cell)
  )

  // 隐藏不可见元素
  hiddenElements.forEach(cell => {
    cell.setVisible(false, { silent: true })
  })

  // 显示可见元素
  visibleElements.forEach(cell => {
    cell.setVisible(true, { silent: true })
  })
}
```

### 2. 何时应当拆分更新为多个帧以避免掉帧？

**答案：** 当**单帧处理时间超过16.67ms**、**更新元素数量过多**、**复杂计算操作**或**DOM操作密集**时，应当拆分更新为多个帧。

**帧拆分策略：**

#### 1. 帧时间限制
```typescript
// src/renderer/queueJob.ts
export class JobQueue {
  private frameInterval = 16.67 // 60fps 的帧间隔

  flushJobs() {
    this.isFlushPending = false
    this.isFlushing = true

    const startTime = this.getCurrentTime()

    let job
    while ((job = this.queue.shift())) {
      job.cb()
      // 检查是否超过帧时间限制
      if (this.getCurrentTime() - startTime >= this.frameInterval) {
        break
      }
    }

    this.isFlushing = false

    // 如果还有未处理的任务，安排到下一帧
    if (this.queue.length) {
      this.queueFlush()
    }
  }
}
```

#### 2. 任务优先级管理
```typescript
// 任务优先级定义
export enum JOB_PRIORITY {
  Update = 1 << 1,        // 普通更新
  RenderEdge = 1 << 2,    // 边渲染
  RenderNode = 1 << 3,    // 节点渲染
  PRIOR = 1 << 20,        // 高优先级
}

// 根据优先级安排任务
requestViewUpdate(
  view: CellView,
  flag: number,
  options: any = {},
  priority: JOB_PRIORITY = JOB_PRIORITY.Update,
  flush = true,
) {
  const priorAction = view.hasAction(flag, ['translate', 'resize', 'rotate'])
  if (priorAction || options.async === false) {
    priority = JOB_PRIORITY.PRIOR
    flush = false
  }

  this.queue.queueJob({
    id: view.cell.id,
    priority,
    cb: () => {
      this.renderViewInArea(view, flag, options)
    },
  })
}
```

#### 3. 批量更新拆分
```typescript
// 大量元素更新时的帧拆分
export class BatchRenderer {
  private batchSize = 50 // 每帧处理的最大元素数量
  private frameTime = 16.67 // 帧时间限制

  // 批量更新元素
  batchUpdateElements(elements: Cell[], updateFn: (cell: Cell) => void) {
    const batches = this.splitIntoBatches(elements, this.batchSize)

    batches.forEach((batch, index) => {
      // 使用 requestAnimationFrame 安排到不同帧
      requestAnimationFrame(() => {
        this.updateBatch(batch, updateFn)
      })
    })
  }

  // 拆分批次
  private splitIntoBatches<T>(items: T[], batchSize: number): T[][] {
    const batches: T[][] = []
    for (let i = 0; i < items.length; i += batchSize) {
      batches.push(items.slice(i, i + batchSize))
    }
    return batches
  }

  // 更新单个批次
  private updateBatch(batch: Cell[], updateFn: (cell: Cell) => void) {
    const startTime = performance.now()

    for (const cell of batch) {
      updateFn(cell)

      // 检查是否超过帧时间限制
      if (performance.now() - startTime > this.frameTime) {
        // 将剩余元素安排到下一帧
        const remaining = batch.slice(batch.indexOf(cell) + 1)
        if (remaining.length > 0) {
          requestAnimationFrame(() => {
            this.updateBatch(remaining, updateFn)
          })
        }
        break
      }
    }
  }
}
```

#### 4. 复杂计算拆分
```typescript
// 复杂计算操作的帧拆分
export class ComplexCalculationRenderer {
  // 复杂布局计算
  calculateComplexLayout(nodes: Node[]): Promise<void> {
    return new Promise((resolve) => {
      const startTime = performance.now()
      const frameTime = 16.67

      let index = 0

      const processBatch = () => {
        const batchStartTime = performance.now()

        while (index < nodes.length) {
          const node = nodes[index]

          // 执行复杂计算
          this.calculateNodeLayout(node)
          index++

          // 检查是否超过帧时间限制
          if (performance.now() - batchStartTime > frameTime) {
            // 安排到下一帧继续处理
            requestAnimationFrame(processBatch)
            return
          }
        }

        // 所有节点处理完成
        resolve()
      }

      processBatch()
    })
  }

  // 单个节点的复杂计算
  private calculateNodeLayout(node: Node) {
    // 复杂的布局算法
    const position = this.forceDirectedLayout(node)
    const size = this.optimalSizeCalculation(node)
    const connections = this.connectionAnalysis(node)

    // 应用计算结果
    node.setPosition(position)
    node.setSize(size)
    node.setConnections(connections)
  }
}
```

#### 5. DOM操作拆分
```typescript
// DOM操作密集时的帧拆分
export class DOMOperationRenderer {
  // 大量DOM操作
  batchDOMOperations(operations: DOMOperation[]) {
    const batches = this.splitIntoBatches(operations, 20) // 每帧最多20个操作

    batches.forEach((batch, index) => {
      requestAnimationFrame(() => {
        this.executeDOMBatch(batch)
      })
    })
  }

  // 执行DOM批次
  private executeDOMBatch(operations: DOMOperation[]) {
    const startTime = performance.now()
    const frameTime = 16.67

    for (const operation of operations) {
      this.executeDOMOperation(operation)

      // 检查是否超过帧时间限制
      if (performance.now() - startTime > frameTime) {
        // 将剩余操作安排到下一帧
        const remaining = operations.slice(operations.indexOf(operation) + 1)
        if (remaining.length > 0) {
          requestAnimationFrame(() => {
            this.executeDOMBatch(remaining)
          })
        }
        break
      }
    }
  }

  // 执行单个DOM操作
  private executeDOMOperation(operation: DOMOperation) {
    switch (operation.type) {
      case 'create':
        this.createElement(operation.element)
        break
      case 'update':
        this.updateElement(operation.element, operation.attrs)
        break
      case 'remove':
        this.removeElement(operation.element)
        break
    }
  }
}
```

#### 6. 自适应帧拆分
```typescript
// 自适应帧拆分策略
export class AdaptiveFrameRenderer {
  private frameTime = 16.67
  private adaptiveBatchSize = 50

  // 自适应批量处理
  adaptiveBatchProcess<T>(
    items: T[],
    processFn: (item: T) => void,
    options: { maxBatchSize?: number; frameTime?: number } = {}
  ) {
    const maxBatchSize = options.maxBatchSize || this.adaptiveBatchSize
    const frameTime = options.frameTime || this.frameTime

    let index = 0

    const processBatch = () => {
      const startTime = performance.now()
      let processedCount = 0

      while (index < items.length && processedCount < maxBatchSize) {
        processFn(items[index])
        index++
        processedCount++

        // 检查是否超过帧时间限制
        if (performance.now() - startTime > frameTime) {
          break
        }
      }

      // 如果还有未处理的元素，安排到下一帧
      if (index < items.length) {
        requestAnimationFrame(processBatch)
      }
    }

    processBatch()
  }

  // 动态调整批次大小
  private adjustBatchSize(processingTime: number, currentBatchSize: number): number {
    if (processingTime < this.frameTime * 0.5) {
      // 处理时间较短，可以增加批次大小
      return Math.min(currentBatchSize * 1.2, 100)
    } else if (processingTime > this.frameTime * 0.8) {
      // 处理时间较长，需要减少批次大小
      return Math.max(currentBatchSize * 0.8, 10)
    }
    return currentBatchSize
  }
}
```

#### 7. 性能监控
```typescript
// 性能监控和帧拆分
export class PerformanceMonitor {
  private frameTime = 16.67
  private performanceHistory: number[] = []

  // 监控帧性能
  monitorFramePerformance(operation: () => void) {
    const startTime = performance.now()

    operation()

    const endTime = performance.now()
    const duration = endTime - startTime

    this.performanceHistory.push(duration)

    // 保持历史记录在合理范围内
    if (this.performanceHistory.length > 100) {
      this.performanceHistory.shift()
    }

    // 如果性能下降，建议拆分
    if (duration > this.frameTime) {
      console.warn(`Frame time exceeded: ${duration}ms`)
      return true // 需要拆分
    }

    return false
  }

  // 获取平均性能
  getAveragePerformance(): number {
    if (this.performanceHistory.length === 0) return 0

    const sum = this.performanceHistory.reduce((a, b) => a + b, 0)
    return sum / this.performanceHistory.length
  }

  // 建议是否拆分
  shouldSplit(): boolean {
    const avgPerformance = this.getAveragePerformance()
    return avgPerformance > this.frameTime * 0.8
  }
}
```

**总结：**

1. **可见区域识别：** 通过虚拟渲染机制，结合视口计算、滚动位置和变换矩阵来确定需要渲染的图形区域
2. **帧拆分时机：** 当单帧处理时间超过16.67ms、更新元素数量过多、复杂计算操作或DOM操作密集时
3. **拆分策略：** 使用任务优先级、批量更新、复杂计算拆分、DOM操作拆分和自适应帧拆分
4. **性能监控：** 通过性能监控来动态调整拆分策略，确保流畅的用户体验



