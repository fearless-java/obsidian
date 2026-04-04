> 源代码路径: `apps/web/src/lib/wallet/hooks/use-wallets.ts`

# useWallets Hook 教程

## 1. 大白话讲讲这个hook的作用

`useWallets` *(一个React hook，用于获取所有已连接链的完整连接信息，基于WalletContext实现)* 是一个用于**获取所有已连接链的完整连接信息**的 Hook。

简单来说：
- 它返回一个包含 `evm`、`svm`、`stellar` 三个属性的对象
- 每个属性是对应链的**完整连接信息**（WalletConnection *(已连接钱包的完整信息对象，包含id、name、namespace、account、chainId等)*）
- 如果某条链没有连接，该属性就是 `undefined`
- 类似于 `useAccounts`（返回地址）和 `useWallet`（返回单个连接）的结合

你可以把它想象成"一次性问系统：所有已连接的链的完整钱包信息是什么？"。

---

## 2. 讲讲为什么需要封装该hook

### 2.1 批量获取多链完整信息

`useWallet` *(单个链的完整连接)* 只能获取**一条链**的连接信息。如果需要同时获取多条链的信息：

```typescript
// 使用 useWallet（需要多次调用）
const evmWallet = useWallet('evm')
const svmWallet = useWallet('svm')
const stellarWallet = useWallet('stellar')

// 使用 useWallets（一次获取）
const { evm, svm, stellar } = useWallets()
```

### 2.2 与其他 Hook 的对比

| Hook | 返回内容 | 筛选能力 |
|------|---------|---------|
| `useAccount` *(单个链的地址)* | 单个地址或 `undefined` | 支持 namespace/chainId |
| `useAccounts` *(所有链的地址)* | `{ evm, svm, stellar }` 每个只有 address | 无（全部获取） |
| `useWallet` *(单个链的完整信息)* | 单个 WalletConnection 或 `undefined` | 支持 namespace/chainId |
| `useWallets` *(所有链的完整信息)* | `{ evm, svm, stellar }` 每个是完整连接 | 无（全部获取） |

### 2.3 提供链 ID 信息

与 `useAccounts` *(只返回地址)* 的区别在于，`useWallets` 返回完整的 `WalletConnection`，包含 `chainId`：

```typescript
const { evm, svm, stellar } = useWallets()

// useAccounts 只能得到地址
evm.address  // '0x...'

// useWallets 还能得到链 ID
evm.chainId   // 1 (Ethereum *(以太坊主网)*)
evm.account   // '0x...'（同 address）
```

---

## 3. 讲讲该hook的执行逻辑和数据流向

### 3.1 函数签名

```typescript
export function useWallets() {
  const { connections } = useWalletContext()

  return useMemo(() => {
    const getFirstWallet = <TNamespace extends WalletNamespace>(
      namespace: TNamespace,
    ) => {
      for (const c of connections) {
        if (c.namespace !== namespace) continue
        return c as WalletConnection<ChainIdForNamespace<typeof namespace>>
      }
      return undefined
    }

    return {
      evm: getFirstWallet('evm'),
      svm: getFirstWallet('svm'),
      stellar: getFirstWallet('stellar'),
    }
  }, [connections])
}
```

### 3.2 输入（Input）

无参数。

### 3.3 输出（Output）

```typescript
{
  evm: WalletConnection<EvmChainId> | undefined,
  svm: WalletConnection<SvmChainId> | undefined,
  stellar: WalletConnection<StellarChainId> | undefined
}
```

| 属性 | 类型 | 说明 |
|------|------|------|
| `evm` | `WalletConnection<EvmChainId> \| undefined` | EVM 连接信息 |
| `svm` | `WalletConnection<SvmChainId> \| undefined` | Solana 连接信息 |
| `stellar` | `WalletConnection<StellarChainId> \| undefined` | Stellar 连接信息 |

### 3.4 WalletConnection 结构

```typescript
interface WalletConnection<TChainId> {
  id: string           // 'evm:io.metamask'
  name: string         // 'MetaMask'
  namespace: 'evm' | 'svm' | 'stellar'
  account: AddressFor<TChainId>  // 钱包地址
  chainId: TChainId    // 链 ID
  icon?: string        // 钱包图标
}
```

### 3.5 内部执行逻辑

```typescript
return useMemo(() => {
  // 1. 定义辅助函数：查找第一个指定 namespace 的连接
  const getFirstWallet = (namespace) => {
    // 遍历所有连接
    for (const c of connections) {
      // 匹配 namespace
      if (c.namespace !== namespace) continue
      // 返回完整的连接对象
      return c
    }
    // 没找到，返回 undefined
    return undefined
  }

  // 2. 返回三条链的连接信息
  return {
    evm: getFirstWallet('evm'),
    svm: getFirstWallet('svm'),
    stellar: getFirstWallet('stellar'),
  }
}, [connections])
```

### 3.6 数据流向图

