> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/_common/lib/common/use-stable-price.ts`

# useStablePrice Hook Tutorial

## 大白话讲讲这个hook的作用

`useStablePrice` 是一个用于获取代币 USD 价格的 hook。它通过：

- 查找该代币与稳定币的交易对
- 计算出 USD 价格

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **池子查询**：查询代币-稳定币交易对
2. **价格计算**：从储备量计算价格

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  currency: Token | undefined     // 代币
  ledgerVersion?: number         // 账本版本
}
```

### 输出（返回值）
```typescript
{
  data: number | undefined       // USD 价格
}
```

### 核心执行逻辑

1. **查询池子**：查找代币与稳定币的交易对
2. **计算价格**：从池子储备量计算价格

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useStablePrice 的 React hook，用来获取某个代币的美元价格。

核心需求：
1. 要能接收一个代币作为参数
2. 要找到这个代币和稳定币（比如 USDC、USDT）之间的交易池
3. 根据池子里两种代币的数量比例来计算出这个代币值多少美元
4. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**选择哪个稳定币**：要用哪个稳定币来计算价格，这个是固定的，由系统默认指定。通常用储量最大、最稳定的那个。

### 这个Hook怎么管理状态

这个hook依赖另一个hook来获取池子数据。所以在你的代码里，要先确保那个获取池子的hook能正常工作，才能正确获取价格数据。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useStablePrice } from '@sushiswap/aptos'

function TokenPrice({ token }: { token: Token }) {
  // 使用 useStablePrice 获取代币 USD 价格
  const { data: price } = useStablePrice({ currency: token })

  return (
    <div>
      <span>{token.symbol}</span>
      <span>${price?.toFixed(6) ?? '—'}</span>
    </div>
  )
}
```

### 常见用法

#### 1. 显示代币价格列表

```tsx
import { useStablePrice, useCommonTokens } from '@sushiswap/aptos'

function PriceList() {
  // 使用 useCommonTokens 获取常用代币
  const { data: tokens } = useCommonTokens()

  return (
    <div>
      <h3>常用代币价格</h3>
      {Object.values(tokens ?? {}).map((token) => (
        <TokenPriceRow key={token.address} token={token} />
      ))}
    </div>
  )
}

function TokenPriceRow({ token }: { token: Token }) {
  // 使用 useStablePrice 获取单个代币价格
  const { data: price } = useStablePrice({ currency: token })

  return (
    <div>
      <span>{token.symbol}</span>
      <span>${price ? price.toFixed(6) : '—'}</span>
    </div>
  )
}
```

#### 2. 计算池子 USD 总价值

```tsx
import { useStablePrice, usePool } from '@sushiswap/aptos'

function PoolUSDValue({ poolAddress }: { poolAddress: string }) {
  // 使用 usePool 获取池子信息
  const { data: pool } = usePool({ poolAddress })

  // 使用 useStablePrice 获取两个代币的价格
  const { data: price0 } = useStablePrice({ currency: pool?.token0 })
  const { data: price1 } = useStablePrice({ currency: pool?.token1 })

  // 计算 USD 总价值
  const reserve0USD = Number(pool?.reserve0) * (price0 ?? 0)
  const reserve1USD = Number(pool?.reserve1) * (price1 ?? 0)
  const totalUSD = reserve0USD + reserve1USD

  return (
    <div>
      <p>池子 USD 总价值: ${totalUSD.toFixed(2)}</p>
    </div>
  )
}
```

#### 3. 带历史版本的价格查询

```tsx
import { useStablePrice, useLedgerVersion } from '@sushiswap/aptos'

function HistoricalTokenPrice({ token }: { token: Token }) {
  // 获取 24 小时前的账本版本
  const { data: version } = useLedgerVersion(86400)

  // 使用历史账本版本查询彼时的价格
  const { data: historicalPrice } = useStablePrice({
    currency: token,
    ledgerVersion: version
  })

  // 当前价格
  const { data: currentPrice } = useStablePrice({ currency: token })

  const priceChange = currentPrice && historicalPrice
    ? ((currentPrice - historicalPrice) / historicalPrice) * 100
    : 0

  return (
    <div>
      <p>当前价格: ${currentPrice?.toFixed(6) ?? '—'}</p>
      <p>24小时前: ${historicalPrice?.toFixed(6) ?? '—'}</p>
      <p>变化: {priceChange.toFixed(2)}%</p>
    </div>
  )
}
```

### Do（推荐做法）

- **组合 usePool 使用**：获取池子信息后计算 USD 价值
- **处理 undefined**：价格可能是 undefined，需要显示占位符
- **使用 toFixed 格式化**：价格显示时使用合理的小数位数
- **考虑历史价格查询**：配合 useLedgerVersion 查询历史价格

### Don't（不推荐做法）

- **不要直接显示 undefined**：使用占位符如 '—' 或 'Loading'
- **不要忽略精度问题**：区块链数字可能需要格式化处理
- **不要假设价格存在**：某些代币可能没有稳定币交易对

### 相关的其他 hooks

- `usePoolsByTokens` *(一个React hook，通过代币对查询池子状态，返回每个代币对的池子是否存在)*：查询代币对池子
- `useStablePrices` *(一个React hook，批量获取多个代币USD价格，是useStablePrice的批量版本)*：批量获取价格
- `usePool` *(一个React hook，用于获取单个流动性池信息，包括代币对、储备量、手续费率等)*：获取池子信息
- `useLedgerVersion` *(一个React hook，用于获取Aptos账本版本，支持查询历史版本的账本号)*：获取账本版本
