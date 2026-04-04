> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/tick/use-density-chart-data.ts`

# useDensityChartData Hook Tutorial

## 大白话讲讲这个hook的作用

`useDensityChartData` *(一个React hook，用于准备流动性密度图表的数据，将处理后的tick数据转换为图表组件需要的格式)* 是一个用于为流动性密度图表准备数据的 hook。它：

- 接收池子信息
- 调用 `useConcentratedActiveLiquidity` *(获取处理后tick数据的hook)* 获取处理后的 tick 数据
- 将数据转换为图表组件需要的格式（ChartEntry[]）

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **数据格式转换**：将 tick 数据转换为 UI 图表需要的格式
2. **组件解耦**：与具体的图表组件类型（ChartEntry）解耦
3. **计算复用**：复用 `useConcentratedActiveLiquidity` 的计算结果

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  pool: PoolInfo | null | undefined   // 池子信息
  enabled?: boolean                   // 是否启用
}
```

### 输出（返回值）
```typescript
{
  data: ChartEntry[]                // 图表数据
  isLoading: boolean
  error: Error | null
}
```

### ChartEntry 结构
```typescript
{
  activeLiquidity: number         // 活动流动性
  price0: number                  // 价格
}
```

### 核心执行逻辑

1. **获取活动流动性数据**：调用 `useConcentratedActiveLiquidity`
2. **格式转换**：将 `TickProcessed[]` 转换为 `ChartEntry[]`
3. **数值转换**：将 bigint 转为 number 便于图表渲染

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个准备流动性密度图表数据的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useDensityChartData的hook，用来准备流动性密度图表需要的数据。然后明确几个关键点。第一，参数是池子信息，池子不同图表数据就不同。第二，调用useConcentratedActiveLiquidity获取处理过的tick数据，这是核心数据源。第三，把这些数据转换成图表组件需要的格式，比如活动流动性和对应的价格。第四，用useMemo缓存转换结果，因为数据转换可能有点耗时。第五，返回React Query的状态，这样组件里能知道数据加载情况。

### 这里面有几个地方特别容易出错

bigint要转成number，因为大多数图表库不支持bigint，直接传进去会出问题。转换的时候要确保liquidityActive数据是有效的，无效的数据要过滤掉。

### 数据刷新这里有讲究

继承useConcentratedActiveLiquidity的状态，那个hook数据怎么样了这边就怎么样。用useMemo避免不必要的重新计算，这个转换不是免费的，能缓存就缓存。这个hook主要是做数据格式转换，把通用数据转成图表组件需要的格式。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useDensityChartData } from '@sushiswap/hooks'
import { LiquidityChart } from './components'

function PoolChart({ pool }: { pool: PoolInfo }) {
  const { data: chartData, isLoading } = useDensityChartData({ pool })

  if (isLoading) return <p>加载图表数据...</p>

  return (
    <LiquidityChart
      data={chartData}
      currentPrice={pool.sqrtPriceX96}
      tickLower={userSelectedLower}
      tickUpper={userSelectedUpper}
    />
  )
}
```

### 常见使用场景

1. **流动性深度图表**：显示不同价格范围的流动性深度
   ```tsx
   const { data } = useDensityChartData({ pool })
   return <AreaChart data={data} />
   ```

2. **范围选择可视化**：帮助用户可视化选择的价格范围
   ```tsx
   const { data: chartData } = useDensityChartData({ pool })
   return (
     <Chart>
       <LiquidityCurve data={chartData} />
       <RangeSelector lower={tickLower} upper={tickUpper} />
     </Chart>
   )
   ```

3. **价格范围建议**：基于图表数据推荐范围
   ```tsx
   const { data } = useDensityChartData({ pool })
   const maxLiquidityTick = data?.reduce((max, t) =>
     t.activeLiquidity > max.activeLiquidity ? t : max
   )
   ```

### Dos and Don'ts

**Dos:**
- ✅ 使用 `useMemo` 缓存转换结果避免重复计算
- ✅ bigint 正确转换为 number
- ✅ 配合 `useConcentratedActiveLiquidity` 确保数据正确

**Don'ts:**
- ❌ 不要在池子未选择时显示空图表
- ❌ 不要忽略图表数据的加载状态
- ❌ 不要直接修改返回的图表数据
