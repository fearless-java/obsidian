> 源代码路径: `apps/web/src/lib/wagmi/hooks/balances/useBalanceWeb3.ts`

# useBalanceWeb3

## 大白话讲讲这个hook的作用

`useBalanceWeb3` 是一个查询钱包地址代币余额的 Hook。它会返回一个地址在指定链上持有某种代币的数量。

大白话：就是查询"我的钱包里有多少钱"。支持查询任何 ERC20 代币的余额，也支持原生币（ETH、MATIC 等）。

它还自动处理：
- 原生币和 ERC20 代币的区分
- 定时刷新（每 10 秒）
- 窗口焦点刷新

## 讲讲为什么需要封装该hook

1. **统一原生币和代币**：原生币用 `useBalance`，ERC20 用 `readContract`，封装后统一返回 `Amount` 类型。

2. **定时刷新**：需要定期刷新余额数据，封装后自动处理。

3. **窗口焦点刷新**：用户切换回页面时应该刷新余额，封装后自动处理。

4. **自动 watch**：结合 `useWatchByInterval` 自动监听更新。

5. **缓存和性能**：通过 React Query 缓存结果，避免重复请求。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseBalanceParams {
  chainId: EvmChainId | undefined    // 链 ID
  currency: EvmCurrency | undefined    // 代币（原生币或 ERC20）
  account: Address | undefined        // 钱包地址
  enabled?: boolean                    // 是否启用
}
```

### 输出 (Outputs)

```typescript
{
  data: Amount<EvmCurrency> | null,  // 余额
  isLoading: boolean,
  isError: boolean,
  refetch: () => void,
  // ... 其他 query 状态
}
```

### 执行逻辑详解

#### 1. 获取原生币余额

```typescript
const { data: nativeBalance, queryKey } = useBalance({
  chainId,
  address: account,
  query: { enabled }
})
```

#### 2. 设置定时刷新

```typescript
useWatchByInterval({ key: queryKey, interval: 10000 })
```

#### 3. 查询函数

```typescript
const queryFn = async () => {
  const data = await queryFnUseBalances({
    chainId,
    config,
    currencies: [currency],
    account,
    nativeBalance,
  })
  // 根据 currency 类型返回对应余额
  return data?.[
    currency.type === 'native' ? zeroAddress : currency.wrap().address
  ] ?? null
}
```

#### 4. React Query 配置

```typescript
return useQuery({
  queryKey: ['useBalance', { chainId, currency, account, nativeBalance: serialize(nativeBalance) }],
  queryFn,
  refetchInterval: 10000,           // 10 秒刷新
  refetchIntervalInBackground: false,  // 窗口不可见时不刷新
  refetchOnWindowFocus: true,       // 窗口获焦时刷新
  enabled: Boolean(chainId && account && enabled)
})
```

### 数据流向图

```
输入参数 (chainId, currency, account)
         ↓
    ┌────────────────────────────────────┐
    │  useBalance 查询原生币余额          │
    │  (如果是原生币)                     │
    └────────────────────────────────────┘
         ↓
    useWatchByInterval 定时刷新监听
         ↓
    ┌────────────────────────────────────┐
    │  queryFnUseBalances 执行查询        │
    │  (通过 readContracts 批量查询)     │
    └────────────────────────────────────┘
         ↓
    根据 currency 类型返回对应 Amount
         ↓
    缓存并返回结果
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：参数校验是必须的**。在开启查询之前，必须确保chainId和account这两个参数都存在。如果其中一个是undefined或者null，那查询肯定查不到任何有意义的数据。可以用一个布尔表达式同时检查所有参数，比如enabled应该同时判断account和currency都存在。

**第二点：nativeBalance要序列化**。原生币的余额数据是一个复杂的对象，不能直接放进queryKey里作为缓存的标识。必须先序列化这个对象，否则React Query的缓存机制可能会出问题。这个序列化过程sushi库已经帮你处理好了。

**第三点：刷新间隔要合理**。定时刷新间隔设置为10秒是一个比较平衡的选择。太短的话可能会触发RPC的rate limit，太长的话用户看到的数据又不够实时。当然这个可以根据实际情况调整，比如价格数据可以更频繁一点。

