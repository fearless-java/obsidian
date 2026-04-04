> 源代码路径: `apps/web/src/lib/hooks/react-query/pools/useConcentratedLiquidityPoolStats.ts`

# useConcentratedLiquidityPoolStats Hook Tutorial

## 1. 大白话讲讲这个hook的作用

`useConcentratedLiquidityPoolStats` 是用来获取某个特定 V3 流动性池子的详细统计信息的 Hook。这些信息包括：池子的交易手续费率（feeAmount）、池子的 TVL（总锁仓量）、当前流动性、Token 价格、以及该池子的激励奖励信息（incentives）等。

简单来说：**当你想要展示一个 SushiSwap V3 池子的详细信息（比如交易页面右上角的那个池子数据卡片），就用这个 Hook。**

---

## 2. 为什么需要封装该hook

### 原始问题
- V3 池子数据来自 Graph 的 Data API（`getV3Pool` *(一个从 The Graph 获取 V3 池子原始数据的 Graph Client 函数)*），这个函数返回的原始数据结构非常复杂，嵌套很深
- 原始数据中的 `swapFee` 是小数形式（如 0.003），需要乘以 1000000 转换为人类可读的手续费率（如 30 代表 0.03%）
- 原始数据中的 `incentives.rewardPerDay` 是字符串，需要转换为 `Amount` *(sushi 库中的代币金额类型，同时包含数 值和代币信息)* 类型的代币数量
- 每个用到池子数据的地方都需要手动做这些转换，容易出错

### 封装带来的好处
1. **数据预处理**：自动将原始数据转换为应用需要的格式，包括手续费计算、奖励金额封装
2. **定时刷新**：设置 `refetchInterval: 10000`（10秒），确保数据相对实时
3. **类型安全**：返回明确类型的统计数据，TypeScript 可以进行类型检查
4. **错误处理**：当池子不存在时优雅返回 `null` 而不是抛出异常

---

## 3. 执行逻辑和数据流向

### 输入 (Params)
```typescript
{
  chainId: SushiSwapV3ChainId | undefined   // 链ID
  address: EvmAddress | undefined            // 池子地址
  enabled?: boolean = true                  // 是否启用
}
```

### 输出 (Return)
```typescript
{
  data: {
    feeAmount: number              // 手续费率（basis points，如 30 = 0.03%）
    tvl: Amount<Currency>          // 总锁仓量
    liquidity: bigint              // 当前流动性
    token0: Token                  // Token0
    token1: Token                  // Token1
    incentives: {
      reward: Amount<Token>        // 每日奖励金额
      rewardPerDay: number
      pool: Address
    }[]
    // ... 其他 hydrateV3Pool 返回的字段
  } | null                         // 池子不存在时返回 null
  isLoading: boolean
  isError: boolean
  // ...
}
```

### 执行流程

```
1. useConcentratedLiquidityPoolStats({ chainId, address })
       |
       v
2. 检查 enabled && chainId && address
   - 不满足则不发起请求
       |
       v
3. 调用 getV3Pool({ chainId, address }) 获取原始池子数据
   - 这是 Graph Client 提供的函数
       |
       v
4. 如果 rawPool 存在:
   a) 执行 hydrateV3Pool(rawPool) 基础转换
   b) 计算 feeAmount = data.swapFee * 1000000
   c) 转换 incentives:
      - incentive.rewardPerDay (string) ->
      - new Amount(rewardToken, parseUnits(incentive.rewardPerDay, decimals))
       |
       v
5. 返回转换后的数据或 null
```

---

## 4. 怎么用 AI 写这个 Hook

### 写代码时要注意什么

1. **queryKey 要包含关键参数**：键名要能区分不同的池子，比如 `['useConcentratedLiquidityPoolStats', { address, chainId }]`。

2. **查不到数据不要报错**：池子不存在的时候返回 null 就好了，这是正常的业务情况，不是错误，别把它当错误处理。

3. **业务计算逻辑要写在一起**：像手续费率要乘以 1000000 这种转换，直接在 Hook 里面做好，别让外面的人自己算。

4. **定时刷新要合理**：V3 池子的数据变化比较快，10 秒刷新一次比较合适。

5. **钱相关的数字要用 Amount 类型**：代币金额不要用普通的 bigint 或 number，要用 `Amount` 类型来保证精度。

### 有什么限制条件

1. **只支持 V3 链**：chainId 必须是 SushiSwap V3 支持的那些链，不是所有链都行。

