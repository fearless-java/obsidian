> 源代码路径: `packages/hooks/src/usePinnedTokens.ts`

# usePinnedTokens

## 1. 大白话讲讲这个 hook 的作用

`usePinnedTokens` *(一个React hook，用于管理用户置顶的代币列表，基于LocalStorage持久化，支持EVM和SVM两条链)* 是一个管理"置顶代币"的 hook。在 DeFi *(Decentralized Finance，去中心化金融)* 应用中，用户可以把自己常用的代币"钉"（pin）到列表顶部，方便快速访问。

这个 hook 负责：
1. **存储**：把用户置顶的代币列表存到 LocalStorage *(浏览器本地存储，容量约5-10MB)*
2. **添加**：用户可以把某个代币置顶
3. **移除**：用户可以取消置顶某个代币
4. **查询**：检查某个代币是否已被置顶

简单来说：它就是你的"常用代币列表"，帮你记住哪些代币被用户钉在了最上面。

## 2. 讲讲为什么需要封装该 hook

### DeFi 应用的需求

在 SushiSwap 这样的 DEX *(Decentralized Exchange，去中心化交易所)* 中，代币列表可能很长。用户经常交易的代币可能不是默认列表的前几个，而是一些特定的代币。用户希望：
- 把常用的代币钉到顶部
- 下次打开时这些代币还在顶部
- 切换链时保留各自的置顶列表

### 技术挑战

1. **跨链支持**：每个链的置顶代币是独立存储的
2. **持久化**：刷新页面后置顶状态不能丢失
3. **去重**：同一个代币不能被置顶两次 *(用 Set *(JavaScript数据结构，存储唯一值)* 自动去重)*
4. **地址标准化**：EVM 地址要用 checksum 格式 *(以太坊地址的大小写混合格式，用于验证地址有效性)*，Solana 地址保持原样
5. **唯一标识**：用 `chainId:address` *(链ID和代币地址的组合)* 格式

### 封装的好处

- **统一接口**：对调用方屏蔽存储细节
- **跨链统一处理**：不同链的置顶列表用同一个 hook 管理
- **数据不可变**：通过 mutate *(修改数据的函数，返回新对象)* 函数修改，保证状态可预测
- **查询高效**：hasToken 直接返回布尔值

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

无直接参数。内部使用 `useLocalStorage` *(将数据持久化到LocalStorage的hook)* 读取 key 为 `'sushi.pinned-tokens'` 的数据。

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| `data` | `Record<string, string[]>` | 各链的置顶代币列表 |
| `mutate` | `(type, currencyId) => void` | 修改函数，type 为 'add' 或 'remove' |
| `hasToken` | `(currency) => boolean` | 检查某个代币是否已被置顶 |

### 数据结构

```typescript
// LocalStorage 中存储的格式
{
  "1": ["1:0x...tokenA", "1:0x...tokenB"],  // Ethereum 链 *(chainId为1)*
  "137": ["137:0x...tokenC", "137:0x...tokenD"],  // Polygon 链 *(chainId为137)*
  "13998111414": ["13998111414:...tokenE"]  // Solana 链 *(chainId为13998111414)*
}
```

### 执行逻辑

```
初始化:
    ↓
useLocalStorage('sushi.pinned-tokens', {}) *(从LocalStorage读取置顶代币数据)*
    ↓
读取 LocalStorage
    ↓
返回 { data: pinnedTokens, mutate, hasToken }

添加置顶 addPinnedToken(currencyId):
    ↓
解析 currencyId → [chainId, address] *(解析'chainId:address'格式)*
    ↓
标准化地址 getAddress(address) *(EVM用checksum格式，SVM保持原样)*
    ↓
setValue(prev => {
    prev[chainId] = Array.from(new Set([...(prev[chainId] || []), `${chainId}:${address}`])) *(使用Set去重)*
    return prev
})
    ↓
data 更新

移除置顶 removePinnedToken(currencyId):
    ↓
解析 currencyId → [chainId, address]
    ↓
setValue(prev => {
    prev[chainId] = Array.from(new Set(prev[chainId].filter(...))) *(过滤掉要移除的代币)*
    return prev
})
    ↓
data 更新

查询 hasToken(currency):
    ↓
解析 currency → chainId 和 address
    ↓
标准化地址
    ↓
检查 pinnedTokens?.[chainId]?.includes(`${chainId}:${address}`) *(检查是否在列表中)*
    ↓
返回 boolean
```

### 数据流

```
LocalStorage ('sushi.pinned-tokens')
    ↓
pinnedTokens (Record<string, string[]>) *(按chainId分组的代币ID列表)*
    ↓
data (原样返回)
    ↓
mutate('add', id) → setValue → LocalStorage *(通过useLocalStorage写入)*
```

## 四、AI 提示词编写教学

