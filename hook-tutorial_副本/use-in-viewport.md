> 源代码路径: `packages/hooks/src/useInViewport.ts`

# useInViewport

## 1. 大白话讲讲这个 hook 的作用

`useInViewport` *(一个React hook，用于检测DOM元素是否进入可视区域，基于IntersectionObserver API实现，常用于懒加载和无限滚动)* 是一个检测"某个元素是否在屏幕可视区域内"的 hook。它使用浏览器原生的 `IntersectionObserver` *(浏览器API，用于观察元素是否进入/离开视口，比scroll事件更高效)* API 来观察元素。

简单来说：
- 当元素进入屏幕视野 → 返回 `true`
- 当元素离开屏幕视野 → 返回 `false`
- 一旦元素曾经进入过视野，就永久返回 `true`（`isLoaded` 标志）

常用于：
- 懒加载图片（图片进入视野才加载）
- 无限滚动（列表底部进入视野时加载更多）
- 动画触发（元素进入视野时播放动画）

## 2. 讲讲为什么需要封装该 hook

### 原生 IntersectionObserver 的问题
```javascript
// 你需要自己写这样的逻辑
const observer = new IntersectionObserver(([entry]) => {
  console.log(entry.isIntersecting)
})
observer.observe(ref.current)
return () => observer.disconnect()
```

### 需要处理好的细节
1. **ref 处理**：需要访问 DOM 元素的 ref *(React中用于获取DOM元素的引用)*
2. **生命周期**：组件卸载时要断开 observer
3. **状态初始化**：初始状态是 false
4. **isLoaded 逻辑**：一旦进入过视野就保持 true
5. **SSR 安全**：服务端没有 IntersectionObserver

### 封装的好处
- **API 简洁**：传入 ref，返回 boolean
- **自动管理生命周期**：自动断开 observer
- **开箱即用**：不需要了解 IntersectionObserver API
- **isLoaded 逻辑**：内置"曾经进入过"的状态

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

| 参数 | 类型 | 说明 |
|------|------|------|
| `ref` | `any` | DOM 元素的 ref *(React的ref，用于获取DOM节点)* |

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| `boolean` | `true/false` | 元素是否在可视区域内（或曾经进入过） |

### 执行逻辑

```
组件挂载:
    ↓
创建 IntersectionObserver *(浏览器原生API，用于检测元素可见性)*，回调函数:
    ↓
entry.isIntersecting === true *(元素进入视口时为true)*
    ↓
setIsLoaded(true) *(标记为"曾经进入过")*
    ↓
setIsInViewport(entry.isIntersecting)
    ↓
如果有 ref.current → observer.observe(ref.current) *(开始观察元素)*

元素进入视野:
    ↓
IntersectionObserver 触发回调
    ↓
setIsInViewport(true)
    ↓
setIsLoaded(true) (如果之前是 false)

元素离开视野:
    ↓
IntersectionObserver 触发回调
    ↓
setIsInViewport(false)
    ↓
isLoaded 保持 true (曾经进入过就不再改变)

返回值: isInViewport || isLoaded *(只要曾经进入过就返回true)*
```

### 数据流

```
ref (DOM 元素)
    ↓
IntersectionObserver.observe() *(开始观察元素)*
    ↓
IntersectionObserver 回调 (entry.isIntersecting) *(元素可见性变化时触发)*
    ↓
isInViewport state → isLoaded state *(两个状态追踪当前和历史)*
    ↓
isInViewport || isLoaded *(返回值：当前可见或曾经可见过)*
```

## 四、AI 提示词编写教学

你正在做一个"元素进入屏幕视野"的小探测器。在网页上经常需要知道用户滚动到某个位置没有，比如图片懒加载、无限滚动加载更多、或者某个动画该开始播放了。

用浏览器提供的 IntersectionObserver API 来做这件事。这个 API 可以观察某个元素什么时候进入或离开了屏幕视野，比自己监听滚动事件要高效得多。

你需要两个状态：一个记录"现在是不是在视野里"，另一个记录"是不是曾经进入过视野"。一开始都是 false。当元素第一次进入视野的时候，"曾经进入过"这个状态就变成 true 了，而且以后永远保持 true，不会再变回去。返回的时候把这两个状态"或"一下，意思就是"当前在视野里"或者"曾经进来过"都返回真。

