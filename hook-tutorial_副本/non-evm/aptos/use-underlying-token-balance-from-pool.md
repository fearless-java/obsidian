> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/pool/lib/use-underlying-token-balance-from-pool.ts`

# useUnderlyingTokenBalanceFromPool Hook Tutorial

## 大白话讲讲这个hook的作用

`useUnderlyingTokenBalanceFromPool` 是一个用于计算用户在池子中的底层代币余额的 hook。它：

- 根据用户的 LP 代币余额
- 池子的总供应量
- 池子的储备量
- 计算用户实际拥有的底层代币数量

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **计算逻辑**：封装 AMM 公式计算
2. **多数据源**：需要组合多个数据

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  totalSupply: number | undefined | null    // LP 总供应量
  reserve0: number | undefined | null       // 储备 0
  reserve1: number | undefined | null       // 储备 1
  token0: { decimals: number } | undefined  // 代币 0 小数位
  token1: { decimals: number } | undefined  // 代币 1 小数位
  balance: number | undefined | null       // 用户 LP 余额
}
```

### 输出（返回值）
```typescript
[string | undefined, string | undefined]   // [底层代币0数量, 底层代币1数量]
```

### 核心执行逻辑

1. **计算份额**：balance / totalSupply
2. **计算底层代币**：份额 * reserve

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useUnderlyingTokenBalanceFromPool 的 React hook，用来计算用户在池子里的实际代币数量。

核心需求：
1. 接收这些参数：LP总供应量、两个代币的储备金、代币的小数位、用户持有的LP数量
2. 用数学公式计算出用户实际拥有的底层代币数量
3. 要处理代币精度转换
4. 返回两个代币的数量
5. 用缓存来保存计算结果

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

1. **防止除以零**：如果池子里 LP 的总供应量是零，那就没法计算比例了，直接返回零就行了。

2. **处理精度**：区块链上的代币金额数字很大很复杂，因为每个代币的小数位不一样。计算的时候要按代币的精度正确转换，不然显示出来的数字会出错。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useUnderlyingTokenBalanceFromPool } from '@sushiswap/aptos'

function UnderlyingBalance() {
  // 假设已有以下数据
  const totalSupply = 1000000
  const reserve0 = 500000
  const reserve1 = 250000
  const balance = 10000

  // 使用 useUnderlyingTokenBalanceFromPool 计算底层代币
  const [amount0, amount1] = useUnderlyingTokenBalanceFromPool({
    totalSupply,
    reserve0,
    reserve1,
    token0: { decimals: 8 },
    token1: { decimals: 6 },
    balance
  })

  return (
    <div>
      <p>底层代币 0: {amount0}</p>
      <p>底层代币 1: {amount1}</p>
    </div>
  )
}
```

### 常见用法

#### 1. 组合池子数据显示用户持仓

```tsx
import { useUnderlyingTokenBalanceFromPool, usePool, useTotalSupply, useTokenBalance } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function MyPoolPosition({ poolAddress }: { poolAddress: string }) {
  const { account } = useAccount()

  // 获取池子信息
  const { data: pool } = usePool(poolAddress)
  // 获取 LP 总供应量
  const { data: coinInfo } = useTotalSupply(pool?.lpTokenAddress)
  // 获取用户 LP 余额
  const { data: lpBalance } = useTokenBalance({
    account: account?.address,
    currency: pool?.lpTokenAddress
  })

  // 使用 useUnderlyingTokenBalanceFromPool 计算用户底层代币
  const [amount0, amount1] = useUnderlyingTokenBalanceFromPool({
    totalSupply: Number(coinInfo?.supply ?? 0),
    reserve0: Number(pool?.reserve0 ?? 0),
    reserve1: Number(pool?.reserve1 ?? 0),
    token0: pool?.token0,
    token1: pool?.token1,
    balance: Number(lpBalance ?? 0)
  })

  return (
    <div>
      <h3>我的持仓</h3>
      <p>{pool?.token0.symbol}: {amount0}</p>
      <p>{pool?.token1.symbol}: {amount1}</p>
    </div>
  )
}
```

#### 2. 计算用户持仓价值

```tsx
import { useUnderlyingTokenBalanceFromPool, usePool, useStablePrices } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function PositionValue({ poolAddress }: { poolAddress: string }) {
  const { account } = useAccount()

  const { data: pool } = usePool(poolAddress)
  const { data: coinInfo } = useTotalSupply(pool?.lpTokenAddress)
  const { data: lpBalance } = useTokenBalance({
    account: account?.address,
    currency: pool?.lpTokenAddress
  })
  const { data: prices } = useStablePrices({
    currencies: [pool?.token0, pool?.token1].filter(Boolean)
  })

  const [amount0, amount1] = useUnderlyingTokenBalanceFromPool({
    totalSupply: Number(coinInfo?.supply ?? 0),
    reserve0: Number(pool?.reserve0 ?? 0),
    reserve1: Number(pool?.reserve1 ?? 0),
    token0: pool?.token0,
    token1: pool?.token1,
    balance: Number(lpBalance ?? 0)
  })

  const value0 = Number(amount0) * (prices?.[pool?.token0?.address ?? ''] ?? 0)
  const value1 = Number(amount1) * (prices?.[pool?.token1?.address ?? ''] ?? 0)
  const totalValue = value0 + value1

  return (
    <div>
      <p>持仓总价值: ${totalValue.toFixed(2)}</p>
    </div>
  )
}
```

#### 3. 计算用户份额比例

```tsx
import { useUnderlyingTokenBalanceFromPool } from '@sushiswap/aptos'

function ShareRatio({ totalSupply, reserve0, reserve1, token0, token1, balance }: Props) {
  // 使用 useUnderlyingTokenBalanceFromPool 计算底层代币
  const [amount0, amount1] = useUnderlyingTokenBalanceFromPool({
    totalSupply,
    reserve0,
    reserve1,
    token0,
    token1,
    balance
  })

  // 计算份额比例
  const shareRatio = totalSupply > 0 ? (balance / totalSupply) * 100 : 0

  return (
    <div>
      <p>持有 LP: {balance}</p>
      <p>占总供应比例: {shareRatio.toFixed(4)}%</p>
      <p>相当于: {amount0} {token0?.symbol}</p>
      <p>相当于: {amount1} {token1?.symbol}</p>
    </div>
  )
}
```

### Do（推荐做法）

- **使用 useMemo 缓存**：避免重复计算
- **处理除零情况**：totalSupply 为 0 时返回 '0'
- **考虑 decimals 格式化**：显示时按代币精度格式化
- **组合 usePool、useTokenBalance 使用**：获取所有需要的数据

### Don't（不推荐做法）

- **不要忽略 decimals**：区块链数字需要精度转换
- **不要除以零**：需要先检查 totalSupply
- **不要假设 balance 存在**：需要处理 undefined 情况

### 相关的其他 hooks

- `usePool` *(一个React hook，用于获取单个流动性池信息，包括代币对、储备量、手续费率等)*：获取池子信息
- `useTotalSupply` *(一个React hook，用于查询LP代币总供应量，返回CoinInfo结构包含supply字段)*：查询总供应量
- `useTokenBalance` *(一个React hook，用于查询单个代币余额，接受account、currency、enabled等参数)*：查询 LP 余额
- `AMM` *(一个技术概念/配置，自动化做市商，一种去中心化交易协议，通过数学公式定价并提供流动性)*：AMM 公式
