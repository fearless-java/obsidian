> 源代码路径: `packages/hooks/src/useWindowSize.ts`

# useWindowSize

## 1. 大白话讲讲这个 hook 的作用

`useWindowSize` *(一个React hook，用于获取浏览器窗口的宽度和高度，基于resize事件监听实现，返回响应式的尺寸对象)* 是一个获取"浏览器窗口大小"的 hook。它返回当前窗口的宽度和高度，当用户调整浏览器窗口大小时，它会自动更新。

```typescript
const { width, height } = useWindowSize()
// width: 1920, height: 1080 (桌面)
// width: 375, height: 812 (手机)
```

简单来说：它让你可以知道"用户的屏幕现在有多大"。

## 2. 讲讲为什么需要封装该 hook

### 原生实现的问题

```javascript
// 你可能会这样写
const [size, setSize] = useState({
  width: window.innerWidth, *(浏览器窗口内容区域的宽度)*
  height: window.innerHeight *(浏览器窗口内容区域的高度)*
})

useEffect(() => {
  const handler = () => setSize({
    width: window.innerWidth,
    height: window.innerHeight
  })
  window.addEventListener('resize', handler) *(resize事件，窗口大小变化时触发)*
  return () => window.removeEventListener('resize', handler)
}, [])

// 问题：
// 1. SSR 时 window 不存在
// 2. 需要手动定义 getSize 函数
// 3. 初始化逻辑和更新逻辑重复
```

### 需要处理好的细节

1. **SSR 安全**：服务端没有 window *(Server-Side Rendering，服务端渲染)*，需要返回 undefined
2. **初始化**：初始值要正确获取
3. **事件监听**：resize 时更新状态
4. **清理工作**：组件卸载时移除监听
5. **性能**：可能需要节流 *(性能优化技术，限制函数执行频率)*（这个 hook 没有做，看具体需求）

### 封装的好处

- **开箱即用**：不需要关心底层实现
- **SSR 安全**：服务端渲染不会报错
- **自动监听**：resize 时自动更新组件状态
- **返回格式统一**：{ width, height } 对象

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

无参数。

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| `{ width, height }` | `{ width: number \| undefined, height: number \| undefined }` | 窗口尺寸 *(SSR时为undefined)* |

### 执行逻辑

```
组件初始化:
    ↓
isClient = typeof window === 'object' *(判断是否为客户端环境)*
    ↓
getSize():
    ↓
isClient → window.innerWidth, window.innerHeight *(获取窗口尺寸)*
    ↓
not isClient → undefined, undefined *(SSR时返回undefined)*
    ↓
useState(getSize) → windowSize = { width, height } *(初始化state)*

组件挂载 useEffect:
    ↓
定义 handleResize *(resize事件处理器)*:
    ↓
setWindowSize(getSize()) *(获取最新尺寸并更新状态)*
    ↓
isClient？ → yes → 添加 resize 监听 → return 清理函数 *(添加resize事件监听)*
    ↓
no → return undefined *(SSR时不添加监听)*
    ↓
组件卸载: removeEventListener *(移除事件监听，防止内存泄漏)*
```

### 数据流

```
window.innerWidth / window.innerHeight *(浏览器原生窗口尺寸API)*
    ↓
getSize() → { width, height } *(封装成对象返回)*
    ↓
windowSize state *(React状态)*
    ↓
返回 { width, height } *(响应式返回，resize时自动更新)*

resize 事件:
    ↓
handleResize → setWindowSize(getSize()) *(事件触发时获取新尺寸)*
    ↓
组件重渲染
```

## 四、AI 提示词编写教学

你正在做一个 React 小工具，用来实时拿到浏览器窗口的大小。

先判断一下当前是不是在浏览器里运行，不是的话就别去碰窗口相关的东西，避免出错。

写一个简单方法，能拿到当前窗口的宽和高；如果不是浏览器环境，就先不返回具体数值。

把拿到的宽高存起来，一开始就先获取一次当前窗口大小。

当窗口大小被拖动改变时，自动重新获取最新的宽高并更新。

页面关掉或不用这个工具时，要自动把监听窗口变化的功能关掉，避免浪费资源。

最后对外只返回两个值：窗口宽度、窗口高度，在非浏览器环境下这两个值可以为空。

## 五、该 hook 的用法教学

### 基本用法

