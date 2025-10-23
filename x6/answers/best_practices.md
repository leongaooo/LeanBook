# X6 最佳实践指南

## 最佳实践原则

### 1. 批量数据修改使用 model.batchUpdate()

**实践说明：** 当需要修改多个数据项时，使用 `model.batchUpdate()` 将多个操作包装为单个事务，提高性能并确保数据一致性。

**实现方式：**

#### A. 基础批量更新
```typescript
// 错误的做法 - 多次单独更新
graph.addNode({ id: 'node1', x: 100, y: 100 })
graph.addNode({ id: 'node2', x: 200, y: 200 })
graph.addEdge({ source: 'node1', target: 'node2' })

// 正确的做法 - 批量更新
graph.model.batchUpdate(() => {
  graph.addNode({ id: 'node1', x: 100, y: 100 })
  graph.addNode({ id: 'node2', x: 200, y: 200 })
  graph.addEdge({ source: 'node1', target: 'node2' })
})
```

#### B. 复杂批量操作
```typescript
// 复杂布局更新
graph.model.batchUpdate(() => {
  // 1. 更新节点位置
  nodes.forEach(node => {
    const newPosition = calculateLayout(node)
    node.setPosition(newPosition.x, newPosition.y)
  })

  // 2. 更新边路径
  edges.forEach(edge => {
    const newPath = calculatePath(edge)
    edge.setVertices(newPath)
  })

  // 3. 更新样式
  cells.forEach(cell => {
    cell.setAttrs({
      fill: getColorByType(cell.getType()),
      stroke: getBorderColor(cell.getStatus())
    })
  })
})
```

#### C. 带选项的批量更新
```typescript
// 带选项的批量更新
graph.model.batchUpdate('layout-update', () => {
  // 执行布局更新
  updateLayout()
}, {
  silent: false, // 触发事件
  dryrun: false // 实际执行
})
```

#### D. 性能优化示例
```typescript
// 大量数据导入
function importLargeDataset(data: any[]) {
  return graph.model.batchUpdate('import-data', () => {
    const nodes = data.filter(item => item.type === 'node')
    const edges = data.filter(item => item.type === 'edge')

    // 批量添加节点
    nodes.forEach(nodeData => {
      graph.addNode(nodeData)
    })

    // 批量添加边
    edges.forEach(edgeData => {
      graph.addEdge(edgeData)
    })
  })
}
```

### 2. 可见区域优先渲染，关注虚拟渲染开关

**实践说明：** 在大图场景下，优先渲染可见区域，使用虚拟渲染机制提高性能。

**实现方式：**

#### A. 启用虚拟渲染
```typescript
// 创建图时启用虚拟渲染
const graph = new Graph({
  container: document.getElementById('container'),
  virtual: true, // 启用虚拟渲染
  width: 800,
  height: 600
})

// 或者动态启用
graph.enableVirtualRender()
```

#### B. 可见区域管理
```typescript
// 自定义可见区域管理
class CustomVirtualRender {
  private graph: Graph
  private renderArea?: Rectangle

  constructor(graph: Graph) {
    this.graph = graph
    this.setupVirtualRender()
  }

  private setupVirtualRender() {
    // 监听视口变化
    this.graph.on('translate', this.updateRenderArea, this)
    this.graph.on('scale', this.updateRenderArea, this)
    this.graph.on('resize', this.updateRenderArea, this)

    // 监听滚动事件
    const scroller = this.graph.getPlugin('scroller')
    if (scroller) {
      scroller.on('pan:start', this.updateRenderArea, this)
      scroller.on('panning', this.updateRenderArea, this)
    }
  }

  private updateRenderArea() {
    if (this.graph.options.virtual) {
      const viewport = this.graph.getGraphArea()
      if (viewport) {
        // 添加缓冲边距
        const margin = 120
        const renderArea = viewport.clone()
        renderArea.inflate(margin, margin)

        // 设置渲染区域
        this.graph.renderer.setRenderArea(renderArea)
      }
    }
  }
}
```

