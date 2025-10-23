## 07 插件体系（Plugins）

目标：掌握插件生命周期、典型插件实现与二次开发范式。

### 1. 插件清单与能力

- 目录：`src/plugin/`
- 常用插件：
  - `Selection`：多选/框选/选择框；
  - `Transform`：节点缩放/旋转/拖拽锚点；
  - `Keyboard`：快捷键绑定；
  - `Clipboard`：复制/剪切/粘贴；
  - `Dnd`：拖拽创建；
  - `Scroller`：滚动视口、吸附、居中；
  - `MiniMap`：小地图；
  - `Snapline`：对齐线；
  - `History`：撤销/重做；
  - `Export`：导出图片/SVG。

### 2. 生命周期与形态

```ts
export type Graph.Plugin = {
  name: string
  init: (graph: Graph, ...options: any[]) => any
  dispose: () => void
  enable?: () => void
  disable?: () => void
  isEnabled?: () => boolean
}
```

接入：`graph.use(plugin, options)`；
控制：`enablePlugins/disablePlugins/isPluginEnabled/disposePlugins`。

### 3. 实战建议

- 不要重复造轮子，优先组合已有插件；
- 遵守 Graph 事件命名约定，统一从 Graph 订阅；
- 资源清理放在 `dispose`，启停幂等；
- 与注册表协作：插件内按需 `registerXxx`/`unregisterXxx`。

### 4. 检查点

- 实现一个“框选 + 复制”的复合插件，如何拆分职责？
- 为什么启用 Scroller 后，Graph 的缩放/居中等 API 被代理到 Scroller？







