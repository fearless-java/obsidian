> 源代码路径: `apps/web/src/lib/hooks/useV2Zap.ts`

# useV2Zap

## 1. 大白话讲讲这个hook的作用

`useV2Zap` *(一个React hook，用于一键式将单个代币投入到SushiSwap V2流动性池，自动完成代币兑换和流动性添加)* 是一个帮你"一键式"把单个代币投入到SushiSwap V2流动性池的hook。

传统操作：
1. 先把代币A换成代币B和代币C（两次交易）
2. 然后再把B和C加入流动性池（又是一笔交易）
3. 最后拿到LP Token

使用 Zap：
1. 你只需要输入一种代币（比如USDT）
2. 合约自动帮你把USDT换成ETH和USDC
3. 然后自动加入ETH/USDC流动性池
4. 你直接拿到LP Token

这个hook就是帮你调用这个Zap合约，计算最优路径，返回交易数据。

## 2. 讲讲为什么需要封装该hook

Zap操作很复杂：
- 需要调用后端API获取最优路径
- 需要处理多跳交换（USDT -> ETH -> USDC -> 池子）
- 需要构建复杂的交易数据
- 需要处理priceImpact计算

封装后：
- 提供简洁的接口，只需传入token和amount
- 统一处理API请求和响应验证
- 自动计算priceImpact
- 返回可直接使用的交易数据

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
{
  chainId: EvmChainId              // 链ID
  fromAddress: Address | undefined // 用户地址
  receiver?: Address               // 接收LP的地址（可选，默认fromAddress）
  amountIn: string | string[]      // 输入金额
  tokenIn: Address | Address[]     // 输入代币
  tokenOut: (Address | Address[]) | undefined  // 输出代币（池子）
  slippage: Percent                // 滑点
  query?: UseQueryOptions          // 可选的查询配置
}
```

**输出：**
```typescript
{
  gas: bigint                      // 预估gas
  amountOut: bigint                 // 输出LP数量
  feeAmount?: bigint[]              // 手续费
  priceImpact: number               // 价格影响（basis points）
  createdAt: number                 // 响应时间戳
  tx: {                             // 交易数据
    data: Hex
    to: Address
    from: Address
    value: bigint
  }
  route?: Route[]                   // 路径信息
}
```

**执行逻辑：**
1. 检查链是否支持Zap（`isZapSupportedChainId` *(判断指定链是否支持Zap功能的函数)*）
2. 验证必要参数（fromAddress, tokenOut, amountIn > 0）
3. 构建URL查询参数，调用 `/api/zap/v2`
4. 转换slippage为basis points格式
5. 解析响应，使用zod schema *(一个TypeScript优先的模式声明和验证库)* 验证
6. 计算priceImpact
7. 返回交易数据和计算结果

**数据流：**
```
输入参数 --> 构建URL --> /api/zap/v2 --> 验证响应 --> 返回交易数据
                      |
                      v
                   计算priceImpact
```

## 4. 怎么给这个hook写AI提示词

这个hook是给V2 Zap用的——简单说就是"一键充值"功能。你扔进去一种代币，它帮你自动换成池子需要的两种代币，然后替你添加到池子里，最后你拿到LP Token。比自己手动操作省事儿多了。

### 写提示词的小技巧

**第一，priceImpact一定要检查。** Zap操作涉及代币兑换，兑换会产生价格影响。如果这个值是 null，说明出问题了，应该报错而不是继续。另外如果价格影响太大，要提醒用户。

**第二，enabled条件要写完整。** 这个hook能不能执行，要看：链是不是支持、地址有没有、金额是不是大于0、tokenOut有没有。任何一个不满足都不应该发请求。

**第三，设置合适的缓存时间。** Zap路径计算挺耗时的，频繁请求没必要。staleTime 设为一分钟比较合理，一分钟内重复请求会直接用缓存。

**第四，路由找不到要能识别。** 用 `isZapRouteNotFoundError` 判断错误类型。路由找不到意味着这个池子不支持 Zap，要给用户明确的提示。

### 写提示词时要注意的条条框框

**只支持特定的链：** 不是所有链都支持 Zap，先用 `isZapSupportedChainId` 检查一下。

**金额必须是字符串且大于0：** amountIn 传进来的是字符串形式的数字，要确保转换成数字后大于 0 才能用。

**滑点要转成basis points：** Percent 对象里存的是 0.005 这种小数，但合约要的是 50（因为 basis points 的分母是 10000）。记得乘以 100 转换一下。

**tokenIn和tokenOut可以是数组：** 这意味着支持多跳兑换。比如输入USDT，池子是ETH/USDC，那USDT先换成ETH，再换成USDC，最后加池子。

### 提示词模板

```
帮我写一个React hook，功能是调用V2 Zap合约实现一键充值。

