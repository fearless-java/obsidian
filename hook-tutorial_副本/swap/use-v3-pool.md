> 源代码路径: `apps/web/src/lib/hooks/api/useV3Pool.ts`

# useV3Pool

## 1. 大白话讲讲这个hook的作用

`useV3Pool` *(一个React hook，用于获取SushiSwap V3集中流动性池的详细信息，包括手续费率、tick范围、流动性等)* 帮你获取SushiSwap V3池子的详细信息。

V3池子与V2不同：
- 有 Fee Tier（手续费率：0.05%, 0.3%, 1%）
- 有 Tick Range（价格区间）
- 流动性是集中式的

返回数据包括：
- token0、token1
- fee（手续费率）
- liquidity（流动性）
- tick（当前tick）
- liquidityUSD

## 2. 讲讲为什么需要封装该hook

封装提供：
- 简洁的查询接口
- 自动hydrate数据
- 条件获取支持

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
params: Partial<GetV3Pool>  // { chainId, address }
shouldFetch?: boolean
```

**输出：**
```typescript
{
  data: V3Pool | null,
  isLoading: boolean,
  error: Error | null
}
```

**执行逻辑：**
1. enabled = shouldFetch && chainId && address
2. 调用 `getV3Pool(params as GetV3Pool)` *(获取V3池子详情的GraphQL函数)*
3. hydrate V3池子数据
4. 返回

## 4. 怎么给这个hook写AI提示词

这个hook拿来查单个V3池子的详细信息。V3和V2不一样，V3有手续费率（fee tier）、有价格区间（tick range）、是集中流动性的。

### 写提示词的小技巧

**第一，chainId和address必须有。** 缺一个都没法查。

**第二，用shouldFetch控制。** 参数不全就不发请求。

### 提示词模板

```
帮我写一个React hook，功能是获取单个V3池子的详细信息。

具体需求：
1. 输入 chainId 和 address
2. 调用 getV3Pool 获取数据
3. 用 hydrateV3Pool 处理
4. 支持 shouldFetch 开关

返回：{ data: V3Pool | null, isLoading, error }
```

### 实际用的例子

```typescript
const { data: v3Pool, isLoading } = useV3Pool({
  chainId: ChainId.ETHEREUM,
  address: '0x...',
}, true)
```

V3池子的数据结构比V2复杂一些，多了fee（手续费率）、tick、tickUpper、tickLower这些字段。

### Production-Ready Example

```typescript
const { data: v3Pool, isLoading } = useV3Pool({
  chainId: ChainId.ETHEREUM,
  address: '0x...',
}, true)
```

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 基础调用
const { data: v3Pool, isLoading, error } = useV3Pool({
  chainId: ChainId.ETHEREUM,
  address: '0x...',
})

// v3Pool 包含V3池子的信息
console.log(v3Pool?.fee / 10000) // 手续费率 (e.g., 0.3%)
console.log(v3Pool?.tick) // 当前tick
console.log(v3Pool?.liquidityUSD) // 流动性USD价值
```

### 常见使用场景

**场景1：V3池子详情展示**

```typescript
const { data: v3Pool } = useV3Pool({ chainId, address })

return (
  <V3PoolInfo>
    <div>代币对: {v3Pool?.token0.symbol} / {v3Pool?.token1.symbol}</div>
    <div>手续费率: {v3Pool?.fee / 10000}%</div>
    <div>当前价格: {v3Pool?.token0Price}</div>
    <div>流动性: ${v3Pool?.liquidityUSD}</div>
    <div>当前Tick: {v3Pool?.tick}</div>
  </V3PoolInfo>
)
```

**场景2：创建V3流动性头寸**

```typescript
const { data: v3Pool } = useV3Pool({ chainId, address })

// 获取当前价格
const currentPrice = v3Pool?.token0Price

// 用户选择价格范围
const [tickLower, tickUpper] = useTickRange()

// 创建流动性头寸
const { data: zapData } = useV3Zap({
  chainId,
  pool: v3Pool,
  amountIn: inputAmount,
  slippage,
  ticks: [tickLower, tickUpper],
})
```

**场景3：V3 vs V2 池子比较**

```typescript
const { data: v2Pool } = useV2Pool({ chainId, address: v2Address })
const { data: v3Pool } = useV3Pool({ chainId, address: v3Address })

return (
  <PoolComparison>
    <div>
      <h3>V2池子</h3>
      <div>手续费: 0.3% (固定)</div>
      <div>流动性: ${v2Pool?.liquidityUSD}</div>
    </div>
    <div>
      <h3>V3池子</h3>
      <div>手续费: {v3Pool?.fee / 10000}%</div>
      <div>流动性: ${v3Pool?.liquidityUSD}</div>
      <div>Tick范围: {v3Pool?.tickLower} - {v3Pool?.tickUpper}</div>
    </div>
  </PoolComparison>
)
```

### Dos and Don'ts

**✅ Do:**
- 确保 chainId 和 address 都存在后再调用
- 使用 `shouldFetch` 控制何时获取
- 处理 `isLoading` 和 `error` 状态
- V3池子有集中流动性特性，选择合适的tick范围很重要

**❌ Don't:**
- 不要在 chainId 或 address 缺失时发起请求
- 不要忽略 V3 池子的 fee tier（手续费率）差异
- 不要混淆 V2 和 V3 的数据结构，它们不同
- 不要忽略 tick 范围，这是 V3 的核心概念
