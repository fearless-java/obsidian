> 源代码路径: `apps/web/src/lib/hooks/react-query/trade/useEvmTradeQuote.ts`

# useEvmTradeQuote

## 1. 大白话讲讲这个hook的作用

`useEvmTradeQuote` *(一个React hook，用于EVM链上的交易报价获取，与useEvmTrade不同，这个hook只返回报价信息不构建交易数据)* 是获取EVM链交易"报价"（不含交易数据）的hook。

与 `useEvmTrade` 的区别：
- `useEvmTrade`：获取完整报价 + 可执行的交易数据
- `useEvmTradeQuote`：只获取报价信息，不构建交易

使用场景：
- 只展示价格，不需要立即交易
- 比较不同交易平台的价格
- 预览交易结果

## 2. 讲讲为什么需要封装该hook

报价获取逻辑：
- 调用 `/quote/v7` 而不是 `/swap/v7`
- 不需要 sender 地址
- 不构建最终交易数据
- 返回格式略有不同

封装后：
- 统一接口
- 自动处理报价计算
- 不返回可执行的tx

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
{
  chainId: EvmChainId
  fromToken: EvmCurrency
  toToken: EvmCurrency
  amount: Amount | undefined
  slippagePercentage: number
  gasPrice?: bigint
  fee?: number
  source?: string
  onlyPools?: Address[]
  enabled?: boolean
}
```

**输出：**
```typescript
// 与useEvmTrade相同的结构，但tx为undefined
{
  swapPrice, priceImpact, amountIn, amountOut, minAmountOut,
  gasSpent, gasSpentUsd, fee, route, status, tx: undefined, ...
}
```

**执行逻辑：**
1. 构建URL：`${API_BASE_URL}/quote/v7/${chainId}`
2. 设置参数（与useEvmTrade类似，但不设置sender）
3. 调用API验证响应
4. select回调处理数据（不包含tx）

## 4. 怎么给这个hook写AI提示词

这个hook和 useEvmTrade 长得很像，但功能更简单：只拿报价，不构建交易数据。适合只需要展示价格、不需要实际发交易的场景，比如价格比较、汇率计算器之类的。

### 写提示词的小技巧

**第一，不需要sender地址。** 既然不构建交易，就不需要知道是谁在买。/quote/v7 接口也不接受sender参数。

**第二，不构建tx。** 返回的结构里 tx 永远是 undefined。只返回价格、数量、费用这些信息。

**第三，和 useEvmTrade 用类似的 select 逻辑。** 数据转换部分可以复用，只是最后不构建tx对象。

### 写提示词时要注意的条条框框

**enabled条件比useEvmTrade简单一些。** 不需要检查address，但 amount、chainId、fromToken、toToken、gasPrice 这些还是要有。

**refetchInterval 同样是 2500ms。** 报价要保持新鲜。

### 提示词模板

```
帮我写一个React hook，功能是获取EVM链的交易报价（不含交易数据）。

具体需求：
1. 调用 /quote/v7 接口（注意不是 /swap/v7）
2. 只获取报价信息，不构建交易数据
3. 返回的结构和 useEvmTrade 一样，但 tx 是 undefined
4. 支持 source 和 onlyPools 过滤

返回：和 useEvmTrade 一样的结构，只是 tx 为 undefined

注意事项：
- 不需要 sender 地址
- enabled 条件里不需要检查 address
- refetchInterval: 2500ms
```

### 实际用的例子

```typescript
const { data: quote } = useEvmTradeQuote({
  chainId: ChainId.ETHEREUM,
  fromToken: ETH,
  toToken: USDC,
  amount: amountIn,
  slippagePercentage: 0.5,
  enabled: Boolean(amountIn),
})

// 只显示价格
console.log(`预计获得 ${quote?.amountOut?.toSignificant()} USDC`)
console.log(quote?.tx) // undefined - 这是和 useEvmTrade 的关键区别
```

适合用来做价格展示、汇率计算，不适合实际发交易。

### Production-Ready Example

```typescript
const { data: quote } = useEvmTradeQuote({
  chainId: ChainId.ETHEREUM,
  fromToken: ETH,
  toToken: USDC,
  amount: amountIn,
  slippagePercentage: 0.5,
  enabled: Boolean(amountIn),
})

// 只显示价格
console.log(`预计获得 ${quote?.amountOut?.toSignificant()} USDC`)
```

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 与useEvmTrade类似，但不返回tx
const { data: quote, isLoading, error } = useEvmTradeQuote({
  chainId: ChainId.ETHEREUM,
  fromToken: ETH,
  toToken: USDC,
  amount: amountIn,
  slippagePercentage: 0.5,
  enabled: Boolean(amountIn),
})

// quote 包含报价信息，但没有 tx
console.log(quote?.amountOut?.toSignificant())
console.log(quote?.priceImpact?.toSignificant())
console.log(quote?.tx) // undefined - 这是与useEvmTrade的关键区别
```

### 常见使用场景

**场景1：价格预览（不发送交易）**

```typescript
const { data: quote } = useEvmTradeQuote({...})

// 实时显示报价
return (
  <PricePreview>
    <div>当前汇率: 1 ETH = {quote?.amountOut?.toSignificant()} USDC</div>
    <div>价格影响: {quote?.priceImpact?.toSignificant()}%</div>
    <div>Gas: ~{quote?.gasSpent} ETH</div>
  </PricePreview>
)
```

**场景2：多平台价格比较**

```typescript
// 获取SushiSwap报价
const { data: sushiQuote } = useEvmTradeQuote({
  ...params,
  source: 'sushi',
})

// 获取Uniswap报价
const { data: uniswapQuote } = useEvmTradeQuote({
  ...params,
  source: 'uniswap',
})

// 展示比较
return (
  <PriceComparison>
    <div>SushiSwap: {sushiQuote?.amountOut?.toSignificant()}</div>
    <div>Uniswap: {uniswapQuote?.amountOut?.toSignificant()}</div>
    <div>最佳价格: {sushiQuote?.amountOut > uniswapQuote?.amountOut ? 'SushiSwap' : 'Uniswap'}</div>
  </PriceComparison>
)
```

**场景3：汇率计算器**

```typescript
const [inputAmount, setInputAmount] = useState('')

const { data: quote } = useEvmTradeQuote({
  chainId,
  fromToken: tokenA,
  toToken: tokenB,
  amount: parseAmount(inputAmount),
  slippagePercentage: 0.5,
})

return (
  <Calculator>
    <input value={inputAmount} onChange={e => setInputAmount(e.target.value)} />
    <div>≈ {quote?.amountOut?.toSignificant()} {tokenB.symbol}</div>
  </Calculator>
)
```

### Dos and Don'ts

**✅ Do:**
- 使用 `enabled` 确保必要参数存在
- 使用 `refetchInterval` 保持报价实时更新
- 适合不需要立即交易的价格展示场景
- 与 `useEvmTrade` 结合使用，先 quote 再 trade

**❌ Don't:**
- 不要尝试使用 `quote.tx` 发送交易，它永远是 undefined
- 不要在没有 amount 时发起请求
- 不要忽略 `priceImpact` 显示，应该告知用户
- 不要在需要实际交易时使用这个 hook，应该用 `useEvmTrade`
