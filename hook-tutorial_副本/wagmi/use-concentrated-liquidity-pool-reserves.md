> 源代码路径: `apps/web/src/lib/wagmi/hooks/pools/hooks/useConcentratedLiquidityPoolReserves.ts`

# useConcentratedLiquidityPoolReserves

## 大白话讲讲这个hook的作用

`useConcentratedLiquidityPoolReserves` 是一个查询 SushiSwap V3 池子实时准备金的 Hook。

大白话：就是查询"这个 V3 池子里两种币各有多少"。与 V2 的简单准备金不同，V3 的准备金计算更复杂，因为流动性集中在价格区间内。

它会返回两种代币在当前价格下的可用数量。

## 讲讲为什么需要封装该hook

1. **V3 特殊性**：V3 流动性是 token0/token1 的组合，不是简单的"各有多少"。

2. **动态计算**：需要根据当前 sqrtPrice 和 tick 范围计算可用数量。

3. **定时刷新**：V3 价格变化快，需要定时刷新。

4. **缓存**：使用 React Query 缓存结果。

5. **与池子关联**：依赖 `useConcentratedLiquidityPool` 的池子数据。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseConcentratedLiquidityPoolReserves {
  pool: V3Pool | null | undefined   // 池子实例
  chainId: SushiSwapV3ChainId
  enabled?: boolean
}
```

### 输出 (Outputs)

```typescript
{
  data: {
    amount0: Amount<EvmToken>,
    amount1: Amount<EvmToken>,
  } | null,
  isLoading: boolean,
  isError: boolean,
}
```

### 执行逻辑详解

#### 1. 查询函数

```typescript
return useQuery({
  queryKey: [
    'useConcentratedLiquidityPoolReserves',
    { token0: pool?.token0, token1: pool?.token1, feeAmount: pool?.swapFee, chainId },
  ],
  queryFn: async () => {
    if (pool) {
      return getConcentratedLiquidityPoolReserves({ pool, config })
    }
    return null
  },
  refetchInterval: 10000,
  enabled: Boolean(enabled && pool),
})
```

#### 2. getConcentratedLiquidityPoolReserves

实际计算逻辑在 `../actions/getConcentratedLiquidityPoolReserves`：
- 根据池子的当前 sqrtPrice
- 计算 token0 和 token1 的可用数量
- 返回 Amount 类型

### 数据流向图

```
输入: pool (V3Pool)
         ↓
    ┌────────────────────────────────────┐
    │  getConcentratedLiquidityPoolReserves│
    │  - 根据 sqrtPrice 计算             │
    │  - 返回 amount0, amount1          │
    └────────────────────────────────────┘
         ↓
    缓存到 queryKey
         ↓
    返回 { amount0, amount1 }
         ↑
    refetchInterval: 10000 定时刷新
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：必须先有池子实例**。这个hook是依赖useConcentratedLiquidityPool的。必须先拿到池子实例，才能查询这个池子的准备金。没有池子实例就没法查。

**第二点：10秒刷新是合理的**。V3池子的价格波动比较频繁，流动性变化也快。10秒刷新一次既能跟踪变化，又不会对RPC造成太大压力。

**第三点：返回的是可用数量不是总量**。这里说的amount0和amount1是在当前价格下可以立即使用的数量，不是池子里的总数量。因为V3的流动性集中在特定价格区间，有些仓位可能不在当前价格范围内。

**第四点：tick范围的影响**。如果你的仓位设定的价格范围不包括当前价格，那这个仓位提供的流动性在当前价格下是不可用的。这就是为什么查询到的可用数量可能是0。

**第五点：交易会锁定流动性**。当有swap交易发生时，会锁定部分流动性用于处理交易。这可能导致可用数量在短时间内减少。

### 约束条件要记住

**第一，需要池子实例**。必须先调用useConcentratedLiquidityPool拿到池子数据，才能用这个hook查询准备金。

