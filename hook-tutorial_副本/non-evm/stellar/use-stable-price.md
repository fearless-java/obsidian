> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/price/use-stable-price.ts`

# useStablePrice Hook Tutorial

## 大白话讲讲这个hook的作用

`useStablePrice` *(一个React hook，用于获取代币的USD价格，通过多个稳定币交易对计算最优价格)* 是一个用于获取代币 USD 价格的 hook。它通过：

- 获取该代币与多个稳定币的交易路由
- 计算最优价格
- 返回最高价格（可能来自不同交易对）

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **多稳定币对比**：从多个稳定币池子获取价格
2. **路由计算**：使用 `getBestRoute` *(计算最优交易路径的函数)* 计算最优路径
3. **Pool Graph**：依赖 `usePoolGraph` *(获取池子关系图的hook)* 提供交易路由数据

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  token: Token | undefined         // 代币信息
}
```

### 输出（返回值）
```typescript
{
  data: number                    // USD 价格
  isLoading: boolean
  isPending: boolean
}
```

### 核心执行逻辑

1. **获取稳定币列表**：使用 `getStableTokens()` *(获取稳定币列表的工具函数)*
2. **构建池图**：使用 `usePoolGraph` 获取路由
3. **遍历计算**：对每个稳定币计算最优路由价格
4. **返回最高**：返回所有价格中的最高值

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个获取代币USD价格的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useStablePrice的hook，用来获取某个区块链上代币的美元价格。然后明确几个关键点。第一，参数是代币信息，有了代币信息才能查价格。第二，从多个稳定币交易对获取价格，比如一个代币可能有USDC交易对、USDT交易对等多个价格来源。第三，用getBestRoute计算最优路径，从多个价格里选一个最好的。第四，返回最佳的USD价格。第五，用React Query管理数据。

### 这里面有几个地方特别容易出错

要计算多个路由，因为一个代币可能和多个稳定币都有交易对，每个交易对的价格都可能不同。选择最优价格通常选最高的那个，因为最高的才代表市场价。依赖usePoolGraph提供路由数据，没有路由数据就没法计算。

### 数据刷新这里有讲究

依赖usePoolGraph的数据，那个数据没准备好这个hook也不能用。价格变化比较快，staleTime设成10秒比较合适，超过10秒就要重新获取了。这个hook是很多功能的基础，比如计算TVL、计算收益等都需要用到价格数据。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useStablePrice } from '@sushiswap/hooks'

function TokenPriceDisplay({ token }: { token: Token }) {
  const { data: usdPrice, isLoading } = useStablePrice({ token })

  if (isLoading) return <p>加载中...</p>

  return (
    <p>USD 价格: ${usdPrice?.toFixed(6) ?? '0'}</p>
  )
}
```

### 常见使用场景

1. **代币 TVL 计算**：计算池子的 USD 总价值
   ```tsx
   const { data: price0 } = useStablePrice({ token: token0 })
   const { data: price1 } = useStablePrice({ token: token1 })
   const tvl = (reserve0 * price0) + (reserve1 * price1)
   ```

2. **Swap 金额计算**：计算 swap 输出的 USD 价值
   ```tsx
   const { data: outPrice } = useStablePrice({ token: tokenOut })
   const usdValue = amountOut * outPrice
   ```

3. **价格变化显示**：显示代币价格的涨跌
   ```tsx
   const [price, setPrice] = useState()
   const { data: currentPrice } = useStablePrice({ token })

   useEffect(() => {
     if (currentPrice !== price) {
       const change = ((currentPrice - price) / price) * 100
       setPrice(currentPrice)
     }
   }, [currentPrice])
   ```

### Dos and Don'ts

**Dos:**
- ✅ 配合 `useLPUsdValue` *(计算池子USD价值的hook)* 使用
- ✅ 设置较短的 `staleTime` 因为价格随时变化
- ✅ 处理加载和错误状态

**Don'ts:**
- ❌ 不要在代币信息未加载时查询
- ❌ 不要忽略 null 检查
- ❌ 不要假设价格总是能获取到