**第四点：Amount类型很方便**。返回的余额数据是Amount类型，这个对象身上有很多好用的方法。比如你想格式化显示余额，可以用toSignificant(6)保留6位有效数字，想比较两个余额的大小，可以直接用gt、lt这些方法。不用担心精度问题，Amount已经帮你处理好了。

### 约束条件要记住

**第一，chainId要是有效的**。必须是区块链网络的ID，比如以太坊主网是1，Polygon是137。如果传个无效的ID，查询肯定会失败。

**第二，account要合法**。必须是有效的以太坊地址格式，比如0x开头的42位字符串。如果地址格式不对，查询也不会有正确的结果。

**第三，currency可以多样**。可以是原生币比如ETH、MATIC，也可以是ERC20代币比如USDC、DAI。代码会根据currency的类型自动选择正确的查询方式。

**第四，enabled控制查询**。如果enabled设为false，那hook就不会发起任何查询。这在你需要条件性地禁用查询的时候很有用。

**第五，查询结果会被缓存**。React Query会自动缓存查询结果。这意味着如果你在多个地方用同一个参数查询同一个余额，不会真的发起多次RPC请求。

### 状态管理要清楚

查询状态有四种。第一种isLoading表示首次加载中，这时候UI可以显示一个骨架屏。第二种isFetching表示正在后台刷新数据，这时候一般不需要显示loading动画。第三种isError表示查询出错了，可能是网络问题或者RPC节点的问题。第四种data就是实际的余额数据，是Amount类型。

刷新数据的触发方式有四种。第一种是定时刷新，每隔10秒自动查一次。第二种是窗口获焦刷新，当用户切换回这个页面时会自动刷新。第三种是手动调用refetch函数触发刷新。第四种是 useWatchByInterval触发刷新。

### 常见错误要避开

**第一个坑：nativeBalance序列化问题**。nativeBalance是一个复杂的对象，如果直接放进queryKey里，React Query可能无法正确识别这是同一个查询。一定要先序列化，或者用sushi库提供的工具来处理。

**第二个坑：代币数量限制**。代码里限制了查询的代币数量不能超过100个。这是为了防止一次性查询太多代币导致RPC的rate limit。如果你的应用确实需要查询很多代币，需要分批查询。

**第三个坑：后台刷新策略**。refetchIntervalInBackground设置为false意味着当浏览器标签页在后台时，不会自动刷新。这是为了节省资源，但用户切回来的时候看到的可能是旧数据。代码会用refetchOnWindowFocus来处理用户切回时的刷新。

**第四个坑：null和undefined的区别**。返回的data可能是null，表示查询没查到数据或者查询出错了。也可能是undefined，表示查询还没执行或者没有启用查询。UI要根据这两种情况分别处理。

**第五个坑：切换链的问题**。用户切换链之后，之前查到的旧链上的数据可能还残留在缓存里。最好在切换链的时候清一下缓存，或者用chainId作为queryKey的一部分来区分不同链的数据。

### 提示词模板

```markdown
帮我创建一个查询钱包代币余额的hook。

需求：
- 支持查询原生币和ERC20代币的余额
- 定时自动刷新，不用用户手动刷新
- 返回的数据要方便显示和比较大小
- 窗口切回来的时候要自动刷新

技术栈：
- React + TypeScript
- wagmi v2 + viem
- @tanstack/react-query v5
- sushi库的Amount类型
- useWatchByInterval做定时刷新

输入：
- chainId：链ID
- account：钱包地址
- currency：要查询的代币
- enabled：是否启用查询

输出：
- data：余额数据，Amount类型
- isLoading：是否首次加载
- isError：是否出错
- refetch：手动刷新函数

实现要点：
1. 原生币和ERC20用不同的方式查询
2. 原生币用zeroAddress作为标识
3. ERC20用代币合约地址作为标识
4. 定时刷新间隔10秒比较合适
5. 窗口获焦时要刷新数据
6. 返回的Amount类型可以直接用

注意：
- 参数要校验有效性
- 不要一次查太多代币
- 后台标签页不要刷新
- 切换链时要清缓存
```

### 实际避坑指南

第一个，序列化的问题。nativeBalance不能直接放进queryKey，必须先序列化。sushi库里应该有提供序列化工具，直接用就行。

