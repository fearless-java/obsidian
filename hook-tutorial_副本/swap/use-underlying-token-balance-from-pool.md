> 源代码路径: `apps/web/src/lib/hooks/useUnderlyingTokenBalanceFromPool.ts`

# useUnderlyingTokenBalanceFromPool

## 1. 大白话讲讲这个hook的作用

`useUnderlyingTokenBalanceFromPool` *(一个React hook，用于计算用户在SushiSwap V2流动性池中的真实代币余额，考虑了手续费收益和份额比例)* 帮你计算你在SushiSwap V2流动性池中的"真实"代币余额。

当你往池子里添加流动性时，你会获得LP Token（流动性凭证）。这个hook根据你持有的LP Token数量，以及池子的储备金数据，计算出你实际拥有的：
- 池子里token0的数量
- 池子里token1的数量

比如你往 ETH/USDT 池子存了100 LP，每个LP代表0.01 ETH + 50 USDT，那这个hook就告诉你：你实际持有 1 ETH 和 5000 USDT。

## 2. 讲讲为什么需要封装该hook

计算用户在一个池子中的"真实"余额很复杂：
- 需要调用合约获取 feeTo（是否开启了手续费）
- 需要获取 kLast（用于计算手续费收益）
- 需要按照数学公式分配代币
- 还要处理新铸造LP还没被计入totalSupply的情况

直接写会导致：
- 合约调用逻辑重复
- 数学计算（getLiquidityValue）分散在多处
- 不处理边界情况（zero totalSupply、fee没开启等）

封装后：
- 封装了所有合约调用
- 统一处理各种边界情况
- 返回 `[token0Balance, token1Balance]` 格式简洁

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
{
  totalSupply: Amount<EvmToken> | undefined | null  // LP Token总供应量
  reserve0: Amount<EvmCurrency> | undefined | null  // 池子储备0
  reserve1: Amount<EvmCurrency> | undefined | null  // 池子储备1
  balance: Amount<EvmCurrency> | undefined | null  // 用户持有的LP数量
}
```

**输出：**
```typescript
[Amount<EvmCurrency>, Amount<EvmCurrency>]  // [token0余额, token1余额]
```

**执行逻辑：**
1. 调用 `useReadContracts` *(wagmi框架的hook，用于批量读取区块链合约数据，支持并行请求和自动缓存)* 并行获取：
   - `feeTo`: 手续费接收地址（判断手续费是否开启）
   - `kLast`: 上次触发手续费的K值（用于计算累积手续费）
2. 验证数据有效性（totalSupply >= balance，防止新铸造LP未计入）
3. 处理边界情况：
   - totalSupply为0：返回0, 0
   - balance > totalSupply（短剧）：返回 undefined, undefined
4. 创建 SushiSwapV2Pool 实例
5. 调用 `pool.getLiquidityValue()` *(池子合约的方法，根据用户LP份额和总供应量计算应得的代币数量)* 计算每个代币的份额
6. 返回 [token0Amount, token1Amount]

**数据流：**
```
用户LP余额 + 池子储备 + totalSupply
         |
         v
调用合约 feeTo, kLast
         |
         v
计算份额：balance / totalSupply * reserve
         |
         v
考虑手续费收益（feeEnabled, kLast）
         |
         v
[token0Amount, token1Amount]
```

## 4. 怎么给这个hook写AI提示词

这个hook算的是你在SushiSwap V2流动性池里真正能拿到多少代币。为啥说"真正"呢？因为你手里拿的是LP Token（代表你在池子里的份额），而不是直接的代币。这个hook会根据你的LP数量、池子储备、还有手续费收益，计算出你实际持有的两种代币数量。

### 写提示词的小技巧

**第一，参数不全就别算了，直接返回undefined。** 四个参数（totalSupply、reserve0、reserve1、balance）任何一个缺了都算不出来，硬算只会出bug。不如直接返回 undefined，让调用方去处理"无数据"的显示。

**第二，合约调用要并行，别串着来。** feeTo 和 kLast 是两个独立的合约调用，应该同时发出去，用 `useReadContracts` 批量处理。串行的话会慢一倍，用户体验就差了。

**第三，结果要用useMemo缓存。** 这个计算涉及到池子合约的调用和一些数学运算，每次渲染都重新算太浪费。池子数据没变的话，就用缓存的结果。

**第四，用Amount类型处理精度。** decimals 不同会导致代币数量不能直接比较。Amount 类型会帮你处理这些，不用担心 18 位精度和 6 位精度的代币混在一起会算错。

### 写提示词时要注意的条条框框

**balance 不能超过 totalSupply：** 如果你持有的LP比总供应量还多，那说明数据有问题——这是个"短裤检测"，发现这种情况直接返回 undefined。

**feeTo是零地址说明手续费没开：** 手续费开关决定了你能不能拿到交易费分红。如果 feeTo 是零地址，那就不用算手续费收益那部分。

**kLast为0且手续费开启的情况要特殊处理：** 这意味着还没累积过手续费，计算的时候要单独判断一下。

**totalSupply为0就直接返回[0, 0]：** 池子空了，没你啥事儿。

### 提示词模板

```
帮我写一个React hook，功能是计算用户在SushiSwap V2池子里的真实代币余额。

