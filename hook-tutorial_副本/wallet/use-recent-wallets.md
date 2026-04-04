> 源代码路径: `apps/web/src/lib/wallet/hooks/use-recent-wallets.ts`

# useRecentWallets Hook 教程

## 1. 大白话讲讲这个hook的作用

`useRecentWallets` *(一个React hook，用于管理用户最近使用过的钱包列表，数据存储在LocalStorage中)* 是一个用于**管理用户最近使用过的钱包列表**的 Hook。

简单来说：
- 它会在用户连接钱包时，自动记录这个钱包的 ID
- 最多记录**3个**最近使用的钱包（可以配置）
- 当用户打开钱包选择列表时，最近使用的钱包会排在最前面
- 数据存储在浏览器的 **LocalStorage** *(浏览器内置的键值存储系统，用于持久化存储小量数据)* 中，刷新页面后仍然保留

你可以把它想象成"浏览器记住你最近用过的钱包，下次打开直接显示，方便快速选择"。

---

## 2. 讲讲为什么需要封装该hook

### 2.1 提升用户体验

在 Web3 DApp *(去中心化应用)* 中，用户经常需要反复连接/断开钱包。通过记录最近使用的钱包：
- 用户可以快速重新连接之前用过的钱包
- 减少用户查找钱包的时间
- 提供更流畅的连接体验

### 2.2 LocalStorage 持久化

钱包选择是**客户端偏好**，不需要存储在服务器端：
- 每个用户的浏览器都是独立的
- 数据量小（只存钱包 ID 列表）
- 使用 LocalStorage 简单高效

### 2.3 防止重复 + 自动去重

当用户连接一个新钱包时：
1. 如果这个钱包已经在列表中，先移除它
2. 把这个钱包加到列表最前面
3. 如果列表超过3个，移除最后一个

```typescript
// 例如当前列表: ['metamask', 'phantom']
// 用户连接了 rabby
// 新列表: ['rabby', 'metamask', 'phantom']

// 用户又连接了 metamask
// 新列表: ['metamask', 'rabby', 'phantom']  // rabby 被移除了
```

---

## 3. 讲讲该hook的执行逻辑和数据流向

### 3.1 函数签名

```typescript
export function useRecentWallets() {
  const [recentWallets, _setRecentWallets] = useLocalStorage<string[]>(
    RECENT_WALLETS_KEY,
    [],
  )

  const addRecentWallet = useCallback(
    (walletId: string) => {
      _setRecentWallets((prev) => {
        const next = [walletId, ...prev.filter((id) => id !== walletId)]
        return next.slice(0, MAX_RECENT_WALLETS)
      })
    },
    [_setRecentWallets],
  )

  return useMemo(
    () => ({
      recentWallets,
      isRecentWallet: (walletId: string) => recentWallets.includes(walletId),
      addRecentWallet,
    }),
    [recentWallets, addRecentWallet],
  )
}
```

### 3.2 常量定义

```typescript
const RECENT_WALLETS_KEY = 'recent-wallets'
const MAX_RECENT_WALLETS = 3
```

### 3.3 输入（Input）

无参数。

### 3.4 输出（Output）

| 属性 | 类型 | 说明 |
|------|------|------|
| `recentWallets` | `string[]` | 最近使用过的钱包 ID 数组 |
| `isRecentWallet` | `(walletId: string) => boolean` | 判断指定钱包 ID 是否在最近列表中 |
| `addRecentWallet` | `(walletId: string) => void` | 添加一个钱包到最近列表 |

### 3.5 内部执行逻辑

#### 3.5.1 数据读取

```typescript
// 从 LocalStorage 读取，默认为空数组
const [recentWallets, _setRecentWallets] = useLocalStorage<string[]>(
  'recent-wallets',
  [],
)
```

#### 3.5.2 添加钱包到最近列表

```typescript
const addRecentWallet = useCallback((walletId: string) => {
  _setRecentWallets((prev) => {
    // 1. 过滤掉当前 walletId（去除重复）
    // 2. 把新 walletId 加到最前面
    // 3. 只保留前 MAX_RECENT_WALLETS 个
    const next = [
      walletId,
      ...prev.filter((id) => id !== walletId)
    ].slice(0, MAX_RECENT_WALLETS)

    return next
  })
}, [_setRecentWallets])
```

#### 3.5.3 判断是否在最近列表中

```typescript
const isRecentWallet = (walletId: string) => recentWallets.includes(walletId)
```

### 3.6 数据流向图

