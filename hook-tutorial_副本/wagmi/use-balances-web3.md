> 源代码路径: `apps/web/src/lib/wagmi/hooks/balances/useBalancesWeb3.ts`

# useBalancesWeb3

## 大白话讲讲这个hook的作用

`useBalancesWeb3` 是一个批量查询多个代币余额的 Hook。它是 `useBalanceWeb3` 的多token版本，一次可以查询任意多个代币的余额。

大白话：就是一次查询"我的钱包里有哪些币，分别有多少"。适合需要展示用户完整资产列表的场景。

它使用 `readContracts` 批量调用来提高效率，同时自动处理原生币余额和 ERC20 代币余额的查询。

## 讲讲为什么需要封装该hook

1. **批量查询效率**：使用 `readContracts` 批量调用，比逐个查询更高效（节省 RPC 调用次数）。

2. **统一处理**：原生币和 ERC20 代币的查询逻辑统一封装。

3. **定时刷新**：封装了 `useWatchByInterval` 定时刷新逻辑。

4. **React Query 缓存**：查询结果会被缓存，方便组件复用。

5. **性能保护**：限制最多 100 个代币，防止过度调用导致 rate limit。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseBalanceParams {
  chainId: EvmChainId | undefined      // 链 ID
  currencies: (EvmCurrency | undefined)[]  // 代币数组
  account: Address | undefined          // 钱包地址
  enabled?: boolean                    // 是否启用
}
```

### 输出 (Outputs)

```typescript
{
  data: Record<string, Amount<EvmCurrency>> | null,  // 余额映射
  isLoading: boolean,
  isError: boolean,
  refetch: () => void,
}
```

### 执行逻辑详解

#### 1. 查询原生币余额

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

#### 3. 批量查询 ERC20 余额

```typescript
const queryFnUseBalances = async ({ chainId, currencies, account, nativeBalance, config }) => {
  // 过滤有效代币
  const [validatedTokens, validatedTokenAddresses] = currencies.reduce<
    [EvmToken[], Address[]]
  >(
    (acc, currencies) => {
      if (chainId && currencies && isAddress(currencies.wrap().address)) {
        acc[0].push(currencies.wrap())
        acc[1].push(currencies.wrap().address)
      }
      return acc
    },
    [[], []],
  )

  // 批量查询 ERC20 余额
  const data = await readContracts(config, {
    contracts: validatedTokenAddresses.map(
      (token) => ({
        chainId,
        address: token,
        abi: erc20Abi,
        functionName: 'balanceOf',
        args: [account],
      }) as const,
    ),
  })

  // 转换结果
  const _data = data.reduce<Record<string, Amount<EvmCurrency>>>(
    (acc, _cur, i) => {
      const amount = data[i].result
      if (typeof amount === 'bigint') {
        acc[validatedTokens[i].address] = new Amount(validatedTokens[i], amount)
      }
      return acc
    },
    {},
  )

  // 添加原生币余额
  _data[zeroAddress] = new Amount(EvmNative.fromChainId(chainId), nativeBalance.value)

  return _data
}
```

#### 4. 性能保护检查

```typescript
useEffect(() => {
  if (currencies && currencies.length > 100) {
    throw new Error(
      'useBalancesWeb3: currencies length > 100, this will hurt performance and cause rate limits',
    )
  }
}, [currencies])
```

### 数据流向图

```
输入: currencies: [token0, token1, ..., native]
         ↓
    ┌────────────────────────────────────┐
    │  useBalance 查询原生币余额          │
    └────────────────────────────────────┘
         ↓
    ┌────────────────────────────────────┐
    │  readContracts 批量查询 ERC20       │
    │  balanceOf for each token          │
    └────────────────────────────────────┘
         ↓
    reduce 转换成 Record<address, Amount>
         ↓
    添加 native 到 zeroAddress key
         ↓
    返回完整余额映射
         ↑
    useWatchByInterval 定时刷新
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：选择批量还是单个要想清楚**。如果你只需要查一个代币的余额，用useBalanceWeb3就够了，没必要用批量查询。批量查询适合需要一次性展示用户所有资产或者多个代币余额的场景。如果只查两三个代币，单独查询可能反而更快。

**第二点：缓存key的设计**。查询的key一定要包含所有会影响结果的变量。比如nativeBalance是一个对象，必须序列化后才能放进key里，否则React Query的缓存机制会出问题。这个序列化过程是必须的，不能省略。