具体需求：
1. 输入这些参数：链ID、用户地址、输入的代币和金额、输出的代币（池子）、滑点
2. 调用后端接口 /api/zap/v2 获取Zap路径
3. 用Zod验证API返回的数据对不对
4. 计算价格影响（priceImpact）
5. 返回交易数据（data, to, from, value这些）
6. 要有 enabled 开关

大概的参数类型：
type UseV2ZapParams = {
  chainId: EvmChainId
  fromAddress: Address | undefined
  amountIn: string | string[]
  tokenIn: Address | Address[]
  tokenOut: (Address | Address[]) | undefined
  slippage: Percent
  query?: UseQueryOptions
}

返回：{ gas, amountOut, feeAmount?, priceImpact, tx, route? }

注意事项：
- slippage 要乘以100转成 basis points
- 用 useQuery 管理请求
- 先检查 isZapSupportedChainId
- 金额必须大于0
- enabled 要检查所有必要参数都满足才执行
```

### 实际用的例子

```typescript
// 调用hook
const { data, isLoading, error } = useV2Zap({
  chainId: ChainId.ETHEREUM,
  fromAddress: account,
  amountIn: '1000000', // 1 USDC (6 decimals)
  tokenIn: USDC.address,
  tokenOut: [ETH.address, USDC.address], // 目标池子
  slippage: new Percent(50, 10000), // 0.5%
})

// 发送交易
if (data?.tx) {
  const hash = await writeContract({
    address: data.tx.to,
    abi: zapAbi,
    functionName: 'zapIn',
    data: data.tx.data,
    value: data.tx.value,
  })
}
```

这里Percent(50, 10000)表示0.5%的滑点。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
const { data, isLoading, error } = useV2Zap({
  chainId: ChainId.ETHEREUM,
  fromAddress: account, // 用户钱包地址
  amountIn: '1000000', // 1 USDC (假设USDC是6 decimals)
  tokenIn: USDC.address,
  tokenOut: [ETH.address, USDC.address], // 目标流动性池的两个代币
  slippage: new Percent(50, 10000), // 0.5% 滑点
})

// data.tx 包含可执行的交易数据
if (data?.tx) {
  // 发送交易
}
```

### 常见使用场景

**场景1：一键充值流动性**

```typescript
const [slippage] = useSlippageTolerance()

const { data, isLoading } = useV2Zap({
  chainId,
  fromAddress: account,
  amountIn: inputAmount.toString(), // 用户输入的代币数量
  tokenIn: tokenIn.address, // 输入的代币
  tokenOut: [token0.address, token1.address], // 目标池子代币
  slippage,
})

// 显示预估输出
console.log(`预计获得 ${data?.amountOut} LP`)
console.log(`价格影响 ${(data?.priceImpact ?? 0) / 100}%`)
```

**场景2：处理Zap特定错误**

```typescript
const { data, error, isLoading } = useV2Zap({...})

// 检测路由未找到
if (error && isZapRouteNotFoundError(error)) {
  showToast({
    type: 'error',
    message: '该池子暂不支持Zap，请手动添加流动性',
  })
}
```

**场景3：显示交易详情**

```typescript
const { data } = useV2Zap({...})

// 交易前显示确认信息
return (
  <ConfirmDialog>
    <div>输入: {formatAmount(data?.amountIn)} USDC</div>
    <div>预计获得: {formatAmount(data?.amountOut)} LP</div>
    <div>价格影响: {(data?.priceImpact ?? 0) / 100}%</div>
    <div>Gas估算: {data?.gas?.toString()}</div>
  </ConfirmDialog>
)
```

### Dos and Don'ts

**✅ Do:**
- 始终检查 `enabled` 条件：链支持、地址有效、金额大于0、tokenOut存在
- 检查 `priceImpact`，如果过高应该警告用户
- 处理 `isZapRouteNotFoundError` 错误，给用户友好提示
- 使用 `staleTime: 60 * 1000` 避免频繁请求

**❌ Don't:**
- 不要在没有检查 `data?.tx` 是否存在时发送交易
- 不要忽略 `priceImpact`，过高的价格影响可能导致用户损失
- 不要对不支持Zap的链发起请求，应该先用 `isZapSupportedChainId` 检查
- 不要使用过大的滑点，0.5%-1%是合理范围，过大可能被盗要挟
