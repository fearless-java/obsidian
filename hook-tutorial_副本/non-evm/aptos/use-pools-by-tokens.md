> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/pool/lib/use-pools-by-tokens.ts`

# usePoolsByTokens Hook Tutorial

## 大白话讲讲这个hook的作用

`usePoolsByTokens` 是一个用于通过代币对查询池子的 hook。它：

- 接受一组代币对
- 返回每个代币对的池子状态（存在/不存在/无效）

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **多池子查询**：支持批量查询多个代币对
2. **状态返回**：返回每对的状态

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  tokens: [Token | undefined, Token | undefined][]   // 代币对数组
  ledgerVersion?: number
}
```

### 输出（返回值）
```typescript
{
  data: [PairState, Pool | null][]   // 每对的池子状态
}
```

### PairState
```typescript
enum PairState {
  LOADING = 'Loading'
  NOT_EXISTS = 'Not Exists'
  EXISTS = 'Exists'
  INVALID = 'Invalid'
}
```

### 核心执行逻辑

1. **获取池子地址**：对每个代币对计算池子资源地址
2. **批量查询**：使用 `usePoolsReserves` 批量获取储备
3. **判断状态**：根据储备是否存在判断池子状态

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 usePoolsByTokens 的 React hook，用来通过代币对查询对应的池子。

核心需求：
1. 接收一组代币对作为参数
2. 计算出每个代币对对应的池子地址
3. 批量查询这些池子的储备金数据
4. 返回每个代币对的池子是否存在等信息
5. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**代币地址要排序**：计算池子地址的时候，要把两个代币的地址排序一下，确保顺序一致。不然的话，同样的两个代币因为顺序不同会计算出不同的池子地址。

### 这个Hook怎么管理状态

这个hook依赖另一个获取池子储备的hook来实现具体功能。在你的代码里，要先确保那个获取池子储备的hook能正常工作。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { usePoolsByTokens } from '@sushiswap/aptos'
import { Token } from '@sushiswap/aptos'

function PoolLookup({ token0, token1 }: { token0: Token; token1: Token }) {
  // 使用 usePoolsByTokens 查询代币对的池子
  const { data: pools } = usePoolsByTokens({
    tokens: [[token0, token1]]
  })

  const [state, pool] = pools?.[0] ?? []

  return (
    <div>
      <p>状态: {state}</p>
      {pool && (
        <div>
          <p>池子地址: {pool.address}</p>
          <p>储备0: {pool.reserve0}</p>
          <p>储备1: {pool.reserve1}</p>
        </div>
      )}
    </div>
  )
}
```

### 常见用法

#### 1. Swap 界面查询交易对

```tsx
import { usePoolsByTokens, useCommonTokens } from '@sushiswap/aptos'

function SwapPairSelector() {
  // 获取常用代币
  const { data: tokens } = useCommonTokens()
  const tokenList = Object.values(tokens ?? {})

  // 选择两个代币
  const [tokenIn, setTokenIn] = useState<Token | undefined>()
  const [tokenOut, setTokenOut] = useState<Token | undefined>()

  // 查询该交易对是否有池子
  const { data: pools } = usePoolsByTokens({
    tokens: [[tokenIn, tokenOut]]
  })

  const [state] = pools?.[0] ?? []
  const poolExists = state === 'Exists'
  const isLoading = state === 'Loading'
  const notExists = state === 'Not Exists'

  return (
    <div>
      <select value={tokenIn?.address} onChange={(e) => setTokenIn(...)}>
        {tokenList.map((t) => <option key={t.address}>{t.symbol}</option>)}
      </select>
      <span>→</span>
      <select value={tokenOut?.address} onChange={(e) => setTokenOut(...)}>
        {tokenList.map((t) => <option key={t.address}>{t.symbol}</option>)}
      </select>
      {isLoading && <div>检查池子...</div>}
      {poolExists && <div className="success">池子存在，可以交易</div>}
      {notExists && <div className="warning">暂无数币池子</div>}
    </div>
  )
}
```

#### 2. 批量检查多个交易对

```tsx
import { usePoolsByTokens } from '@sushiswap/aptos'
import { useState } from 'react'

function MultiPairChecker() {
  const [pairs, setPairs] = useState<[Token | undefined, Token | undefined][]>([])
  // 批量查询多个代币对
  const { data: results } = usePoolsByTokens({ tokens: pairs })

  return (
    <div>
      <h3>批量池子检查结果</h3>
      {results?.map(([state, pool], i) => (
        <div key={i}>
          Pair {i}: {state}
          {pool && <span> 储备: {pool.reserve0}</span>}
        </div>
      ))}
    </div>
  )
}
```

#### 3. 根据池子存在性显示不同 UI

```tsx
import { usePoolsByTokens } from '@sushiswap/aptos'

function TradeButton({ tokenIn, tokenOut, amount }: { tokenIn: Token; tokenOut: Token; amount: string }) {
  // 查询池子是否存在
  const { data: pools } = usePoolsByTokens({
    tokens: [[tokenIn, tokenOut]]
  })

  const [state, pool] = pools?.[0] ?? []

  if (!tokenIn || !tokenOut) {
    return <button disabled>选择代币</button>
  }

  if (state === 'Loading') {
    return <button disabled>检查池子...</button>
  }

  if (state === 'Not Exists') {
    return <button disabled>暂无数币池子</button>
  }

  if (state === 'Invalid') {
    return <button disabled>无效交易对</button>
  }

  return (
    <button>
      Swap {amount} {tokenIn.symbol} → {tokenOut.symbol}
    </button>
  )
}
```

### Do（推荐做法）

- **使用 PairState 判断状态**：根据状态显示不同 UI
- **处理 LOADING 状态**：批量查询需要等待
- **代币顺序不影响结果**：内部会排序计算地址
- **组合 usePools 显示完整信息**：获取池子后显示详细信息

### Don't（不推荐做法）

- **不要直接假设池子存在**：需要检查 PairState
- **不要忽略 LOADING 状态**：批量查询有等待时间
- **不要传入 undefined 代币**：会导致 INVALID 状态

### 相关的其他 hooks

- `usePoolsReserves` *(一个React hook，批量查询池子储备量，接受池子资源地址数组，返回每个池子的储备量)*：批量查询储备
- `usePool` *(一个React hook，用于获取单个流动性池信息，包括代币对、储备量、手续费率等)*：获取单个池子
- `PairState` *(一个TypeScript类型/配置，枚举类型，表示交易对的状态，包括LOADING、NOT_EXISTS、EXISTS、INVALID)*：交易对状态枚举
- `Pool` *(一个TypeScript类型/配置，表示流动性池的数据结构，包含token0、token1、reserve0、reserve1、fee等字段)*：池子数据类型
