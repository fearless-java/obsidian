> 源代码路径: `apps/web/src/lib/wallet/hooks/use-account.ts`

# useEvmWallets Hook 教程

## 1. 大白话讲讲这个hook的作用

`useEvmWallets` *(一个React hook，用于获取所有EVM兼容钱包列表，基于wagmi的useConnectors实现)* 是一个用于**获取所有 EVM 兼容钱包列表**的 Hook。

简单来说：
- 它返回 `WalletWithState[]` *(一个接口类型，包含钱包的id、name、icon、安装状态等信息)*，包含所有支持的 EVM 钱包
- 每 个钱包对象告诉你：
  - 是否已安装到浏览器（`isInstalled` *(布尔值，表示该钱包扩展是否已安装在用户浏览器中)*）
  - 当前是否可用（`isAvailable` *(布尔值，表示当前环境是否可以使用该钱包进行连接)*）
  - 最近是否使用过（`isRecent` *(布尔值，标记该钱包是否在用户的最近使用列表中)*）
- 内置检测 Safe App 环境（如果是 Safe 网页，会标记 Safe 钱包为可用）

你可以把它想象成"问系统：所有支持的 EVM 钱包有哪些？用户装了几个？最近用了哪个？"

---

## 2. 讲讲为什么需要封装该hook

### 2.1 EVM 钱包生态复杂

EVM 钱包有很多类型：
- **浏览器扩展钱包**：MetaMask *(最流行的EVM钱包浏览器扩展，提供以太坊账户管理)*、Rabby *(一个注重安全的EVM浏览器扩展钱包)* 等
- **WalletConnect 协议钱包**：通过二维码连接 *(一种去中心化的钱包连接协议，支持多种钱包)*

- **交易所钱包**：Coinbase Wallet *(Coinbase交易所推出的钱包，集成在Coinbase平台)*

- **Safe App**：在 Safe 多签钱包环境中运行 *(Safe是一个多签钱包解决方案，提供安全的多签功能)*

- **Porto**：特殊的钱包 *(一个特定的钱包适配器类型)*

每种类型有不同的检测方式和连接逻辑。

### 2.2 动态检测已安装的扩展

通过 `useConnectors()`（来自 wagmi *(一个流行的React Hooks库，用于管理以太坊钱包连接)*）获取浏览器已安装的 EVM 钱包：

```typescript
const connectors = useConnectors()
// connectors 包含了所有已安装的浏览器扩展钱包
// 通过 filter 筛选 injected 类型
```

### 2.3 Safe App 环境检测

Safe 钱包比较特殊，它不是浏览器扩展，而是通过 Safe App 环境运行：

```typescript
// Safe App 检测
const [isSafeAvailable, setIsSafeAvailable] = useState(false)

useEffect(() => {
  let cancelled = false
  ;(async () => {
    const ok = await isSafeAppAvailable()
    if (!cancelled) setIsSafeAvailable(ok)
  })()
  return () => { cancelled = true }
}, [])
```

### 2.4 配置化的钱包列表

`EVM_WALLETS` *(一个配置文件，定义了所有支持的EVM钱包的元数据，包括id、name、icon、adapterId等)* 配置文件定义了所有支持的钱包列表，包括：
- 钱包 ID、名称、图标
- 适配器类型（决定如何连接）
- 钱包官网 URL

---

## 3. 讲讲该hook的执行逻辑和数据流向

### 3.1 函数签名

