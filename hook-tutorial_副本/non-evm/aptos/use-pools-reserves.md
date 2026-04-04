> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/pool/lib/use-pools-reserves.ts`

# usePoolsReserves Hook Tutorial

## 大白话讲讲这个hook的作用

`usePoolsReserves` 是一个用于批量查询池子储备量的 hook。它：

- 接受池子资源地址数组
- 返回每个池子的储备量

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **批量查询**：一次查询多个池子储备
2. **并行处理**：使用 Promise.allSettled

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  poolAddresses: string[] | undefined   // 池子资源地址数组
  ledgerVersion?: number
}
```

### 输出（返回值）
```typescript
{
  data: Record<string, PoolReserve>   // { 资源地址: 储备量 }
}
```

### PoolReserve 结构
```typescript
{
  reserve0: string   // Token0 储备
  reserve1: string   // Token1 储备
  timestamp: string  // 时间戳
}
```

### 核心执行逻辑

1. **批量请求**：对每个地址调用 API
2. **Promise.allSettled**：处理部分失败
3. **映射返回**：转为 Record 格式

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 usePoolsReserves 的 React hook，用来批量查询多个池子的储备金。

核心需求：
1. 接收一个池子地址数组作为参数
2. 一次性查询所有池子的储备金数据
3. 如果其中某个池子查失败了，不要影响其他的，继续查能查的
4. 返回的格式要用池子地址作为key，储备数据作为value
5. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**容错处理**：查询多个池子的时候，可能有些能查到，有些查不到。这个时候不能因为有几个失败就直接全部失败，而是要让能查到的继续正常工作。查不到的返回一个空或者错误标记就行。

### 这个Hook怎么管理状态

这是批量查询，有失败也不会阻塞整体。使用 Promise.allSettled 而不是 Promise.all 可以确保即使部分查询失败，整体请求也不会被拒绝。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { usePoolsReserves } from '@sushiswap/aptos'

function PoolReservesList({ poolAddresses }: { poolAddresses: string[] }) {
  // 使用 usePoolsReserves 批量查询储备量
  const { data: reserves } = usePoolsReserves({
    poolAddresses
  })

  return (
    <div>
      {poolAddresses.map((address) => (
        <div key={address}>
          <p>池子: {address}</p>
          <p>储备0: {reserves?.[address]?.reserve0}</p>
          <p>储备1: {reserves?.[address]?.reserve1}</p>
        </div>
      ))}
    </div>
  )
}
```

### 常见用法

#### 1. 批量获取池子储备计算 TVL

```tsx
import { usePoolsReserves, usePools, useStablePrices } from '@sushiswap/aptos'

function PoolsWithTVL() {
  // 获取所有池子
  const { data: pools } = usePools()
  // 批量获取储备
  const { data: reserves } = usePoolsReserves({
    poolAddresses: pools?.map((p) => p.address)
  })
  // 获取价格
  const { data: prices } = useStablePrices({
    currencies: pools?.flatMap((p) => [p.token0, p.token1]) ?? []
  })

  // 计算每个池子的 TVL
  const poolsWithTVL = pools?.map((pool) => {
    const reserve = reserves?.[pool.address]
    const price0 = prices?.[pool.token0.address] ?? 0
    const price1 = prices?.[pool.token1.address] ?? 0
    const tvl =
      Number(reserve?.reserve0 ?? 0) * price0 +
      Number(reserve?.reserve1 ?? 0) * price1
    return { ...pool, tvl }
  })

  return (
    <div>
      {poolsWithTVL?.map((pool) => (
        <div key={pool.address}>
          {pool.token0.symbol} / {pool.token1.symbol}
          TVL: ${pool.tvl.toFixed(2)}
        </div>
      ))}
    </div>
  )
}
```

#### 2. 实时监控储备变化

```tsx
import { usePoolsReserves } from '@sushiswap/aptos'
import { useEffect, useState } from 'react'

function LiveReserveMonitor({ poolAddress }: { poolAddress: string }) {
  // 实时获取储备（5秒刷新）
  const { data: reserves } = usePoolsReserves({
    poolAddresses: [poolAddress],
    refetchInterval: 5000
  })

  const reserve = reserves?.[poolAddress]

  return (
    <div>
      <h3>实时储备监控</h3>
      <p>Token0 储备: {reserve?.reserve0}</p>
      <p>Token1 储备: {reserve?.reserve1}</p>
      <p>更新时间: {reserve?.timestamp}</p>
    </div>
  )
}
```

#### 3. 处理部分池子查询失败

```tsx
import { usePoolsReserves } from '@sushiswap/aptos'

function RobustPoolList({ poolAddresses }: { poolAddresses: string[] }) {
  // 批量查询储备，部分失败不影响其他
  const { data: reserves, isError } = usePoolsReserves({
    poolAddresses
  })

  return (
    <div>
      {poolAddresses.map((address) => {
        const reserve = reserves?.[address]
        const hasData = reserve?.reserve0 !== undefined

        return (
          <div key={address}>
            <span>池子: {address}</span>
            {hasData ? (
              <>
                <span>储备0: {reserve.reserve0}</span>
                <span>储备1: {reserve.reserve1}</span>
              </>
            ) : (
              <span className="error">数据不可用</span>
            )}
          </div>
        )
      })}
      {isError && <div className="warning">部分池子查询失败</div>}
    </div>
  )
}
```

### Do（推荐做法）

- **使用 Promise.allSettled 处理部分失败**：一个池子失败不影响其他
- **设置合理的刷新间隔**：不需要实时刷新的场景可以用较长间隔
- **处理 undefined 情况**：某些池子可能查询失败
- **组合 usePools 获取地址列表**：从池子列表提取地址

### Don't（不推荐做法）

- **不要假设所有查询都成功**：部分失败是正常的
- **不要忽略错误状态**：isError 可以提醒用户
- **不要设置过短的刷新间隔**：大量池子会消耗 API 资源

### 相关的其他 hooks

- `usePool` *(一个React hook，用于获取单个流动性池信息，包括代币对、储备量、手续费率等)*：获取单个池子
- `usePools` *(一个React hook，用于获取网络中所有流动性池，从合约资源中读取所有交易对信息)*：获取所有池子
- `useStablePrices` *(一个React hook，批量获取多个代币USD价格，是useStablePrice的批量版本)*：批量获取价格
- `PoolReserve` *(一个TypeScript类型/配置，表示池子的储备信息，包含reserve0、reserve1、timestamp等字段)*：池子储备类型
