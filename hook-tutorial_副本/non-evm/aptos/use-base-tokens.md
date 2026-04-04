> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/_common/lib/common/use-base-tokens.ts`

# useBaseTokens Hook Tutorial

## 大白话讲讲这个hook的作用

`useBaseTokens` 是一个用于获取 Aptos 网络基础代币列表的 hook。它从 tokenlist 配置中读取：

- 所有支持的代币列表
- 每个代币的地址、名称、符号、小数位等

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **配置管理**：代币列表来自配置文件
2. **React Query 集成**：缓存代币列表

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
- 无参数

### 输出（返回值）
```typescript
{
  data: Record<string, Token>   // 代币映射表
}
```

### Token 结构
```typescript
{
  address: string
  decimals: number
  name: string
  symbol: string
  logoURI?: string
}
```

### 核心执行逻辑

1. **从配置读取**：从 `tokenlists[network].tokens` 读取
2. **转换为映射**：使用地址作为 key

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useBaseTokens 的 React hook，用来获取链上的基础代币列表。

核心需求：
1. 要从配置文件里读取代币列表数据
2. 返回的格式要用代币地址作为key，对象作为value的结构
3. 用 React Query 来管理数据缓存和更新

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

1. **网络差异**：不同的网络（比如主网和测试网）使用的代币列表是不一样的。你的代码需要能够区分当前连接的是哪个网络，然后加载对应的代币列表。

2. **特殊代币符号**：有些代币的符号字段可能比较特殊，不是普通的英文字母，比如跨链资产的符号会带有特殊前缀或者后缀。你的代码要能正确处理这些情况。

### 这个Hook怎么管理状态

因为代币列表是相对稳定的数据，不会频繁变化，所以建议使用 `placeholderData: keepPreviousData` 这个配置。它的作用是：当新数据正在加载的时候，先继续显示旧的数据，而不是显示加载中的状态。这样用户看到的内容就不会闪烁或者出现短暂的空白。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useBaseTokens } from '@sushiswap/aptos'

function TokenList() {
  // 使用 useBaseTokens 获取基础代币列表
  const { data: tokens } = useBaseTokens()

  return (
    <div>
      {Object.values(tokens ?? {}).map((token) => (
        <div key={token.address}>
          {token.symbol} - {token.name}
        </div>
      ))}
    </div>
  )
}
```

### 常见用法

#### 1. 获取特定代币信息

```tsx
import { useBaseTokens } from '@sushiswap/aptos'

function TokenInfo({ address }: { address: string }) {
  // 使用 useBaseTokens 获取代币映射
  const { data: tokens } = useBaseTokens()

  // 通过地址查找代币
  const token = tokens?.[address]

  if (!token) return null

  return (
    <div>
      <img src={token.logoURI} alt={token.symbol} />
      <span>{token.symbol}</span>
      <span>{token.name}</span>
      <span>精度: {token.decimals}</span>
    </div>
  )
}
```

#### 2. 与其他 hooks 组合使用

```tsx
import { useBaseTokens, useTokenBalance } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function MyTokenBalances() {
  const { account } = useAccount()
  // 使用 useBaseTokens 获取代币列表
  const { data: tokens } = useBaseTokens()

  return (
    <div>
      {Object.values(tokens ?? {}).map((token) => (
        <TokenBalanceRow
          key={token.address}
          token={token}
          account={account?.address}
        />
      ))}
    </div>
  )
}
```

#### 3. 创建代币选择器

```tsx
import { useBaseTokens } from '@sushiswap/aptos'

function TokenSelector({ onSelect }: { onSelect: (token: Token) => void }) {
  // 使用 useBaseTokens 获取代币列表
  const { data: tokens } = useBaseTokens()
  const [query, setQuery] = useState('')

  // 过滤代币
  const filteredTokens = Object.values(tokens ?? {}).filter(
    (token) =>
      token.symbol.toLowerCase().includes(query.toLowerCase()) ||
      token.name.toLowerCase().includes(query.toLowerCase())
  )

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="搜索代币..."
      />
      <div>
        {filteredTokens.map((token) => (
          <div key={token.address} onClick={() => onSelect(token)}>
            {token.symbol}
          </div>
        ))}
      </div>
    </div>
  )
}
```

### Do（推荐做法）

- **使用 `keepPreviousData` 避免闪烁**：配置 `placeholderData: keepPreviousData`
- **使用地址作为 key**：代币映射使用地址作为键值
- **组合 useTokenWithCache 使用**：获取完整代币信息
- **注意 decimals 处理**：金额计算时需要考虑代币精度

### Don't（不推荐做法）

- **不要硬编码代币地址**：使用配置文件或 API 获取
- **不要忽略网络区分**：不同网络代币列表不同
- **不要直接使用 symbol 作为 key**：可能存在重复 symbol 的代币

### 相关的其他 hooks

- `useCommonTokens` *(一个React hook，用于获取常用/热门代币列表，只返回APT、lzWBTC、lzUSDC等基础代币)*：获取常用代币
- `useCustomTokens` *(一个React hook，用于管理用户自定义添加的代币列表，支持从localStorage持久化)*：管理自定义代币
- `useTokenWithCache` *(一个React hook，带缓存的代币信息查询hook，先检查本地缓存，缓存未命中时从链上API获取)*：带缓存的代币查询
- `Token` *(一个TypeScript类型/配置，定义代币的基本信息结构，包含address、decimals、name、symbol、logoURI等字段)*：代币类型定义
