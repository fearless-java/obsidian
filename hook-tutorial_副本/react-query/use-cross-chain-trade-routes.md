> 源代码路径: `apps/web/src/lib/hooks/react-query/cross-chain-trade/useCrossChainTradeRoutes.ts`

# useCrossChainTradeRoutes Hook Tutorial

## 1. 大白话讲讲这个hook的作用

`useCrossChainTradeRoutes` 是用来查询跨链交易路由的 Hook。当用户想要把代币从一条链 swap 到另一条链（如从 Ethereum 跨到 Arbitrum）时，这个 Hook 会查询所有可用的跨链路径、每个路径的价格、以及具体的 step 信息。

简单来说：**就是帮你查"从 A 链的 X 代币换到 B 链的 Y 代币，有哪几条路可以走、每条路要花多少钱"。**

---

## 2. 为什么需要封装该hook

### 原始问题
1. **参数构造复杂**：需要把 `Amount<Currency>`、slippage、token 地址等信息转换成 URL 查询参数
2. **原生代币地址特殊处理**：原生代币（如 ETH）需要用 `getNativeAddress()` *(一个函数，用于获取某条链上原生代币的合约地址，如 ETH 的 Wrapped 代币地址)* 获取合约地址
3. **响应数据处理**：API 返回的 routes 需要直接返回给调用方
4. **定时刷新**：跨链价格变动较频繁，需要 20 秒刷新

### 封装带来的好处
1. **参数自动转换**：原生代币、精度处理等都由 Hook 内部处理
2. **返回直接的路由数据**：调用方拿到 routes 后可以直接展示
3. **20 秒自动刷新**：保持跨链价格相对实时
4. **完整的类型支持**：泛型保证 chainId0 和 chainId1 的类型安全

---

## 3. 执行逻辑和数据流向

### 输入 (Params)
```typescript
{
  fromAmount?: Amount<CurrencyFor<TChainId0>>   // 源链代币数量
  toToken?: CurrencyFor<TChainId1>             // 目标链代币
  fromAddress?: AddressFor<TChainId0>         // 源链地址（可选）
  toAddress?: AddressFor<TChainId1>           // 目标链地址（可选）
  slippage: Percent                            // 滑点
  order?: 'CHEAPEST' | 'FASTEST'               // 排序方式
}
```

### 输出 (Return)
```typescript
{
  data: CrossChainRoute[]  // 路由数组
  isLoading: boolean
  isError: boolean
  // ...
}
```

### 执行流程

```
1. useCrossChainTradeRoutes(params)
       |
       v
2. 检查 toToken && fromAmount?.gt(0n)
       |
       v
3. 构造 URL 查询参数:
   - fromChainId: fromAmount.currency.chainId
   - toChainId: toToken.chainId
   - fromTokenAddress: 原生代币用 getNativeAddress()，否则用 token.address
   - toTokenAddress: 同上
   - fromAmount: fromAmount.amount.toString()
   - slippage: slippage.toString({ fixed: 2 }) / 100
   - fromAddress/toAddress: 可选参数
   - order: 可选参数
       |
       v
4. POST /api/cross-chain/routes
       |
       v
5. 获取返回的 routes 数组
       |
       v
6. 返回 routes
```

---

## 4. 怎么用 AI 写这个 Hook

### 写代码时要注意什么

1. **泛型要支持链类型**：用 `TChainId0` 和 `TChainId1` 这样的泛型参数来保证链 ID 的类型安全。

2. **原生代币要特殊处理**：ETH 这种原生代币不是 ERC20 代币，要用 `getNativeAddress()` 获取它的包装代币地址才能查。

3. **slippage 要转换**：Percent 类型要转成小数，比如 0.5% 要变成 0.005。

4. **可选参数要有就加，没有就不加**：URL 里面只加那些实际有值的可选参数，别加一堆空的。

### 有什么限制条件

1. **fromAmount 和 toToken 必须有**：这两个参数是构造查询的基础，缺一个都没法发请求。

2. **依赖内部 API**：调的是 `/api/cross-chain/routes`，不是第三方接口。

3. **20 秒刷新一次**：跨链的价格变化比较快，20 秒刷新一次比较合适。

4. **依赖 Step 类型定义**：需要 `~evm/api/cross-chain/schemas` 里面的类型定义。

### 状态怎么管理

| 状态 | 怎么管 | 什么意思 |
|------|--------|---------|
| 路由数据 | React Query 缓存 | 按参数缓存 |
| 定时刷新 | refetchInterval: 20s | 20秒自动刷新 |
| 启用开关 | toToken && fromAmount?.gt(0n) | fromAmount 大于 0 才查 |

---

### 完整AI提示词模板

