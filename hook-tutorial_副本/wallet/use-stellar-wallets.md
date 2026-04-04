> 源代码路径: `apps/web/src/lib/wallet/namespaces/stellar/provider/use-stellar-wallets.ts`

# useStellarWallets Hook 教程

## 1. 大白话讲讲这个hook的作用

`useStellarWallets` *(一个React hook，用于获取所有Stellar区块链钱包列表，基于stellar-wallets-kit SDK异步获取)* 是一个用于**获取所有 Stellar 区块链钱包列表**的 Hook。

简单来说：
- 它返回 `WalletWithState[]` *(包含钱包状态信息的接口，有isInstalled、isAvailable、isRecent等属性)*，包含所有支持的 Stellar 钱包（如 Freighter 等）
- 每个钱包对象告诉你是否已安装、是否可用、是否最近使用过
- 它通过 `stellar-wallets-kit` SDK *(Stellar区块链的钱包SDK，提供了获取支持钱包列表的接口)* 动态获取支持的钱包列表
- 数据是**异步获取**的，初始值为空数组

你可以把它想象成"问系统：所有支持的 Stellar 钱包有哪些？用户装了几个？最近用了哪个？"。

---

## 2. 讲讲为什么需要封装该hook

### 2.1 使用 Stellar SDK 获取钱包列表

与其他链不同，Stellar 钱包列表是通过 `StellarWalletsKit` *(Stellar的钱包工具包，用于获取支持的钱包列表)* 动态获取的：

```typescript
import { stellarWalletKit } from '../config'

const fetchWallets = async () => {
  const supportedWallets = await stellarWalletKit.getSupportedWallets()
  setWallets(supportedWallets)
}
```

### 2.2 异步加载钱包列表

Stellar 钱包列表不是同步可用的，需要异步请求：

```typescript
const [wallets, setWallets] = useState<ISupportedWallet[]>([])

useEffect(() => {
  const fetchWallets = async () => {
    const supportedWallets = await stellarWalletKit.getSupportedWallets()
    setWallets(supportedWallets)
  }
  fetchWallets()
}, [])
```

这意味着初始渲染时 `wallets` 是空数组。

### 2.3 分离的内部 Hook

代码中使用了**两个** hook：

1. `_useStellarWallets`（内部使用）：管理异步获取的钱包列表
2. `useStellarWallets`（导出）：添加 `isRecentWallet` *(来自useRecentWallets的函数，用于判断钱包是否在最近列表中)* 标记

```typescript
// 内部 hook - 管理异步数据
const _useStellarWallets = () => {
  const [wallets, setWallets] = useState<ISupportedWallet[]>([])
  useEffect(() => { ... }, [])
  return { wallets }
}

// 导出的 hook - 添加最近使用标记
export function useStellarWallets() {
  const { isRecentWallet } = useRecentWallets()
  const { wallets } = _useStellarWallets()
  // ...添加 isRecent 标记
}
```

---

## 3. 讲讲该hook的执行逻辑和数据流向

### 3.1 函数签名

```typescript
const _useStellarWallets = () => {
  const [wallets, setWallets] = useState<ISupportedWallet[]>([])
  useEffect(() => { ... }, [])
  return { wallets }
}

export function useStellarWallets() {
  const { isRecentWallet } = useRecentWallets()
  const { wallets } = _useStellarWallets()

  return useMemo(() => {
    const map = new Map<string, WalletWithState>()

    for (const wallet of wallets) {
      const walletId = `stellar:${wallet.id.toLowerCase()}`

      map.set(walletId, {
        id: walletId,
        namespace: 'stellar',
        name: wallet.name,
        icon: wallet.icon ?? '',
        adapterId: StellarAdapterId.Standard,
        isInstalled: wallet.isAvailable,  // ⚠️ 注意：这里直接用 isAvailable
        isAvailable: wallet.isAvailable,
        isRecent: isRecentWallet(walletId),
        url: wallet.url,
      })
    }

    return Array.from(map.values())
  }, [isRecentWallet, wallets])
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
| `id` | `string` | 如 `'stellar:freighter'` |
| `namespace` | `'stellar'` | 固定为 'stellar' |
| `name` | `string` | 钱包名称 |
| `icon` | `string` | 钱包图标 |
| `adapterId` | `StellarAdapterId.Standard` *(Stellar钱包的标准适配器类型)* | 固定为 `'stellar-standard'` |
| `isInstalled` | `boolean` | 是否已安装 |
| `isAvailable` | `boolean` | 是否可用 |
| `isRecent` | `boolean` | 是否最近使用过 |
| `url` | `string \| undefined` | 钱包官网/下载链接 |

### 3.4 Stellar 适配器类型

```typescript
export enum StellarAdapterId {
  Standard = 'stellar-standard'
}
```

### 3.5 数据流向图

```
useStellarWallets()
     │
     ├──► _useStellarWallets()
     │         │
     │         └──► stellarWalletKit.getSupportedWallets()
     │                    (异步获取)
     │                    │
     │                    ▼
     │              wallets: ISupportedWallet[]
     │
     ├──► useRecentWallets() ──► isRecentWallet(id)
     │
     ▼
