> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/_common/lib/common/use-network.ts`

# useNetwork Hook Tutorial

## 大白话讲讲这个hook的作用

`useNetwork` 是一个用于获取当前 Aptos 网络配置的 hook。它返回：

- 网络名称（mainnet/testnet）
- 该网络的配置（API URL、合约地址等）

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **网络适配**：根据连接的钱包网络返回对应配置
2. **配置集中**：将网络相关配置统一管理

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
- 无参数，从 Wallet Adapter 获取网络

### 输出（返回值）
```typescript
{
  network: SupportedNetwork      // 网络名称
  default_stable: Token        // 默认稳定币
  contracts: {                 // 合约地址
    swap: string
    masterchef: string
  }
  api: {                      // API 配置
    fetchUrlPrefix: string
    graphqlUrl: string
  }
}
```

### 核心执行逻辑

1. **从钱包获取**：通过 `useWallet` 获取网络名
2. **返回配置**：根据网络名返回对应配置

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useNetwork 的 React hook，用来获取当前连接的网络配置信息。

核心需求：
1. 要从钱包适配器获取当前连接的是哪个网络
2. 返回的内容要包含网络名称、API地址、合约地址等配置信息
3. 要处理钱包没连接的情况，这时候使用一个默认的网络配置

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

1. **处理默认情况**：当用户还没连接钱包的时候，要能提供一个默认的网络配置，而不是返回空或者报错。

2. **验证网络是否支持**：不是所有的网络都可能被支持，代码里要能判断当前网络是不是一个我们支持的网络。如果不支持，要给出提示或者切换到支持的网络。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useNetwork } from '@sushiswap/aptos'

function NetworkInfo() {
  // 使用 useNetwork 获取当前网络配置
  const { network, contracts, api } = useNetwork()

  return (
    <div>
      <p>当前网络: {network}</p>
      <p>Swap 合约: {contracts.swap}</p>
      <p>API: {api.graphqlUrl}</p>
    </div>
  )
}
```

### 常见用法

#### 1. 根据网络配置 API 调用

```tsx
import { useNetwork } from '@sushiswap/aptos'
import { useQuery } from '@tanstack/react-query'

function PoolList() {
  // 使用 useNetwork 获取网络配置
  const { network, api } = useNetwork()

  // 使用网络配置的 API URL
  const { data: pools } = useQuery({
    queryKey: ['pools', network],
    queryFn: async () => {
      const response = await fetch(`${api.fetchUrlPrefix}/pools`)
      return response.json()
    }
  })

  return <div>池子数量: {pools?.length}</div>
}
```

#### 2. 网络切换时自动刷新数据

```tsx
import { useNetwork } from '@sushiswap/aptos'
import { useQuery } from '@tanstack/react-query'
import { useEffect } from 'react'

function TokenBalances({ account }: { account: string }) {
  // 使用 useNetwork 获取当前网络
  const { network } = useNetwork()

  // 使用 network 作为 queryKey，当网络切换时会自动重新获取数据
  const { data: balances, refetch } = useQuery({
    queryKey: ['balances', network, account],
    queryFn: () => fetchBalances(account, network),
    enabled: !!account
  })

  // 网络切换时刷新数据
  useEffect(() => {
    refetch()
  }, [network])

  return <div>余额数量: {Object.keys(balances ?? {}).length}</div>
}
```

#### 3. 获取网络对应的合约地址

```tsx
import { useNetwork } from '@sushiswap/aptos'

function FarmInfo() {
  // 使用 useNetwork 获取网络配置
  const { network, contracts } = useNetwork()

  return (
    <div>
      <h3>网络: {network}</h3>
      <div>
        <p>Swap 合约: {contracts.swap}</p>
        <p>MasterChef 合约: {contracts.masterchef}</p>
      </div>
      {/* 可以在合约交互时使用这些地址 */}
    </div>
  )
}
```

### Do（推荐做法）

- **组合 useAccount 使用**：先检查钱包连接状态，再获取网络配置
- **使用 network 作为缓存 key**：确保不同网络的数据分开缓存
- **设置默认网络处理**：钱包未连接时使用默认网络
- **网络切换时刷新数据**：使用 useEffect 监听 network 变化

### Don't（不推荐做法）

- **不要硬编码合约地址**：使用配置文件或 hook 返回的配置
- **不要忽略网络切换**：切换网络后需要刷新相关数据
- **不要假设只有 mainnet**：支持 testnet 等其他网络

### 相关的其他 hooks

- `useAccount` *(一个React hook，用于获取Aptos钱包连接状态，返回connected、account、network、wallet等信息)*：获取钱包状态
- `usePool` *(一个React hook，用于获取单个流动性池信息，包括代币对、储备量、手续费率等)*：查询池子信息
- `usePools` *(一个React hook，用于获取网络中所有流动性池，从合约资源中读取所有交易对信息)*：获取所有池子
- `useWallet` *(一个React hook，来自@aptos-labs/wallet-adapter-react，用于获取钱包连接状态和执行钱包操作)*：Aptos 钱包适配器 hook
