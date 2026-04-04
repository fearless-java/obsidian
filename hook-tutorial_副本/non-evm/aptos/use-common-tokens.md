> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/_common/lib/common/use-common-tokens.ts`

# useCommonTokens Hook Tutorial

## 大白话讲讲这个hook的作用

`useCommonTokens` 是一个用于获取常用/热门代币的 hook。它只返回一部分基础代币：

- APT（原生代币）
- lzWBTC、lzUSDC、lzUSDT、lzWETH（跨链封装代币）

这是一个精简版的代币列表，用于快速选择常用代币。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **过滤常用代币**：从完整代币列表中过滤出常用代币
2. **简化选择**：帮助用户快速找到常用资产

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
- 无参数

### 输出（返回值）
```typescript
{
  data: Record<string, Token>   // 常用代币映射表
}
```

### 核心执行逻辑

1. **获取基础代币**：从 tokenlist 读取
2. **过滤**：只保留 allowedSymbols 列表中的代币

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useCommonTokens 的 React hook，用来获取常用的代币列表。

核心需求：
1. 从完整的代币列表中筛选出最常用的那几个，比如 APT、WBTC、USDC、USDT、ETH 等
2. 返回的格式要用代币地址作为key，对象作为value的结构
3. 用 React Query 来管理数据缓存和更新

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**硬编码的过滤逻辑**：代码里会用一个固定集合来存储允许的代币符号（比如 APT、lzWBTC、lzUSDC 等）。只有在这个集合里的代币才会被返回。这是快速获取常用代币的简单方式。

### 这个Hook怎么管理状态

建议使用 `placeholderData: keepPreviousData` 这个配置。它的作用是：当新数据正在加载的时候，先继续显示旧的数据，而不是显示加载中的加载动画或者文字。这样用户在等待的时候看到的内容是稳定的，不会闪烁。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useCommonTokens } from '@sushiswap/aptos'

function CommonTokenList() {
  // 使用 useCommonTokens 获取常用代币
  const { data: tokens } = useCommonTokens()

  return (
    <div>
      {Object.values(tokens ?? {}).map((token) => (
        <div key={token.address}>
          <img src={token.logoURI} alt={token.symbol} />
          <span>{token.symbol}</span>
        </div>
      ))}
    </div>
  )
}
```

### 常见用法

#### 1. 快速选择常用代币（Swap 界面常用）

```tsx
import { useCommonTokens } from '@sushiswap/aptos'

function QuickTokenSelect({ selected, onSelect }: Props) {
  // 使用 useCommonTokens 获取常用代币
  const { data: tokens } = useCommonTokens()

  const commonTokenList = Object.values(tokens ?? {})

  return (
    <div className="flex gap-2">
      {commonTokenList.map((token) => (
        <button
          key={token.address}
          onClick={() => onSelect(token)}
          className={selected?.address === token.address ? 'selected' : ''}
        >
          {token.symbol}
        </button>
      ))}
    </div>
  )
}
```

#### 2. 获取常用代币作为默认选项

```tsx
import { useCommonTokens } from '@sushiswap/aptos'
import { useState } from 'react'

function SwapInterface() {
  // 使用 useCommonTokens 获取常用代币
  const { data: tokens } = useCommonTokens()

  // 设置默认的输入和输出代币
  const [tokenIn, setTokenIn] = useState<Token | undefined>()
  const [tokenOut, setTokenOut] = useState<Token | undefined>()

  // 初始化默认代币（通常第一个是 APT）
  useEffect(() => {
    const commonList = Object.values(tokens ?? {})
    if (commonList.length > 0 && !tokenIn) {
      setTokenIn(commonList[0]) // 默认选 APT
    }
  }, [tokens, tokenIn])

  // ... rest of component
}
```

#### 3. 与自定义代币合并

```tsx
import { useCommonTokens, useCustomTokens } from '@sushiswap/aptos'

function AllTokens() {
  // 使用 useCommonTokens 获取常用代币
  const { data: commonTokens } = useCommonTokens()
  // 使用 useCustomTokens 获取自定义代币
  const { data: customTokens } = useCustomTokens()

  // 合并常用代币和自定义代币
  const allTokens = {
    ...commonTokens,
    ...customTokens
  }

  return (
    <div>
      {Object.values(allTokens).map((token) => (
        <TokenRow key={token.address} token={token} />
      ))}
    </div>
  )
}
```

### Do（推荐做法）

- **用于快速选择 UI**：在 Swap、发送等界面显示常用代币
- **与 useCustomTokens 组合**：合并常用代币和用户自定义代币
- **使用 keepPreviousData**：避免数据闪烁
- **显示代币图标**：使用 logoURI 显示代币图标

### Don't（不推荐做法）

- **不要用于完整代币列表**：这只是常用代币，不是全部
- **不要硬编码 allowedSymbols**：应该从配置读取
- **不要忽略 decimals**：金额计算时需要注意精度

### 相关的其他 hooks

- `useBaseTokens` *(一个React hook，用于获取Aptos网络所有基础代币列表，从tokenlist配置中读取)*：获取所有基础代币
- `useCustomTokens` *(一个React hook，用于管理用户自定义添加的代币列表，支持从localStorage持久化)*：管理自定义代币
- `useSortedTokenList` *(一个React hook，用于搜索和排序代币列表，支持根据query过滤代币并按相关性排序)*：代币搜索排序
- `Token` *(一个TypeScript类型/配置，定义代币的基本信息结构，包含address、decimals、name、symbol、logoURI等字段)*：代币类型定义
