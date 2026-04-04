> 源代码路径: `apps/web/src/lib/hooks/react-query/cross-chain-trade/useCrossChainTradeStep.ts`

# useCrossChainTradeStep Hook Tutorial

## 1. 大白话讲讲这个hook的作用

`useCrossChainTradeStep` 是用来查询跨链交易中某个具体 step 的详细报价和执行信息的 Hook。与 `useCrossChainTradeRoutes` 查询所有路由不同，这个 Hook 只查询一个具体 step（如 "通过 LiFi 跨链"）的详细信息，包括：tokenIn、tokenOut、amountIn、amountOut、priceImpact 等。

简单来说：**`useCrossChainTradeRoutes` 查"有哪些路"，`useCrossChainTradeStep` 查"这条路具体怎么走、花多少钱"。**

---

## 2. 为什么需要封装该hook

### 原始问题
1. **POST 请求体复杂**：step 数据需要 stringify 后放在请求体中
2. **响应数据需要大量转换**：API 返回的原始 token 数据需要转换为 Currency *(sushi库中的代币类型，支持原生代币和ERC20代币)* 对象
3. **原生代币特殊处理**：需要判断 token 是否是原生代币，然后调用 `nativeFromChainId` *(一个函数，用于从链ID获取该链的原生代币Currency对象)* 或 `newToken` *(一个函数，用于创建ERC20代币的Currency对象)*
4. **金融计算复杂**：需要计算 `fromAmountUSD`、`toAmountUSD`、`priceImpact` 等

### 封装带来的好处
1. **返回可直接使用的 Currency 对象**：tokenIn、tokenOut 都是 Currency 类型
2. **Amount 类型封装**：amountIn、amountOut、amountOutMin 都是 Amount *(sushi库中的代币金额类型)* 类型
3. **价格影响计算**：自动计算 priceImpact
4. **10 秒自动刷新**：保持报价实时

---

## 3. 执行逻辑和数据流向

### 输入 (Params)
```typescript
{
  step: Step<TChainId0, TChainId1, 'lifi'> | undefined  // 具体的 step 对象
  enabled?: boolean = true
}
```

### 输出 (Return)
```typescript
{
  ...CrossChainStepResponse  // API 返回的原始数据
  tokenIn: CurrencyFor<TChainId0>      // 输入代币
  tokenOut: CurrencyFor<TChainId1>    // 输出代币
  amountIn: Amount<CurrencyFor<TChainId0>>   // 输入数量
  amountOut: Amount<CurrencyFor<TChainId1>>  // 输出数量
  amountOutMin: Amount<CurrencyFor<TChainId1>> // 最小输出
  priceImpact: Percent                 // 价格影响
}
```

### 执行流程

```
1. useCrossChainTradeStep({ step, enabled })
       |
       v
2. 检查 enabled && step
       |
       v
3. 构造 POST 请求:
   POST /api/cross-chain/step
   Body: stringify(step)
       |
       v
4. 获取 CrossChainStepResponse
       |
       v
5. 转换 token 对象:
   a) 如果是原生代币 -> nativeFromChainId(chainId)
   b) 如果是 ERC20 -> newToken({ ...token })
       |
       v
6. 转换金额:
   amountIn = new Amount(tokenIn, parsedStep.action.fromAmount)
   amountOut = new Amount(tokenOut, parsedStep.estimate.toAmount)
   amountOutMin = new Amount(tokenOut, parsedStep.estimate.toAmountMin)
       |
       v
7. 计算 USD 和 priceImpact:
   fromAmountUSD = fromToken.priceUSD * amountIn / 10**decimals
   toAmountUSD = toToken.priceUSD * amountOut / 10**decimals
   priceImpact = new Percent({ numerator: (fromAmountUSD/toAmountUSD - 1) * 10000, denominator: 10000 })
       |
       v
8. 返回完整数据
```

---

## 4. 怎么用 AI 写这个 Hook

### 写代码时要注意什么

