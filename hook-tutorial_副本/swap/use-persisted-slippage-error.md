> 源代码路径: `apps/web/src/lib/hooks/usePersistedSlippageError.ts`

# usePersistedSlippageError

## 1. 大白话讲讲这个hook的作用

`usePersistedSlippageError` *(一个React hook，用于持久化显示滑点相关错误，即使在refetch期间也保持显示错误信息)* 是一个"记住"交易滑点错误的hook。

问题场景：
- 用户发起交易失败，错误是"滑点过低"
- React Query开始重新获取数据（refetch）
- 重新获取时，旧的错误信息丢了（被新的loading状态覆盖）
- 用户不知道刚才为什么失败

这个hook帮你：
- 检测"滑点过低"的特定错误
- 即使在refetch期间也保持显示这个错误
- 只有交易成功后才清除错误

## 2. 讲讲为什么需要封装该hook

Wagmi在refetch时会清除错误状态：
- 旧错误虽然有，但被loading状态覆盖
- UI无法区分"正在加载"和"之前失败过"
- 用户体验不好

封装后：
- 错误被"持久化"到本地state
- 直到下次成功才清除
- 提供 `show` 标志控制是否显示

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
{
  isSuccess: boolean    // 交易是否成功
  error: Error | null   // 当前错误
}
```

**输出：**
```typescript
{
  isSlippageError: boolean  // 是否是滑点错误
  show: boolean              // 是否应该显示错误
  setShow: (show: boolean) => void  // 控制显示
}
```

**执行逻辑：**
1. 检测错误信息是否包含滑点相关的关键词：
   - "Minimal output balance violation"
   - "MinimalOutputBalanceViolation"
   - "0x963b34a5"（错误代码）
2. 如果匹配，保存错误到 `persistedError`，设置 `show = true`
3. 如果 `isSuccess === true`，清除 `persistedError`
4. 返回 `isSlippageError`（是否有持久化错误）、`show`、`setShow`

**数据流：**
```
error?.message
         |
         v
    检测滑点关键词
         |
         v
    是 --> persistedError = error, show = true
    否 --> 保持
         |
         v
isSuccess === true --> persistedError = null
         |
         v
{ isSlippageError, show, setShow }
```

## 4. 怎么给这个hook写AI提示词

这个hook的核心功能是把滑点相关的错误"记住"，让它不会在数据重新获取的时候消失。写提示词的时候要注意以下几个要点：

### 写提示词的小技巧

**第一，只盯着滑点相关的错误处理。** 不是所有的错误都需要持久化，只有那种"滑点太低导致交易失败"的错误才值得记住。如果把所有错误都存起来，界面会很杂乱，用户也不知道该咋处理。

**第二，交易一旦成功了，错误就必须清除。** 这是个很重要的逻辑——只有当 `isSuccess` 变成 true 的时候，才能把存起来的错误清掉。不然用户早就交易成功了，错误提示还挂着，这就很奇怪了。

**第三，记得给组件留个关闭错误的口子。** `setShow` 这个函数就是干这个用的。有时候用户可能自己知道怎么解决（比如手动调高滑点），这时候他应该能把这个提示关掉。

### 写提示词时要注意的条条框框

**错误识别范围要明确：** 这个hook只认三种滑点相关的错误——"Minimal output balance violation"、"MinimalOutputBalanceViolation"，还有错误代码 "0x963b34a5"。其他错误一律不管。

**存错误的地方有讲究：** 错误是存在内存里的state中的，不是存在URL里也不是localStorage。这是刻意为之的，因为这种临时性的提示没必要持久化，刷新页面就没了也正常。

**检测逻辑要跟着依赖变：** 当 error 或者 isSuccess 变了的时候，要重新检测一次有没有滑点错误。useEffect 就是用来干这个的。

### 提示词模板

```
帮我写一个React hook，功能是"记住"滑点相关的错误信息。