具体需求：
1. 输入四个东西：LP代币总供应量、池子储备0、池子储备1、用户持有的LP数量
2. 输出用户在池子里对应的两个代币数量
3. 要调用合约查 feeTo（手续费开关）和 kLast（累计手续费）来判断手续费
4. 用池子合约的 getLiquidityValue 方法来算
5. 要处理各种边界情况：totalSupply为0返回[0,0]，balance大于totalSupply返回undefined

输入类型：
interface Params {
  totalSupply: Amount<EvmToken> | undefined | null
  reserve0: Amount<EvmCurrency> | undefined | null
  reserve1: Amount<EvmCurrency> | undefined | null
  balance: Amount<EvmCurrency> | undefined | null
}

返回：[Amount<EvmCurrency> | undefined, Amount<EvmCurrency> | undefined]

注意事项：
- 用 useReadContracts 并行读 feeTo 和 kLast
- feeTo !== zeroAddress 说明手续费开关是开着的
- kLast 要传给 getLiquidityValue 来算手续费收益
- 用 useMemo 缓存结果
```

### 实际用的例子

```typescript
// 拿到池子数据
const { data: pool } = useV2Pool(poolId)

// 拿到用户LP余额
const { data: lpBalance } = useBalance(account, liquidityToken)

// 用hook计算真实余额
const [token0Amount, token1Amount] = useUnderlyingTokenBalanceFromPool({
  totalSupply: pool?.totalSupply,
  reserve0: pool?.reserve0,
  reserve1: pool?.reserve1,
  balance: lpBalance,
})

// 显示
console.log(`你持有 ${token0Amount?.toSignificant()} ${token0.symbol}`)
```

这里用 `toSignificant()` 来显示代币数量，它会自动处理精度问题，显示成人类可读的形式。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 获取池子数据
const { data: pool } = useV2Pool(poolId)

// 获取用户LP余额
const { data: lpBalance } = useBalance(account, liquidityToken)

// 使用hook计算真实余额
const [token0Amount, token1Amount] = useUnderlyingTokenBalanceFromPool({
  totalSupply: pool?.totalSupply,
  reserve0: pool?.reserve0,
  reserve1: pool?.reserve1,
  balance: lpBalance,
})

// token0Amount 和 token1Amount 是 Amount 类型
console.log(token0Amount?.toSignificant()) // 显示代币数量
console.log(token1Amount?.toExact()) // 显示精确数量
```

### 常见使用场景

**场景1：显示用户流动性组合**

```typescript
const [token0Amount, token1Amount] = useUnderlyingTokenBalanceFromPool({
  totalSupply: pool?.totalSupply,
  reserve0: pool?.reserve0,
  reserve1: pool?.reserve1,
  balance: lpBalance,
})

// 显示用户的真实持仓
return (
  <div>
    <span>ETH: {token0Amount?.toSignificant()}</span>
    <span>USDT: {token1Amount?.toSignificant()}</span>
  </div>
)
```

**场景2：计算总资产价值**

```typescript
const [token0Amount, token1Amount] = useUnderlyingTokenBalanceFromPool({...})

// 获取美元价值
const dollarValues = useTokenAmountDollarValues({
  chainId,
  amounts: [token0Amount, token1Amount],
})

const totalValue = dollarValues.reduce((sum, val) => sum + val, 0)
console.log(`总价值: $${totalValue.toFixed(2)}`)
```

**场景3：提取流动性时的预览**

```typescript
const [token0Amount, token1Amount] = useUnderlyingTokenBalanceFromPool({...})

// 用户点击"提取流动性"时显示将获得的代币
const handleWithdraw = () => {
  showConfirmDialog({
    title: '提取流动性',
    message: `你将获得: ${token0Amount?.toSignificant()} ${token0.symbol} + ${token1Amount?.toSignificant()} ${token1.symbol}`,
  })
}
```

### Dos and Don'ts

**✅ Do:**
- 确保所有输入参数都存在时才进行计算
- 使用 `useMemo` 缓存结果，避免重复计算
- 处理 undefined 返回值，UI需要优雅显示"无数据"
- 理解返回值是 `Amount` 类型，使用其方法（如 `toSignificant()`）进行显示

**❌ Don't:**
- 不要假设返回值一定存在，需要检查 undefined 情况
- 不要直接对 Amount 进行数学运算，应该用 `Amount` 的 mul、div 等方法
- 不要忽略手续费收益，这是用户的重要收入来源
- 不要在 balance > totalSupply 时继续计算，这是一个异常状态（短剧检测失败）
