> 源代码路径: `apps/web/src/lib/wagmi/hooks/positions/hooks/useConcentratedLiquidityPositions.ts`

# useConcentratedLiquidityPositions

## 大白话讲讲这个hook的作用

`useConcentratedLiquidityPositions` 是一个查询用户在 SushiSwap V3 所有流动性仓位的 Hook。

大白话：就是"查查我在 V3 提供了哪些流动性，分别是多少钱"。它会返回用户所有活跃的 V3 仓位，包括：
- 仓位所在的池子信息
- 质押的 token0 和 token1 数量
- 手续费收益（未领取的）
- 仓位的美元价值

## 讲讲为什么需要封装该hook

1. **复杂数据聚合**：需要从多个数据源聚合数据（Positions + Pools + Prices）。

2. **USD 价值计算**：需要结合价格数据计算仓位的美元价值。

3. **多链支持**：需要跨多个链查询仓位。

4. **缓存优化**：数据量大，需要合理的缓存策略。

5. **自定义 Token**：支持用户添加的自定义 Token。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseConcentratedLiquidityPositionsParams {
  account: Address | undefined       // 钱包地址
  chainIds: readonly SushiSwapV3ChainId[]  // 要查询的链
  enabled?: boolean
}
```

### 输出 (Outputs)

```typescript
{
  data: UseConcentratedLiquidityPositionsData[] | undefined,
  isError: boolean,
  isInitialLoading: boolean,
}
```

### 执行逻辑详解

#### 1. 查询仓位数据

```typescript
const { data: positions } = useQuery({
  queryKey: ['useConcentratedLiquidityPositions', { chainIds, account, prices }],
  queryFn: async () => {
    const positions = await getConcentratedLiquidityPositions({
      account,
      chainIds,
      config,
    })

    // 获取 token0/token1 的完整信息
    const positionsWithTokens = await Promise.all(
      positions.map(async (position) => {
        const [token0Data, token1Data] = await Promise.all([
          getTokenWithCacheQueryFn({ chainId, hasToken, customTokens, address: position.token0, config }),
          getTokenWithCacheQueryFn({ chainId, hasToken, customTokens, address: position.token1, config }),
        ])
        return { ...position, token0: token0Data, token1: token1Data }
      })
    )

    // 获取池子信息
    const pools = await getConcentratedLiquidityPools({ poolKeys, config })

    // 计算 USD 价值
    // ...
  },
  refetchInterval: Number.POSITIVE_INFINITY,  // 不自动刷新
  enabled: Boolean(account && chainIds && enabled && (!isPriceInitialLoading || isPriceError)),
})
```

#### 2. 计算仓位价值

```typescript
const amountToUsd = (amount: Amount<EvmToken>) => {
  const _price = prices?.get(chainId)?.get(amount.currency.address)
  if (!amount?.gt(0n) || !_price) return 0
  const price = Number(Number(amount.toString()) * Number(_price.toFixed(10)))
  return price
}

const positionUSD = amountToUsd(position.amount0) + amountToUsd(position.amount1)
const unclaimedUSD = amountToUsd(fees[0]) + amountToUsd(fees[1])
```

### 数据流向图

```
输入: account, chainIds
         ↓
    ┌────────────────────────────────────┐
    │  getConcentratedLiquidityPositions │
    │  (从 subgraph 或链上查询)            │
    └────────────────────────────────────┘
         ↓
    ┌────────────────────────────────────┐
    │  getTokenWithCacheQueryFn          │
    │  (获取完整 token 信息)               │
    └────────────────────────────────────┘
         ↓
    ┌────────────────────────────────────┐
    │  getConcentratedLiquidityPools     │
    │  (获取池子数据)                     │
    └────────────────────────────────────┘
         ↓
    ┌────────────────────────────────────┐
    │  计算 positionUSD + unclaimedUSD   │
    │  (结合价格计算美元价值)             │
    └────────────────────────────────────┘
         ↓
    返回仓位列表 + 价值数据
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：USD价值依赖价格数据**。仓位的美元价值是通过价格数据计算出来的。如果价格数据还没加载完成，USD价值就无法计算，可能显示为0或者loading状态。UI要处理这种情况。

