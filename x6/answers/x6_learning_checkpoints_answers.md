# X6 学习检查点和思考题答案

## 5. 学习检查点

### 5.1 你能否画出「Graph - Model - Renderer - View - Plugin - Registry」关系图？

**答案：** 已绘制架构关系图，详见 `x6_architecture_diagram.md`。

**核心关系：**
- **Graph（控制器）**：协调 Model 与 View，提供画布级 API
- **Model（数据层）**：管理节点、边、集合，提供查询、增删改、拓扑分析
- **Renderer（渲染层）**：组织调度视图的创建/更新/销毁，保障渲染效率
- **View（视图层）**：负责 Cell 的可视化呈现（DOM/SVG/HTML），事件绑定、命中检测
- **Plugin（插件）**：以插件形式扩展交互与编辑功能
- **Registry（注册表）**：统一可插拔要素的注册与查找

### 5.2 你能指出 Graph 的构造顺序以及为何这样安排？

**答案：** Graph 的构造顺序如下：

```typescript
constructor(options: Partial<GraphOptions.Manual>) {
  super()
  this.options = GraphOptions.get(options)
  this.css = new Css(this)                    // 1. CSS 管理器
  this.view = new GraphView(this)            // 2. 视图容器
  this.defs = new Defs(this)                 // 3. SVG 定义
  this.coord = new Coord(this)               // 4. 坐标管理器
  this.transform = new Transform(this)       // 5. 变换管理器
  this.highlight = new Highlight(this)       // 6. 高亮管理器
  this.grid = new Grid(this)                 // 7. 网格管理器
  this.background = new Background(this)     // 8. 背景管理器

  // 9. 数据模型（核心）
  if (this.options.model) {
    this.model = this.options.model
  } else {
    this.model = new Model()
    this.model.graph = this
  }

  // 10. 渲染器（依赖 Model）
  this.renderer = new ViewRenderer(this)
  this.panning = new Panning(this)           // 11. 平移管理器
  this.mousewheel = new Wheel(this)          // 12. 滚轮管理器
  this.virtualRender = new VirtualRender(this) // 13. 虚拟渲染
  this.size = new Size(this)                 // 14. 尺寸管理器
}
```

**构造顺序的原因：**

1. **基础能力优先**：CSS、View、Defs、Coord 等基础管理器先创建
2. **依赖关系**：Model 必须在 Renderer 之前创建，因为 Renderer 需要监听 Model 事件
3. **渲染相关**：Renderer 创建后，再创建交互相关的管理器（Panning、MouseWheel）
4. **性能优化**：VirtualRender 和 Size 管理器最后创建，用于性能优化

### 5.3 你能说明一次 graph.addNode() 到视图出现的完整链路吗？

**答案：** 完整链路如下：

```
1. graph.addNode(metadata, options)
   ↓
2. this.model.addNode(node, options)
   ↓
3. this.addCell(node, options)
   ↓
4. this.collection.add(this.prepareCell(cell, options), options)
   ↓
5. Collection.add() 触发事件：
   - this.trigger('added', args)
   - cell.notify('added', { ...args })
   ↓
6. Model 触发事件：
   - model.notify('cell:added', args)
   ↓
7. Scheduler 监听 Model 事件：
   - this.model.on('cell:added', this.onCellAdded, this)
   ↓
8. Scheduler.onCellAdded() 调用：
   - this.renderViews([cell], options)
   ↓
9. Scheduler.renderViews() 处理：
   - 创建 CellView：this.createCellView(cell)
   - 设置视图状态：viewItem.state = Scheduler.ViewState.CREATED
   - 请求视图更新：this.requestViewUpdate(view, flag, options)
   ↓
10. Scheduler.requestViewUpdate() 调度：
    - 加入任务队列：this.queue.queueJob()
    - 执行渲染：this.renderViewInArea(view, flag, options)
    ↓
11. CellView 渲染到 DOM：
    - view.render() 创建 DOM 元素
    - view.mount() 挂载到容器
    - 视图出现在画布上
```