#### C. 性能监控
```typescript
// 虚拟渲染性能监控
class VirtualRenderMonitor {
  private graph: Graph
  private performanceMetrics = {
    renderTime: 0,
    visibleCells: 0,
    totalCells: 0
  }

  constructor(graph: Graph) {
    this.graph = graph
    this.setupMonitoring()
  }

  private setupMonitoring() {
    this.graph.on('render:done', this.onRenderDone, this)
  }

  private onRenderDone() {
    const startTime = performance.now()

    // 计算可见单元格数量
    const visibleCells = this.graph.model.getCells().filter(cell => {
      const bbox = cell.getBBox()
      const renderArea = this.graph.renderer.getRenderArea()
      return renderArea ? renderArea.isIntersectWithRect(bbox) : true
    })

    const endTime = performance.now()

    this.performanceMetrics = {
      renderTime: endTime - startTime,
      visibleCells: visibleCells.length,
      totalCells: this.graph.model.getCells().length
    }

    // 性能警告
    if (this.performanceMetrics.renderTime > 16.67) {
      console.warn('Render time exceeded 16.67ms:', this.performanceMetrics)
    }
  }
}
```

### 3. 插件注意 dispose 与 enable/disable 幂等

**实践说明：** 插件需要正确实现 `dispose` 方法清理资源，`enable/disable` 方法需要幂等，确保多次调用不会产生副作用。

**实现方式：**

#### A. 正确的插件生命周期
```typescript
export class BestPracticePlugin implements Graph.Plugin {
  public name = 'best-practice-plugin'
  private graph: Graph
  private isEnabled = false
  private isDisposed = false
  private resources = new DisposableSet()

  constructor(private options: PluginOptions = {}) {}

  init(graph: Graph) {
    if (this.isDisposed) {
      throw new Error('Plugin has been disposed')
    }

    this.graph = graph
    this.setupResources()
  }

  enable() {
    if (this.isDisposed) {
      throw new Error('Plugin has been disposed')
    }

    if (this.isEnabled) {
      return // 幂等：已经启用，直接返回
    }

    this.isEnabled = true
    this.startListening()
    this.setupUI()
  }

  disable() {
    if (this.isDisposed) {
      return // 幂等：已经销毁，直接返回
    }

    if (!this.isEnabled) {
      return // 幂等：已经禁用，直接返回
    }

    this.isEnabled = false
    this.stopListening()
    this.cleanupUI()
  }

  @disposable()
  dispose() {
    if (this.isDisposed) {
      return // 幂等：已经销毁，直接返回
    }

    this.isDisposed = true

    // 先禁用
    this.disable()

    // 清理资源
    this.resources.dispose()

    // 清理事件监听器
    this.graph.off(null, null, this)
  }

  private setupResources() {
    // 添加资源到集合
    const resource1 = new DisposableDelegate(() => {
      console.log('Resource 1 disposed')
    })
    this.resources.add(resource1)

    const resource2 = new DisposableDelegate(() => {
      console.log('Resource 2 disposed')
    })
    this.resources.add(resource2)
  }

  private startListening() {
    this.graph.on('node:click', this.onNodeClick, this)
    this.graph.on('edge:click', this.onEdgeClick, this)
  }

  private stopListening() {
    this.graph.off('node:click', this.onNodeClick, this)
    this.graph.off('edge:click', this.onEdgeClick, this)
  }

  private setupUI() {
    // 创建UI元素
    const button = document.createElement('button')
    button.textContent = 'Plugin Action'
    button.onclick = this.onButtonClick.bind(this)
    document.body.appendChild(button)

    // 添加到资源集合
    this.resources.add(new DisposableDelegate(() => {
      button.remove()
    }))
  }

  private cleanupUI() {
    // UI清理逻辑
  }

  private onNodeClick = (args: any) => {
    if (!this.isEnabled) return
    console.log('Node clicked:', args)
  }

  private onEdgeClick = (args: any) => {
    if (!this.isEnabled) return
    console.log('Edge clicked:', args)
  }

  private onButtonClick = () => {
    if (!this.isEnabled) return
    console.log('Button clicked')
  }
}
```

#### B. 插件状态管理
```typescript
// 插件状态管理
export class PluginStateManager {
  private plugins = new Map<string, PluginState>()

  registerPlugin(plugin: Graph.Plugin) {
    const state: PluginState = {
      plugin,
      isEnabled: false,
      isDisposed: false,
      resources: new DisposableSet()
    }
    this.plugins.set(plugin.name, state)
  }

  enablePlugin(name: string) {
    const state = this.plugins.get(name)
    if (!state || state.isDisposed) {
      throw new Error(`Plugin ${name} not found or disposed`)
    }

    if (state.isEnabled) {
      return // 幂等
    }

    state.isEnabled = true
    state.plugin.enable?.()
  }

  disablePlugin(name: string) {
    const state = this.plugins.get(name)
    if (!state || state.isDisposed) {
      return // 幂等
    }

    if (!state.isEnabled) {
      return // 幂等
    }

    state.isEnabled = false
    state.plugin.disable?.()
  }

  disposePlugin(name: string) {
    const state = this.plugins.get(name)
    if (!state || state.isDisposed) {
      return // 幂等
    }

    state.isDisposed = true
    state.plugin.dispose()
    state.resources.dispose()
    this.plugins.delete(name)
  }
}
```

