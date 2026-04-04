> 源代码路径: `apps/web/src/lib/pool/v3/use-ticks.ts`

# useTicks Tutorial

## 1. 大白话讲讲这个hook的作用

`useTicks` *(一个React hook，用于批量获取Uniswap V3池子中所有已初始化tick的数据)* 是用来批量获取Uniswap V3池子中所有已初始化tick数据的hook。

简单来说：
- Uniswap V3每个池子都有一个"tick bitmap"，记录哪些tick位置已经被初始化了
- 这个hook会查询当前价格附近（比如±1250个tick范围）的所有已初始化tick
- 返回每个已初始化tick的：tick索引、流动性Gross、流动性Net

这是做流动性深度图、计算活跃流动性、确定价格区间的基础数据。

## 2. 讲讲为什么需要封装该hook

获取tick数据涉及多个复杂的智能合约交互：

1. **批量读取**：需要调用TickLens合约 *(Uniswap V3的辅助合约，提供批量查询tick数据的功能)* 的 `getPopulatedTicksInWord` 方法
2. **Bitmap计算**：需要根据当前tick计算需要查询哪些word（每个word包含256个tick）*(tick bitmap将256个连续的tick组织成一个word，用于高效存储和查询)*
3. **数据合并**：多个word的tick数据需要合并、去重、排序
4. **地址计算**：需要先计算池子地址才能查询

