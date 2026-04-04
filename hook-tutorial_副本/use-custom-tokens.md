> 源代码路径: `packages/hooks/src/useCustomTokens.ts`

# useCustomTokens

## 1. 大白话讲讲这个 hook 的作用

`useCustomTokens` *(一个React hook，用于管理用户自定义添加的代币列表，支持EVM和SVM两条链，基于LocalStorage持久化)* 是一个管理用户自定义添加的 Token（代币）的 hook。在 DeFi *(Decentralized Finance，去中心化金融，如DEX、借贷等)* 应用中，用户可以添加任何 ERC-20 *(Ethereum Request for Comment 20，代币标准，规定了代币接口，如transfer、balanceOf等)* 代币或 Solana SPL *(Solana Program Library，Solana链上的代币标准，类似以太坊的ERC-20)* 代币到自己的"自定义代币列表"中。这个 hook 负责：

1. **存储**：把用户添加的自定义代币存到 LocalStorage *(浏览器本地存储，容量约5-10MB，可跨会话持久保存)*
2. **读取**：从 LocalStorage 读取之前保存的自定义代币
3. **添加**：用户可以添加新的自定义代币
4. **删除**：用户可以移除已添加的自定义代币
5. **查询**：检查某个代币是否已经在自定义列表中

简单来说：它就是你的"自定义代币小本本"，帮你记住用户自己添加的各种代币。

## 2. 讲讲为什么需要封装该 hook

### DeFi 应用的需求
在 SushiSwap 这样的 DEX *(Decentralized Exchange，去中心化交易所，如Uniswap、SushiSwap)* 中，用户交易的不一定是默认列表里的主流代币，可能是：
- 新上线的 Token
- 社区发行的 Token
- 用户自己发现的小众 Token

### 技术挑战
1. **跨链支持**：需要同时支持 EVM（以太坊等）和 SVM（Solana）链的代币
2. **持久化**：用户刷新页面后，自定义代币不能丢失
3. **数据转换**：存储的是简单数据，需要转换成 Token 对象
4. **唯一标识**：用 `chainId:address` *(链ID和代币地址的组合，用于唯一标识某条链上的某个代币)* 作为唯一 key
5. **性能**：自定义代币列表不能每次渲染都重新计算

### 封装的好处
- **统一接口**：对调用方屏蔽存储细节
- **跨链统一处理**：EVM 和 SVM 使用同一套逻辑
- **数据不可变**：通过 mutate *(修改数据的函数，通常返回新对象而非修改原对象)* 函数修改，保证状态可预测
- **memo 优化**：返回值用 useMemo *(React hook，用于缓存计算结果，避免每次渲染都重新计算)* 包裹，避免不必要的重渲染

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

无直接参数。内部使用 `useLocalStorage` *(将数据持久化到浏览器LocalStorage的hook，支持跨标签页同步)* 读取 key 为 `'sushi.customTokens'` 的数据。

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| `data` | `Record<string, Token>` | 所有自定义代币，key 是 `chainId:address` |
| `mutate` | `(type, currency[]) => void` | 修改函数，type 为 'add' 或 'remove' |
| `hasToken` | `(currency) => boolean` | 检查某个代币是否已添加 |

### 执行逻辑

```
初始化:
    ↓
useLocalStorage('sushi.customTokens', {}) *(从LocalStorage读取自定义代币数据)*
    ↓
读取 LocalStorage 中的 JSON *(JavaScript Object Notation，轻量级数据交换格式)* 数据
    ↓
hydrate(value) → 转换成 Token 对象 *(将存储的JSON数据转换成Token类的实例)*
    ↓
返回 { data, mutate, hasToken }

添加代币 addCustomToken(currencies):
    ↓
将 Token 对象转换成存储格式 { chainId, address, decimals, name, symbol, logoUrl }
    ↓
setValue(prev => { ...prev, 'chainId:address': data })
    ↓
触发 useLocalStorage 的事件监听
    ↓
storedValue 更新 → hydrate 重新执行
    ↓
data 更新

移除代币 removeCustomToken(currency):
    ↓
setValue(prev => Object.entries(prev).filter(...))
    ↓
移除对应的 'chainId:address' key
    ↓
data 更新
```

### 数据流

```
LocalStorage ('sushi.customTokens')
    ↓
value (原始 JSON 对象)
    ↓
hydrate(value) → 转换成 EvmToken/SvmToken 实例 *(EvmToken用于以太坊等EVM兼容链，SvmToken用于Solana)*
    ↓
data (Token 对象 Map)

用户调用 mutate('add', [token])
    ↓
addCustomToken → 数据转换
    ↓
setValue → 写入 LocalStorage
    ↓
触发事件 → storedValue 更新
    ↓
data 重新计算
```

