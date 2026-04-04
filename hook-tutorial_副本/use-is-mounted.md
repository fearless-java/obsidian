> 源代码路径: `packages/hooks/src/useIsMounted.ts`

# useIsMounted

## 1. 大白话讲讲这个 hook 的作用

`useIsMounted` *(一个React hook，用于检测组件是否已挂载，防止在组件卸载后执行异步状态更新，返回布尔值表示挂载状态)* 是一个检测"组件是否还在 mounted（挂载中）"的 hook。

它的作用很简单：
- 在组件刚创建时返回 `false`
- 在 `useEffect` 执行后（组件渲染完成）返回 `true`
- 在组件卸载后返回 `false`

简单来说：它告诉你"这个组件现在还在不在屏幕上"。

## 2. 讲讲为什么需要封装该 hook

### 解决异步操作的问题

最常见的场景是数据请求：

```typescript
function Component() {
  const [data, setData] = useState(null)

  useEffect(() => {
    fetchData().then(setData) // 异步操作 *(异步操作指不会立即完成的操作，如网络请求)*
  }, [])

  // 问题：如果用户在这个请求完成前离开了页面
  // setData 会被调用，但组件已经卸载了
  // React 会警告：Can't perform a React state update on an unmounted component
}
```

### 需要 useIsMounted 的场景

```typescript
function Component() {
  const [data, setData] = useState(null)
  const isMounted = useIsMounted()

  useEffect(() => {
    fetchData().then((result) => {
      if (isMounted()) { // *(调用函数获取最新挂载状态)*
        setData(result)
      }
    })
  }, [isMounted])

  // 现在只有组件还在mounted时才会更新状态
}
```

### 封装的好处
- **防止内存泄漏警告**：避免在组件卸载后 setState *(更新React组件状态，会触发重新渲染)*
- **API 简洁**：返回函数，调用即得布尔值
- **性能好**：只是简单布尔状态
- **用途明确**：用于 async/await 或 Promise *(Promise是ES6的异步编程解决方案，代表一个异步操作的最终结果)* 场景

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

无参数。

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| `mounted` | `boolean` | 组件是否已挂载 |

### 执行逻辑

```
组件初始化:
    ↓
mounted = false (初始状态)
    ↓
组件渲染完成
    ↓
useEffect 执行: setMounted(true) *(useEffect是React的副作用钩子，在渲染后执行)*
    ↓
mounted = true
    ↓
组件卸载:
    ↓
useEffect 清理函数... (mounted 状态不会变) *(状态不会自动变回false)*
    ↓
mounted 仍然是 true (但组件已经卸载)
```

### 数据流

```
组件挂载 → useEffect → setMounted(true)
    ↓
返回值: true

(组件卸载后，mounted 仍为 true，但调用方应该已经不使用这个值了)
```

## 四、AI 提示词编写教学

你正在做一个组件"存活状态"检测的小工具。为什么要检测这个呢？因为有时候组件已经"消失"了（用户导航走了，或者条件渲染关掉了），但之前发起的异步请求才刚刚返回。如果这时候还去更新组件的状态，React 会报警告。

这个小工具的使用方式是：在组件刚创建的时候它返回 false，等到组件真的在屏幕上显示出来了（也就是 useEffect 执行之后）它才变成 true。组件消失之后它继续保持 true（不会变回 false），但这没关系，因为组件消失后根本不会再渲染了，谁来关心这个值呢。

实现起来其实特别简单：用一个状态来记录"挂没挂载"，初始值是 false，然后在 useEffect 里设置成 true。useEffect 的依赖数组是空的，意思是只在组件"挂上去"的时候执行一次，之后再也不动它。

有些版本的实现会返回函数而不是布尔值，这样在回调函数里调用的时候能拿到"此时此刻"组件还在不在，但布尔值版本其实也够用了。

### 变体：返回函数而非值

```markdown
## 变体：返回函数版本

有些实现返回函数而非布尔值：

```typescript
export function useIsMounted() {
  const ref = useRef(false)
  useEffect(() => {
    ref.current = true
    return () => {
      ref.current = false
    }
  }, [])
  return () => ref.current
}
```

这样在回调中调用 `isMounted()` 可以获取最新值。

## 使用区别
```typescript
// 返回布尔值版本
const isMounted = useIsMounted()
if (isMounted) setData(data) // 可能已经不对了

// 返回函数版本
const isMounted = useIsMounted()
if (isMounted()) setData(data) // 获取最新值
```
```