具体需求是这样的：
1. 这个hook要监听两个东西：一个是 error（当前的错误对象），一个是 isSuccess（交易是不是成功了）
2. 当发现错误信息里包含"Minimal output balance violation"或者"MinimalOutputBalanceViolation"或者错误代码"0x963b34a5"的时候，就把这个错误存起来
3. 只有当 isSuccess 变成 true 的时候，才把存起来的错误清掉
4. 要返回三个东西：一个是 isSlippageError（现在有没有存着的滑点错误），一个是 show（要不要显示这个错误），一个是 setShow（用来手动关掉错误提示的函数）

返回类型大概是这样的：
{
  isSlippageError: boolean      // 是不是滑点错误
  show: boolean                 // 要不要显示
  setShow: (show: boolean) => void  // 可以手动控制显示隐藏
}

注意事项：
- 用useState存持久化的错误
- 用useEffect监听error和isSuccess的变化
- setShow是给组件用来主动关闭错误提示的
```

### 实际用的例子

```typescript
// 先拿到交易的状态和错误
const { isLoading } = useSwapTrade()
const { error } = useSendTransaction()

// 用这个hook来判断是不是滑点错误，并且要不要显示
const { isSlippageError, show } = usePersistedSlippageError({
  isSuccess: !isLoading && !error,
  error,
})

// 展示错误提示
{show && isSlippageError && (
  <Alert type="error" message="滑点过低，请提高滑点后重试" />
)}
```

这个例子里，当交易还在loading状态的时候，`isSuccess` 是 false，所以即使有错误也不会清除。等交易真正成功了（`isLoading && !error` 都满足），错误才会被清理掉。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 配合交易hook使用
const { isLoading } = useSwapTrade()
const { error } = useSendTransaction()

// 检测滑点错误
const { isSlippageError, show, setShow } = usePersistedSlippageError({
  isSuccess: !isLoading && !error, // 交易完成且没有错误
  error,
})

// 显示错误
{show && isSlippageError && (
  <Alert
    type="error"
    message="滑点过低，请提高滑点后重试"
    onClose={() => setShow(false)}
  />
)}
```

### 常见使用场景

**场景1：交易失败后显示滑点提示**

```typescript
const { write, isLoading: isPending } = useWriteContract()
const { wait, isLoading: isWaiting } = useWaitForTransaction()
const { error } = useSendTransaction()

const isSuccess = !isPending && !isWaiting && !error

const { isSlippageError, show } = usePersistedSlippageError({
  isSuccess,
  error,
})

// 用户交易失败后，自动显示滑点错误提示
return (
  <div>
    {show && isSlippageError && (
      <SlippageErrorBanner onDismiss={() => setShow(false)} />
    )}
    <SwapButton onClick={handleSwap} disabled={isPending || isWaiting} />
  </div>
)
```

**场景2：区分不同类型的错误**

```typescript
const { isSlippageError, show } = usePersistedSlippageError({
  isSuccess,
  error,
})

// 只有滑点错误才显示特殊处理
{show && isSlippageError ? (
  <ErrorDialog title="滑点过低">
    <p>交易价格变化太大，建议提高滑点 tolerance</p>
    <Button onClick={() => setShow(false)}>我知道了</Button>
  </ErrorDialog>
) : (
  // 其他错误显示通用错误
  error && <ErrorDialog title="交易失败" message={error.message} />
)}
```

**场景3：自动提高滑点建议**

```typescript
const { isSlippageError, show } = usePersistedSlippageError({...})

// 检测到滑点错误时，提供快速调整按钮
{show && isSlippageError && (
  <div className="slippage-warning">
    <span>滑点过低</span>
    <Button onClick={increaseSlippage}>将滑点调整为1%</Button>
  </div>
)}
```

### Dos and Don'ts

**✅ Do:**
- 在 `isSuccess` 变为 true 时自动清除错误
- 使用 `setShow` 允许用户主动关闭错误提示
- 将 `isSlippageError` 和 `show` 结合使用，只有都为 true 才显示
- 将此hook与交易流程的状态结合使用（isPending, isWaiting, error）

**❌ Don't:**
- 不要在所有错误上显示滑点提示，只处理特定的滑点相关错误
- 不要在 `isSuccess` 为 false 时清除 `persistedError`
- 不要混淆 `isSlippageError` 和 `show` 的用途
- 不要忽略 `isSuccess` 的判断逻辑，应该综合考虑 isPending、isWaiting 等状态
