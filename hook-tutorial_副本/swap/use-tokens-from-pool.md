> 源代码路径: `apps/web/src/lib/hooks/useTokensFromPool.ts`

# useTokensFromPool

## 1. 大白话讲讲这个hook的作用

`useTokensFromPool` *(一个React hook，用于从SushiSwap V2流动性池中提取代币信息，自动处理原生币和包装币的转换)* 是一个从流动性池子中提取出两个代币信息的hook。

当你有一个SushiSwap V2流动性池时，这个池子包含：
- token0：第一个代币（如USDT）
- token1：第二个代币（如ETH）
- liquidityToken：代表你在池子里份额的LP token *(流动性代币凭证，用于证明你对该池子的所有权，可以按比例获得交易手续费收入)*

这个hook帮你把这些原始的池子数据，转换成可以直接使用的 `EvmToken` *(EVM链上的代币标准类型，包含address、name、decimals、symbol等属性)* 对象。而且如果代币是原生币（如ETH），它会自动转换成 `EvmNative` *(EVM链上的原生货币类型（如ETH），不是ERC20代币，直接用ChainId标识而不是地址)* 类型，而不是包装后的wETH。

## 2. 讲讲为什么需要封装该hook

不封装会有的问题：
- 每个使用池子的地方都要写一遍 token 转换逻辑
- 原生币和包装币的判断逻辑重复
- liquidityToken 的地址计算方式（从pool.id提取）分散在各处
- 类型不统一，有些地方用token0.address，有些地方用token0

封装后：
- 统一的数据结构：返回 `{ token0, token1, liquidityToken }`
- 统一处理原生币/包装币的转换
- 提供工具函数 `getTokensFromPool`，可以在不用hook的地方使用
- 类型安全，确保返回的都是标准 `EvmToken` 类型

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
pool: {
  id: EvmID | EvmAddress           // 池子ID，格式如 "1:0x..." 或直接是地址
  token0: { address, name, decimals, symbol }
  token1: { address, name, decimals, symbol }
  chainId: EvmChainId
}
```

**输出：**
```typescript
{
  token0: EvmToken | EvmNative     // 第一个代币或原生币
  token1: EvmToken | EvmNative     // 第二个代币或原生币
  liquidityToken: EvmToken         // 流动性代币（LP token）
}
```

**执行逻辑：**
1. 接收池子的原始数据
2. 创建 `_token0` 和 `_token1` 的 `EvmToken` 对象
3. 检查 token0 是否是原生币的包装币（wETH），如果是则替换为 `EvmNative`
4. 同理处理 token1
5. 从 pool.id 中提取或直接使用作为 liquidityToken 地址
6. 创建 liquidityToken（SLP Token，decimals=18）

**数据流：**
```
Pool原始数据 --> 创建EvmToken --> 检查原生币包装 --> 替换为EvmNative
                              --> 创建LP Token --> 返回{token0, token1, liquidityToken}
```

## 4. 怎么给这个hook写AI提示词

这个hook做的事情挺直观的：把一个池子的原始数据扔进去，然后拿到三个好用的代币对象。它最厉害的地方是会帮你自动处理原生币和包装币的转换——比如池子里的 wETH 会自动变成 ETH。

### 写提示词的小技巧

**第一，缓存是必须的。** 这个hook干的是纯计算活儿，每次渲染都重新跑一遍挺浪费的。用 `useMemo` 包一下，只有当 pool 数据真的变了才重新算。

**第二，原生币的判断要用对方法。** 怎么知道一个代币是不是原生币的包装币呢？用 `EvmNative.fromChainId()` 这个方法，它能通过链ID拿到对应链的原生币（比如以太坊上就是 ETH）。

**第三，LP代币的参数是固定的。** decimals 永远是 18，symbol 永远是 "SLP"，这两个不用从任何地方读，就是写死的。

### 写提示词时要注意的条条框框

**pool.id 的格式要能兼容两种：** 有时候是 "chainId:address" 这种完整格式，有时候直接就是个 address。代码里要能处理这两种情况。

**判断包装币的逻辑不能少：** wETH、wMATIC 这些都是对应链上原生币的包装。检测到是包装币的时候，要替换成真正的原生币类型，不然后续处理可能会出问题。

**LP代币地址从哪里来：** 从 pool.id 里提取就行。有时候 pool.id 本身就是地址，有时候是 "chainId:address" 这种组合格式。

### 提示词模板

```
帮我写一个React hook，功能是从SushiSwap V2流动性池里提取出三个代币对象。

