> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/pool/lib/farm/use-rewards-per-day.ts`

# useRewardsPerDay Hook Tutorial

## 大白话讲讲这个hook的作用

`useRewardsPerDay` 是一个用于计算 Farm 每日奖励的 hook。它根据：

- Farm 的分配点数（alloc_point）
- Farm 的总分配点数
- SUSHI 每秒产量

计算出该 Farm 池子每日能获得的奖励。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **奖励计算**：封装奖励计算逻辑
2. **Decimals 处理**：处理代币精度转换

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  farms: FarmLP | undefined           // Farm 数据
  farmIndex: number | undefined      // Farm 索引
  decimals: number | undefined       // 奖励代币小数位
}
```

### 输出（返回值）
```typescript
string   // 每日奖励数量（字符串格式）
```

### 核心执行逻辑

1. **计算份额**：alloc_point / total_alloc_point
2. **计算每日**：sushi_per_second * 86400 * 份额
3. **格式化**：按 decimals 格式化

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useRewardsPerDay 的 React hook，用来计算一个农场每天能获得多少奖励。

核心需求：
1. 接收这些参数：农场数据、农场索引、代币精度
2. 根据这个农场占总奖励的比例，计算出每天能分到多少
3. 算出每天的总奖励数量
4. 按照代币的精度格式化成正确的数字
5. 用缓存来保存计算结果

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**用86400算每天**：一天有多少秒？86400。所以如果知道每秒产多少，用这个数字乘以86400就是每天的产量。

### 这个Hook怎么管理状态

建议用缓存来保存计算结果。因为这个计算涉及好几个数据，而且可能会被频繁调用，用缓存可以避免重复计算。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useRewardsPerDay } from '@sushiswap/aptos'
import { useFarms } from '@sushiswap/aptos'

function DailyReward({ farmIndex }: { farmIndex: number }) {
  // 获取 Farm 信息
  const { data: farms } = useFarms()

  // 使用 useRewardsPerDay 计算每日奖励
  const dailyReward = useRewardsPerDay({
    farms,
    farmIndex,
    decimals: 8
  })

  return (
    <div>
      <p>Farm #{farmIndex} 每日奖励</p>
      <p>{dailyReward} SUSHI</p>
    </div>
  )
}
```

### 常见用法

#### 1. 显示所有 Farm 每日奖励

```tsx
import { useRewardsPerDay, useFarms } from '@sushiswap/aptos'

function AllFarmsRewards() {
  // 获取所有 Farm 信息
  const { data: farms } = useFarms()

  // 获取所有 Farm 的每日奖励
  const farmRewards = Object.keys(farms ?? {}).map((farmIndex) => {
    const reward = useRewardsPerDay({
      farms,
      farmIndex: Number(farmIndex),
      decimals: 8
    })
    return { farmIndex: Number(farmIndex), reward }
  })

  return (
    <div>
      <h3>各 Farm 每日奖励</h3>
      {farmRewards.map(({ farmIndex, reward }) => (
        <div key={farmIndex}>
          Farm #{farmIndex}: {reward} SUSHI
        </div>
      ))}
    </div>
  )
}
```

#### 2. 计算年化收益（APR）

```tsx
import { useRewardsPerDay, usePools, useTotalSupply } from '@sushiswap/aptos'

function FarmAPR({ farmIndex, poolAddress }: { farmIndex: number; poolAddress: string }) {
  // 获取每日奖励
  const { data: farms } = useFarms()
  const dailyReward = useRewardsPerDay({
    farms,
    farmIndex,
    decimals: 8
  })

  // 获取池子信息
  const { data: pool } = usePool(poolAddress)
  // 获取 LP 总供应量
  const { data: coinInfo } = useTotalSupply(pool?.lpTokenAddress)
  // 假设 SUSHI 价格
  const sushiPrice = 10 // 实际应从 useStablePrice 获取

  // 计算年化收益
  const dailyRewardValue = Number(dailyReward) * sushiPrice
  const totalSupplyValue = Number(coinInfo?.supply) * lpPrice // 需要计算 LP 价格
  const apr = (dailyRewardValue * 365) / totalSupplyValue * 100

  return (
    <div>
      <p>Farm #{farmIndex} APR</p>
      <p>{apr.toFixed(2)}%</p>
    </div>
  )
}
```

#### 3. 用户预计收益

```tsx
import { useRewardsPerDay, useUserPool } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function EstimatedReward({ farmIndex }: { farmIndex: number }) {
  const { account } = useAccount()

  // 获取用户池子信息
  const { data: userPoolInfo } = useUserPool(account?.address)
  // 获取每日奖励
  const { data: farms } = useFarms()
  const dailyReward = useRewardsPerDay({
    farms,
    farmIndex,
    decimals: 8
  })

  // 用户是否在此 Farm 质押
  const isStaking = userPoolInfo?.pids.includes(farmIndex) ?? false

  if (!isStaking) {
    return <div>未参与此 Farm</div>
  }

  return (
    <div>
      <p>预计每日奖励: {dailyReward} SUSHI</p>
      <p>预计每月奖励: {Number(dailyReward) * 30} SUSHI</p>
    </div>
  )
}
```

### Do（推荐做法）

- **使用 useMemo 缓存**：避免重复计算
- **按 decimals 格式化**：返回格式化后的字符串
- **组合 useFarms 获取 Farm 数据**：需要 farms 参数
- **SECONDS_PER_DAY = 86400**：使用正确的常量

### Don't（不推荐做法）

- **不要忘记 decimals**：需要正确处理精度
- **不要假设 all undefined**：需要检查 farms 存在
- **不要忽略边界情况**：farmIndex 超出范围应该处理

### 相关的其他 hooks

- `useFarms` *(一个React hook，获取所有Farm池子信息，返回FarmLP数据结构，包含alloc_point、lps等字段)*：获取 Farm 信息
- `useUserRewards` *(一个React hook，计算用户待领取的Farm奖励，根据用户质押信息和Farm参数计算可领取数量)*：计算待领取奖励
- `usePool` *(一个React hook，用于获取单个流动性池信息，包括代币对、储备量、手续费率等)*：获取池子信息
- `alloc_point` *(一个配置/概念，表示Farm池子的奖励分配点数，决定该池子占总奖励的份额)*：分配点数