```
useWallets()
     │
     ▼
useWalletContext()
     │
     ▼
connections: WalletConnection[]
     │
     ├──► getFirstWallet('evm')     ──► evm: WalletConnection | undefined
     ├──► getFirstWallet('svm')      ──► svm: WalletConnection | undefined
     └──► getFirstWallet('stellar') ──► stellar: WalletConnection | undefined
```

### 3.7 使用示例

```typescript
// ✅ 获取所有连接的完整信息
function MultiChainDashboard() {
  const { evm, svm, stellar } = useWallets()

  return (
    <div>
      {evm && <EvmPanel wallet={evm} />}
      {svm && <SvmPanel wallet={svm} />}
      {stellar && <StellarPanel wallet={stellar} />}
    </div>
  )
}

// ✅ 使用链 ID 进行网络操作
function NetworkBadge() {
  const { evm } = useWallets()

  if (!evm) return null

  return (
    <Badge
      chainId={evm.chainId}
      onClick={() => switchChain({ chainId: evm.chainId })}
    />
  )
}

// ✅ 结合 useChainIds
function ConnectedChainIds() {
  const { evm, svm, stellar } = useWallets()
  const chainIds = useChainIds()

  return (
    <div>
      <p>已连接 {chainIds.length} 条链</p>
      {evm && <p>Ethereum (ChainId: {evm.chainId})</p>}
      {svm && <p>Solana (ChainId: {svm.chainId})</p>}
    </div>
  )
}
```

---

## 四、AI 提示词编写教学

你正在做一个需要同时获取多条链连接信息的工具。这个 Hook 会同时返回三条链的完整连接信息，用起来很方便。

先确定一件事：这个 Hook 返回的是一个对象，不是直接的地址。对象里有 evm、svm、stellar 三个属性，每个属性是对应链的完整连接信息。所以要用解构赋值分别获取每条链，比如 `const { evm, svm, stellar } = useWallets()`。

最常见的用法是展示多链资产。你可以对每条链检查是否存在，如果存在就渲染对应的余额组件。比如 `evm && <EvmAssetCard address={evm.account} />`。用户可能只连了部分链，不是三条链都连着，一定要有这个判断。

有时候你只想检查用户是否连接了任何链。在头部组件里，可以把三个对象用或运算组合起来判断。如果有任意一条存在连接，就显示用户菜单按钮；如果都没有，就显示连接按钮。

还有个场景是遍历所有已连接的链来做操作。你可以把三条链组合成一个数组，过滤掉不存在的连接，得到一个只有已连接链的数组。这样就可以用 map 来统一渲染或者做其他处理。

新手最容易犯的错是不检查对象是否存在就直接访问属性。比如直接 `console.log(evm.chainId)`，但如果用户没连接 EVM 链，evm 就是 undefined，访问它的属性会报错。正确做法是先检查 `if (evm)`，只有存在的情况下才访问它的属性。

另一个常犯的错是在用 filter 过滤 undefined 之后忘记类型标注。如果你直接用 `.filter(Boolean)`，TypeScript 只知道过滤掉了 falsy 值，但不知道过滤后数组里一定不是 undefined 了。所以类型还是 `(WalletConnection | undefined)[]`，不是 `WalletConnection[]`。如果你后面要直接访问数组元素的属性，TypeScript 会报错。正确做法是用类型守卫 `.filter((w): w is WalletConnection => !!w)` 来明确告诉 TypeScript 过滤后的一定是 WalletConnection 类型。

每个属性都可能是 undefined。用户可能只连接了部分链，所以任何使用某条链连接的地方，都要先检查它是否存在。特别是访问链 ID、账户地址这些属性时，如果那条链根本没连接，直接访问会报错。

还有个重要区别：这个 Hook 返回的是 WalletConnection 类型，不是 WalletWithState 类型。WalletConnection 有 account、chainId、name 这些已连接钱包才有的属性，但没有 isInstalled、isAvailable、isRecent 这些状态属性。如果你想检查钱包是否安装，要用 `useEvmWallets` 那个 Hook，它返回的是 WalletWithState 类型。

按 namespace 查找只返回第一个匹配的连接。如果用户同时连了多个 EVM 钱包，这个 Hook 只返回其中第一个。要获取所有连接，得用另一个 Hook。

这个 Hook 自己不管理状态，是纯读取的。数据来自上层 Context 提供的 connections 数组。当用户的连接状态变化时，底层的 store 会更新这个数组，然后这个 Hook 会重新计算，返回三条链各自的连接信息或者 undefined。

---

## 5. 该 hook 的用法教学

### 5.1 基础用法

