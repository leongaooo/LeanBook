# X6 Shape 层检查点答案

## 检查点问题解答

### 1. 何时需要自定义 markup 而不是仅用 attrs？

**答案：** 当需要**复杂的DOM结构**、**多层级元素**、**特殊命名空间**或**动态内容**时，需要使用自定义 markup 而不是仅用 attrs。

**Markup vs Attrs 对比：**

#### 1. Attrs 的局限性
```typescript
// 仅使用 attrs 的简单形状
export const Rect = createShape('rect', {
  attrs: {
    body: {
      refWidth: '100%',
      refHeight: '100%',
      fill: '#f5f5f5',
      stroke: '#333',
    },
  },
})
```

**Attrs 适用场景：**
- 简单的单一元素形状
- 只需要修改现有元素的属性
- 标准的 SVG 元素（rect、circle、path 等）

#### 2. 需要自定义 Markup 的场景

**A. 复杂DOM结构**
```typescript
// 需要多个嵌套元素
export const ComplexNode = Base.define({
  shape: 'complex-node',
  markup: [
    // 外层容器
    {
      tagName: 'g',
      selector: 'container',
      children: [
        // 背景
        {
          tagName: 'rect',
          selector: 'background',
        },
        // 图标
        {
          tagName: 'circle',
          selector: 'icon',
        },
        // 文本标签
        {
          tagName: 'text',
          selector: 'label',
        },
        // 状态指示器
        {
          tagName: 'rect',
          selector: 'status',
        },
      ],
    },
  ],
  attrs: {
    background: {
      refWidth: '100%',
      refHeight: '100%',
      fill: '#fff',
      stroke: '#ddd',
    },
    icon: {
      refCx: 20,
      refCy: 20,
      refR: 8,
      fill: '#1890ff',
    },
    label: {
      refX: 40,
      refY: 20,
      textAnchor: 'start',
      dominantBaseline: 'middle',
    },
    status: {
      refX: '100%',
      refY: 0,
      refWidth: 8,
      refHeight: 8,
      fill: '#52c41a',
    },
  },
})
```

**B. 混合命名空间**
```typescript
// HTML 和 SVG 混合使用
export const HybridNode = Base.define({
  shape: 'hybrid-node',
  markup: [
    // SVG 背景
    {
      tagName: 'rect',
      selector: 'background',
    },
    // HTML 内容区域
    {
      tagName: 'foreignObject',
      selector: 'html-content',
      children: [
        {
          tagName: 'div',
          ns: Dom.ns.xhtml,
          selector: 'content',
          style: {
            width: '100%',
            height: '100%',
            padding: '8px',
            boxSizing: 'border-box',
          },
        },
      ],
    },
    // SVG 装饰元素
    {
      tagName: 'path',
      selector: 'decoration',
    },
  ],
  attrs: {
    background: {
      refWidth: '100%',
      refHeight: '100%',
      fill: '#f0f0f0',
    },
    'html-content': {
      refWidth: '100%',
      refHeight: '100%',
    },
    decoration: {
      d: 'M0,0 L20,0 L20,20 L0,20 Z',
      fill: 'none',
      stroke: '#1890ff',
    },
  },
})
```

**C. 动态内容结构**
```typescript
// 根据数据动态生成结构
export const DynamicNode = Base.define({
  shape: 'dynamic-node',
  markup: [
    {
      tagName: 'g',
      selector: 'container',
      children: [
        // 基础结构
        {
          tagName: 'rect',
          selector: 'body',
        },
        // 动态端口容器
        {
          tagName: 'g',
          selector: 'ports',
        },
        // 动态标签容器
        {
          tagName: 'g',
          selector: 'labels',
        },
      ],
    },
  ],
  // 动态生成子元素
  propHooks(metadata) {
    const { ports, labels, ...others } = metadata

    // 动态添加端口 markup
    if (ports && Array.isArray(ports)) {
      const portMarkup = ports.map((port, index) => ({
        tagName: 'circle',
        selector: `port-${port.id || index}`,
        attrs: {
          refCx: port.x,
          refCy: port.y,
          refR: port.radius || 4,
        },
      }))

      // 合并到 markup 中
      ObjectExt.setByPath(others, 'markup/0/children', [
        ...others.markup[0].children,
        ...portMarkup,
      ])
    }

    return others
  },
})
```

