> 源代码路径: `apps/web/src/lib/wallet/components/wallet-connectors-list/use-wallets-list.ts`

# useWalletsList Hook 教程

## 1. 大白话讲讲这个hook的作用

`useWalletsList` *(一个React hook，用于获取所有可用钱包列表，包含EVM/Solana/Stellar三条链，基于WalletsRegistry实现)* 是一个用于**获取所有可用钱包列表**的 Hook（包含 EVM、Solana、Stellar 三条链）。

简单来说：
- 它返回的是一个**扁平化的钱包数组** `WalletWithState[]` *(包含钱包状态信息的接口，有isInstalled、isAvailable、isRecent等属性)*
- 这个列表包含了**所有支持的钱包**，包括：
  - 已安装/可用的钱包
  - 未安装/不可用的钱包
  - 最近使用过的钱包（标记 `isRecent: true`）
- 每 个钱包都有状态标记：`isInstalled`（是否安装）、`isAvailable`（是否可用）、`isRecent`（最近是否使用过）

你可以把它想象成"问系统：所有支持的钱包有哪些？每个的安装状态如何？"

---

## 2. 讲讲为什么需要封装该hook

### 2.1 统一的钱包选择列表

在钱包选择 UI 中，需要展示所有支持的钱包选项：

```typescript
// 以前：可能需要分别调用
const evmWallets = useEvmWallets()
const svmWallets = useSvmWallets()
const stellarWallets = useStellarWallets()

// 现在：统一获取
const allWallets = useWalletsList()
```

### 2.2 底层实现机制

`useWalletsList` 是基于 `useWalletsRegistry` 实现的，它内部调用了三个 namespace 特定的 hook：
- `useEvmWallets()` *(获取EVM链的钱包列表)*
- `useSvmWallets()` *(获取Solana链的钱包列表)*
- `useStellarWallets()` *(获取Stellar链的钱包列表)*

然后将结果存储在一个 `Map<WalletNamespace, WalletWithState[]>` 中，通过 `WalletsRegistryProvider` 统一管理。

### 2.3 扁平化的数据结构

```typescript
// useEvmWallets 返回: WalletWithState[]
// useSvmWallets 返回: WalletWithState[]
// useStellarWallets 返回: WalletWithState[]

// useWalletsList 返回: WalletWithState[] (所有链的合并)
[
  ...evmWallets,    // EVM 钱包
  ...svmWallets,    // Solana 钱包
  ...stellarWallets // Stellar 钱包
]
```

### 2.4 与 namespace-specific hooks 的关系

| Hook | 作用域 | 返回类型 |
|------|--------|---------|
| `useEvmWallets` *(EVM链专用)* | 仅 EVM | `WalletWithState[]` |
| `useSvmWallets` *(Solana链专用)* | 仅 Solana | `WalletWithState[]` |
| `useStellarWallets` *(Stellar链专用)* | 仅 Stellar | `WalletWithState[]` |
| `useWalletsList` *(通用)* | 所有链 | `WalletWithState[]` (扁平化) |

---

## 3. 讲讲该hook的执行逻辑和数据流向

### 3.1 函数签名

```typescript
export const useWalletsList = () => {
  const { wallets } = useWalletsRegistry()

  return useMemo(() => Array.from(wallets.values()).flat(), [wallets])
}
```

### 3.2 输入（Input）

无参数。

### 3.3 输出（Output）

```typescript
WalletWithState[]
```

`WalletWithState` 接口：

```typescript
interface WalletWithState extends Wallet {
  isInstalled: boolean  // 是否已安装到浏览器
  isAvailable: boolean // 当前是否可用
  isRecent: boolean     // 最近是否使用过
}
```

`Wallet` 接口：

```typescript
interface Wallet {
  id: string                        // 'evm:io.metamask'
  namespace: 'evm' | 'svm' | 'stellar'
  name: string                      // 'MetaMask'
  icon: string                      // base64 或 URL
  adapterId: string                 // 适配器 ID
  url?: string                      // 钱包官网
  uid?: string                      // 钱包 uid
}
```

