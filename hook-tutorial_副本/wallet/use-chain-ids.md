> 源代码路径: `apps/web/src/lib/wallet/hooks/use-chain-ids.ts`

# useChainIds Hook 教程

## 1. 大白话讲讲这个hook的作用

`useChainIds` *(一个React hook，用于获取当前已连接钱包所在链的ChainId列表，基于useWallets实现)* 是一个用于**获取当前已连接钱包所在链的 ChainId 列表**的 Hook。

简单来说：
- 当用户连接了钱包，它能告诉你用户在哪条链上
- 如果用户同时连接了 Ethereum 和 Solana，会返回 `[1, 101]`（两个 ChainId *(区块链网络的唯一标识符，如Ethereum主网是1，Polygon是137)*）
- 如果用户只连接了 Ethereum，就返回 `[1]`
- 这个 Hook 返回的是**去重后**的 ChainId 数组

你可以把它想象成"问系统：当前连接的钱包都在哪些链上？"

---

## 2. 讲讲为什么需要封装该hook

### 2.1 链 ID 聚合需求

在多链应用中，经常需要知道用户当前所在的链环境。比如：
- 判断是否需要切换链
- 获取对应链的网络配置
- 决定显示哪些链的资产

### 2.2 与 useWallets 的关系

`useChainIds` 实际上是基于 `useWallets` *(一个hook，返回所有已连接链的完整连接信息)* 实现的，是对 `useWallets` 返回数据的**进一步处理**。

```typescript
// useWallets 返回
{
  evm: { chainId: 1, account: '0x...', ... },
  svm: { chainId: 101, account: '...', ... },
  stellar: undefined  // 未连接
}

// useChainIds 只提取 chainId，并去重
[1, 101]
```

### 2.3 去重逻辑

用户可能通过不同的钱包连接了同一条链多次，或者存在重复数据。`useChainIds` 使用 `Set` *(JavaScript内置对象，用于存储唯一值，自动去重)* 来去重：

```typescript
[evm?.chainId, svm?.chainId, stellar?.chainId]
  .filter(i => i !== undefined)  // 过滤掉 undefined
  .filter((v, i, a) => a.indexOf(v) === i)  // 去重（简化的 Set 逻辑）
// 或者
[...new Set([evm?.chainId, svm?.chainId, stellar?.chainId].filter(Boolean))]
```

---

## 3. 讲讲该hook的执行逻辑和数据流向

### 3.1 函数签名

```typescript
export const useChainIds = () => {
  const { evm, svm, stellar } = useWallets()
  return useMemo(() => {
    return [
      ...new Set(
        [evm?.chainId, svm?.chainId, stellar?.chainId].filter(
          (i) => i !== undefined,
        ),
      ),
    ]
  }, [evm?.chainId, svm?.chainId, stellar?.chainId])
}
```

### 3.2 输入（Input）

无参数。

### 3.3 输出（Output）

```typescript
Array<EvmChainId | SvmChainId | StellarChainId>
```

| 返回值 | 类型 | 说明 |
|--------|------|------|
| 数组 | `(EvmChainId \| SvmChainId \| StellarChainId)[]` | 已连接链的 ChainId 数组 |

### 3.4 常见 ChainId 映射

| 链 | ChainId | 示例值 |
|----|---------|--------|
| Ethereum *(以太坊区块链，主网)* | EvmChainId *(EVM兼容链的ChainId类型定义)* | `1` |
| Polygon *(Polygon Layer2区块链网络)* | EvmChainId | `137` |
| Arbitrum *(以太坊Layer2扩展解决方案)* | EvmChainId | `42161` |
| Solana *(Solana区块链网络)* | SvmChainId *(Solana链的ChainId类型定义)* | `101` |
| Stellar *(Stellar区块链网络)* | StellarChainId *(Stellar链的ChainId类型定义)* | `'stellar'` (字符串) |

### 3.5 内部执行逻辑

