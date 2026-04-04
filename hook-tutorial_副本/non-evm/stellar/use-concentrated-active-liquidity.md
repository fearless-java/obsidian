> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/tick/use-concentrated-active-liquidity.ts`

# useConcentratedActiveLiquidity Hook Tutorial

## 大白话讲讲这个hook的作用

`useConcentratedActiveLiquidity` *(一个React hook，用于计算集中流动性图表数据，处理tick数据得到每个价格位置的活动流动性)* 是一个用于计算"集中流动性图表数据"的 hook。它处理 tick 数据，计算：

- 每个 tick 位置的活动流动性
- 流动性在不同价格范围的分布

这个 hook 是渲染流动性密度图表的核心数据来源。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **复杂算法**：需要遍历 tick 数据计算累积流动性
2. **数据转换**：将原始 tick 数据转换为图表需要的格式
3. **与 tick 数据解耦**：依赖 `useTicks` *(获取池子tick数据的hook)* 获取原始数据

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  pool: PoolInfo | null | undefined  // 池子信息
  enabled?: boolean                   // 是否启用
}
```

### 输出（返回值）
```typescript
{
  data: TickProcessed[]              // 处理后的 tick 数据
  isLoading: boolean
  activeTick: number
}
```

### TickProcessed 结构
```typescript
{
  tick: number                       // tick 索引
  liquidityActive: bigint            // 该位置的活动流动性
  liquidityNet: bigint               // 该 tick 的净流动性变化
  price0: string                    // 该 tick 的价格
}
```

### 核心执行逻辑

1. **获取 tick 数据**：使用 `useTicks` 获取原始 tick 数据
2. **找到活动 tick**：根据池子当前 tick 找到对应的 tick 索引
3. **累积计算**：从活动 tick 向两边遍历，累积流动性变化
4. **价格转换**：调用 `calculatePriceFromTick` *(将tick转换为价格)* 转换为价格字符串

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个计算集中流动性图表数据的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useConcentratedActiveLiquidity的hook，用来计算集中流动性的图表数据。然后明确几个关键点。第一，参数是池子信息，包括各种池子的核心参数。第二，依赖useTicks获取原始的tick数据，这是计算的基础。第三，从当前tick开始向两边遍历，累积计算每个tick位置的流动性变化，这个过程要搞清楚累加还是累减。第四，返回处理后的tick列表，每个tick包含当前位置的活跃流动性、净流动性变化和对应的价格。第五，用useMemo缓存计算结果，因为这个计算可能比较耗时。第六，用React Query管理tick数据的获取。

### 这里面有几个地方特别容易出错

tick遍历有方向讲究，向tick增加的方向是累加，向减少的方向是累减，这个要搞清楚。当前tick的处理要看它是不是在用户选择的范围内，如果在范围内处理方式不同。这个hook依赖pool.tick和tick数据，两个都要能用才能计算。

### 数据刷新这里有讲究

依赖useTicks的加载状态，那个hook数据没准备好这个也不能用。计算结果用useMemo缓存，避免每次渲染都重新算。这个hook是给图表用的，计算量可能比较大，做好缓存很重要。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useConcentratedActiveLiquidity } from '@sushiswap/hooks'

function LiquidityChart({ pool }: { pool: PoolInfo }) {
  const { data: tickData, activeTick } = useConcentratedActiveLiquidity({
    pool,
  })

  return (
    <div>
      <LiquidityChartComponent data={tickData} activeTick={activeTick} />
    </div>
  )
}
```

### 常见使用场景

1. **流动性密度图表**：显示不同价格范围的流动性分布
   ```tsx
   const { data: processedTicks } = useConcentratedActiveLiquidity({ pool })
   return <AreaChart data={processedTicks} />
   ```

2. **活动 tick 高亮**：在图表上标记当前价格位置
   ```tsx
   const { activeTick } = useConcentratedActiveLiquidity({ pool })
   return <Chart><CurrentPriceMarker tick={activeTick} /></Chart>
   ```

3. ** Tick 范围选择参考**：帮助用户选择合适的 tick 范围
   ```tsx
   const { data: ticks } = useConcentratedActiveLiquidity({ pool })
   const highLiquidityTicks = ticks?.filter(t => t.liquidityActive > threshold)
   ```

### Dos and Don'ts

**Dos:**
- ✅ 配合 `useDensityChartData` *(准备图表数据的hook)* 使用
- ✅ 使用 `useMemo` 缓存计算结果
- ✅ 正确处理 tick 遍历方向

**Don'ts:**
- ❌ 不要在池子数据未加载时调用
- ❌ 不要忽略加载状态
- ❌ 不要直接修改返回的数据
