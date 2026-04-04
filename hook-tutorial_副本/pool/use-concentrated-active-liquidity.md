> 源代码路径: `apps/web/src/lib/pool/v3/use-concentrated-active-liquidity.ts`

# useConcentratedActiveLiquidity Tutorial

## 1. 大白话讲讲这个hook的作用

`useConcentratedActiveLiquidity` *(一个React hook，用于获取Uniswap V3风格集中流动性池的流动性分布图数据)* 是用来获取Uniswap V3风格集中流动性池的"流动性分布图"数据的 hook。

简单来说：
- 它会查询指定交易对（token0/token1）和费率（fee amount）对应的V3池子
- 获取当前活跃 tick（价格位置）周围的所有已初始化的tick数据
- 计算每个tick位置的活跃流动性（liquidityActive）
- 返回一个处理好的tick数组，每个tick包含：价格、净流动性、活跃流动性等信息

这个hook是做V3池子流动性深度图和价格区间选择的基础。

## 2. 讲讲为什么需要封装该hook

Uniswap V3的集中流动性机制允许用户只在特定价格范围内提供流动性，这使得获取和处理tick数据变得复杂：

1. **数据获取复杂**：需要从TickLens合约 *(一个Uniswap V3的辅助合约，用于批量查询tick数据)* 批量读取多个bitmap index的tick数据
2. **计算逻辑复杂**：需要找到当前活跃tick，然后分别向前和向后计算每个tick的"活跃流动性"
3. **数据格式转换**：原始tick数据需要转换为可用的格式（tickIdx、liquidityNet等）

封装成hook后：
- 统一管理数据获取状态（loading、error）
- 自动处理池子地址计算和tickSpacing *(tick间距，不同费率对应不同的最小价格跳动单位)*
- 隐藏复杂的计算逻辑，提供干净的API
- 与React Query *(一个React数据获取和缓存库)* 集成，支持缓存和自动刷新

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  chainId: SushiSwapV3ChainId *(链ID标识符，用于指定在哪个区块链上操作)*          // 链ID
  token0: EvmCurrency | undefined      // 第一个代币
  token1: EvmCurrency | undefined      // 第二个代币
  feeAmount: SushiSwapV3FeeAmount | undefined  // 费率 (100, 500, 3000, 10000)
  enabled?: boolean                   // 是否启用查询
}
```

### 输出（Return Value）
```typescript
{
  isLoading: boolean                  // 是否正在加载
  error: Error | null                  // 错误信息
  activeTick: number | undefined      // 当前活跃tick
  data: TickProcessed[] | undefined    // 处理后的tick数组
}