```typescript
import { useWindowSize } from './useWindowSize'

function WindowSizeDisplay() {
  const { width, height } = useWindowSize()

  return (
    <div>
      窗口宽度: {width}px
      窗口高度: {height}px
    </div>
  )
}
```

### 常见使用场景

🟩 **响应式网格布局**

```typescript
function ResponsiveGrid({ children }) {
  const { width } = useWindowSize()

  const getColumns = () => {
    if (!width) return 1
    if (width > 1200) return 4
    if (width > 900) return 3
    if (width > 600) return 2
    return 1
  }

  return (
    <div style={{
      display: 'grid',
      gridTemplateColumns: `repeat(${getColumns()}, 1fr)`,
      gap: '16px'
    }}>
      {children}
    </div>
  )
}
```

🟩 **动态全屏容器**

```typescript
function FullscreenContainer({ children }) {
  const { width, height } = useWindowSize()

  return (
    <div style={{
      width: `${width}px`,
      height: `${height}px`,
      overflow: 'hidden'
    }}>
      {children}
    </div>
  )
}
```

🟩 **条件渲染布局**

```typescript
function AdaptiveLayout() {
  const { width } = useWindowSize()
  const isMobile = width && width < 768

  return (
    <div>
      {isMobile ? <MobileNavigation /> : <DesktopNavigation />}
      <main>
        {isMobile ? <MobileListView /> : <DesktopTableView />}
      </main>
    </div>
  )
}
```

### Dos and Don'ts

✅ **Do: 处理 undefined 值**

```typescript
// 正确：SSR 时值为 undefined
function Component() {
  const { width } = useWindowSize()

  const columns = width && width > 1000 ? 4 : width && width > 600 ? 2 : 1
  // 或者使用默认值
  const safeWidth = width || 0

  return <div style={{ width: safeWidth }}>Content</div>
}
```

❌ **Don't: 假设 width 和 height 总是有值**

```typescript
// 错误：SSR 时会是 undefined
function Component() {
  const { width } = useWindowSize()

  // 错误：直接使用可能产生 NaN
  const halfWidth = width / 2

  // 正确：检查后再使用
  const halfWidth = width ? width / 2 : 0
}
```

✅ **Do: 结合 useCallback 优化 resize 处理**

```typescript
// 正确：避免每次渲染都创建新函数
const handleResize = useCallback(() => {
  setWindowSize(getSize())
}, [])

useEffect(() => {
  window.addEventListener('resize', handleResize)
  return () => window.removeEventListener('resize', handleResize)
}, [handleResize])
```

❌ **Don't: 在渲染时执行昂贵的计算**

```typescript
// 错误：每次渲染都计算
function Component() {
  const { width } = useWindowSize()

  // 错误：不要在这里做昂贵计算
  const expensiveValue = calculateExpensive(width)

  // 正确：用 useMemo 缓存
  const expensiveValue = useMemo(() => calculateExpensive(width), [width])
}
```

✅ **Do: 考虑使用 useDebounce 处理频繁 resize**

```typescript
// 正确：避免 resize 事件过于频繁
function DebouncedWindowSize() {
  const [windowSize, setWindowSize] = useState({ width: 0, height: 0 })
  const { width, height } = useWindowSize()
  const debouncedWidth = useDebounce(width, 200)
  const debouncedHeight = useDebounce(height, 200)

  useEffect(() => {
    setWindowSize({ width: debouncedWidth, height: debouncedHeight })
  }, [debouncedWidth, debouncedHeight])

  return windowSize
}
```

❌ **Don't: 同时使用多个窗口尺寸 hook**

```typescript
// 错误：不需要多次创建事件监听
function Component() {
  const size1 = useWindowSize() // 第一个监听器
  const size2 = useWindowSize() // 第二个监听器
  // 两个独立的事件监听器，不必要

  // 正确：共享同一个 hook 实例
  const { width, height } = useWindowSize()
```

✅ **Do: SSR 时返回安全的默认值**

```typescript
// 正确：SSR 时返回 undefined，调用方处理
function ServerComponent() {
  const { width } = useWindowSize()
  // SSR 时 width 是 undefined
  return <div>{width ? `宽度: ${width}` : '加载中'}</div>
}
```

❌ **Don't: 在 useEffect 外部使用 window.innerWidth**

```typescript
// 错误：SSR 时 window 不存在
const width = window.innerWidth // 服务端报错

// 正确：使用 hook
function Component() {
  const { width } = useWindowSize() // 安全，SSR 返回 undefined
}
```