### 4. 注册项命名加业务前缀，避免冲突

**实践说明：** 注册项（路由、连接器、锚点等）命名时添加业务前缀，避免与内置注册项冲突。

**实现方式：**

#### A. 路由注册
```typescript
// 错误的做法 - 可能冲突
Graph.registerRouter('custom', customRouter)

// 正确的做法 - 添加业务前缀
Graph.registerRouter('myapp-custom', customRouter)
Graph.registerRouter('myapp-smart', smartRouter)
Graph.registerRouter('myapp-orthogonal', orthogonalRouter)
```

#### B. 连接器注册
```typescript
// 业务前缀命名
Graph.registerConnector('myapp-smooth', smoothConnector)
Graph.registerConnector('myapp-straight', straightConnector)
Graph.registerConnector('myapp-curved', curvedConnector)
```

#### C. 锚点注册
```typescript
// 锚点注册
Graph.registerAnchor('myapp-center', centerAnchor)
Graph.registerAnchor('myapp-perimeter', perimeterAnchor)
Graph.registerAnchor('myapp-orthogonal', orthogonalAnchor)
```

#### D. 属性注册
```typescript
// 属性注册
Graph.registerAttr('myapp-gradient', gradientAttr)
Graph.registerAttr('myapp-shadow', shadowAttr)
Graph.registerAttr('myapp-border', borderAttr)
```

#### E. 命名规范
```typescript
// 命名规范
const NAMING_CONVENTIONS = {
  // 格式：{业务前缀}-{功能描述}
  router: 'myapp-{function}',
  connector: 'myapp-{function}',
  anchor: 'myapp-{function}',
  attr: 'myapp-{function}',
  filter: 'myapp-{function}',
  grid: 'myapp-{function}',
  marker: 'myapp-{function}',
  tool: 'myapp-{function}'
}

// 实际使用
const BUSINESS_PREFIX = 'myapp'
const FUNCTION_NAME = 'custom'

const routerName = `${BUSINESS_PREFIX}-${FUNCTION_NAME}`
Graph.registerRouter(routerName, customRouter)
```

### 5. 变更需附带测试与文档片段

**实践说明：** 任何代码变更都需要附带相应的测试用例和文档片段，确保代码质量和可维护性。

**实现方式：**

#### A. 测试用例
```typescript
// 测试用例示例
describe('CustomRouter', () => {
  let graph: Graph

  beforeEach(() => {
    graph = new Graph({
      container: document.createElement('div'),
      width: 800,
      height: 600
    })
  })

  afterEach(() => {
    graph.dispose()
  })

  it('should register custom router', () => {
    // 注册自定义路由
    Graph.registerRouter('myapp-custom', customRouter)

    // 创建边并测试路由
    const edge = graph.addEdge({
      source: { x: 100, y: 100 },
      target: { x: 300, y: 300 },
      router: {
        name: 'myapp-custom',
        args: { step: 20 }
      }
    })

    // 验证路由效果
    const vertices = edge.getVertices()
    expect(vertices).toHaveLength(2)
    expect(vertices[0]).toEqual({ x: 100, y: 100 })
    expect(vertices[1]).toEqual({ x: 300, y: 300 })
  })

  it('should handle edge cases', () => {
    // 测试边界情况
    const edge = graph.addEdge({
      source: { x: 0, y: 0 },
      target: { x: 0, y: 0 },
      router: {
        name: 'myapp-custom',
        args: { step: 0 }
      }
    })

    expect(edge.getVertices()).toBeDefined()
  })
})
```

