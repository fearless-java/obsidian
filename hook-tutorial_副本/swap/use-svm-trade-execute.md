> 源代码路径: `apps/web/src/lib/hooks/react-query/trade/useSvmTradeExecute.ts`

# useSvmTradeExecute

## 1. 大白话讲讲这个hook的作用

`useSvmTradeExecute` *(一个React mutation hook，用于Solana链(SVM)上执行交易，支持wrap/unwrap和普通交易两种类型)* 是Solana链（SVM）上执行交易的mutation hook。

它帮你：
1. 签名交易（使用 Solana signer）
2. 如果是 wrap/unwrap（SOL/wSOL），直接签名发送
3. 如果是普通交易，调用 Jupiter 执行API完成交易

## 2. 讲讲为什么需要封装该hook

SVM交易执行复杂：
- 需要处理 wrap/unwrap 特殊情况
- 需要调用 Jupiter 特定的API
- 需要签名和发送交易
- 需要处理各种错误

封装后：
- 统一处理各种交易类型
- 自动处理 wrap/unwrap 逻辑
- 简洁的mutation接口

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
variables: UseSvmTradeParams
// {
//   chainId: SvmChainId
//   fromToken: SvmToken
//   toToken: SvmToken
//   amount: Amount
//   order?: Order  // Jupiter订单
//   requestId?: string
//   unsignedBytes?: Uint8Array
//   recipient?: string
// }
```

**输出：**
```typescript
{
  mutate: (params?) => void
  mutateAsync: (params?) => Promise
  data: TransactionSignature | undefined
  error: Error | null
  isPending: boolean
}
```

**执行逻辑：**
1. 解析参数：requestId、unsignedBytes
2. 如果是 wrap/unwrap：
   - 调用 `signer.signAndSendTransaction(unsignedBytes)` *(Solana签名者的方法，用于签名并直接发送交易)*
   - 直接返回签名
3. 如果是普通交易：
   - 签名交易：`signer.signTransaction(unsignedBytes)` *(Solana签名者的方法，用于签名交易但不发送)*
   - 调用 `/api/jupiter/ultra/execute` 获取最终签名
   - 返回

**数据流：**
```
输入参数
    |
    v
是wrap/unwrap? --> signAndSendTransaction --> 返回签名
    |
    否
    v
signTransaction --> /api/jupiter/ultra/execute --> 返回签名
```

## 4. 怎么给这个hook写AI提示词

这是Solana链（SVM）上执行交易的hook。它是个mutation（写入操作），不是query（读取）。两种情况：SOL/wSOL转换走wrap/unwrap逻辑，其他交易走Jupiter路由。

### 写提示词的小技巧

**第一，用useMutation，不是useQuery。** 这是执行交易，不是查数据。交易执行是副作用操作，要用 mutation。

**第二，区分wrap/unwrap和普通交易。** SOL和wSOL之间的转换比较特殊，直接签名发送就行了，不用走Jupiter的execute接口。

**第三，retry设成false。** 交易失败了一般是用户点了取消或者链上出问题了，重试没意义。重复发送可能会出问题。

### 写提示词时要注意的条条框框

**只支持SVM链。** EVM链的交易要用别的hook处理。

**unsignedBytes必须有。** 这是待签名的交易数据，没有的话啥都做不了。

**普通交易需要requestId。** wrap/unwrap不需要，但普通Jupiter交易需要靠requestId关联到之前的报价。

### 提示词模板

```
帮我写一个React hook，功能是在Solana链（SVM）上执行交易。

具体需求：
1. 用 useMutation，不是 useQuery
2. 支持两种类型：wrap/unwrap（SOL/wSOL转换）和普通交易
3. wrap/unwrap 直接签名发送（signAndSendTransaction）
4. 普通交易先签名（signTransaction），再调用 /api/jupiter/ultra/execute 获取最终签名
5. 用 Solana 的 signer 来签名