**D. 特殊功能需求**
```typescript
// 需要特殊功能的节点
export const InteractiveNode = Base.define({
  shape: 'interactive-node',
  markup: [
    // 主容器
    {
      tagName: 'g',
      selector: 'container',
      children: [
        // 可拖拽区域
        {
          tagName: 'rect',
          selector: 'drag-area',
          attrs: {
            fill: 'transparent',
            cursor: 'move',
          },
        },
        // 可调整大小的句柄
        {
          tagName: 'rect',
          selector: 'resize-handle',
          attrs: {
            fill: '#1890ff',
            cursor: 'nw-resize',
          },
        },
        // 可点击的按钮
        {
          tagName: 'circle',
          selector: 'action-button',
          attrs: {
            fill: '#52c41a',
            cursor: 'pointer',
          },
        },
        // 文本内容
        {
          tagName: 'text',
          selector: 'label',
        },
      ],
    },
  ],
  attrs: {
    'drag-area': {
      refWidth: '100%',
      refHeight: '100%',
    },
    'resize-handle': {
      refX: '100%',
      refY: '100%',
      refWidth: 8,
      refHeight: 8,
      refX: -8,
      refY: -8,
    },
    'action-button': {
      refCx: '100%',
      refCy: 0,
      refR: 6,
      refCx: -6,
    },
    label: {
      refX: '50%',
      refY: '50%',
      textAnchor: 'middle',
      dominantBaseline: 'middle',
    },
  },
})
```

#### 3. 自定义 Markup 的优势

**A. 结构清晰**
```typescript
// 清晰的层次结构
markup: [
  {
    tagName: 'g',
    selector: 'container',
    children: [
      { tagName: 'rect', selector: 'background' },
      { tagName: 'text', selector: 'label' },
      { tagName: 'circle', selector: 'icon' },
    ],
  },
]
```

**B. 选择器管理**
```typescript
// 通过选择器精确控制
attrs: {
  background: { fill: '#fff' },
  label: { fontSize: 14 },
  icon: { fill: '#1890ff' },
}
```

**C. 动态扩展**
```typescript
// 可以动态添加/移除元素
propHooks(metadata) {
  // 根据数据动态生成 markup
  return others
}
```

#### 4. 使用建议

**使用 Attrs 的场景：**
- 简单的几何形状
- 只需要修改现有元素属性
- 标准的 SVG 元素

**使用自定义 Markup 的场景：**
- 需要多个嵌套元素
- 混合 HTML 和 SVG
- 需要动态内容结构
- 复杂的交互功能
- 特殊的命名空间需求

### 2. 文本/HTML 节点与 SVG 节点混用的取舍是什么？

**答案：** 混用需要权衡**功能丰富性**与**性能/兼容性**，HTML 节点提供丰富的交互能力，SVG 节点提供更好的性能和兼容性。

**对比分析：**

#### 1. HTML 节点特点

**优势：**
```typescript
// HTML 节点 - 丰富的交互能力
export const HTMLNode = Base.define({
  shape: 'html-node',
  markup: [
    {
      tagName: 'foreignObject',
      selector: 'fo',
      children: [
        {
          tagName: 'div',
          ns: Dom.ns.xhtml,
          selector: 'content',
          style: {
            width: '100%',
            height: '100%',
            padding: '8px',
            boxSizing: 'border-box',
          },
        },
      ],
    },
  ],
  attrs: {
    fo: {
      refWidth: '100%',
      refHeight: '100%',
    },
  },
})

// 使用 HTML 节点
const htmlNode = graph.addNode({
  shape: 'html-node',
  x: 100,
  y: 100,
  width: 200,
  height: 100,
  html: `
    <div style="display: flex; flex-direction: column; height: 100%;">
      <input type="text" placeholder="输入内容" style="margin-bottom: 8px;">
      <button onclick="alert('点击了按钮')">点击我</button>
      <div style="flex: 1; background: #f0f0f0; padding: 4px;">
        这是可编辑的内容区域
      </div>
    </div>
  `,
})
```

**HTML 节点优势：**
- **丰富的交互**：支持表单元素、事件处理
- **CSS 支持**：完整的 CSS 样式支持
- **DOM 操作**：可以直接操作 DOM 元素
- **第三方库**：可以集成 React、Vue 等框架

**HTML 节点劣势：**
- **性能开销**：需要额外的 DOM 操作
- **兼容性问题**：foreignObject 支持有限
- **渲染差异**：不同浏览器渲染效果可能不同
- **事件冒泡**：需要特殊处理事件传播

#### 2. SVG 节点特点

**优势：**
```typescript
// SVG 节点 - 高性能和兼容性
export const SVGNode = Base.define({
  shape: 'svg-node',
  markup: [
    {
      tagName: 'rect',
      selector: 'background',
    },
    {
      tagName: 'text',
      selector: 'label',
    },
    {
      tagName: 'circle',
      selector: 'icon',
    },
  ],
  attrs: {
    background: {
      refWidth: '100%',
      refHeight: '100%',
      fill: '#fff',
      stroke: '#ddd',
    },
    label: {
      refX: '50%',
      refY: '50%',
      textAnchor: 'middle',
      dominantBaseline: 'middle',
      fontSize: 14,
    },
    icon: {
      refCx: 20,
      refCy: 20,
      refR: 8,
      fill: '#1890ff',
    },
  },
})
```

**SVG 节点优势：**
- **高性能**：原生 SVG 渲染，性能更好
- **兼容性好**：所有现代浏览器都支持
- **矢量图形**：缩放不失真
- **事件处理**：统一的事件系统
- **内存占用小**：不需要额外的 DOM 节点