```typescript
export function useEvmWallets() {
  const connectors = useConnectors()
  const { isRecentWallet } = useRecentWallets()

  const injectedConnectors = useMemo(
    () => connectors.filter(isInjectedConnector),
    [connectors],
  )

  const [isSafeAvailable, setIsSafeAvailable] = useState(false)

  // Safe app 检测...
  useEffect(() => { ... }, [])

  return useMemo(() => {
    // 构建钱包列表...
  }, [injectedConnectors, isSafeAvailable, isRecentWallet])
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
| `id` | `string` | 如 `'evm:io.metamask'` |
| `namespace` | `'evm'` | 固定为 'evm' |
| `name` | `string` | 钱包名称 |
| `icon` | `string` | 钱包图标（base64 *(一种编码格式，用于将二进制数据转为文本字符串，常用于图片base64编码)*） |
| `adapterId` | `EvmAdapterId` *(枚举类型，定义EVM钱包的适配器类型，如Injected、WalletConnect等)* | 适配器类型 |
| `isInstalled` | `boolean` | 是否已安装 |
| `isAvailable` | `boolean` | 是否可用 |
| `isRecent` | `boolean` | 是否最近使用过 |
| `uid` | `string \| undefined` | 连接器 UID（仅 injected *(指浏览器扩展类型钱包，通过window.ethereum接口连接)*） |

### 3.4 内部执行逻辑

```typescript
return useMemo(() => {
  const map = new Map<string, WalletWithState>()

  // 1. 处理已安装的 injected 钱包（浏览器扩展）
  for (const connector of injectedConnectors) {
    const walletId = `evm:${connector.id.toLowerCase()}`

    map.set(walletId, {
      id: walletId,
      namespace: 'evm',
      name: connector.name,
      icon: connector.icon ?? '',
      adapterId: EvmAdapterId.Injected,
      isInstalled: true,
      isAvailable: true,
      isRecent: isRecentWallet(walletId),
      uid: connector.uid,
    })
  }

  // 2. 处理配置中的其他钱包
  for (const wallet of EVM_WALLETS) {
    // 跳过已有 uid 的（已在上面处理）
    if (map.get(wallet.id)?.uid) continue

    // WalletConnect, Porto, CoinbaseWallet, MetaMask 总是 available
    if ([WalletConnect, Porto, CoinbaseWallet, MetaMask].includes(wallet.adapterId)) {
      map.set(wallet.id, {
        ...wallet,
        isInstalled: false,
        isAvailable: true,
        isRecent: isRecentWallet(wallet.id),
      })
      continue
    }

    // Safe: 仅在 Safe 环境中 available
    if (wallet.adapterId === EvmAdapterId.Safe) {
      if (isSafeAvailable) {
        map.set(wallet.id, {
          ...wallet,
          isInstalled: true,
          isAvailable: true,
          isRecent: isRecentWallet(wallet.id),
        })
      }
      continue
    }

    // 其他钱包：不可用
    map.set(wallet.id, {
      ...wallet,
      isInstalled: false,
      isAvailable: false,
      isRecent: isRecentWallet(wallet.id),
    })
  }

  return Array.from(map.values())
}, [injectedConnectors, isSafeAvailable, isRecentWallet])
```

### 3.5 EVM 钱包适配器类型

```typescript
export enum EvmAdapterId {
  Injected = 'evm-injected',         // 浏览器扩展
  MetaMask = 'evm-metamask',         // MetaMask 专用适配器
  Porto = 'evm-porto',               // Porto 钱包
  Safe = 'evm-safe',                  // Safe 多签
  WalletConnect = 'evm-walletconnect', // WalletConnect
  CoinbaseWallet = 'evm-coinbasewallet' // Coinbase Wallet
}
```

### 3.6 数据流向图

```
useEvmWallets()
     │
     ├──► useConnectors() ──► injectedConnectors[]
     │                          (浏览器已安装的扩展)
     │
     ├──► useRecentWallets() ──► isRecentWallet(id)
     │
     ├──► useState(false) ──► isSafeAvailable
     │      (异步检测 Safe App 环境)
     │
     └──► EVM_WALLETS (配置)
              │
              ▼
         [合并 + 状态计算]
              │
              ▼
         WalletWithState[]
