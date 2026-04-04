> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/_common/lib/common/use-token-with-cache.ts`

# useTokenWithCache Hook Tutorial

## 大白话讲讲这个hook的作用

`useTokenWithCache` 是一个带有缓存的代币信息查询 hook。它：

- 先检查自定义代币列表
- 如果未找到，从链上 API 获取代币信息

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **缓存优先**：优先使用本地缓存
2. **链上查询**：缓存未命中时查询链上

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  address: string                    // 代币地址
  enabled?: boolean                  // 是否启用
  keepPreviousData?: boolean         // 保持旧数据
}
```

### 输出（返回值）
```typescript
{
  data: Token | undefined           // 代币信息
}
```

### 核心执行逻辑

1. **检查缓存**：先查 customTokens
2. **链上查询**：缓存未命中时调用 API

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useTokenWithCache 的 React hook，用来查询代币信息，但要有缓存机制。

核心需求：
1. 接收一个代币地址作为参数
2. 先看看本地缓存里有没有这个代币的信息，有就直接用
3. 缓存里没有的话，再去链上的接口获取
4. 要支持切换代币的时候保留旧数据的显示
5. 用 React Query 来管理数据请求和缓存

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**用地址做缓存的key**：缓存的时候，要用代币的地址来区分不同的代币，而不是用代币的名字。因为地址是唯一的，而名字可能会重复。

### 这个Hook怎么管理状态

1. **不要重试**：如果查不到某个代币的信息，不要一直重试，直接放弃。因为这个代币可能真的不存在，重试也没用。

2. **保持旧数据**：当用户切换选择的代币时，可以先用旧数据显示着，等新数据加载完再更新，这样界面不会闪烁。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useTokenWithCache } from '@sushiswap/aptos'

function TokenInfo({ address }: { address: string }) {
  // 使用 useTokenWithCache 获取代币信息（带缓存）
  const { data: token } = useTokenWithCache({ address })

  if (!token) return <div>加载中...</div>

  return (
    <div>
      <h3>{token.symbol}</h3>
      <p>{token.name}</p>
      <p>精度: {token.decimals}</p>
    </div>
  )
}
```

### 常见用法

#### 1. 带保持旧数据的代币切换

```tsx
import { useTokenWithCache } from '@sushiswap/aptos'
import { useState } from 'react'

function TokenDetail() {
  const [tokenAddress, setTokenAddress] = useState('')

  // 使用 useTokenWithCache 获取代币信息，切换时保持旧数据
  const { data: token } = useTokenWithCache({
    address: tokenAddress,
    keepPreviousData: true // 切换时保持旧数据显示
  })

  return (
    <div>
      <input
        value={tokenAddress}
        onChange={(e) => setTokenAddress(e.target.value)}
        placeholder="输入代币地址"
      />
      {token && (
        <div>
          <p>符号: {token.symbol}</p>
          <p>名称: {token.name}</p>
          <p>精度: {token.decimals}</p>
        </div>
      )}
    </div>
  )
}
```

#### 2. 条件启用的代币查询

```tsx
import { useTokenWithCache } from '@sushiswap/aptos'
import { useState } from 'react'

function ConditionalTokenQuery() {
  const [address, setAddress] = useState('')
  // 验证地址格式（简单的 0x 开头检查）
  const isValidAddress = address.startsWith('0x') && address.length > 10

  // 只有地址有效时才启用查询
  const { data: token } = useTokenWithCache({
    address,
    enabled: isValidAddress
  })

  return (
    <div>
      <input
        value={address}
        onChange={(e) => setAddress(e.target.value)}
        placeholder="输入代币地址"
      />
      {isValidAddress && (
        token ? <TokenInfo token={token} /> : <div>加载中...</div>
      )}
    </div>
  )
}
```

#### 3. 组合自定义代币列表

```tsx
import { useTokenWithCache, useCustomTokens } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function TokenBalanceWithInfo({ address }: { address: string }) {
  // 先检查是否是自定义代币
  const { data: customTokens } = useCustomTokens()

  // 如果是自定义代币，使用自定义代币的信息
  // 否则使用链上查询
  const { data: token } = useTokenWithCache({ address })

  // 从自定义代币列表获取信息
  const customToken = customTokens?.[address]

  // 优先使用自定义代币信息，否则使用链上查询结果
  const finalToken = customToken ?? token

  return (
    <div>
      {finalToken ? (
        <>
          <h3>{finalToken.symbol}</h3>
          <p>{finalToken.name}</p>
        </>
      ) : (
        <div>加载中...</div>
      )}
    </div>
  )
}
```

### Do（推荐做法）

- **使用 keepPreviousData**：切换代币时保持旧数据，避免闪烁
- **设置 enabled 条件**：地址无效时不发起请求
- **组合 useCustomTokens**：自定义代币优先使用本地缓存
- **设置 retry: false**：代币不存在时不重复请求

### Don't（不推荐做法）

- **不要在循环中调用**：代币列表应该用批量查询
- **不要忽略 enabled**：无效地址不应该发起请求
- **不要直接信任返回数据**：需要处理 undefined 情况

### 相关的其他 hooks

- `useCustomTokens` *(一个React hook，用于管理用户自定义添加的代币列表，支持从localStorage持久化)*：自定义代币管理
- `useBaseTokens` *(一个React hook，用于获取Aptos网络所有基础代币列表，从tokenlist配置中读取)*：基础代币列表
- `useTokenBalance` *(一个React hook，用于查询单个代币余额，接受account、currency、enabled等参数)*：代币余额查询
- `keepPreviousData` *(一个React Query配置选项，用于在数据更新时保持旧数据不闪烁，直到新数据加载完成)*：React Query 配置选项
