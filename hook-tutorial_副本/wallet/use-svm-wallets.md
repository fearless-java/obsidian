> 源代码路径: `apps/web/src/lib/wallet/namespaces/svm/provider/use-svm-wallets.ts`

# useSvmWallets Hook 教程

## 1. 大白话讲讲这个hook的作用

`useSvmWallets` *(一个React hook，用于获取所有Solana区块链钱包列表，基于@solana/connector的useWalletInfo实现)* 是一个用于**获取所有 Solana 区块链钱包列表**的 Hook。

简单来说：
- 它返回 `WalletWithState[]` *(包含钱包状态信息的接口，有isInstalled、isAvailable、isRecent等属性)*，包含所有支持的 Solana 钱包（如 Solflare、Phantom、MetaMask）
- 每个钱包对象告诉你是否已安装、是否可用、是否最近使用过
- 它通过 `useWalletInfo()` hook *(Solana SDK提供的hook，用于获取已连接的Solana钱包信息)* 获取**当前已连接**的 Solana 钱包
- 与 EVM 不同，Solana 钱包的检测依赖于 `sushi`'s Solana SDK

你可以把它想象成"问系统：所有支持的 Solana 钱包有哪些？用户当前连接了哪个？"。

---

## 2. 讲讲为什么需要封装该hook

### 2.1 Solana 钱包连接机制不同

与 EVM 的浏览器扩展检测不同，Solana 钱包通常通过以下方式连接：
- **Window.solana** 对象（Phantom, Solflare 等浏览器扩展）
- **Backpack** 等标准接口
- 通过 `@solana/connector` SDK *(Solana的钱包连接SDK，提供了统一的钱包接口)* 统一管理

### 2.2 使用 Solana SDK 的 useWalletInfo

SushiSwap 使用 `@solana/connector` 的 `useWalletInfo` hook 来获取已连接的钱包信息：

```typescript
import { useWalletInfo } from '@solana/connector'

export function useSvmWallets() {
  const { wallets } = useWalletInfo()
  // wallets 是已连接的 Solana 钱包列表
}
```

### 2.3 配置化钱包列表

`SVM_WALLETS` *(Solana钱包的配置文件)* 配置定义了所有支持但**未连接**的 Solana 钱包：
- Solflare *(Solana生态流行的钱包)*
- MetaMask (Solana) *(MetaMask的Solana版本)*
- Phantom *(Solana生态最流行的钱包之一)*

如果这些钱包没有通过 `useWalletInfo` 返回，说明它们没有连接，会被标记为 `isInstalled: false, isAvailable: false`。

---

## 3. 讲讲该hook的执行逻辑和数据流向

### 3.1 函数签名

```typescript
export function useSvmWallets() {
  const { wallets } = useWalletInfo()
  const { isRecentWallet } = useRecentWallets()

  return useMemo(() => {
    const map = new Map<string, WalletWithState>()

    // 1. 处理已连接的钱包
    for (const wallet of wallets) {
      const walletId = `svm:${wallet.name.toLowerCase()}`

      map.set(walletId, {
        id: walletId,
        namespace: 'svm',
        name: wallet.name,
        icon: wallet.icon ?? '',
        adapterId: SvmAdapterId.Standard,
        isInstalled: true,      // 已连接 = 已安装
        isAvailable: true,      // 已连接 = 可用
        isRecent: isRecentWallet(walletId),
      })
    }

    // 2. 添加配置中的其他钱包
    for (const wallet of SVM_WALLETS) {
      // 跳过已添加的
      if (map.has(wallet.id)) continue

      map.set(wallet.id, {
        ...wallet,
        isInstalled: false,
        isAvailable: false,
        isRecent: isRecentWallet(wallet.id),
      })
    }

    return Array.from(map.values())
  }, [wallets, isRecentWallet])
}
```

### 3.2 输入（Input）

无参数。

### 3.3 输出（Output）

```typescript
WalletWithState[]
```