要观察的元素通过 ref 传进来，所以在开始观察之前要先检查一下 ref 指向的元素到底有没有。如果 ref.current 还是空的（比如元素还没渲染出来），就不要急着去观察它。

等组件不用了的时候，记得告诉观察器"不要再观察这个元素了"，把它断开连接，不然可能会出问题。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
import { useInViewport } from './useInViewport'
import { useRef } from 'react'

function LazyImage({ src, placeholder }) {
  const ref = useRef(null)
  const inViewport = useInViewport(ref)

  return (
    <img
      ref={ref}
      src={inViewport ? src : placeholder}
      alt="lazy loaded"
    />
  )
}
```

### 常见使用场景

🟩 **图片懒加载**

```typescript
function ImageWithLazyLoad({ src }) {
  const ref = useRef(null)
  const inViewport = useInViewport(ref)

  return (
    <div ref={ref}>
      {inViewport && <img src={src} alt="loaded" />}
    </div>
  )
}
```

🟩 **无限滚动**

```typescript
function InfiniteScrollList() {
  const ref = useRef(null)
  const inViewport = useInViewport(ref)
  const [items, setItems] = useState([])

  useEffect(() => {
    if (inViewport) {
      loadMoreItems().then(newItems => setItems([...items, ...newItems]))
    }
  }, [inViewport])

  return (
    <div>
      {items.map(item => <ListItem key={item.id} {...item} />)}
      <div ref={ref}>Loading...</div>
    </div>
  )
}
```

🟩 **动画触发**

```typescript
function AnimatedElement() {
  const ref = useRef(null)
  const inViewport = useInViewport(ref)

  return (
    <div
      ref={ref}
      className={inViewport ? 'animate-fade-in' : 'opacity-0'}
    >
      Content to animate
    </div>
  )
}
```

### Dos and Don'ts

✅ **Do: 为观察的元素添加 ref**

```typescript
// 正确：需要为要观察的元素添加 ref
const ref = useRef(null)
const inViewport = useInViewport(ref)
return <div ref={ref}>观察我</div>
```

❌ **Don't: 不传 ref 或传错误的 ref**

```typescript
// 错误：没有 ref 就无法观察任何元素
const inViewport = useInViewport() // 没有 ref

// 错误：ref 要指向要观察的元素
const ref = useRef(null)
const wrongRef = useRef(null)
const inViewport = useInViewport(wrongRef) // wrongRef 没有被使用
return <div ref={ref}>但观察的是另一个</div>
```

✅ **Do: 在 ref 对应的元素挂载后再检查**

```typescript
// 正确：元素已经渲染到 DOM 中
useEffect(() => {
  if (ref.current && inViewport) {
    // ref.current 存在，可以安全使用
    doSomething()
  }
}, [inViewport])
```

❌ **Don't: 在 ref.current 为 null 时使用它**

```typescript
// 错误：组件未挂载时 ref.current 是 null
console.log(ref.current.dataset) // 可能报错
```

✅ **Do: 结合 useEffect 做数据加载**

```typescript
// 正确：在进入视野时加载数据
useEffect(() => {
  if (inViewport && !hasLoaded) {
    fetchData()
  }
}, [inViewport])
```

❌ **Don't: 用 inViewport 做条件渲染的主要内容**

```typescript
// 错误：进入视野后永久返回 true，不适合做切换
return inViewport ? <HeavyComponent /> : <LightComponent />
// 上面这种场景建议用两个独立的状态
```

✅ **Do: 配合 SSR 使用**

```typescript
// 正确：SSR 时 ref.current 不存在，但 hook 会正常工作
function ServerComponent() {
  const ref = useRef(null)
  const inViewport = useInViewport(ref)
  // SSR 时 inViewport 为 false
  return <div ref={ref}>{inViewport && <Image />}</div>
}
```

❌ **Don't: 在 IntersectionObserver 回调中执行重操作**

```typescript
// 错误：回调应该轻量，每次可见性变化都会触发
observer.callback = ([entry]) => {
  // 错误：不要在这里执行复杂计算
  heavyComputation(entry)
  // 正确：只更新状态，让 useEffect 处理其他逻辑
  setIsIntersecting(entry.isIntersecting)
}
```
