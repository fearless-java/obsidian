> 源代码路径: `apps/web/src/lib/wagmi/hooks/pools/hooks/useConcentratedLiquidityPool.ts`

# useConcentratedLiquidityPool

## 大白话讲讲这个hook的作用

`useConcentratedLiquidityPool` 是一个查询 SushiSwap V3 ( Concentrated Liquidity ) 池子信息的 Hook。

大白话：就是查询"某个交易对在某个费率等级下的池子信息"。V3 的特点是流动性集中在特定价格区间，所以需要指定 token0、token1 和 feeAmount。

它会调用 `getConcentratedLiquidityPool` 获取：
- 池子地址
- 当前价格 (sqrtPrice)
- 流动性
- Tick 信息
- 手续费率

## 讲讲为什么需要封装该hook

1. **V3 池子计算复杂**：V3 池子地址通过 factory 计算，不是直接已知。

2. **价格转换**：sqrtPriceX96 需要转换成人类可读的价格。

3. **费率等级**：V3 有 0.05%, 0.3%, 1% 等多种费率池。

4. **缓存优化**：使用 React Query 缓存池子数据。

5. **定时刷新**：V3 池子价格变化快，需要定时刷新。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseConcentratedLiquidityPool {
  token0: EvmCurrency | undefined    // 币对 A
  token1: EvmCurrency | undefined    // 币对 B
  chainId: SushiSwapV3ChainId        // 链 ID
  feeAmount: SushiSwapV3FeeAmount | undefined  // 费率等级
  enabled?: boolean
}
```

### FeeAmount 枚举

```typescript
enum SushiSwapV3FeeAmount {
  LOW = 500,      // 0.05%
  MEDIUM = 3000,  // 0.3%
  HIGH = 10000,   // 1%
}
```

### 输出 (Outputs)

```typescript
{
  data: SushiSwapV3Pool | null,  // 池子信息
  isLoading: boolean,
  isError: boolean,
}
```

### 执行逻辑详解

#### 1. 查询函数

```typescript
const { config } = useConfig()

return useQuery({
  queryKey: ['useConcentratedLiquidityPool', { chainId, token0, token1, feeAmount }],
  queryFn: async () => {
    if (!token0 || !token1 || !feeAmount) throw new Error()
    return getConcentratedLiquidityPool({
      chainId,
      token0,
      token1,
      feeAmount,
      config,
    })
  },
  refetchInterval: 10000,  // 10 秒刷新
  enabled: Boolean(enabled && feeAmount && chainId && token0 && token1),
})
```

#### 2. getConcentratedLiquidityPool

实际逻辑在 `../actions/getConcentratedLiquidityPool`，会：
- 计算池子地址
- 获取池子的实时数据
- 返回 SushiSwapV3Pool 实例

### 数据流向图

```
输入: token0, token1, feeAmount, chainId
         ↓
    getConcentratedLiquidityPool 计算
    ┌────────────────────────────────────┐
    │  1. computePoolAddress            │
    │  2. getPoolData (sqrtPrice, etc)  │
    │  3. 创建 SushiSwapV3Pool 实例      │
    └────────────────────────────────────┘
         ↓
    缓存到 queryKey
         ↓
    返回 SushiSwapV3Pool
         ↑
    refetchInterval: 10000 定时刷新
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：参数必须完整**。token0、token1、feeAmount、chainId这四个参数缺一不可。任何 一个没提供都无法查到正确的池子。所以enabled的判断要同时检查这四个参数都有值才行。

**第二点：费率选择有讲究**。V3池子有三种费率：0.05%、0.3%、1%。费率越低的手续费越少，但流动性集中的地方价差可能更大。不同交易对适合不同费率：主流币对适合低费率，波动大的交易对适合高费率。

**第三点：10秒刷新是平衡之选**。刷新太快会增加RPC消耗，太慢数据又不够实时。10秒是一个比较合理的间隔，既能保持数据相对新鲜，又不会对节点造成太大压力。

**第四点：池子不存在返回null**。如果输入的参数对应的池子根本不存在，hook不会报错，而是返回null。这是一个温和的处理方式，调用方检查一下返回数据是否为null就知道池子存不存在了。