1. **step 来自 Routes 查询结果**：这个 step 是用户从路由查询结果里选中的，不是随便传一个就能用。

2. **原生代币转换要判断**：先用 `getNativeAddress` 判断是不是原生代币，再用 `nativeFromChainId` 获取正确的 Currency 对象。

3. **ERC20 代币用 newToken 创建**：普通的 ERC20 代币要用 `newToken` 来构造 Currency 对象。

4. **priceImpact 要按 USD 价值算**：价格影响是基于美元价值计算的，不是看代币数量。

5. **queryKey 要序列化**：因为 step 是对象，直接用会出问题，要用 stringify 转成字符串才能当 key 用。

### 有什么限制条件

1. **step 不能为空**：enabled 检查里有 `!!step`，step 不存在就不发请求。

2. **依赖 LiFi**：step 的类型是 `Step<..., 'lifi'>`，只有 LiFi 的 step 才能用。

3. **是 POST 请求**：不是 GET，需要构造请求体。

4. **10 秒刷新一次**：比路由查询刷新更频繁，因为具体 step 的报价变化更快。

### 状态怎么管理

| 状态 | 怎么管 | 什么意思 |
|------|--------|---------|
| Step 数据 | React Query 缓存 | 按 step 缓存 |
| 定时刷新 | refetchInterval: 10s | 10秒自动刷新 |
| 启用开关 | enabled && step | step 必须存在 |
| queryKey | stringify | 对象要序列化才能当 key |

---

### 完整AI提示词模板

```
你是一个 React Query + 跨链交易专家。请为以下场景编写 Hook:

【场景描述】
需要查询跨链交易中某个具体 step 的详细报价。
用户选择了某条路由后，需要查询该路由下每个 step 的具体执行信息。

【技术要求】
1. 使用 @tanstack/react-query useQuery
2. POST /api/cross-chain/step
3. Body: stringify(step) (step 是 LiFi 的 step 对象)

【Token 转换逻辑】
fromToken = getNativeAddress(parsedStep.action.fromToken.chainId) === parsedStep.action.fromToken.address
  ? nativeFromChainId(parsedStep.action.fromToken.chainId)
  : newToken(parsedStep.action.fromToken)

toToken = 同上逻辑

【金额转换】
amountIn = new Amount(tokenIn, parsedStep.action.fromAmount)
amountOut = new Amount(tokenOut, parsedStep.estimate.toAmount)
amountOutMin = new Amount(tokenOut, parsedStep.estimate.toAmountMin)

【USD 和 PriceImpact 计算】
fromAmountUSD = Number(fromToken.priceUSD) * Number(amountIn.amount) / 10**tokenIn.decimals
toAmountUSD = Number(toToken.priceUSD) * Number(amountOut.amount) / 10**tokenOut.decimals
priceImpact = new Percent({ numerator: Math.floor((fromAmountUSD / toAmountUSD - 1) * 10000), denominator: 10000 })

【参数】
{
  step: Step<TChainId0, TChainId1, 'lifi'> | undefined
  enabled?: boolean
}

【缓存配置】
- refetchInterval: 10000 (10秒)
- enabled: Boolean(enabled && step)
- queryKeyHashFn: stringify (因为 queryKey 包含对象)

【最佳实践】
- 原生代币用 nativeFromChainId
- ERC20 用 newToken
- priceImpact 基于 USD 计算
- step 为空时不发起请求

请输出完整代码。
```

---

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useCrossChainTradeStep } from '@sushiswap/react-query'
import { useCrossChainTradeRoutes } from '@sushiswap/react-query'

