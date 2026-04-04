> 源代码路径: `packages/hooks/src/useTimeout.ts`

# useTimeout

## 1. 大白话讲讲这个 hook 的作用

`useTimeout` *(一个React hook，用于实现setTimeout延时器功能，使用ref避免闭包陷阱，支持暂停和自动清理)* 是一个让你可以在 React 组件中使用"延时器"的 hook，类似于原生的 `setTimeout` *(JavaScript原生函数，用于延时执行代码)*，但它：
1. 可以在组件中直接使用，不需要手动管理生命周期
2. 回调函数更新时能自动获取最新值（不会闭包陷阱 *(闭包陷阱指回调函数捕获创建时的变量值，而非最新值)*）
3. 传入 `delay = null` 时会取消延时器
4. 组件卸载时自动清理

简单来说：它让你在 React 里方便地使用"等一等再执行"的功能。

## 2. 讲讲为什么需要封装该 hook

### 原生 setTimeout 在 React 中的问题

```javascript
// 错误示例：闭包陷阱
useEffect(() => {
  const id = setTimeout(() => {
    console.log(count) // count 永远是初始值！
  }, 1000)
  return () => clearTimeout(id)
}, []) // count 变化时不会重新执行

// 正确但繁琐：需要把 count 放到依赖里
useEffect(() => {
  const id = setTimeout(() => {
    console.log(count)
  }, 1000)
  return () => clearTimeout(id)
}, [count])
```

### 需要处理好的细节

1. **闭包陷阱**：setTimeout 回调会捕获创建时的变量值，callback 变化时不会更新
2. **清理工作**：组件卸载时要清除定时器
3. **延迟控制**：delay 为 null 时要取消
4. **callback 更新**：callback 变化时不需要重启定时器（因为 refs *(React hook，用于存储可变数据)*）

### 封装的好处

- **不闭包**：使用 ref 存储最新 callback
- **自动清理**：useEffect 自动处理清理工作
- **用法直观**：和 setTimeout API 类似
- **与 useInterval 对称**：配套使用 *(useInterval用于间隔重复，useTimeout用于单次延时)*

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

| 参数 | 类型 | 说明 |
|------|------|------|
| `callback` | `() => void` | 要执行的回调函数 |
| `delay` | `number \| null` | 延迟时间（毫秒），null 表示不执行 *(1000ms = 1秒)* |

### 输出

无返回值（执行副作用）。

### 执行逻辑

```
组件挂载:
    ↓
savedCallback.current = callback *(ref存储最新回调，避免闭包问题)*
    ↓
useIsomorphicLayoutEffect *(React hook，在DOM更新后同步执行，SSR时跳过)*
    ↓
savedCallback.current = callback (同步最新 *(立即同步最新值到ref)*
    ↓
useEffect:
    ↓
delay 存在？ → yes → setTimeout(savedCallback.current, delay) *(延时器，delay毫秒后执行)*
    ↓
no → return undefined
    ↓
组件卸载: clearTimeout (如果存在) *(清除定时器，防止内存泄漏)*

delay 变化:
    ↓
先清理旧的 timeout *(避免多个定时器同时运行)*
    ↓
delay !== null？ → 创建新的 timeout
    ↓
return undefined
```

### 数据流

```
callback (函数)
    ↓
savedCallback.current = callback (ref 更新 *(ref存储最新回调函数)*
    ↓
setTimeout tick *(延时器触发)*
    ↓
savedCallback.current() (调用最新 callback)
```

## 四、AI 提示词编写教学

你正在做一个"等一等再执行"的小工具。就像 setTimeout 一样，等指定的时间到了再去执行某个操作，但要适合在 React 的环境里安全使用。

首先要解决的还是闭包陷阱问题。如果直接把这个要执行的函数放进 setTimeout 里，等时间到了触发的时候，函数里用到的变量可能已经过时了。解决方法是始终用 ref 来存这个函数，这样等时间到了调用的时候，调用的是 ref 里最新的版本。

需要用 ref 同步最新的回调函数。这里的关键是要用"同构布局 effect"，因为这个操作涉及到 DOM 层面，SSR 的时候不能跑，所以用这个特殊的版本来确保只在浏览器环境执行。

时间到了就执行 ref 里存的函数。如果传进来的"等多久"变成 null，说明不想执行了，那就不要创建定时器。传 0 的话并不是立即执行，setTimeout 会把它当成"下一个事件循环 tick 再执行"来处理。

