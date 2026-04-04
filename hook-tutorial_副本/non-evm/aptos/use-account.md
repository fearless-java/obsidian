> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/_common/lib/common/use-account.ts`

# useAccount Hook Tutorial

## 大白话讲讲这个hook的作用

`useAccount` 是一个用于获取 Aptos 钱包连接状态的 hook。它返回：

- 钱包是否已连接（connected）
- 钱包地址（account?.address）
- 钱包提供的网络信息

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **Wallet Adapter 封装**：封装 `@aptos-labs/wallet-adapter-react` 的 useWallet hook
2. **简化使用**：提供简洁的 API 获取钱包状态

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
- 无参数，直接使用 Wallet Adapter 的 useWallet

### 输出（返回值）
```typescript
{
  connected: boolean           // 是否已连接
  account: { address: string } | null  // 钱包账户
  network: NetworkName | null  // 连接的网络
  wallet: Wallet | null        // 钱包信息
}
```

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useAccount 的 React hook，用来获取钱包的连接状态。

核心需求：
1. 能够判断钱包是否已经连接
2. 能获取到钱包的地址信息
3. 能获取到当前连接的网络信息
4. 要用 @aptos-labs/wallet-adapter-react 这个库来实现

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

1. **上下文依赖**：这个hook必须在钱包提供者（WalletProvider）的环境里才能正常工作。就像你不能在组件外面使用useState一样，这个hook也需要在正确的上下文中运行。

2. **空值判断**：获取到的账户信息可能是空的，比如用户还没连接钱包的时候。所以在你的代码里，任何用到account的地方都要先检查它是否存在，不然程序会报错。

### 这个Hook怎么管理状态

这是一个同步的、实时的状态管理。它不需要像数据请求那样用React Query来管理缓存和刷新。因为钱包连接状态是由用户的操作直接决定的——用户点击连接按钮，状态就变了；用户断开连接，状态也变了。所以它只需要直接返回当前状态就行，不需要额外的异步处理。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useAccount } from '@sushiswap/wagmi'
import { useEffect } from 'react'

function WalletStatus() {
  // 使用 useAccount hook 获取钱包连接状态
  const { connected, account, network, wallet } = useAccount()

  if (!connected) {
    return <div>请先连接钱包</div>
  }

  return (
    <div>
      <p>钱包地址: {account?.address}</p>
      <p>网络: {network}</p>
      <p>钱包类型: {wallet?.name}</p>
    </div>
  )
}
```

### 常见用法

#### 1. 条件渲染（钱包连接状态）

```tsx
function SwapButton() {
  // 使用 useAccount 获取连接状态
  const { connected } = useAccount()

  // 根据连接状态条件渲染
  return connected ? (
    <button>进行 Swap</button>
  ) : (
    <button onClick={connectWallet}>连接钱包</button>
  )
}
```

#### 2. 获取钱包地址用于查询

```tsx
function TokenBalance() {
  // 获取当前连接的钱包地址
  const { account } = useAccount()

  // 使用地址查询余额
  const { data: balance } = useTokenBalance({
    account: account?.address,
    currency: '0x1::aptos_coin::AptosCoin'
  })

  return <span>余额: {balance}</span>
}
```

#### 3. 网络切换检测

```tsx
function NetworkWarning() {
  // 获取当前连接的网络
  const { network } = useAccount()

  // 检测是否在支持的网络上
  const isSupported = network === 'mainnet' || network === 'testnet'

  if (!isSupported) {
    return <div className="warning">请切换到支持的网络</div>
  }

  return null
}
```

### Do（推荐做法）

- **总是进行连接状态检查**：在使用钱包功能前先检查 `connected` 状态
- **处理 null 情况**：account 和 wallet 可能是 null，需要进行空值检查
- **在 WalletProvider 内使用**：确保在钱包上下文内调用 hook
- **组合多个 hooks 使用**：配合 `useNetwork` 获取网络配置

### Don't（不推荐做法）

- **不要在 WalletProvider 外使用**：这会导致错误或返回 null
- **不要忽略 null 检查**：直接访问 `account.address` 可能导致应用崩溃
- **不要在条件渲染外依赖钱包状态做权限控制**：使用专门的权限 hook

### 相关的其他 hooks

- `useNetwork` *(一个React hook，用于获取当前链网络配置信息，返回网络名称、默认稳定币、合约地址等配置)*：获取网络配置
- `useDisconnect` *(一个React hook，用于断开钱包连接，调用时会清除钱包连接状态)*：断开连接
- `useConnect` *(一个React hook，用于获取可用的钱包连接选项列表)*：获取可连接的钱包列表
