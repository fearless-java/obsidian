> 源代码路径: `apps/web/src/lib/wallet/hooks/use-accounts.ts`

# useAccounts Hook 教程

## 1. 大白话讲讲这个hook的作用

`useAccounts` *(一个React hook，用于同时获取所有已连接链的账户地址，基于WalletContext的connections数组实现)* 是一个用于**同时获取所有已连接链的账户地址**的 Hook。

简单来说：
- 与 `useAccount` *(一个React hook，用于获取单个链的账户地址)* 只返回**一个**地址不同，`useAccounts` 返回一个包含**三个**地址的对象
- 这个对象分别包含 `evm`、`svm`、`stellar` 三条链的地址
- 每条链的地址是 `{ address: ... }` 的形式
- 如果某条链没有连接，对应的 address 就是 `undefined`

你可以把它想象成"一次性问系统：所有已连接的链的地址分别是什么？"

---

## 2. 讲讲为什么需要封装该hook

### 2.1 批量获取多链地址

在多链 DApp *(去中心化应用，支持多条区块链)* 中，经常需要同时展示用户在多条链上的信息。比如：
- 展示用户 EVM 链的 ETH *(以太坊区块链的原生代币)* 余额
- 展示用户 Solana 链的 SOL *(Solana区块链的原生代币)* 余额
- 展示用户 Stellar 链的 XLM *(Stellar区块链的原生代币)* 余额

如果用 `useAccount`，你需要调用三次：
```typescript
const evmAddress = useAccount('evm')
const svmAddress = useAccount('svm')
const stellarAddress = useAccount('stellar')
```

而 `useAccounts` 一次返回所有：
```typescript
const { evm, svm, stellar } = useAccounts()
// evm.address, svm.address, stellar.address
```

### 2.2 统一的数据结构

`useAccounts` 返回的结构是固定的：
```typescript
{
  evm: { address: AddressFor<'evm'> | undefined },
  svm: { address: AddressFor<'svm'> | undefined },
  stellar: { address: AddressFor<'stellar'> | undefined }
}
```

这种结构便于：
- **解构赋值** *(ES6语法，允许从对象中提取属性为变量)*获取各个链的地址
- **遍历**处理所有链
- **类型安全**地访问每个链的地址

### 2.3 与 useWallets 的区别

| Hook | 返回内容 | 适用场景 |
|------|---------|---------|
| `useAccounts` *(一个hook，返回所有链的地址对象)* | 仅地址 `{ address }` | 只关心地址，不需要连接详情 |
| `useWallets` *(一个hook，返回所有链的完整连接对象)* | 完整连接信息 `{ id, name, chainId, account, ... }` | 需要连接详情，如链 ID、连接状态等 |

---

## 3. 讲讲该hook的执行逻辑和数据流向

### 3.1 函数签名

```typescript
export function useAccounts() {
  const { connections } = useWalletContext()

  return useMemo(() => {
    const getFirstAddress = <TNamespace extends WalletNamespace>(
      namespace: TNamespace,
    ) => {
      for (const c of connections) {
        if (c.namespace !== namespace) continue
        return c.account as AddressFor<ChainIdForNamespace<typeof namespace>>
      }
      return undefined
    }

    return {
      evm: { address: getFirstAddress('evm') },
      svm: { address: getFirstAddress('svm') },
      stellar: { address: getFirstAddress('stellar') },
    }
  }, [connections])
}
```

### 3.2 输入（Input）

无参数。

### 3.3 输出（Output）

```typescript
{
  evm: { address: `0x${string}` | undefined },
  svm: { address: string | undefined },
  stellar: { address: string | undefined }
}
```

| 属性 | 地址类型 | 说明 |
|------|---------|------|
| `evm.address` | `0x${string} \| undefined` | EVM 链地址，未连接时为 `undefined` |
| `svm.address` | `string \| undefined` | Solana 链地址，未连接时为 `undefined` |
| `stellar.address` | `string \| undefined` | Stellar 链地址，未连接时为 `undefined` |

### 3.4 内部执行逻辑

