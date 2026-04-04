> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/_common/lib/common/use-user-handle.ts`

# useUserHandle Hook Tutorial

## 大白话讲讲这个hook的作用

`useUserHandle` 是一个用于获取用户在 MasterChef 合约中持仓信息的 hook 集合：

- **useUserPool**：获取用户的池子信息（pid 列表）
- **useUserHandle**：获取用户的 stake handle（用于查询质押信息）

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **Farm 交互**：封装与 MasterChef 合约的交互
2. **Handle 查找**：获取用户质押信息的 table handle

## 讲讲该hook的执行逻辑和数据流向

### useUserPool
**输入：**
```typescript
address: string | undefined     // 用户地址
```

**输出：**
```typescript
{
  data: PoolUserInfo | undefined
}
```

### useUserHandle
**输入：**
```typescript
{
  userHandle: PoolUserInfo | undefined
}
```

**输出：**
```typescript
{
  data: UserStakes | undefined
}
```

### 核心执行逻辑

1. **查询资源**：从用户账户获取 PoolUserInfo 资源
2. **获取 handle**：从资源中获取 pid_to_user_info handle

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useUserHandle 的 React hook，用来查询用户在流动性农场里的持仓信息。

核心需求：
1. 查询用户在资金池合约里的个人持仓信息
2. 获取一个叫 pid_to_user_info 的句柄
3. 用这个句柄去查询用户具体的质押详情
4. 用 React Query 来管理数据请求
5. 要能定时刷新数据

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**质押信息变化快**：用户在农场里的质押情况可能会经常变动，比如转入转出LP代币、领取奖励等操作都会改变数据。所以要定期去刷新数据，而不是查一次就完了。

### 这个Hook怎么管理状态

建议设置每2秒自动刷新一次。这个频率足够及时地反映用户的最新持仓情况，又不会给服务器太大压力。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useUserPool, useUserHandle } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function UserFarmInfo() {
  const { account } = useAccount()

  // 第一步：获取用户的 PoolUserInfo
  const { data: userPoolInfo } = useUserPool(account?.address)

  // 第二步：使用 PoolUserInfo 获取用户的质押信息
  const { data: userStakes } = useUserHandle({ userHandle: userPoolInfo })

  return (
    <div>
      <p>PID 列表: {userPoolInfo?.pids.join(', ')}</p>
      <p>质押信息: {JSON.stringify(userStakes)}</p>
    </div>
  )
}
```

### 常见用法

#### 1. 获取用户所有 Farm 持仓

```tsx
import { useUserPool, useUserHandle, useFarms } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function UserFarmPositions() {
  const { account } = useAccount()

  // 获取用户的池子信息
  const { data: userPoolInfo } = useUserPool(account?.address)
  // 获取用户的质押信息
  const { data: userStakes } = useUserHandle({ userHandle: userPoolInfo })
  // 获取所有 Farm 信息
  const { data: farms } = useFarms()

  // 用户质押的 PID 列表
  const stakedPids = userPoolInfo?.pids ?? []

  return (
    <div>
      <h3>我的 Farm 持仓</h3>
      {stakedPids.length === 0 ? (
        <div>暂未参与任何 Farm</div>
      ) : (
        stakedPids.map((pid) => {
          const farm = farms?.[pid]
          const stake = userStakes?.[pid]
          return (
            <FarmPositionRow
              key={pid}
              pid={pid}
              farm={farm}
              stake={stake}
            />
          )
        })
      )}
    </div>
  )
}
```

#### 2. 组合用户奖励计算

```tsx
import { useUserPool, useUserHandle, useUserRewards } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function UserFarmRewards() {
  const { account } = useAccount()

  // 获取用户池子信息
  const { data: userPoolInfo } = useUserPool(account?.address)
  // 获取质押信息
  const { data: userStakes } = useUserHandle({ userHandle: userPoolInfo })
  // 获取 Farm 数据
  const { data: farms } = useFarms()

  const pids = userPoolInfo?.pids ?? []

  return (
    <div>
      <h3>我的待领取奖励</h3>
      {pids.map((pid) => (
        <div key={pid}>
          <p>Pool {pid}</p>
          {/* 计算该池子的待领取奖励 */}
          <UserRewardForPool pid={pid} />
        </div>
      ))}
    </div>
  )
}

function UserRewardForPool({ pid }: { pid: number }) {
  const { account } = useAccount()

  const { data: userPoolInfo } = useUserPool(account?.address)
  const { data: userStakes } = useUserHandle({ userHandle: userPoolInfo })
  const { data: farms } = useFarms()

  // 获取 PID 索引
  const pIdIndex = userPoolInfo?.pids.findIndex((p) => p === pid)
  const farmIndex = pid

  // 使用 useUserRewards 计算奖励
  const reward = useUserRewards({
    farms,
    stakes: userStakes,
    pIdIndex,
    farmIndex
  })

  return <span>待领: {reward}</span>
}
```

#### 3. 检测用户是否在某个池子质押

```tsx
import { useUserPool } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function IsStakingInPool({ pid }: { pid: number }) {
  const { account } = useAccount()

  // 获取用户池子信息
  const { data: userPoolInfo } = useUserPool(account?.address)

  // 检查是否在指定池子质押
  const isStaking = userPoolInfo?.pids.includes(pid) ?? false

  return (
    <div>
      {isStaking ? (
        <button>进入池子</button>
      ) : (
        <button>质押获取奖励</button>
      )}
    </div>
  )
}
```

### Do（推荐做法）

- **使用定时刷新**：质押信息变化频繁，设置 2 秒刷新
- **分步调用**：先 useUserPool 再 useUserHandle
- **组合 useUserRewards 计算奖励**：获取质押信息后计算奖励
- **处理 undefined**：用户可能没有质押任何池子

### Don't（不推荐做法）

- **不要跳过 useUserPool 直接调用 useUserHandle**：需要先获取 PoolUserInfo
- **不要设置过长的刷新间隔**：质押操作需要快速响应
- **不要假设用户有质押**：需要检查 pids 是否为空

### 相关的其他 hooks

- `useFarms` *(一个React hook，获取所有Farm池子信息，返回FarmLP数据结构，包含alloc_point、lps等字段)*：获取 Farm 信息
- `useUserRewards` *(一个React hook，计算用户待领取的Farm奖励，根据用户质押信息和Farm参数计算可领取数量)*：计算待领取奖励
- `useTokenBalance` *(一个React hook，用于查询单个代币余额，接受account、currency、enabled等参数)*：查询 LP 代币余额
- `PoolUserInfo` *(一个TypeScript类型/配置，表示用户在池子中的持仓信息，包含pids数组等字段)*：用户池子信息类型