| 属性 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | 如 `'svm:solflare'` |
| `namespace` | `'svm'` | 固定为 'svm' |
| `name` | `string` | 钱包名称 |
| `icon` | `string` | 钱包图标（base64 *(一种将二进制数据编码为可打印字符的编码方式)*） |
| `adapterId` | `SvmAdapterId.Standard` *(Solana钱包的标准适配器类型)* | 固定为 `'svm-standard'` |
| `isInstalled` | `boolean` | 是否已安装 |
| `isAvailable` | `boolean` | 是否可用 |
| `isRecent` | `boolean` | 是否最近使用过 |
| `url` | `string \| undefined` | 钱包官网 |

### 3.4 Solana 钱包适配器类型

```typescript
export enum SvmAdapterId {
  Standard = 'svm-standard'
}
```

### 3.5 SVM_WALLETS 配置

```typescript
export const SVM_WALLETS: Wallet[] = [
  {
    id: 'svm:solflare',
    namespace: 'svm',
    name: 'Solflare',
    adapterId: SvmAdapterId.Standard,
    url: 'https://solflare.com'
  },
  {
    id: 'svm:metamask',
    namespace: 'svm',
    name: 'MetaMask',
    adapterId: SvmAdapterId.Standard,
    url: 'https://metamask.io'
  },
  {
    id: 'svm:phantom',
    namespace: 'svm',
    name: 'Phantom',
    adapterId: SvmAdapterId.Standard,
    url: 'https://phantom.com'
  }
]
```

### 3.6 数据流向图

```
useSvmWallets()
     │
     ├──► useWalletInfo() ──► wallets[]
     │                        (已连接的 Solana 钱包)
     │
     ├──► useRecentWallets() ──► isRecentWallet(id)
     │
     └──► SVM_WALLETS (配置)
              │
              ▼
         [合并: 已连接 + 配置]
              │
              ▼
         WalletWithState[]
```

### 3.7 与 useEvmWallets 的对比

| 特性 | `useEvmWallets` *(EVM链钱包)* | `useSvmWallets` *(Solana链钱包)* |
|------|-----------------|-----------------|
| 数据源 | `useConnectors()` (wagmi) | `useWalletInfo()` (Solana SDK) |
| 已安装检测 | 浏览器扩展 + wagmi | 已连接 = 已安装 |
| Safe 支持 | 有 | 无 |
| WalletConnect | 有 | 无 |
| 适配器类型 | 多种 (Injected, WalletConnect, etc.) | 只有 Standard |

---

## 四、AI 提示词编写教学

你正在做一个 Solana 钱包选择界面，需要展示所有支持的 Solana 钱包。这个 Hook 返回所有 Solana 钱包的列表，每个钱包都带着自己的状态信息。

先确定一件事：这是专门给 Solana 链用的，只能获取 Solana 区块链的数据。如果你在 Ethereum 或 Stellar 链上用这个 Hook，得到的 namespace 始终是 'svm'。另外，要获取已连接的 Solana 钱包信息，应该用 `useWallet('svm')` 那个 Hook。

最基本的用法就是获取 Solana 钱包列表然后渲染出来。这个 Hook 返回的钱包列表里，已安装（已连接）的和未安装的是混在一起的，你可以在 UI 上用不同的样式区分开。比如给已连接的钱包加一个 'connected' 的类名，给未安装的加 'not-installed' 的类名。同时如果有"最近使用"标记的钱包，也可以显示一个标签。

另一个常见需求是只显示可以连接的钱包。可以通过过滤方法筛选出 `isAvailable` 为 true 的钱包，然后渲染成按钮列表供用户点击连接。

连接 Solana 钱包的时候，要用 Solana SDK 的 connect 方法，而不是以太坊的那套。

