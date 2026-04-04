> 源代码路径: `packages/hooks/src/usePrevious.ts`

# usePrevious

## 1. 大白话讲讲这个 hook 的作用

`usePrevious` *(一个React hook，用于获取上一个渲染周期的值，基于useRef实现，常用于值变化比较)* 是一个返回"上一个值"的 hook。

当你需要知道某个状态变化前的值时（比如表单修改前后的对比），这个 hook 非常有用：

```typescript
const [count, setCount] = useState(0)
const previousCount = usePrevious(count)

// count: 0 → 1 → 2
// previousCount: undefined → 0 → 1
```

简单来说：它让你可以访问"上一次渲染时的值"。

## 2. 讲讲为什么需要封装该 hook

### 原生实现的问题

```javascript
// 你可能会这样写（但这是错误的）
const [previous, setPrevious] = useState()
useEffect(() => {
  setPrevious(count) // 这会多渲染一次 *(每次count变化都会触发额外渲染)*
}, [count])

// 或者用 ref（但逻辑容易出错）
const previousRef = useRef()
useEffect(() => {
  previousRef.current = count
}, [count])
// previousRef.current 是上一次的值
```

### useRef 方法的问题

```javascript
// useRef 方法在渲染期间读取会有问题
const previousCount = previousRef.current // 可能不是"上一个"
```

### 封装的好处

- **语义清晰**：返回"上一个值"而不是 ref.current
- **正确时机**：useEffect 在 DOM 更新后、渲染前设置值 *(React的更新流程：render -> useEffect执行)*
- **可选初始值**：可以指定初始值
- **类型安全**：泛型支持 *(泛型T可以是任何类型)*

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `value` | `T` | - | 当前的 value *(要追踪其历史值的当前变量)* |
| `initialValue` | `T` | `undefined` | 初始时的"上一个值" |

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| `T \| undefined` | 泛型 T 或 undefined | 上一次渲染时的 value |

### 执行逻辑

```
组件渲染:
    ↓
usePrevious(value) 调用
    ↓
ref.current = value (useEffect 之前？不对...)

useEffect 执行顺序 *(React渲染和副作用执行顺序)*:
    ↓
useEffect #1: ref.current = value (更新 ref)
    ↓
useEffect #2: 返回 ref.current (获取旧值)

渲染期间:
    ↓
返回 ref.current (此时还是旧值) *(渲染时ref还没更新，返回的是上一轮的值)*
    ↓
下一次渲染...
```

### 数据流

```
value (当前渲染的值)
    ↓
useEffect 执行: ref.current = value *(useEffect在DOM更新后执行，此时才更新ref)*
    ↓
ref.current 存储旧值 *(ref存储的是上一次渲染时的值)*
    ↓
下一次渲染: 返回 ref.current (旧值)
```

## 四、AI 提示词编写教学

你正在做一个"记住上一个值"的小工具。比如你想知道某个数字从几变成几了，或者用户改表单的时候字段到底变了没有，这时候就需要"上一个值"来比较。

用一个 ref 来存"上一个值"。ref 和 state 不一样，ref 改了不会触发重新渲染，但它可以自由地存任何值。

每次渲染完成后（useEffect 执行的时候），把当前的值存进 ref 里。注意这时候其实存的是"上一次渲染时的值"，因为 useEffect 是在渲染之后才跑的。

等下一次渲染开始的时候，这个 hook 返回的就是存在 ref 里的旧值。这样在渲染的过程中就能拿到"上一次的值"来做比较了。

ref 的初始值可以指定，也可以不指定（默认 undefined）。第一次渲染的时候返回的就是这个初始值。

实现的代码特别短：建一个 ref，接收到新值的时候用 useEffect 把 ref 更新一下，然后直接返回 ref.current 就好。

## 五、该 hook 的用法教学

### 基本用法

```typescript
import { usePrevious } from './usePrevious'
import { useState } from 'react'

function Counter() {
  const [count, setCount] = useState(0)
  const previousCount = usePrevious(count)

  return (
    <div>
      <p>Now: {count}, Before: {previousCount}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  )
}
```

### 常见使用场景

🟩 **表单变化检测**

```typescript
function FormWithChangeDetection() {
  const [formData, setFormData] = useState({ name: '', email: '' })
  const previousFormData = usePrevious(formData)

  const handleSubmit = () => {
    if (previousFormData) {
      const changedFields = Object.keys(formData).filter(
        key => formData[key] !== previousFormData[key]
      )
      console.log('Changed fields:', changedFields)
    }
    submitForm(formData)
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={formData.name}
        onChange={e => setFormData(prev => ({ ...prev, name: e.target.value }))}
      />
      <input
        value={formData.email}
        onChange={e => setFormData(prev => ({ ...prev, email: e.target.value }))}
      />
      <button type="submit">提交</button>
    </form>
  )
}
```

🟩 **动画触发**

```typescript
function AnimatedNumber({ value }) {
  const previousValue = usePrevious(value)

  useEffect(() => {
    if (previousValue !== undefined && value > previousValue) {
      triggerIncreaseAnimation()
    } else if (previousValue !== undefined && value < previousValue) {
      triggerDecreaseAnimation()
    }
  }, [value, previousValue])

  return <div>{value}</div>
}
```

🟩 **条件渲染控制**

```typescript
function DataList({ items, filter }) {
  const previousFilter = usePrevious(filter)

  useEffect(() => {
    if (previousFilter !== filter) {
      // filter 变化了，重新加载数据
      loadData(filter)
    }
  }, [filter, previousFilter])

  return <List items={items} />
}
```

### Dos and Don'ts

✅ **Do: 在渲染时使用 previous 值**

```typescript
// 正确：渲染时返回上一轮的值
function Component({ value }) {
  const previousValue = usePrevious(value)

  return (
    <div>
      <span>Current: {value}</span>
      <span>Previous: {previousValue}</span>
    </div>
  )
}
```

❌ **Don't: 在 useEffect 中期望获取最新值**

```typescript
// 错误：useEffect 执行时 previousValue 已经是上一个值了
useEffect(() => {
  console.log(previousValue) // 这是"上上一个"值！
}, [value, previousValue])
```

✅ **Do: 使用带初始值的版本**

```typescript
// 正确：指定初始值
const previousUser = usePrevious(currentUser, null)
// 初始渲染时返回 null，而不是 undefined
```

❌ **Don't: 假设 previous 值一定存在**

```typescript
// 错误：第一次渲染时 previousValue 是 undefined
if (previousValue === currentValue) { // 第一次会是 undefined
  // ...
}

// 正确：检查 undefined
if (previousValue !== undefined && previousValue === currentValue) {
  // ...
}
```

✅ **Do: 结合 useEffect 监听变化**

```typescript
// 正确：在 useEffect 中监听值变化
useEffect(() => {
  if (previousValue !== undefined) {
    console.log('Value changed from', previousValue, 'to', value)
  }
}, [value, previousValue])
```

❌ **Don't: 在渲染时直接修改 previous 值**

```typescript
// 错误：不要修改 ref 的值
const previous = usePrevious(value)
previous.current = value // 这没有任何意义！
```

✅ **Do: 使用泛型保持类型安全**

```typescript
// 正确：泛型自动推导
const previousCount = usePrevious(count) // number | undefined
const previousUser = usePrevious(user) // User | undefined

// 正确：显式指定类型
const previousIds = usePrevious<number[]>([])
```
