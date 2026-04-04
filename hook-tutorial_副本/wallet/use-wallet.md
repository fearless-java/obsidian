> 源代码路径: `apps/web/src/lib/wallet/hooks/use-wallet.ts`

# useWallet Hook 教程

## 1. 大白话讲讲这个hook的作用

`useWallet` *(一个React hook，用于获取当前已连接钱包的完整连接信息，基于WalletContext的connections数组实现)* 是一个用于**获取当前已连接钱包的完整连接信息**的 Hook。

简单来说：
- 与 `useAccount` *(一个hook，只返回钱包地址)* 只返回地址不同，`useWallet` 返回**完整的连接对象**
- 这个对象包含：钱包 ID、名称、namespace、链 ID、账户地址、图标等
- 支持按 namespace（链类型）或 chainId（链 ID）筛选
- 如果用户没有连接任何钱包，返回 `undefined`

你可以把它想象成"问系统：当前连接的完整钱包信息是什么？"，而 `useAccount` 只是回答"地址是什么"。

---

## 2. 讲讲为什么需要封装该hook

### 2.1 需要完整的连接信息

很多时候，我们不仅需要地址，还需要其他信息：

```typescript
// useAccount 只能给你地址
const address = useAccount('evm')
// address: '0x1234...'

// useWallet 给你完整信息
const wallet = useWallet('evm')
// wallet: {
//   id: 'evm:io.metamask',
//   name: 'MetaMask',
//   namespace: 'evm',
//   chainId: 1,
//   account: '0x1234...',
//   icon: 'data:image/...'
// }
```

使用场景：
- 显示钱包图标和名称
- 获取当前链 ID 进行网络切换
- 访问钱包的 uid（用于某些特殊操作）

### 2.2 多维度筛选

`useWallet` 支持两种筛选方式：

1. **按 namespace 筛选**：`'evm'` | `'svm'` | `'stellar'`
2. **按 chainId 筛选**：`1` (Ethereum *(以太坊主网)*) | `101` (Solana *(Solana主网)*) | `'stellar'` (Stellar *(Stellar网络)*)

这比 `useAccount` *(一个hook，通过chainId查找时会先映射到namespace)* 更灵活。

### 2.3 与其他 Hook 的对比

| Hook | 返回内容 | 适用场景 |
|------|---------|---------|
| `useAccount` *(单个链的地址)* | 仅地址 `address` | 只关心地址 |
| `useAccounts` *(所有链的地址)* | `{ evm, svm, stellar }` 每个只有 address | 需要多链地址 |
| `useWallet` *(单个链的完整信息)* | 完整 WalletConnection *(已连接钱包的完整信息对象)* 对象 | 需要连接详情 |
| `useWallets` *(所有链的完整信息)* | `{ evm, svm, stellar }` 每个是完整连接 | 需要多链完整信息 |

---

## 3. 讲讲该hook的执行逻辑和数据流向

### 3.1 函数签名（重载）

```typescript
// 按 namespace 获取
export function useWallet<TNamespace extends WalletNamespace>(
  namespace?: TNamespace,
): WalletConnection<ChainIdForNamespace<TNamespace>> | undefined

// 按 chainId 获取
export function useWallet<
  TChainId extends EvmChainId | SvmChainId | StellarChainId,
>(chainId: TChainId): WalletConnection<TChainId> | undefined

// 无参数版本
export function useWallet(
  filter?: EvmChainId | SvmChainId | StellarChainId | WalletNamespace,
)
```

### 3.2 输入（Input）

| 参数 | 类型 | 说明 |
|------|------|------|
| `filter` | `EvmChainId \| SvmChainId \| StellarChainId \| WalletNamespace \| undefined` | 可选参数，用于筛选 |

- 如果传**数字**（如 `1`、`101`），按 `chainId` *(区块链网络的唯一标识符)* 查找
- 如果传**字符串**（如 `'evm'`），按 `namespace` *(区块链命名空间，如'evm'、'svm'、'stellar')* 查找
- 不传：默认返回 `connections[0]`