```typescript
return useMemo(() => {
  // 1. 收集三条链的 chainId（可能是 undefined）
  const chainIds = [
    evm?.chainId,    // number | undefined
    svm?.chainId,     // number | undefined
    stellar?.chainId, // number | undefined
  ]

  // 2. 过滤掉 undefined
  const definedChainIds = chainIds.filter((i) => i !== undefined)

  // 3. Set 去重（虽然通常不会有重复）
  const uniqueChainIds = [...new Set(definedChainIds)]

  // 4. 返回数组
  return uniqueChainIds
}, [evm?.chainId, svm?.chainId, stellar?.chainId])
```

### 3.6 数据流向图

```
useChainIds()
     │
     ▼
useWallets()
     │
     ├──► evm?.chainId ──┐
     ├──► svm?.chainId  ─┼─► [filter] ─► [Set去重] ─► [chainId1, chainId2, ...]
     └──► stellar?.chainId─┘
```

### 3.7 使用示例

```typescript
// 用户只连接了 MetaMask（Ethereum）
const chainIds = useChainIds()
// chainIds = [1]

// 用户连接了 MetaMask 和 Phantom
const chainIds = useChainIds()
// chainIds = [1, 101]

// 用户连接了三条链
const chainIds = useChainIds()
// chainIds = [1, 101, 'stellar']
```

---

## 四、AI 提示词编写教学

你正在做一个需要知道用户当前在哪些链上的功能。这个 Hook 会返回用户已连接的所有链的 ID，用起来很直接。

先确定一件事：这个 Hook 返回的是数组，不是单独一个值。哪怕用户只连了一条链，返回的也是 `[1]` 这样的数组，不是 `1`。这是最容易搞错的地方。

最常见的用法是检查用户是否在特定链上。比如想知道用户在不在以太坊主网，就用 `chainIds.includes(1)` 来判断，返回 true 就是在了，false 就是不在。想检查 Polygon 就用 `chainIds.includes(137)`。但要注意，Stellar 的 ID 是字符串 `'stellar'`，不是数字，EVM 和 Solana 的 ID 都是数字。

有时候你只想知道用户有没有连任何链。简单，看数组长度就行。大于 0 就是至少连了一条，等于 0 就是完全没连。这个信息对于决定显示连接按钮还是用户界面很有用。

还有个常见场景是根据已连接的链来展示网络配置。你可以遍历所有链 ID，为每条链生成配置对象，包含了 RPC 地址、浏览器链接、符号等信息，这样能给用户一个完整的网络列表。

新手最容易犯的错是把返回值当单个值用。比如直接 `const chainId = useChainIds()` 然后 `switchNetwork(chainId)`，但 chainId 是数组，应该传数字而不是数组。正确的做法是先检查数组里有没有那个链 ID，如果有才切换。

另一个常犯的错是假设用户只连了一条链。比如 `const [chainId] = useChainIds()` 然后只处理这一条，但如果用户同时连了以太坊和 Solana，你的逻辑就只处理了第一条，漏掉了 Solana。应该用循环或者 map 处理所有连接的链。

不同链的 ID 类型不一样。EVM 链（以太坊、Polygon、Arbitrum）和 Solana 的 ID 都是数字，但 Stellar 的 ID 是字符串 `'stellar'`。所以返回的数组可能是 `[1, 101, 'stellar']` 这样的混合类型。如果你要对 ID 做类型相关的操作，比如判断是不是数字，需要用类型守卫来过滤。

用户可能一条链都没连接。这种情况下返回的是空数组 `[]`，不是 null 也不是 undefined。任何直接访问数组元素的地方都要先检查长度。

这个 Hook 自己不管理状态，纯粹是根据传入的连接信息计算出一个链 ID 数组。当用户连接或断开钱包时，底层数据变化了，它就重新计算返回新的数组。

---

## 5. 该 hook 的用法教学

### 5.1 基础用法

