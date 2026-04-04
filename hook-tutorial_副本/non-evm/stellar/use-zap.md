> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/zap/use-zap.ts`

# useZap Hook Tutorial

## 大白话讲讲这个hook的作用

`useZap` *(一个React hook，用于一键添加流动性，用户只需投入单一代币系统自动swap并添加流动性)* 是一个用于"Zap"添加流动性的 hook。Zap 是一种一键添加流动性功能：

- 用户只需投入单一代币（如 USDC）
- 系统自动将其 swap 成两种代币
- 然后添加到流动性池

这个功能大大简化了添加流动性的流程。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **复杂路由**：Zap 需要先 swap 再添加流动性
2. **Quote 获取**：需要先获取 zap quote
3. **多签名**：需要签授权条目（Auth Entries）

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  poolAddress: string              // 池子地址
  tokenIn: Token                   // 输入代币
  amountIn: string                 // 输入数量
  tokenInDecimals: number          // 输入代币小数位
  token0: Token                    // 池子 Token0
  token1: Token                    // 池子 Token1
  tickLower: number                // 价格范围下限
  tickUpper: number                // 价格范围上限
  slippage?: number               // 滑点容忍度，默认 0.005
  userAddress: string
  signTransaction: (xdr: string) => Promise<string>
  signAuthEntry: (entryPreimageXdr: string) => Promise<string>
  routeToken0: RouteWithTokens | null  // 到 Token0 的路由
  routeToken1: RouteWithTokens | null  // 到 Token1 的路由
}
```

### 输出（返回值）
```typescript
{
  mutate: () => void
  mutateAsync: () => Promise<{ txHash, userAddress }>
  isPending: boolean
}
```

### 核心执行逻辑

1. **获取 Quote**：调用 `zapRouterClient.quote_zap_in` *(Zap路由器方法，获取Zap操作的预期结果)* 获取预期输出
2. **构建交易**：调用 `zapRouterClient.zap_in` *(Zap路由器方法，执行Zap添加流动性)* 构建交易
3. **签名 Auth Entries**：签授权条目
4. **签名交易**：签主交易
5. **提交**：提交到网络

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个Zap添加流动性的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useZap的hook，用来在某个区块链上做Zap添加流动性，就是一键添加。用户只需要投入单一代币，系统自动帮你换成两种代币然后添加到池子里。然后明确几个关键点。第一，参数包括池子地址、输入代币和数量、两种目标代币、tick范围、滑点容忍度、用户地址、两个签名函数，还有到两个目标代币的路由。第二，先获取zap quote，这个quote会告诉你预期能得到多少代币。第三，构建zap交易，这个交易会自动处理swap和添加流动性。第四，Stellar上有特殊的多签名机制，需要签Auth Entries和Transaction两层。第五，提供Toast通知。第六，用React Query的useMutation来处理。

### 这里面有几个地方特别容易出错

多签名流程要处理好，Auth Entries和Transaction都要签，漏了任何一个交易都不会成功。Quote结果要验证，万一quote失败了不能继续。路由必须有，没有路由就没法做swap操作。

### 数据刷新这里有讲究

用useMutation处理，能知道当前是否在处理中。成功后要立即invalidate pool和positions相关查询。Zap是个好功能，大大简化了添加流动性的流程，用户不用自己先做swap，直接一键搞定。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useZap } from '@sushiswap/hooks'
import { useStellarWallet } from './wallet'
import { parseUnits } from 'ethers'

function ZapLiquidityForm({ pool }: { pool: PoolInfo }) {
  const { address, signTransaction, signAuthEntry } = useStellarWallet()
  const [amountIn, setAmountIn] = useState('')

  const { mutate: zap, isPending } = useZap({
    onSuccess: (result) => {
      toast.success('流动性添加成功！')
      queryClient.invalidateQueries(['positions'])
      queryClient.invalidateQueries(['pool-balances'])
    },
    onError: (error) => {
      toast.error(`Zap 失败: ${error.message}`)
    }
  })

  const handleZap = () => {
    zap({
      poolAddress: pool.address,
      tokenIn: usdcToken, // 用户只有 USDC
      amountIn: parseUnits(amountIn, 6).toString(),
      tokenInDecimals: 6,
      token0: pool.token0,
      token1: pool.token1,
      tickLower: -1000,
      tickUpper: 1000,
      slippage: 0.005,
      userAddress: address,
      signTransaction,
      signAuthEntry,
      routeToken0: usdcToToken0Route,
      routeToken1: usdcToToken1Route,
    })
  }

  return (
    <div>
      <input value={amountIn} onChange={(e) => setAmountIn(e.target.value)} placeholder="输入 USDC 数量" />
      <button onClick={handleZap} disabled={isPending}>
        {isPending ? '处理中...' : '一键添加流动性'}
      </button>
    </div>
  )
}
```

### 常见使用场景

1. **简化添加流程**：用户只有单一代币时一键添加
   ```tsx
   // 用户有大量 USDC，想添加流动性但不想自己swap
   zap({
     tokenIn: usdc,
     amountIn: usdcAmount,
     // 自动完成 swap + 添加
   })
   ```

2. **显示 Zap 预览**：显示预期的 swap 数量和添加结果
   ```tsx
   const { data: quote } = useZapQuote({
     poolAddress,
     tokenIn,
     amountIn,
     // ...
   })
   // 显示: 将获得 X Token0 和 Y Token1
   ```

3. **多代币 Zap**：支持任意单一代币输入
   ```tsx
   const supportedTokens = [usdc, usdt, dai]
   // 用户选择任意一个作为输入
   ```

### Dos and Don'ts

**Dos:**
- ✅ 提供正确的路由信息
- ✅ 使用 `slippage` 提供滑点保护
- ✅ 处理 Auth Entry 和 Transaction 两层签名

**Don'ts:**
- ❌ 不要忽略路由检查
- ❌ 不要在没有足够余额时调用
- ❌ 不要在 isPending 时允许重复提交
