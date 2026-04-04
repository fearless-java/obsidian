> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/_common/lib/common/use-sorted-token-list.ts`

# useSortedTokenList Hook Tutorial

## 大白话讲讲这个hook的作用

`useSortedTokenList` 是一个用于搜索和排序代币列表的 hook。它：

- 根据 query 过滤代币
- 按相关性排序
- 如果有余额数据，优先显示有余额的代币

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **搜索逻辑**：处理 symbol、name、address 匹配
2. **排序逻辑**：综合相关性和余额排序
3. **防抖**：避免频繁搜索

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  query: string                    // 搜索词
  tokenMap?: Record<string, Token> // 代币列表
  customTokenMap?: Record<string, Token>  // 自定义代币列表
  balanceMap?: Record<string, number>    // 余额映射
}
```

### 输出（返回值）
```typescript
{
  data: TokenWithBalance[] | Token[]  // 排序后的代币列表
}
```

### 核心执行逻辑

1. **合并代币**：合并基础代币和自定义代币
2. **过滤**：匹配 query
3. **排序**：按余额和相关性排序

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useSortedTokenList 的 React hook，用来搜索和排序代币列表。

核心需求：
1. 要能接收搜索词、代币地图、余额地图这些参数
2. 要能在代币的符号、名称、地址里搜索匹配的内容
3. 搜索结果要排序：有钱余额的代币要排在前面，剩下的按匹配度排序
4. 搜索词要防抖处理，就是等用户停下来的意思
5. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

1. **防抖处理**：用户打字的时候不要立即搜索，要等用户暂停大约250毫秒后再搜索。这样可以避免用户还在输入的时候就触发搜索，减少没必要的计算和请求。

2. **余额优先排序**：用户钱包里有钱的代币要显示在最前面，因为用户肯定更关心自己有资产的代币。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useSortedTokenList, useBaseTokens } from '@sushiswap/aptos'
import { useState } from 'react'

function TokenSearch() {
  // 使用 useBaseTokens 获取代币列表
  const { data: tokenMap } = useBaseTokens()
  // 本地搜索状态
  const [query, setQuery] = useState('')

  // 使用 useSortedTokenList 搜索和排序代币
  const { data: sortedTokens } = useSortedTokenList({
    query,
    tokenMap
  })

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="搜索代币..."
      />
      {sortedTokens?.map((token) => (
        <TokenRow key={token.address} token={token} />
      ))}
    </div>
  )
}
```

### 常见用法

#### 1. 带余额显示的代币搜索

```tsx
import { useSortedTokenList, useBaseTokens, useTokenBalances } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function TokenSearchWithBalance() {
  const { account } = useAccount()
  // 使用 useBaseTokens 获取代币列表
  const { data: tokenMap } = useBaseTokens()
  // 使用 useTokenBalances 获取余额
  const { data: balanceMap } = useTokenBalances({
    account: account?.address,
    currencies: Object.keys(tokenMap ?? {})
  })

  const [query, setQuery] = useState('')

  // 使用 useSortedTokenList 搜索和排序代币，有余额的优先显示
  const { data: sortedTokens } = useSortedTokenList({
    query,
    tokenMap,
    balanceMap
  })

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="搜索代币..."
      />
      {sortedTokens?.map((token) => (
        <TokenRow
          key={token.address}
          token={token}
          balance={balanceMap?.[token.address]}
        />
      ))}
    </div>
  )
}
```

#### 2. 合并基础代币和自定义代币的搜索

```tsx
import { useSortedTokenList, useBaseTokens, useCustomTokens } from '@sushiswap/aptos'

function AllTokensSearch() {
  // 获取基础代币
  const { data: baseTokens } = useBaseTokens()
  // 获取自定义代币
  const { data: customTokens } = useCustomTokens()
  const [query, setQuery] = useState('')

  // 合并代币列表
  const allTokens = {
    ...baseTokens,
    ...customTokens
  }

  // 使用 useSortedTokenList 搜索全部代币
  const { data: sortedTokens } = useSortedTokenList({
    query,
    tokenMap: allTokens
  })

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="搜索所有代币..."
      />
      <div className="token-list">
        {sortedTokens?.map((token) => (
          <TokenRow key={token.address} token={token} />
        ))}
      </div>
    </div>
  )
}
```

#### 3. 带防抖的搜索组件

```tsx
import { useSortedTokenList, useBaseTokens } from '@sushiswap/aptos'
import { useDebouncedValue } from '@sushiswap/hooks'

function DebouncedTokenSearch() {
  const { data: tokenMap } = useBaseTokens()
  const [query, setQuery] = useState('')

  // 使用防抖，避免频繁搜索
  const debouncedQuery = useDebouncedValue(query, 250)

  // 使用 useSortedTokenList 搜索代币
  const { data: sortedTokens } = useSortedTokenList({
    query: debouncedQuery,
    tokenMap
  })

  return (
    <div>
      {/* 实时更新输入框，不触发搜索 */}
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="输入代币名称、符号或地址..."
      />
      {/* 显示防抖后的搜索结果 */}
      <div className="results">
        {sortedTokens?.map((token) => (
          <TokenRow key={token.address} token={token} />
        ))}
      </div>
    </div>
  )
}
```

### Do（推荐做法）

- **使用防抖处理 query**：避免频繁搜索，使用 `useDebounce` 或 `useDebouncedValue`
- **组合 useBaseTokens 和 useCustomTokens**：提供完整的代币列表
- **显示余额优先**：有余额的代币优先显示，提升用户体验
- **支持多种匹配**：symbol、name、address 都能匹配

### Don't（不推荐做法）

- **不要直接在输入时搜索**：使用防抖减少计算
- **不要忽略余额排序**：有余额的代币应该优先显示
- **不要缺少空状态处理**：没有搜索结果时应该提示用户

### 相关的其他 hooks

- `useBaseTokens` *(一个React hook，用于获取Aptos网络所有基础代币列表，从tokenlist配置中读取)*：获取基础代币
- `useCustomTokens` *(一个React hook，用于管理用户自定义添加的代币列表，支持从localStorage持久化)*：管理自定义代币
- `useTokenBalances` *(一个React hook，用于查询代币余额，支持单个和批量查询，返回代币地址到余额的映射)*：查询代币余额
- `useDebounce` *(一个React hook/工具函数，用于防抖处理，将快速变化的输入延迟处理以提高性能)*：防抖处理 hook
