> 源代码路径: `apps/web/src/lib/hooks/usePoolsByTokenPair.ts`

# usePoolsByTokenPair

## 1. 大白话讲讲这个hook的作用

`usePoolsByTokenPair` *(一个React hook，用于根据两个代币对查询SushiSwap V3所有可用的流动性池)* 帮你查找两个代币之间所有可用的SushiSwap V3流动性池。

比如你想知道 ETH 和 USDT 之间有哪些池子，这个hook会：
1. 接收两个代币的ID
2. 调用图数据库查询
3. 返回所有符合条件的V3池子列表

## 2. 讲讲为什么需要封装该hook

直接调用图数据库很麻烦：
- 需要处理图查询的复杂性
- 需要处理加载状态、错误状态
- 简单的 `enabled` 逻辑（两个token都必须存在）会重复

封装后：
- 提供简洁的接口
- 自动处理加载状态
- 与react-query集成，支持缓存

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
tokenId0?: EvmID   // 第一个代币ID，格式 "chainId:address"
tokenId1?: EvmID   // 第二个代币ID
```

**输出：**
```typescript
{
  data: Pool[],      // 池子数组
  isLoading: boolean,
  error: Error | null
}
```

**执行逻辑：**
1. 如果任一token不存在，直接返回空数组（`enabled: false`）
2. 调用 `getV3PoolsByTokenPair(tokenId0, tokenIdId)` *(查询两个代币之间所有V3池子的GraphQL函数)* 查询
3. 返回结果

**数据流：**
```
tokenId0 + tokenId1 --> getV3PoolsByTokenPair --> 图数据库
                                               |
                                               v
                                          Pool数组 --> 组件使用
```

## 4. 怎么给这个hook写AI提示词

这个hook干的事儿就是：给你两个代币，找出它们之间所有可用的SushiSwap V3池子。比如查 ETH 和 USDT，会返回所有费率（0.05%、0.3%、1%）的ETH/USDT池子。

### 写提示词的小技巧

**第一，两个token都得有才行。** 只有一个token的话没法确定交易对，查询也没意义。所以 enabled 条件要写成 `!!tokenId0 && !!tokenId1`，两个都有才执行。

**第二，用queryKey管好缓存。** React Query会根据queryKey缓存结果。同样的查询重复发的话，直接拿缓存就行，不用重新请求。

**第三，错误要让调用方知道。** 查图数据库可能失败，失败了要把错误抛上去，别吞掉。

### 写提示词时要注意的条条框框

**tokenId格式是"chainId:address"：** 不能只传地址，必须带上链ID。比如 "1:0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"，1代表以太坊主网。

**任一token缺失就不发请求：** 这是硬条件，写在enabled里。

**返回的是数组：** 可能有一个或多个池子（不同费率）。

### 提示词模板

```
帮我写一个React hook，功能是根据两个代币查找它们之间的所有V3池子。

具体需求：
1. 输入两个代币ID，格式是 "chainId:address"
2. 调用图数据库查询这两个代币之间的所有池子
3. 只有两个token都存在时才执行查询
4. 返回池子数组

返回类型：
{
  data: Pool[],
  isLoading: boolean,
  error: Error | null
}

注意事项：
- enabled 条件要写成 !!tokenId0 && !!tokenId1
- 调用 getV3PoolsByTokenPair 函数
- 用 useQuery 包装
```

### 实际用的例子

```typescript
const tokenId0 = '1:0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2' // WETH
const tokenId1 = '1:0xdAC17F958D2ee523a2206206994597C13D831ec7' // USDT

const { data: pools, isLoading } = usePoolsByTokenPair(tokenId0, tokenId1)
```

这就是查以太坊上 WETH/USDT 的所有V3池子。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 构建tokenId格式: "chainId:address"
const tokenId0 = `${chainId}:${token0.address}`
const tokenId1 = `${chainId}:${token1.address}`

// 查询池子
const { data: pools, isLoading, error } = usePoolsByTokenPair(tokenId0, tokenId1)

// pools 是 Pool[] 类型，包含所有符合条件的V3池子
pools?.forEach(pool => {
  console.log(`Fee: ${pool.fee / 10000}%`) // V3手续费率
  console.log(`Tick: ${pool.tick}`) // 当前tick
})
```

### 常见使用场景

**场景1：选择最佳池子进行交易**

```typescript
const { data: pools } = usePoolsByTokenPair(tokenId0, tokenId1)

// 按手续费率排序（通常选择最低费率的池子）
const sortedPools = pools?.sort((a, b) => a.fee - b.fee)
const bestPool = sortedPools?.[0]

// 选择手续费最低且流动性最好的池子
const optimalPool = pools?.reduce((best, pool) => {
  if (!best || pool.liquidity > best.liquidity) return pool
  return best
}, null as Pool | null)
```

**场景2：展示所有可用池子**

```typescript
const { data: pools, isLoading } = usePoolsByTokenPair(tokenId0, tokenId1)

// 显示池子列表
return (
  <PoolList>
    {pools?.map(pool => (
      <PoolRow
        key={pool.id}
        pool={pool}
        fee={pool.fee / 10000}
        liquidity={pool.liquidity}
      />
    ))}
  </PoolList>
)
```

**场景3：获取特定手续费率的池子**

```typescript
const { data: pools } = usePoolsByTokenPair(tokenId0, tokenId1)

// 找到0.3%手续费的池子（最常见的V3池子）
const pool_03 = pools?.find(p => p.fee === 3000) // 3000 = 0.3% in basis points
const pool_05 = pools?.find(p => p.fee === 500) // 500 = 0.05%
const pool_1 = pools?.find(p => p.fee === 10000) // 10000 = 1%
```

### Dos and Don'ts

**✅ Do:**
- 确保 tokenId0 和 tokenId1 都存在后再调用 hook
- 使用合适的 queryKey 便于缓存管理
- 按手续费率或流动性排序选择最优池子
- 处理 `isLoading` 状态显示加载指示器

**❌ Don't:**
- 不要在 tokenId 不完整时发起请求
- 不要忽略 `error` 状态，应该展示错误信息
- 不要假设只有一个池子，V3 可能有多个手续费率的池子
- 不要忽略池子的流动性，低流动性池子可能滑点很大