### 3.3 输出（Output）

```typescript
WalletConnection<TChainId> | undefined
```

| 属性 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | 钱包唯一标识符，如 `'evm:io.metamask'` |
| `name` | `string` | 钱包名称，如 `'MetaMask'` |
| `namespace` | `'evm' \| 'svm' \| 'stellar'` | 链类型 |
| `account` | `AddressFor<TChainId>` | 钱包地址 |
| `chainId` | `TChainId` | 链 ID |
| `icon` | `string \| undefined` | 钱包图标（base64 *(一种将二进制数据编码为可打印字符的编码方式，常用于图片数据传输)* 或 URL） |

### 3.4 内部执行逻辑

```typescript
export function useWallet(filter) {
  // 1. 从 Context 获取连接
  const { connections } = useWalletContext()

  // 2. 解析 filter
  const namespace = typeof filter === 'string' ? filter : undefined
  const chainId = typeof filter === 'number' ? filter : undefined

  // 3. 查找匹配的连接
  return useMemo(() => {
    let connection

    if (typeof chainId === 'number') {
      // 按 chainId 查找
      connection = connections.find((c) => c.chainId === chainId)
    } else if (typeof namespace === 'string') {
      // 按 namespace 查找
      connection = connections.find((c) => c.namespace === namespace)
    } else {
      // 默认返回第一个
      connection = connections[0]
    }

    return connection  // 可能是 undefined
  }, [connections, chainId, namespace])
}
```

### 3.5 数据流向图

```
useWallet('evm')
     │
     ├──► filter = 'evm' (string) ──► namespace = 'evm'
     │
     ▼
useWalletContext()
     │
     ▼
connections: WalletConnection[]
     │
     └──► connections.find(c => c.namespace === 'evm')
              │
              ▼
         WalletConnection | undefined
```

### 3.6 使用示例

```typescript
// ✅ 获取 EVM 连接
const evmWallet = useWallet('evm')
// { id: 'evm:io.metamask', name: 'MetaMask', chainId: 1, account: '0x...', ... }

// ✅ 按 chainId 获取
const ethWallet = useWallet(1)  // Ethereum
const solWallet = useWallet(101)  // Solana

// ✅ 获取第一个连接（不指定 filter）
const firstWallet = useWallet()

// ✅ 安全使用
function WalletInfo() {
  const wallet = useWallet('evm')
  if (!wallet) return <ConnectButton />

  return (
    <div>
      <img src={wallet.icon} alt={wallet.name} />
      <span>{wallet.name}</span>
      <span>{wallet.account}</span>
      <ChainBadge chainId={wallet.chainId} />
    </div>
  )
}
```

---

## 四、AI 提示词编写教学

你正在做一个需要显示用户已连接钱包信息的功能，可能还需要切换网络什么的。这个 Hook 返回的是完整的已连接钱包信息，不只是地址。

先确定一件事：返回值可能是 undefined。用户可能根本没连接那条链的钱包，所以一定要先检查存在了再使用它的属性。

这个 Hook 最常见的用法是展示钱包按钮。你可以用它来显示图标、名称和截断后的地址。如果用户还没连接任何钱包，就显示连接按钮。关键是要先检查返回值是否存在，只有在有连接的情况下才能访问钱包对象的属性，比如 `wallet.account`、`wallet.chainId`。

另一个常见场景是网络切换。你可以获取当前连接的链 ID，然后判断用户是不是在目标链上。比如你想切换到 Polygon，先检查当前链 ID 是不是 137，如果不是就调用切换函数。按钮文字也要动态显示，如果已经在 Polygon 上了就显示"已是 Polygon"，否则显示"切换到 Polygon"。

如果你的应用支持多链，还可以同时获取多条链的连接信息。比如同时调用 `useWallet('evm')` 和 `useWallet('svm')`，然后分别显示对应的钱包徽章。这样用户一眼就能看到自己在多条链上的连接状态。