### 3.4 内部执行逻辑

```typescript
return useMemo(() => {
  // 1. wallets 是 Map<WalletNamespace, WalletWithState[]>
  // wallets.values() 取出所有数组迭代器
  // [[evmWallets], [svmWallets], [stellarWallets]]

  // 2. Array.from() 转为数组
  // [[evmWallets], [svmWallets], [stellarWallets]]

  // 3. .flat() 扁平化
  // [evmWallet1, evmWallet2, ..., svmWallet1, ..., stellarWallet1, ...]

  return Array.from(wallets.values()).flat()
}, [wallets])
```

### 3.5 数据流向图

```
useWalletsList()
     │
     ▼
useWalletsRegistry()
     │
     ├──► useEvmWallets()      ─┐
     ├──► useSvmWallets()       ─┼─► Map<Namespace, WalletWithState[]>
     └──► useStellarWallets()  ─┘
              │
              ▼
         wallets: Map {
           'evm' => [WalletWithState, ...],
           'svm' => [WalletWithState, ...],
           'stellar' => [WalletWithState, ...]
         }
              │
              ▼
         Array.from(wallets.values()).flat()
              │
              ▼
         WalletWithState[] (扁平数组)
```

### 3.6 使用示例

```typescript
// ✅ 获取所有钱包列表
function WalletSelector() {
  const wallets = useWalletsList()

  return (
    <div>
      {wallets.map((wallet) => (
        <WalletOption key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}

// ✅ 按状态筛选
function InstalledWallets() {
  const wallets = useWalletsList()

  const installed = wallets.filter((w) => w.isInstalled)

  return (
    <div>
      {installed.map((wallet) => (
        <WalletOption key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}

// ✅ 按 namespace 筛选
function EvmOnlyWallets() {
  const wallets = useWalletsList()

  const evmWallets = wallets.filter((w) => w.namespace === 'evm')

  return (
    <div>
      {evmWallets.map((wallet) => (
        <WalletOption key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}

// ✅ 最近使用的钱包优先排序
function RecentFirstWalletList() {
  const wallets = useWalletsList()

  const sorted = useMemo(() => {
    return [...wallets].sort((a, b) => {
      // 最近使用的排前面
      if (a.isRecent && !b.isRecent) return -1
      if (!a.isRecent && b.isRecent) return 1
      // 已安装的排前面
      if (a.isInstalled && !b.isInstalled) return -1
      if (!a.isInstalled && b.isInstalled) return 1
      return 0
    })
  }, [wallets])

  return (
    <div>
      {sorted.map((wallet) => (
        <WalletOption key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}
```

---

## 四、AI 提示词编写教学

你正在做一个钱包选择列表组件，需要展示所有支持的钱包。这个 Hook 返回的是所有链的所有钱包列表，非常适合这个场景。

先确定一件事：这个 Hook 必须在 WalletsRegistryProvider 内使用。它依赖上层 Provider 提供的数据，如果在没有 Provider 的组件树里调用，可能会报错或者返回空数据。所以确保你的组件在最外层被 Provider 包裹着。

最常见的用法是对钱包进行分组展示。你可以用 reduce 方法把所有钱包按链类型分组，比如分成 EVM 组、Solana 组和 Stellar 组。然后在渲染时按组来显示，每组有个标题，标题下面是该组的钱包列表。

另一个常见需求是根据钱包的状态显示不同的内容。钱包可能有几种状态：不可用、可用但未安装、已安装。不可用的钱包应该显示为灰色和禁用状态；可用但未安装的钱包应该显示安装链接；已安装的钱包应该显示为可点击的连接按钮。如果这个钱包还是最近使用过的，可以额外显示一个"最近"标签。

新手常犯的一个错是直接修改返回的数组。这个 Hook 返回的数组是只读数据，不能直接用 push、splice 等方法来修改。如果需要修改（比如添加一个自定义钱包），应该先拷贝一份，然后操作拷贝。

