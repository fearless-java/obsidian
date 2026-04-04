> 源代码路径: `packages/hooks/src/useDebounce.ts`

# useDebounce

## 1. 大白话讲讲这个 hook 的作用

`useDebounce` *(一个React hook，用于延迟更新值，防止频繁触发，当值快速变化时只取最后一个值，常用于搜索输入防抖)* 是一个"防抖"hook。它的作用是：当你有一个快速变化的值（比如用户输入），但你不想每次变化都去执行某个操作（比如 API 请求），而是等用户"安静"下来之后再执行。

举个例子：
- 用户在搜索框输入 "bitcoin"
- 没有 debounce：发送 8 次请求（b, bi, bit, ...）
- 有 debounce：只发 1 次请求（等用户停笔 300-500ms 后）

简单来说：它让你可以"等一等"再行动，等输入稳定了再处理。

## 2. 讲讲为什么需要封装该 hook

### 原生实现的问题
```javascript
// 你需要自己写这样的逻辑
const [debouncedValue, setDebouncedValue] = useState(value)
useEffect(() => {
  const timer = setTimeout(() => setDebouncedValue(value), delay)
  return () => clearTimeout(timer)
}, [value, delay])
```
这段代码虽然不长，但每次用到都要复制粘贴，容易出错。

### 需要处理好的细节
1. **定时器清理**：组件卸载时要清定时器，否则可能 setState 到 unmounted 组件
2. **值变化时重启定时器**：如果 delay 期间值又变了，要取消之前的定时器，重新开始
3. **delay 变化时也要重启**：delay 变了，原来的定时器就没意义了
4. **泛型支持**：值的类型要保持一致，T 还是 T

### 封装的好处
- **不重复造轮子**：开箱即用，经过验证
- **逻辑正确**：正确处理所有边界情况
- **类型安全**：泛型确保类型不会丢
- **可读性好**：代码意图清晰

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

| 参数 | 类型 | 说明 |
|------|------|------|
| `value` | `T` | 要防抖的值（任意类型） *(泛型T可以是任何类型：string, number, object等)* |
| `delay` | `number` | 延迟时间（毫秒） *(1000ms = 1秒)* |

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| `debouncedValue` | `T` | 防抖后的值 |

### 执行逻辑

```
初始渲染:
    ↓
debouncedValue = initialValue（或 undefined）
    ↓
设置定时器，delay ms 后设置 debouncedValue = value
    ↓
返回 debouncedValue

值变化 (value → newValue):
    ↓
clearTimeout(旧定时器) *(清除上一个未执行的定时器)*
    ↓
设置新定时器，delay ms 后设置 debouncedValue = newValue
    ↓
返回 debouncedValue (仍是旧值，直到定时器触发)

组件卸载:
    ↓
clearTimeout(当前定时器)
    ↓
不执行任何 setState（防止内存泄漏）
```

### 数据流

```
原始值 (快速变化)
    ↓
setTimeout(delay) *(延时器，在指定毫秒后执行)*
    ↓
防抖后的值 (稳定后更新)
```

## 四、AI 提示词编写教学

你正在做一个防抖小工具。防抖的意思是：当用户快速输入什么东西的时候，不要每次输入都去处理，而是等用户"冷静"下来之后再处理。比如用户打字，每打一个字就去请求一次接口太浪费了，应该等用户停个几百毫秒不再输入了再去请求。

这个工具接收两个东西：要防抖的值（比如用户的搜索输入），还有就是"等多久"才处理（通常 300 到 500 毫秒比较合适）。

工作方式是这样的：每次值变了，就先把之前可能还在等的定时器清掉，然后重新开始一个定时器。如果在等待的过程中值又变了，那就再清掉重来。只有等用户真的停下来了，定时器才会跑完，这时候才把最新的值放出去。

如果等的时间（delay）变了，也要重启定时器，因为之前等的那个时间已经没意义了。

组件不用了的时候要把定时器清掉，不然可能会出问题。

返回的就是那个"等完才出来"的值。这个值的类型要和使用时传入的类型一样，不能变。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
import { useDebounce } from './useDebounce'
import { useState } from 'react'

