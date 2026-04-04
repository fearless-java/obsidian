> 源代码路径: `packages/hooks/src/useCopyClipboard.ts`

# useCopyClipboard

## 1. 大白话讲讲这个 hook 的作用

`useCopyClipboard` *(一个React hook，用于复制文本到用户剪贴板，基于navigator.clipboard API或document.execCommand实现，返回复制状态)* 是一个用于复制文本到剪贴板的 hook。它做的事情很简单：
1. 提供一个复制函数，调用后把任意文本复制到用户的剪贴板
2. 返回一个状态 `isCopied`，告诉你复制成功了没有（true = 成功，false = 失败）
3. 自动在指定时间后把 `isCopied` 重置为 `false`，这样 UI 可以显示"已复制！"的提示，然后消失

简单来说：它让你点一下按钮就能复制内容，并知道复制成没成功。

## 2. 讲讲为什么需要封装该 hook

浏览器原生有 `navigator.clipboard.writeText()` *(浏览器原生API，用于将文本写入剪贴板，现代浏览器支持)* API，但存在以下问题：

### 兼容性
- 旧版浏览器（尤其是 Safari）可能不支持 Clipboard API
- 需要做降级处理（如使用 `document.execCommand('copy')` *(已废弃的复制命令，兼容旧浏览器)*）

### 第三方库封装
- `copy-to-clipboard` *(一个npm包，封装了各种浏览器的复制操作，统一返回true/false表示成功失败)* 库已经处理了各种兼容性问题
- 提供统一的接口，返回 `true/false` 表示成功失败

### 状态管理
- 复制操作是"一次性"的，需要状态来追踪是否刚刚复制过
- 需要自动重置状态，否则 UI 会一直显示"已复制"
- 需要清理定时器，防止内存泄漏 *(内存泄漏指分配的内存无法回收，导致应用占用越来越多内存)*

### UX 需求
- 复制后通常要显示一个临时提示（如"Copied!"）
- 提示显示时间需要可配置（默认 500ms）
- 复制函数需要用 `useCallback` *(React hook，用于缓存函数引用，避免每次渲染创建新函数，优化性能)* 包裹，避免每次渲染都创建新函数

封装后可以：开箱即用、兼容性好、状态自动管理、内存安全。

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `timeout` | `number` | `500` | 复制成功状态重置的延迟时间（毫秒） *(1000ms = 1秒)* |

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| `[isCopied, copyFunction]` | `[boolean, (text: string) => void]` | isCopied 表示是否刚复制成功，copyFunction 是复制函数 |

### 执行逻辑

```
初始状态: isCopied = false

用户调用 copyFunction('hello'):
    ↓
copy-to-clipboard('hello') *(第三方库，处理各种浏览器兼容性)*
    ↓
返回是否复制成功 → setIsCopied(true/false)
    ↓
如果复制成功，启动 setTimeout(delay ms) *(延时器，在指定毫秒后执行回调)*
    ↓
时间到达后 → setIsCopied(false)
    ↓
清理定时器（下次渲染或卸载时）
```

### 数据流

```
用户调用 copyFunction(text)
    ↓
copy-to-clipboard(text) *(统一处理各浏览器兼容性)*
    ↓
boolean (success)
    ↓
isCopied state = true
    ↓
等待 timeout ms
    ↓
isCopied state = false
```

## 四、AI 提示词编写教学

你正在做一个复制粘贴的小工具，用户点一下就能把文本复制到剪贴板里，而且还能知道复制成没成功。

先准备一个状态，用来记录"现在是不是刚复制过"。一开始当然是 false，表示还没复制过。

写一个复制函数，这个函数接收要复制的文本，然后调用系统级的剪贴板复制功能。复制操作完成后，会返回一个结果告诉你到底成功没有。成功了就把状态改成 true。

复制成功之后，过一小会儿（可以配置这个时间，默认 500 毫秒）要把状态重置回 false，不然界面会一直显示"已复制"。这就需要用一个定时器来帮忙。

复制函数本身要用缓存包裹一下，保证每次渲染都是同一个函数引用，这样能避免一些不必要的性能问题。

当"已复制"状态变成 true 的时候，启动一个定时器，等够了时间就自动把状态改回 false。如果在这个过程中用户又复制了一次，那就先停掉之前的定时器，重新开始计时。组件销毁的时候也要记得把定时器清掉，防止出问题。

最后对外返回两个东西：一个是说"复制成功了没有"的布尔值，另一个就是那个复制函数本身。复制失败的情况也要处理，比如用户拒绝了剪贴板权限，这时候就返回 false 就好。

## 五、该 hook 的用法教学

### 基本用法

```typescript
import { useCopyClipboard } from './useCopyClipboard'

function CopyButton({ textToCopy }) {
  const [isCopied, copy] = useCopyClipboard()

  return (
    <button onClick={() => copy(textToCopy)}>
      {isCopied ? '已复制!' : '复制'}
    </button>
  )
}
```

### 常见使用场景

🟩 **复制链接**

```typescript
function ShareButton({ url }) {
  const [isCopied, copy] = useCopyClipboard()

  return (
    <button onClick={() => copy(url)}>
      {isCopied ? '链接已复制!' : '分享链接'}
    </button>
  )
}
```

🟩 **复制钱包地址**

```typescript
function WalletAddress({ address }) {
  const [isCopied, copy] = useCopyClipboard()

  return (
    <div>
      <span>{address}</span>
      <button onClick={() => copy(address)}>
        {isCopied ? '已复制' : '复制地址'}
      </button>
    </div>
  )
}
```

🟩 **复制代码片段**

```typescript
function CodeBlock({ code }) {
  const [isCopied, copy] = useCopyClipboard()

  return (
    <div>
      <pre>{code}</pre>
      <button onClick={() => copy(code)}>
        {isCopied ? '代码已复制!' : '复制代码'}
      </button>
    </div>
  )
}
```

### Dos and Don'ts

✅ **Do: 保持简洁的反馈文案**

```typescript
// 正确：简洁明了的反馈
<button onClick={() => copy(text)}>
  {isCopied ? '已复制' : '复制'}
</button>
```

❌ **Don't: 反馈时间过长**

```typescript
// 错误：5秒太长了，用户体验不好
const [isCopied, copy] = useCopyClipboard(5000)
```

✅ **Do: 处理复制失败的情况**

```typescript
// 正确：检查返回值
const handleCopy = (text) => {
  const success = copy(text)
  if (!success) {
    alert('复制失败，请手动复制')
  }
}
```

❌ **Don't: 在复制函数中使用 async/await**

```typescript
// 错误：copy-to-clipboard 是同步的，不需要 async
const handleCopy = async (text) => {
  await copy(text) // 不需要 await
  setIsCopied(true)
}
```

✅ **Do: 自定义超时时间**

```typescript
// 正确：对于需要长时间显示的场景
const [isCopied, copy] = useCopyClipboard(2000) // 2秒

// 对于短时反馈
const [isCopied, copy] = useCopyClipboard(300) // 300毫秒
```

❌ **Don't: 复制空字符串**

```typescript
// 错误：复制空字符串没有意义
copy('')

// 正确：确保有内容再复制
if (text) copy(text)
```

✅ **Do: 结合 useCallback 优化子组件**

```typescript
// 正确：避免不必要的重渲染
const CopyButton = useCallback(({ onCopy }) => {
  const [isCopied, copy] = useCopyClipboard()
  return <button onClick={() => { copy(text); onCopy() }}>复制</button>
}, [text])
```