```

### 3.7 使用示例

```typescript
// ✅ 获取 EVM 钱包列表
function EvmWalletList() {
  const wallets = useEvmWallets()

  return (
    <div>
      {wallets.map((wallet) => (
        <EvmWalletOption key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}

// ✅ 按状态分组展示
function GroupedEvmWallets() {
  const wallets = useEvmWallets()

  const connected = wallets.filter((w) => w.isInstalled)
  const available = wallets.filter((w) => !w.isInstalled && w.isAvailable)
  const unavailable = wallets.filter((w) => !w.isAvailable)

  return (
    <div>
      {connected.length > 0 && (
        <Section title="已连接">
          {connected.map((w) => <WalletButton key={w.id} wallet={w} />)}
        </Section>
      )}
      {available.length > 0 && (
        <Section title="可连接">
          {available.map((w) => <WalletButton key={w.id} wallet={w} />)}
        </Section>
      )}
    </div>
  )
}

// ✅ 最近使用的优先
function RecentEvmWallets() {
  const wallets = useEvmWallets()

  const sorted = useMemo(() => {
    return [...wallets].sort((a, b) => {
      if (a.isRecent !== b.isRecent) return a.isRecent ? -1 : 1
      return 0
    })
  }, [wallets])

  return sorted.map((wallet) => <WalletOption key={wallet.id} wallet={wallet} />)
}
```

---

## 四、AI 提示词编写教学

你正在做一个 EVM 钱包选择界面，需要展示所有支持的以太坊系钱包。这个 Hook 返回所有 EVM 钱包的列表，每个钱包都带着自己的状态信息。

先确定一件事：这是专门给 EVM 链用的，只能获取以太坊系区块链的数据。如果你在 Solana 或 Stellar 链上用这个 Hook，得到的 namespace 始终是 'evm'。另外，这个 Hook 依赖 wagmi 库的 `useConnectors`，所以必须确保你的项目在 wagmi 的 WagmiConfigProvider 包裹下才能正常工作。

钱包列表组件要根据不同的状态显示不同的内容。如果钱包完全不可用，就显示一个灰色的禁用状态，告诉用户这个钱包暂时用不了。如果钱包可用但还没安装，那就是一个下载链接，用户可以点击去官网下载。如果钱包已经安装好了，那就是一个可以点击的连接按钮，同时如果这个钱包是用户最近用过的，还要显示一个"最近"的标签来提示用户。

连接钱包的时候，要记得检查钱包状态。不是随便拿个钱包就能连接的，要确保它既已安装又当前可用才能尝试连接。用 if 语句检查 `isInstalled && isAvailable`，两个都为 true 才进行连接。

还有个特殊情况是 Safe 钱包。Safe 是一个多签钱包解决方案，如果你想让某些功能只在 Safe 环境里使用，就得先检测用户是不是在 Safe 环境里。可以通过查找适配器 ID 为 'evm-safe' 的钱包，然后检查它是否已安装来判断。

新手经常犯的一个错是不检查钱包状态就直接尝试连接。比如直接拿列表里的第一个钱包来连接，但这个钱包可能根本没安装，这样肯定会失败。

另一个容易混淆的地方是 `isInstalled` 和 `isAvailable` 的区别。`isInstalled` 表示钱包是否已经安装到浏览器里，而 `isAvailable` 表示当前是否可以用来连接。比如 WalletConnect 钱包，它其实不需要在浏览器里安装什么扩展，因为它是通过扫描二维码来连接的，所以 `isInstalled` 是 false 但 `isAvailable` 是 true。这种情况下用户需要跳转到下载页面获取连接二维码，而不是直接连接。

Safe 钱包的检测是异步的。在页面刚加载的时候，程序还在检测用户是否在 Safe 环境里，所以 `isSafeAvailable` 一开始是 false。这意味着 Safe 钱包可能一开始显示为不可用，等检测完成后才会突然出现。这是正常现象，不用担心。

钱包 ID 有固定的格式。EVM 钱包的 ID 都带 `evm:` 前缀，比如 `evm:io.metamask` 代表 MetaMask，`evm:io.rabby` 代表 Rabby，`evm:walletconnect` 代表 WalletConnect。给 AI 描述需求的时候，要用带前缀的完整 ID。

这个 Hook 会管理三块状态。第一块是 `injectedConnectors`，这是一个计算结果，依赖于 wagmi 的 `useConnectors` 返回的连接器列表。每次连接器列表变化时，这个值就会重新计算。

第二块是 `isSafeAvailable`，这是通过异步检测 Safe 环境得到的状态。程序会通过 `isSafeAppAvailable` 这个函数去检测当前是否在 Safe App 环境里。

第三块是最终的 Map，它把注入的连接器和配置里的钱包合并起来，最终返回一个包含所有钱包状态的列表。这个 Map 的依赖是前面两块状态加上 `isRecentWallet` 函数。

---

## 5. 该 hook 的用法教学

### 5.1 基础用法

```typescript
import { useEvmWallets } from '@sushiswap/wallet'

// ✅ 最基本的使用
function WalletList() {
  const wallets = useEvmWallets()

  return (
    <div>
      {wallets.map((wallet) => (
        <div key={wallet.id}>
          <img src={wallet.icon} alt={wallet.name} />
          <span>{wallet.name}</span>
          <span>{wallet.isInstalled ? '已安装' : '未安装'}</span>
        </div>
      ))}
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景1：按状态分组的钱包选择器

```typescript
function GroupedWalletSelector() {
  const wallets = useEvmWallets()

  // 按状态分组
  const { installed, available, unavailable } = useMemo(() => {
    return {
      installed: wallets.filter((w) => w.isInstalled && w.isAvailable),
      available: wallets.filter((w) => !w.isInstalled && w.isAvailable),
      unavailable: wallets.filter((w) => !w.isAvailable),
    }
  }, [wallets])

  return (
    <div className="wallet-selector">
      {/* 最近使用 */}
      {installed.some((w) => w.isRecent) && (
        <Section title="最近使用">
          {installed.filter((w) => w.isRecent).map((w) => (
            <WalletButton key={w.id} wallet={w} />
          ))}
        </Section>
      )}

      {/* 已安装且可用 */}
      <Section title="已连接">
        {installed.filter((w) => !w.isRecent).map((w) => (
          <WalletButton key={w.id} wallet={w} />
        ))}
      </Section>

      {/* 可用但未安装（如 WalletConnect） */}
      <Section title="其他钱包">
        {available.map((w) => (
          <WalletButton key={w.id} wallet={w} />
        ))}
      </Section>
    </div>
  )
}
```

#### 场景2：带加载状态的钱包列表

```typescript
function WalletListWithLoading() {
  const wallets = useEvmWallets()

  // 初始加载状态
  if (wallets.length === 0) {
    return (
      <div className="wallet-loading">
        <Skeleton count={3} height={60} />
      </div>
    )
  }

  return (
    <div className="wallet-list">
      {wallets.map((wallet) => (
        <WalletOption key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}
```

#### 场景3：检测特定钱包是否可用

```typescript
function MetaMaskOnlyFeature() {
  const wallets = useEvmWallets()
  const metaMask = wallets.find((w) => w.id === 'evm:io.metamask')

  if (!metaMask) {
    return <span>MetaMask 不可用</span>
  }

  return (
    <button
      disabled={!metaMask.isInstalled}
      onClick={() => connectWallet(metaMask)}
    >
      {metaMask.isInstalled ? '连接 MetaMask' : '安装 MetaMask'}
    </button>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **始终检查 `isInstalled` 和 `isAvailable`**
   ```typescript
   if (wallet.isInstalled && wallet.isAvailable) {
     // 可以直接连接
   } else if (wallet.isAvailable && !wallet.isInstalled) {
     // 需要跳转安装页面
     window.open(wallet.url)
   }
   ```

2. **使用 `useMemo` 优化钱包列表的筛选和排序**
   ```typescript
   const sortedWallets = useMemo(() => {
     return [...wallets].sort((a, b) => {
       // 最近使用的优先
       if (a.isRecent !== b.isRecent) return a.isRecent ? -1 : 1
       // 已安装的优先
       if (a.isInstalled !== b.isInstalled) return a.isInstalled ? -1 : 1
       return 0
     })
   }, [wallets])
   ```

3. **处理 Safe App 环境**
   ```typescript
   // Safe 钱包需要在 Safe App 环境中才能使用
   const safeWallet = wallets.find((w) => w.adapterId === 'evm-safe')
   if (safeWallet?.isAvailable) {
     // 显示 Safe 专用功能
   }
   ```

4. **使用 `key` 属性确保列表正确渲染**
   ```typescript
   {wallets.map((wallet) => (
     <WalletOption key={wallet.id} wallet={wallet} />
   ))}
   ```

#### ❌ Don'ts

1. **不要在未检查状态的情况下直接连接**
   ```typescript
   // ❌ 错误
   connect(wallets[0])

   // ✅ 正确
   const wallet = wallets[0]
   if (wallet.isInstalled && wallet.isAvailable) {
     connect(wallet)
   }
   ```

2. **不要混淆 `isInstalled` 和 `isAvailable`**
   ```typescript
   // isInstalled: 钱包是否已安装到浏览器
   // isAvailable: 当前是否可以用于连接
   // 这两个是不同的概念！

   // ❌ 错误理解
   if (!wallet.isInstalled) {
     // 不能连接
   }

   // ✅ 正确理解
   // WalletConnect 钱包：isInstalled=false, isAvailable=true
   // 用户可以通过扫描二维码连接
   ```

3. **不要直接修改返回的钱包列表**
   ```typescript
   // ❌ 错误
   wallets.push(newWallet)

   // ✅ 正确
   const modifiedWallets = [...wallets, newWallet]
   ```

4. **不要在条件渲染中假设钱包列表不为空**
   ```typescript
   // ❌ 错误
   const firstWallet = wallets[0] // 可能不存在
   ```

5. **不要忽略 Safe 检测的异步性**
   ```typescript
   // Safe App 检测是异步的，初始渲染时 isSafeAvailable 为 false
   // 避免依赖 Safe 钱包立即出现
   ```