新手最容易犯的错是不检查返回值就直接使用。比如直接 `const wallet = useWallet('evm')` 然后 `console.log(wallet.account)`，但如果用户根本没连接 EVM 钱包，wallet 就是 undefined，访问它的属性会直接报错。正确做法是先检查 `if (wallet)`，只有存在的情况下才使用它的属性。

另一个容易混淆的地方是传数字和传字符串的区别。传数字比如 `useWallet(1)` 是按 chainId 查找，不是按 namespace。这意味着如果用户连接的是 Polygon（chainId=137），那 `useWallet(1)` 会返回 undefined，因为找不到 chainId 等于 1 的连接。如果你想找 EVM 链，应该传字符串 `useWallet('evm')`，这样会找到第一个 namespace 为 'evm' 的连接。

按 chainId 查找可能返回 undefined。比如用户连的是以太坊主网（chainId=1），那 `useWallet(1)` 能找到，但 `useWallet(137)` 就找不到，因为用户不在 Polygon 上。

按 namespace 查找只返回第一个匹配的连接。如果用户通过多个浏览器扩展同时连接了多个 EVM 钱包，这个 Hook 只会返回其中第一个。要获取所有连接，得用 `useWallets`。

还有个重要区别：这个 Hook 返回的是 WalletConnection 类型，不是 WalletWithState 类型。WalletConnection 有 account 和 chainId，但没有 isInstalled、isAvailable 这些属性。如果你想检查钱包是否安装，要用 `useEvmWallets` 那个 Hook，它返回的是 WalletWithState 类型，有 isInstalled 等属性，但没有 account。

这个 Hook 自己不管理状态，是纯读取的。数据来自上层 Context 提供的 connections 数组。当用户连接或断开钱包时，底层数据变化了，它就重新计算并返回新的连接信息或者 undefined。

---

## 5. 该 hook 的用法教学

### 5.1 基础用法