**第三点：代币数量有限制**。代码里限制了最多一次查100个代币。这不是随便设的，是因为以太坊的批量调用有gas限制，太多的话交易数据会超限。如果你的应用确实需要查超过100个代币，要分批次来查。

**第四点：地址要校验**。传入的代币地址必须是用isAddress验证过的合法以太坊地址。如果地址格式不对，整个查询都会失败。所以最好在把代币传给hook之前先校验一下。

**第五点：返回结构很方便**。返回的是一个Record结构，key是代币地址，value是Amount类型。这样的结构在渲染资产列表的时候特别方便，可以用地址直接找到对应的余额。

### 约束条件要记住

**第一，数组不能超过100**。currencies数组最多只能有100个元素。如果超过100个，代码会主动抛出一个错误，防止性能问题。

**第二，所有代币必须在同一链上**。这是批量查询的限制。所有要查询的代币必须在传入的chainId对应的链上。如果有跨链的需求，需要分别查询。

**第三，account要合法**。必须是有效的以太坊地址。地址格式不对的话，balanceOf调用会失败。

**第四，原生币用zeroAddress标识**。在返回的Record里，原生币的余额是用zeroAddress作为key的，而不是用代币符号或者其他什么。zeroAddress是viem库提供的一个常量。

**第五，批量调用有上限**。虽然代码允许查100个代币，但实际能查到的数量可能还取决于链的状态和RPC节点的限制。

### 状态管理要清楚

查询状态有四种。isLoading表示首次加载中，isFetching表示正在后台刷新，isError表示出错了，data就是实际的余额数据，是一个Record结构。

刷新数据有四种方式。定时刷新每10秒一次，useWatchByInterval会监听时间间隔来刷新，refetchOnWindowFocus会在窗口获焦时刷新，手动调用refetch可以随时触发一次查询。

### 常见错误要避开

**第一个坑：batch调用有上限**。虽然代码允许最多100个代币，但以太坊的批量调用实际上有gas和数据大小的限制。如果RPC节点对批量调用有限制，可能查不到100个那么多。

**第二个坑：不是真正的multicall**。readContracts虽然看起来是批量调用，但实际上还是逐个执行的，不是真正的以太坊multicall。不过这不影响使用，只是性能上没有真正的批量调用那么高效。

**第三个坑：少量代币用批量反而慢**。如果你只需要查两三个代币，用批量查询不一定比单独查询快。因为批量查询内部要做数据转换和组装，而单独查询更直接。

**第四个坑：缓存一致性**。如果多个组件都在用这个hook查询同一个账户的余额，React Query会确保他们拿到的是同一份缓存数据，不会出现数据不一致的情况。

**第五个坑：空数组会被跳过**。如果传入的currencies是一个空数组，查询会被禁用，不会发起任何RPC请求。这其实是合理的行为，但如果你期望空数组也返回一个空结果，可能会困惑。

### 提示词模板

```markdown
帮我创建一个批量查询代币余额的hook。

需求：
- 一次查询多个代币的余额
- 包括原生币和ERC20代币
- 定时自动刷新，不用手动刷新
- 返回的数据结构要方便查找

技术栈：
- React + TypeScript
- wagmi v2 + viem
- @tanstack/react-query v5
- sushi库
- useWatchByInterval

输入：
- chainId：链ID
- account：钱包地址
- currencies：代币数组，最多100个
- enabled：是否启用

输出：
- data：Record结构，key是地址，value是Amount
- isLoading：是否首次加载
- isFetching：是否正在刷新
- isError：是否出错

实现要点：
1. 用readContracts批量查询ERC20余额
2. 原生币单独用useBalance查询
3. 把所有余额组装成Record返回
4. 原生币用zeroAddress作为key
5. 定时刷新间隔10秒
6. 窗口获焦时刷新

注意：
- 代币数量不能超过100个
- 所有代币必须在同一链上
- 地址要校验有效性
- 空数组不会发起查询
```

### 实际避坑指南

第一个，分批查询超过100个代币。如果确实需要查很多代币，要分成多个批次。比如第一批查0-99，第二批查100-199。每批之间可以加个延迟，防止RPC过载。

第二个，少量代币不用批量。如果你只需要查两三个代币，直接用useBalanceWeb3会更简单直接，不用走批量查询的逻辑。

第三个，空数组处理。如果传入空数组，查询会被禁用。这其实是好的行为，因为没必要查一个空列表。但如果你的UI需要显示"没有代币"而不是"加载中"，要注意这个区别。

