> 源代码路径: `apps/web/src/lib/hooks/react-query/trade/useEvmTrade.ts`

# useEvmTrade

## 1. 大白话讲讲这个hook的作用

`useEvmTrade` *(一个React hook，用于EVM链上的交易报价和交易数据构建，支持获取最优路径、计算滑点、估算gas等)* 是EVM链上进行交易的核心hook，负责获取交易报价和构建交易数据。

它帮你：
1. 调用SushiSwap API获取最优交易路径
2. 计算预期输出金额、最小输出金额（考虑滑点）
3. 计算gas估算和费用
4. 构建可执行的交易数据（tx）

这是EVM链Swap功能的核心数据源。

## 2. 讲讲为什么需要封装该hook

交易逻辑非常复杂：
- 需要调用多个API端点
- 需要处理原生币/ERC20的区别
- 需要计算各种费用（UI fee、gas）
- 需要处理手续费白名单逻辑
- RedSnwapper链有特殊处理

封装后：
- 统一的接口，隐藏复杂性
- 自动处理链、token类型
- 提供完整的交易数据

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
  tokenTax?: Percent
  recipient?: Address
  source?: string
  onlyPools?: Address[]
  enabled?: boolean
}
```

**输出：**
```typescript
{
  swapPrice: Price | undefined        // 交易价格
  priceImpact: Percent                 // 价格影响
  amountIn: Amount | undefined         // 输入金额
  amountOut: Amount | undefined        // 输出金额
  minAmountOut: Amount | undefined     // 最小输出（考虑滑点）
  gasSpent: string | undefined        // gas消耗（人类可读）
  gasSpentUsd: string | undefined    // gas USD价值
  fee: string | undefined             // 手续费描述
  route: Route | undefined             // 路由信息
  status: string | undefined           // 状态
  tx: { from, to, data, value, gas, gasPrice } | undefined  // 交易数据
  tokenTax: Percent | undefined
  routingSource: string | undefined
}
```

**执行逻辑：**
1. 构建URL：`${API_BASE_URL}/swap/v7/${chainId}`
2. 设置各种参数：tokenIn, tokenOut, amount, maxSlippage, sender
3. 如果不在白名单，添加fee参数
4. 调用API，获取响应
5. 使用 `tradeValidator02` *(验证交易响应数据的验证器函数)* 验证响应
6. 使用 `apiAdapter02To01` *(将API响应格式转换为内部标准格式的适配器函数)* 转换格式
7. 使用 select 回调转换数据：
   - 计算 minAmountOut（扣除slippage和fee）
   - 计算 gasSpent（考虑gas margin）
   - 计算 gasSpentUsd
   - 构建 tx 对象

**数据流：**
```
输入参数 --> /swap/v7 API --> 验证 --> 转换 --> 数据处理 --> 返回
                         |
                         v
                    计算minAmountOut, gasSpent, fee
```

## 4. 怎么给这个hook写AI提示词

这是EVM链上做交易的核心hook——拿到报价、算好各种费用、构建好交易数据。调用方只需要拿着交易数据去发交易就行了。

### 写提示词的小技巧

**第一，enabled条件要写完整。** amount、fromToken、toToken 这些必须有。任何一个缺失都不应该发请求。

**第二，select回调里做数据转换。** API返回的原始数据要转成好用的格式，计算最小输出（扣滑点）、计算gas费用这些都在select里处理。

**第三，报价要保持新鲜。** refetchInterval 设成 2500ms（2.5秒），每2.5秒自动刷新一次报价。交易价格变动快，必须经常更新。

**第四，缓存要关掉。** gcTime 设成 0，意思是每次渲染都是新请求，不走缓存。报价不新鲜的话用户会吃亏。

### 写提示词时要注意的条条框框

**amount必须是正经Amount对象。** undefined或者null都不行，写enabled条件的时候要检查。

**slippagePercentage是小数形式。** 0.5 代表 0.5%，不是 50。容易搞混，写代码的时候别搞错了。

**tokenTax会影响最小输出。** 有些代币有税，实际到手的会比理论上少一些。minAmountOut 的计算要把这个考虑进去。

**gasPrice有默认值。** 如果没传的话默认是 50n（50 * 10^9 wei，也就是 50 gwei）。

### 提示词模板

```
帮我写一个React hook，功能是获取EVM链的交易报价并构建交易数据。

具体需求：
1. 输入：链ID、输入代币、输出代币、金额、滑点百分比等
2. 调用 SushiSwap 的 /swap/v7 接口获取报价
3. 构建交易数据（tx对象，包含data、to、from、value这些）
4. 计算各种费用：gas消耗、gas的美元价值、手续费等
5. 返回完整的交易信息

