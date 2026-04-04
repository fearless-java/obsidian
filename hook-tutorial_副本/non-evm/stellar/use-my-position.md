> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/position/use-my-position.ts`

# useMyPosition Hook Tutorial

## 大白话讲讲这个hook的作用

`useMyPosition` *(一个React hook，用于获取用户在特定池子中的流动性头寸汇总，聚合头寸、本金、费用和池子信息)* 是一个用于获取用户在池子中的流动性头寸汇总的 hook。它聚合了：

- 用户在该池子中的所有流动性头寸
- 每个头寸的本金（principal）数量
- 每个头寸的未收集费用（fees）
- 关联的池子信息

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **多数据源聚合**：需要组合用户头寸数据、池子数据、代币信息、费用数据等
2. **批量查询优化**：使用 `useQueries` *(React Query的hook，用于批量执行多个查询)* 批量获取多个池子的数据
3. **分组处理**：按池子分组批量获取 principal 数据
4. **汇总计算**：计算总本金和总费用

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  userAddress?: string           // 用户钱包地址
  poolAddress?: string          // 可选：只获取特定池子的头寸
  excludeDust?: boolean         // 是否排除Dust（小额）头寸
}
```

### 输出（返回值）
```typescript
{
  data: {
    positions: PositionSummary[]  // 头寸汇总列表
    isLoading: boolean
    error: Error | null
  }
}

interface PositionSummary {
  tokenId: number
  token0: Token
  token1: Token
  liquidity: string
  principalToken0: bigint
  principalToken1: bigint
  feesToken0: bigint
  feesToken1: bigint
  pool: string
  tickLower: number
  tickUpper: number
  fee: number
}
```

### 核心执行逻辑

1. **获取用户头寸**：调用 `useUserPositions` *(获取用户所有流动性头寸的hook)* 获取用户所有头寸
2. **获取关联池子**：对每个头寸查找对应的池子地址
3. **批量获取本金**：按池子分组，批量调用 `getPositionsPrincipalBatch` *(批量获取头寸本金的合约方法)*
4. **汇总数据**：合并头寸信息、池子信息、本金数据

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个获取用户流动性头寸汇总的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useMyPosition的hook，用来获取用户在某个区块链池子中的流动性头寸汇总。然后明确几个关键点。第一，参数要包括用户钱包地址，还可以选择只获取特定池子的头寸。第二，获取用户所有的头寸数据。第三，获取每个头寸关联的池子信息，因为头寸是挂在池子上的。第四，批量获取每个头寸的本金数量，这需要调用专门的合约方法。第五，汇总所有信息返回头寸汇总列表，包含头寸ID、代币信息、流动性数量、本金、费用、关联池子、手续费率等。第六，支持按池子过滤，方便只看某个特定池子的头寸。第七，用useQueries来优化批量查询，比一个个用useQuery要高效很多。

### 这里面有几个地方特别容易出错

数据流比较复杂，涉及多个数据源和多次查询，所以每一步都要注意处理null的情况。用useQueries批量查询的时候要按池子分组，把同一个池子的头寸放在一起查，减少合约调用次数。各种数据可能为null，比如池子信息可能还没加载完，这种情况要处理妥当。

### 数据刷新这里有讲究

头寸数据相对稳定，变化不频繁，所以staleTime可以设成1分钟。用useQueries代替多个useQuery，这样能大幅提高性能，特别是在用户头寸比较多的时候。这个hook涉及多个数据源的聚合，做好了能大大简化上层组件的逻辑。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useMyPosition } from '@sushiswap/hooks'
import { useStellarWallet } from './wallet'

function MyPositions() {
  const { address } = useStellarWallet()
  const { data, isLoading } = useMyPosition({ userAddress: address })

  if (isLoading) return <p>加载中...</p>

  return (
    <div>
      {data?.positions.map((pos) => (
        <PositionCard key={pos.tokenId} position={pos} />
      ))}
    </div>
  )
}
```

### 常见使用场景

1. **钱包持仓概览**：显示用户所有头寸汇总
   ```tsx
   const { data } = useMyPosition({ userAddress: address })
   const totalFees = data?.positions.reduce((sum, p) => sum + p.feesToken0, 0n)
   ```

2. **单个池子头寸**：获取用户在特定池子的头寸
   ```tsx
   const { data } = useMyPosition({
     userAddress: address,
     poolAddress: specificPool,
   })
   ```

3. **收集费用按钮**：显示可收集的费用
   ```tsx
   {position.feesToken0 > 0n && (
     <CollectFeesButton tokenId={position.tokenId} />
   )}
   ```

### Dos and Don'ts

**Dos:**
- ✅ 配合 `useCollectFees` *(执行收集费用操作的hook)* 收集费用
- ✅ 使用 `excludeDust` 过滤小额头寸
- ✅ 汇总计算总本金和总费用

**Don'ts:**
- ❌ 不要假设用户一定有头寸
- ❌ 不要忽略 isLoading 状态
- ❌ 不要直接比较 bigint 和数字
