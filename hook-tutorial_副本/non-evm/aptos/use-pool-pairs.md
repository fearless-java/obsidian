> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/swap/lib/swap-get-route/use-pool-pairs.ts`

# usePoolPairs Hook Tutorial

## 大白话讲讲这个hook的作用

`usePoolPairs` 是一个用于获取池子交易对储备比例的 hook。它：

- 查询指定代币对的池子储备
- 计算储备比例（价格）

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **池子查询**：查询特定代币对的池子
2. **比例计算**：计算储备比例

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
- 从 `usePoolState` 获取 token0、token1

### 输出（返回值）
```typescript
{
  poolReserves: PoolReserve | null  // 池子储备
  poolPairRatio: number              // 储备比例
}
```

### 核心执行逻辑

1. **查询储备**：查询代币对的 TokenPairReserve 资源
2. **计算比例**：reserve1 / reserve0 或 reverse

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 usePoolPairs 的 React hook，用来获取池子里两个代币的储备比例。

核心需求：
1. 从池子状态里获取两个代币的信息
2. 查询这两个代币对的 TokenPairReserve 资源
3. 计算出两个代币的储备比例
4. 把计算出来的比例更新到池子状态里
5. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**处理代币顺序**：计算比例的时候要搞清楚是1个代币A等于多少个代币B，还是反过来。顺序搞反了，比例就变成倒数了。

### 这个Hook怎么管理状态

建议设置每8秒自动刷新一次。池子的储备比例会随着交易不断变化，所以要定期刷新保持数据新鲜。获取到新数据后，还要用副作用来更新池子状态。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { usePoolPairs } from '@sushiswap/aptos'

function PoolRatio() {
  // 使用 usePoolPairs 获取储备比例
  const { poolReserves, poolPairRatio, isLoading } = usePoolPairs()

  if (isLoading) return <div>加载中...</div>

  return (
    <div>
      <p>储备比例</p>
      <p>{poolPairRatio}</p>
    </div>
  )
}
```

### 常见用法

#### 1. 显示交易价格

```tsx
import { usePoolPairs } from '@sushiswap/aptos'

function TradingPrice() {
  // 使用 usePoolPairs 获取储备比例
  const { poolPairRatio, poolReserves, isLoading } = usePoolPairs()

  if (isLoading) return <div>计算价格中...</div>

  const price = poolPairRatio

  return (
    <div>
      <p>当前价格</p>
      <p>1 {poolReserves?.token0} = {price} {poolReserves?.token1}</p>
    </div>
  )
}
```

#### 2. 价格更新监控

```tsx
import { usePoolPairs } from '@sushiswap/aptos'
import { useState, useEffect } from 'react'

function PriceTicker() {
  // 使用 usePoolPairs 获取储备比例（自动刷新）
  const { poolPairRatio, poolReserves } = usePoolPairs()
  const [priceHistory, setPriceHistory] = useState<number[]>([])

  useEffect(() => {
    if (poolPairRatio) {
      setPriceHistory((prev) => [...prev.slice(-9), poolPairRatio])
    }
  }, [poolPairRatio])

  const priceChange = priceHistory.length > 1
    ? priceHistory[priceHistory.length - 1] - priceHistory[0]
    : 0

  return (
    <div>
      <p>当前价格: {poolPairRatio}</p>
      <p>8秒变化: {priceChange.toFixed(6)}</p>
    </div>
  )
}
```

#### 3. 储备更新检测

```tsx
import { usePoolPairs } from '@sushiswap/aptos'
import { useEffect, useRef } from 'react'

function ReserveMonitor() {
  const { poolReserves } = usePoolPairs()
  const prevRef = useRef(poolReserves)

  useEffect(() => {
    if (prevRef.current?.reserve0 !== poolReserves?.reserve0 ||
        prevRef.current?.reserve1 !== poolReserves?.reserve1) {
      console.log('储备发生变化')
    }
    prevRef.current = poolReserves
  }, [poolReserves])

  return (
    <div>
      <p>reserve0: {poolReserves?.reserve0}</p>
      <p>reserve1: {poolReserves?.reserve1}</p>
    </div>
  )
}
```

### Do（推荐做法）

- **使用 refetchInterval 自动刷新**：8秒刷新保持数据新鲜
- **组合 useEffect 监听变化**：检测储备变化
- **显示加载状态**：初始加载需要时间
- **考虑代币顺序**：比例计算要考虑 token0/token1 顺序

### Don't（不推荐做法）

- **不要忽略刷新间隔**：过长的间隔会导致数据过期
- **不要假设比例固定**：AMM 池子比例随时变化
- **不要忽略方向**：需要知道 1 token0 =多少 token1

### 相关的其他 hooks

- `usePool` *(一个React hook，用于获取单个流动性池信息，包括代币对、储备量、手续费率等)*：获取池子信息
- `usePoolsReserves` *(一个React hook，批量查询池子储备量，接受池子资源地址数组，返回每个池子的储备量)*：批量查询储备
- `useSwap` *(一个React hook，用于获取swap路由，根据输入输出代币和池子列表计算最优swap路径)*：获取 swap 路由
- `PoolPairReserve` *(一个TypeScript类型/配置，表示池子的交易对储备，包含reserve0、reserve1等字段)*：交易对储备类型
