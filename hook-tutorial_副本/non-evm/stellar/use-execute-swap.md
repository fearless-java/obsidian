> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/swap/use-execute-swap.ts`

# useExecuteSwap / useExecuteMultiHopSwap Hook Tutorial

## 大白话讲讲这个hook的作用

`useExecuteSwap` *(一个React hook，用于执行单跳swap交易，封装了完整的swap流程和通知)* 和 `useExecuteMultiHopSwap` *(一个React hook，用于执行多跳swap交易，通过多个池子进行交换)* 是用于执行 swap 交易的 mutation hooks：

- **useExecuteSwap**：单跳 swap，直接从 TokenIn 交换到 TokenOut
- **useExecuteMultiHopSwap**：多跳 swap，通过多个池子进行交换

## 讲讲为什么需要封装该hook

封装这些 hooks 的原因：

1. **交易封装**：封装 swap 交易的完整流程
2. **Toast 通知**：显示交易进度和结果
3. **多跳支持**：支持复杂的路由交换
4. **缓存失效**：成功后刷新余额和头寸数据

## 讲讲该hook的执行逻辑和数据流向

### useExecuteSwap 参数
```typescript
{
  userAddress: string
  tokenIn: Token
  tokenOut: Token
  amountIn: bigint
  amountOutMinimum: bigint
  recipient: string
  deadline?: number
  sqrtPriceLimitX96?: bigint
  fee: number
}
```

### useExecuteMultiHopSwap 参数
```typescript
{
  userAddress: string
  path: string[]                 // 路由路径（代币地址数组）
  fees: number[]                 // 每个池子的手续费率
  amountIn: bigint
  amountOutMinimum: bigint
  recipient: string
  deadline?: number
}
```

### 输出（返回值）
```typescript
{
  mutate: () => void
  mutateAsync: () => Promise<{ result, params }>
  isPending: boolean
}
```

### 核心执行逻辑

1. **显示 info toast**：开始 swap 时显示
2. **执行 swap**：调用 `swapService.swapExactInputSingle` *(单跳swap服务方法)* 或 `swapExactInput` *(多跳swap服务方法)*
3. **显示 success toast**：成功后显示交换结果
4. **刷新缓存**：invalidate *(React Query方法，使查询缓存失效)* token balances、positions

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个执行swap交易的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useExecuteSwap的hook，用来执行某个区块链上的swap交易。然后明确几个关键点。第一，参数包括输入代币、输出代币、输入数量、最小输出数量、接收地址、手续费率，还可以设置价格限制和截止时间。第二，支持单跳swap，就是直接一代币换另一代币，也支持多跳swap，就是通过中间代币换，比如USDC->ETH->WBTC。第三，封装swap合约调用，这个比较复杂。第四，提供Toast通知，让用户知道交易进度和结果。第五，成功后要刷新代币余额和头寸数据。第六，用React Query的useMutation来处理。

### 这里面有几个地方特别容易出错

amountOutMinimum是最小输出数量，用于滑点保护，防止因为价格波动导致实际收到的比预期的少太多。deadline是交易截止时间，防止交易卡住无限等待。签名函数必须有，没有签名交易不会被网络接受。

### 数据刷新这里有讲究

用useMutation处理，能知道当前是否在处理中，防止用户重复点击。成功后要立即invalidate token和positions相关查询，余额变了就要刷新。这个hook是DEX的核心功能，做好了用户就能顺利完成交易。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useExecuteSwap } from '@sushiswap/hooks'
import { useStellarWallet } from './wallet'
import { parseUnits } from 'ethers'

function SwapForm() {
  const { address, signTransaction } = useStellarWallet()
  const [amountIn, setAmountIn] = useState('')

  const { mutate: executeSwap, isPending } = useExecuteSwap({
    onSuccess: (result) => {
      toast.success(`Swap 成功！获得 ${result.amountOut} ${tokenOut.symbol}`)
      queryClient.invalidateQueries(['token-balances'])
    },
    onError: (error) => {
      toast.error(`Swap 失败: ${error.message}`)
    }
  })

  const handleSwap = () => {
    const slippage = 0.005 // 0.5%
    const amountIn最小单位 = parseUnits(amountIn, tokenIn.decimals)
    const amountOutMinimum = calculateMinimumOut(amountOut, slippage)

    executeSwap({
      userAddress: address,
      tokenIn,
      tokenOut,
      amountIn: amountIn最小单位,
      amountOutMinimum,
      recipient: address,
      fee: 3000,
      signTransaction,
    })
  }

  return (
    <div>
      <input value={amountIn} onChange={(e) => setAmountIn(e.target.value)} />
      <button onClick={handleSwap} disabled={isPending}>
        {isPending ? 'Swap 中...' : 'Swap'}
      </button>
    </div>
  )
}
```

### 常见使用场景

1. **完整 Swap 流程**：带余额检查和授权
   ```tsx
   const { mutate: swap } = useExecuteSwap({
     onSuccess: () => {
       queryClient.invalidateQueries(['balances'])
     }
   })
   ```

2. **多跳 Swap**：通过中间代币进行交换
   ```tsx
   executeMultiHopSwap({
     path: [tokenA, intermediateToken, tokenB],
     fees: [3000, 3000],
     // ...
   })
   ```

3. **Swap 后操作**：Swap 成功后执行其他操作
   ```tsx
   const { mutate: swap } = useExecuteSwap({
     onSuccess: (result) => {
       toast.success('Swap 成功！')
       // 可以添加流动性或执行其他操作
     }
   })
   ```

### Dos and Don'ts

**Dos:**
- ✅ 使用 `amountOutMinimum` 提供滑点保护
- ✅ 使用 `deadline` 防止交易无限期等待
- ✅ 成功后刷新余额查询

**Don'ts:**
- ❌ 不要设置过低的滑点保护
- ❌ 不要忽略 isPending 状态
- ❌ 不要忘记签名函数