#### B. 文档片段
```typescript
/**
 * 自定义路由器
 *
 * @example
 * ```typescript
 * // 注册自定义路由
 * Graph.registerRouter('myapp-custom', customRouter)
 *
 * // 使用自定义路由
 * const edge = graph.addEdge({
 *   source: { x: 100, y: 100 },
 *   target: { x: 300, y: 300 },
 *   router: {
 *     name: 'myapp-custom',
 *     args: { step: 20 }
 *   }
 * })
 * ```
 *
 * @param vertices - 路径点数组
 * @param options - 路由选项
 * @param edgeView - 边视图实例
 * @returns 新的路径点数组
 */
export const customRouter: RouterDefinition<CustomRouterOptions> = function(
  vertices: Point.PointLike[],
  options: CustomRouterOptions = {},
  edgeView: EdgeView
) {
  // 实现逻辑
  return vertices
}
```

#### C. 变更记录
```typescript
// 变更记录
const CHANGELOG = {
  version: '1.0.0',
  changes: [
    {
      type: 'feature',
      description: '添加自定义路由支持',
      files: ['src/router/custom.ts'],
      tests: ['src/router/__tests__/custom.test.ts'],
      docs: ['docs/router/custom.md']
    },
    {
      type: 'fix',
      description: '修复路由计算错误',
      files: ['src/router/calculator.ts'],
      tests: ['src/router/__tests__/calculator.test.ts'],
      docs: ['docs/router/calculator.md']
    }
  ]
}
```

### 6. 先改用例/示例驱动，再沉到核心代码

**实践说明：** 开发新功能时，先编写用例和示例，验证功能正确性，再将其下沉到核心代码中。

**实现方式：**

#### A. 用例驱动开发
```typescript
// 第一步：编写用例
describe('NewFeature', () => {
  it('should work as expected', () => {
    // 用例描述
    const graph = new Graph({
      container: document.createElement('div'),
      width: 800,
      height: 600
    })

    // 使用新功能
    const result = graph.newFeature({
      option1: 'value1',
      option2: 'value2'
    })

    // 验证结果
    expect(result).toBeDefined()
    expect(result.success).toBe(true)
  })
})
```

#### B. 示例驱动开发
```typescript
// 第二步：编写示例
const example = {
  title: '新功能示例',
  description: '展示如何使用新功能',
  code: `
    // 创建图实例
    const graph = new Graph({
      container: document.getElementById('container'),
      width: 800,
      height: 600
    })

    // 使用新功能
    const result = graph.newFeature({
      option1: 'value1',
      option2: 'value2'
    })

    // 处理结果
    if (result.success) {
      console.log('功能执行成功')
    } else {
      console.error('功能执行失败:', result.error)
    }
  `,
  expected: '功能执行成功'
}
```

#### C. 核心代码实现
```typescript
// 第三步：实现核心代码
export class Graph {
  // 新功能实现
  newFeature(options: NewFeatureOptions): NewFeatureResult {
    // 参数验证
    if (!options.option1) {
      throw new Error('option1 is required')
    }

    // 功能实现
    try {
      const result = this.executeNewFeature(options)
      return {
        success: true,
        data: result
      }
    } catch (error) {
      return {
        success: false,
        error: error.message
      }
    }
  }

  private executeNewFeature(options: NewFeatureOptions): any {
    // 具体实现逻辑
    return {
      processed: true,
      timestamp: Date.now()
    }
  }
}
```

#### D. 测试覆盖
```typescript
// 第四步：完善测试覆盖
describe('NewFeature', () => {
  let graph: Graph

  beforeEach(() => {
    graph = new Graph({
      container: document.createElement('div'),
      width: 800,
      height: 600
    })
  })

  afterEach(() => {
    graph.dispose()
  })

  it('should work with valid options', () => {
    const result = graph.newFeature({
      option1: 'value1',
      option2: 'value2'
    })

    expect(result.success).toBe(true)
    expect(result.data).toBeDefined()
  })

  it('should handle invalid options', () => {
    expect(() => {
      graph.newFeature({
        option1: '',
        option2: 'value2'
      })
    }).toThrow('option1 is required')
  })

  it('should handle errors gracefully', () => {
    // 模拟错误情况
    const result = graph.newFeature({
      option1: 'value1',
      option2: 'error'
    })

    expect(result.success).toBe(false)
    expect(result.error).toBeDefined()
  })
})
```

## 总结

这些最佳实践确保了X6框架的：

1. **性能优化**：批量更新、虚拟渲染
2. **资源管理**：正确的插件生命周期
3. **命名规范**：避免冲突的注册项命名
4. **代码质量**：测试覆盖、文档完整
5. **开发流程**：用例驱动、示例验证

遵循这些实践可以显著提高代码质量、可维护性和开发效率。