```typescript
import { useChainIds } from '@sushiswap/wallet'

// ✅ 最基本的使用
function ChainIdsDisplay() {
  const chainIds = useChainIds()

  return (
    <div>
      <p>已连接 {chainIds.length} 条链</p>
      <p>ChainIds: {chainIds.join(', ')}</p>
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景1：检查特定链是否已连接

```typescript
function ChainChecker() {
  const chainIds = useChainIds()

  const isOnEthereum = chainIds.includes(1)
  const isOnPolygon = chainIds.includes(137)
  const isOnSolana = chainIds.includes(101)
  const isOnStellar = chainIds.includes('stellar')

  return (
    <div className="chain-checker">
      <p>Ethereum: {isOnEthereum ? '✅' : '❌'}</p>
      <p>Polygon: {isOnPolygon ? '✅' : '❌'}</p>
      <p>Solana: {isOnSolana ? '✅' : '❌'}</p>
      <p>Stellar: {isOnStellar ? '✅' : '❌'}</p>
    </div>
  )
}
```

#### 场景2：根据连接链显示对应功能

```typescript
function FeatureBasedOnChain() {
  const chainIds = useChainIds()

  const canSwap = chainIds.includes(1) || chainIds.includes(137) || chainIds.includes(42161)
  const canBridge = chainIds.includes(1) && chainIds.includes(137)
  const canStakeSolana = chainIds.includes(101)

  return (
    <div className="features">
      {canSwap && <SwapButton />}
      {canBridge && <BridgeButton />}
      {canStakeSolana && <SolanaStakeButton />}
      {!canSwap && !canBridge && !canStakeSolana && (
        <p>请连接支持的链以使用更多功能</p>
      )}
    </div>
  )
}
```

#### 场景3：动态网络配置获取

```typescript
function NetworkConfigs() {
  const chainIds = useChainIds()

  const networkConfigs = useMemo(() => {
    return chainIds.map((chainId) => {
      // 根据chainId获取对应配置
      const config = getNetworkConfig(chainId)
      return {
        chainId,
        name: config.name,
        rpcUrl: config.rpcUrl,
        explorer: config.explorer,
        symbol: config.symbol,
      }
    })
  }, [chainIds])

  return (
    <div className="network-list">
      {networkConfigs.map((config) => (
        <NetworkCard key={config.chainId} config={config} />
      ))}
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **始终处理空数组情况**
   ```typescript
   // ✅ 正确
   const chainIds = useChainIds()
   if (chainIds.length === 0) {
     return <ConnectPrompt />
   }
   ```

2. **使用 includes 检查特定链**
   ```typescript
   // ✅ 正确
   if (chainIds.includes(1)) {
     // 在Ethereum上
   }
   ```

3. **注意链ID类型的混合**
   ```typescript
   // ✅ 正确 - Stellar使用字符串
   const chainIds = useChainIds()
   const isStellar = chainIds.includes('stellar' as any)
   ```

4. **使用 useMemo 缓存派生数据**
   ```typescript
   // ✅ 正确
   const evmChainIds = useMemo(() => {
     return chainIds.filter((id): id is number => typeof id === 'number')
   }, [chainIds])
   ```

#### ❌ Don'ts

1. **不要假设返回单个值**
   ```typescript
   // ❌ 错误
   const chainId = useChainIds()
   switchNetwork(chainId)

   // ✅ 正确
   const chainIds = useChainIds()
   if (chainIds.includes(targetChainId)) {
     switchNetwork(targetChainId)
   }
   ```

2. **不要假设至少有一条链连接**
   ```typescript
   // ❌ 错误
   const [primaryChain] = useChainIds()
   getConfig(primaryChain) // primaryChain可能是undefined

   // ✅ 正确
   const chainIds = useChainIds()
   if (chainIds.length > 0) {
     const primaryChain = chainIds[0]
   }
   ```

3. **不要忽略链ID类型差异**
   ```typescript
   // ❌ 错误 - Stellar使用字符串ID
   const chainId = useChainIds()[0]
   if (chainId === 1) { // 对Stellar无效
     // ...
   }

   // ✅ 正确 - 使用includes或类型守卫
   if (chainIds.includes(1)) {
     // Ethereum逻辑
   }
   ```

4. **不要在依赖数组中直接使用对象**
   ```typescript
   // ❌ 错误 - 对象引用会变化
   }, [evm, svm, stellar])

   // ✅ 正确 - 使用具体属性
   }, [evm?.chainId, svm?.chainId, stellar?.chainId])
   ```
