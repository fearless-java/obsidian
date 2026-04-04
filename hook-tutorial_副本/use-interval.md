> 源代码路径: `packages/hooks/src/useInterval.ts`

# useInterval

## 1. 大白话讲讲这个 hook 的作用

`useInterval` *(一个React hook，用于实现setInterval定时器功能，使用ref避免闭包陷阱，支持暂停和立即执行)* 是一个让你可以在 React 组件中使用"定时器"的 hook，类似于原生的 `setInterval` *(JavaScript原生函数，用于设置每隔一段时间执行一次的回调)*，但它：
1. 可以在组件中直接使用，不需要手动管理生命周期
2. 回调函数更新时能自动获取最新值（不会闭包陷阱 *(闭包陷阱指回调函数捕获的是创建时的变量值，而不是最新值)*）
3. 可以控制是否立即执行第一次回调（leading 参数）
4. 传入 `delay = null` 时会停止定时器

简单来说：它让你在 React 里方便地使用"每隔一段时间执行一次"的功能。

## 2. 讲讲为什么需要封装该 hook

### 原生 setInterval 在 React 中的问题

```javascript
// 错误示例：闭包陷阱
useEffect(() => {
  const id = setInterval(() => {
    console.log(count) // count 永远是初始值！
  }, 1000)
  return () => clearInterval(id)
}, []) // count 变化时不会重新执行

// 正确但繁琐：需要把 count 放到依赖里
useEffect(() => {
  const id = setInterval(() => {
    console.log(count)
  }, 1000)
  return () => clearInterval(id)
}, [count])
```

### 需要处理好的细节
1. **闭包陷阱**：setInterval 回调会捕获创建时的变量值，callback 变化时不会更新
2. **清理工作**：组件卸载时要清除定时器
3. **延迟控制**：delay 为 null 时要停止，为 0 时要立即重复执行
4. **leading 模式**：需要决定是否立即执行第一次

### 封装的好处
- **不闭包**：使用 ref *(React hook，用于存储可变数据，不触发重新渲染)* 存储最新 callback，每次 tick 用最新值
- **自动清理**：useEffect 自动处理清理工作
- **可控性强**：可以随时暂停（delay=null）
- **用法直观**：和 setInterval API 类似

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `callback` | `() => void` | - | 要定期执行的回调函数 |
| `delay` | `null \| number` | - | 间隔时间（毫秒），null 表示停止 *(1000ms = 1秒)* |
| `leading` | `boolean` | `true` | 是否立即执行第一次 *(true则在定时器创建时立即执行一次)* |

### 输出

无直接返回值（执行副作用）。

### 执行逻辑

```
组件挂载:
    ↓
savedCallback.current = callback *(ref存储最新回调，避免闭包)*
    ↓
如果 delay !== null *(null表示停止)*:
    ↓
如果 leading === true → 立即执行一次 callback
    ↓
设置 setInterval *(JavaScript定时器，每隔delay毫秒执行一次)*，每 delay ms 执行一次 savedCallback.current
    ↓
返回清理函数 → clearInterval *(清除定时器)*

delay 变化:
    ↓
先清理旧的 interval *(避免多个定时器同时运行)*
    ↓
如果 delay !== null → 创建新的 interval
    ↓
leading === true → 立即执行一次

组件卸载:
    ↓
清理 interval *(防止内存泄漏)*
```

### 数据流

```
callback (函数)
    ↓
savedCallback.current = callback (ref 更新 *(ref是React hook，用于存储可变数据)*)
    ↓
setInterval tick *(定时器触发)*
    ↓
savedCallback.current() (调用最新 callback)
```

## 四、AI 提示词编写教学

你正在做一个定时重复执行的小工具。就像 setInterval 一样，每隔一段时间就执行一次指定的代码，但要在 React 的环境里安全地工作。

首先要解决的问题是"闭包陷阱"。如果直接把这个要执行的函数放进定时器里，等定时器真正触发的时候，这个函数里用到的变量可能已经过时了（比如一个 count 值永远是 0）。解决办法是用 ref 来存这个函数，ref 里的值随时可以更新，但不会触发重新渲染。

