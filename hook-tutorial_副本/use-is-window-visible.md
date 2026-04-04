> 源代码路径: `packages/hooks/src/useIsWindowVisible.ts`

# useIsWindowVisible

## 1. 大白话讲讲这个 hook 的作用

`useIsWindowVisible` *(一个React hook，用于检测页面是否对用户可见，基于Document Visibility API，返回true表示用户正在看这个页面)* 是一个检测"用户当前是否能看见这个页面"的 hook。

它检测的是浏览器的标签页是否被用户看到：
- 用户在看这个标签页 → `true`
- 用户切换到了其他标签页 → `false`
- 用户最小化了浏览器窗口 → `false`

简单来说：它告诉你"用户现在在看这个页面吗？"

## 2. 讲讲为什么需要封装该 hook

### 重要应用场景

1. **节省资源**：当用户看不到页面时，不需要执行动画、轮询等耗性能的操作
2. **暂停视频/音频**：用户切换标签时暂停，节省流量
3. **停止轮询**：用户不看页面时停止 API 轮询，回来再恢复
4. **游戏暂停**：当用户切走时暂停游戏逻辑

### 原生 API 的问题

```javascript
// 原生写法
document.addEventListener('visibilitychange', () => {
  console.log(document.visibilityState) // 'visible' 或 'hidden'
})
// 问题：需要手动管理事件监听和清理
// 问题：SSR 时 document 不存在
```

### 封装的好处
- **自动清理**：组件卸载时自动移除监听
- **SSR 安全**：服务端不会报错
- **状态驱动**：返回布尔值，直接用于条件渲染
- **性能优化**：visibilitychange 事件触发状态更新

## 3. 讲讲该 hook 的的执行逻辑和数据流向

### 输入（参数）

无参数。

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| `boolean` | `true/false` | 页面是否可见 |

### 执行逻辑

```
组件初始化:
    ↓
visibilityState 存在？ → yes → visibilityState !== 'hidden' → focused = true *(检查document.visibilityState，'hidden'表示页面不可见)*
    ↓
no → focused = true (假设可见 *(SSR时假设页面可见，返回true)*)
    ↓
useEffect *(React副作用钩子，用于处理副作用如事件监听)*:
    ↓
visibilityState 不支持？ → 直接返回 *(浏览器不支持Visibility API时直接返回*
    ↓
添加 visibilitychange 事件监听 *(监听页面可见性变化事件)*
    ↓
事件触发时 → listener → setFocused(isWindowVisible())
    ↓
组件卸载: 移除事件监听 *(清理事件监听器，防止内存泄漏)*
```

### 数据流

```
document.visibilityState ('visible' | 'hidden') *(Document Visibility API提供的属性，表示页面可见性状态)*
    ↓
visibilitychange 事件 *(页面可见性变化时触发的事件*
    ↓
isWindowVisible() → document.visibilityState !== 'hidden'
    ↓
focused state → setFocused() *(React状态更新函数)*
    ↓
返回 focused
```

## 四、AI 提示词编写教学

你正在做一个"页面现在对用户可见吗"的小工具。什么叫可见？就是用户正在看这个标签页，或者这个标签页只是被切到后台了但还在。不可见的情况包括：用户切换到了别的标签页，或者把浏览器最小化了。

用浏览器提供的 Document Visibility API 来检测这个状态。页面上有一个叫 visibilityState 的东西，它的值可能是"visible"（正在看）、"hidden"（被遮住了）、或者其他一些状态。我们只关心"hidden"这种情况，其他都当成可见。

不过要小心，服务端渲染的时候可没有 document 这个东西，得先检查一下这个 API 存不存在。要是不存在或者判断不了，就默认当成可见的（true），这样比较安全。

当页面的可见性发生变化的时候（用户切换标签页了），浏览器会触发一个叫 visibilitychange 的事件。我们就监听这个事件，每次触发的时候就更新一下状态。

页面的可见性变了，要去查一下当前的状态然后更新。用 useCallback 把这个处理函数包裹一下，保证它的引用稳定，不会每次渲染都重新创建一个新的。组件不用了的时候要把监听去掉，不然会内存泄漏。

## 五、该 hook 的用法教学

### 基本用法