function SearchInput() {
  const [query, setQuery] = useState('')
  const debouncedQuery = useDebounce(query, 300) // 300ms 防抖

  useEffect(() => {
    if (debouncedQuery) {
      // 执行搜索请求
      searchAPI(debouncedQuery)
    }
  }, [debouncedQuery])

  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="搜索..."
    />
  )
}
```

### 常见使用场景

🟩 **搜索输入防抖**

```typescript
function SearchComponent() {
  const [searchTerm, setSearchTerm] = useState('')
  const debouncedSearch = useDebounce(searchTerm, 500)

  useEffect(() => {
    // 只有当用户停止输入500ms后才执行搜索
    fetchResults(debouncedSearch)
  }, [debouncedSearch])

  return <input onChange={(e) => setSearchTerm(e.target.value)} />
}
```

🟩 **窗口 resize 防抖**

```typescript
function ResizeComponent() {
  const [width, setWidth] = useState(window.innerWidth)
  const debouncedWidth = useDebounce(width, 200)

  useEffect(() => {
    // 用户停止调整窗口200ms后才计算布局
    calculateLayout(debouncedWidth)
  }, [debouncedWidth])

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth)
    window.addEventListener('resize', handleResize)
    return () => window.removeEventListener('resize', handleResize)
  }, [])

  return <div>Width: {debouncedWidth}</div>
}
```

🟩 **表单验证防抖**

```typescript
function FormWithValidation() {
  const [email, setEmail] = useState('')
  const [error, setError] = useState('')
  const debouncedEmail = useDebounce(email, 300)

  useEffect(() => {
    if (debouncedEmail) {
      validateEmail(debouncedEmail)
        .then(() => setError(''))
        .catch(() => setError('无效的邮箱格式'))
    }
  }, [debouncedEmail])

  return (
    <div>
      <input value={email} onChange={(e) => setEmail(e.target.value)} />
      {error && <span className="error">{error}</span>}
    </div>
  )
}
```

### Dos and Don'ts

✅ **Do: 根据场景选择合适的 delay 时间**

```typescript
// 搜索输入：300-500ms 比较合适
const debouncedQuery = useDebounce(query, 300)

// 窗口 resize：200-300ms 足够
const debouncedWidth = useDebounce(width, 200)

// 动画控制：可能需要更短的 delay
const debouncedValue = useDebounce(value, 50)
```

❌ **Don't: delay 时间过长或过短**

```typescript
// 错误：2秒太长，用户体验差
const debouncedQuery = useDebounce(query, 2000)

// 错误：0ms 等于没做防抖
const debouncedQuery = useDebounce(query, 0)
```

✅ **Do: 在 useEffect 中使用 debounced 值**

```typescript
// 正确：在 useEffect 中使用防抖后的值
useEffect(() => {
  fetchData(debouncedValue)
}, [debouncedValue])
```

❌ **Don't: 在原始值上执行昂贵操作**

```typescript
// 错误：不要在原始值上执行 API 请求
useEffect(() => {
  fetchData(value) // 每次输入都会触发！
}, [value])
```

✅ **Do: 处理空值情况**

```typescript
// 正确：检查防抖后的值是否有效
useEffect(() => {
  if (debouncedQuery && debouncedQuery.length > 2) {
    fetchResults(debouncedQuery)
  }
}, [debouncedQuery])
```

❌ **Don't: 忘记清理组件卸载时的定时器**

```typescript
// 错误：useDebounce 内部已经处理了清理，不需要额外操作
// 但如果你的 useEffect 中有异步操作，需要注意
useEffect(() => {
  const controller = new AbortController()
  fetchData(debouncedQuery, { signal: controller.signal })
  return () => controller.abort()
}, [debouncedQuery])
```

✅ **Do: 泛型支持复杂对象**

```typescript
// 正确：useDebounce 支持任意类型
const [filters, setFilters] = useState({ category: '', price: 0 })
const debouncedFilters = useDebounce(filters, 500)
```

❌ **Don't: 用 debounced 值做精确比较**

```typescript
// 错误：防抖后的值可能和原始值不同，不适合做严格比较
if (debouncedValue === originalValue) { ... }
```