**关键事件流：**
- **数据层**：Model → Collection → Cell 事件
- **调度层**：Scheduler 监听 Model 事件，调度视图更新
- **视图层**：CellView 创建、渲染、挂载到 DOM

## 6. 思考题

### 6.1 若要增加"对齐参考线"的新高阶功能，你会做成「插件」还是「注册项」？为什么？

**答案：** 应该做成**插件（Plugin）**。

**原因分析：**

**插件的特点：**
- 完整的生命周期：`init` → `enable/disable` → `dispose`
- 可以监听 Graph 事件，实现复杂的交互逻辑
- 可以管理自己的状态和资源
- 可以动态启用/禁用

**注册项的特点：**
- 主要用于可复用的算法或组件
- 如 router、connector、filter 等
- 相对简单，无状态或状态较少

**对齐参考线功能分析：**
1. **复杂交互逻辑**：需要监听节点拖拽事件，计算对齐位置
2. **状态管理**：需要管理参考线的显示/隐藏状态
3. **生命周期**：需要初始化、启用、禁用、清理资源
4. **事件监听**：需要监听 `node:moving`、`node:moved` 等事件
5. **DOM 操作**：需要创建、更新、销毁参考线元素

**实现建议：**
```typescript
export class AlignmentGuide implements Graph.Plugin {
  public name = 'alignment-guide'

  init(graph: Graph) {
    this.graph = graph
    this.startListening()
  }

  private startListening() {
    this.graph.on('node:moving', this.onNodeMoving, this)
    this.graph.on('node:moved', this.onNodeMoved, this)
  }

  private onNodeMoving({ node, e }: EventArgs['node:moving']) {
    // 计算对齐参考线
    this.showAlignmentGuides(node)
  }
}
```

### 6.2 如果要支持 WebGL 渲染，应在哪一层进行替换与抽象收敛？

**答案：** 应该在**View 层**进行替换与抽象收敛。

**分析：**

**当前架构：**
```
Graph → Renderer → Scheduler → CellView → DOM/SVG
```

**WebGL 替换方案：**
```
Graph → Renderer → Scheduler → WebGLCellView → WebGL Canvas
```

**具体实现策略：**

1. **抽象层设计**：
   ```typescript
   // 渲染接口抽象
   interface IRenderer {
     render(cell: Cell): void
     update(cell: Cell): void
     remove(cell: Cell): void
   }

   // DOM 渲染器
   class DOMRenderer implements IRenderer {
     // 当前实现
   }

   // WebGL 渲染器
   class WebGLRenderer implements IRenderer {
     // WebGL 实现
   }
   ```

2. **替换点**：
   - **CellView 层**：创建 `WebGLCellView` 替代 `CellView`
   - **渲染调度**：Scheduler 根据配置选择渲染器
   - **视图创建**：通过工厂模式创建对应的视图

3. **抽象收敛层**：
   ```typescript
   // 在 Scheduler 中
   protected createCellView(cell: Cell) {
     const rendererType = this.graph.options.renderer // 'dom' | 'webgl'
     if (rendererType === 'webgl') {
       return new WebGLCellView(cell)
     }
     return new CellView(cell)
   }
   ```

4. **配置方式**：
   ```typescript
   const graph = new Graph({
     container: '#container',
     renderer: 'webgl', // 或 'dom'
     // 其他配置...
   })
   ```

**为什么在 View 层：**
- **职责单一**：View 层负责具体的渲染实现
- **接口稳定**：Renderer 和 Scheduler 的接口不需要改变
- **扩展性好**：可以支持多种渲染后端（DOM、WebGL、Canvas2D）
- **性能优化**：WebGL 渲染可以充分利用 GPU 加速

**实现要点：**
1. 保持 CellView 的接口一致性
2. 抽象几何计算和变换逻辑
3. 统一事件系统和交互处理
4. 提供渲染器切换机制
