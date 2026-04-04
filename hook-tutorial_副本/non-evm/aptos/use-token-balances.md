> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/_common/lib/common/use-token-balances.ts`

# useTokenBalances Hook Tutorial

## 大白话讲讲这个hook的作用

`useTokenBalances` 是一个用于查询代币余额的 hook 集合：

- **useTokenBalance**：查询单个代币余额
- **useTokenBalances**：批量查询多个代币余额

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **API 调用封装**：封装 Aptos API 的余额查询
2. **批量优化**：支持批量查询减少 API 调用

## 讲讲该hook的执行逻辑和数据流向

### useTokenBalance 参数
```typescript
{
  account?: string      // 钱包地址
  currency?: string     // 代币地址
  enabled?: boolean
  refetchInterval?: number
}
```

### useTokenBalances 参数
```typescript
{
  account?: string      // 钱包地址
  currencies: string[]  // 代币地址数组
  enabled?: boolean
  refetchInterval?: number
}
```

### 输出

**useTokenBalance:**
```typescript
{
  data: number | undefined  // 余额
}
```

**useTokenBalances:**
```typescript
{
  data: Record<string, number>  // { 代币地址: 余额 }
}
```

### 核心执行逻辑

1. **构建查询**：使用 Aptos API 的 getCurrentFungibleAssetBalances
2. **过滤**：按 owner_address 和 asset_type 过滤
3. **映射返回**：返回余额映射

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useTokenBalances 的 React hook，用来查询代币余额。

核心需求：
1. 既要能查单个代币的余额，也要能一次查很多个代币的余额
2. 要调用 Aptos 的接口来获取余额数据
3. 返回的格式要用代币地址作为key，余额数字作为value
4. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

1. **调用正确的接口**：Aptos 有专门的索引接口来查余额，代码里要用对这个接口。

2. **处理接口出错的情况**：如果查余额的时候接口返回错误，不要让整个程序崩溃，而是返回一个0作为余额。这样至少不会影响用户看到其他正常的数据。

## 5. 该 hook 的用法教学

### 基本用法

#### 单个代币余额

```tsx
import { useTokenBalance } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function SingleBalance() {
  const { account } = useAccount()

  // 使用 useTokenBalance 查询单个代币余额
  const { data: balance } = useTokenBalance({
    account: account?.address,
    currency: '0x1::aptos_coin::AptosCoin'
  })

  return <div>APT 余额: {balance}</div>
}
```

#### 批量查询余额

```tsx
import { useTokenBalances } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'
import { useBaseTokens } from '@sushiswap/aptos'

function BatchBalances() {
  const { account } = useAccount()
  // 获取代币列表
  const { data: tokens } = useBaseTokens()
  const tokenAddresses = Object.keys(tokens ?? {})

  // 使用 useTokenBalances 批量查询余额
  const { data: balances } = useTokenBalances({
    account: account?.address,
    currencies: tokenAddresses
  })

  return (
    <div>
      {tokenAddresses.map((addr) => (
        <div key={addr}>
          {tokens?.[addr]?.symbol}: {balances?.[addr]}
        </div>
      ))}
    </div>
  )
}
```

### 常见用法

#### 1. 显示用户资产列表

```tsx
import { useTokenBalances, useBaseTokens, useStablePrices } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function AssetList() {
  const { account } = useAccount()
  // 获取代币列表
  const { data: tokens } = useBaseTokens()
  // 批量获取余额
  const { data: balances } = useTokenBalances({
    account: account?.address,
    currencies: Object.keys(tokens ?? {})
  })
  // 批量获取价格
  const { data: prices } = useStablePrices({
    currencies: Object.values(tokens ?? {})
  })

  // 计算总资产 USD 价值
  const totalUSD = Object.entries(balances ?? {}).reduce((sum, [addr, balance]) => {
    const price = prices?.[addr] ?? 0
    return sum + balance * price
  }, 0)

  return (
    <div>
      <div>总资产: ${totalUSD.toFixed(2)}</div>
      {Object.entries(balances ?? {}).map(([addr, balance]) => (
        <AssetRow
          key={addr}
          token={tokens?.[addr]}
          balance={balance}
          price={prices?.[addr]}
        />
      ))}
    </div>
  )
}
```

#### 2. 带自动刷新的余额监控

```tsx
import { useTokenBalances } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function LiveBalance({ tokenAddress }: { tokenAddress: string }) {
  const { account } = useAccount()

  // 使用 useTokenBalance 查询余额，2秒自动刷新
  const { data: balance } = useTokenBalance({
    account: account?.address,
    currency: tokenAddress,
    refetchInterval: 2000 // 2秒刷新
  })

  return (
    <div>
      余额: {balance}
      <span className="live-indicator">●</span>
    </div>
  )
}
```

#### 3. 余额为 0 的代币过滤

```tsx
import { useTokenBalances, useBaseTokens } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function NonZeroBalances() {
  const { account } = useAccount()
  // 获取代币列表
  const { data: tokens } = useBaseTokens()
  // 批量获取余额
  const { data: balances } = useTokenBalances({
    account: account?.address,
    currencies: Object.keys(tokens ?? {})
  })

  // 过滤出余额大于 0 的代币
  const nonZeroBalances = Object.entries(balances ?? {}).filter(
    ([, balance]) => Number(balance) > 0
  )

  return (
    <div>
      {nonZeroBalances.map(([addr, balance]) => (
        <div key={addr}>
          {tokens?.[addr]?.symbol}: {balance}
        </div>
      ))}
    </div>
  )
}
```

### Do（推荐做法）

- **批量查询使用 useTokenBalances**：多个代币余额使用批量版本
- **设置 refetchInterval 监控变化**：需要实时监控时设置刷新间隔
- **处理 undefined**：余额可能是 undefined
- **组合 useStablePrices 计算总资产**：批量获取价格后计算总价值

### Don't（不推荐做法）

- **不要循环调用单个查询**：使用批量版本
- **不要忽略 enabled 参数**：可以用于条件禁用查询
- **不要假设余额是数字**：可能需要 Number() 转换

### 相关的其他 hooks

- `useStablePrices` *(一个React hook，批量获取多个代币USD价格，是useStablePrice的批量版本)*：批量获取价格
- `useBaseTokens` *(一个React hook，用于获取Aptos网络所有基础代币列表，从tokenlist配置中读取)*：获取代币列表
- `useCustomTokens` *(一个React hook，用于管理用户自定义添加的代币列表，支持从localStorage持久化)*：获取自定义代币
- `getCurrentFungibleAssetBalances` *(一个Aptos API方法，用于查询当前账户的Fungible Asset余额，支持按owner_address和asset_type过滤)*：Aptos 余额查询 API