function CrossChainStepDetails({ route }) {
  const [selectedStepIndex, setSelectedStepIndex] = useState(0)
  const selectedStep = route.steps[selectedStepIndex]

  const { data: stepDetails, isLoading } = useCrossChainTradeStep({
    step: selectedStep,
    enabled: !!selectedStep,
  })

  if (isLoading) return <StepDetailsSkeleton />
  if (!stepDetails) return null

  return (
    <div>
      <div className="step-header">
        <span>Step {selectedStepIndex + 1}: {stepDetails.tokenIn.symbol} → {stepDetails.tokenOut.symbol}</span>
      </div>

      <div className="step-details">
        <DetailRow label="You pay" value={stepDetails.amountIn.toSignificant(6)} />
        <DetailRow label="You receive" value={stepDetails.amountOut.toSignificant(6)} />
        <DetailRow label="Minimum received" value={stepDetails.amountOutMin.toSignificant(6)} />
        <DetailRow label="Price impact" value={`${stepDetails.priceImpact.toFixed(2)}%`} />
      </div>
    </div>
  )
}
```

### 常见使用场景

**场景1：跨链交易确认页面**
```tsx
function CrossChainConfirm({ route, selectedStepIndex }) {
  const step = route.steps[selectedStepIndex]

  const { data: details, isLoading } = useCrossChainTradeStep({
    step,
    enabled: !!step,
  })

  return (
    <ConfirmBox>
      <div className="flex justify-between">
        <span>Input</span>
        <span>{details?.amountIn.toSignificant(6)} {details?.tokenIn.symbol}</span>
      </div>

      <ArrowDownIcon />

      <div className="flex justify-between">
        <span>Output (estimated)</span>
        <span>{details?.amountOut.toSignificant(6)} {details?.tokenOut.symbol}</span>
      </div>

      <div className="price-impact">
        Price Impact: {details?.priceImpact.toFixed(2)}%
      </div>

      <div className="min-output">
        Minimum output: {details?.amountOutMin.toSignificant(6)} {details?.tokenOut.symbol}
      </div>

      <GasEstimate step={details} />
    </ConfirmBox>
  )
}
```

**场景2：多步骤跨链显示**
```tsx
function MultiStepCrossChain({ route }) {
  return (
    <div className="route-steps">
      {route.steps.map((step, index) => (
        <StepCard key={index} step={step} index={index} />
      ))}
    </div>
  )
}

function StepCard({ step, index }) {
  const { data: details } = useCrossChainTradeStep({
    step,
    enabled: !!step,
  })

  if (!details) return <StepSkeleton />

  return (
    <Card>
      <CardHeader>
        <StepBadge index={index + 1} />
        <BridgeIcon bridge={step.tool} />
      </CardHeader>

      <CardBody>
        <div className="flex items-center gap-2">
          <TokenAmount amount={details.amountIn} token={details.tokenIn} />
          <ArrowRightIcon />
          <TokenAmount amount={details.amountOut} token={details.tokenOut} />
        </div>

        <div className="text-sm text-gray-500 mt-2">
          via {step.tool}
        </div>
      </CardBody>
    </Card>
  )
}
```

### Dos and Don'ts

**Do（推荐做法）：**
- ✅ 使用 `enabled: !!step` 确保 step 存在时才查询
- ✅ 使用 `details?.amountIn.toSignificant(6)` 格式化金额显示
- ✅ 使用 `details?.priceImpact.toFixed(2)` 显示价格影响百分比
- ✅ 使用 `amountOutMin` 显示最小输出保证

**Don't（避免做法）：**
- ❌ 不要在 step 为空或 undefined 时调用
- ❌ 不要直接显示 raw amount，应该用 toSignificant() 格式化
- ❌ 不要忽略 priceImpact，应该显示给用户看
- ❌ 不要假设 step 的结构，应该等待数据加载

### 注意事项

1. **依赖 Routes 返回的 step**：这个 hook 的 step 参数通常来自 useCrossChainTradeRoutes 的返回值

2. **10秒刷新**：step 详情比路由列表刷新更频繁（10秒 vs 20秒）

3. **原生代币自动转换**：API 返回的代币地址会自动转换为 Currency 对象

4. **返回完整的 Currency 和 Amount 对象**：tokenIn/tokenOut 是 Currency，amountIn/amountOut 是 Amount

5. **priceImpact 是 Percent 类型**：可以直接使用 `.toFixed()` 或 `.toString()` 方法