参数类型大概这样：
interface UseSvmTradeParams {
  chainId: SvmChainId
  fromToken: SvmToken
  toToken: SvmToken
  amount: Amount
  order?: Order
  requestId?: string
  unsignedBytes?: Uint8Array
  recipient?: string
}

返回：useMutation 标准结果

注意事项：
- 只支持 SVM 链
- 要处理 wrap/unwrap 的特殊情况
- retry: false，不要重试失败的交易
```

### 实际用的例子

```typescript
const { mutate, isPending } = useSvmTradeExecute(params)

<Button
  onClick={() => mutate()}
  disabled={isPending}
>
  {isPending ? '处理中...' : '执行交易'}
</Button>
```

mutation 用 mutate() 触发，mutateAsync() 返回 Promise，可以 await 获取交易签名。

### Production-Ready Example

```typescript
const { mutate, isPending } = useSvmTradeExecute(params)

<Button
  onClick={() => mutate()}
  disabled={isPending}
>
  {isPending ? '处理中...' : '执行交易'}
</Button>
```

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 这是useMutation，不是useQuery
const {
  mutate,
  mutateAsync,
  data: signature,
  error,
  isPending,
} = useSvmTradeExecute({
  chainId: SvmChainId.SOLANA,
  fromToken: SOL,
  toToken: USDC,
  amount: amountIn,
  order: jupiterOrder,
  requestId: '...',
  unsignedBytes: unsignedTx,
})

// 发送交易
mutate()

// 或者使用mutateAsync（返回Promise）
const handleSwap = async () => {
  try {
    const signature = await mutateAsync()
    console.log('Transaction signature:', signature)
  } catch (err) {
    console.error('Swap failed:', err)
  }
}
```

### 常见使用场景

**场景1：执行SVM交易**

```typescript
// 先获取报价
const { data: quote } = useSvmTradeQuote({...})

// 执行交易
const { mutate, isPending, error } = useSvmTradeExecute({
  chainId: SvmChainId.SOLANA,
  fromToken: SOL,
  toToken: USDC,
  amount: amountIn,
  order: quote?.order,
  requestId: quote?.requestId,
  unsignedBytes: quote?.unsignedBytes,
})

return (
  <Button
    onClick={() => mutate()}
    disabled={isPending || !quote}
  >
    {isPending ? '交易处理中...' : '确认交易'}
  </Button>
)
```

**场景2：SOL/wSOL 转换**

```typescript
// wrap SOL to wSOL
const { mutate: wrapSol } = useSvmTradeExecute({
  chainId: SvmChainId.SOLANA,
  fromToken: SOL,
  toToken: wSOL,
  amount: amountIn,
  unsignedBytes: wrapTx,
})

// unwrap wSOL to SOL
const { mutate: unwrapSol } = useSvmTradeExecute({
  chainId: SvmChainId.SOLANA,
  fromToken: wSOL,
  toToken: SOL,
  amount: amountIn,
  unsignedBytes: unwrapTx,
})
```

**场景3：交易状态追踪**

```typescript
const { data: signature, isPending, error } = useSvmTradeExecute({...})

// 监听交易确认
useEffect(() => {
  if (signature) {
    const connection = new Connection(clusterApiUrl('mainnet-beta'))
    connection.confirmTransaction(signature).then((result) => {
      if (result.value.err) {
        console.error('Transaction failed')
      } else {
        console.log('Transaction confirmed!')
      }
    })
  }
}, [signature])
```

### Dos and Don'ts

**✅ Do:**
- 使用 `mutate` 或 `mutateAsync` 触发交易
- 处理 `isPending` 状态禁用按钮
- 监听 `signature` 确认交易结果
- 处理 `error` 展示失败信息

**❌ Don't:**
- 不要在不需要交易时调用这个 hook
- 不要忽略 `isPending` 状态，用户需要知道交易正在进行
- 不要假设交易会成功，应该处理错误情况
- 不要在 SVM 链上使用，应该用 `useEvmTrade` 处理 EVM 链
