> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/position/use-position-active-liquidity.ts`

# usePositionActiveLiquidity Hook Tutorial

## 大白话讲讲这个hook的作用

`usePositionActiveLiquidity` *(一个React hook，用于计算头寸在当前价格下的活动流动性，决定头寸能获得多少费用分成)* 是一个用于计算头寸活动流动性的 hook。它计算：

- 当前价格下，头寸的流动性中有多少是"活跃"的
- 这个数值与池子总活跃流动性的比例决定头寸能获得多少费用分成

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **复杂数学计算**：需要调用多个 Soroban 合约函数
2. **价格依赖**：需要获取池子当前 sqrtPrice *(当前价格)*
3. **tick 范围计算**：根据 tickLower、tickUpper 计算活动流动性

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  poolAddress: string | null       // 池子地址
  scaledAmount0: bigint            // Token0 数量（已缩放）
  scaledAmount1: bigint            // Token1 数量（已缩放）
  tickLower: number               // 价格范围下限
  tickUpper: number               // 价格范围上限
}
```

### 输出（返回值）
```typescript
{
  data: bigint                     // 活动流动性数量
}
```

### 核心执行逻辑

1. **获取当前价格**：调用 `getCurrentSqrtPrice(poolAddress)` *(Soroban合约方法，获取池子当前价格)*
2. **计算边界价格**：使用 `tickToSqrtPrice` *(将tick索引转换为价格)* 计算 tick 对应的价格
3. **计算活动流动性**：调用 `calculateActiveLiquidity` *(计算活动流动性的数学函数)* 计算

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个计算头寸活动流动性的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似usePositionActiveLiquidity的hook，用来计算头寸在当前价格下的活动流动性。然后明确几个关键点。第一，参数要包括池子地址、头寸的代币数量、tick范围上下限。第二，获取池子的当前价格，因为活动流动性取决于当前价格相对于用户选择的价格范围。第三，根据当前价格和tick范围计算出头寸中真正处于活跃状态的流动性数量，这个值决定了头寸能获得多少费用分成。第四，返回活动流动性的数值。第五，用React Query管理数据。

### 这里面有几个地方特别容易出错

参数里的代币数量是已缩放的bigint，不是人类能看懂的格式，传入之前要确认已经做过缩放处理。依赖池子的当前价格，没有价格就算不出活动流动性。这个计算涉及一些数学运算，要把逻辑搞清楚再写。

### 数据刷新这里有讲究

价格变化会影响计算结果，所以staleTime要设短一点，比如10秒，超过10秒就要重新算了。这个hook做的是计算而不是查询，依赖于池子价格数据。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { usePositionActiveLiquidity } from '@sushiswap/hooks'

function PositionDetails({ position }: { position: Position }) {
  const { data: activeLiquidity } = usePositionActiveLiquidity({
    poolAddress: position.pool,
    scaledAmount0: position.scaledAmount0,
    scaledAmount1: position.scaledAmount1,
    tickLower: position.tickLower,
    tickUpper: position.tickUpper,
  })

  return (
    <p>活动流动性: {activeLiquidity?.toString()}</p>
  )
}
```

### 常见使用场景

1. **费用分成计算**：计算头寸能获得多少费用
   ```tsx
   const { data: activeLiquidity } = usePositionActiveLiquidity(...)
   const shareRatio = activeLiquidity / pool.totalActiveLiquidity
   const feesEarned = totalFees * shareRatio
   ```

2. **头寸健康度评估**：活动流动性占总流动性的比例
   ```tsx
   const healthRatio = activeLiquidity / position.liquidity
   if (healthRatio < 0.5) {
     toast.warning('头寸大部分不在当前价格范围内')
   }
   ```

3. **价格范围建议**：基于活动流动性推荐调整范围
   ```tsx
   if (activeLiquidity === 0n) {
     return <AdjustRangePrompt currentTick={pool.tick} />
   }
   ```

### Dos and Don'ts

**Dos:**
- ✅ 使用 `usePoolInfo` *(获取池子详细信息的hook)* 获取池子价格数据
- ✅ 设置较短的 `staleTime` 因为价格变化会影响结果
- ✅ 参数传入已缩放的数量（scaledAmount）

**Don'ts:**
- ❌ 不要传入人类可读格式的数量
- ❌ 不要忽略 tick 范围的有效性检查
- ❌ 不要在价格变化时不重新计算
