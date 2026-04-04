> 源代码路径: `apps/web/src/lib/hooks/api/useV2Pool.ts`

# useV2Pool

## 1. 大白话讲讲这个hook的作用

`useV2Pool` *(一个React hook，用于获取SushiSwap V2池子的详细信息，包括代币、储备金、流动性等)* 帮你获取单个SushiSwap V2池子的详细信息。

返回数据包括：
- token0、token1（代币信息）
- reserve0、reserve1（储备金数量）
- totalSupply（LP总供应量）
- liquidityToken（LP代币信息）
- liquidityUSD（流动性USD价值）

## 2. 讲讲为什么需要封装该hook

池子数据需要：
- 从图数据库获取原始数据
- hydrate（合成为完整对象）
- 转换为 Amount 类型便于计算
- 额外生成 liquidityToken

封装后：
- 自动获取并处理数据
- 返回可直接使用的格式
- 提供 liquidityToken 用于后续balance查询

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
poolId: PoolId  // { chainId, address }
```

**输出：**
```typescript
{
  isLoading: boolean,
  error: Error | null,
  data: {
    token0: EvmToken
    token1: EvmToken
    liquidityToken: EvmToken
    liquidityUSD: number
    reserve0: Amount<token0>
    reserve1: Amount<token1>
    totalSupply: Amount<liquidityToken>
  } | null
}
```

**执行逻辑：**
1. 验证 chainId 是有效的 V2 chainId
2. 调用 `getV2Pool({ chainId, address })` *(获取V2池子详情的GraphQL函数)*
3. hydrate 原始池子数据
4. 使用 `useMemo` *(React hook，用于记忆化计算结果，避免在每次渲染时重新计算，只有依赖变化时才重新执行)* 计算：
   - liquidityToken（使用 getLiquidityTokenFromPool）
   - reserve0、reserve1 转为 Amount
   - totalSupply 转为 Amount

**数据流：**
```
poolId { chainId, address }
         |
         v
getV2Pool --> rawPool
         |
         v
hydrateV2Pool --> pool object
         |
         v
getLiquidityTokenFromPool --> liquidityToken
         |
         v
Amount conversion --> reserve0, reserve1, totalSupply
         |
         v
return data
```

## 4. 怎么给这个hook写AI提示词

这个hook拿来查单个V2池子的详细信息——两个代币是什么、储备金多少、LP总供应量多少、流动性值多少美金。还能拿到 liquidityToken（LP代币），方便后续查用户持有多少LP。

### 写提示词的小技巧

**第一，先验证chainId。** 不是所有链都有V2池子，用 `isSushiSwapV2ChainId` 检查一下。不支持的链就别发请求了。

**第二，queryKey要处理BigInt。** poolId里的address是地址类型，可能涉及BigInt。序列化的时候要能处理这个，用 `stringify` 当 hash 函数比较靠谱。

**第三，数据转换放useMemo里。** getV2Pool返回的是原始数据，要转成Amount类型、处理liquidityToken，这些计算放useMemo缓存住。

### 写提示词时要注意的条条框框

**chainId必须是V2支持的链。** V2和V3的池子不兼容，链ID写错了查不出来。

**poolId.address可以是两种格式。** 可以是 "chainId:address" 组合格式，也可以直接就是地址。代码里要能处理这两种。

### 提示词模板

```
帮我写一个React hook，功能是获取单个V2池子的详细信息。

具体需求：
1. 输入 poolId，包含 chainId 和 address
2. 调用 getV2Pool 获取原始数据
3. 用 hydrateV2Pool 处理数据
4. 生成 liquidityToken
5. 转换为 Amount 类型
6. 验证 chainId 是有效的 V2 链

poolId 的类型：
poolId: { chainId: SushiSwapV2ChainId, address: EvmAddress }