**第二点：仓位数据不自动刷新**。这个hook设置的是refetchInterval: Infinity，意味着不会自动定时刷新。仓位变化不频繁，用户主动刷新就够了。

**第三点：多链数据汇总**。用户可能在多条链上都有V3仓位。这个hook支持同时查询多个链的仓位，返回的数据会汇总在一起。

**第四点：支持自定义代币**。用户可能添加了一些 sushi 库里没有的代币。hook 要能处理这些自定义代币，需要额外的查询逻辑。

**第五点：过滤无效仓位**。如果一个仓位对应的池子已经被销毁或者不存在了，这个仓位就是无效的，应该过滤掉不显示。

### 约束条件要记住

**第一，依赖subgraph或链上枚举**。仓位数据要么从subgraph查询，要么从链上枚举获取。不同链的支持情况可能不同。

**第二，价格可能未加载**。USD价值依赖价格数据。如果价格查询还在进行中，positionUSD可能是0或者undefined。

**第三，自定义代币要额外处理**。用户添加的代币不在默认列表里，需要单独查询代币信息，处理起来更复杂。

**第四，Infinity间隔不刷新**。设置refetchInterval为Infinity表示不自动刷新。用户想看最新数据需要手动触发refetch。

### 状态管理要清楚

数据来自三个地方：subgraph提供仓位列表，链上查询提供token详情，价格API提供USD价值。

状态有两种。isInitialLoading表示初始加载，可能还在等价格数据。isError表示任何数据源出错了。data是完整的仓位数组。

刷新需要手动触发。因为设置的是Infinity间隔，不会自动刷新。用户可以通过UI按钮主动刷新数据。

### 常见错误要避开

**第一个坑：价格和仓位的时序**。价格数据可能比仓位数据先加载完成。这时候USD价值可能还无法计算。UI要处理这种情况，不能假设data返回时USD价值就一定有效。

**第二个坑：subgraph有延迟**。subgraph不是实时的，从链上到subgraph可能有延迟。链上最新的仓位变动可能不会立即出现在subgraph查询结果里。

**第三个坑：自定义代币查询**。用户添加的代币如果不在默认列表里，需要走额外的查询逻辑。这部分逻辑比较复杂，可能需要更多时间。

**第四个坑：USD精度损失**。计算USD价值时涉及多次转换，从bigint到Amount再到number，可能有精度损失。大额仓位要特别注意。

**第五个坑：tick范围的理解**。V3的仓位有tickLower和tickUpper，代表价格范围。当前价格落在这个范围内时仓位才是有效的。UI可以给用户展示这个信息。

### 提示词模板

```markdown
帮我创建一个查询用户所有V3流动性仓位的hook。

需求：
- 查询用户在多条链上的所有V3仓位
- 获取每个仓位的详细信息
- 计算USD价值和未领手续费
- 支持用户自定义的代币

技术栈：
- React + TypeScript
- wagmi v2 + viem
- @tanstack/react-query v5
- sushi库
- subgraph数据

输入：
- account：钱包地址
- chainIds：要查询的链ID数组
- enabled：是否启用

输出：
- data：仓位数组，每个包含详细信息
- isError：是否出错
- isInitialLoading：是否初始加载中

每个仓位包含：
- chainId：所在链
- token0、token1：两种代币信息
- pool：池子信息
- liquidity：流动性数量
- tickLower、tickUpper：价格范围
- fees：未领手续费
- positionUSD：美元价值
- unclaimedUSD：未领手续费的美元价值

实现要点：
1. 从subgraph获取仓位列表
2. 获取每个仓位的完整代币信息
3. 获取对应池子数据
4. 结合价格计算USD价值
5. 过滤无效仓位

注意：
- USD价值依赖价格数据
- 不自动刷新，需手动refetch
- 支持多链汇总
- 处理自定义代币
- 过滤无效仓位
```

### 实际避坑指南

第一个，耐心等待加载。初始加载时可能要等价格数据也查完才能看到USD价值。UI应该显示loading状态而不是错误。

第二个，理解subgraph延迟。如果用户刚操作完仓位变化，subgraph可能还没同步过来。让用户知道这是正常的。

第三个，不自动刷新的设计。仓位数据不像价格那样频繁变化，设置为Infinity是合理的。用户想看最新数据就手动点刷新按钮。

