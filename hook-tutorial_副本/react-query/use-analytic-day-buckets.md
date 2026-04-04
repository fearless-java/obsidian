> 源代码路径: `apps/web/src/lib/hooks/react-query/explore/use-analytics-day-buckets.ts`

# useAnalyticDayBuckets Hook Tutorial

## 1. 大白话讲讲这个hook的作用

`useAnalyticDayBuckets` 是用来获取 SushiSwap 各链上的分析数据时间桶（Day Buckets）的 Hook。这些数据通常用于展示交易量、流动性等历史数据的时间序列图表，帮助用户了解池子或协议的历史表现。

简单来说：**就是获取"某个链上的历史数据分析数据"——可以理解为是一个按天聚合的时间序列数据，用于画图表、做分析。**

---

## 2. 为什么需要封装该hook

### 原始问题
1. **链 ID 校验复杂**：不是所有链都支持这种分析查询，需要 `isPoolChainId` 检查
2. **依赖 Graph Client**：数据来自 `@sushiswap/graph-client/data-api` 的 `getAnalyticsDayBuckets`
3. **定时刷新**：分析数据15分钟刷新一次比较合理
4. **启用条件检查**：需要同时满足 enabled 和 isPoolChainId(chainId)

### 封装带来的好处
1. **参数校验前置**：在 queryFn 内部检查 chainId 有效性并抛出明确错误
2. **封装 Graph Client 调用**：把对 Graph Client 函数的调用封装在 Hook 内
3. **缓存配置优化**：15 分钟刷新间隔适合分析数据的更新频率

---

## 3. 执行逻辑和数据流向

### 输入 (Params)
```typescript
{
  chainId: ChainId       // 链ID
  enabled: boolean       // 是否启用
}
```

### 输出 (Return)
```typescript
{
  data: DayBucket[]  // 取决于 getAnalyticsDayBuckets 的返回类型
  isLoading: boolean
  isError: boolean
  // ...
}
```

### 执行流程

```
1. useAnalyticsDayBuckets({ chainId, enabled })
       |
       v
2. 检查 enabled && isPoolChainId(chainId)
       |
       v
3. 调用 getAnalyticsDayBuckets({ chainId })
   - 这是 @sushiswap/graph-client/data-api 的函数
       |
       v
4. 返回数据
```

---

## 4. 怎么用 AI 写这个 Hook

### 写代码时要注意什么

1. **参数校验要在 queryFn 里面做**：传给查询函数的参数对查询结果有影响，所以在 queryFn 内部要再做一次最终校验。

2. **校验失败要抛明确错误**：如果参数不对，要抛出一个能看懂的错误消息，方便排查问题。

3. **缓存间隔要配置好**：refetchInterval 设成 15 分钟就够了，分析数据不用刷新那么频繁。

4. **enabled 检查要做好**：确保整个查询可以被外面的人关掉。

### 有什么限制条件

1. **chainId 必须支持分析功能**：要用 `isPoolChainId` 检查一下，不是所有链都能查分析数据。

2. **依赖 Graph Client**：底层调用的是 `@sushiswap/graph-client/data-api` 的 `getAnalyticsDayBuckets`，这个库要有。

3. **enabled 是必传的**：这个参数不能省略，必须显式传入。

### 状态怎么管理

| 状态 | 怎么管 | 什么意思 |
|------|--------|---------|
| 分析数据 | React Query 缓存 | 按 chainId 缓存 |
| 定时刷新 | refetchInterval: 15m | 15分钟自动刷新一次 |
| 启用开关 | enabled && isPoolChainId | 两个条件同时满足才查 |

---

### 完整AI提示词模板

```
你是一个 React Query + 数据分析专家。请为以下场景编写 Hook:

【场景描述】
需要获取 SushiSwap 各链上的分析数据时间桶（Day Buckets），
这些数据用于展示交易量、流动性等历史数据的时间序列。

【技术要求】
1. 使用 @tanstack/react-query useQuery
2. 使用 @sushiswap/graph-client/data-api 的 getAnalyticsDayBuckets
3. 链 ID 必须通过 isPoolChainId 检查

【参数】
{
  chainId: ChainId
  enabled: boolean
}

【校验】
if (!chainId) throw new Error('chainId is required')
if (!isPoolChainId(chainId)) throw new Error('Invalid pool chainId')

【缓存配置】
- refetchInterval: 15 分钟 (ms('15m'))
- enabled: Boolean(enabled && isPoolChainId(chainId))

【Graph Client 集成】
import { getAnalyticsDayBuckets, isPoolChainId } from '@sushiswap/graph-client/data-api'

【最佳实践】
- 参数校验在 queryFn 内部
- 明确错误消息
- 使用 ms() 处理时间字符串

请输出完整代码。
```

