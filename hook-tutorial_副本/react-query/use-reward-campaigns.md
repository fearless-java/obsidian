> 源代码路径: `apps/web/src/lib/hooks/react-query/rewards/useRewardCampaigns.ts`

# useRewardCampaigns Hook Tutorial

## 1. 大白话讲讲这个hook的作用

`useRewardCampaigns` 是用来查询某个特定池子的 Merkl 流动性激励campaign（活动）的 Hook。它返回该池子当前有哪些激励活动、每个活动的奖励代币是什么、活动时间范围、以及奖励数量。

简单来说：**当你想要知道"这个池子现在有哪些流动性挖矿奖励可以拿"，就用这个 Hook。它告诉你有哪些活动、奖励多少、是否还在进行中。**

---

## 2. 为什么需要封装该hook

### 原始问题
1. **Merkl API 格式复杂**：返回的 campaign 数据是嵌套的 JSON，需要用 Zod schema *(一个TypeScript优先的数据验证库，用于验证和转换API返回的数据)* 解析和类型转换
2. **时间状态计算**：需要根据当前时间判断活动是"进行中"还是"已结束"，`isLive = now >= startTimestamp && now <= endTimestamp`
3. **数据类型转换**：
   - `rewardToken` 字符串地址 -> `new EvmToken()` *(sushi库中的ERC20代币类型)* 对象
   - `amount` 字符串大数 -> `amount / 10 ** decimals` 人类可读数
4. **启用条件复杂**：只有当 pool 地址和 chainId 都存在时才应该请求

### 封装带来的好处
1. **自动类型转换**：返回可直接使用的 `EvmToken` 对象而非字符串地址
2. **实时状态计算**：自动计算 `isLive` 标志，无需前端手动判断
3. **数据精度处理**：大数除以 decimals 得到人类可读的代币数量
4. **缓存优化**：15 秒 staleTime 适合活动数据，1 分钟 GC 保留缓存

---

## 3. 执行逻辑和数据流向

### 输入 (Params)
```typescript
{
  pool: EvmAddress | undefined     // 池子地址
  chainId: EvmChainId | undefined  // 链ID
  enabled?: boolean = true         // 是否启用
}
```

### 输出 (Return)
```typescript
RewardCampaign[] = {
  id: string
  startTimestamp: number
  endTimestamp: number
  isLive: boolean                  // 当前时间是否在活动期间
  rewardToken: EvmToken           // 奖励代币对象
  amount: number                   // 奖励数量（已转换 decimals）
  // ... 其他 merklCampaignsValidator 的字段
}[]
```

### 执行流程

```
1. useRewardCampaigns({ pool, chainId, enabled })
       |
       v
2. 检查 enabled && pool && chainId
       |
       v
3. 构建 URL: https://api.merkl.xyz/v4/campaigns
   参数: ?chainId=${chainId}&mainParameter=${poolAddress}&test=false
       |
       v
4. 调用 API 并用 merklCampaignsValidator.parse(json) 验证数据
       |
       v
5. 数据转换:
   a) 计算 isLive: now >= startTimestamp && now <= endTimestamp
   b) 转换 rewardToken: new EvmToken(parsed.rewardToken)
   c) 转换 amount: +parsed.amount / 10 ** parsed.rewardToken.decimals
       |
       v
6. 返回转换后的 RewardCampaign[] 数组
```

---

## 4. 怎么用 AI 写这个 Hook

### 写代码时要注意什么

1. **启用条件要同时满足多个参数**：不是只有一个开关，而是 pool 和 chainId 都存在才发请求。

2. **外部 API 数据要验证**：Merkl 返回的数据必须用 Zod validator 过一遍，确保格式没问题再用。

3. **活动时间要实时算**：活动是"进行中"还是"已结束"，要根据当前时间来判断，不要相信 API 返回的状态字段。

4. **大数要转成能看懂的数字**：区块链上的代币金额都是很大的整数，要除以 decimals 才能变成人类能认出来的数字。

5. **缓存时间要适中**：活动数据不会变来变去，15 秒的 staleTime 比较合理，不用频繁刷新。

### 有什么限制条件

1. **pool 和 chainId 缺一不可**：两个参数必须同时存在，差一个都不应该发请求。

2. **依赖 Merkl API**：API 地址是固定的 `https://api.merkl.xyz/v4/campaigns`。

3. **依赖 validator**：返回的数据结构必须符合 `merklCampaignsValidator` 定义的格式。

4. **isLive 是算出来的**：这不是 API 返回的字段，而是每次查询时根据当前时间现场计算的。

### 状态怎么管理

| 状态 | 怎么管 | 什么意思 |
|------|--------|---------|
| 活动数据 | React Query 缓存 | 按 [pool, chainId] 来缓存 |
| isLive 状态 | 实时计算 | 每次查询都用当前时间重新算一遍 |
| 加载/错误 | React Query 标准 | 自动处理 |
| 缓存时间 | staleTime: 15s, gcTime: 60s | 短期缓存，1分钟后清理 |
| 启用开关 | enabled && pool && chainId | 三个条件同时满足才查 |

---

### 完整AI提示词模板