**第二，计算依赖当前价格**。准备金数量是根据当前的sqrtPrice计算的。如果价格变了，同一个池子的可用数量也会变。

**第三，范围外的仓位会计为0**。如果一个仓位的tick范围不包括当前价格，那这个仓位贡献的流动性在计算可用数量时会被忽略。

**第四，只支持V3链**。V3池子和V2池子的计算方式不同。这个hook只适用于V3，不适用于V2。

### 状态管理要清楚

查询状态有三种。isLoading表示加载中，isError表示出错了，data是准备金数据，包含amount0和amount1两个Amount对象。如果pool为null，data也会是null。

刷新依赖两种方式。一种是定时刷新每10秒一次，另一种是依赖池子数据变化时自动刷新。

### 常见错误要避开

**第一个坑：完全依赖价格**。计算出来的数量完全取决于当前的sqrtPrice。如果价格波动大，每次查到的数量可能都不一样。这是V3的特性，不是bug。

**第二个坑：不是简单相加**。不是把池子里两种代币的总余额相加就完事了。要根据当前价格和你持有的流动性份额来按比例计算你实际能拿到多少。

**第三个坑：流动性范围**。只有当前价格落在仓位的tickLower和tickUpper范围内，这个仓位的流动性才算入可用数量。超出范围的仓位会被忽略。

**第四个坑：整数舍入**。计算过程中不可避免会有精度损失。大额交易的时候，这种舍入可能造成明显差异。

**第五个坑：闪电贷影响**。如果最近的交易里有闪电贷，池子的余额可能在瞬间变化很大。查询结果可能和实际情况有出入。

### 提示词模板

```markdown
帮我创建一个查询V3池子准备金的hook。

需求：
- 查询池子里两种代币的可用数量
- 根据当前价格计算，不是简单相加
- 定时刷新保持数据新鲜

技术栈：
- React + TypeScript
- wagmi v2 + viem
- @tanstack/react-query v5
- sushi库

输入：
- pool：来自useConcentratedLiquidityPool的池子实例
- chainId：链ID
- enabled：是否启用

输出：
- data：包含amount0和amount1的对象
- 为null表示没有池子数据

实现要点：
1. 依赖池子实例，需要先获取池子
2. 根据当前sqrtPrice计算可用数量
3. 只计算当前价格范围内的流动性
4. 定时刷新10秒一次

注意：
- 必须先有pool实例才能查询
- 数量会随价格变化
- 范围外的仓位不计入
- 只适用于V3池子
```

### 实际避坑指南

第一个，先获取池子。这个hook不能单独使用，必须依赖useConcentratedLiquidityPool先拿到池子实例。

第二个，价格影响大。V3的价格是实时变化的，所以每次查到的准备金数量可能都不同。这是正常现象。

第三个，不是总量。注意返回的是"可用数量"不是"总数量"。如果用户有仓位不在当前价格范围内，他们能用的数量可能是0。

第四个，多种刷新结合。定时刷新10秒一次能保证数据基本新鲜，但如果需要更实时的数据，可以结合其他刷新机制。

3. **流动性范围**：只有当前 tick 在流动性范围内的部分才计入。

4. **整数舍入**：计算时有精度损失，大额交易影响明显。

5. **Flash Loan 影响**：如果有 flash loan，余额可能瞬间变化。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useConcentratedLiquidityPoolReserves` 用于查询 V3 池子准备金。基本用法如下：

```typescript
import { useConcentratedLiquidityPoolReserves, useConcentratedLiquidityPool } from '@sushiswap/wag'