你正在做一个"置顶代币"的小工具。在 DeFi 应用里，代币列表可能很长，用户经常交易的那些代币会被"钉"到最上面，方便快速找到。这个工具帮用户记住哪些代币被钉了，而且刷新页面后这些置顶的代币不能消失。

这个工具要同时支持以太坊链和 Solana 链的代币。不同链的代币是分开存的，比如以太坊上置顶的代币和 Polygon 上置顶的代币是两套不同的列表。用链 ID 来区分，比如"1"是以太坊主网，"137"是 Polygon，"13998111414"是 Solana。

每条代币的标识用"链ID:地址"这样的格式，比如"1:0xABC..."表示以太坊上地址为 0xABC 的代币。

添加置顶的时候，如果用户手抖点了两次，不能让同一个代币出现两次。用 Set（集合）来自动去重。

删除置顶的时候，从列表里把对应的 ID 过滤掉就好。

检查某个代币有没有被置顶，输入可以是代币对象，也可以是字符串 ID，输出布尔值。以太坊的地址要做标准化处理（checksum 格式），Solana 的地址保持原样不动。

持久化存储用的是浏览器本地空间。返回的东西包括：当前所有置顶代币的数据、一个用来增删操作的修改函数、一个检查函数。为了性能，返回的东西都用缓存包裹一下。

## 五、该 hook 的用法教学

### 基本用法

```typescript
import { usePinnedTokens } from './usePinnedTokens'

function PinnedTokensManager() {
  const { data, mutate, hasToken } = usePinnedTokens()

  // 添加代币到置顶列表
  const handlePin = (currencyId) => {
    mutate('add', currencyId)
  }

  // 从置顶列表移除
  const handleUnpin = (currencyId) => {
    mutate('remove', currencyId)
  }

  // 检查是否已置顶
  const isPinned = hasToken(currencyId)

  return (
    <div>
      {/* 渲染置顶代币列表 */}
    </div>
  )
}
```

### 常见使用场景

🟩 **置顶/取消置顶按钮**

```typescript
function TokenRow({ token }) {
  const { mutate, hasToken } = usePinnedTokens()
  const isPinned = hasToken(token)

  return (
    <div>
      <span>{token.symbol}</span>
      <button onClick={() => isPinned ? mutate('remove', token.id) : mutate('add', token.id)}>
        {isPinned ? '取消置顶' : '置顶'}
      </button>
    </div>
  )
}
```

🟩 **显示各链置顶代币**

```typescript
function PinnedTokensList({ chainId }) {
  const { data } = usePinnedTokens()
  const pinnedIds = data[chainId] || []

  return (
    <div>
      {pinnedIds.map(id => (
        <TokenItem key={id} currencyId={id} />
      ))}
    </div>
  )
}
```

🟩 **代币选择器中的置顶标记**

```typescript
function TokenSelector({ tokens, selectedToken }) {
  const { hasToken } = usePinnedTokens()

  return (
    <div>
      {tokens.map(token => (
        <TokenOption
          key={token.id}
          token={token}
          isSelected={token.id === selectedToken}
          isPinned={hasToken(token)}
        />
      ))}
    </div>
  )
}
```

### Dos and Don'ts

✅ **Do: 使用 mutate 函数修改数据**

```typescript
// 正确：使用 mutate 函数添加/移除
mutate('add', '1:0x...')
mutate('remove', '1:0x...')
```

❌ **Don't: 直接修改 data 对象**

```typescript
// 错误：不要直接修改
data[chainId].push(currencyId)

// 正确：通过 mutate 修改
mutate('add', currencyId)
```

✅ **Do: 使用 hasToken 检查状态**

```typescript
// 正确：使用 hasToken 方法
const isPinned = hasToken(token)
if (isPinned) {
  // 已置顶
}
```

❌ **Don't: 用 data 直接判断**

```typescript
// 错误：data 是嵌套对象，不要直接检查
if (data[chainId]?.includes(currencyId)) {
  // 不推荐这种方式
}
```

✅ **Do: 使用 currency ID 格式**

```typescript
// 正确：ID 格式为 'chainId:address'
const currencyId = `${chainId}:${address}`
```

❌ **Don't: 混用不同格式的 ID**

```typescript
// 错误：不要混用不同格式
mutate('add', '0x...') // 缺少 chainId
mutate('add', 'ethereum:0x...') // 错误的格式

// 正确：使用 'chainId:address' 格式
mutate('add', '1:0xA0b86a33E5AbA3D0b2E8c6E5c5B4D3A2E1b0C')
```

✅ **Do: 跨链数据隔离**

```typescript
// 正确：不同链的置顶列表是独立存储的
const ethPinned = data['1'] || []
const polygonPinned = data['137'] || []
const solanaPinned = data['13998111414'] || []
```

❌ **Don't: 假设所有链共享同一个列表**

```typescript
// 错误：不要假设所有链共享置顶列表
const allPinned = data // 不是这样的格式
```
