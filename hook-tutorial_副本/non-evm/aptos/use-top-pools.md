> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/pool/lib/use-top-pools.ts`

# useTopPools Hook Tutorial

## 大白话讲讲这个hook的作用

`useTopPools` 是一个用于获取热门池子列表的 hook。它从 SushiSwap 数据 API 获取：

- 交易量最高的池子
- 包含 TVL、交易量等数据

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **数据 API 封装**：封装 graph-client 的调用
2. **代币信息补全**：补全池子的代币信息

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
enabled?: boolean        // 是否启用
```

### 输出（返回值）
```typescript
{
  data: TopPool[]       // 热门池子列表
}
```

### TopPool 结构
```typescript
{
  address: string
  token0: Token
  token1: Token
  volumeUSD1d: number
  totalApr1d: number
  // ...
}
```

### 核心执行逻辑

1. **获取热门池子**：调用 `getTopNonEvmPools`
2. **补全代币**：根据代币地址获取代币信息

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useTopPools 的 React hook，用来获取热门的池子列表。

核心需求：
1. 从数据接口获取交易量最大的那些池子
2. 把每个池子缺少的代币信息补全
3. 返回包含完整信息的热门池子列表
4. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**代币信息要补全**：从接口拿到的池子数据可能只有代币地址，缺少代币的详细信息比如名称、符号、小数位等。这些信息要从代币列表里找到对应的代币来补全。

### 这个Hook怎么管理状态

热门池子列表变化不频繁，因为交易量排名不会时刻都在变。所以可以设置较长的缓存时间，不用频繁地去请求数据。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useTopPools } from '@sushiswap/aptos'

function TopPoolsList() {
  // 使用 useTopPools 获取热门池子
  const { data: pools, isLoading } = useTopPools()

  if (isLoading) return <div>加载中...</div>

  return (
    <div>
      {pools?.map((pool) => (
        <div key={pool.address}>
          <span>{pool.token0.symbol} / {pool.token1.symbol}</span>
          <span>交易量: ${pool.volumeUSD1d.toFixed(2)}</span>
          <span>APR: {pool.totalApr1d.toFixed(2)}%</span>
        </div>
      ))}
    </div>
  )
}
```

### 常见用法

#### 1. 显示热门池子排行榜

```tsx
import { useTopPools } from '@sushiswap/aptos'

function TopPoolsRanking() {
  // 使用 useTopPools 获取热门池子
  const { data: pools } = useTopPools()

  // 按交易量排序（可能已排序，确保一下）
  const sortedPools = pools?.sort((a, b) => b.volumeUSD1d - a.volumeUSD1d)

  return (
    <div>
      <h3>热门池子排行榜</h3>
      {sortedPools?.slice(0, 10).map((pool, i) => (
        <div key={pool.address} className="rank-item">
          <span className="rank">#{i + 1}</span>
          <span>{pool.token0.symbol} / {pool.token1.symbol}</span>
          <span>24h 交易量: ${pool.volumeUSD1d.toFixed(0)}</span>
          <span>APR: {pool.totalApr1d.toFixed(2)}%</span>
        </div>
      ))}
    </div>
  )
}
```

#### 2. 显示高收益池子

```tsx
import { useTopPools } from '@sushiswap/aptos'

function HighYieldPools() {
  // 使用 useTopPools 获取热门池子
  const { data: pools } = useTopPools()

  // 按 APR 排序
  const highYieldPools = pools
    ?.filter((pool) => pool.totalApr1d > 0)
    .sort((a, b) => b.totalApr1d - a.totalApr1d)
    .slice(0, 5)

  return (
    <div>
      <h3>高收益池子</h3>
      {highYieldPools?.map((pool) => (
        <div key={pool.address}>
          <span>{pool.token0.symbol} / {pool.token1.symbol}</span>
          <span className="yield">APR: {pool.totalApr1d.toFixed(2)}%</span>
        </div>
      ))}
    </div>
  )
}
```

#### 3. 组合 TVL 和交易量显示

```tsx
import { useTopPools, usePools } from '@sushiswap/aptos'

function PoolAnalytics() {
  // 获取热门池子
  const { data: topPools } = useTopPools()
  // 获取全部池子 TVL
  const { data: allPools } = usePools()

  // 计算热门池子在总 TVL 中的占比
  const topPoolAddresses = new Set(topPools?.map((p) => p.address))
  const topPoolsTVL = allPools
    ?.filter((p) => topPoolAddresses.has(p.address))
    .reduce((sum, p) => sum + Number(p.reserve0) + Number(p.reserve1), 0)

  return (
    <div>
      <h3>热门池子概览</h3>
      <div>热门池子数量: {topPools?.length}</div>
      <div>24h 总交易量: ${topPools?.reduce((sum, p) => sum + p.volumeUSD1d, 0).toFixed(2)}</div>
    </div>
  )
}
```

### Do（推荐做法）

- **按交易量或 APR 排序**：帮助用户发现优质池子
- **设置合适的缓存时间**：热门池子不需要频繁刷新
- **过滤无效 APR**：APR 为 0 的池子可能有问题
- **组合 usePools 获取完整 TVL**：计算热门池子总 TVL

### Don't（不推荐做法）

- **不要每次都获取完整列表**：使用缓存避免频繁请求
- **不要忽略 isLoading**：数据来自 API，需要时间
- **不要假设所有池子都有 APR**：新池子可能 APR 为 0

### 相关的其他 hooks

- `usePools` *(一个React hook，用于获取网络中所有流动性池，从合约资源中读取所有交易对信息)*：获取所有池子
- `usePoolsExtended` *(一个React hook，获取带USD价值的池子列表，扩展了usePools，添加USD价值计算)*：带 USD 价值的池子
- `getTopNonEvmPools` *(一个API方法/函数，从SushiSwap数据API获取热门非EVM链池子列表，包含交易量、APR等数据)*：获取热门池子 API
- `TopPool` *(一个TypeScript类型/配置，表示热门池子的数据结构，包含address、token0、token1、volumeUSD1d、totalApr1d等字段)*：热门池子类型
