> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/pool/lib/use-user-position-pools.ts`

# useUserPositionPools Hook Tutorial

## 大白话讲讲这个hook的作用

`useUserPositionPools` 是一个用于获取用户持仓池子列表的 hook。它：

- 查询用户持有的 LP 代币
- 返回对应的池子信息

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **用户持仓**：查询用户在哪些池子有持仓
2. **数据合并**：合并池子信息与 APR 数据

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
address: string          // 用户地址
enabled?: boolean        // 是否启用
```

### 输出（返回值）
```typescript
{
  data: PoolExtendedWithAprVolume[]   // 用户持仓的池子列表
}
```

### 核心执行逻辑

1. **查询用户资源**：获取用户的 CoinStore 资源
2. **过滤 LP 代币**：找到所有 LP 代币
3. **匹配池子**：与全部池子匹配

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useUserPositionPools 的 React hook，用来获取用户持仓的池子列表。

核心需求：
1. 查询用户账户里的 CoinStore 资源
2. 过滤出用户持有的 LP 代币
3. 把这些 LP 和对应的池子信息匹配起来
4. 还要加上年化收益率和交易量这些数据
5. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**过滤正确类型**：用户账户里可能有很多种 CoinStore 资源，要只保留 LP 代币类型的，其他普通代币的 CoinStore 要过滤掉。

### 这个Hook怎么管理状态

建议设置不在组件挂载时自动刷新。因为用户持仓数据不需要每次进页面都刷新，除非用户主动操作了。这个可以减少不必要的请求。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useUserPositionPools } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function MyPositions() {
  const { account } = useAccount()

  // 使用 useUserPositionPools 获取用户持仓池子
  const { data: positions, isLoading } = useUserPositionPools({
    address: account?.address
  })

  if (isLoading) return <div>加载中...</div>

  return (
    <div>
      <h3>我的持仓 ({positions?.length})</h3>
      {positions?.map((pool) => (
        <PositionRow key={pool.address} pool={pool} />
      ))}
    </div>
  )
}
```

### 常见用法

#### 1. 显示用户所有持仓详情

```tsx
import { useUserPositionPools, useStablePrices } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function MyPositionsDetailed() {
  const { account } = useAccount()

  // 获取用户持仓池子
  const { data: positions } = useUserPositionPools({
    address: account?.address
  })
  // 获取价格
  const { data: prices } = useStablePrices({
    currencies: positions?.flatMap((p) => [p.token0, p.token1]) ?? []
  })

  return (
    <div>
      {positions?.map((pool) => {
        const lpValue = /* 计算 LP 价值 */
        const apr = pool.totalApr1d
        return (
          <div key={pool.address} className="position-card">
            <div className="pool-name">
              {pool.token0.symbol} / {pool.token1.symbol}
            </div>
            <div className="pool-stats">
              <div>LP 数量: {pool.lpBalance}</div>
              <div>价值: ${lpValue.toFixed(2)}</div>
              <div>APR: {apr.toFixed(2)}%</div>
            </div>
          </div>
        )
      })}
    </div>
  )
}
```

#### 2. 总资产统计

```tsx
import { useUserPositionPools } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function TotalAssetValue() {
  const { account } = useAccount()

  // 获取用户持仓池子
  const { data: positions } = useUserPositionPools({
    address: account?.address
  })

  // 计算总持仓价值
  const totalValue = positions?.reduce((sum, pool) => {
    return sum + (pool.reserveUSD ?? 0) * (pool.lpBalance / pool.totalSupply)
  }, 0) ?? 0

  return (
    <div>
      <h3>总资产</h3>
      <div className="total-value">${totalValue.toFixed(2)}</div>
      <div>持仓数量: {positions?.length}</div>
    </div>
  )
}
```

#### 3. 条件显示（有无持仓）

```tsx
import { useUserPositionPools } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function PositionSection() {
  const { account } = useAccount()

  // 获取用户持仓
  const { data: positions } = useUserPositionPools({
    address: account?.address
  })

  const hasPositions = positions && positions.length > 0

  return (
    <div>
      {hasPositions ? (
        <div className="positions-list">
          {positions?.map((pool) => (
            <PositionRow key={pool.address} pool={pool} />
          ))}
        </div>
      ) : (
        <div className="no-positions">
          <p>您还没有参与任何池子</p>
          <button onClick={() => navigate('/pools')}>浏览池子</button>
        </div>
      )}
    </div>
  )
}
```

### Do（推荐做法）

- **组合 useStablePrices 计算价值**：计算持仓 USD 价值
- **处理无持仓情况**：没有持仓时显示引导 UI
- **显示 APR 信息**：帮助用户了解收益
- **使用 refetchOnMount: false**：避免不必要刷新

### Don't（不推荐做法）

- **不要在无钱包时启用**：需要检查 account 存在
- **不要忽略 isLoading**：持仓查询需要时间
- **不要假设所有池子都有 APR**：新池子可能没有 APR

### 相关的其他 hooks

- `usePool` *(一个React hook，用于获取单个流动性池信息，包括代币对、储备量、手续费率等)*：获取池子信息
- `usePoolsExtended` *(一个React hook，获取带USD价值的池子列表，扩展了usePools，添加USD价值计算)*：带 USD 价值的池子
- `useStablePrices` *(一个React hook，批量获取多个代币USD价格，是useStablePrice的批量版本)*：批量获取价格
- `PoolExtendedWithAprVolume` *(一个TypeScript类型/配置，扩展的池子类型，包含APR和交易量等数据)*：持仓池子类型