## 5. 该 hook 的用法教学

### 基本用法

```typescript
import { useIsMounted } from './useIsMounted'

function DataComponent() {
  const [data, setData] = useState(null)
  const isMounted = useIsMounted()

  useEffect(() => {
    fetchData().then((result) => {
      if (isMounted) { // *(布尔值版本)*
        setData(result)
      }
    })
  }, [])

  return <div>{data}</div>
}
```

### 常见使用场景

🟩 **异步数据获取**

```typescript
function UserProfile({ userId }) {
  const [user, setUser] = useState(null)
  const isMounted = useIsMounted()

  useEffect(() => {
    fetchUser(userId)
      .then(data => {
        if (isMounted) setUser(data)
      })
      .catch(error => {
        if (isMounted) setError(error)
      })
  }, [userId])

  if (!user) return <Loading />
  return <div>{user.name}</div>
}
```

🟩 **Promise 场景**

```typescript
function AsyncComponent() {
  const [result, setResult] = useState(null)
  const isMounted = useIsMounted()

  const handleSubmit = async () => {
    const response = await submitForm()
    if (isMounted) {
      setResult(response)
    }
  }

  return (
    <div>
      <button onClick={handleSubmit}>提交</button>
      {result && <div>{result.message}</div>}
    </div>
  )
}
```

🟩 **WebSocket 连接**

```typescript
function WebSocketComponent() {
  const [messages, setMessages] = useState([])
  const isMounted = useIsMounted()

  useEffect(() => {
    const ws = new WebSocket('wss://example.com')

    ws.onmessage = (event) => {
      if (isMounted) {
        setMessages(prev => [...prev, event.data])
      }
    }

    return () => {
      ws.close()
    }
  }, [isMounted])

  return <MessageList messages={messages} />
}
```

### Dos and Don'ts

✅ **Do: 使用函数式更新 setState**

```typescript
// 正确：使用函数式更新
setData(prevData => {
  if (isMounted) return newData
  return prevData
})

// 或者直接检查
if (isMounted) {
  setData(newData)
}
```

❌ **Don't: 在 useEffect 依赖中包含 isMounted（布尔值版本）**

```typescript
// 错误：isMounted 不会变，加到依赖里没意义
useEffect(() => {
  fetchData().then(setData)
}, [isMounted]) // isMounted 永远是 true

// 正确：依赖应该包含你要用的值
useEffect(() => {
  fetchData(userId).then(setData)
}, [userId])
```

✅ **Do: 在 Promise 回调中检查**

```typescript
// 正确：在异步操作的回调中检查
fetchData().then(data => {
  if (isMounted) setData(data)
})
```

❌ **Don't: 用 isMounted 阻止组件卸载时的清理**

```typescript
// 错误：isMounted 不能阻止清理工作
useEffect(() => {
  return () => {
    // 这里不应该用 isMounted
    // 清理工作应该在组件卸载时执行
    cleanup()
  }
}, [])
```

✅ **Do: 结合 useCallback 使用**

```typescript
// 正确：在回调中使用
const handleClick = useCallback(async () => {
  const result = await api.call()
  if (isMounted) setResult(result)
}, [isMounted])
```

❌ **Don't: 忘记组件卸载后异步操作仍可能完成**

```typescript
// 错误：假设 isMounted 能完全阻止所有问题
useEffect(() => {
  // 即使设置了 isMounted，网络请求仍然会完成
  // 只是不会处理结果而已
  fetchData().then(setData)
}, [])

// 正确：仍然需要检查
useEffect(() => {
  fetchData().then(data => {
    if (isMounted) setData(data)
  })
}, [])
```

✅ **Do: SSR 时返回 false**

```typescript
// 正确：SSR 时 isMounted 为 false
function ServerComponent() {
  const isMounted = useIsMounted()
  // SSR 时 isMounted 是 false
  return isMounted ? <ClientOnly /> : <Fallback />
}
```

❌ **Don't: 在渲染时使用 isMounted 做条件渲染**

```typescript
// 错误：isMounted 是为了异步操作设计的
// 初始渲染时 isMounted 为 false，可能导致闪烁
return (
  <div>
    {isMounted ? <ActualContent /> : <Loading />} // 可能闪烁
  </div>
)

// 正确：用专门的状态做条件渲染
const [showContent, setShowContent] = useState(false)
useEffect(() => {
  fetchData().then(() => {
    if (isMounted) setShowContent(true)
  })
}, [])
```
