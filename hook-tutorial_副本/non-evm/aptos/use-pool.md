> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/pool/lib/use-pool.ts`

# usePool Hook Tutorial

## 大白话讲讲这个hook的作用

`usePool` 是一个用于获取单个流动性池信息的 hook。它返回：

- 代币对信息
- 储备量
- 手续费率等

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **合约交互**：封装 Aptos 合约的池子查询

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
poolAddress: string          // 池子地址（TokenPairMetadata 类型）
```

### 输出（返回值）
```typescript
{
  data: Pool | null         // 池子信息
}
```

### 核心执行逻辑

1. **构建资源类型**：`TokenPairMetadata<PoolAddress>`
2. **调用 API**：获取池子资源

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 usePool 的 React hook，用来获取流动性池子的详细信息。

核心需求：
1. 接收池子地址作为参数
2. 去链上查询这个池子的 TokenPairMetadata 资源
3. 把查到的原始数据转换成我们需要的格式
4. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**资源类型要写对**：Aptos链上每个资源都有自己固定的类型标识。查询的时候要写对这个类型，不然查不到数据。TokenPairMetadata 资源的类型格式是固定的，你的代码要能正确构建出来。

### 这个Hook怎么管理状态

建议使用 `placeholderData: keepPreviousData` 这个配置。它的作用是：当新数据正在加载的时候，先继续显示旧的数据，而不是显示加载中的状态。这样用户看到的内容就不会闪烁或者出现短暂的空白。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { usePool } from '@sushiswap/aptos'

function PoolInfo({ poolAddress }: { poolAddress: string }) {
  // 使用 usePool 获取池子信息
  const { data: pool } = usePool(poolAddress)

  if (!pool) return <div>加载中...</div>

  return (
    <div>
      <h3>{pool.token0.symbol} / {pool.token1.symbol}</h3>
      <p>储备0: {pool.reserve0}</p>
      <p>储备1: {pool.reserve1}</p>
      <p>手续费率: {pool.fee}</p>
    </div>
  )
}
```

### 常见用法

#### 1. 显示池子详细信息

```tsx
import { usePool, useStablePrices } from '@sushiswap/aptos'

function PoolDetails({ poolAddress }: { poolAddress: string }) {
  // 获取池子信息
  const { data: pool } = usePool(poolAddress)
  // 获取代币价格
  const { data: prices } = useStablePrices({
    currencies: [pool?.token0, pool?.token1].filter(Boolean)
  })

  if (!pool) return <div>加载中...</div>

  const price0 = prices?.[pool.token0.address] ?? 0
  const price1 = prices?.[pool.token1.address] ?? 0
  const reserve0USD = Number(pool.reserve0) * price0
  const reserve1USD = Number(pool.reserve1) * price1
  const totalUSD = reserve0USD + reserve1USD

  return (
    <div>
      <div className="pool-header">
        <img src={pool.token0.logoURI} />
        <span>{pool.token0.symbol}</span>
        <span>/</span>
        <img src={pool.token1.logoURI} />
        <span>{pool.token1.symbol}</span>
      </div>
      <div className="pool-stats">
        <div>总锁仓</div>
        <div>${totalUSD.toFixed(2)}</div>
        <div>储备0</div>
        <div>{Number(pool.reserve0).toFixed(4)}</div>
        <div>储备1</div>
        <div>{Number(pool.reserve1).toFixed(4)}</div>
      </div>
    </div>
  )
}
```

#### 2. 组合 LP 持有量计算

```tsx
import { usePool, useTotalSupply, useUnderlyingTokenBalanceFromPool } from '@sushiswap/aptos'
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

  // 计算用户底层代币数量
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
      <p>我的 {pool?.token0.symbol} 数量: {amount0}</p>
      <p>我的 {pool?.token1.symbol} 数量: {amount1}</p>
    </div>
  )
}
```

#### 3. 池子状态检测

```tsx
import { usePool } from '@sushiswap/aptos'

function PoolStatus({ poolAddress }: { poolAddress: string }) {
  // 获取池子信息
  const { data: pool, isLoading } = usePool(poolAddress)

  if (isLoading) return <div>检查池子状态...</div>

  if (!pool) {
    return <div className="error">池子不存在</div>
  }

  const reserve0 = Number(pool.reserve0)
  const reserve1 = Number(pool.reserve1)
  const isEmpty = reserve0 === 0 && reserve1 === 0
  const hasLiquidity = reserve0 > 0 || reserve1 > 0

  return (
    <div>
      {isEmpty && <div className="warning">池子为空</div>}
      {hasLiquidity && <div>池子有流动性</div>}
    </div>
  )
}
```

### Do（推荐做法）

- **使用 keepPreviousData**：避免数据切换时的闪烁
- **组合 useStablePrices 计算 USD 价值**：计算池子 TVL
- **处理 pool 为 null**：池子可能不存在
- **组合 useTotalSupply 计算份额**：计算用户持仓比例

### Don't（不推荐做法）

- **不要直接访问 pool 属性没检查 null**：需要先检查 pool 是否存在
- **不要忽略 isLoading 状态**：数据加载需要时间
- **不要假设池子有流动性**：需要检查储备量是否为 0

### 相关的其他 hooks

- `usePools` *(一个React hook，用于获取网络中所有流动性池，从合约资源中读取所有交易对信息)*：获取所有池子
- `usePoolsByTokens` *(一个React hook，通过代币对查询池子状态，返回每个代币对的池子是否存在)*：通过代币对查询池子
- `useStablePrices` *(一个React hook，批量获取多个代币USD价格，是useStablePrice的批量版本)*：批量获取价格
- `Pool` *(一个TypeScript类型/配置，表示流动性池的数据结构，包含token0、token1、reserve0、reserve1、fee等字段)*：池子数据类型
