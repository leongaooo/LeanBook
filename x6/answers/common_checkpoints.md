# X6 Common 层检查点答案

## 检查点问题解答

### 1. 你能解释 Basecoat 的职责与 dispose 的重要性吗？

**答案：** `Basecoat` 是X6框架的**基础类**，提供**事件系统**和**资源管理**能力，`dispose` 方法确保**资源正确释放**，防止**内存泄漏**。

**Basecoat 的职责：**

#### 1. 基础类定义
```typescript
// src/common/base/basecoat.ts
export class Basecoat<A extends EventArgs = any>
  extends Events<A>
  implements Disposable
{
  @disposable()
  dispose() {
    this.off() // 清理所有事件监听器
  }
}

export interface Basecoat extends Disposable {}
```

**核心职责：**
- **事件系统**：继承 `Events` 类，提供事件监听和触发能力
- **资源管理**：实现 `Disposable` 接口，支持资源释放
- **类型安全**：提供泛型支持，确保事件参数类型安全

#### 2. 事件系统职责
```typescript
// Basecoat 提供的事件能力
class MyComponent extends Basecoat<MyComponent.EventArgs> {
  constructor() {
    super()

    // 监听事件
    this.on('change', this.handleChange, this)
    this.on('destroy', this.handleDestroy, this)
  }

  // 触发事件
  notifyChange(data: any) {
    this.trigger('change', { data })
  }

  // 事件参数类型定义
  interface EventArgs {
    'change': { data: any }
    'destroy': { reason: string }
  }
}
```

#### 3. 资源管理职责
```typescript
// 资源管理示例
class ResourceManager extends Basecoat {
  private resources: Set<IDisposable> = new Set()
  private timers: Set<number> = new Set()

  // 添加资源
  addResource(resource: IDisposable) {
    this.resources.add(resource)
  }

  // 添加定时器
  addTimer(callback: () => void, delay: number) {
    const timer = setTimeout(callback, delay)
    this.timers.add(timer)
    return timer
  }

  // 资源释放
  @disposable()
  dispose() {
    // 释放所有资源
    this.resources.forEach(resource => resource.dispose())
    this.resources.clear()

    // 清理所有定时器
    this.timers.forEach(timer => clearTimeout(timer))
    this.timers.clear()

    // 清理事件监听器
    this.off()
  }
}
```

#### 4. dispose 的重要性

**A. 防止内存泄漏**
```typescript
// 内存泄漏示例
class BadComponent {
  private timer: number
  private listeners: Function[] = []

  constructor() {
    // 创建定时器
    this.timer = setInterval(() => {
      console.log('timer running')
    }, 1000)

    // 添加事件监听器
    document.addEventListener('click', this.handleClick)
    this.listeners.push(this.handleClick)
  }

  // 没有 dispose 方法，资源无法释放
  // 定时器会一直运行，事件监听器会一直存在
}

// 正确的资源管理
class GoodComponent extends Basecoat {
  private timer: number
  private listeners: Function[] = []

  constructor() {
    super()
    this.timer = setInterval(() => {
      console.log('timer running')
    }, 1000)

    document.addEventListener('click', this.handleClick)
    this.listeners.push(this.handleClick)
  }

  @disposable()
  dispose() {
    // 清理定时器
    if (this.timer) {
      clearInterval(this.timer)
      this.timer = 0
    }

    // 清理事件监听器
    this.listeners.forEach(listener => {
      document.removeEventListener('click', listener)
    })
    this.listeners = []

    // 清理内部事件
    this.off()
  }
}
```

**B. 确保资源正确释放**
```typescript
// 复杂组件的资源管理
class ComplexComponent extends Basecoat {
  private canvas: HTMLCanvasElement
  private ctx: CanvasRenderingContext2D
  private animationId: number
  private observers: MutationObserver[] = []

  constructor() {
    super()
    this.canvas = document.createElement('canvas')
    this.ctx = this.canvas.getContext('2d')!

    // 开始动画
    this.startAnimation()

    // 添加观察者
    this.addObservers()
  }

  private startAnimation() {
    const animate = () => {
      this.draw()
      this.animationId = requestAnimationFrame(animate)
    }
    animate()
  }

  private addObservers() {
    const observer = new MutationObserver(() => {
      this.handleMutation()
    })
    observer.observe(document.body, { childList: true })
    this.observers.push(observer)
  }

  @disposable()
  dispose() {
    // 停止动画
    if (this.animationId) {
      cancelAnimationFrame(this.animationId)
    }

    // 清理观察者
    this.observers.forEach(observer => observer.disconnect())
    this.observers = []

    // 清理画布
    this.canvas.remove()

    // 清理事件
    this.off()
  }
}
```