每次那个要执行的函数变化了，就更新一下 ref 存着的最新版本。真正定时触发的时候，调用 ref 里存的那个函数，这样永远用的是最新的。

接下来管好定时器本身。如果设置了"每次开始前先执行一次"（leading 参数），那在设置定时器之前就先调用一次。然后设置一个每隔一段时间就触发一次的定时器。

当"等多久"这个参数变成 null 的时候，说明要停下来，那就不要创建定时器。组件不用了的时候要把定时器清掉，防止内存泄漏或者奇怪的行为。

注意不要把要执行的函数放进定时器相关的依赖数组里，那样每次函数变化都会重新创建定时器，可能会出问题。只要管好"等多久"和"是否立即执行"这两个参数就行。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
import { useInterval } from './useInterval'
import { useState } from 'react'

function Timer() {
  const [count, setCount] = useState(0)

  useInterval(() => {
    setCount(c => c + 1)
  }, 1000) // 每秒增加1

  return <div>Count: {count}</div>
}
```

### 常见使用场景

🟩 **轮询数据**

```typescript
function PollForData() {
  const [data, setData] = useState(null)

  useInterval(async () => {
    const result = await fetchLatestData()
    setData(result)
  }, 5000) // 每5秒轮询一次

  return <div>{data ? JSON.stringify(data) : 'Loading...'}</div>
}
```

🟩 **暂停/恢复定时器**

```typescript
function PausableTimer() {
  const [count, setCount] = useState(0)
  const [isRunning, setIsRunning] = useState(true)

  useInterval(() => {
    setCount(c => c + 1)
  }, isRunning ? 1000 : null) // null 时停止

  return (
    <div>
      <span>Count: {count}</span>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Resume'}
      </button>
    </div>
  )
}
```

🟩 **倒计时**

```typescript
function Countdown({ seconds }) {
  const [remaining, setRemaining] = useState(seconds)

  useInterval(() => {
    if (remaining > 0) {
      setRemaining(r => r - 1)
    }
  }, 1000)

  return <div>{remaining}s</div>
}
```

### Dos and Don'ts

✅ **Do: 使用 ref 避免闭包问题**

```typescript
// 正确：ref 存储最新值
useInterval(() => {
  setCount(count + 1) // 这里的 count 永远是初始值！
}, 1000)

// 正确：使用函数式更新
useInterval(() => {
  setCount(c => c + 1) // 正确：使用函数获取最新值
}, 1000)
```

❌ **Don't: 在 callback 中直接引用外部变量**

```typescript
// 错误：闭包陷阱，value 永远是初始值
const [value, setValue] = useState(initialValue)
useInterval(() => {
  doSomething(value) // value 永远是初始值
}, 1000)
```

✅ **Do: 使用 null 停止定时器**

```typescript
// 正确：delay=null 时停止
useInterval(() => {
  doSomething()
}, isActive ? 1000 : null)
```

❌ **Don't: 使用 0 作为 delay 来"停止"**

```typescript
// 错误：delay=0 会导致立即重复执行
useInterval(() => {
  doSomething()
}, 0) // 这会导致死循环！
```

✅ **Do: 记得清理定时器**

```typescript
// 正确：组件卸载时自动清理
function MyComponent() {
  useInterval(() => {
    fetchData()
  }, 5000)
  // 无需手动清理，useInterval 内部处理
}
```

❌ **Don't: 手动管理 setInterval**

```typescript
// 错误：不要手动 setInterval，应该用 hook
useEffect(() => {
  const id = setInterval(() => doSomething(), 1000)
  return () => clearInterval(id)
}, [])
```

✅ **Do: 设置 leading=false 如果不想立即执行**

```typescript
// 正确：不立即执行，等 delay 后再开始
useInterval(() => {
  setCount(c => c + 1)
}, 1000, false) // 1秒后才第一次执行，不是立即
```

❌ **Don't: 同时使用多个 delay 变化**

```typescript
// 错误：delay 频繁变化会导致定时器反复重建
const [delay, setDelay] = useState(1000)
useEffect(() => {
  setDelay(getNewDelay()) // 不要在 useEffect 中修改 delay
}, [something])
```
