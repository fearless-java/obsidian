> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/pool/lib/farm/use-user-rewards.ts`

# useUserRewards Hook Tutorial

## 大白话讲讲这个hook的作用

`useUserRewards` 是一个用于计算用户待领取奖励的 hook。它根据：

- 用户的质押信息
- Farm 的奖励参数
- 计算用户当前可领取的奖励数量

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **奖励计算**：封装 MasterChef 奖励计算公式
2. **精度处理**：处理 ACC_SUSHI_PRECISION

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  farms: FarmLP | undefined           // Farm 数据
  stakes: UserStakes | undefined     // 用户质押信息
  pIdIndex: number | undefined       // PID 索引
  farmIndex: number | undefined      // Farm 索引
}
```

### 输出（返回值）
```typescript
number   // 待领取奖励数量
```

### 核心执行逻辑

1. **计算每份**：从 farms 获取 acc_sushi_per_share
2. **计算总额**：amount * acc_per_share / PRECISION
3. **减去已领**：减去 reward_debt

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useUserRewards 的 React hook，用来计算用户能领取多少还没领取的奖励。

核心需求：
1. 接收这些参数：农场数据、用户质押信息、用户PID索引、农场索引
2. 用 MasterChef 的公式来计算奖励
3. 要考虑已经领取过的部分要扣掉
4. 用精度因子来处理大数计算
5. 用缓存来保存计算结果

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

1. **精度因子**：奖励计算涉及除法的时候，为了保证结果准确，需要用一个很大的精度因子（1000000000000）来处理。这是为了避免大数运算时出现精度损失。

2. **检查索引是否有效**：传入的索引可能超出范围了，比如用户根本没有在那个农场质押。计算之前要先检查索引是不是有效的。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useUserRewards, useUserPool, useUserHandle } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function PendingRewards({ farmIndex }: { farmIndex: number }) {
  const { account } = useAccount()

  // 获取用户池子信息
  const { data: userPoolInfo } = useUserPool(account?.address)
  // 获取用户质押信息
  const { data: userStakes } = useUserHandle({ userHandle: userPoolInfo })

  // 获取 PID 索引
  const pIdIndex = userPoolInfo?.pids.findIndex((p) => p === farmIndex)

  // 使用 useUserRewards 计算待领取奖励
  const pendingReward = useUserRewards({
    farms: undefined, // 需要从 context 获取
    stakes: userStakes,
    pIdIndex,
    farmIndex
  })

  return (
    <div>
      <p>待领取奖励</p>
      <p>{pendingReward} SUSHI</p>
    </div>
  )
}
```

### 常见用法

#### 1. 显示用户在所有 Farm 的待领奖励

```tsx
import { useUserRewards, useUserPool, useUserHandle, useFarms } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function AllPendingRewards() {
  const { account } = useAccount()

  // 获取用户池子信息
  const { data: userPoolInfo } = useUserPool(account?.address)
  // 获取用户质押信息
  const { data: userStakes } = useUserHandle({ userHandle: userPoolInfo })
  // 获取 Farm 信息
  const { data: farms } = useFarms()

  const pids = userPoolInfo?.pids ?? []

  return (
    <div>
      <h3>我的待领取奖励</h3>
      {pids.map((pid) => {
        const pIdIndex = userPoolInfo?.pids.findIndex((p) => p === pid)
        const reward = useUserRewards({
          farms,
          stakes: userStakes,
          pIdIndex,
          farmIndex: pid
        })

        return (
          <div key={pid}>
            Farm #{pid}: {reward} SUSHI
          </div>
        )
      })}
    </div>
  )
}
```

#### 2. 总待领奖励显示

```tsx
import { useUserRewards, useUserPool, useUserHandle, useFarms } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function TotalPendingRewards() {
  const { account } = useAccount()

  // 获取用户池子信息
  const { data: userPoolInfo } = useUserPool(account?.address)
  // 获取用户质押信息
  const { data: userStakes } = useUserHandle({ userHandle: userPoolInfo })
  // 获取 Farm 信息
  const { data: farms } = useFarms()

  const pids = userPoolInfo?.pids ?? []

  // 计算总待领奖励
  const totalRewards = pids.reduce((sum, pid) => {
    const pIdIndex = userPoolInfo?.pids.findIndex((p) => p === pid)
    const reward = useUserRewards({
      farms,
      stakes: userStakes,
      pIdIndex,
      farmIndex: pid
    })
    return sum + reward
  }, 0)

  return (
    <div>
      <h3>总待领取</h3>
      <p>{totalRewards} SUSHI</p>
    </div>
  )
}
```

#### 3. 领取按钮状态

```tsx
import { useUserRewards, useUserPool, useUserHandle } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function ClaimButton({ farmIndex }: { farmIndex: number }) {
  const { account } = useAccount()

  const { data: userPoolInfo } = useUserPool(account?.address)
  const { data: userStakes } = useUserHandle({ userHandle: userPoolInfo })
  const { data: farms } = useFarms()

  const pIdIndex = userPoolInfo?.pids.findIndex((p) => p === farmIndex)
  const pendingReward = useUserRewards({
    farms,
    stakes: userStakes,
    pIdIndex,
    farmIndex
  })

  const hasStaked = userPoolInfo?.pids.includes(farmIndex) ?? false
  const hasRewards = pendingReward > 0

  return (
    <button disabled={!hasStaked || !hasRewards}>
      {!hasStaked
        ? '未质押'
        : !hasRewards
        ? '无奖励'
        : `领取 ${pendingReward} SUSHI`}
    </button>
  )
}
```

### Do（推荐做法）

- **使用 useMemo 缓存**：避免重复计算
- **检查 pIdIndex 有效性**：findIndex 可能返回 -1
- **组合 useFarms、useUserHandle 使用**：获取所有必要数据
- **处理 reward_debt**：已领取部分需要扣除

### Don't（不推荐做法）

- **不要忽略精度处理**：使用正确的 PRECISION
- **不要假设用户有质押**：需要检查 pids 是否包含
- **不要忘记检查 undefined**：参数可能是 undefined

### 相关的其他 hooks

- `useFarms` *(一个React hook，获取所有Farm池子信息，返回FarmLP数据结构，包含alloc_point、lps等字段)*：获取 Farm 信息
- `useUserHandle` *(一个React hook，用于获取用户在MasterChef合约中的持仓信息，包括PoolUserInfo和UserStakes)*：获取用户质押信息
- `useRewardsPerDay` *(一个React hook，计算Farm每日奖励，根据alloc_point和总分配点数计算每个池子的每日奖励)*：计算每日奖励
- `ACC_SUSHI_PRECISION` *(一个配置/常量，表示SUSHI奖励的精度因子，值为1000000000000，用于除法时保持精度)*：精度常量
