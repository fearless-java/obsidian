> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/pool/lib/farm/use-farms.ts`

# useFarms / useIsFarm Hook Tutorial

## 大白话讲讲这个hook的作用

`useFarms` 和 `useIsFarm` 是用于查询 Farm 信息的 hooks：

- **useFarms**：获取所有 Farm 池子信息
- **useIsFarm**：检查某个池子是否是 Farm 池子

## 讲讲为什么需要封装该hook

封装这些 hooks 的原因：

1. **MasterChef 交互**：封装与 MasterChef 合约的交互
2. **Farm 判断**：判断池子是否在 Farm 中

## 讲讲该hook的执行逻辑和数据流向

### useFarms
**输出：**
```typescript
{
  data: FarmLP | undefined   // Farm 合约数据
}
```

### useIsFarm
**输入：**
```typescript
{
  poolAddress: string       // 池子地址
  farms: FarmLP | undefined // Farm 数据
}
```

**输出：**
```typescript
number | undefined   // Farm 索引，或 undefined
```

### 核心执行逻辑

1. **查询 Farm 资源**：获取 MasterChef 的 MasterChef 资源
2. **检查池子**：在 lps 数组中查找池子

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useFarms 的 React hook，用来查询流动性农场的信息。

核心需求：
1. 查询 MasterChef 合约里的 MasterChef 资源
2. 返回农场相关的数据
3. 还要能检查某个池子是不是在农场里
4. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**配置合约地址**：查询的时候需要知道 MasterChef 合约的地址，这个地址要先配置好。不同的网络这个地址可能不一样。

## 5. 该 hook 的用法教学

### 基本用法

#### 获取所有 Farm 信息

```tsx
import { useFarms } from '@sushiswap/aptos'

function FarmList() {
  // 使用 useFarms 获取所有 Farm 信息
  const { data: farms } = useFarms()

  return (
    <div>
      <h3>Farm 列表</h3>
      {farms && Object.entries(farms).map(([pid, farm]) => (
        <FarmRow key={pid} pid={Number(pid)} farm={farm} />
      ))}
    </div>
  )
}
```

#### 检查池子是否是 Farm 池子

```tsx
import { useFarms, useIsFarm } from '@sushiswap/aptos'
import { usePool } from '@sushiswap/aptos'

function FarmIndicator({ poolAddress }: { poolAddress: string }) {
  // 获取 Farm 信息
  const { data: farms } = useFarms()
  // 获取池子信息
  const { data: pool } = usePool(poolAddress)

  // 使用 useIsFarm 检查是否是 Farm 池子
  const farmIndex = useIsFarm({ poolAddress, farms })

  const isFarmPool = farmIndex !== undefined

  return (
    <div>
      {isFarmPool ? (
        <span className="badge farm">Farm #{farmIndex}</span>
      ) : (
        <span className="badge">非 Farm 池</span>
      )}
    </div>
  )
}
```

### 常见用法

#### 1. 显示 Farm 奖励信息

```tsx
import { useFarms, useRewardsPerDay } from '@sushiswap/aptos'

function FarmRewardInfo({ farmIndex }: { farmIndex: number }) {
  // 获取 Farm 信息
  const { data: farms } = useFarms()
  const farm = farms?.[farmIndex]

  // 使用 useRewardsPerDay 计算每日奖励
  const rewardsPerDay = useRewardsPerDay({
    farms,
    farmIndex,
    decimals: 8
  })

  return (
    <div>
      <h3>Farm #{farmIndex}</h3>
      <div>每日奖励: {rewardsPerDay}</div>
      <div>分配点数: {farm?.alloc_point}</div>
    </div>
  )
}
```

#### 2. 用户 Farm 持仓

```tsx
import { useFarms, useUserPool, useUserHandle, useUserRewards } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function MyFarmPosition({ pid }: { pid: number }) {
  const { account } = useAccount()

  // 获取 Farm 信息
  const { data: farms } = useFarms()
  // 获取用户池子信息
  const { data: userPoolInfo } = useUserPool(account?.address)
  // 获取用户质押信息
  const { data: userStakes } = useUserHandle({ userHandle: userPoolInfo })

  // 找到 PID 索引
  const pIdIndex = userPoolInfo?.pids.findIndex((p) => p === pid)
  const farmIndex = pid

  // 计算奖励
  const pendingReward = useUserRewards({
    farms,
    stakes: userStakes,
    pIdIndex,
    farmIndex
  })

  const isStaking = userPoolInfo?.pids.includes(pid) ?? false

  return (
    <div>
      {isStaking ? (
        <>
          <p>质押中</p>
          <p>待领取奖励: {pendingReward}</p>
        </>
      ) : (
        <p>未参与此 Farm</p>
      )}
    </div>
  )
}
```

#### 3. Farm 状态指示器

```tsx
import { useFarms, useIsFarm } from '@sushiswap/aptos'

function PoolFarmStatus({ poolAddress }: { poolAddress: string }) {
  // 获取 Farm 信息
  const { data: farms } = useFarms()
  // 检查是否是 Farm 池子
  const farmIndex = useIsFarm({ poolAddress, farms })

  if (farmIndex === undefined) {
    return null
  }

  return (
    <div className="farm-badge">
      <span>🌾 Farm #{farmIndex}</span>
    </div>
  )
}
```

### Do（推荐做法）

- **使用 useIsFarm 判断**：判断池子是否是 Farm 池子
- **组合 useRewardsPerDay 计算奖励**：计算每日奖励
- **组合 useUserRewards 计算待领奖励**：计算用户待领取的奖励
- **检查 farmIndex !== undefined**：判断是否在 Farm 中

### Don't（不推荐做法）

- **不要假设所有池子都是 Farm**：需要检查
- **不要忽略 Farm 索引**：farmIndex 是重要的参数
- **不要忘记处理 undefined**：需要检查池子是否在 Farm 中

### 相关的其他 hooks

- `useUserRewards` *(一个React hook，计算用户待领取的Farm奖励，根据用户质押信息和Farm参数计算可领取数量)*：计算待领取奖励
- `useRewardsPerDay` *(一个React hook，计算Farm每日奖励，根据alloc_point和总分配点数计算每个池子的每日奖励)*：计算每日奖励
- `useUserHandle` *(一个React hook，用于获取用户在MasterChef合约中的持仓信息，包括PoolUserInfo和UserStakes)*：获取用户质押信息
- `FarmLP` *(一个TypeScript类型/配置，表示Farm的数据结构，包含alloc_point、lps等字段)*：Farm 数据类型