另一个错是不处理空状态。虽然说 map 在空数组上运行不会报错，但如果你的 UI 设计上需要在没有钱包时显示特定的空状态提示（比如说"正在加载..."或者骨架屏），就应该显式检查数组长度。

还有个重要区别：这个 Hook 返回的是 WalletWithState 类型，不是 WalletConnection 类型。WalletWithState 描述的是钱包本身的状态，有 isInstalled、isAvailable、isRecent 这些属性，但没有 account 和 chainId 这些已连接信息。如果你想获取用户已连接的账户信息，应该用 `useWallet` 或者 `useAccounts` 那个 Hook。

钱包 ID 格式是带前缀的。EVM 钱包的 ID 是 `evm:io.metamask` 这样的，Solana 的是 `svm:phantom`，Stellar 的是 `stellar:freighter`。在比较钱包 ID 时要使用完整的带前缀的 ID。

这个 Hook 依赖 WalletsRegistryProvider。Provider 内部会调用三个链特定的钱包 Hook 来获取各自链的钱包列表，然后存在一个 Map 里。这个 Hook 所做的就是把这个 Map 转换成一个扁平的数组，每次底层数据更新时都会重新计算。

---

## 5. 该 hook 的用法教学

### 5.1 基础用法

```typescript
import { useWalletsList } from '@sushiswap/wallet'

// ✅ 最基本的使用
function WalletList() {
  const wallets = useWalletsList()

  return (
    <div className="wallet-list">
      {wallets.map((wallet) => (
        <WalletCard key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景1：分组钱包选择器

```typescript
function GroupedWalletSelector() {
  const wallets = useWalletsList()

  const groupedWallets = useMemo(() => {
    return wallets.reduce((groups, wallet) => {
      const namespace = wallet.namespace
      if (!groups[namespace]) {
        groups[namespace] = []
      }
      groups[namespace].push(wallet)
      return groups
    }, {} as Record<string, WalletWithState[]>)
  }, [wallets])

  return (
    <div className="wallet-selector">
      {Object.entries(groupedWallets).map(([namespace, list]) => (
        <div key={namespace} className="wallet-group">
          <h3 className="namespace-title">{namespace.toUpperCase()}</h3>
          <div className="wallet-grid">
            {list.map((wallet) => (
              <WalletCard key={wallet.id} wallet={wallet} />
            ))}
          </div>
        </div>
      ))}
    </div>
  )
}
```

#### 场景2：带状态的钱包卡片

```typescript
function WalletCard({ wallet }: { wallet: WalletWithState }) {
  const { addRecentWallet } = useRecentWallets()
  const { connect } = useWalletActions()

  const handleConnect = async () => {
    if (!wallet.isAvailable) return

    if (wallet.isInstalled) {
      await connect(wallet.id)
      addRecentWallet(wallet.id)
    } else if (wallet.url) {
      window.open(wallet.url, '_blank')
    }
  }

  return (
    <div className={`wallet-card ${!wallet.isAvailable ? 'disabled' : ''}`}>
      <img src={wallet.icon} alt={wallet.name} className="wallet-icon" />
      <div className="wallet-info">
        <span className="wallet-name">{wallet.name}</span>
        <div className="wallet-badges">
          {wallet.isRecent && <Badge type="recent">最近</Badge>}
          {wallet.isInstalled && <Badge type="installed">已安装</Badge>}
          {!wallet.isInstalled && wallet.isAvailable && (
            <Badge type="available">可安装</Badge>
          )}
          {!wallet.isAvailable && <Badge type="unavailable">不可用</Badge>}
        </div>
      </div>
      <button
        onClick={handleConnect}
        disabled={!wallet.isAvailable}
        className="connect-btn"
      >
        {wallet.isInstalled ? '连接' : wallet.url ? '安装' : '不可用'}
      </button>
    </div>
  )
}
```

#### 场景3：排序后的钱包列表（最近优先）

```typescript
function SortedWalletList() {
  const wallets = useWalletsList()

  const sortedWallets = useMemo(() => {
    return [...wallets].sort((a, b) => {
      // 1. 最近使用的排最前
      if (a.isRecent !== b.isRecent) {
        return a.isRecent ? -1 : 1
      }
      // 2. 已安装的排前
      if (a.isInstalled !== b.isInstalled) {
        return a.isInstalled ? -1 : 1
      }
      // 3. 按名称排序
      return a.name.localeCompare(b.name)
    })
  }, [wallets])

  return (
    <div className="wallet-list">
      {sortedWallets.map((wallet) => (
        <WalletCard key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}
```

#### 场景4：推荐钱包展示

```typescript
function RecommendedWallets() {
  const wallets = useWalletsList()

  const recommendedIds = [
    'evm:io.metamask',
    'evm:walletconnect',
    'svm:phantom',
    'stellar:freighter',
  ]

  const recommendedWallets = useMemo(() => {
    return wallets.filter((w) => recommendedIds.includes(w.id))
  }, [wallets])

  if (recommendedWallets.length === 0) {
    return null
  }

  return (
    <div className="recommended-wallets">
      <h3>推荐钱包</h3>
      <div className="recommended-grid">
        {recommendedWallets.map((wallet) => (
          <WalletCard key={wallet.id} wallet={wallet} />
        ))}
      </div>
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **使用 useMemo 缓存排序和筛选结果**
   ```typescript
   // ✅ 正确
   const sortedWallets = useMemo(() => {
     return [...wallets].sort((a, b) => {
       // 排序逻辑
     })
   }, [wallets])
   ```

2. **使用唯一id作为React key**
   ```typescript
   // ✅ 正确
   {wallets.map((wallet) => (
     <WalletCard key={wallet.id} wallet={wallet} />
   ))}
   ```

3. **处理空状态**
   ```typescript
   // ✅ 正确
   if (wallets.length === 0) {
     return <LoadingState />
   }
   ```

4. **按namespace分组时确保类型安全**
   ```typescript
   // ✅ 正确
   const grouped = wallets.reduce((acc, wallet) => {
     if (!acc[wallet.namespace]) {
       acc[wallet.namespace] = []
     }
     acc[wallet.namespace].push(wallet)
     return acc
   }, {} as Record<string, WalletWithState[]>)
   ```

#### ❌ Don'ts

1. **不要直接修改返回的数组**
   ```typescript
   // ❌ 错误
   wallets.push(newWallet)

   // ✅ 正确
   const modifiedWallets = [...wallets, newWallet]
   ```

2. **不要混淆WalletWithState和WalletConnection**
   ```typescript
   // ❌ 错误 - WalletWithState没有account属性
   const wallets = useWalletsList()
   wallets[0].account // 编译错误

   // ✅ 正确 - WalletWithState有isInstalled, isAvailable, isRecent
   // WalletConnection有account, chainId, name等
   ```

3. **不要忘记检查isAvailable**
   ```typescript
   // ❌ 错误 - 不检查可用性就尝试连接
   const handleConnect = async (wallet: WalletWithState) => {
     await connect(wallet.id) // 可能失败
   }

   // ✅ 正确 - 先检查
   if (wallet.isInstalled && wallet.isAvailable) {
     await connect(wallet.id)
   }
   ```

4. **不要使用index作为key**
   ```typescript
   // ❌ 错误
   {wallets.map((wallet, index) => (
     <WalletCard key={index} wallet={wallet} />
   ))}

   // ✅ 正确 - 使用唯一id
   {wallets.map((wallet) => (
     <WalletCard key={wallet.id} wallet={wallet} />
   ))}
   ```

5. **不要假设wallets数组不为空**
   ```typescript
   // ❌ 错误
   const firstWallet = wallets[0]
   firstWallet.isInstalled // 可能报错

   // ✅ 正确
   if (wallets.length > 0) {
     const firstWallet = wallets[0]
   }
   ```