具体需求：
1. 输入一个池子的数据，包含：id、token0信息、token1信息、chainId
2. 输出三个代币对象：token0、token1、还有liquidityToken（就是LP token）
3. 重要：如果发现token是原生币的包装币（比如wETH、wMATIC这些），要转换成真正的原生币类型
4. liquidityToken的地址从pool.id里取，decimals固定是18，symbol固定是"SLP"

输入类型大概是这样：
interface Pool {
  id: string
  token0: { address: string; name: string; decimals: number; symbol: string }
  token1: { address: string; name: string; decimals: number; symbol: string }
  chainId: number
}

返回类型：
{ token0: EvmToken | EvmNative, token1: EvmToken | EvmNative, liquidityToken: EvmToken }

注意事项：
- 用EvmNative.fromChainId()来判断原生币
- 记得用useMemo缓存结果，别每次渲染都重新算
```

### 实际用的例子

```typescript
// 拿到池子数据后，用这个hook提取出三个代币
const { token0, token1, liquidityToken } = useTokensFromPool(pool)

// 用来显示交易对
console.log(`Swap ${token0.symbol} for ${token1.symbol}`)

// 用来查余额
const token0Balance = useBalance(account, token0)
```

这个hook返回的token对象都是标准类型，可以直接用于后续的余额查询、交易构建等操作。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 从池子数据中提取代币信息
const { token0, token1, liquidityToken } = useTokensFromPool(pool)

// pool结构示例
const pool = {
  id: '1:0x397ff1542f962076d0bfe58ea045ffa2d347aca0', // SushiSwap V2池子地址
  token0: { address: '0xdAC17F958D2ee523a2206206994597C13D831ec7', name: 'Tether USD', decimals: 6, symbol: 'USDT' },
  token1: { address: '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2', name: 'Wrapped Ether', decimals: 18, symbol: 'WETH' },
  chainId: 1
}
```

### 常见使用场景

**场景1：获取池子代币余额**

```typescript
const { token0, token1, liquidityToken } = useTokensFromPool(pool)

// 获取用户在各个代币上的余额
const token0Balance = useBalance(account, token0)
const token1Balance = useBalance(account, token1)
const lpBalance = useBalance(account, liquidityToken)
```

**场景2：在交易组件中使用**

```typescript
const { token0, token1, liquidityToken } = useTokensFromPool(pool)

// 用于显示交易对
<span>{token0.symbol} / {token1.symbol}</span>

// 用于构建交易
const [token0Amount, token1Amount] = useUnderlyingTokenBalanceFromPool({
  totalSupply: pool?.totalSupply,
  reserve0: pool?.reserve0,
  reserve1: pool?.reserve1,
  balance: lpBalance,
})
```

**场景3：处理原生币自动转换**

```typescript
const { token0, token1 } = useTokensFromPool(pool)

// 如果池子是 ETH/USDT，则 token1 会自动转换为 EvmNative 类型
// 而不是 Wrapped Ether (wETH)
// 这对于显示和计算非常重要

console.log(token1.symbol) // 'ETH' 而不是 'WETH'
```

### Dos and Don'ts

**✅ Do:**
- 使用返回的 token 对象进行所有代币操作（余额查询、交易等）
- 利用自动转换的原生币类型进行正确的UI显示
- 使用 `useMemo` 缓存池子数据，避免重复计算
- 理解返回值可能是 `EvmToken | EvmNative`，根据具体情况处理

**❌ Don't:**
- 不要直接使用 pool.token0.address，应该用 `useTokensFromPool` 转换后再用
- 不要假设 token 一定是 ERC20 代币，原生币（ETH）会自动转换
- 不要忽略 liquidityToken，它对于获取用户的LP份额至关重要
- 不要在 token0/token1 上做数学运算时忘记处理精度差异（decimals不同）