```
LocalStorage: 'recent-wallets'
     │
     ▼
useLocalStorage<string[]>()
     │
     ├──► recentWallets: string[]
     │         │
     │         └──► isRecentWallet(walletId) ──► boolean
     │
     └──► _setRecentWallets (内部更新函数)
               │
               ├──► addRecentWallet(walletId)
               │         │
               │         └──► [walletId, ...prev].slice(0, 3)
               │                   │
               │                   ▼
               │              写入 LocalStorage
               │
               └──► 触发 re-render
```

### 3.7 使用示例

```typescript
// 场景1：判断某个钱包是否在最近列表中
const { isRecentWallet } = useRecentWallets()
const isMetaMaskRecent = isRecentWallet('evm:io.metamask')
// true 或 false

// 场景2：获取最近钱包列表
const { recentWallets } = useRecentWallets()
// ['evm:io.metamask', 'svm:phantom', 'evm:io.rabby']

// 场景3：添加钱包到最近列表
const { addRecentWallet } = useRecentWallets()
addRecentWallet('evm:io.metamask')
```

---

## 四、AI 提示词编写教学

你正在做一个会记住用户行为的工具，让用户下次打开钱包列表时，最近用过的钱包能排在最前面。这个 Hook 就是负责这件事的。

先确定一件事：钱包 ID 必须带前缀。正确格式是 `evm:io.metamask`、`svm:phantom`、`stellar:freighter` 这样的，包含了命名空间前缀。如果只传 `metamask` 这种缺了前缀的 ID，系统是识别不了的。

这个 Hook 最常见的用途是在钱包列表里标记哪些是用户最近用过的。比如你有一个钱包列表组件，可以用 `isRecentWallet(walletId)` 来判断这个钱包是否在最近列表里，然后显示一个"最近"的标签。

另一个常见需求是把最近用过的钱包排在最前面。配合 `useMemo` 对钱包列表进行排序，让最近使用过的排在前面。排序逻辑很简单：最近使用的权重为 1，非最近使用的权重为 0，然后比较这两个权重值来决定顺序。

还有一种情况是连接成功后自动记录到最近列表。用户在钱包列表里点击连接，等连接成功后再调用 `addRecentWallet(walletId)` 把钱包 ID 记录进去。为什么要等成功才添加？因为失败的情况下不应该污染最近使用记录，用户下次打开列表不应该看到失败过的钱包。

新手常犯的一个错是在连接失败时也添加到最近列表。应该在 try-catch 块里，只有成功的情况下才调用添加函数。失败时打印错误日志就行了，但不应该添加到最近列表。

另一个错是直接修改返回的数组。有人可能觉得拿到了 recentWallets 数组然后直接 push 进去，但这是不行的。这个数组是只读的，修改它不会生效，而且会违反 React 的数据不可变原则。正确做法是调用 `addRecentWallet` 函数来添加。

数据存在浏览器的 LocalStorage 里。这意味着刷新页面后数据还在，但只在同一个域名下有效。不同标签页共享同一份数据，但如果在一个标签页里修改了最近列表，其他标签页不会自动更新，除非刷新。

最多只能记录 3 个最近钱包。超过 3 个时，最早记录的那个会被挤掉。这是刻意设计的限制，避免列表太长。

这个 Hook 用了一个封装过的 LocalStorage Hook 来管理状态。读取时会从 LocalStorage 里拿数据，修改时会先更新内存状态然后同步到 LocalStorage，最后触发 React 重新渲染。

---

## 5. 该 hook 的用法教学

### 5.1 基础用法