function PoolReserves({ pool }) {
  const { data: reserves, isLoading } = useConcentratedLiquidityPoolReserves({
    pool,
    chainId,
  })

  if (isLoading) return <div>加载中...</div>
  if (!reserves) return <div>无准备金</div>

  return (
    <div>
      <div>Token0: {reserves.amount0.toSignificant(6)}</div>
      <div>Token1: {reserves.amount1.toSignificant(6)}</div>
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景一：显示池子深度

```typescript
function PoolDepth({ pool }) {
  const { data: reserves } = useConcentratedLiquidityPoolReserves({
    pool,
    chainId,
  })

  if (!reserves) return <div>无池子</div>

  const totalValue = reserves.amount0.toUsdValue(price0)
    + reserves.amount1.toUsdValue(price1)

  return (
    <div>
      <div>池子深度: ${totalValue.toFixed(2)}</div>
      <div>{pool?.token0.symbol}: {reserves.amount0.toSignificant(4)}</div>
      <div>{pool?.token1.symbol}: {reserves.amount1.toSignificant(4)}</div>
    </div>
  )
}
```

#### 场景二：计算交易滑点

```typescript
function SlippageCalc({ pool, amountIn }) {
  const { data: reserves } = useConcentratedLiquidityPoolReserves({
    pool,
    chainId,
  })

  if (!reserves) return <div>计算中...</div>

  // 简单滑点估算（实际计算更复杂）
  const reserve0 = reserves.amount0.toBigInt()
  const reserve1 = reserves.amount1.toBigInt()
  const amountInInt = amountIn.toBigInt()

  // x * y = k 公式
  const amountOut = (reserve1 * amountInInt) / (reserve0 + amountInInt)

  const slippage = Number(amountOut) / Number(reserve1) * 100

  return (
    <div>预计滑点: {slippage.toFixed(2)}%</div>
  )
}
```

#### 场景三：获取全部费率池的准备金

```typescript
function AllPoolsReserves({ token0, token1 }) {
  const lowPool = useConcentratedLiquidityPool({
    token0, token1,
    feeAmount: SushiSwapV3FeeAmount.LOW,
  })

  const mediumPool = useConcentratedLiquidityPool({
    token0, token1,
    feeAmount: SushiSwapV3FeeAmount.MEDIUM,
  })

  const lowReserves = useConcentratedLiquidityPoolReserves({
    pool: lowPool.data,
    chainId,
  })

  const mediumReserves = useConcentratedLiquidityPoolReserves({
    pool: mediumPool.data,
    chainId,
  })

  return (
    <div>
      <div>0.05% 池: {lowReserves.data?.amount0.toSignificant(4)}</div>
      <div>0.3% 池: {mediumReserves.data?.amount0.toSignificant(4)}</div>
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **先检查 pool 是否存在**
   ```typescript
   enabled: Boolean(pool)
   // 或者
   if (!pool) return <div>无池子</div>
   ```

2. **使用 Amount 类型的方法**
   ```typescript
   // 正确：使用 Amount 的方法
   reserves.amount0.toSignificant(6)
   reserves.amount1.toUsdValue(price)

   // 错误：不要直接访问 .amount
   reserves.amount0.amount // 这是 bigint
   ```

3. **理解这是可用数量不是总数量**
   ```typescript
   // V3 返回的是当前价格下可用的数量
   // 流动性范围外的仓位不计入
   ```

#### ❌ Don'ts

1. **不要在没有 pool 时调用**
   ```typescript
   // 错误：pool 为 null/undefined 时返回 null
   const { data } = useConcentratedLiquidityPoolReserves({
     pool: undefined, // 错误
   })

   // 正确：先检查或等待 pool 加载
   if (!pool) return
   ```

2. **不要用于 V2 池子**
   ```typescript
   // 错误：这是 V3 专用的 hook
   const reserves = useConcentratedLiquidityPoolReserves({
     pool: v2Pool, // V2 池子不适用
   })

   // 正确：V2 使用 useSushiSwapV2Pools
   ```

3. **不要忽略定时刷新**
   ```typescript
   // 10 秒刷新是合理的
   // 不要改成过短间隔
   ```

4. **不要用于价格计算的主要来源**
   ```typescript
   // 准备金变化可能滞后
   // 价格应该用 pool.price 而不是 reserves 推算
   ```
