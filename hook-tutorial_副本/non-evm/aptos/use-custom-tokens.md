> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/_common/lib/common/use-custom-tokens.ts`

# useCustomTokens Hook Tutorial

## 大白话讲讲这个hook的作用

`useCustomTokens` 是一个用于管理用户自定义代币的 hook。它类似于 Stellar 版本：

- 添加自定义代币到本地存储
- 删除自定义代币
- 检查代币是否存在

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **localStorage 持久化**：保存用户的自定义代币
2. **简化操作**：提供 add/remove/mutate API

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
- 无参数

### 输出（返回值）
```typescript
{
  data: Record<string, Token>    // 自定义代币映射表
  mutate: (type: 'add' | 'remove', currency: Token[]) => void
  hasToken: (currency: Token | string) => boolean
}
```

### 核心执行逻辑

1. **读取 localStorage**：从 `sushi.customTokens.aptos` 读取
2. **添加**：将代币添加到对象
3. **删除**：从对象中移除
4. **检查**：检查代币是否在列表中

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useCustomTokens 的 React hook，用来管理用户自己添加的代币。

核心需求：
1. 用户添加的代币要保存到浏览器本地，这样刷新页面后还在
2. 要提供几个常用的操作：添加代币、删除代币、检查某个代币是否已添加
3. 用 localStorage 来实现数据的持久化存储
4. 返回的数据结构要包含代币数据和操作方法

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

1. **数据要持久化**：用户添加的自定义代币要保存到本地存储里，这样用户下次打开网页时还能看到自己添加过的代币。

2. **用地址找代币**：在存储和查找代币的时候，要用代币的地址作为标识，而不是用代币的名称或者符号。因为不同的代币可能有相同的名字，但地址一定是唯一的。

### 这个Hook怎么管理状态

建议使用专门管理 localStorage 的 hook（叫 useLocalStorage）而不是普通的 useState。区别在于，useLocalStorage 会自动把数据同步到浏览器的本地存储里，而你不用手动写读取和写入的代码。这样开发和维护都会简单很多。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useCustomTokens } from '@sushiswap/aptos'

function CustomTokenManager() {
  // 使用 useCustomTokens 获取自定义代币管理接口
  const { data: customTokens, mutate, hasToken } = useCustomTokens()

  // 检查某个代币是否是自定义代币
  const isCustom = hasToken('0x1234...')

  // 添加自定义代币
  const addToken = (token: Token) => {
    mutate('add', [token])
  }

  // 删除自定义代币
  const removeToken = (token: Token) => {
    mutate('remove', [token])
  }

  return (
    <div>
      <h3>自定义代币 ({Object.keys(customTokens ?? {}).length})</h3>
      {Object.values(customTokens ?? {}).map((token) => (
        <div key={token.address}>
          {token.symbol}
          <button onClick={() => removeToken(token)}>删除</button>
        </div>
      ))}
    </div>
  )
}
```

### 常见用法

#### 1. 添加自定义代币功能

```tsx
import { useCustomTokens } from '@sushiswap/aptos'
import { useState } from 'react'

function AddCustomToken() {
  // 使用 useCustomTokens 获取自定义代币管理接口
  const { mutate, hasToken } = useCustomTokens()
  const [tokenAddress, setTokenAddress] = useState('')
  const [error, setError] = useState('')

  const handleAddToken = async () => {
    // 验证地址格式
    if (!tokenAddress.startsWith('0x')) {
      setError('无效的 Aptos 地址格式')
      return
    }

    // 获取代币信息（通常需要调用 API 或使用 useTokenWithCache）
    const tokenInfo = await fetchTokenInfo(tokenAddress)

    if (!tokenInfo) {
      setError('无法获取代币信息')
      return
    }

    // 添加到自定义代币列表
    mutate('add', [tokenInfo])
    setTokenAddress('')
  }

  return (
    <div>
      <input
        value={tokenAddress}
        onChange={(e) => setTokenAddress(e.target.value)}
        placeholder="输入代币地址"
      />
      <button onClick={handleAddToken}>添加</button>
      {error && <div className="error">{error}</div>}
    </div>
  )
}
```

#### 2. 与代币列表合并显示

```tsx
import { useBaseTokens, useCustomTokens, useSortedTokenList } from '@sushiswap/aptos'

function TokenList() {
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

  // 使用 useSortedTokenList 搜索排序
  const { data: sortedTokens } = useSortedTokenList({
    query,
    tokenMap: allTokens
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

#### 3. 检测代币是否已添加

```tsx
import { useCustomTokens } from '@sushiswap/aptos'
import { Token } from '@sushiswap/aptos'

function AddTokenButton({ token }: { token: Token }) {
  // 使用 useCustomTokens 获取自定义代币管理接口
  const { hasToken, mutate } = useCustomTokens()

  // 检查是否已添加
  const isAdded = hasToken(token)

  const handleClick = () => {
    if (isAdded) {
      mutate('remove', [token])
    } else {
      mutate('add', [token])
    }
  }

  return (
    <button onClick={handleClick}>
      {isAdded ? '已添加' : '添加代币'}
    </button>
  )
}
```

### Do（推荐做法）

- **使用 mutate 函数**：通过 mutate 添加或删除代币，不要直接修改 data
- **使用 hasToken 检查**：添加前先检查是否已存在
- **组合 baseTokens 使用**：合并基础代币和自定义代币提供完整列表
- **处理并发更新**：mutate 会合并更新，不需要手动处理

### Don't（不推荐做法）

- **不要直接修改 data**：使用 mutate 函数进行操作
- **不要用 symbol 作为 key**：可能存在重复 symbol，使用地址作为 key
- **不要忽略 localStorage 限制**：注意存储大小限制

### 相关的其他 hooks

- `useBaseTokens` *(一个React hook，用于获取Aptos网络所有基础代币列表，从tokenlist配置中读取)*：获取基础代币
- `useCommonTokens` *(一个React hook，用于获取常用/热门代币列表，只返回APT、lzWBTC、lzUSDC等基础代币)*：获取常用代币
- `useTokenWithCache` *(一个React hook，带缓存的代币信息查询hook，先检查本地缓存，缓存未命中时从链上API获取)*：获取代币详情
- `useLocalStorage` *(一个React hook，用于同步React状态与localStorage，支持自动序列化和反序列化)*：localStorage 同步 hook