**SVG 节点劣势：**
- **交互能力有限**：不支持复杂的表单元素
- **样式限制**：CSS 支持有限
- **文本处理**：文本换行、对齐等处理复杂
- **第三方集成**：难以集成复杂的 UI 库

#### 3. 混用策略

**A. 按功能分层**
```typescript
// 基础结构用 SVG，交互内容用 HTML
export const HybridNode = Base.define({
  shape: 'hybrid-node',
  markup: [
    // SVG 基础结构
    {
      tagName: 'g',
      selector: 'container',
      children: [
        {
          tagName: 'rect',
          selector: 'background',
        },
        {
          tagName: 'path',
          selector: 'border',
        },
      ],
    },
    // HTML 交互内容
    {
      tagName: 'foreignObject',
      selector: 'content',
      children: [
        {
          tagName: 'div',
          ns: Dom.ns.xhtml,
          selector: 'html-content',
        },
      ],
    },
  ],
  attrs: {
    background: {
      refWidth: '100%',
      refHeight: '100%',
      fill: '#f0f0f0',
    },
    border: {
      d: 'M0,0 L100%,0 L100%,100% L0,100% Z',
      fill: 'none',
      stroke: '#1890ff',
      strokeWidth: 2,
    },
    content: {
      refWidth: '100%',
      refHeight: '100%',
    },
  },
})
```

**B. 按性能需求选择**
```typescript
// 性能敏感的场景使用 SVG
export const PerformanceNode = Base.define({
  shape: 'performance-node',
  markup: [
    {
      tagName: 'rect',
      selector: 'body',
    },
    {
      tagName: 'text',
      selector: 'label',
    },
  ],
  // 使用 SVG 原生属性，避免 DOM 操作
  attrs: {
    body: {
      refWidth: '100%',
      refHeight: '100%',
      fill: '#fff',
      stroke: '#ddd',
    },
    label: {
      refX: '50%',
      refY: '50%',
      textAnchor: 'middle',
      dominantBaseline: 'middle',
    },
  },
})

// 交互丰富的场景使用 HTML
export const InteractiveNode = Base.define({
  shape: 'interactive-node',
  markup: [
    {
      tagName: 'foreignObject',
      selector: 'fo',
      children: [
        {
          tagName: 'div',
          ns: Dom.ns.xhtml,
          selector: 'content',
        },
      ],
    },
  ],
  attrs: {
    fo: {
      refWidth: '100%',
      refHeight: '100%',
    },
  },
})
```

**C. 按兼容性需求选择**
```typescript
// 检测浏览器支持情况
export function createAdaptiveNode() {
  const supportForeignObject = Platform.SUPPORT_FOREIGNOBJECT

  if (supportForeignObject) {
    // 支持 foreignObject，使用 HTML
    return Base.define({
      shape: 'adaptive-node',
      markup: [
        {
          tagName: 'foreignObject',
          selector: 'fo',
          children: [
            {
              tagName: 'div',
              ns: Dom.ns.xhtml,
              selector: 'content',
            },
          ],
        },
      ],
    })
  } else {
    // 不支持 foreignObject，使用 SVG
    return Base.define({
      shape: 'adaptive-node',
      markup: [
        {
          tagName: 'rect',
          selector: 'background',
        },
        {
          tagName: 'text',
          selector: 'label',
        },
      ],
    })
  }
}
```

#### 4. 最佳实践

**A. 性能优先场景**
```typescript
// 大量节点，性能优先
const performanceGraph = new Graph({
  container: document.getElementById('container'),
  // 使用 SVG 节点
  defaultNode: {
    shape: 'rect',
    attrs: {
      body: {
        fill: '#fff',
        stroke: '#ddd',
      },
    },
  },
})
```

**B. 交互优先场景**
```typescript
// 复杂交互，功能优先
const interactiveGraph = new Graph({
  container: document.getElementById('container'),
  // 使用 HTML 节点
  defaultNode: {
    shape: 'html',
    html: `
      <div style="padding: 8px;">
        <input type="text" placeholder="输入内容">
        <button>操作</button>
      </div>
    `,
  },
})
```

**C. 混合使用场景**
```typescript
// 根据节点类型选择
const hybridGraph = new Graph({
  container: document.getElementById('container'),
  // 基础节点用 SVG
  defaultNode: {
    shape: 'rect',
  },
  // 特殊节点用 HTML
  nodes: [
    {
      id: '1',
      shape: 'rect',
      x: 100,
      y: 100,
    },
    {
      id: '2',
      shape: 'html',
      x: 200,
      y: 100,
      html: '<div>HTML 内容</div>',
    },
  ],
})
```

**总结：**
- **SVG 节点**：适合性能敏感、大量节点、简单交互的场景
- **HTML 节点**：适合复杂交互、丰富内容、第三方集成的场景
- **混用策略**：根据具体需求选择，可以按功能分层或按性能需求选择
- **兼容性考虑**：检测浏览器支持情况，提供降级方案