```typescript
import { useIsWindowVisible } from './useIsWindowVisible'

function PollingComponent() {
  const [data, setData] = useState(null)
  const isVisible = useIsWindowVisible()

  useEffect(() => {
    if (!isVisible) return // 页面不可见时不请求

    const fetchData = async () => {
      const result = await api.getData()
      setData(result)
    }

    fetchData()
    const interval = setInterval(fetchData, 5000)

    return () => clearInterval(interval)
  }, [isVisible])

  return <div>{data}</div>
}
```

### 常见使用场景

🟩 **控制轮询**

```typescript
function AutoRefreshList() {
  const [items, setItems] = useState([])
  const isVisible = useIsWindowVisible()

  useEffect(() => {
    if (!isVisible) return

    const interval = setInterval(() => {
      fetchItems().then(setItems)
    }, 10000)

    return () => clearInterval(interval)
  }, [isVisible])

  return <List items={items} />
}
```

🟩 **暂停视频/音频**

```typescript
function VideoPlayer({ src }) {
  const [isPlaying, setIsPlaying] = useState(false)
  const isVisible = useIsWindowVisible()

  useEffect(() => {
    if (!isVisible && isPlaying) {
      videoRef.current.pause()
    } else if (isVisible && isPlaying) {
      videoRef.current.play()
    }
  }, [isVisible, isPlaying])

  return <video ref={videoRef} src={src} />
}
```

🟩 **记录用户活跃时间**

```typescript
function AnalyticsTracker() {
  const isVisible = useIsWindowVisible()
  const startTimeRef = useRef(Date.now())

  useEffect(() => {
    if (isVisible) {
      startTimeRef.current = Date.now()
    } else {
      const duration = Date.now() - startTimeRef.current
      recordActiveTime(duration)
    }
  }, [isVisible])
}
```

### Dos and Don'ts

✅ **Do: 页面不可见时停止轮询**

```typescript
// 正确：节省资源
useEffect(() => {
  if (!isVisible) return

  const interval = setInterval(fetchData, 5000)
  return () => clearInterval(interval)
}, [isVisible])
```

❌ **Don't: 在页面不可见时执行大量计算**

```typescript
// 错误：仍然消耗 CPU
useEffect(() => {
  if (!isVisible) {
    heavyComputation() // 仍然在后台运行
  }
}, [isVisible])
```

✅ **Do: 页面恢复时刷新数据**

```typescript
// 正确：页面恢复可见时获取最新数据
useEffect(() => {
  if (isVisible) {
    fetchData()
  }
}, [isVisible])
```

❌ **Don't: 依赖 isVisible 做关键业务逻辑**

```typescript
// 错误：用户可能只是最小化了一下
if (!isVisible) {
  // 不应该在这里做重要的状态保存
  saveImportantState()
}
```

✅ **Do: 结合 useCallback 优化**

```typescript
// 正确：避免不必要的重新渲染
const handleVisibilityChange = useCallback(() => {
  setIsVisible(document.visibilityState === 'visible')
}, [])

useEffect(() => {
  document.addEventListener('visibilitychange', handleVisibilityChange)
  return () => document.removeEventListener('visibilitychange', handleVisibilityChange)
}, [handleVisibilityChange])
```

❌ **Don't: 在组件外部使用 visibilityState**

```typescript
// 错误：visibilityState 是随页面状态变化的
const isVisible = document.visibilityState === 'visible'
// 这个值不会自动更新

// 正确：使用 hook 获取响应式的值
const isVisible = useIsWindowVisible()
```

✅ **Do: SSR 时返回 true（假设可见）**

```typescript
// 正确：SSR 时 isVisible 为 true
function ServerComponent() {
  const isVisible = useIsWindowVisible()
  // SSR 时 isVisible 是 true
  return <div>{isVisible ? '可见' : '不可见'}</div>
}
```

❌ **Don't: 忽略 visibilityState 的其他值**

```typescript
// 错误：visibilityState 可能有 'prerender', 'unloaded'
if (document.visibilityState === 'visible') {
  // 这会漏掉 'prerender' 状态
}

// 正确：只关心 'hidden'
const isVisible = document.visibilityState !== 'hidden'
```