参数类型大概这样：
{
  chainId: EvmChainId
  fromToken: EvmCurrency
  toToken: EvmCurrency
  amount: Amount | undefined
  slippagePercentage: number
  gasPrice?: bigint
  tokenTax?: Percent
  recipient?: Address
  source?: string
  onlyPools?: Address[]
  enabled?: boolean
}

返回：
{
  swapPrice, priceImpact, amountIn, amountOut, minAmountOut,
  gasSpent, gasSpentUsd, fee, route, status, tx, tokenTax, routingSource
}

注意：
- refetchInterval: 2500ms，保持报价经常刷新
- gcTime: 0，不缓存，每次都是新请求
- retry: false，失败立即返回不重试
- 数据转换在 select 回调里做
- gas 要加 margin
```

### 实际用的例子

```typescript
const { data: trade } = useEvmTrade({
  chainId: ChainId.ETHEREUM,
  fromToken: ETH,
  toToken: USDC,
  amount: amountIn,
  slippagePercentage: 0.5,
  gasPrice: gasPrice,
  enabled: Boolean(amountIn && fromToken && toToken),
})

// 发送交易
if (trade?.tx) {
  const hash = await writeContract({
    address: trade.tx.to,
    abi: routerAbi,
    functionName: 'swap',
    data: trade.tx.data,
    value: trade.tx.value,
  })
}
```

enabled条件检查了 amountIn、fromToken、toToken 都存在才发请求。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
const { data: trade, isLoading, error } = useEvmTrade({
  chainId: ChainId.ETHEREUM,
  fromToken: ETH,
  toToken: USDC,
  amount: amountIn,
  slippagePercentage: 0.5, // 0.5% 滑点
  gasPrice: gasPrice,
  enabled: Boolean(amountIn && fromToken && toToken),
})

// trade 包含完整的交易信息
console.log(trade?.amountOut?.toSignificant()) // 输出数量
console.log(trade?.minAmountOut?.toSignificant()) // 最小输出（考虑滑点）
console.log(trade?.priceImpact?.toSignificant()) // 价格影响
console.log(trade?.tx) // 可执行的交易数据
```

### 常见使用场景

**场景1：交易预览**

```typescript
const { data: trade } = useEvmTrade({...})

return (
  <TradePreview>
    <div>输入: {trade?.amountIn?.toSignificant()} {trade?.amountIn?.currency.symbol}</div>
    <div>输出: {trade?.amountOut?.toSignificant()} {trade?.amountOut?.currency.symbol}</div>
    <div>最低输出: {trade?.minAmountOut?.toSignificant()}</div>
    <div>价格影响: {trade?.priceImpact?.toSignificant()}%</div>
    <div>Gas: {trade?.gasSpent} (${trade?.gasSpentUsd})</div>
    <div>路由: {trade?.routingSource}</div>
  </TradePreview>
)
```

**场景2：发送交易**

```typescript
const { data: trade } = useEvmTrade({...})

const handleSwap = async () => {
  if (!trade?.tx) {
    throw new Error('No transaction data')
  }

  // 发送交易
  const hash = await writeContract({
    address: trade.tx.to,
    abi: routerAbi,
    functionName: 'swap',
    data: trade.tx.data,
    value: trade.tx.value,
    gas: trade.tx.gas,
    gasPrice: trade.tx.gasPrice,
  })

  console.log('Transaction hash:', hash)
}
```

**场景3：价格比较**

```typescript
// 获取多个交易平台的报价
const { data: sushiswapTrade } = useEvmTrade({
  ...params,
  source: 'sushi',
})

const { data: uniswapTrade } = useEvmTrade({
  ...params,
  source: 'uniswap',
})

// 比较最优价格
const bestTrade = sushiswapTrade?.amountOut > uniswapTrade?.amountOut
  ? sushiswapTrade
  : uniswapTrade
```

### Dos and Don'ts

**✅ Do:**
- 使用 `enabled` 确保必要参数存在时才发起请求
- 始终检查 `trade?.tx` 是否存在再发送交易
- 使用 `refetchInterval` 保持报价更新（如2500ms）
- 处理 `priceImpact`，过高时警告用户

**❌ Don't:**
- 不要在没有检查 `data?.tx` 的情况下发送交易
- 不要忽略 `error`，应该展示给用户
- 不要使用过时的报价（注意 gcTime: 0）
- 不要忽略 `minAmountOut`，这是用户能获得的最低数量
