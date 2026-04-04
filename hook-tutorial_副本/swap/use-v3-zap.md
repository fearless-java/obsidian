> 源代码路径: `apps/web/src/lib/hooks/useV3Zap.ts`

# useV3Zap

## 1. 大白话讲讲这个hook的作用

`useV3Zap` *(一个React hook，用于一键式将代币投入到SushiSwap V3集中流动性池，自动完成代币兑换和流动性头寸创建)* 是一个帮你"一键式"把代币投入到SushiSwap V3 CLMM（集中流动性池）的hook。

与V2不同，V3的Zap更复杂：
- V3池子有tick范围（价格区间）的概念
- 你需要指定在哪个价格区间提供流动性
- Zap会自动帮你：换token -> 存入CLMM池 -> 创建流动性头寸

这个hook帮你：
1. 调用后端API计算最优路径
2. 获取创建流动性头寸的交易数据
3. 计算创建后的头寸价值和priceImpact

## 2. 讲讲为什么需要封装该hook

V3 Zap比V2 Zap更复杂：
- 需要处理 tick 范围选择
- 需要获取当前价格来计算头寸的USDC价值
- 需要从合约层面处理CLMM的特殊性
- priceImpact计算需要额外获取价格数据

封装后：
- 统一处理价格获取和头寸计算
- 自动构建V3 Position对象
- 封装了复杂的bundle处理逻辑
- 返回可直接使用的交易数据

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
{
  chainId: SushiSwapV3ChainId      // 链ID
  sender: Address | undefined      // 发送者地址
  receiver?: Address               // 接收者地址
  pool: SushiSwapV3Pool | undefined // V3池子
  amountIn: Amount<EvmCurrency>    // 输入金额
  slippage: Percent                // 滑点
  ticks: [number, number] | undefined  // tick范围
  query?: UseQueryOptions
}
```

**输出：**
```typescript
{
  createdAt: number
  tx: { data, to, from, value }
  amountsOut: Record<Address, bigint>  // 各种代币输出
  gas: bigint
  bundle: Bundle[]                    // 操作bundle
  priceImpact: number                 // BIPS
}
```

**执行逻辑：**
1. 检查链支持、sender、pool、ticks有效性
2. 获取当前价格（`usePrices` *(获取代币价格的hook，返回代币地址到价格的映射)*）
3. 构建URL参数调用 `/api/zap/v3`
4. 解析响应，验证zod schema
5. 计算流动性头寸的USD价值：
   - 获取token0和token1的价格
   - 创建Position对象
   - 计算 `amount0USD + amount1USD`
6. 计算 priceImpact：`((inputUSD - outputUSD) / inputUSD) * 10000`
7. 返回交易数据和计算结果

**数据流：**
```
输入参数 + 价格数据
         |
         v
调用 /api/zap/v3 --> 交易数据
         |
         v
计算头寸USD价值 --> amountOutUSD
         |
         v
计算priceImpact --> 返回{tx, priceImpact, ...}
```

## 4. 怎么给这个hook写AI提示词

V3 Zap比V2复杂不少，因为V3有"集中流动性"的概念。你要指定一个价格区间（tick范围），只有价格在这个区间内时你的资金才会参与做市。这个hook帮你完成整个操作：换代币 -> 创建V3头寸 -> 拿到交易数据。

### 写提示词的小技巧

**第一，enabled条件里pool和ticks缺一不可。** V3 Zap必须知道你在哪个池子、做哪个价格区间。任何一个不存在都不应该发请求。

**第二，价格数据是必须有的。** 创建头寸后要计算这个头寸值多少钱（USD）。如果价格是 null，整个计算就卡住了，所以要抛错。

**第三，金额要大于0。** `amountIn.gt(0n)` 这个检查确保你不会传个0进去。0 amount的Zap没意义。

**第四，设置合理的缓存时间。** V3 Zap计算也挺耗时的，staleTime 一分钟比较合理。

### 写提示词时要注意的条条框框

**只支持V3链：** V2和V3的Zap接口不一样，这个hook只处理V3。

**pool和ticks必须同时存在：** pool告诉你哪个池子，ticks告诉你在哪个价格区间。分开来都没意义。

**ticks格式是[tickLower, tickUpper]：** 前者是区间下限，后者是上限。比如[-1000, 1000]表示价格相对当前价±1000个tick。

**priceImpact用basis points表示：** 100 means 1%，5000 means 50%。显示的时候要除以100才是百分比。

### 提示词模板

```
帮我写一个React hook，功能是调用V3集中流动性池的Zap接口。