```typescript
import { useWallet } from '@sushiswap/wallet'

// ✅ 最基本的使用
function WalletInfo() {
  const wallet = useWallet('evm')

  if (!wallet) {
    return <ConnectButton />
  }

  return (
    <div>
      <img src={wallet.icon} alt={wallet.name} />
      <span>{wallet.name}</span>
      <span>{truncateAddress(wallet.account)}</span>
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景1：钱包按钮组件

```typescript
function WalletButton() {
  const wallet = useWallet('evm')

  if (!wallet) {
    return (
      <button className="connect-btn">
        <WalletIcon />
        <span>连接钱包</span>
      </button>
    )
  }

  return (
    <button className="wallet-btn">
      <img src={wallet.icon} alt={wallet.name} className="wallet-icon" />
      <span className="wallet-name">{wallet.name}</span>
      <span className="wallet-address">{truncateAddress(wallet.account)}</span>
    </button>
  )
}
```

#### 场景2：网络切换器

```typescript
function NetworkSwitcher() {
  const wallet = useWallet('evm')
  const targetChains = [
    { id: 1, name: 'Ethereum', icon: ethIcon },
    { id: 137, name: 'Polygon', icon: polygonIcon },
    { id: 42161, name: 'Arbitrum', icon: arbitrumIcon },
  ]

  const switchTo = async (chainId: number) => {
    if (!wallet || wallet.chainId === chainId) return
    try {
      await switchChain({ chainId })
    } catch (error) {
      console.error('切换网络失败:', error)
    }
  }

  return (
    <div className="network-switcher">
      {targetChains.map((chain) => (
        <button
          key={chain.id}
          onClick={() => switchTo(chain.id)}
          className={wallet?.chainId === chain.id ? 'active' : ''}
        >
          <img src={chain.icon} alt={chain.name} />
          <span>{chain.name}</span>
        </button>
      ))}
    </div>
  )
}
```

#### 场景3：多链头部组件

```typescript
function MultiChainHeader() {
  const evmWallet = useWallet('evm')
  const svmWallet = useWallet('svm')
  const stellarWallet = useWallet('stellar')

  return (
    <header className="multi-chain-header">
      {evmWallet && (
        <WalletBadge
          chain="Ethereum"
          icon={evmWallet.icon}
          address={evmWallet.account}
          chainId={evmWallet.chainId}
        />
      )}
      {svmWallet && (
        <WalletBadge
          chain="Solana"
          icon={svmWallet.icon}
          address={svmWallet.account}
          chainId={svmWallet.chainId}
        />
      )}
      {stellarWallet && (
        <WalletBadge
          chain="Stellar"
          icon={stellarWallet.icon}
          address={stellarWallet.account}
        />
      )}
      {!evmWallet && !svmWallet && !stellarWallet && (
        <ConnectWalletButton />
      )}
    </header>
  )
}
```

#### 场景4：根据链ID显示不同功能

```typescript
function ChainSpecificFeatures() {
  const wallet = useWallet('evm')

  if (!wallet) {
    return <ConnectPrompt />
  }

  const features = getFeaturesForChain(wallet.chainId)

  return (
    <div className="features">
      {features.map((feature) => (
        <FeatureButton key={feature.id} feature={feature} />
      ))}
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **始终检查 undefined**
   ```typescript
   // ✅ 正确
   const wallet = useWallet('evm')
   if (!wallet) return <ConnectButton />
   // 使用 wallet.account 等
   ```

2. **使用 namespace 或 chainId 明确指定链**
   ```typescript
   // ✅ 正确 - 明确指定namespace
   const wallet = useWallet('evm')

   // ✅ 正确 - 按chainId查找
   const wallet = useWallet(1)
   ```

3. **使用可选链操作符安全访问属性**
   ```typescript
   // ✅ 正确
   const chainId = wallet?.chainId
   ```

4. **在多链场景下分别获取每个链的连接**
   ```typescript
   // ✅ 正确
   const evmWallet = useWallet('evm')
   const svmWallet = useWallet('svm')
   const stellarWallet = useWallet('stellar')
   ```

#### ❌ Don'ts

1. **不要直接访问可能为undefined的属性**
   ```typescript
   // ❌ 错误
   const wallet = useWallet('evm')
   console.log(wallet.account.toLowerCase()) // wallet可能是undefined

   // ✅ 正确
   const wallet = useWallet('evm')
   if (wallet) {
     console.log(wallet.account.toLowerCase())
   }
   ```

2. **不要混淆WalletConnection和WalletWithState**
   ```typescript
   // ❌ 错误 - WalletConnection没有isInstalled属性
   const wallet = useWallet('evm')
   if (wallet.isInstalled) { // 编译错误

   // ✅ 正确 - WalletConnection有account和chainId
   // WalletWithState有isInstalled、isAvailable、isRecent
   ```

3. **不要假设第一个连接是特定链的**
   ```typescript
   // ❌ 错误 - 不传参数可能返回任意链的连接
   const wallet = useWallet()
   wallet.namespace // 可能是任意值

   // ✅ 正确 - 明确指定链
   const wallet = useWallet('evm')
   ```

4. **不要用chainId和namespace混合查询**
   ```typescript
   // ❌ 错误 - 行为不确定
   useWallet('evm') // 传入字符串
   useWallet(1) // 传入数字
   // 如果同时传，chainId优先

   // ✅ 正确 - 选择一种方式并坚持使用
   useWallet('evm')
   // 或
   useWallet(1)
   ```

5. **不要忘记返回undefined的情况**
   ```typescript
   // ❌ 错误
   const wallet = useWallet('evm')
   const chainName = getChainName(wallet.chainId) // wallet可能是undefined

   // ✅ 正确
   const wallet = useWallet('evm')
   if (wallet) {
     const chainName = getChainName(wallet.chainId)
   }
   ```
