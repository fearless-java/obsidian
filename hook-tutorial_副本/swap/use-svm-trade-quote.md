> 源代码路径: `apps/web/src/lib/hooks/react-query/trade/useSvmTradeQuote.ts`

# useSvmTradeQuote

## 1. 大白话讲讲这个hook的作用

`useSvmTradeQuote` *(一个React hook，用于Solana链(SVM)上获取交易报价，自动处理wrap/unwrap和普通交易两种逻辑)* 是Solana链（SVM）上获取交易报价的hook。

它帮你：
1. 调用 Jupiter API 获取最优路径
2. 计算交易价格、输出金额
3. 计算 gas/手续费
4. 处理 SOL/wSOL 的 wrap/unwrap 情况

## 2. 讲讲为什么需要封装该hook

SVM报价复杂：
- 分为 wrap/unwrap 和普通交易两种逻辑
- 需要处理 fee mint 选项
- 需要计算 Solana 的 lamports 费用
- 需要处理 UI fee

封装后：
- 自动选择正确的报价逻辑
- 处理 wrap/unwrap 分支
- 返回统一的格式

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
params: UseSvmTradeParams | undefined
// {
//   chainId: SvmChainId
//   fromToken: SvmToken
//   toToken: SvmToken
//   amount: Amount
//   recipient?: string
// }
```

**输出：**
```typescript
{
  swapPrice: Price | undefined        // 交易价格
  priceImpact: Percent                 // 价格影响
  amountIn: Amount | undefined         // 输入
  amountOut: Amount | undefined       // 输出
  minAmountOut: Amount | undefined     // 最小输出
  gasSpent: string | undefined        // SOL消耗
  gasSpentUsd: string | undefined    // gas USD
  fee: string | undefined             // 手续费
  route: Order | undefined            // 订单信息
  status: 'Success' | 'Failed'
  tx: Transaction | undefined         // 交易对象
  tokenTax: Percent | undefined
  routingSource: string | undefined
  type: 'swap' | 'wrap/unwrap'
}
```

**执行逻辑：**
1. 检测是否是 wrap/unwrap：`useWrapUnwrapTrade(fromToken, toToken)` *(检测SOL/wSOL转换的hook)*
2. 如果是 wrap/unwrap：
   - 使用 `useWrapUnwrapQuote`
   - 计算 wrap/unwrap 的价格和手续费
3. 如果是普通交易：
   - 调用 `/api/jupiter/ultra/order`
   - 处理 fee mint 选项
   - 获取报价
   - select 回调处理数据

**数据流：**
```
params { fromToken, toToken, amount }
         |
         v
useWrapUnwrapTrade检测类型
         |
    wrap/unwrap --> useWrapUnwrapQuote --> 返回wrap/unwrap报价
         |
    普通交易
         |
         v
/api/jupiter/ultra/order --> 处理fee --> 返回交易报价
```

## 4. 怎么给这个hook写AI提示词

这是Solana链上获取交易报价的hook，和SVM执行hook配合着用——先quote报价，再execute执行。它也会自动检测是SOL/wSOL转换还是普通Jupiter路由交易。

### 写提示词的小技巧

**第一，wrap/unwrap和普通交易走不同逻辑。** 用 useWrapUnwrapTrade 检测一下是哪一种。wrap/unwrap 用独立的报价接口，普通交易走Jupiter的order接口。

**第二，refetchInterval可以设长一点。** SVM链不像EVM那么多高频交易，5秒刷新一次报价够了。

**第三，处理fee mint选项。** 有些代币转账要收手续费（fee），这个要算进去。

### 写提示词时要注意的条条框框

**只支持SVM链。** EVM链的报价要用别的hook。

**必须有fromToken、toToken、amount。** 缺任何一个都拿不到报价。

**gcTime设成0。** 报价不缓存，每次都是新请求。

### 提示词模板

```
帮我写一个React hook，功能是获取Solana链（SVM）上的交易报价。