**第五点：feeAmount要用枚举**。feeAmount不能随便传数字，必须用SushiSwapV3FeeAmount这个枚举里的值。LOW是500代表0.05%，MEDIUM是3000代表0.3%，HIGH是10000代表1%。

### 约束条件要记住

**第一，V3只在特定链**。V3版本的池子不是所有链都有，主要在以太坊主网和Polygon等链上。如果你在一个不支持的链上调用，会返回null。

**第二，token0和token1不能相同**。你自己不能跟自己做交易对。如果传了相同的代币，查询会失败或者返回null。

**第三，feeAmount必须匹配**。池子在创建时就定好了费率。你查询的时候必须指定正确的费率，才能找到对应的池子。同一个交易对可能有多个费率的池子。

**第四，池子可能不存在**。并不是所有token0+token1+feeAmount组合都有对应的池子。如果没有池子，返回null而不是报错。

### 状态管理要清楚

查询状态有三种。isLoading表示初始加载中，isFetching表示正在后台刷新，isError表示出错了。data是池子数据，如果池子不存在就是null。

刷新方式有两种。一种是定时刷新，每10秒自动查一次。另一种是手动调用refetch函数触发刷新。

### 常见错误要避开

**第一个坑：池子不存在不是错误**。如果传入的参数对应的池子不存在，hook返回的是null而不是抛出异常。所以调用方要养成检查返回数据的好习惯。

**第二个坑：价格精度问题**。V3的价格是sqrtPriceX96格式，这是个很大的整数。直接显示给用户是不行的，需要转换成人类可读的价格格式。

**第三个坑：Tick范围的影响**。V3的流动性是有价格范围的，超出范围的仓位在当前价格下可能是无效的。这会影响显示的流动性数量。

**第四个坑：Flash SWAP的影响**。V3支持闪电贷功能，这可能导致池子的状态在短时间内剧烈变化。如果你在短时间内多次查询同一池子，可能会看到不同的数据。

**第五个坑：同交易对多池子**。同一个token0+token1组合可能有多个费率的池子。比如USDC/WETH有0.05%费率的池子，也有0.3%费率的池子。查询时要指定feeAmount才能精确定位。

### 提示词模板

```markdown
帮我创建一个查询SushiSwap V3池子信息的hook。

需求：
- 根据token0、token1、feeAmount查找池子
- 获取池子的价格、流动性等实时数据
- 定时刷新保持数据新鲜

技术栈：
- React + TypeScript
- wagmi v2 + viem
- @tanstack/react-query v5
- sushi库

费率枚举：
- LOW = 500，0.05%费率
- MEDIUM = 3000，0.3%费率
- HIGH = 10000，1%费率

输入：
- token0：代币A
- token1：代币B
- chainId：链ID
- feeAmount：费率等级
- enabled：是否启用

输出：
- data：池子信息，为null表示不存在
- isLoading：是否加载中
- isError：是否出错

实现要点：
1. 用getConcentratedLiquidityPool获取池子
2. 缓存key包含所有参数
3. 每10秒定时刷新
4. 池子不存在返回null

注意：
- 四个参数缺一不可
- token0和token1不能相同
- 费率必须匹配池子创建时的设置
- V3只在特定链上可用
```

### 实际避坑指南

第一个，null不是错误。如果池子不存在，不要以为程序出错了。这是正常的情况，调用方检查一下data是否为null即可。

第二个，价格转换。V3的sqrtPriceX96不能直接显示，要用sushi库提供的工具转换成可读格式。比如toSignificant(6)保留6位有效数字。

第三个，多池子查询。如果你想查一个交易对的所有费率池子，要分别查每个feeAmount对应的池子。V3支持多费率池子是它的特色。

第四个，链上查询限制。不是所有链都支持V3。在使用前最好检查一下当前链是否在支持列表里。

2. **价格精度**：V3 的价格是 sqrtPriceX96，需要特殊转换。

3. **Tick 范围**：流动性有上下界，超出范围的仓位可能无效。

4. **Flash SWAP**：V3 支持 flash swap，可能影响价格。

