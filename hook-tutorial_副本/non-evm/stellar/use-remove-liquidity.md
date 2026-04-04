> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/liquidity/use-remove-liquidity.ts`

# useRemoveLiquidity Hook Tutorial

## 大白话讲讲这个hook的作用

`useRemoveLiquidity` *(一个React hook，用于移除流动性并收集本金和费用，封装了decreaseLiquidity和collectFees两个操作)* 是一个用于移除流动性的 mutation hook。它：

- 减少指定头寸的流动性
- 收集头寸的本金和费用
- 同时执行 decreaseLiquidity 和 collectFees *(两个相关的合约方法)*

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **两步操作**：需要先 decrease 再 collect
2. **Toast 通知**：显示操作进度
3. **缓存失效**：成功后刷新相关数据

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  tokenId: number                   // 头寸 ID
  liquidity: bigint                // 要移除的流动性数量
  amount0Min: bigint               // 最小 Token0 数量（滑点保护）
  amount1Min: bigint               // 最小 Token1 数量
  token0: Token                   // Token0 信息
  token1: Token                   // Token1 信息
  poolAddress: string              // 池子地址
}
```

### 输出（返回值）
```typescript
{
  mutate: () => void
  mutateAsync: () => Promise<{ decreaseResult, collectResult }>
  isPending: boolean
}
```

### 核心执行逻辑

1. **显示 info toast**：开始移除时显示
2. **减少流动性**：调用 `decreaseLiquidity` *(减少流动性头寸的合约方法)*
3. **收集代币**：调用 `collectFees` *(收集头寸本金和费用的合约方法)*
4. **显示 success toast**：成功后显示收集的代币数量
5. **刷新缓存**：invalidate *(React Query方法，使查询缓存失效)* 相关查询

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个移除流动性的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useRemoveLiquidity的hook，用来从某个区块链上的流动性池移除流动性。然后明确几个关键点。第一，参数包括头寸ID、要移除的流动性数量、最小代币数量0和1（用于滑点保护）、两种代币的信息、池子地址。第二，移除流动性实际上是两个操作，先减少流动性，再收集本金和费用，这个hook把它们封装在一起了。第三，支持签名函数。第四，提供Toast通知，显示收集到的代币数量。第五，成功后要刷新相关查询。第六，用React Query的useMutation来处理。

### 这里面有几个地方特别容易出错

这是两步操作，先decreaseLiquidity再collectFees，两个都要执行才能完整。amount0Min和amount1Min是滑点保护，设置成0容易被MEV攻击，但设太高又可能交易失败，要合理设置。签名函数必须有，没有签名的交易不会被网络接受。

### 数据刷新这里有讲究

用useMutation处理，能知道当前是否在处理中。成功后要立即invalidate positions和pool相关查询，数据变了就要刷新。这个hook和添加流动性是对应的，做好了用户就能完整地管理自己的流动性头寸。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useRemoveLiquidity } from '@sushiswap/hooks'

function RemoveLiquidityCard({ position }: { position: Position }) {
  const [liquidityToRemove, setLiquidityToRemove] = useState(position.liquidity)

  const { mutate: remove, isPending } = useRemoveLiquidity({
    onSuccess: (result) => {
      toast.success(
        `移除成功！获得 ${result.collectResult.amount0} Token0 和 ${result.collectResult.amount1} Token1`
      )
      queryClient.invalidateQueries(['positions'])
      queryClient.invalidateQueries(['pool-balances'])
    },
    onError: (error) => {
      toast.error(`移除失败: ${error.message}`)
    }
  })

  const handleRemove = () => {
    remove({
      tokenId: position.tokenId,
      liquidity: liquidityToRemove,
      amount0Min: 0n,
      amount1Min: 0n,
      token0: position.token0,
      token1: position.token1,
      poolAddress: position.pool,
    })
  }

  return (
    <div>
      <input
        type="range"
        min={0}
        max={position.liquidity}
        value={liquidityToRemove}
        onChange={(e) => setLiquidityToRemove(BigInt(e.target.value))}
      />
      <button onClick={handleRemove} disabled={isPending}>
        {isPending ? '移除中...' : '移除流动性'}
      </button>
    </div>
  )
}
```

### 常见使用场景

1. **完全移除头寸**：移除所有流动性并关闭头寸
   ```tsx
   remove({
     liquidity: position.liquidity,
     // ...
   })
   ```

2. **部分移除**：只移除部分流动性
   ```tsx
   const fraction = 0.5 // 50%
   const liquidityToRemove = position.liquidity * fraction
   remove({ liquidity: liquidityToRemove, ... })
   ```

3. **移除并收集**：移除流动性同时收集未收集的费用
   ```tsx
   const { mutate: removeAndCollect } = useRemoveLiquidity({
     onSuccess: (result) => {
       toast.success('本金和费用已收集！')
     }
   })
   ```

### Dos and Don'ts

**Dos:**
- ✅ 使用 `amount0Min/1Min` 提供滑点保护
- ✅ 成功后刷新 positions 和 balances 查询
- ✅ 支持部分移除

**Don'ts:**
- ❌ 不要设置为 0 的滑点保护（容易被 MEV 攻击）
- ❌ 不要在 isPending 时允许重复提交
- ❌ 不要忽略签名函数