```typescript
import { useRecentWallets } from '@sushiswap/wallet'

// ✅ 基本用法
function RecentWalletDisplay() {
  const { recentWallets, isRecentWallet, addRecentWallet } = useRecentWallets()

  return (
    <div>
      <p>最近使用的钱包: {recentWallets.length} 个</p>
      {recentWallets.map((id) => (
        <WalletBadge key={id} walletId={id} />
      ))}
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景1：钱包列表排序（最近使用优先）

```typescript
function WalletListSorted() {
  const { isRecentWallet } = useRecentWallets()
  const wallets = useWalletsList()

  const sortedWallets = useMemo(() => {
    return [...wallets].sort((a, b) => {
      // 1. 最近使用的排最前面
      if (a.isRecent !== b.isRecent) {
        return a.isRecent ? -1 : 1
      }
      // 2. 已安装的排前面
      if (a.isInstalled !== b.isInstalled) {
        return a.isInstalled ? -1 : 1
      }
      return 0
    })
  }, [wallets, isRecentWallet])

  return (
    <div className="wallet-list">
      {sortedWallets.map((wallet) => (
        <WalletItem key={wallet.id} wallet={wallet} />
      ))}
    </div>
  )
}
```

#### 场景2：最近使用标签显示

```typescript
function WalletItemWithBadge({ wallet }: { wallet: WalletWithState }) {
  const { isRecentWallet } = useRecentWallets()
  const isRecent = isRecentWallet(wallet.id)

  return (
    <div className={`wallet-item ${isRecent ? 'recent' : ''}`}>
      <img src={wallet.icon} alt={wallet.name} />
      <span>{wallet.name}</span>
      {isRecent && <span className="recent-badge">最近使用</span>}
      <ActionButton wallet={wallet} />
    </div>
  )
}
```

#### 场景3：连接成功后添加到最近

```typescript
function ConnectWalletButton({ wallet }: { wallet: WalletWithState }) {
  const { addRecentWallet } = useRecentWallets()

  const handleConnect = async () => {
    try {
      await connectWallet(wallet)
      // ✅ 连接成功后才添加
      addRecentWallet(wallet.id)
    } catch (error) {
      console.error('连接失败:', error)
      // 失败时不添加
    }
  }

  return (
    <button onClick={handleConnect} disabled={!wallet.isAvailable}>
      {wallet.isInstalled ? '连接' : '安装'}
    </button>
  )
}
```

#### 场景4：独立的最近钱包快速连接

```typescript
function RecentWalletsQuickConnect() {
  const { recentWallets } = useRecentWallets()
  const allWallets = useWalletsList()

  // 获取最近钱包的详细信息
  const recentWalletDetails = useMemo(() => {
    return recentWallets
      .map((id) => allWallets.find((w) => w.id === id))
      .filter(Boolean) as WalletWithState[]
  }, [recentWallets, allWallets])

  if (recentWalletDetails.length === 0) {
    return null
  }

  return (
    <div className="recent-wallets">
      <h3>最近使用</h3>
      <div className="recent-wallet-grid">
        {recentWalletDetails.map((wallet) => (
          <QuickConnectButton key={wallet.id} wallet={wallet} />
        ))}
      </div>
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **仅在连接成功时添加到最近列表**
   ```typescript
   // ✅ 正确
   async function connect(walletId: string) {
     try {
       await connectAsync(walletId)
       addRecentWallet(walletId) // 成功后才添加
     } catch (error) {
       // 失败处理
     }
   }
   ```

2. **使用完整的钱包ID（带namespace前缀）**
   ```typescript
   // ✅ 正确
   addRecentWallet('evm:io.metamask')
   addRecentWallet('svm:phantom')
   addRecentWallet('stellar:freighter')
   ```

3. **使用 isRecentWallet 函数判断状态**
   ```typescript
   // ✅ 正确
   const { isRecentWallet } = useRecentWallets()
   if (isRecentWallet(wallet.id)) {
     // 显示最近使用标记
   }
   ```

4. **使用 useMemo 优化排序**
   ```typescript
   // ✅ 正确
   const sortedWallets = useMemo(() => {
     return [...wallets].sort((a, b) => {
       if (isRecentWallet(a.id) !== isRecentWallet(b.id)) {
         return isRecentWallet(a.id) ? -1 : 1
       }
       return 0
     })
   }, [wallets, isRecentWallet])
   ```

#### ❌ Don'ts

1. **不要在连接失败时添加到最近列表**
   ```typescript
   // ❌ 错误
   async function connect(walletId: string) {
     await connectAsync(walletId)
     addRecentWallet(walletId) // 即使失败也添加
   }

   // ✅ 正确
   async function connect(walletId: string) {
     try {
       await connectAsync(walletId)
       addRecentWallet(walletId)
     } catch (e) {
       // 不添加
     }
   }
   ```

2. **不要直接修改 recentWallets**
   ```typescript
   // ❌ 错误
   const { recentWallets } = useRecentWallets()
   recentWallets.push(newId)

   // ✅ 正确
   const { addRecentWallet } = useRecentWallets()
   addRecentWallet(newId)
   ```

3. **不要使用不带namespace的walletId**
   ```typescript
   // ❌ 错误
   addRecentWallet('metamask')

   // ✅ 正确
   addRecentWallet('evm:io.metamask')
   ```

4. **不要假设 recentWallets 长度固定**
   ```typescript
   // ❌ 错误
   const latest = recentWallets[0]

   // ✅ 正确
   if (recentWallets.length > 0) {
     const latest = recentWallets[0]
   }
   ```

5. **不要在每次渲染时都调用 addRecentWallet**
   ```typescript
   // ❌ 错误
   addRecentWallet(wallet.id) // 这会在每次渲染时都调用

   // ✅ 正确 - 只在用户点击时调用
   onClick={() => handleConnect(wallet.id)}
   ```
