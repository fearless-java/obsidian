> 源代码路径: `apps/web/src/lib/hooks/api/useSushiV2UserPositions.ts`

# useSushiV2UserPositions

## 1. 大白话讲讲这个hook的作用

`useSushiV2UserPositions` *(一个React hook，用于获取用户在SushiSwap V2上的所有流动性头寸信息)* 帮你获取用户在SushiSwap V2上的所有流动性头寸。

头寸信息包括：
- 用户在哪些池子有存款
- 每个池子的存款金额
- 对应的代币数量
- 价值（USD）

## 2. 讲讲为什么需要封装该hook

封装提供：
- 简洁的参数接口
- 条件获取支持
- 类型安全的结果

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
{
  chainId: EvmChainId     // 链ID
  user: Address           // 用户地址
  ...其他GetV2Positions参数
}
shouldFetch?: boolean
```

**输出：**
```typescript
{
  data: V2Position[],     // V2头寸数组
  isLoading: boolean,
  error: Error | null
}
```

**执行逻辑：**
1. 调用 `getV2Positions(args as GetV2Positions)` *(获取用户V2流动性头寸的GraphQL函数)*
2. enabled = shouldFetch && chainId && user
3. 返回结果

## 4. 怎么给这个hook写AI提示词

这个hook拿来查用户在SushiSwap V2上的所有流动性头寸。你在哪些池子存了钱、每个池子存了多少、现在值多少美金，都能查到。

### 写提示词的小技巧

**第一，chainId和user必须有。** 缺任何一个都没法查，要么缺链ID不知道去哪查，要么缺地址不知道查谁。

**第二，用shouldFetch控制。** 比如用户没连接钱包，或者页面根本不是显示持仓的，那就别发请求了。

### 提示词模板

```
帮我写一个React hook，功能是获取用户在SushiSwap V2上的所有流动性头寸。

具体需求：
1. 调用 getV2Positions 获取数据
2. chainId 和 user 是必需参数，必须有
3. 支持 shouldFetch 开关

返回：{ data: V2Position[], isLoading, error }
```

### 实际用的例子

```typescript
const { data: positions } = useSushiV2UserPositions({
  chainId: ChainId.ETHEREUM,
  user: account,
})
```

拿到数组之后就可以遍历显示每个头寸的详情了。

### Production-Ready Example

```typescript
const { data: positions } = useSushiV2UserPositions({
  chainId: ChainId.ETHEREUM,
  user: account,
})
```

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 基础调用
const { data: positions, isLoading, error } = useSushiV2UserPositions({
  chainId: ChainId.ETHEREUM,
  user: account,
})

// positions 是 V2Position[] 类型
console.log(positions?.length) // 头寸数量
positions?.forEach(position => {
  console.log(`池子: ${position.pool.id}`)
  console.log(`LP数量: ${position.balance}`)
  console.log(`价值: $${position.liquidityUSD}`)
})
```

### 常见使用场景

**场景1：用户资产总览**

```typescript
const { data: positions } = useSushiV2UserPositions({
  chainId,
  user: account,
})

// 计算总资产
const totalValue = positions?.reduce(
  (sum, pos) => sum + (pos.liquidityUSD ?? 0),
  0
)

return (
  <Portfolio>
    <TotalValue value={totalValue} />
    {positions?.map(pos => (
      <PositionCard key={pos.pool.id} position={pos} />
    ))}
  </Portfolio>
)
```

**场景2：提取流动性**

```typescript
const { data: positions } = useSushiV2UserPositions({...})

// 选择要提取的头寸
const handleWithdraw = (position) => {
  // 计算提取的代币数量
  const { token0Amount, token1Amount } = getUnderlyingTokens(position)
  showConfirmDialog({
    title: '提取流动性',
    message: `你将获得 ${token0Amount} ${token0.symbol} 和 ${token1Amount} ${token1.symbol}`,
    onConfirm: () => executeWithdraw(position.pool.id),
  })
}
```

**场景3：收益历史**

```typescript
const { data: positions } = useSushiV2UserPositions({...})

// 显示每个头寸的年化收益率
return (
  <div>
    {positions?.map(pos => (
      <PositionRow key={pos.pool.id}>
        <PoolInfo pool={pos.pool} />
        <APR apr={pos.feeApr} />
        <Earnings feeEarned={pos.feeEarned} />
      </PositionRow>
    ))}
  </div>
)
```

### Dos and Don'ts

**✅ Do:**
- 确保 chainId 和 user 都存在后再调用
- 使用 `shouldFetch` 控制何时获取
- 处理 `isLoading` 状态
- 合并多个头寸计算总资产

**❌ Don't:**
- 不要在 chainId 或 user 缺失时发起请求
- 不要假设用户一定有头寸，检查 `positions?.length`
- 不要忽略 `error` 处理
- 不要缓存太久，用户的头寸状态可能随时变化