2. **依赖 Graph Client**：这个 Hook 底层用的是 `@sushiswap/graph-client/data-api` 的 `getV3Pool` 和 `hydrateV3Pool`，没有这个库就跑不起来。

3. **地址不能为空**：如果 address 或 chainId 是 undefined，就不会发请求，直接跳过。

4. **返回结构受限于 Graph Client**：最后返回什么字段，取决于 Graph Client 的 hydrateV3Pool 返回什么。

### 状态怎么管理

| 状态 | 怎么管 | 什么意思 |
|------|--------|---------|
| 池子数据 | React Query 缓存 | 用 queryKey 缓存，多个组件用同一个 Hook 会共享数据 |
| 加载状态 | React Query isLoading | 自动处理 |
| 错误状态 | React Query isError | getV3Pool 报错的时候会触发 |
| 实时更新 | refetchInterval | 每 10 秒自动重新拉数据 |
| 启用开关 | enabled 参数 | 手动控制要不要查 |

---

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useConcentratedLiquidityPoolStats } from '@sushiswap/react-query'

function PoolStatsCard({ chainId, poolAddress }: { chainId: number; poolAddress: string }) {
  const { data: poolStats, isLoading, isError } = useConcentratedLiquidityPoolStats({
    chainId,
    address: poolAddress,
  })

  if (isLoading) return <div>Loading...</div>
  if (isError) return <div>Error loading pool stats</div>
  if (!poolStats) return <div>Pool not found</div>

  return (
    <div>
      <div>Fee Tier: {(poolStats.feeAmount / 10000).toFixed(2)}%</div>
      <div>TVL: ${poolStats.tvl.toSignificant(2)}</div>
      <div>Incentives: {poolStats.incentives.length} pools</div>
    </div>
  )
}
```

### 常见使用场景

**场景1：交易页面池子信息卡片**
```tsx
function PoolInfoCard({ poolAddress }: { poolAddress: string }) {
  const { data: stats } = useConcentratedLiquidityPoolStats({
    chainId: 1, // Ethereum
    address: poolAddress,
  })

  return (
    <Card>
      <CardHeader>Pool Statistics</CardHeader>
      <CardBody>
        <FeeTier feeAmount={stats?.feeAmount} />
        <TVLDisplay tvl={stats?.tvl} />
        <LiquidityDisplay liquidity={stats?.liquidity} />
        <IncentivesList incentives={stats?.incentives} />
      </CardBody>
    </Card>
  )
}
```

**场景2：V3 池子列表页**
```tsx
function V3PoolList({ pools }: { pools: V3Pool[] }) {
  return (
    <div>
      {pools.map((pool) => (
        <PoolRow key={pool.address}>
          <PoolStatsBadge poolAddress={pool.address} chainId={pool.chainId} />
        </PoolRow>
      ))}
    </div>
  )
}

function PoolStatsBadge({ poolAddress, chainId }) {
  const { data: stats } = useConcentratedLiquidityPoolStats({ chainId, address: poolAddress })

  if (!stats) return null

  return (
    <div className="flex gap-2">
      <span>Fee: {(stats.feeAmount / 10000).toFixed(2)}%</span>
      <span>TVL: ${stats.tvl.toSignificant(2)}</span>
    </div>
  )
}
```

### Dos and Don'ts

**Do（推荐做法）：**
- ✅ 使用 `isLoading` 和 `isError` 处理加载和错误状态
- ✅ 池子不存在时返回 null，应该显示"Pool not found"而非报错
- ✅ 使用 `Amount` 类型的 `.toSignificant()` 或 `.toFixed()` 方法格式化显示
- ✅ 依赖 `refetchInterval: 10s` 自动刷新，不需要手动刷新

**Don't（避免做法）：**
- ❌ 不要在 poolStats 为 null 时访问其属性，应该先检查
- ❌ 不要设置过短的 refetchInterval，10秒已经足够实时
- ❌ 不要在没有 chainId 或 address 时传入 undefined，应该检查后再调用
- ❌ 不要直接显示 bigint 类型的 liquidity，应该转换为可读格式

### 注意事项

1. **feeAmount 是 basis points**：0.03% 的手续费存储为 3000，而不是 0.03。转换公式是 `feeAmount / 10000 = 百分比`

2. **池子不存在返回 null**：这不是错误情况，是正常的业务逻辑。当 pool address 正确但池子还没创建时就会返回 null

3. **10秒自动刷新**：这个 Hook 会自动每10秒刷新数据，不需要外部触发

4. **依赖 Graph Client**：这个 Hook 依赖 `@sushiswap/graph-client/data-api`，需要确保 Graph Client 配置正确