第四个，大额仓位的精度。计算USD价值时如果数字太大，可能有精度损失。如果对精度要求很高，可能需要特殊处理。

3. **自定义代币**：用户添加的 token 不在默认列表，需要额外查询。

4. **USD 精度**：计算 USD 时可能丢失精度，大额仓位尤其需要注意。

5. **Tick 范围**：V3 仓位的 tick 范围影响是否在当前价格范围内。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useConcentratedLiquidityPositions` 用于查询用户所有 V3 仓位。基本用法如下：

```typescript
import { useConcentratedLiquidityPositions } from '@sushiswap/wag'

function MyPositions({ account }) {
  const { data: positions, isLoading, isError } = useConcentratedLiquidityPositions({
    account,
    chainIds: [EvmChainId.ETHEREUM, EvmChainId.POLYGON],
    enabled: Boolean(account),
  })

  if (isLoading) return <div>加载中...</div>
  if (isError) return <div>查询失败</div>
  if (!positions?.length) return <div>暂无仓位</div>

  return (
    <div>
      {positions.map((position) => (
        <PositionCard key={position.id} position={position} />
      ))}
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景一：显示所有仓位列表

```typescript
function PositionsList({ account }) {
  const { data: positions } = useConcentratedLiquidityPositions({
    account,
    chainIds: supportedChainIds,
  })

  const totalValue = positions?.reduce(
    (sum, pos) => sum + (pos.position?.positionUSD || 0),
    0
  ) || 0

  return (
    <div>
      <div>总价值: ${totalValue.toFixed(2)}</div>
      {positions?.map((pos) => (
        <PositionRow key={pos.id} position={pos} />
      ))}
    </div>
  )
}
```

#### 场景二：显示仓位详情

```typescript
function PositionDetail({ position }) {
  return (
    <div>
      <div>
        {position.token0.symbol} / {position.token1.symbol}
      </div>
      <div>费率: {position.pool.feeAmount / 10000}%</div>
      <div>
        范围: {(position.tickLower / 1.0001 ** 1).toFixed(0)} -
        {(position.tickUpper / 1.0001 ** 1).toFixed(0)}
      </div>
      <div>流动性: {position.liquidity.toString()}</div>
      <div>仓位价值: ${position.position?.positionUSD.toFixed(2)}</div>
      <div>待领手续费: ${position.position?.unclaimedUSD.toFixed(2)}</div>
    </div>
  )
}
```

#### 场景三：增加流动性

```typescript
function IncreasePosition({ position }) {
  // 使用 position 的池子信息
  const { pool } = position

  const { write, isPending } = useIncreaseLiquidity({
    pool,
    tickLower: position.tickLower,
    tickUpper: position.tickUpper,
    amount0: amountToAdd0,
    amount1: amountToAdd1,
  })

  return (
    <button onClick={write} disabled={isPending}>
      增加流动性
    </button>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **处理加载状态**
   ```typescript
   if (isInitialLoading) return <Skeleton />

   // 价格可能还在加载，但仓位数据已返回
   ```

2. **多链查询**
   ```typescript
   // 可以同时查询多个链的仓位
   chainIds: [EvmChainId.ETHEREUM, EvmChainId.POLYGON, EvmChainId.ARBITRUM]
   ```

3. **处理价格依赖**
   ```typescript
   // USD 价值依赖价格数据
   // 如果价格没加载，positionUSD 可能是 0 或 undefined
   const value = position.position?.positionUSD || 0
   ```

#### ❌ Don'ts

1. **不要期望实时更新**
   ```typescript
   // refetchInterval: Infinity 不自动刷新
   // 需要手动 refetch
   const { refetch } = useConcentratedLiquidityPositions({ ... })

   // 或者用户主动刷新时才更新
   ```

2. **不要忽略 chainId**
   ```typescript
   // 必须指定要查询哪些链
   chainIds: [EvmChainId.ETHEREUM] // 只查以太坊
   ```

3. **不要假设所有链都支持**
   ```typescript
   // V3 不是所有链都部署了
   // 需要检查 supportedChainIds
   ```

4. **不要忽略仓位范围**
   ```typescript
   // 检查仓位是否在当前价格范围内
   // 超出范围的仓位不能直接交易
   const inRange = currentTick >= tickLower && currentTick <= tickUpper
   ```