```typescript
return useMemo(() => {
  // 1. 定义辅助函数：在 connections 中查找第一个指定 namespace 的连接
  const getFirstAddress = (namespace) => {
    // 遍历所有连接
    for (const c of connections) {
      // 匹配 namespace
      if (c.namespace !== namespace) continue
      // 返回账户地址
      return c.account
    }
    // 没找到，返回 undefined
    return undefined
  }

  // 2. 返回三条链的地址
  return {
    evm: { address: getFirstAddress('evm') },
    svm: { address: getFirstAddress('svm') },
    stellar: { address: getFirstAddress('stellar') },
  }
}, [connections])
```

### 3.5 数据流向图

```
useAccounts()
     │
     ▼
useWalletContext()
     │
     ▼
connections: WalletConnection[]
     │
     ├──► [c.namespace === 'evm']  ──► evm.address
     ├──► [c.namespace === 'svm']  ──► svm.address
     └──► [c.namespace === 'stellar'] ──► stellar.address
```

### 3.6 与 useAccount 的对比

| 特性 | `useAccount` *(单个链地址获取)* | `useAccounts` *(所有链地址获取)* |
|------|-------------|---------------|
| 返回值 | 单个地址或 `undefined` | 包含三条链地址的对象 |
| 参数 | 可接受 filter（namespace/chainId） | 无参数 |
| 遍历 connections | find（找到第一个） | find（每条链找第一个） |
| 使用场景 | 只需要一条链的地址 | 需要多链地址 |

---

## 四、AI 提示词编写教学

你正在做一个会同时用到三条链的应用，需要获取用户在这些链上的地址。这个Hook就是干这个的——一问就返回所有已连接链的地址，用起来很方便。

先确定一件事：这个Hook返回的是一个对象，不是直接的地址。对象里有 evm、svm、stellar 三个属性，每个属性下面才是 address。所以要记得先用解构拿出各条链，再从每条链里取 address。

最常见的场景是展示多链资产。你可以把三条链都解构出来，然后分别检查每条链有没有地址。如果有就渲染对应的资产卡片，没有就不渲染。用户可能只连了以太坊没连 Solana，这种情况很常见，别假设用户三条链都连着。

有时候你只想知道用户有没有连接任何链。简单，把三个地址用或运算组合起来判断就行了。三条链都没连的话那就是 undefined/ falsy，有任意一条连着就能判断出已连接。

还有个容易忽略的地方：用 React Query 批量请求余额的时候，enabled 参数必须确保是布尔值。地址可能是 undefined，而 undefined 在 JavaScript 里是 falsy，直接传给 enabled 会导致查询行为异常。用 `!!address` 或者 `Boolean(address)` 转一下就行。

新手最常犯的错是不检查地址存在就拿来用。比如直接拿地址去发交易，但用户根本没连那条链，地址是 undefined，程序直接报错。一定要先检查，可以用 if 语句，也可以用短路求值 `address && doSomething(address)`。

还有个容易混淆的地方：这个 Hook 返回的是 `{ evm: { address }, svm: { address }, stellar: { address } }` 这样的对象结构，不是直接的地址。如果你直接 `const accounts = useAccounts()` 然后用 `accounts.evm`，你拿到的是 `{ address: '0x...' }` 这个对象，不是地址字符串。必须 `accounts.evm.address` 才能取到真正的地址。

另外，这个 Hook 不接受任何参数。有人可能觉得可以传个 'evm' 字符串来只获取 EVM 地址，这是不行的。如果只需要单条链，用 `useAccount` 而不是这个。

每条链的地址都可能不存在(undefined)。用户可能只连了以太坊，Solana 的地址就是 undefined。在做任何链上操作之前，都要先确认地址存在。特别是转账、签名这类操作，地址不存在肯定会出问题。

这个 Hook 自己不管理状态，它是纯读取的。数据来自上层的 Context，每次用户连接或断开钱包，底层 store 更新了，这个 Hook 会重新计算返回最新的地址对象。

---

## 5. 该 hook 的用法教学

### 5.1 基础用法