interface TickProcessed {
  tick: number           // tick索引
  liquidityActive: bigint *(一个大整数类型，用于表示区块链上的大数字)* // 该tick位置的活跃流动性
  liquidityNet: bigint   // 该tick的净流动性变化
  price0: string         // token0的价格（固定8位小数）
}
```

### 执行逻辑
1. 调用 `useConcentratedLiquidityPool` *(一个React hook，用于获取Uniswap V3池子的基本信息如当前价格和流动性)* 获取池子基本信息（liquidity、tickCurrent）
2. 根据token0/token1/feeAmount计算池子地址
3. 计算当前价格对应的bitmap索引范围（默认覆盖±1250个tick）
4. 使用 `useTicks` *(一个React hook，用于批量获取Uniswap V3池子的已初始化tick数据)* 批量从TickLens合约获取所有已初始化word的tick数据
5. 找到活跃tick在tick数组中的位置（pivot）
6. 以pivot为界，分别向前和向后使用 `computeSurroundingTicks` *(一个函数，用于计算指定tick周围每个tick的活跃流动性)* 计算每个tick的活跃流动性
7. 合并得到完整的 `TickProcessed[]` 数组

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于获取Uniswap V3集中流动性池中，当前价格位置周围的流动性分布数据。它会返回每个已初始化tick的详细信息，包括价格、净流动性和活跃流动性。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合@tanstack/react-query来处理数据获取和状态管理，用wagmi和viem来与区块链合约进行交互。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数包括链ID、两个代币对象、费率数量、周围tick数量（默认1250个）和一个控制是否启用的开关。输出应该包含加载状态、错误信息、当前活跃tick以及处理后的tick数组。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先要获取池子的基本信息，然后计算目标池子地址，接着确定需要查询的bitmap索引范围，批量调用合约获取tick数据，找到关键位置后计算每个tick的活跃流动性，最后返回处理好的数据。

### 需要特别注意的约束条件

**参数校验很重要**

一定要处理代币未定义的情况。当传入的token0或token1是undefined时，应该使用skipToken来跳过查询，避免发出无效请求。

**费率与tick间距的对应关系**

不同的费率对应不同的tick间距，这是一个很容易出错的地方。具体来说，费率100对应tick间距50，费率500对应10，费率3000对应60，费率10000对应200。

**bitmap索引的计算方式**

bitmap索引的计算公式是Math.floor(tick / tickSpacing / 256)，这个计算过程需要注意类型的正确性。另外int16类型的范围是-32768到32767，所以不是所有的范围都可以查询。

**大数字处理**

计算过程中会涉及bigint类型的大数字运算，需要注意溢出的问题。同时价格计算需要正确处理两个代币的顺序。

**批量读取优化**

多个合约读取操作需要使用useReadContracts来批量处理，这样可以提高性能。还可以考虑设置合理的refetchInterval或依赖blockNumber变化来实现数据刷新。

### 状态管理的要点

**区分isLoading和isFetching**

这里有一个常见的误区：要使用isLoading而不是isFetching。isLoading只在首次加载时为true，而isFetching在每次请求时都会为true。

**保持状态与数据分离**

返回的状态应该包含isLoading、error和data这三个独立的字段，而不是混在一起。这样便于UI层做判断和渲染。

**分层的加载状态**

这个hook涉及两个层次的加载：池子信息的加载状态和tick数据的加载状态。可以分别用isPoolLoading和isLoading来表示。

**使用useMemo优化**

计算结果需要用useMemo缓存，避免不必要的重复计算。特别是池子地址计算、合约读取配置、数据处理逻辑都应该放在useMemo中。

**queryKey的设计**

queryKey应该包含足够的信息来区分不同的查询。建议使用['concentrated-liquidity', 'active-liquidity', chainId, poolAddress]这样的格式。

**保持旧数据**

可以考虑使用keepPreviousData来保持旧数据，直到新数据加载完成，这样可以避免用户在等待时看到空白界面。

**条件查询控制**

enabled参数应该能够控制是否执行查询，这样可以支持条件渲染等场景。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于获取Uniswap V3集中流动性池中当前价格位置周围的流动性分布数据。

这个hook需要返回每个已初始化tick的详细信息，包括tick索引、该位置的活跃流动性、净流动性变化以及对应的价格。

需要使用@tanstack/react-query来管理数据获取和状态，使用wagmi和viem来与区块链交互。

输入参数包括链ID、两个代币对象、费率数量、周围tick数量（默认1250个）和启用开关。输出包含加载状态、池子加载状态、错误信息、当前活跃tick以及处理后的tick数组。

实现步骤：
1. 首先获取池子的基本信息，包括当前流动性和当前tick
2. 根据工厂合约计算目标池子地址
3. 根据当前tick和费率确定活跃tick的位置
4. 计算需要查询的bitmap索引范围
5. 批量调用TickLens合约获取所有已初始化的tick数据
6. 找到关键位置（pivot）作为计算的起点
7. 从pivot向两侧分别计算每个tick的活跃流动性
8. 计算每个tick对应的价格

重要约束：
- 必须处理代币未定义的情况，使用skipToken跳过查询
- tick间距必须根据费率正确映射
- bitmap索引计算需要注意类型范围
- bigint运算需要注意溢出问题
- 价格计算需要正确处理代币顺序
- 使用useMemo缓存所有计算结果
- queryKey需要包含足够的信息来区分不同的查询

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useConcentratedActiveLiquidity } from '@sushiswap/hooks'
import { useCurrency } from '@sushiswap/ui'

function PoolLiquidityChart() {
  const { data: token0 } = useCurrency(WETH_ADDRESS)
  const { data: token1 } = useCurrency(USDC_ADDRESS)

  const { data: ticks, isLoading, activeTick } = useConcentratedActiveLiquidity({
    chainId: 1,
    token0: token0,
    token1: token1,
    feeAmount: 3000, // 0.3%费率池
    enabled: Boolean(token0 && token1),
  })

  if (isLoading) return <div>加载中...</div>

  return (
    <div>
      <p>当前活跃Tick: {activeTick}</p>
      <p>已初始化Tick数量: {ticks?.length}</p>
      {/* 渲染流动性分布图 */}
    </div>
  )
}
```

### 常见使用场景

1. **流动性深度图展示**
   - 在池子详情页展示价格周围的流动性分布
   - 帮助用户了解不同价格区的流动性深度

2. **价格区间选择辅助**
   - 在添加流动性时，推荐合适的价格区间
   - 显示当前价格附近的流动性集中程度

3. **实时价格监控**
   - 监控活跃tick位置变化
   - 结合refetchInterval实现实时更新

### Dos and Don'ts

**Dos：**
- ✅ 确保token0和token1都定义后再传入，避免不必要的查询
- ✅ 使用enabled参数控制查询启用状态
- ✅ 合理设置numSurroundingTicks范围，范围越大数据越多但性能开销越大
- ✅ 考虑使用refetchInterval实现数据自动刷新

**Don'ts：**
- ❌ 不要在token未定义时传入undefined，这会导致不必要的错误处理
- ❌ 不要设置过大的numSurroundingTicks（如10000+），这会导致大量合约调用
- ❌ 不要忽略isLoading状态，UI应该处理加载中状态
- ❌ 不要在渲染时直接使用bigint，需要先转换为可显示的格式