第四个，切换链的问题。用户切换链后，之前的数据可能还残留在缓存里。用chainId作为queryKey的一部分可以避免这个问题，切换链后queryKey就变了。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useBalancesWeb3` 用于批量查询多个代币余额。基本用法如下：

```typescript
import { useBalancesWeb3, useBalanceWeb3 } from '@sushiswap/wag'
import { zeroAddress } from 'viem'

function WalletAssets({ account, chainId, currencies }) {
  const { data: balances, isLoading, refetch } = useBalancesWeb3({
    chainId,
    currencies, // 代币数组
    account,
    enabled: Boolean(account && currencies.length > 0),
  })

  if (isLoading) return <div>加载中...</div>

  return (
    <div>
      {currencies.map((currency) => {
        const address = currency.type === 'native' ? zeroAddress : currency.wrap().address
        const balance = balances?.[address]
        return (
          <div key={address}>
            {currency.symbol}: {balance?.toSignificant(6) ?? '0'}
          </div>
        )
      })}
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景一：资产列表展示

```typescript
function AssetList({ account, chainId }) {
  const allTokens = [ETH, USDC, USDT, DAI, WBTC]

  const { data: balances } = useBalancesWeb3({
    chainId,
    currencies: allTokens,
    account,
  })

  const totalValue = allTokens.reduce((sum, token) => {
    const addr = token.type === 'native' ? zeroAddress : token.address
    const balance = balances?.[addr]
    const value = balance ? balance.toUsdValue(priceMap[addr]) : 0
    return sum + value
  }, 0)

  return (
    <div>
      <div>总价值: ${totalValue.toFixed(2)}</div>
      {allTokens.map(token => (
        <AssetRow key={token.address} token={token} balance={balances?.[token.address]} />
      ))}
    </div>
  )
}
```

#### 场景二：批量余额检查

```typescript
function CheckAllowances({ account, spender, tokens }) {
  // 查询所有代币余额
  const { data: balances } = useBalancesWeb3({
    chainId,
    currencies: tokens,
    account,
  })

  // 检查哪些余额足够
  const sufficientTokens = tokens.filter(token => {
    const addr = token.type === 'native' ? zeroAddress : token.address
    const balance = balances?.[addr]
    return balance && balance.gt(0)
  })

  return (
    <div>
      <div>有余额的代币: {sufficientTokens.length}</div>
    </div>
  )
}
```

#### 场景三：定时刷新监控

```typescript
function MonitorBalances({ account, chainId }) {
  const { data: balances, isFetching, refetch } = useBalancesWeb3({
    chainId,
    currencies: [ETH, USDC, WBTC],
    account,
  })

  // 余额会自动每 10 秒刷新
  // isFetching 为 true 时表示正在后台刷新

  return (
    <div>
      <div>ETH: {balances?.[zeroAddress]?.toSignificant(6)}</div>
      <div>USDC: {balances?.[USDC.address]?.toSignificant(6)}</div>
      {isFetching && <span>刷新中...</span>}
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **使用 zeroAddress 获取原生币余额**
   ```typescript
   import { zeroAddress } from 'viem'

   const ethBalance = balances?.[zeroAddress]
   const erc20Balance = balances?.[tokenAddress]
   ```

2. **检查 balances 是否存在**
   ```typescript
   if (!balances) return <div>加载中...</div>

   // 安全访问
   const balance = balances?.[address]
   ```

3. **分批处理大量代币**
   ```typescript
   // 超过 100 个时分批查询
   const batch1 = tokens.slice(0, 100)
   const batch2 = tokens.slice(100, 200)

   const result1 = useBalancesWeb3({ currencies: batch1, ... })
   const result2 = useBalancesWeb3({ currencies: batch2, ... })
   ```

#### ❌ Don'ts

1. **不要查询太多代币**
   ```typescript
   // 错误：会触发 rate limit
   useBalancesWeb3({ currencies: allTokens }) // 500 个代币

   // 正确：限制在 100 个以内
   ```

2. **不要在每次 render 时传递新数组**
   ```typescript
   // 错误：每次 render 创建新数组
   useBalancesWeb3({ currencies: [ETH, USDC, ...] })

   // 正确：使用 useMemo 缓存数组
   const currencies = useMemo(() => [ETH, USDC, ...], [])
   ```

3. **不要忽略 zeroAddress**
   ```typescript
   // 错误：原生币查询
   balances?.['ETH'] // 不存在这个 key

   // 正确：使用 zeroAddress
   balances?.[zeroAddress]
   ```

4. **不要忘记 enabled 检查**
   ```typescript
   // 错误
   enabled: true

   // 正确
   enabled: Boolean(account && currencies.length > 0)
   ```