```typescript
import { useAccounts } from '@sushiswap/wallet'

// ✅ 最基本的使用
function AllAddresses() {
  const { evm, svm, stellar } = useAccounts()

  return (
    <div>
      <p>EVM: {evm.address || '未连接'}</p>
      <p>SVM: {svm.address || '未连接'}</p>
      <p>Stellar: {stellar.address || '未连接'}</p>
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景1：多链资产概览

```typescript
function MultiChainOverview() {
  const { evm, svm, stellar } = useAccounts()

  return (
    <div className="multi-chain-overview">
      {evm.address && (
        <ChainCard
          chain="Ethereum"
          address={evm.address}
          balance={<EthBalance address={evm.address} />}
        />
      )}
      {svm.address && (
        <ChainCard
          chain="Solana"
          address={svm.address}
          balance={<SolBalance address={svm.address} />}
        />
      )}
      {stellar.address && (
        <ChainCard
          chain="Stellar"
          address={stellar.address}
          balance={<XlmBalance address={stellar.address} />}
        />
      )}
    </div>
  )
}
```

#### 场景2：统一的钱包地址展示（带截断）

```typescript
function TruncatedAddresses() {
  const { evm, svm, stellar } = useAccounts()

  const truncate = (address: string) => {
    return `${address.slice(0, 6)}...${address.slice(-4)}`
  }

  return (
    <div className="address-list">
      {evm.address && (
        <div className="address-item">
          <ChainIcon chain="evm" />
          <span>{truncate(evm.address)}</span>
        </div>
      )}
      {svm.address && (
        <div className="address-item">
          <ChainIcon chain="svm" />
          <span>{truncate(svm.address)}</span>
        </div>
      )}
      {stellar.address && (
        <div className="address-item">
          <ChainIcon chain="stellar" />
          <span>{truncate(stellar.address)}</span>
        </div>
      )}
    </div>
  )
}
```

#### 场景3：检测连接状态显示不同UI

```typescript
function ConnectionStatus() {
  const { evm, svm, stellar } = useAccounts()

  const hasAnyConnection = evm.address || svm.address || stellar.address
  const connectedChains = []

  if (evm.address) connectedChains.push('EVM')
  if (svm.address) connectedChains.push('Solana')
  if (stellar.address) connectedChains.push('Stellar')

  return (
    <div>
      {hasAnyConnection ? (
        <div className="connected-status">
          <p>已连接 {connectedChains.length} 条链</p>
          <p>链: {connectedChains.join(', ')}</p>
          <button onClick={disconnectAll}>断开所有</button>
        </div>
      ) : (
        <div className="disconnected-status">
          <p>未连接任何钱包</p>
          <ConnectWalletButton />
        </div>
      )}
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **始终检查 address 是否存在**
   ```typescript
   // ✅ 正确 - 使用短路求值
   {evm.address && <Component />}

   // ✅ 正确 - 使用条件判断
   if (evm.address) {
     return <Component address={evm.address} />
   }
   ```

2. **解构获取地址**
   ```typescript
   // ✅ 正确 - 使用解构
   const { evm, svm, stellar } = useAccounts()
   const evmAddress = evm.address
   ```

3. **在enabled条件中使用双重否定确保是布尔值**
   ```typescript
   // ✅ 正确
   const queries = useQueries({
     queries: [
       { queryKey: ['balance', evm.address], enabled: !!evm.address },
     ]
   })
   ```

4. **批量处理多链数据**
   ```typescript
   // ✅ 正确 - 使用useQueries批量获取
   const queries = useQueries({
     queries: [
       { key: ['evm-tokens', evm.address], enabled: !!evm.address },
       { key: ['svm-tokens', svm.address], enabled: !!svm.address },
     ]
   })
   ```

#### ❌ Don'ts

1. **不要直接使用 address 而不检查**
   ```typescript
   // ❌ 错误 - address 可能是 undefined
   const { evm } = useAccounts()
   console.log(evm.address.toLowerCase())

   // ✅ 正确 - 先检查
   const { evm } = useAccounts()
   if (evm.address) {
     console.log(evm.address.toLowerCase())
   }
   ```

2. **不要假设所有链都已连接**
   ```typescript
   // ❌ 错误
   const { stellar } = useAccounts()
   stellar.address.startsWith('G') // 可能报错

   // ✅ 正确 - 始终处理undefined情况
   ```

3. **不要将整个对象当作地址使用**
   ```typescript
   // ❌ 错误
   const accounts = useAccounts()
   sendTo(accounts) // accounts 是对象，不是地址

   // ✅ 正确
   const { evm } = useAccounts()
   if (evm.address) sendTo(evm.address)
   ```

4. **不要忘记处理 undefined 的链**
   ```typescript
   // ❌ 错误 - 假设svm总是连接
   const { svm } = useAccounts()
   svm.address // 可能是undefined
   ```