[Map 合并 + isRecent 标记]
     │
     ▼
WalletWithState[]
```

### 3.6 与其他链 Hook 的对比

| 特性 | `useEvmWallets` *(EVM链钱包)* | `useSvmWallets` *(Solana链钱包)* | `useStellarWallets` *(Stellar链钱包)* |
|------|-----------------|-----------------|---------------------|
| 数据源 | `useConnectors()` (wagmi) | `useWalletInfo()` | `stellarWalletKit` |
| 获取方式 | 同步 | 同步 | **异步** |
| 初始值 | 有数据 | 有数据 | **空数组** |
| Safe 支持 | 有 | 无 | 无 |
| 适配器类型 | 多种 | Standard | Standard |

### 3.7 使用示例

```typescript
// ✅ 基本用法（注意异步加载）
function StellarWalletList() {
  const wallets = useStellarWallets()

  // wallets 初始可能是空数组
  if (wallets.length === 0) {
    return <LoadingSkeleton count={3} />
  }

  return (
    <div>
      {wallets.map((wallet) => (
        <StellarWalletOption key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}

// ✅ 异步加载完成后再渲染
function StellarWalletSelector() {
  const wallets = useStellarWallets()

  return (
    <div>
      {wallets.length > 0 ? (
        wallets.map((wallet) => (
          <WalletOption key={wallet.id} wallet={wallet} />
        ))
      ) : (
        <LoadingState />
      )}
    </div>
  )
}

// ✅ 连接钱包
async function connectStellar(wallet: WalletWithState) {
  await stellarWalletKit.openWallet({
    walletId: wallet.id,
    session: await stellarWalletKit.getNewSession(),
    // ... 其他参数
  })
}
```

---

## 四、AI 提示词编写教学

你正在做一个 Stellar 钱包选择界面，需要展示所有支持的 Stellar 钱包。这个 Hook 返回所有 Stellar 钱包的列表，但有一个非常重要的特点：它是异步加载的。

先确定一件事：页面刚加载时钱包列表是空的，要等 SDK 返回数据后才会显示。所以你的 UI 必须处理这个加载状态。最简单的做法是检查数组长度，如果为 0 就显示骨架屏或者加载指示器，等有数据了再渲染钱包列表。

另一个常见需求是只显示可用的钱包。可以通过过滤方法筛选出 `isAvailable` 为 true 的钱包。如果过滤后数组为空，要显示一个友好的提示，告诉用户暂无可用的 Stellar 钱包。

还有一种情况是根据安装状态显示不同内容。如果钱包已安装，就显示连接按钮；如果未安装但有下载链接，就显示下载链接。给已安装的钱包可以加一个绿色标签，已连接的钱包也可以额外显示一个"最近"标签。

新手常犯的一个错是不处理异步加载状态。有些人可能觉得反正钱包列表最后会有的，就直接渲染 `wallets.map(...)`。但实际上在页面刚加载的时候 wallets 是空数组，map 不会报错，但会渲染出空白内容，用户体验很差。正确做法是先检查数组长度，如果为 0 就显示加载状态。

另一个容易混淆的地方是 `isInstalled` 和 `isAvailable` 的概念。在 Stellar 中，这两个状态是相等的，来自 SDK 的实现里 `isInstalled` 直接使用了 `isAvailable` 的值。这和 EVM 不一样，在 EVM 里可能有"已安装但未连接"的状态，但在 Stellar 里"已安装"就等于"可用"，不存在中间状态。

异步加载是必须的。在页面刚加载时，钱包列表是空数组 `[]`，不是 null 也不是 undefined。所以任何访问列表元素的地方都要先检查数组长度。你不能假设 `wallets[0]` 一定存在。

`isInstalled` 和 `isAvailable` 在 Stellar 里是等价的。SDK 的实现里，这两个状态是同一个值。如果你写代码时假设存在"已安装但不可用"或者"可用但未安装"的状态，在 Stellar 里是不会成立的。

钱包 ID 有固定的格式。Stellar 钱包的 ID 都带 `stellar:` 前缀，而且全部小写，比如 `stellar:freighter` 代表 Freighter 钱包，`stellar:albedo` 代表 Albedo 钱包。

这个 Hook 依赖 `stellar-wallets-kit` SDK，需要正确配置。异步获取过程中如果出错，代码里目前没有体现错误处理，你需要自己添加。

这个 Hook 会管理三块状态。第一块是 `_useStellarWallets` 内部的状态，通过 `useState` 存储 SDK 返回的钱包列表，并通过 `useEffect` 异步获取数据。

第二块是 `isRecentWallet`，来自另一个 Hook。这个状态存在 LocalStorage 里，记录用户最近使用过的钱包。

第三块是最终的 Map，它把 SDK 返回的钱包和 `isRecent` 标记合并起来，最终返回一个包含所有钱包状态的列表。这个 Map 的依赖包括异步获取的钱包列表和 `isRecentWallet` 函数。

---

## 5. 该 hook 的用法教学

### 5.1 基础用法

```typescript
import { useStellarWallets } from '@sushiswap/wallet'

// ✅ 最基本的使用（注意异步加载）
function StellarWalletList() {
  const wallets = useStellarWallets()

  // 初始渲染时可能是空数组
  if (wallets.length === 0) {
    return <LoadingSkeleton count={3} />
  }

  return (
    <div className="wallet-list">
      {wallets.map((wallet) => (
        <StellarWalletCard key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景1：带加载状态的钱包选择器

```typescript
function StellarWalletSelector() {
  const wallets = useStellarWallets()
  const { addRecentWallet } = useRecentWallets()

  const handleConnect = async (wallet: WalletWithState) => {
    if (!wallet.isAvailable) return

    if (wallet.isInstalled) {
      await connectStellarWallet(wallet.id)
      addRecentWallet(wallet.id)
    } else if (wallet.url) {
      window.open(wallet.url, '_blank')
    }
  }

  return (
    <div className="stellar-wallet-selector">
      <h3>连接 Stellar 钱包</h3>

      {wallets.length === 0 ? (
        <div className="loading-state">
          <Skeleton width={200} height={40} />
          <Skeleton width={200} height={40} />
          <Skeleton width={200} height={40} />
        </div>
      ) : (
        <div className="wallet-grid">
          {wallets.map((wallet) => (
            <div
              key={wallet.id}
              className={`wallet-card ${wallet.isInstalled ? 'installed' : ''}`}
            >
              <img
                src={wallet.icon}
                alt={wallet.name}
                className="wallet-icon"
              />
              <span className="wallet-name">{wallet.name}</span>
              {wallet.isInstalled && <Badge>已连接</Badge>}
              {wallet.isRecent && <Badge>最近</Badge>}
              <button onClick={() => handleConnect(wallet)}>
                {wallet.isInstalled ? '连接' : '安装'}
              </button>
            </div>
          ))}
        </div>
      )}
    </div>
  )
}
```

#### 场景2：可用的钱包列表

```typescript
function AvailableStellarWallets() {
  const wallets = useStellarWallets()

  const availableWallets = useMemo(() => {
    return wallets.filter((w) => w.isAvailable)
  }, [wallets])

  if (wallets.length === 0) {
    return <LoadingState />
  }

  if (availableWallets.length === 0) {
    return (
      <div className="empty-state">
        <p>暂无可用的 Stellar 钱包</p>
        <p>请安装 Freighter 钱包</p>
        <a href="https://www.freighter.app" target="_blank">
          下载 Freighter
        </a>
      </div>
    )
  }

  return (
    <div className="wallet-list">
      {availableWallets.map((wallet) => (
        <WalletCard key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}
```

#### 场景3：Stellar 专用功能入口

```typescript
function StellarFeatureGate() {
  const wallet = useWallet('stellar')

  if (!wallet) {
    return (
      <div className="connect-prompt">
        <p>连接 Stellar 钱包以使用 Stellar 功能</p>
        <StellarWalletSelector />
      </div>
    )
  }

  return (
    <div className="stellar-features">
      <div className="wallet-info">
        <img src={wallet.icon} alt={wallet.name} />
        <span>{wallet.name}</span>
        <span className="address">{truncateAddress(wallet.account)}</span>
      </div>
      <div className="features">
        <StellarBalance address={wallet.account} />
        <StellarSwap address={wallet.account} />
        <StellarSend address={wallet.account} />
      </div>
    </div>
  )
}
```

#### 场景4：错误处理和重试

```typescript
function StellarWalletListWithRetry() {
  const [error, setError] = useState<string | null>(null)
  const [isLoading, setIsLoading] = useState(true)

  const wallets = useStellarWallets()

  useEffect(() => {
    if (wallets.length === 0 && !isLoading && !error) {
      // 如果持续没有数据，可能出错了
      setError('加载钱包列表失败')
    }
  }, [wallets.length, isLoading, error])

  if (wallets.length === 0) {
    if (error) {
      return (
        <div className="error-state">
          <p>{error}</p>
          <button onClick={() => window.location.reload()}>
            重试
          </button>
        </div>
      )
    }
    return <LoadingSkeleton count={3} />
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

1. **始终处理初始空数组状态**
   ```typescript
   // ✅ 正确 - 处理加载状态
   if (wallets.length === 0) {
     return <LoadingSkeleton count={3} />
   }
   ```

2. **使用 useMemo 缓存筛选结果**
   ```typescript
   // ✅ 正确
   const availableWallets = useMemo(() => {
     return wallets.filter((w) => w.isAvailable)
   }, [wallets])
   ```

3. **理解 Stellar 中 isInstalled = isAvailable**
   ```typescript
   // ✅ 正确 - Stellar中这两个状态等价
   if (wallet.isInstalled) {
     // 等同于 wallet.isAvailable
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

1. **不要不处理异步加载**
   ```typescript
   // ❌ 错误 - 初始渲染时wallets是空数组
   const wallets = useStellarWallets()
   return wallets.map((wallet) => <WalletCard wallet={wallet} />)

   // ✅ 正确 - 始终处理空状态
   if (wallets.length === 0) return <Loading />
   return wallets.map((wallet) => <WalletCard wallet={wallet} />)
   ```

2. **不要混淆 Stellar 和其他链的状态逻辑**
   ```typescript
   // ❌ 错误 - Stellar中isInstalled直接使用isAvailable
   if (wallet.isInstalled && !wallet.isAvailable) {
     // 这个逻辑在Stellar中不可能成立
   }
   ```

3. **不要直接修改钱包列表**
   ```typescript
   // ❌ 错误
   wallets.push(newWallet)

   // ✅ 正确
   const modifiedWallets = [...wallets, newWallet]
   ```

4. **不要假设wallets立即有数据**
   ```typescript
   // ❌ 错误
   const firstWallet = wallets[0]
   firstWallet.isInstalled // 可能是undefined

   // ✅ 正确
   if (wallets.length > 0 && wallets[0]) {
     const firstWallet = wallets[0]
   }
   ```

5. **不要混淆 WalletWithState 和 WalletConnection**
   ```typescript
   // ❌ 错误 - WalletWithState没有account属性
   const wallets = useStellarWallets()
   wallets[0].account // 编译错误

   // ✅ 正确 - 使用useWallet获取已连接信息
   const wallet = useWallet('stellar')
   if (wallet) {
     wallet.account // 正确
   }
   ```