5. **多池子**：同一交易对可能有多个费率的池子，需要分别查询。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useConcentratedLiquidityPool` 用于查询 SushiSwap V3 池子信息。基本用法如下：

```typescript
import { useConcentratedLiquidityPool, SushiSwapV3FeeAmount } from '@sushiswap/wag'

function PoolInfo({ token0, token1, chainId }) {
  const { data: pool, isLoading } = useConcentratedLiquidityPool({
    token0,
    token1,
    chainId,
    feeAmount: SushiSwapV3FeeAmount.MEDIUM, // 0.3% 费率
  })

  if (isLoading) return <div>加载中...</div>
  if (!pool) return <div>池子不存在</div>

  return (
    <div>
      <div>池子地址: {pool.address}</div>
      <div>当前价格: {pool.price?.toSignificant(6)}</div>
      <div>流动性: {pool.liquidity.toString()}</div>
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景一：显示交易价格

```typescript
function SwapPrice({ token0, token1, chainId }) {
  const { data: pool } = useConcentratedLiquidityPool({
    token0,
    token1,
    chainId,
    feeAmount: SushiSwapV3FeeAmount.MEDIUM,
  })

  if (!pool) return <div>无池子</div>

  // pool.price 是 token1/token0 的价格
  return (
    <div>
      1 {token0.symbol} = {pool.price?.toSignificant(6)} {token1.symbol}
    </div>
  )
}
```

#### 场景二：选择最优费率池

```typescript
function BestPool({ token0, token1, chainId }) {
  const lowPool = useConcentratedLiquidityPool({
    token0, token1, chainId,
    feeAmount: SushiSwapV3FeeAmount.LOW,
  })

  const mediumPool = useConcentratedLiquidityPool({
    token0, token1, chainId,
    feeAmount: SushiSwapV3FeeAmount.MEDIUM,
  })

  const highPool = useConcentratedLiquidityPool({
    token0, token1, chainId,
    feeAmount: SushiSwapV3FeeAmount.HIGH,
  })

  // 选择有流动性的最低费率池
  const bestPool = lowPool.data || mediumPool.data || highPool.data

  return <div>最优池子费率: {bestPool?.feeAmount / 10000}%</div>
}
```

#### 场景三：获取池子进行交易

```typescript
function TradePool({ token0, token1, chainId, amount }) {
  const { data: pool } = useConcentratedLiquidityPool({
    token0,
    token1,
    chainId,
    feeAmount: SushiSwapV3FeeAmount.MEDIUM,
  })

  const { data: reserves } = useConcentratedLiquidityPoolReserves({
    pool,
    chainId,
  })

  if (!pool || !reserves) return <div>加载中...</div>

  return (
    <div>
      <div>可用 {token0.symbol}: {reserves.amount0.toSignificant(6)}</div>
      <div>可用 {token1.symbol}: {reserves.amount1.toSignificant(6)}</div>
      <SwapButton pool={pool} />
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **确保参数完整**
   ```typescript
   // 缺少任何一个参数都不会查询
   enabled: Boolean(token0 && token1 && feeAmount && chainId)
   ```

2. **处理池子不存在的情况**
   ```typescript
   if (!pool) {
     return <div>该费率池子不存在</div>
   }
   ```

3. **使用正确的 FeeAmount**
   ```typescript
   // 不要用数字直接传入
   feeAmount: 3000 // 错误

   // 正确
   feeAmount: SushiSwapV3FeeAmount.MEDIUM
   ```

#### ❌ Don'ts

1. **不要假设池子一定存在**
   ```typescript
   // 错误
   const price = pool.price

   // 正确
   if (!pool) return <NoPool />
   const price = pool.price
   ```

2. **不要用相同的 token**
   ```typescript
   // 错误：V3 要求 token0 !== token1
   token0: USDC,
   token1: USDC,

   // 正确
   token0: USDC,
   token1: WETH,
   ```

3. **不要忽略定时刷新**
   ```typescript
   // 10 秒刷新是合理的
   // 不要改成过短的间隔（如 1 秒）
   ```

4. **不要在不同链间混用池子**
   ```typescript
   // 错误：token 和 chainId 必须匹配
   token0: TOKEN_ON_ETHEREUM,
   chainId: EvmChainId.POLYGON, // 错误
   ```