## 四、AI 提示词编写教学

你在做一个代币管理的小工具，用户可以往里面添加自己找到的各种代币，这些代币可能是以太坊链上的，也可能是 Solana 链上的，而且用户刷新页面后这些自定义代币不能消失。

先决定把这些数据存在哪里。既然是持久化需求，那就用浏览器本地的存储空间。给这批数据起个唯一的名字，比如"sushi.customTokens"，这样才不会和其他数据混在一起。

每条代币记录要能说清楚它是哪条链上的、地址是什么。用"链ID:地址"这样的格式来唯一标识一条代币，比如"1:0xABC..."是以太坊上的某个代币，"13998111414:7xK..."是 Solana 上的。

用户添加代币的时候，要把代币对象转成可以存储的简单数据格式（名字、符号、地址、小数位数等），然后写进本地存储。注意以太坊的地址要用标准格式，Solana 的地址保持原样。

删除代币的时候相反，要从存储里把对应的那条记录去掉。

还需要一个检查函数，输入一个代币，能告诉你"这个代币在不在用户的自定义列表里"。这个检查要支持直接传代币对象，也支持传字符串 ID。

最后返回三样东西：一个包含所有自定义代币的数据对象、一个用来增删操作的修改函数、还有一个检查函数。为了性能，这些返回的东西最好都缓存一下，不要每次渲染都重新计算。

## 五、该 hook 的用法教学

### 基本用法

```typescript
import { useCustomTokens } from './useCustomTokens'

function CustomTokenManager() {
  const { data, mutate, hasToken } = useCustomTokens()

  // 添加代币
  const handleAddToken = (token) => {
    mutate('add', [token])
  }

  // 移除代币
  const handleRemoveToken = (token) => {
    mutate('remove', [token])
  }

  // 检查代币是否存在
  const isAdded = hasToken(token)

  return (
    <div>
      {/* 渲染自定义代币列表 */}
    </div>
  )
}
```

### 常见使用场景

🟩 **添加自定义代币**

```typescript
function AddTokenForm() {
  const { mutate } = useCustomTokens()

  const handleSubmit = (tokenData) => {
    // tokenData 需要包含: chainId, address, decimals, name, symbol
    const token = new EvmToken(tokenData)
    mutate('add', [token])
  }

  return <form onSubmit={handleSubmit}>...</form>
}
```

🟩 **检查代币是否已添加**

```typescript
function TokenList({ tokens }) {
  const { hasToken } = useCustomTokens()

  return (
    <div>
      {tokens.map(token => (
        <div key={token.id}>
          <span>{token.symbol}</span>
          {hasToken(token) && <span>已添加</span>}
        </div>
      ))}
    </div>
  )
}
```

🟩 **显示自定义代币列表**

```typescript
function CustomTokenList() {
  const { data } = useCustomTokens()

  const customTokens = Object.values(data)

  return (
    <div>
      {customTokens.map(token => (
        <div key={token.id}>
          <img src={token.logoUrl} alt={token.symbol} />
          <span>{token.symbol}</span>
          <span>{token.name}</span>
        </div>
      ))}
    </div>
  )
}
```

### Dos and Don'ts

✅ **Do: 使用 mutate 函数修改数据**

```typescript
// 正确：使用 mutate 函数
mutate('add', [newToken])
mutate('remove', [existingToken])
```

❌ **Don't: 直接修改 data 对象**

```typescript
// 错误：不要直接修改
data[tokenId] = newToken
```

✅ **Do: 使用 hasToken 检查代币是否存在**

```typescript
// 正确：使用 hasToken 方法
if (hasToken(token)) {
  // 代币已添加
}
```

❌ **Don't: 用 data 直接判断**

```typescript
// 错误：data 是 Record，不要直接检查
if (data[tokenId]) {
  // 不推荐这种方式
}
```

✅ **Do: 传入完整的 Token 对象**

```typescript
// 正确：传入完整的 Token 对象
mutate('add', [EvmToken.fromCurrency(currency)])
```

❌ **Don't: 传入不完整的数据**

```typescript
// 错误：必须包含必要字段
mutate('add', [{ symbol: 'TEST' }]) // 缺少 address, chainId 等
```

✅ **Do: 使用 chainId:address 格式的 ID**

```typescript
// 正确：ID 格式为 'chainId:address'
const tokenId = `${chainId}:${address}`
```

❌ **Don't: 混用不同链的代币**

```typescript
// 错误：不同链的代币要分开处理
// EVM 代币和 SVM 代币的地址格式不同
```
