> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/position/use-position-principal.ts`

# usePositionPrincipal Hook Tutorial

## 大白话讲讲这个hook的作用

`usePositionPrincipal` *(一个React hook，用于获取流动性头寸的实际本金数量，与用户投入的数量可能不同因为费用累积会影响)* 是一个用于获取头寸"本金"代币数量的 hook。它返回头寸中实际存放的：

- Token0 数量
- Token1 数量

这与用户添加流动性时投入的数量可能不同（因为费用累积会影响）。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **合约调用封装**：需要调用 PositionService *(头寸服务，提供头寸相关的合约交互)* 获取本金数据
2. **React Query 集成**：管理数据获取和缓存

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
tokenId: number | undefined    // 头寸 ID
```

### 输出（返回值）
```typescript
{
  data: {
    amount0: bigint            // Token0 本金
    amount1: bigint             // Token1 本金
  }
}
```

### 核心执行逻辑

1. **调用服务**：使用 `positionService.getPositionPrincipal(tokenId)` *(PositionService方法，获取头寸的本金数量)* 获取数据
2. **返回结果**：返回本金数量

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个获取头寸本金数量的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似usePositionPrincipal的hook，用来获取某个区块链上流动性头寸的实际本金数量。然后明确几个关键点。第一，参数是头寸ID，tokenId，没有这个就没法查具体是哪个头寸。第二，调用positionService来获取头寸的本金数据，这个服务封装了合约调用。第三，返回本金数量，包括amount0和amount1，分别是对应两种代币的本金数量。第四，用React Query管理数据。

### 这里面有几个地方特别容易出错

参数校验一定要做，tokenId是undefined的时候不能发起查询，查了也是白查还容易出错。本金数量和用户当初投入的数量可能不一样，因为费用会累积进来，这个要心里有数。

### 数据刷新这里有讲究

staleTime可以设成30秒，不必要频繁刷新，retry可以设成1次，失败了再试一次。这个hook涉及合约调用，所以可能会失败，重试机制要有。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { usePositionPrincipal } from '@sushiswap/hooks'

function PositionCard({ tokenId }: { tokenId: number }) {
  const { data: principal } = usePositionPrincipal(tokenId)

  return (
    <div>
      <p>本金 Token0: {principal?.amount0.toString()}</p>
      <p>本金 Token1: {principal?.amount1.toString()}</p>
    </div>
  )
}
```

### 常见使用场景

1. **显示头寸详情**：展示头寸的实际本金
   ```tsx
   const { data: principal } = usePositionPrincipal(position.tokenId)
   ```

2. **计算盈亏**：比较本金与当前价值
   ```tsx
   const pnl = currentValue - (principal?.amount0 + principal?.amount1)
   ```

3. **移除流动性预览**：预览移除后能拿回多少
   ```tsx
   const { data: principal } = usePositionPrincipal(tokenId)
   const shareRatio = liquidityToRemove / position.liquidity
   const token0ToReceive = principal?.amount0 * shareRatio
   ```

### Dos and Don'ts

**Dos:**
- ✅ 配合 `usePosition` *(获取单个头寸详情)* 使用
- ✅ 使用 `useMemo` 缓存计算结果
- ✅ 检查 tokenId 有效性后再查询

**Don'ts:**
- ❌ 不要在 tokenId 为 undefined 时调用
- ❌ 不要忽略 bigint 与数字的比较
- ❌ 不要假设本金与投入相等（费用会累积）