**C. 支持装饰器自动释放**
```typescript
// 使用 @disposable 装饰器
class AutoDisposeComponent extends Basecoat {
  private resource: IDisposable

  constructor() {
    super()
    this.resource = new DisposableDelegate(() => {
      console.log('resource disposed')
    })
  }

  // 方法执行后自动调用 dispose
  @disposable()
  destroy() {
    console.log('component destroyed')
    // 方法执行后会自动调用 this.dispose()
  }
}
```

#### 5. 最佳实践

**A. 资源管理模式**
```typescript
// 资源管理最佳实践
class BestPracticeComponent extends Basecoat {
  private resources = new DisposableSet()
  private timers = new Set<number>()
  private listeners = new Set<Function>()

  constructor() {
    super()
    this.setupResources()
  }

  private setupResources() {
    // 添加资源到集合
    const resource1 = new DisposableDelegate(() => {
      console.log('resource1 disposed')
    })
    this.resources.add(resource1)

    // 添加定时器
    const timer = setTimeout(() => {}, 1000)
    this.timers.add(timer)

    // 添加事件监听器
    const listener = () => {}
    document.addEventListener('click', listener)
    this.listeners.add(listener)
  }

  @disposable()
  dispose() {
    // 释放所有资源
    this.resources.dispose()

    // 清理定时器
    this.timers.forEach(timer => clearTimeout(timer))
    this.timers.clear()

    // 清理事件监听器
    this.listeners.forEach(listener => {
      document.removeEventListener('click', listener)
    })
    this.listeners.clear()

    // 清理内部事件
    this.off()
  }
}
```

### 2. 何时应下沉通用逻辑到 common 以便复用？

**答案：** 当逻辑在**多个模块中重复出现**、**功能相对独立**、**不依赖特定业务**且**具有通用性**时，应当下沉到 common 层。

**下沉通用逻辑的判断标准：**

#### 1. 重复性判断
```typescript
// 在多个模块中重复出现的逻辑
// 模块A
class ModuleA {
  private formatDate(date: Date): string {
    return date.toISOString().split('T')[0]
  }
}

// 模块B
class ModuleB {
  private formatDate(date: Date): string {
    return date.toISOString().split('T')[0]
  }
}

// 模块C
class ModuleC {
  private formatDate(date: Date): string {
    return date.toISOString().split('T')[0]
  }
}

// 应该下沉到 common
// src/common/date/index.ts
export function formatDate(date: Date): string {
  return date.toISOString().split('T')[0]
}
```

#### 2. 功能独立性判断
```typescript
// 功能独立的工具函数
// 应该下沉到 common
export class StringUtils {
  static camelCase(str: string): string {
    return str.replace(/-([a-z])/g, (_, letter) => letter.toUpperCase())
  }

  static kebabCase(str: string): string {
    return str.replace(/([A-Z])/g, '-$1').toLowerCase()
  }

  static capitalize(str: string): string {
    return str.charAt(0).toUpperCase() + str.slice(1)
  }
}

// 不应该下沉的业务逻辑
class BusinessLogic {
  // 依赖特定业务，不应该下沉
  calculateUserScore(user: User): number {
    return user.purchases * 0.1 + user.reviews * 0.05
  }
}
```

#### 3. 通用性判断
```typescript
// 通用性高的逻辑
// 应该下沉到 common
export class ArrayUtils {
  static unique<T>(array: T[]): T[] {
    return [...new Set(array)]
  }

  static groupBy<T, K>(array: T[], keyFn: (item: T) => K): Map<K, T[]> {
    const groups = new Map<K, T[]>()
    array.forEach(item => {
      const key = keyFn(item)
      if (!groups.has(key)) {
        groups.set(key, [])
      }
      groups.get(key)!.push(item)
    })
    return groups
  }
}

// 通用性低的逻辑
class SpecificLogic {
  // 只适用于特定场景，不应该下沉
  calculateGraphLayout(nodes: GraphNode[]): Layout {
    // 特定的图形布局算法
  }
}
```

#### 4. 依赖关系判断
```typescript
// 无外部依赖的逻辑
// 应该下沉到 common
export class MathUtils {
  static clamp(value: number, min: number, max: number): number {
    return Math.min(Math.max(value, min), max)
  }

  static lerp(a: number, b: number, t: number): number {
    return a + (b - a) * t
  }
}

// 有外部依赖的逻辑
class DependentLogic {
  // 依赖外部库，不应该下沉
  calculateHash(data: string): string {
    return crypto.subtle.digest('SHA-256', new TextEncoder().encode(data))
  }
}
```

#### 5. 实际下沉示例