具体需求：
1. 输入：链ID、发送者地址、池子对象、输入金额、滑点、tick范围（价格区间）
2. 调用 /api/zap/v3 获取创建流动性头寸所需的交易数据
3. 用 usePrices 拿到当前价格，算出头寸值多少USD
4. 计算 priceImpact（价格影响）
5. 返回交易数据和头寸信息

大概的参数类型：
type UseV3ZapParams = {
  chainId: SushiSwapV3ChainId
  sender: Address | undefined
  pool: SushiSwapV3Pool | undefined
  amountIn: Amount<EvmCurrency>
  slippage: Percent
  ticks: [number, number] | undefined
}

返回：{ tx, amountsOut, gas, bundle, priceImpact }

注意事项：
- pool 必须存在才能执行
- ticks 是 [tickLower, tickUpper] 格式
- 需要用 usePrices 获取价格
- 用 SushiSwapV3Pool 计算头寸
- priceImpact 用 basis points 表示（除以100才是百分比）
```

### 实际用的例子

```typescript
// 调用hook
const { data, isLoading, error } = useV3Zap({
  chainId: ChainId.ETHEREUM,
  sender: account,
  pool: v3Pool,
  amountIn: new Amount(ETH, '1000000000000000000'),
  slippage: new Percent(50, 10000),
  ticks: [-1000, 1000], // tick范围
})

// 显示价格影响
console.log(`价格影响: ${data?.priceImpact / 100}%`)
```

priceImpact 是 50 的话，除以100就是0.5%，表示这个操作会带来0.5%的价格影响。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 获取V3池子
const { data: pool } = useV3Pool({ chainId, address: poolAddress })

// 定义tick范围（价格区间）
const tickLower = -1000
const tickUpper = 1000

const { data, isLoading, error } = useV3Zap({
  chainId: ChainId.ETHEREUM,
  sender: account,
  pool: pool,
  amountIn: new Amount(ETH, '1000000000000000000'), // 1 ETH
  slippage: new Percent(50, 10000), // 0.5%
  ticks: [tickLower, tickUpper],
})

// data.tx 包含创建流动性头寸的交易数据
```

### 常见使用场景

**场景1：创建集中流动性头寸**

```typescript
// 用户选择价格区间后一键充值
const { data, isLoading } = useV3Zap({
  chainId,
  sender: account,
  pool: v3Pool,
  amountIn: inputAmount,
  slippage,
  ticks: [tickLower, tickUpper], // 用户选择的价格区间
})

// 显示预估结果
console.log(`Token0 输入: ${data?.amountsOut[token0.address]}`)
console.log(`Token1 输入: ${data?.amountsOut[token1.address]}`)
console.log(`价格影响: ${(data?.priceImpact ?? 0) / 100}%`)
```

**场景2：获取V3头寸的美元价值**

```typescript
const { data: v3Pool } = useV3Pool({...})
const { data: zapData } = useV3Zap({...})
const { data: prices } = usePrices({ chainId })

// V3 Zap会自动计算头寸的美元价值
// 计算公式: amount0 * price0 + amount1 * price1
```

**场景3：Tick范围选择器集成**

```typescript
// 典型的V3 tick范围选择UI
const [tickLower, setTickLower] = useState(-1000)
const [tickUpper, setTickUpper] = useState(1000)

const { data, isLoading } = useV3Zap({
  chainId,
  sender: account,
  pool: v3Pool,
  amountIn,
  slippage,
  ticks: [tickLower, tickUpper],
})

// 实时预览Zap结果
useEffect(() => {
  if (data) {
    updatePreview({
      amount0: data.amountsOut[token0.address],
      amount1: data.amountsOut[token1.address],
    })
  }
}, [data])
```

### Dos and Don'ts

**✅ Do:**
- 确保 pool 和 ticks 都存在后再调用 hook
- 使用 `enabled` 条件控制是否发起请求
- 检查 `priceImpact`，过高时提醒用户
- 使用 `usePrices` 获取价格数据用于计算
- ticks 范围要合理，太窄可能导致无常损失

**❌ Don't:**
- 不要在 pool 或 ticks 不存在时发起请求
- 不要忽略价格数据获取失败的情况
- 不要选择过窄的 tick 范围，这可能导致仓位容易越界
- 不要忽略 V3 和 V2 的区别，V3 有集中流动性的概念
