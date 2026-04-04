> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/pool/lib/use-tokens-from-pool.ts`

# useTokensFromPool Hook Tutorial

## 大白话讲讲这个hook的作用

`useTokensFromPool` 是一个用于从池子信息中提取代币信息的 hook。它：

- 接收池子对象
- 返回池子的 token0 和 token1

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **数据合并**：合并 baseTokens 和池子代币信息
2. **优先级**：优先使用 baseTokens 中的信息

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
row: Pool | undefined     // 池子对象
```

### 输出（返回值）
```typescript
{
  token0: Token | undefined
  token1: Token | undefined
}
```

### 核心执行逻辑

1. **检查池子**：确保池子对象存在
2. **合并信息**：baseTokens 有则用 baseTokens，否则用池子数据

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useTokensFromPool 的 React hook，用来从池子信息里提取出代币数据。

核心需求：
1. 接收一个池子对象作为参数
2. 把从池子里拿到的代币信息和基础代币列表合并
3. 返回这个池子的第一个代币和第二个代币
4. 用缓存来保存结果，避免重复计算

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**基础代币优先**：如果基础代币列表里有这个代币的信息，就用基础代币的。因为基础代币列表里的信息更完整、更可靠。池子里的代币信息可能比较简单。

### 这个Hook怎么管理状态

建议用缓存来保存计算结果。因为这个hook可能会被频繁调用，每次都重新合并计算会比较耗时。用了缓存，同样的输入就能直接返回之前计算好的结果。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useTokensFromPool } from '@sushiswap/aptos'
import { usePool } from '@sushiswap/aptos'

function PoolTokens({ poolAddress }: { poolAddress: string }) {
  // 获取池子信息
  const { data: pool } = usePool(poolAddress)
  // 使用 useTokensFromPool 提取代币信息
  const { token0, token1 } = useTokensFromPool(pool)

  return (
    <div>
      <div>
        <img src={token0?.logoURI} alt={token0?.symbol} />
        <span>{token0?.symbol}</span>
      </div>
      <div>
        <img src={token1?.logoURI} alt={token1?.symbol} />
        <span>{token1?.symbol}</span>
      </div>
    </div>
  )
}
```

### 常见用法

#### 1. 池子代币信息展示

```tsx
import { useTokensFromPool, usePool } from '@sushiswap/aptos'

function PoolTokenInfo({ poolAddress }: { poolAddress: string }) {
  // 获取池子
  const { data: pool } = usePool(poolAddress)
  // 提取代币信息
  const { token0, token1 } = useTokensFromPool(pool)

  return (
    <div className="pool-header">
      <div className="token">
        {token0?.logoURI && <img src={token0.logoURI} />}
        <span>{token0?.symbol}</span>
        <span>{token0?.name}</span>
      </div>
      <span className="separator">/</span>
      <div className="token">
        {token1?.logoURI && <img src={token1.logoURI} />}
        <span>{token1?.symbol}</span>
        <span>{token1?.name}</span>
      </div>
    </div>
  )
}
```

#### 2. 组合价格查询

```tsx
import { useTokensFromPool, usePool, useStablePrices } from '@sushiswap/aptos'

function PoolWithPrices({ poolAddress }: { poolAddress: string }) {
  // 获取池子
  const { data: pool } = usePool(poolAddress)
  // 提取代币
  const { token0, token1 } = useTokensFromPool(pool)
  // 查询价格
  const { data: prices } = useStablePrices({
    currencies: [token0, token1].filter(Boolean)
  })

  const price0 = prices?.[token0?.address ?? '']
  const price1 = prices?.[token1?.address ?? '']

  return (
    <div>
      <div>
        {token0?.symbol}: ${price0?.toFixed(6) ?? '—'}
      </div>
      <div>
        {token1?.symbol}: ${price1?.toFixed(6) ?? '—'}
      </div>
    </div>
  )
}
```

#### 3. 批量池子代币渲染

```tsx
import { useTokensFromPool, usePools } from '@sushiswap/aptos'

function PoolList() {
  // 获取所有池子
  const { data: pools } = usePools()

  return (
    <div>
      {pools?.map((pool) => {
        // 从每个池子提取代币
        const { token0, token1 } = useTokensFromPool(pool)
        return (
          <div key={pool.address}>
            {token0?.symbol} / {token1?.symbol}
          </div>
        )
      })}
    </div>
  )
}
```

### Do（推荐做法）

- **使用 useMemo 缓存**：避免重复计算
- **优先使用 baseTokens 信息**：确保代币信息完整
- **处理 undefined 池子**：池子不存在时返回 undefined
- **组合其他 hooks 使用**：配合 useStablePrices 获取价格

### Don't（不推荐做法）

- **不要在循环外调用时使用池子变量**：内部需要池子对象
- **不要忽略 undefined**：需要处理池子不存在的情况
- **不要假设 token0/token1 存在**：可能返回 undefined

### 相关的其他 hooks

- `usePool` *(一个React hook，用于获取单个流动性池信息，包括代币对、储备量、手续费率等)*：获取池子信息
- `usePools` *(一个React hook，用于获取网络中所有流动性池，从合约资源中读取所有交易对信息)*：获取所有池子
- `useBaseTokens` *(一个React hook，用于获取Aptos网络所有基础代币列表，从tokenlist配置中读取)*：获取基础代币
- `useStablePrices` *(一个React hook，批量获取多个代币USD价格，是useStablePrice的批量版本)*：获取价格