组件不用了要把定时器清掉，防止内存泄漏。

## 五、该 hook 的用法教学

### 基本用法

```typescript
import { useTimeout } from './useTimeout'
import { useState } from 'react'

function DelayedMessage() {
  const [show, setShow] = useState(false)

  useTimeout(() => {
    setShow(true)
  }, 2000) // 2秒后显示

  return show ? <div>消息已显示</div> : <div>等待中...</div>
}
```

### 常见使用场景

🟩 **延迟显示加载状态**

```typescript
function LoadingButton({ onClick, children }) {
  const [isLoading, setIsLoading] = useState(false)
  const [showSpinner, setShowSpinner] = useState(false)

  const handleClick = async () => {
    setIsLoading(true)
    // 如果1秒内没完成，才显示spinner
    useTimeout(() => setShowSpinner(true), 1000)

    try {
      await onClick()
    } finally {
      setIsLoading(false)
      setShowSpinner(false)
    }
  }

  return (
    <button onClick={handleClick} disabled={isLoading}>
      {showSpinner ? '加载中...' : children}
    </button>
  )
}
```

🟩 **输入防抖**

```typescript
function DebouncedSearch() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])

  const handleSearch = async (searchQuery) => {
    const data = await searchAPI(searchQuery)
    setResults(data)
  }

  useTimeout(() => {
    if (query) {
      handleSearch(query)
    }
  }, 300) // 300ms 防抖

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {results.map(r => <ResultItem key={r.id} {...r} />)}
    </div>
  )
}
```

🟩 **自动隐藏提示**

```typescript
function Toast({ message, duration = 3000 }) {
  const [visible, setVisible] = useState(true)

  useTimeout(() => {
    setVisible(false)
  }, duration)

  if (!visible) return null

  return <div className="toast">{message}</div>
}
```

### Dos and Don'ts

✅ **Do: 使用 ref 避免闭包问题**

```typescript
// 正确：ref 存储最新值
useTimeout(() => {
  setValue(count + 1) // 这里的 count 永远是初始值！
}, 1000)

// 正确：使用函数式更新
useTimeout(() => {
  setValue(v => v + 1) // 正确：使用函数获取最新值
}, 1000)
```

❌ **Don't: 在 callback 中直接引用外部变量**

```typescript
// 错误：闭包陷阱，value 永远是初始值
const [value, setValue] = useState(initialValue)
useTimeout(() => {
  doSomething(value) // value 永远是初始值
}, 1000)
```

✅ **Do: 使用 null 停止定时器**

```typescript
// 正确：delay=null 时停止
useTimeout(() => {
  doSomething()
}, isActive ? 1000 : null)
```

❌ **Don't: 使用 0 作为 delay 来"立即执行"**

```typescript
// 错误：delay=0 会在下一个 tick 执行，不是立即
useTimeout(() => {
  doSomething()
}, 0) // 会在下一个事件循环执行

// 如果需要立即执行，直接调用即可
doSomething() // 不要用 timeout
```

✅ **Do: 组件卸载时自动清理**

```typescript
// 正确：组件卸载时自动清理
function MyComponent() {
  useTimeout(() => {
    saveData()
  }, 5000)
  // 无需手动清理，hook 内部处理了
}
```

❌ **Don't: 在 useEffect 中手动管理 setTimeout**

```typescript
// 错误：不要手动管理
useEffect(() => {
  const id = setTimeout(() => doSomething(), 1000)
  return () => clearTimeout(id)
}, [])

// 正确：使用 hook
useTimeout(() => doSomething(), 1000)
```

✅ **Do: 结合 useDebounce 和 useTimeout**

```typescript
// 正确：分工明确
// useDebounce 用于"输入"侧的防抖
const debouncedQuery = useDebounce(query, 300)

// useTimeout 用于"输出"侧的延迟反馈
useTimeout(() => setShowTooltip(true), 500)
```

❌ **Don't: 混用 useInterval 和 useTimeout 的职责**

```typescript
// 错误：不要用 useTimeout 实现定期执行
useTimeout(() => {
  doSomething()
  useTimeout(() => {...}, 1000) // 不要嵌套！
}, 1000)

// 正确：定期执行应该用 useInterval
useInterval(() => doSomething(), 1000)
```