新手常犯的一个错是假设列表里第一个就是已连接的钱包。实际上如果没有连接任何 Solana 钱包，列表里第一个可能是配置里的默认钱包（Phantom、Solflare 等），它们都是未安装状态。正确做法是先过滤出 `isInstalled` 为 true 的钱包，然后再检查长度是否大于 0。

另一个常犯的错是混淆 EVM 和 SVM。虽然两者都有 `namespace` 属性，但 EVM 的 namespace 是 'evm'，SVM 的 namespace 是 'svm'。更重要的是，SVM 的地址格式和 EVM 完全不同，EVM 是 '0x' 开头的十六进制地址，SVM 是 Base58 编码的字符串，所以不能混用。

Solana 钱包的状态逻辑和 EVM 不一样。在 EVM 里，钱包可能有"已安装但未连接"的状态。但在 Solana 里，已连接就等于已安装且可用，未连接就等于未安装且不可用。不存在"已安装但未连接"这种中间状态。

钱包 ID 有固定的格式。Solana 钱包的 ID 都带 `svm:` 前缀，比如 `svm:solflare` 代表 Solflare，`svm:phantom` 代表 Phantom，`svm:metamask` 代表 MetaMask 的 Solana 版本。

这个 Hook 依赖 Solana SDK，需要正确配置 `@solana/connector`。如果 SDK 配置不对，获取到的数据可能不准确。

这个 Hook 会管理三块状态。第一块是 `wallets from useWalletInfo`，这是从外部 Solana SDK Hook 获取的已连接钱包列表。

第二块是 `isRecentWallet`，来自另一个 Hook。这个状态存在 LocalStorage 里，记录用户最近使用过的钱包。

第三块是最终的 Map，它把已连接的钱包和配置里的默认钱包合并起来，然后给每个钱包加上 `isRecent` 标记。最终返回一个包含所有钱包状态的列表。

---

## 5. 该 hook 的用法教学

### 5.1 基础用法