封装成hook后：
- 自动处理池子地址计算
- 自动确定需要查询的bitmap索引范围
- 批量执行多个合约调用
- 返回整理好的tick数组

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  token0: EvmCurrency | undefined      // 第一个代币
  token1: EvmCurrency | undefined      // 第二个代币
  chainId: SushiSwapV3ChainId *(链ID标识符，用于指定在哪个区块链上操作)*           // 链ID
  feeAmount: SushiSwapV3FeeAmount | undefined  // 费率
  numSurroundingTicks?: number         // 周围tick数量，默认1250
  enabled?: boolean                    // 是否启用
}
```

### 输出（Return Value）
```typescript
{
  data: {
    tickIdx: number    // tick索引
    liquidityNet: bigint *(一个大整数类型，用于表示区块链上的大数字)*  // 流动性净变化
  }[] | undefined,
  isLoading: boolean,
  error: Error | null,
  // ... 其他useReadContracts的属性
}
```

### 执行逻辑
1. 调用 `useConcentratedLiquidityPool` *(一个React hook，用于获取Uniswap V3池子的基本信息如当前价格和流动性)* 获取池子的 `tickCurrent`
2. 根据 feeAmount 获取 tickSpacing
3. 计算 activeTick = nearestUsableTick(tickCurrent, tickSpacing) *(将当前tick转换为给定tickSpacing下的有效tick)*
4. 根据 token0/token1/feeAmount/chainId 计算池子地址
5. 计算最小/最大 bitmap 索引：
   ```
   minIndex = bitmapIndex(activeTick - numSurroundingTicks * tickSpacing, tickSpacing)
   maxIndex = bitmapIndex(activeTick + numSurroundingTicks * tickSpacing, tickSpacing)
   ```
6. 使用 `useReadContracts` *(一个React hook，用于批量执行多个合约只读调用)* 批量调用 `getPopulatedTicksInWord`
7. 合并所有 word 的 tick 数据
8. 转换为 `{ tickIdx, liquidityNet }` 格式
9. 按 tickIdx 排序返回

### Bitmap Index计算
```typescript
const bitmapIndex = (tick: number, tickSpacing: number) => {
  return Math.floor(tick / tickSpacing / 256)
}
```

### 关键ABI
```typescript
{
  inputs: [
    { internalType: 'address', name: 'pool', type: 'address' },
    { internalType: 'int16', name: 'tickBitmapIndex', type: 'int16' },
  ],
  name: 'getPopulatedTicksInWord',
  outputs: [
    {
      components: [
        { internalType: 'int24', name: 'tick', type: 'int24' },
        { internalType: 'int128', name: 'liquidityNet', type: 'int128' },
        { internalType: 'uint128', name: 'liquidityGross', type: 'uint128' },
      ],
      internalType: 'struct ITickLens.PopulatedTick[]',
      name: 'populatedTicks',
      type: 'tuple[]',
    },
  ],
  stateMutability: 'view',
  type: 'function',
}
```

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于获取Uniswap V3池子指定价格范围内的所有已初始化tick数据。它会批量查询TickLens合约，返回每个tick的索引和流动性信息。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合@tanstack/react-query的useReadContracts来进行批量合约读取，使用wagmi和viem来与区块链合约进行交互。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数包括两个代币对象、链ID、费率数量、周围tick数量（默认1250个）和启用开关。输出应该包含tick索引和流动性净变化的数据数组。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先要通过useConcentratedLiquidityPool获取池子的当前tick，然后根据tick间距计算活跃tick位置，接着计算需要查询的bitmap索引范围，用useReadContracts批量调用TickLens合约的getPopulatedTicksInWord方法，最后合并、转换和排序返回的数据。

### 需要特别注意的约束条件

**bitmap索引范围的计算**

minIndex到maxIndex之间可能有数百个word需要查询。计算公式是bitmapIndex = Math.floor(tick / tickSpacing / 256)。需要注意的是，int16类型的范围是-32768到32767，所以不是所有的tick范围都可以查询。

**useReadContracts的配置**

useReadContracts的allowFailure应该设为false，因为如果有word不存在就说明这个范围没有初始化的tick，查询会失败。每个word的tick数据需要使用concat合并，而不是push。

**排序和类型注意**

排序使用的是tickIdx，这是一个number类型的字段。排序后返回的数组应该按tickIdx升序排列。

**activeTick的计算方式**

activeTick需要用nearestUsableTick函数来计算，传入池子的当前tick和对应的tickSpacing。池子地址的计算需要使用computeSushiSwapV3PoolAddress函数，传入工厂合约地址和代币信息。

**TickLens合约地址**

链上的TickLens合约地址使用SUSHISWAP_V3_TICK_LENS[chainId]来获取，不同的链有不同的地址。

**查询条件检查**

如果minIndex或maxIndex未定义，不应该发起任何查询。只有当这两个值以及池子地址都定义时，才能执行查询。

### 状态管理的要点

**使用useMemo缓存配置**

contractReads配置需要用useMemo缓存，这样可以避免每次渲染时都重新创建配置对象。

**数据转换和排序也在useMemo中**

处理数据转换和排序的逻辑也应该放在useMemo中，确保只有在数据变化时才重新处理。

**queryKey的设计**

queryKey应该包含chainId、poolAddress、minIndex和maxIndex，这样每个不同的查询都有唯一的标识。格式可以是['ticks', chainId, poolAddress, minIndex, maxIndex]。

**考虑添加自动刷新**

可以添加refetchInterval来定期刷新数据，实现实时更新的效果。enabled参数应该同时影响池子查询和tick查询。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于获取Uniswap V3池子指定价格范围内的所有已初始化tick数据。

这个hook需要批量查询TickLens合约，返回每个tick的索引和流动性信息。

需要使用@tanstack/react-query的useReadContracts来进行批量合约读取，使用wagmi和viem来与区块链合约交互。

输入参数包括两个代币对象（都可能是undefined）、链ID、费率数量（也可能是undefined）、周围tick数量（默认1250个）和启用开关。

输出包含一个数据数组，每个元素有tick索引和流动性净变化两个字段，还有标准的loading和error状态。

实现步骤：
1. 首先通过useConcentratedLiquidityPool获取池子的当前tick
2. 根据费率获取对应的tick间距
3. 计算活跃tick位置：nearestUsableTick(pool.tickCurrent, tickSpacing)
4. 计算池子地址，使用computeSushiSwapV3PoolAddress函数
5. 根据活跃tick和周围tick数量计算需要查询的bitmap索引范围
6. 用useMemo构建contractReads数组，对范围内每个索引创建一个合约调用
7. 使用useReadContracts批量执行这些调用
8. 在useMemo中合并所有返回的tick数据，转换为指定格式，按tick索引排序

重要约束：
- bitmap索引计算使用int16类型，注意范围限制
- useReadContracts的allowFailure应设为false
- 合并tick数据时使用concat而非push
- 排序后返回的数组应该按tickIdx升序
- 只有当所有必要参数都定义时才执行查询

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useTicks } from '@sushiswap/hooks'
import { useCurrency } from '@sushiswap/ui'

function TicksDisplay() {
  const { data: token0 } = useCurrency(WETH_ADDRESS)
  const { data: token1 } = useCurrency(USDC_ADDRESS)

  const { data: ticks, isLoading, error } = useTicks({
    chainId: 1,
    token0: token0,
    token1: token1,
    feeAmount: 3000,
    numSurroundingTicks: 1000,
    enabled: Boolean(token0 && token1),
  })

  if (isLoading) return <div>加载中...</div>
  if (error) return <div>错误: {error.message}</div>

  return (
    <div>
      <h3>已初始化的Ticks ({ticks?.length})</h3>
      {ticks?.slice(0, 10).map((tick) => (
        <div key={tick.tickIdx}>
          Tick: {tick.tickIdx}, LiquidityNet: {tick.liquidityNet.toString()}
        </div>
      ))}
    </div>
  )
}
```

### 常见使用场景

1. **流动性深度图**
   - 展示不同价格位置的流动性分布
   - 帮助用户了解市场的流动性分布

2. **计算活跃流动性**
   - 结合 `useConcentratedActiveLiquidity` 计算价格周围的活跃流动性
   - 用于添加流动性时的价格区间推荐

3. **查找最近的流动性边界**
   - 在限价单中使用，找到最近的流动性点
   - 辅助价格 Impact 计算

### Dos and Don'ts

**Dos：**
- ✅ 确保token0和token1都定义后再启用查询
- ✅ 合理设置numSurroundingTicks，范围太大会产生大量合约调用
- ✅ 处理bigint类型时使用toString()或format方法转换
- ✅ 使用useMemo缓存计算结果

**Don'ts：**
- ❌ 不要设置过大的numSurroundingTicks（如5000+），会导致性能问题
- ❌ 不要直接渲染bigint，需要先转换格式
- ❌ 不要忽略isLoading和error状态的处理
- ❌ 不要在tick数据上直接进行复杂的链上计算，应该在合约层面处理
