> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/pool/use-pool-daily-volume-usd.ts`

# usePoolDailyVolumeUsd Hook Tutorial

## 大白话讲讲这个hook的作用

`usePoolDailyVolumeUsd` *(一个React hook，用于获取池子24小时交易量，直接返回USD计价的面板数据)* 是一个用于查询池子 24 小时交易量的 hook。它返回以 USD 计价的池子日交易量，帮助用户了解池子的流动性活跃程度。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **数据聚合**：从 `useTopPools` *(获取热门池子列表的hook)* 获取所有池子的交易量数据
2. **查找特定池子**：根据池子地址找到对应的交易量数据
3. **USD 计价**：直接返回 USD 计价的数值方便 UI 使用

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
pairAddress?: string            // 池子/交易对地址
```

### 输出（返回值）
```typescript
{
  data: string                  // 日交易量（USD 格式字符串）
}
```

### 核心执行逻辑

1. **获取顶级池子**：调用 `useTopPools()` 获取所有池子数据
2. **查找匹配**：根据 `pairAddress` 在池子列表中找到对应池子
3. **返回交易量**：返回池子的 `volumeUSD1d` *(池子24小时交易量的USD价值)* 字段

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个查询池子日交易量的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似usePoolDailyVolumeUsd的hook，用来查询池子的日交易量，以美元计价。然后明确几个关键点。第一，参数就是池子地址，很简单。第二，从useTopPools获取所有池子的列表，因为交易量数据在那里。第三，根据池子地址在列表里找到对应的池子。第四，返回日交易量，格式是美元字符串。第五，用React Query的keepPreviousData在加载新数据时保持旧数据。

### 这里面有几个地方特别容易出错

依赖useTopPools的数据，所以那个hook的数据要先准备好。池子如果不在热门池子列表里，就返回'0'，不要返回undefined或者报错，这种情况是正常的，不算错误。

### 数据刷新这里有讲究

依赖useTopPools数据可用，这个数据获取到了这个hook才能工作。交易量数据不需要频繁刷新，因为日交易量是24小时内的累计，不会每秒都在变，staleTime可以设成15分钟。这个hook本质上是做数据查找而不是实时计算，所以逻辑比较简单。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { usePoolDailyVolumeUsd } from '@sushiswap/hooks'

function PoolCard({ poolAddress }: { poolAddress: string }) {
  const { data: dailyVolume } = usePoolDailyVolumeUsd(poolAddress)

  return (
    <div>
      <p>24小时交易量: ${dailyVolume ?? '0'}</p>
    </div>
  )
}
```

### 常见使用场景

1. **池子活跃度显示**：在池子列表中显示各池子的交易量
   ```tsx
   const volume = usePoolDailyVolumeUsd(pool.address)
   const isActive = Number(volume) > 10000 // 超过1万美金视为活跃
   ```

2. **排序依据**：按交易量排序显示池子
   ```tsx
   const sortedPools = pools.sort((a, b) =>
     Number(usePoolDailyVolumeUsd(b.address)) - Number(usePoolDailyVolumeUsd(a.address))
   )
   ```

3. **TVL 对比**：交易量与 TVL 的比率反映资金效率
   ```tsx
   const dailyVolume = Number(volumeData)
   const tvl = Number(poolTvl)
   const volumeToTvlRatio = dailyVolume / tvl
   ```

### Dos and Don'ts

**Dos:**
- ✅ 返回的是字符串格式，可直接用于显示
- ✅ 池子不在列表中时返回 '0'，不需要额外处理
- ✅ 配合池子列表一起使用

**Don'ts:**
- ❌ 不要用于实时价格计算，这是历史数据
- ❌ 不要假设数据总是存在，未找到返回 '0'
- ❌ 不要设置过短的 staleTime，交易量不需要频繁更新