---

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useAnalyticDayBuckets } from '@sushiswap/react-query'

function VolumeChart({ chainId }: { chainId: number }) {
  const { data: dayBuckets, isLoading } = useAnalyticDayBuckets({
    chainId,
    enabled: true,
  })

  if (isLoading) return <ChartSkeleton />
  if (!dayBuckets?.length) return <EmptyState>No analytics data</EmptyState>

  return (
    <ResponsiveContainer width="100%" height={400}>
      <LineChart data={dayBuckets}>
        <XAxis dataKey="date" />
        <YAxis />
        <Tooltip />
        <Line type="monotone" dataKey="volumeUSD" stroke="#8884d8" />
      </LineChart>
    </ResponsiveContainer>
  )
}
```

### 常见使用场景

**场景1：TVL 历史数据展示**
```tsx
function TVLHistoryChart({ chainId }: { chainId: number }) {
  const { data: dayBuckets } = useAnalyticDayBuckets({
    chainId,
    enabled: true,
  })

  const chartData = dayBuckets?.map((bucket) => ({
    date: bucket.date,
    tvl: bucket.tvlUSD,
  }))

  return (
    <AreaChart data={chartData}>
      <XAxis dataKey="date" />
      <YAxis />
      <Tooltip formatter={(value) => `$${Number(value).toLocaleString()}`} />
      <Area
        type="monotone"
        dataKey="tvl"
        stroke="#82ca9d"
        fill="#82ca9d"
        fillOpacity={0.3}
      />
    </AreaChart>
  )
}
```

**场景2：多指标综合分析面板**
```tsx
function AnalyticsDashboard({ chainId }: { chainId: number }) {
  const { data: dayBuckets } = useAnalyticDayBuckets({
    chainId,
    enabled: true,
  })

  const stats = useMemo(() => {
    if (!dayBuckets?.length) return null

    const totalVolume = dayBuckets.reduce((sum, b) => sum + Number(b.volumeUSD), 0)
    const avgTVL = dayBuckets.reduce((sum, b) => sum + Number(b.tvlUSD), 0) / dayBuckets.length
    const lastBucket = dayBuckets[dayBuckets.length - 1]

    return {
      totalVolume,
      avgTVL,
      currentTVL: lastBucket?.tvlUSD,
      dataPoints: dayBuckets.length,
    }
  }, [dayBuckets])

  return (
    <Dashboard>
      <StatCard label="Total Volume" value={stats?.totalVolume} />
      <StatCard label="Average TVL" value={stats?.avgTVL} />
      <StatCard label="Current TVL" value={stats?.currentTVL} />
      <StatCard label="Data Points" value={stats?.dataPoints} />
    </Dashboard>
  )
}
```

### Dos and Don'ts

**Do（推荐做法）：**
- ✅ 使用 `isPoolChainId` 验证 chainId 是否支持分析数据
- ✅ 使用 `enabled` 参数控制是否启用查询
- ✅ 使用 `dayBuckets?.length` 检查数据是否存在
- ✅ 使用 `useMemo` 计算聚合统计数据

**Don't（避免做法）：**
- ❌ 不要假设所有链都支持分析数据，应该先检查 isPoolChainId
- ❌ 不要在没有数据时渲染图表，应该显示空状态
- ❌ 不要设置过短的 refetchInterval，15分钟已经足够
- ❌ 不要忽略 isLoading 状态，应该显示加载状态

### 注意事项

1. **需要 isPoolChainId 验证**：不是所有链都支持分析数据查询

2. **返回数组结构**：每个元素是一个时间桶，包含该日期的各种分析指标

3. **15分钟刷新**：分析数据不需要实时刷新，15分钟是合理的间隔

4. **数据类型是 DayBucket**：具体字段取决于 Graph Client 的返回类型，通常包括 date、volumeUSD、tvlUSD 等

5. **enabled 必须为 true**：这个 hook 要求 enabled 参数是必需的