第二个，分批查询。如果确实需要查询超过100个代币的余额，要分批次查询。比如第一批查前100个，第二批查后面的。每批之间可以加个小延迟，防止触发rate limit。

第三个，null和undefined的区分。UI代码要区分这两种情况：null表示查过了但没查到或出错了，undefined表示还没查过。显示给用户的时候可以分别处理，比如null显示"查询失败"，undefined显示"加载中"。

第四个，切换链的处理。用户切换链后，之前查到的数据可能还有残留。可以用chainId作为queryKey的一部分，这样切换链的时候queryKey就变了，React Query会自动去查新链的数据。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useBalanceWeb3` 用于查询钱包的代币余额。基本用法如下：

```typescript
import { useBalanceWeb3 } from '@sushiswap/wag'

function WalletBalance({ account, chainId, currency }) {
  const { data: balance, isLoading, refetch } = useBalanceWeb3({
    chainId,
    currency,
    account,
    enabled: Boolean(account && currency),
  })

  if (isLoading) return <div>加载中...</div>
  if (!balance) return <div>无余额</div>

  return (
    <div>
      <span>余额: {balance.toSignificant(6)} {currency.symbol}</span>
      <button onClick={() => refetch()}>刷新</button>
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景一：显示钱包余额

```typescript
function BalanceDisplay({ account, chainId }) {
  // ETH 余额
  const ethBalance = useBalanceWeb3({
    chainId,
    currency: EvmNative.ETHEREUM,
    account,
  })

  // USDC 余额
  const usdcBalance = useBalanceWeb3({
    chainId,
    currency: USDC,
    account,
  })

  return (
    <div>
      <div>ETH: {ethBalance.data?.toSignificant(6)}</div>
      <div>USDC: {usdcBalance.data?.toSignificant(6)}</div>
    </div>
  )
}
```

#### 场景二：余额不足检查

```typescript
function SwapButton({ amount, currency, account }) {
  const { data: balance } = useBalanceWeb3({
    chainId,
    currency,
    account,
  })

  const insufficientBalance = balance && balance.lt(amount)

  return (
    <button disabled={insufficientBalance}>
      {insufficientBalance ? '余额不足' : '交换'}
    </button>
  )
}
```

#### 场景三：组合 useWatchByInterval

```typescript
function RealTimeBalance({ account, chainId, currency }) {
  const { data: balance, isLoading } = useBalanceWeb3({
    chainId,
    currency,
    account,
  })

  // 余额自动每 10 秒刷新，不需要额外处理
  // useWatchByInterval 已内置

  return <div>{balance?.toSignificant(6)} {currency.symbol}</div>
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **使用 Amount 类型进行比较**
   ```typescript
   // 正确
   if (balance.gt(amount)) { ... }

   // 错误：不要直接用 bigint
   if (balance.amount > amount.amount) { ... }
   ```

2. **处理加载状态**
   ```typescript
   if (isLoading) return <Skeleton />
   if (!data) return <div>查询失败</div>
   ```

3. **格式化显示**
   ```typescript
   // 使用 toSignificant 或 toFixed 格式化
   balance.toSignificant(6) // 保留 6 位有效数字
   balance.toFixed(2)       // 保留 2 位小数
   ```

#### ❌ Don'ts

1. **不要忽略 enabled**
   ```typescript
   // 错误
   enabled: true

   // 正确
   enabled: Boolean(account && currency && chainId)
   ```

2. **不要在 render 中发起过多查询**
   ```typescript
   // 错误：每个代币都独立查询
   tokens.map(token => useBalanceWeb3({ token, account }))

   // 正确：使用 useBalancesWeb3 批量查询
   useBalancesWeb3({ currencies: tokens, account })
   ```

3. **不要假设余额立即更新**
   ```typescript
   // 错误：交易后立即检查余额
   await swap()
   const balance = data // 可能还是旧值

   // 正确：等待刷新或手动 refetch
   await swap()
   await refetch()
   ```

4. **不要忘记原生币类型**
   ```typescript
   // 错误
   currency: 'ETH' // 字符串不是正确的类型

   // 正确
   currency: EvmNative.ETHEREUM
   ```
