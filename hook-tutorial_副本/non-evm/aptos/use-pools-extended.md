> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/pool/lib/use-pools-extended.ts`

# usePoolsExtended Hook Tutorial

## 大白话讲讲这个hook的作用

`usePoolsExtended` 是一个用于获取带 USD 价值的池子列表的 hook。它扩展了 `usePools`：

- 添加 USD 价值计算
- 合并基础代币信息

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **数据组合**：组合池子、代币、价格数据
2. **USD 计价**：计算每个池子的 USD 总价值

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
- 无参数

### 输出（返回值）
```typescript
{
  data: PoolExtended[]    // 带 USD 价值的池子列表
  isLoading: boolean
}
```

### PoolExtended 结构
```typescript
{
  // ...Pool 基本字段
  token0: Token
  token1: Token
  reserve0USD: number    // USD 价值
  reserve1USD: number
  reserveUSD: number     // 总价值
}
```

### 核心执行逻辑

1. **获取池子**：使用 `usePools`
2. **获取价格**：使用 `useStablePrices`
3. **计算 USD**：储备量 * 价格

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 usePoolsExtended 的 React hook，用来获取带美元价值的池子列表。

核心需求：
1. 要组合多个数据来源：池子信息、代币价格、基础代币列表
2. 计算出每个池子的美元总价值
3. 返回包含了美元价值的池子列表
4. 用缓存来优化计算，避免重复计算

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**依赖多个数据源**：这个hook需要好几个其他的数据都准备好才能正常工作。比如池子信息、代币价格等，哪个没准备好都没法完成计算。

### 这个Hook怎么管理状态

建议用缓存来保存计算结果。因为池子数据量可能很大，而且美元价值的计算涉及到价格数据和池子数据的组合计算，比较耗时。用缓存可以避免每次渲染都重新计算。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { usePoolsExtended } from '@sushiswap/aptos'

function PoolListWithUSD() {
  // 使用 usePoolsExtended 获取带 USD 价值的池子列表
  const { data: pools, isLoading } = usePoolsExtended()

  if (isLoading) return <div>加载中...</div>

  return (
    <div>
      {pools?.map((pool) => (
        <div key={pool.address}>
          <span>{pool.token0.symbol} / {pool.token1.symbol}</span>
          <span>TVL: ${pool.reserveUSD.toFixed(2)}</span>
        </div>
      ))}
    </div>
  )
}
```

### 常见用法

#### 1. 按 TVL 排序的池子列表

```tsx
import { usePoolsExtended } from '@sushiswap/aptos'

function TopPoolsByTVL() {
  // 使用 usePoolsExtended 获取带 USD 价值的池子
  const { data: pools, isLoading } = usePoolsExtended()

  // 按 TVL 降序排序
  const sortedPools = pools?.sort((a, b) => b.reserveUSD - a.reserveUSD)

  if (isLoading) return <div>加载中...</div>

  return (
    <div>
      <h3>池子排行榜（按 TVL）</h3>
      {sortedPools?.slice(0, 10).map((pool, i) => (
        <div key={pool.address}>
          <span>#{i + 1}</span>
          <span>{pool.token0.symbol} / {pool.token1.symbol}</span>
          <span>TVL: ${pool.reserveUSD.toFixed(2)}</span>
        </div>
      ))}
    </div>
  )
}
```

#### 2. 显示池子的详细 USD 信息

```tsx
import { usePoolsExtended } from '@sushiswap/aptos'

function PoolUSDDetails({ poolAddress }: { poolAddress: string }) {
  // 使用 usePoolsExtended 获取池子列表
  const { data: pools } = usePoolsExtended()
  // 找到指定池子
  const pool = pools?.find((p) => p.address === poolAddress)

  if (!pool) return <div>池子不存在</div>

  return (
    <div>
      <h3>{pool.token0.symbol} / {pool.token1.symbol}</h3>
      <div className="stats">
        <div>总锁仓 (USD)</div>
        <div>${pool.reserveUSD.toLocaleString()}</div>

        <div>{pool.token0.symbol} 储备</div>
        <div>{Number(pool.reserve0).toFixed(4)}</div>
        <div>{pool.token0.symbol} USD 价值</div>
        <div>${pool.reserve0USD.toFixed(2)}</div>

        <div>{pool.token1.symbol} 储备</div>
        <div>{Number(pool.reserve1).toFixed(4)}</div>
        <div>{pool.token1.symbol} USD 价值</div>
        <div>${pool.reserve1USD.toFixed(2)}</div>
      </div>
    </div>
  )
}
```

#### 3. 组合筛选功能

```tsx
import { usePoolsExtended } from '@sushiswap/aptos'
import { useState } from 'react'

function FilteredPools() {
  // 使用 usePoolsExtended 获取带 USD 价值的池子
  const { data: pools } = usePoolsExtended()
  const [minTVL, setMinTVL] = useState<number>(0)

  // 按 TVL 筛选
  const filteredPools = pools?.filter(
    (pool) => pool.reserveUSD >= minTVL
  )

  // 按 TVL 排序
  const sortedPools = filteredPools?.sort((a, b) => b.reserveUSD - a.reserveUSD)

  return (
    <div>
      <div>
        <label>最小 TVL (USD): </label>
        <input
          type="number"
          value={minTVL}
          onChange={(e) => setMinTVL(Number(e.target.value))}
        />
      </div>
      <div>池子数量: {sortedPools?.length}</div>
      {sortedPools?.map((pool) => (
        <PoolRow key={pool.address} pool={pool} />
      ))}
    </div>
  )
}
```

### Do（推荐做法）

- **按 TVL 排序显示**：帮助用户发现优质池子
- **使用 useMemo 优化**：大量池子计算需要缓存
- **组合筛选功能**：按 TVL、交易量等筛选
- **显示 USD 价值更直观**：帮助用户理解池子规模

### Don't（不推荐做法）

- **不要缺少加载状态**：依赖多个数据源，加载时间较长
- **不要忽略排序**：大量池子需要排序才能展示
- **不要在 useMemo 中执行昂贵计算**：内部已优化

### 相关的其他 hooks

- `usePools` *(一个React hook，用于获取网络中所有流动性池，从合约资源中读取所有交易对信息)*：获取所有池子
- `useStablePrices` *(一个React hook，批量获取多个代币USD价格，是useStablePrice的批量版本)*：批量获取价格
- `useBaseTokens` *(一个React hook，用于获取Aptos网络所有基础代币列表，从tokenlist配置中读取)*：获取代币列表
- `PoolExtended` *(一个TypeScript类型/配置，扩展的池子类型，在Pool基础上增加了reserve0USD、reserve1USD、reserveUSD等USD价值字段)*：带 USD 价值的池子类型
