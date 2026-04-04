> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/tick/use-ticks.ts`

# useTicks Hook Tutorial

## 大白话讲讲这个hook的作用

`useTicks` *(一个React hook，用于获取池子的所有已初始化tick位置数据，包括净流动性和总流动性)* 是一个用于获取池子 tick 数据的 hook。它返回池子的：

- 所有已初始化的 tick 位置
- 每个 tick 的净流动性变化（liquidityNet）
- 每个 tick 的总流动性（liquidityGross）

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **合约调用封装**：调用 PoolLens 合约 *(池子镜头合约，提供池子相关数据的查询)* 获取 tick 数据
2. **范围优化**：只获取当前价格附近的 tick，而不是全部
3. **数据处理**：处理返回的原始数据并排序

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  pool: PoolInfo | null | undefined   // 池子信息
  numSurroundingTicks?: number       // 周围 tick 数量，默认 1250
  enabled?: boolean                   // 是否启用
}
```

### 输出（返回值）
```typescript
{
  data: PopulatedTick[]             // 填充的 tick 数据
}
```

### PopulatedTick 结构
```typescript
{
  tickIdx: number                  // tick 索引
  liquidityNet: bigint             // 净流动性变化
  liquidityGross: bigint           // 总流动性
}
```

### 核心执行逻辑

1. **计算范围**：根据当前 tick 和 numSurroundingTicks 确定要获取的 tick 范围
2. **调用合约**：使用 `getPopulatedTicksInRange` *(PoolLens合约方法，获取指定范围内的已初始化tick)* 获取 tick 数据
3. **排序返回**：按 tickIdx 排序

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个获取池子tick数据的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useTicks的hook，用来获取某个区块链上池子的tick数据。然后明确几个关键点。第一，参数是池子信息和周围tick数量，周围tick数量决定要获取多少个tick，默认1250个差不多了。第二，调用PoolLens合约获取指定范围内的tick数据，PoolLens是一个专门提供池子相关数据的合约。第三，返回的数据包括tickIdx、liquidityNet、liquidityGross，分别是tick索引、净流动性变化和总流动性。第四，按tickIdx排序返回，便于使用。第五，用React Query管理数据，支持定时刷新。

### 这里面有几个地方特别容易出错

范围不能太大，获取太多tick会很慢，影响性能。合约返回的数据可能被截断，如果有警告信息要处理一下。

### 数据刷新这里有讲究

staleTime可以设成30秒，tick数据变化不算太频繁。refetchInterval设成60秒，定时刷新保持数据最新。这个hook是其他很多tick相关hook的基础数据来源，用好缓存能提高整体性能。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useTicks } from '@sushiswap/hooks'

function TicksDisplay({ pool }: { pool: PoolInfo }) {
  const { data: ticks, isLoading } = useTicks({
    pool,
    numSurroundingTicks: 1000,
  })

  if (isLoading) return <p>加载中...</p>

  return (
    <div>
      <h3>Tick 数据</h3>
      <ul>
        {ticks?.slice(0, 10).map((tick) => (
          <li key={tick.tickIdx}>
            Tick {tick.tickIdx}: Net={tick.liquidityNet.toString()}
          </li>
        ))}
      </ul>
    </div>
  )
}
```

### 常见使用场景

1. **计算活动流动性**：与其他 hook 配合计算头寸活动性
   ```tsx
   const { data: ticks } = useTicks({ pool })
   // 用于 useConcentratedActiveLiquidity
   ```

2. **显示流动性分布**：展示不同 tick 的流动性
   ```tsx
   const tickLiquidity = ticks?.map(t => ({
     tick: t.tickIdx,
     liquidity: t.liquidityGross,
   }))
   ```

3. **查找最近的 tick**：找到当前价格最近的已初始化 tick
   ```tsx
   const nearestTick = ticks?.reduce((nearest, t) =>
     Math.abs(t.tickIdx - currentTick) < Math.abs(nearest.tickIdx - currentTick)
       ? t
       : nearest
   )
   ```

### Dos and Don'ts

**Dos:**
- ✅ 配合 `useConcentratedActiveLiquidity` *(计算活动流动性的hook)* 使用
- ✅ 设置合理的 `numSurroundingTicks` 避免数据过大
- ✅ 使用定时刷新保持数据最新

**Don'ts:**
- ❌ 不要获取过大的范围，会影响性能
- ❌ 不要忽略合约返回截断数据的警告
- ❌ 不要直接修改返回的 tick 数据