```
你是一个 React Query + 跨链交易专家。请为以下场景编写 Hook:

【场景描述】
需要查询跨链交易路由。当用户要从 A 链 swap 代币到 B 链时，
需要查询所有可用的跨链路径和报价。

【技术要求】
1. 使用 @tanstack/react-query useQuery
2. GET /api/cross-chain/routes
3. URL 参数构造:
   - fromChainId: fromAmount.currency.chainId
   - toChainId: toToken.chainId
   - fromTokenAddress: 原生代币用 getNativeAddress(chainId)，否则用 token.address
   - toTokenAddress: 同上
   - fromAmount: fromAmount.amount.toString()
   - slippage: parseFloat(slippage.toString({ fixed: 2 })) / 100

【参数】
interface UseCrossChainTradeRoutesParams {
  fromAmount?: Amount<Currency>
  toToken?: Currency
  fromAddress?: Address
  toAddress?: Address
  slippage: Percent
  order?: 'CHEAPEST' | 'FASTEST'
}

【泛型约束】
TChainId0 extends XSwapSupportedChainId
TChainId1 extends XSwapSupportedChainId

【原生代币处理】
if (token.type === 'native') {
  address = getNativeAddress(token.chainId)
} else {
  address = token.address
}

【缓存配置】
- refetchInterval: 20000 (20秒)
- enabled: Boolean(params.toToken && params.fromAmount?.gt(0n))

【可选参数】
只有存在的参数才添加到 URL:
params.fromAddress && url.searchParams.set('fromAddress', params.fromAddress)

【最佳实践】
- 泛型保证链 ID 类型安全
- 原生代币用 getNativeAddress
- 可选参数条件性添加
- 数量为 0 时不查询

请输出完整代码。
```

---

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useCrossChainTradeRoutes } from '@sushiswap/react-query'
import { useToken } from '@sushiswap/token'
import { Amount, Percent } from 'sushi'

function CrossChainSwap({ fromToken, toToken, amount, slippage }) {
  const fromAmount = Amount.fromRawAmount(fromToken, amount)
  const { data: routes, isLoading } = useCrossChainTradeRoutes({
    fromAmount,
    toToken,
    slippage: new Percent(0.5, 100), // 0.5%
  })

  if (isLoading) return <RoutesSkeleton />
  if (!routes?.length) return <NoRoutesFound />

  return (
    <div>
      <h3>Available Routes ({routes.length})</h3>
      {routes.map((route, index) => (
        <RouteCard key={index} route={route} />
      ))}
    </div>
  )
}
```

### 常见使用场景

**场景1：跨链交易页面**
```tsx
function CrossChainSwapPage() {
  const [fromToken, setFromToken] = useState(null)
  const [toToken, setToToken] = useState(null)
  const [amount, setAmount] = useState('')

  const fromAmount = fromToken && amount
    ? Amount.fromRawAmount(fromToken, parseEther(amount))
    : undefined

  const { data: routes, isLoading, isError } = useCrossChainTradeRoutes({
    fromAmount,
    toToken,
    slippage: userSlippageTolerance,
    order: 'CHEAPEST',
  })

  return (
    <div className="swap-container">
      <TokenInput
        label="From"
        token={fromToken}
        amount={amount}
        onAmountChange={setAmount}
        onTokenChange={setFromToken}
      />

      <SwitchTokensButton onClick={switchTokens} />

      <TokenInput
        label="To"
        token={toToken}
        disabled
        displayAmount={routes?.[0]?.estimatedOutput}
      />

      <RouteList routes={routes} isLoading={isLoading} />

      <SwapButton disabled={!routes?.length || isLoading} />
    </div>
  )
}
```

**场景2：选择最佳路由**
```tsx
function BestRouteSelector({ routes, onSelect }) {
  const [selectedIndex, setSelectedIndex] = useState(0)

  const cheapestRoute = routes?.[0]
  const fastestRoute = routes?.find((r) => r.tags?.includes('FASTEST'))

  const displayRoute = selectedIndex === 0 ? cheapestRoute : fastestRoute

  return (
    <div>
      <RouteTabs>
        <Tab
          selected={selectedIndex === 0}
          onClick={() => setSelectedIndex(0)}
          label="Cheapest"
          value={cheapestRoute?.estimatedOutput}
        />
        <Tab
          selected={selectedIndex === 1}
          onClick={() => setSelectedIndex(1)}
          label="Fastest"
          value={fastestRoute?.estimatedOutput}
        />
      </RouteTabs>

      {displayRoute && (
        <RouteDetails route={displayRoute} onSelect={() => onSelect(displayRoute)} />
      )}
    </div>
  )
}
```

### Dos and Don'ts

**Do（推荐做法）：**
- ✅ 使用 `Amount.fromRawAmount()` 创建 Amount 对象
- ✅ 使用 `new Percent(value, denominator)` 创建滑点
- ✅ 检查 `routes?.length` 确认有可用路由
- ✅ 监听 `isLoading` 显示加载状态

**Don't（避免做法）：**
- ❌ 不要在 fromAmount 为 0 或 undefined 时调用
- ❌ 不要假设一定返回路由，应该检查空数组
- ❌ 不要忽略滑点参数，应该传递给 hook
- ❌ 不要使用原生数字创建 Amount，应该用 Amount.fromRawAmount

### 注意事项

1. **fromAmount 必须大于 0**：如果 fromAmount 是 0 或 undefined，查询不会执行

2. **原生代币自动处理**：hook 内部会处理原生代币（如 ETH）到 Wrapped 代币地址的转换

3. **20秒自动刷新**：跨链价格变动频繁，不需要手动刷新

4. **泛型链类型**：TChainId0 和 TChainId1 是泛型参数，保证类型安全

5. **返回路由数组**：每个路由包含不同的跨链方案，用户可以切换选择