```typescript
import { useWallets } from '@sushiswap/wallet'

// ✅ 最基本的使用
function AllWalletsInfo() {
  const { evm, svm, stellar } = useWallets()

  return (
    <div>
      <div>EVM: {evm ? `${evm.name} (${evm.account})` : '未连接'}</div>
      <div>SVM: {svm ? `${svm.name} (${svm.account})` : '未连接'}</div>
      <div>Stellar: {stellar ? `${stellar.name} (${stellar.account})` : '未连接'}</div>
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景1：多链资产面板

```typescript
function MultiChainAssetPanel() {
  const { evm, svm, stellar } = useWallets()

  return (
    <div className="asset-panel">
      {evm && (
        <EvmAssetCard
          address={evm.account}
          chainId={evm.chainId}
          walletIcon={evm.icon}
        />
      )}
      {svm && (
        <SvmAssetCard
          address={svm.account}
          chainId={svm.chainId}
          walletIcon={svm.icon}
        />
      )}
      {stellar && (
        <StellarAssetCard
          address={stellar.account}
          walletIcon={stellar.icon}
        />
      )}
      {!evm && !svm && !stellar && <EmptyState />}
    </div>
  )
}
```

#### 场景2：连接状态头部

```typescript
function ConnectedHeader() {
  const { evm, svm, stellar } = useWallets()

  const truncate = (address: string) => {
    return `${address.slice(0, 6)}...${address.slice(-4)}`
  }

  return (
    <header className="connected-header">
      {evm && (
        <div className="wallet-chip">
          <img src={evm.icon} alt={evm.name} className="wallet-icon" />
          <span>Ethereum</span>
          <span className="address">{truncate(evm.account)}</span>
        </div>
      )}
      {svm && (
        <div className="wallet-chip">
          <img src={svm.icon} alt={svm.name} className="wallet-icon" />
          <span>Solana</span>
          <span className="address">{truncate(svm.account)}</span>
        </div>
      )}
      {stellar && (
        <div className="wallet-chip">
          <img src={stellar.icon} alt={stellar.name} className="wallet-icon" />
          <span>Stellar</span>
          <span className="address">{truncate(stellar.account)}</span>
        </div>
      )}
      <button onClick={showWalletMenu}>钱包菜单</button>
    </header>
  )
}
```

#### 场景3：断开连接功能

```typescript
function DisconnectButtons() {
  const { evm, svm, stellar } = useWallets()

  const disconnect = async (namespace: 'evm' | 'svm' | 'stellar') => {
    try {
      await disconnectAsync(namespace)
    } catch (error) {
      console.error('断开连接失败:', error)
    }
  }

  return (
    <div className="disconnect-panel">
      {evm && (
        <button onClick={() => disconnect('evm')}>
          断开 Ethereum
        </button>
      )}
      {svm && (
        <button onClick={() => disconnect('svm')}>
          断开 Solana
        </button>
      )}
      {stellar && (
        <button onClick={() => disconnect('stellar')}>
          断开 Stellar
        </button>
      )}
    </div>
  )
}
```

#### 场景4：多链功能入口

```typescript
function MultiChainFeatures() {
  const { evm, svm, stellar } = useWallets()

  return (
    <div className="features-grid">
      {evm && (
        <FeatureCard
          chain="Ethereum"
          icon={evm.icon}
          features={['Swap', 'Bridge', 'Stake']}
        />
      )}
      {svm && (
        <FeatureCard
          chain="Solana"
          icon={svm.icon}
          features={['Swap', 'Stake SOL']}
        />
      )}
      {stellar && (
        <FeatureCard
          chain="Stellar"
          icon={stellar.icon}
          features={['Swap', 'Send']}
        />
      )}
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **解构获取每个链的连接信息**
   ```typescript
   // ✅ 正确
   const { evm, svm, stellar } = useWallets()
   ```

2. **始终检查 undefined**
   ```typescript
   // ✅ 正确
   if (evm) {
     console.log(evm.chainId)
   }
   ```

3. **使用类型守卫进行类型收窄**
   ```typescript
   // ✅ 正确
   const wallets = [evm, svm, stellar].filter((w): w is WalletConnection => !!w)
   ```

4. **结合 useChainIds 使用**
   ```typescript
   // ✅ 正确
   const { evm, svm, stellar } = useWallets()
   const chainIds = useChainIds()
   // chainIds.length > 0 表示至少有一条链连接
   ```

#### ❌ Don'ts

1. **不要假设所有链都连接了**
   ```typescript
   // ❌ 错误
   const { stellar } = useWallets()
   stellar.account.startsWith('G') // stellar可能是undefined

   // ✅ 正确
   const { stellar } = useWallets()
   if (stellar) {
     stellar.account.startsWith('G')
   }
   ```

2. **不要直接访问 WalletConnection 没有的属性**
   ```typescript
   // ❌ 错误 - WalletConnection没有isInstalled
   const { evm } = useWallets()
   if (evm.isInstalled) { // 编译错误

   // ✅ 正确 - WalletConnection有account, chainId, name等
   // WalletWithState有isInstalled, isAvailable, isRecent
   ```

3. **不要忘记解构**
   ```typescript
   // ❌ 错误 - wallets是整个对象
   const wallets = useWallets()
   wallets.evm.account // 不直观

   // ✅ 正确 - 解构更清晰
   const { evm } = useWallets()
   if (evm) evm.account
   ```

4. **不要在依赖数组中直接使用对象**
   ```typescript
   // ❌ 错误
   }, [evm, svm, stellar])

   // ✅ 正确 - 使用具体属性
   }, [evm?.chainId, svm?.chainId, stellar?.chainId])
   ```