具体需求：
1. 输入：链ID、输入代币、输出代币、数量、接收地址（可选）
2. 自动检测是 wrap/unwrap（SOL/wSOL）还是普通交易
3. wrap/unwrap 走独立的报价逻辑
4. 普通交易调用 Jupiter 的 order 接口
5. 计算手续费和gas

参数类型：
{
  chainId: SvmChainId
  fromToken: SvmToken
  toToken: SvmToken
  amount: Amount
  recipient?: string
}

返回：
{
  swapPrice, priceImpact, amountIn, amountOut, minAmountOut,
  gasSpent, gasSpentUsd, fee, route, status, tx, type
}

注意事项：
- 用 useWrapUnwrapTrade 检测交易类型
- refetchInterval: 5000ms
- gcTime: 0
- 手续费用 SVM_UI_FEE_BIPS 计算
```

### 实际用的例子

```typescript
const { data: quote, isLoading } = useSvmTradeQuote({
  chainId: SvmChainId.SOLANA,
  fromToken: SOL,
  toToken: USDC,
  amount: amountIn,
})

// 显示报价
console.log(`获得 ${quote?.amountOut?.toSignificant()} USDC`)
```

type字段可以告诉你这是wrap/unwrap还是普通swap。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
const { data: quote, isLoading, error } = useSvmTradeQuote({
  chainId: SvmChainId.SOLANA,
  fromToken: SOL,
  toToken: USDC,
  amount: amountIn,
})

// quote 包含报价信息
console.log(quote?.type) // 'swap' 或 'wrap/unwrap'
console.log(quote?.amountOut?.toSignificant()) // 输出数量
console.log(quote?.tx) // Solana交易对象
```

### 常见使用场景

**场景1：SVM交易预览**

```typescript
const { data: quote } = useSvmTradeQuote({...})

return (
  <TradePreview>
    {quote?.type === 'wrap/unwrap' ? (
      <div>SOL/wSOL 转换</div>
    ) : (
      <div>Jupiter 路由交易</div>
    )}
    <div>输入: {quote?.amountIn?.toSignificant()} {quote?.amountIn?.currency.symbol}</div>
    <div>输出: {quote?.amountOut?.toSignificant()} {quote?.amountOut?.currency.symbol}</div>
    <div>Gas: {quote?.gasSpent} SOL</div>
  </TradePreview>
)
```

**场景2：SOL 转换为 USDC**

```typescript
const { data: quote } = useSvmTradeQuote({
  chainId: SvmChainId.SOLANA,
  fromToken: SOL,
  toToken: USDC,
  amount: amountIn,
})

// 获取 unsignedBytes 后执行交易
const { mutate } = useSvmTradeExecute({
  chainId: SvmChainId.SOLANA,
  fromToken: SOL,
  toToken: USDC,
  amount: amountIn,
  unsignedBytes: quote?.tx?.unsignedBytes,
  requestId: quote?.requestId,
})
```

**场景3：检测 wrap/unwrap**

```typescript
const { data: quote } = useSvmTradeQuote({...})

// 检测是否是 SOL/wSOL 转换
if (quote?.type === 'wrap/unwrap') {
  // wrap/unwrap 处理逻辑
  console.log('Converting SOL to wSOL or vice versa')
} else {
  // 普通Jupiter交易
  console.log('Jupiter swap route:', quote?.route)
}
```

### Dos and Don'ts

**✅ Do:**
- 使用 `type` 字段判断是 wrap/unwrap 还是普通交易
- 使用 `refetchInterval: 5000ms` 保持报价更新
- 处理 `isLoading` 状态
- 区别处理 wrap/unwrap 和普通交易的 UI 显示

**❌ Don't:**
- 不要在 SVM 链上使用，应该用 `useEvmTradeQuote` 处理 EVM 链
- 不要忽略 `type` 字段，它决定了交易执行方式
- 不要假设一定有 `tx`，检查 `quote?.tx` 是否存在
- 不要忽略 `fee` 和 `gasSpent` 的显示，用户需要知道成本