**A. 函数工具类**
```typescript
// src/common/function/index.ts
export class FunctionExt {
  // 防抖
  static debounce<T extends (...args: any[]) => any>(
    func: T,
    wait: number,
    immediate = false
  ): T {
    let timeout: NodeJS.Timeout | null = null
    return ((...args: Parameters<T>) => {
      const later = () => {
        timeout = null
        if (!immediate) func(...args)
      }
      const callNow = immediate && !timeout
      if (timeout) clearTimeout(timeout)
      timeout = setTimeout(later, wait)
      if (callNow) func(...args)
    }) as T
  }

  // 节流
  static throttle<T extends (...args: any[]) => any>(
    func: T,
    wait: number
  ): T {
    let timeout: NodeJS.Timeout | null = null
    let previous = 0
    return ((...args: Parameters<T>) => {
      const now = Date.now()
      if (!previous) previous = now
      const remaining = wait - (now - previous)
      if (remaining <= 0 || remaining > wait) {
        if (timeout) {
          clearTimeout(timeout)
          timeout = null
        }
        previous = now
        func(...args)
      } else if (!timeout) {
        timeout = setTimeout(() => {
          previous = Date.now()
          timeout = null
          func(...args)
        }, remaining)
      }
    }) as T
  }
}
```

**B. DOM 工具类**
```typescript
// src/common/dom/index.ts
export class DomUtils {
  // 创建元素
  static createElement<T extends keyof HTMLElementTagNameMap>(
    tagName: T,
    attrs?: Record<string, any>,
    children?: (Node | string)[]
  ): HTMLElementTagNameMap[T] {
    const element = document.createElement(tagName)

    if (attrs) {
      Object.entries(attrs).forEach(([key, value]) => {
        if (key === 'style' && typeof value === 'object') {
          Object.assign(element.style, value)
        } else if (key === 'className') {
          element.className = value
        } else {
          element.setAttribute(key, value)
        }
      })
    }

    if (children) {
      children.forEach(child => {
        if (typeof child === 'string') {
          element.appendChild(document.createTextNode(child))
        } else {
          element.appendChild(child)
        }
      })
    }

    return element
  }

  // 查找元素
  static findElement(selector: string, parent: Element = document): Element | null {
    return parent.querySelector(selector)
  }

  // 添加事件监听器
  static addEventListener(
    element: Element,
    event: string,
    handler: EventListener,
    options?: AddEventListenerOptions
  ): () => void {
    element.addEventListener(event, handler, options)
    return () => element.removeEventListener(event, handler, options)
  }
}
```

**C. 对象工具类**
```typescript
// src/common/object/index.ts
export class ObjectUtils {
  // 深度合并
  static deepMerge<T extends Record<string, any>>(target: T, ...sources: Partial<T>[]): T {
    if (!sources.length) return target
    const source = sources.shift()

    if (this.isObject(target) && this.isObject(source)) {
      for (const key in source) {
        if (this.isObject(source[key])) {
          if (!target[key]) Object.assign(target, { [key]: {} })
          this.deepMerge(target[key], source[key])
        } else {
          Object.assign(target, { [key]: source[key] })
        }
      }
    }

    return this.deepMerge(target, ...sources)
  }

  // 检查是否为对象
  static isObject(item: any): boolean {
    return item && typeof item === 'object' && !Array.isArray(item)
  }

  // 获取嵌套属性值
  static getNestedValue(obj: any, path: string): any {
    return path.split('.').reduce((current, key) => current?.[key], obj)
  }

  // 设置嵌套属性值
  static setNestedValue(obj: any, path: string, value: any): void {
    const keys = path.split('.')
    const lastKey = keys.pop()!
    const target = keys.reduce((current, key) => {
      if (!current[key]) current[key] = {}
      return current[key]
    }, obj)
    target[lastKey] = value
  }
}
```

#### 6. 下沉策略

**A. 渐进式下沉**
```typescript
// 第一步：识别重复逻辑
class ComponentA {
  private formatData(data: any): string {
    return JSON.stringify(data, null, 2)
  }
}

class ComponentB {
  private formatData(data: any): string {
    return JSON.stringify(data, null, 2)
  }
}

// 第二步：提取到 common
// src/common/format/index.ts
export function formatData(data: any): string {
  return JSON.stringify(data, null, 2)
}

// 第三步：重构使用
class ComponentA {
  private formatData(data: any): string {
    return formatData(data)
  }
}
```

**B. 测试驱动下沉**
```typescript
// 先写测试
describe('Common Utils', () => {
  it('should format data correctly', () => {
    const data = { name: 'test', value: 123 }
    const result = formatData(data)
    expect(result).toBe('{\n  "name": "test",\n  "value": 123\n}')
  })
})

// 再实现功能
export function formatData(data: any): string {
  return JSON.stringify(data, null, 2)
}
```

**总结：**
- **Basecoat** 提供事件系统和资源管理能力
- **dispose** 确保资源正确释放，防止内存泄漏
- **通用逻辑下沉** 的判断标准：重复性、独立性、通用性、无外部依赖
- **下沉策略**：渐进式下沉、测试驱动、功能独立