```typescript
import { useSvmWallets } from '@sushiswap/wallet'

// ✅ 最基本的使用
function SvmWalletList() {
  const wallets = useSvmWallets()

  return (
    <div className="wallet-list">
      {wallets.map((wallet) => (
        <SvmWalletCard key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景1：Solana 钱包选择器

```typescript
function SvmWalletSelector() {
  const wallets = useSvmWallets()
  const { addRecentWallet } = useRecentWallets()

  const handleConnect = async (wallet: WalletWithState) => {
    if (!wallet.isAvailable) return

    if (wallet.isInstalled) {
      await connectSolanaWallet(wallet.id)
      addRecentWallet(wallet.id)
    } else if (wallet.url) {
      window.open(wallet.url, '_blank')
    }
  }

  return (
    <div className="svm-wallet-selector">
      <h3>连接 Solana 钱包</h3>
      <div className="wallet-grid">
        {wallets.map((wallet) => (
          <div
            key={wallet.id}
            className={`wallet-card ${wallet.isInstalled ? 'installed' : ''}`}
          >
            <img src={wallet.icon} alt={wallet.name} className="wallet-icon" />
            <span className="wallet-name">{wallet.name}</span>
            {wallet.isInstalled && <Badge>已连接</Badge>}
            {wallet.isRecent && <Badge>最近</Badge>}
            <button onClick={() => handleConnect(wallet)}>
              {wallet.isInstalled ? '连接' : '安装'}
            </button>
          </div>
        ))}
      </div>
    </div>
  )
}
```

#### 场景2：已连接钱包优先展示

```typescript
function PrioritizedSvmWallets() {
  const wallets = useSvmWallets()

  const sortedWallets = useMemo(() => {
    return [...wallets].sort((a, b) => {
      // 已连接的排前面
      if (a.isInstalled !== b.isInstalled) {
        return a.isInstalled ? -1 : 1
      }
      // 最近使用的排前面
      if (a.isRecent !== b.isRecent) {
        return a.isRecent ? -1 : 1
      }
      return 0
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

#### 场景3：Solana 专用功能入口

```typescript
function SolanaFeatureGate() {
  const wallet = useWallet('svm')

  if (!wallet) {
    return (
      <div className="connect-prompt">
        <p>连接 Solana 钱包以使用 Solana 功能</p>
        <SvmWalletSelector />
      </div>
    )
  }

  return (
    <div className="solana-features">
      <div className="wallet-info">
        <img src={wallet.icon} alt={wallet.name} />
        <span>{wallet.name}</span>
        <span>{truncateAddress(wallet.account)}</span>
      </div>
      <div className="features">
        <SolanaBalance address={wallet.account} />
        <SolanaSwap address={wallet.account} />
        <SolanaStake address={wallet.account} />
      </div>
    </div>
  )
}
```

#### 场景4：空状态处理

```typescript
function SvmWalletListWithEmpty() {
  const wallets = useSvmWallets()

  // 检查是否有已安装/可用的钱包
  const hasAvailableWallets = wallets.some((w) => w.isAvailable)

  if (!hasAvailableWallets) {
    return (
      <div className="empty-state">
        <p>暂无可用的 Solana 钱包</p>
        <p>请安装以下钱包之一:</p>
        <ul>
          <li><a href="https://phantom.com" target="_blank">Phantom</a></li>
          <li><a href="https://solflare.com" target="_blank">Solflare</a></li>
          <li><a href="https://metamask.io" target="_blank">MetaMask</a></li>
        </ul>
      </div>
    )
  }

  return (
    <div className="wallet-list">
      {wallets.map((wallet) => (
        <WalletCard key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **理解 SVM 的安装状态含义**
   ```typescript
   // ✅ 正确 - SVM中已连接 = 已安装
   if (wallet.isInstalled) {
     // 用户已连接该钱包
   }
   ```

2. **使用 useMemo 优化排序**
   ```typescript
   // ✅ 正确
   const sortedWallets = useMemo(() => {
     return [...wallets].sort((a, b) => {
       // 排序逻辑
     })
   }, [wallets])
   ```

3. **处理空状态**
   ```typescript
   // ✅ 正确
   if (wallets.length === 0) {
     return <LoadingState />
   }
   ```

4. **使用唯一id作为key**
   ```typescript
   // ✅ 正确
   {wallets.map((wallet) => (
     <WalletCard key={wallet.id} wallet={wallet} />
   ))}
   ```

#### ❌ Don'ts

1. **不要混淆 SVM 和 EVM 的状态逻辑**
   ```typescript
   // ❌ 错误 - SVM中不存在"已安装但未连接"的状态
   if (!wallet.isInstalled && wallet.isAvailable) {
     // 这个逻辑在SVM中不成立
   }

   // ✅ 正确 - SVM中 isInstalled === isAvailable（对于已连接的钱包）
   ```

2. **不要假设有已连接的钱包**
   ```typescript
   // ❌ 错误
   const firstWallet = wallets[0]
   firstWallet.isInstalled // 可能不存在

   // ✅ 正确 - 先过滤
   const installed = wallets.filter((w) => w.isInstalled)
   if (installed.length > 0) {
     // 有已连接的钱包
   }
   ```

3. **不要直接修改钱包列表**
   ```typescript
   // ❌ 错误
   wallets.push(newWallet)

   // ✅ 正确
   const modifiedWallets = [...wallets, newWallet]
   ```

4. **不要混淆 WalletWithState 和 WalletConnection**
   ```typescript
   // ❌ 错误 - WalletWithState没有account属性
   const wallets = useSvmWallets()
   wallets[0].account // 编译错误

   // ✅ 正确 - 使用useWallet获取已连接信息
   const wallet = useWallet('svm')
   if (wallet) {
     wallet.account // 正确
   }
   ```

5. **不要使用不带namespace前缀的ID**
   ```typescript
   // ❌ 错误
   addRecentWallet('phantom')

   // ✅ 正确
   addRecentWallet('svm:phantom')
   ```