```
你是一个 React Query + DeFi 专家。请为以下场景编写 Hook:

【场景描述】
需要查询某个特定池子的 Merkl 流动性激励活动（Campaign）。
Merkl 是 SushiSwap 的链上流动性激励协议，池子运营者可以创建激励活动吸引流动性。

【技术要求】
1. 使用 @tanstack/react-query useQuery
2. 调用 Merkl API: https://api.merkl.xyz/v4/campaigns
3. 参数: chainId, mainParameter=pool地址, test=false
4. 使用 merklCampaignsValidator 验证返回数据
5. 计算 isLive: 当前时间是否在 [startTimestamp, endTimestamp] 区间

【数据转换】
1. rewardToken: 字符串地址 -> new EvmToken(parsed.rewardToken)
2. amount: 大数字符串 / 10 ** decimals -> 人类可读数字

【参数】
interface UseRewardCampaignsParams {
  pool: EvmAddress | undefined
  chainId: EvmChainId | undefined
  enabled?: boolean
}

【返回】
RewardCampaign[] = {
  isLive: boolean       // 当前是否进行中
  rewardToken: EvmToken // 奖励代币对象
  amount: number        // 奖励数量
  // ... 其他活动字段
}

【启用条件】
enabled: Boolean(enabled && pool && chainId)

【缓存配置】
- staleTime: 15000 (15秒)
- gcTime: 60000 (1分钟)

【最佳实践】
- 用 Zod validator 验证外部 API 数据
- isLive 基于当前时间计算，不是 API 字段
- 大数 / decimals 转换为可读数量
- queryKey 包含 pool 和 chainId

请输出完整代码。
```

---

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useRewardCampaigns } from '@sushiswap/react-query'

function PoolIncentives({ poolAddress, chainId }: { poolAddress: string; chainId: number }) {
  const { data: campaigns, isLoading } = useRewardCampaigns({
    pool: poolAddress,
    chainId,
  })

  if (isLoading) return <div>Loading...</div>
  if (!campaigns?.length) return <div>No active campaigns</div>

  return (
    <div>
      <h3>Active Incentives</h3>
      {campaigns.map((campaign) => (
        <CampaignCard key={campaign.id} campaign={campaign} />
      ))}
    </div>
  )
}

function CampaignCard({ campaign }) {
  return (
    <Card>
      <CardBody>
        <div>Reward Token: {campaign.rewardToken.symbol}</div>
        <div>Amount: {campaign.amount.toSignificant(4)}</div>
        <div>Status: {campaign.isLive ? 'Live' : 'Ended'}</div>
        <div>
          Period: {new Date(campaign.startTimestamp * 1000).toLocaleDateString()} -{' '}
          {new Date(campaign.endTimestamp * 1000).toLocaleDateString()}
        </div>
      </CardBody>
    </Card>
  )
}
```

### 常见使用场景

**场景1：池子详情页的激励信息展示**
```tsx
function PoolDetailIncentives({ pool }: { pool: V3Pool }) {
  const { data: campaigns } = useRewardCampaigns({
    pool: pool.address,
    chainId: pool.chainId,
  })

  const liveCampaigns = campaigns?.filter((c) => c.isLive)
  const totalRewards = liveCampaigns?.reduce((sum, c) => sum + c.amount, 0) ?? 0

  return (
    <div>
      <h3>Incentive Campaigns ({liveCampaigns?.length ?? 0} active)</h3>
      {totalRewards > 0 && <div>Total Rewards: {totalRewards.toSignificant(4)}</div>}
      {liveCampaigns?.map((campaign) => (
        <LiveCampaignRow key={campaign.id} campaign={campaign} />
      ))}
    </div>
  )
}
```

**场景2：活动即将开始/结束的提醒**
```tsx
function UpcomingCampaigns({ poolAddress, chainId }: { poolAddress: string; chainId: number }) {
  const { data: campaigns } = useRewardCampaigns({
    pool: poolAddress,
    chainId,
  })

  const now = Date.now() / 1000

  const upcomingCampaigns = campaigns?.filter((c) => c.startTimestamp > now)
  const endingSoon = campaigns?.filter((c) => c.isLive && c.endTimestamp - now < 86400 * 3) // 3天内结束

  return (
    <div>
      {upcomingCampaigns?.length > 0 && (
        <section>
          <h4>Upcoming Campaigns</h4>
          {upcomingCampaigns.map((c) => (
            <Alert key={c.id} message={`${c.rewardToken.symbol} rewards starting soon!`} />
          ))}
        </section>
      )}
      {endingSoon?.length > 0 && (
        <section>
          <h4>Ending Soon</h4>
          {endingSoon.map((c) => (
            <Alert key={c.id} type="warning" message={`${c.rewardToken.symbol} ending in 3 days!`} />
          ))}
        </section>
      )}
    </div>
  )
}
```

### Dos and Don'ts

**Do（推荐做法）：**
- ✅ 使用 `isLive` 字段过滤出当前正在进行中的活动
- ✅ 使用 `rewardToken` 对象的完整代币信息（symbol, decimals 等）
- ✅ 使用 `Amount` 类型的 `.toSignificant()` 格式化显示奖励数量
- ✅ 同时展示 live 和已结束的活动，让用户了解历史活动

**Don't（避免做法）：**
- ❌ 不要假设 campaigns 数组一定有数据，应该检查长度
- ❌ 不要忽略 `isLive` 的实时计算，每次查询都会重新计算
- ❌ 不要直接显示原始 `amount` 数字，应该用代币 decimals 转换
- ❌ 不要在 pool 或 chainId 为空时调用，应该先检查

### 注意事项

1. **isLive 是实时计算的**：每次查询都会根据当前时间重新计算 `isLive`，所以活动开始或结束后状态会自动更新

2. **返回数组而非对象**：这个 hook 返回的是 `RewardCampaign[]` 数组，一个池子可能有多个活动

3. **奖励代币是完整的 EvmToken 对象**：可以直接使用 `campaign.rewardToken.symbol`、`campaign.rewardToken.decimals` 等

4. **15秒缓存**：活动数据15秒后变 stale，适合活动这种不会频繁变化的数据

5. **时间戳是秒级**：API 返回的 timestamp 是秒级 JavaScript 时间戳，需要乘以 1000 转换为 Date 对象
