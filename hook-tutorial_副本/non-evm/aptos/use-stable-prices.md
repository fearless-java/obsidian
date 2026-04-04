> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/_common/lib/common/use-stable-prices.ts`

# useStablePrices Hook Tutorial

## 大白话讲讲这个hook的作用

`useStablePrices` 是一个用于批量获取多个代币 USD 价格的 hook。它是 `useStablePrice` 的批量版本。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **批量优化**：一次性查询多个代币价格
2. **共享计算**：复用价格计算逻辑

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  currencies: Token[] | undefined  // 代币数组
  ledgerVersion?: number          // 账本版本
}
```

### 输出（返回值）
```typescript
{
  data: Record<string, number>   // { 代币地址: 价格 }
  isLoading: boolean
}
```

### 核心执行逻辑

1. **构建交易对**：为每个代币与稳定币构建交易对
2. **批量查询**：使用 `usePoolsByTokens` 批量获取
3. **计算价格**：计算每个代币的价格

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useStablePrices 的 React hook，用来批量获取多个代币的美元价格。

核心需求：
1. 要能接收一个代币数组作为参数
2. 一次性查询所有代币和稳定币之间的交易池信息
3. 计算出每个代币值多少美元
4. 返回的格式要用代币地址作为key，价格数字作为value
5. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**批量查询**：不是一个个去查，而是要把所有的代币对放在一起，一次性查询完。这样可以大大减少网络请求的次数，提高性能。

### 这个Hook怎么管理状态

这个hook依赖另一个获取池子数据的hook来实现功能。在你的代码里，要先确保那个获取池子的hook能正常工作。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useStablePrices, useBaseTokens } from '@sushiswap/aptos'

function TokenPrices() {
  // 使用 useBaseTokens 获取代币列表
  const { data: tokens } = useBaseTokens()
  const tokenArray = Object.values(tokens ?? {})

  // 使用 useStablePrices 批量获取价格
  const { data: prices, isLoading } = useStablePrices({
    currencies: tokenArray
  })

  if (isLoading) return <div>加载中...</div>

  return (
    <div>
      {tokenArray.map((token) => (
        <div key={token.address}>
          {token.symbol}: ${prices?.[token.address]?.toFixed(6) ?? '—'}
        </div>
      ))}
    </div>
  )
}
```

### 常见用法

#### 1. 批量显示代币价格

```tsx
import { useStablePrices, useCommonTokens } from '@sushiswap/aptos'

function PriceTable() {
  // 使用 useCommonTokens 获取常用代币
  const { data: tokens } = useCommonTokens()
  const tokenArray = Object.values(tokens ?? {})

  // 使用 useStablePrices 批量获取价格
  const { data: prices } = useStablePrices({
    currencies: tokenArray
  })

  return (
    <table>
      <thead>
        <tr>
          <th>代币</th>
          <th>价格</th>
        </tr>
      </thead>
      <tbody>
        {tokenArray.map((token) => (
          <tr key={token.address}>
            <td>{token.symbol}</td>
            <td>${prices?.[token.address]?.toFixed(6) ?? '—'}</td>
          </tr>
        ))}
      </tbody>
    </table>
  )
}
```

#### 2. 批量计算池子 USD 价值

```tsx
import { useStablePrices, usePools } from '@sushiswap/aptos'

function PoolsWithUSD() {
  // 获取所有池子
  const { data: pools } = usePools()
  // 使用 useStablePrices 批量获取所有相关代币的价格
  const { data: prices } = useStablePrices({
    currencies: pools?.flatMap((p) => [p.token0, p.token1]) ?? []
  })

  return (
    <div>
      {pools?.map((pool) => {
        const price0 = prices?.[pool.token0.address] ?? 0
        const price1 = prices?.[pool.token1.address] ?? 0
        const reserve0USD = Number(pool.reserve0) * price0
        const reserve1USD = Number(pool.reserve1) * price1
        const totalUSD = reserve0USD + reserve1USD

        return (
          <div key={pool.address}>
            {pool.token0.symbol}/{pool.token1.symbol}
            <span>TVL: ${totalUSD.toFixed(2)}</span>
          </div>
        )
      })}
    </div>
  )
}
```

#### 3. 带过滤的批量价格获取

```tsx
import { useStablePrices, useBaseTokens, useCustomTokens } from '@sushiswap/aptos'

function SelectedTokenPrices({ addresses }: { addresses: string[] }) {
  // 获取代币映射
  const { data: baseTokens } = useBaseTokens()
  const { data: customTokens } = useCustomTokens()
  const allTokens = { ...baseTokens, ...customTokens }

  // 筛选出指定地址的代币
  const selectedTokens = addresses
    .map((addr) => allTokens[addr])
    .filter(Boolean) as Token[]

  // 批量获取价格
  const { data: prices } = useStablePrices({
    currencies: selectedTokens
  })

  return (
    <div>
      {selectedTokens.map((token) => (
        <div key={token.address}>
          {token.symbol}: ${prices?.[token.address]?.toFixed(6) ?? '—'}
        </div>
      ))}
    </div>
  )
}
```

### Do（推荐做法）

- **批量使用优于单个调用**：多个代币价格查询使用 useStablePrices
- **组合 usePools 获取代币列表**：从池子列表提取所有相关代币
- **使用 isLoading 显示加载状态**：批量查询可能需要等待
- **处理 undefined 价格**：某些代币可能没有价格数据

### Don't（不推荐做法）

- **不要在循环中调用单个 useStablePrice**：使用批量版本
- **不要忽略 isLoading**：批量查询有明显的等待时间
- **不要假设所有代币都有价格**：某些新上线的代币可能没有交易对

### 相关的其他 hooks

- `useStablePrice` *(一个React hook，用于获取单个代币的USD价格，通过查找代币与稳定币的交易对并计算储备量比例得出价格)*：单个价格查询
- `usePoolsByTokens` *(一个React hook，通过代币对查询池子状态，返回每个代币对的池子是否存在)*：查询代币对池子
- `usePools` *(一个React hook，用于获取网络中所有流动性池，从合约资源中读取所有交易对信息)*：获取所有池子
- `useBaseTokens` *(一个React hook，用于获取Aptos网络所有基础代币列表，从tokenlist配置中读取)*：获取代币列表
