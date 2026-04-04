> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/pool/lib/use-pools.ts`

# usePools Hook Tutorial

## 大白话讲讲这个hook的作用

`usePools` 是一个用于获取网络中所有流动性池的 hook。它从合约资源中读取所有交易对信息。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **批量获取**：一次性获取所有池子
2. **数据转换**：将原始数据转换为 Pool 格式

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
enabled?: boolean          // 是否启用
```

### 输出（返回值）
```typescript
{
  data: Pool[]            // 池子数组
}
```

### 核心执行逻辑

1. **查询资源**：查询 swap 合约的所有资源
2. **过滤**：只保留 TokenPairMetadata 类型
3. **转换**：转换为 Pool 格式

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 usePools 的 React hook，用来获取链上所有的流动性池子。

核心需求：
1. 查询swap合约里的所有 TokenPairMetadata 类型的资源
2. 把这些原始数据转换成我们需要的池子格式
3. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**过滤正确的数据**：从合约里查出来的资源有很多种，你要只保留那些是我们需要的类型的数据，其他的过滤掉。

### 这个Hook怎么管理状态

建议使用 `placeholderData: keepPreviousData` 这个配置。它的作用是：当新数据正在加载的时候，先继续显示旧的数据，而不是显示加载中的状态。这样用户看到的内容就不会闪烁。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { usePools } from '@sushiswap/aptos'

function AllPools() {
  // 使用 usePools 获取所有池子
  const { data: pools, isLoading } = usePools()

  if (isLoading) return <div>加载中...</div>

  return (
    <div>
      <h3>全部池子 ({pools?.length})</h3>
      {pools?.map((pool) => (
        <PoolRow key={pool.address} pool={pool} />
      ))}
    </div>
  )
}
```

### 常见用法

#### 1. 池子列表页面

```tsx
import { usePools, useStablePrices } from '@sushiswap/aptos'

function PoolList() {
  // 获取所有池子
  const { data: pools } = usePools()
  // 批量获取价格
  const { data: prices } = useStablePrices({
    currencies: pools?.flatMap((p) => [p.token0, p.token1]) ?? []
  })

  // 计算每个池子的 TVL
  const poolsWithTVL = pools?.map((pool) => {
    const price0 = prices?.[pool.token0.address] ?? 0
    const price1 = prices?.[pool.token1.address] ?? 0
    const tvl = Number(pool.reserve0) * price0 + Number(pool.reserve1) * price1
    return { ...pool, tvl }
  }).sort((a, b) => b.tvl - a.tvl) // 按 TVL 排序

  return (
    <div>
      {poolsWithTVL?.map((pool) => (
        <PoolRow key={pool.address} pool={pool} tvl={pool.tvl} />
      ))}
    </div>
  )
}
```

#### 2. 按代币过滤池子

```tsx
import { usePools } from '@sushiswap/aptos'

function PoolsFilteredByToken({ tokenAddress }: { tokenAddress: string }) {
  // 获取所有池子
  const { data: pools } = usePools()

  // 过滤出包含指定代币的池子
  const filteredPools = pools?.filter(
    (pool) =>
      pool.token0.address === tokenAddress ||
      pool.token1.address === tokenAddress
  )

  return (
    <div>
      <h3>包含该代币的池子 ({filteredPools?.length})</h3>
      {filteredPools?.map((pool) => (
        <PoolRow key={pool.address} pool={pool} />
      ))}
    </div>
  )
}
```

#### 3. 组合自定义代币获取用户持仓池子

```tsx
import { usePools, useCustomTokens } from '@sushiswap/aptos'

function PoolsWithCustomTokens() {
  // 获取所有池子
  const { data: pools } = usePools()
  // 获取自定义代币
  const { data: customTokens } = useCustomTokens()
  const customTokenAddresses = Object.keys(customTokens ?? {})

  // 过滤出包含自定义代币的池子
  const poolsWithCustom = pools?.filter(
    (pool) =>
      customTokenAddresses.includes(pool.token0.address) ||
      customTokenAddresses.includes(pool.token1.address)
  )

  return (
    <div>
      <h3>包含自定义代币的池子</h3>
      {poolsWithCustom?.map((pool) => (
        <PoolRow key={pool.address} pool={pool} />
      ))}
    </div>
  )
}
```

### Do（推荐做法）

- **使用 keepPreviousData**：避免数据切换闪烁
- **组合 useStablePrices 计算 TVL**：批量获取价格后计算
- **按 TVL 排序显示**：帮助用户发现优质池子
- **组合代币过滤**：帮助用户快速找到目标池子

### Don't（不推荐做法）

- **不要在移动端一次加载所有池子**：数据量大可能导致性能问题
- **不要忽略 isLoading**：显示加载状态提升体验
- **不要假设池子都有流动性**：需要检查储备量

### 相关的其他 hooks

- `usePool` *(一个React hook，用于获取单个流动性池信息，包括代币对、储备量、手续费率等)*：获取单个池子
- `useStablePrices` *(一个React hook，批量获取多个代币USD价格，是useStablePrice的批量版本)*：批量获取价格
- `usePoolsExtended` *(一个React hook，获取带USD价值的池子列表，扩展了usePools，添加USD价值计算)*：带 USD 价值的池子
- `Pool` *(一个TypeScript类型/配置，表示流动性池的数据结构，包含token0、token1、reserve0、reserve1、fee等字段)*：池子数据类型