返回：
{
  isLoading: boolean
  error: Error | null
  data: {
    token0: EvmToken
    token1: EvmToken
    liquidityToken: EvmToken
    liquidityUSD: number
    reserve0: Amount | null
    reserve1: Amount | null
    totalSupply: Amount | null
  } | null
}
```

### 实际用的例子

```typescript
const { data: pool, isLoading } = useV2Pool({
  chainId: ChainId.ETHEREUM,
  address: '0x...',
})

// 获取用户LP余额
const { data: lpBalance } = useBalance(account, pool?.liquidityToken)
```

拿到 pool 之后就能查用户在这个池子里的LP持仓，然后进一步算出真实代币数量。

### Production-Ready Example

```typescript
const { data: pool, isLoading } = useV2Pool({
  chainId: ChainId.ETHEREUM,
  address: '0x...',
})

// 获取用户LP余额
const { data: lpBalance } = useBalance(account, pool?.liquidityToken)

// 获取用户真实代币数量
const [token0Amount, token1Amount] = useUnderlyingTokenBalanceFromPool({
  totalSupply: pool?.totalSupply,
  reserve0: pool?.reserve0,
  reserve1: pool?.reserve1,
  balance: lpBalance,
})
```

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 基础调用
const { data: pool, isLoading, error } = useV2Pool({
  chainId: ChainId.ETHEREUM,
  address: '0x397ff1542f962076d0bfe58ea045ffa2d347aca0',
})

// pool 包含池子的所有信息
console.log(pool?.token0.symbol) // 第一个代币符号
console.log(pool?.reserve0?.toSignificant()) // 储备0数量
console.log(pool?.liquidityUSD) // 流动性USD价值
```

### 常见使用场景

**场景1：获取池子信息和用户头寸**

```typescript
const { data: pool } = useV2Pool({ chainId, address: poolAddress })

// 获取用户LP余额
const { data: lpBalance } = useBalance(account, pool?.liquidityToken)

// 获取用户真实代币数量
const [token0Amount, token1Amount] = useUnderlyingTokenBalanceFromPool({
  totalSupply: pool?.totalSupply,
  reserve0: pool?.reserve0,
  reserve1: pool?.reserve1,
  balance: lpBalance,
})

return (
  <PoolInfo>
    <div>池子: {pool?.token0.symbol} / {pool?.token1.symbol}</div>
    <div>你的LP: {lpBalance?.toSignificant()}</div>
    <div>真实持仓: {token0Amount?.toSignificant()} + {token1Amount?.toSignificant()}</div>
  </PoolInfo>
)
```

**场景2：计算份额比例**

```typescript
const { data: pool } = useV2Pool({...})
const { data: lpBalance } = useBalance(account, pool?.liquidityToken)

// 计算用户占池子的份额
const sharePercent = pool?.totalSupply && lpBalance
  ? lpBalance.div(pool.totalSupply).multiply(100)
  : 0
```

**场景3：添加/提取流动性**

```typescript
const { data: pool } = useV2Pool({...})

// 添加流动性
const handleAddLiquidity = (amount0, amount1) => {
  return executeAddLiquidity({
    token0: pool?.token0,
    token1: pool?.token1,
    amount0,
    amount1,
  })
}

// 提取流动性
const handleRemoveLiquidity = (lpAmount) => {
  return executeRemoveLiquidity({
    pool: pool,
    lpAmount,
  })
}
```

### Dos and Don'ts

**✅ Do:**
- 使用 `isSushiSwapV2ChainId` 验证 chainId
- 使用 `useMemo` 缓存复杂计算
- 结合 `useBalance` 获取用户头寸
- 使用 `useUnderlyingTokenBalanceFromPool` 计算用户真实代币

**❌ Don't:**
- 不要对无效的 chainId 调用
- 不要直接使用 reserve0/reserve1 进行计算，应该用 Amount 方法
- 不要忽略 `isLoading` 状态
- 不要假设 pool 数据总是存在，处理 null 情况
