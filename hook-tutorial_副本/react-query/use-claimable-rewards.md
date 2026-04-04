> 源代码路径: `apps/web/src/lib/hooks/react-query/rewards/useClaimableRewards.ts`

# useClaimableRewards Hook Tutorial

## 1. 大白话讲讲这个hook的作用

`useClaimableRewards` 是用来查询用户在 SushiSwap 的 Merkl 流动性挖矿中可领取奖励的 Hook。它会查询用户在所有支持的链上的未领取奖励金额，并把这些数据整理成可以直接用于"领取"交易的数据格式（claimArgs）。

简单来说：**这个 Hook 就是帮你查"你能领多少钱"，以及"怎么领"——它不仅告诉你有多少奖励，还准备好了领取交易需要的参数。**

---

## 2. 为什么需要封装该hook

### 原始问题
1. **多链并行查询**：Merkl API 需要在多条链上分别查询（SushiSwap 支持多条 EVM 链），需要用 `Promise.allSettled` *(一个Promise方法，用于并行执行多个Promise，即使部分失败也不会导致整体失败)* 并行请求
2. **复杂的数据聚合**：多个池子的同种代币奖励需要合并（如两个 ETH/DAI 池子的 DAI 奖励要累加）
3. **领取参数构建**：Merkl 领取需要特殊的 claimArgs 格式，包含 `[receivers[], tokens[], amounts[], proofs[][]]`，这个构建逻辑复杂
4. **实时状态计算**：需要计算"未领取 = 总奖励 - 已领取"，并判断是否大于 0

### 封装带来的好处
1. **多链聚合**：自动在所有 MERKL_SUPPORTED_CHAIN_IDS *(Merkl协议支持的链ID数组)* 上并行查询并聚合结果
2. **数据合并**：同一链上相同代币的奖励自动累加
3. **零值过滤**：已领取完的奖励（unclaimed === 0n *(bigint的0值，用于精确的整数计算)*）不显示
4. **USD 换算**：同时计算出每个奖励的 USD 价值和总 USD 价值
5. **可直接领取**：返回的 claimArgs 可直接用于构建领取交易

---

## 3. 执行逻辑和数据流向

### 输入 (Params)
```typescript
{
  chainIds?: MerklChainId[]          // 要查询的链ID数组，默认所有Merkl支持的链
  account: Address | undefined        // 用户钱包地址
  enabled?: boolean = true            // 是否启用
}
```

### 输出 (Return)
```typescript
{
  [chainId: MerklChainId]: {
    chainId: MerklChainId                        // 链ID
    rewardAmounts: Record<string, Amount>         // 代币地址 -> 金额
    rewardAmountsUSD: Record<string, number>     // 代币地址 -> USD价值
    totalRewardsUSD: number                       // 该链总奖励USD价值
    claimArgs: [Address[], Address[], bigint[], Hex[][]]
      // [0]: receivers - 领取人地址数组
      // [1]: tokens - 代币地址数组
      // [2]: amounts - 金额数组
      // [3]: proofs -  merkle 证明数组
  }
}
```

### 执行流程

```
1. useClaimableRewards({ chainIds, account })
       |
       v
2. 检查 enabled && account，不满足则不请求
       |
       v
3. 构建 URL: https://api.merkl.xyz/v4/users/${account}/rewards
       |
       v
4. 并行查询所有链 (Promise.allSettled):
   - 对每个 chainId 调用 API
   - 用 merklRewardsValidator 验证返回数据
       |
       v
5. 过滤和聚合数据:
   a) 过滤 status !== 'fulfilled' 或结果为空的数据
   b) 计算 unclaimed = reward.amount - reward.claimed
   c) 过滤 unclaimed === 0n 的奖励
   d) 按 chainId 分组
   e) 同链同代币合并 (accumulate amounts)
       |
       v
6. 构建 claimArgs:
   - claimArgs[0].push(account)         // 领取人
   - claimArgs[1].push(tokenAddress)    // 代币地址
   - claimArgs[2].push(reward.amount)   // 金额
   - claimArgs[3].push(reward.proofs)   // merkle证明
       |
       v
7. 计算 USD 价值:
   - unclaimedAmount * token.price = USD value
   - 累加到 totalRewardsUSD
       |
       v
8. 返回按 chainId 组织的数据结构
```

---

## 4. 怎么用 AI 写这个 Hook

### 写代码时要注意什么

1. **多链查询要用 Promise.allSettled**：同时查多条链，如果有一条链的请求失败了，其他的正常返回，不会互相影响。

2. **金额计算要用 bigint 或 Amount 类型**：奖励金额是区块链上的大数，要用专门的类型来处理，不能用普通的数字。计算方式是 `unclaimed = amount - claimed`。

3. **数据按链 ID 分组**：返回的数据结构用 `Record<chainId, ...>` 来组织，这样每条链的数据是分开的。

4. **已经领完的不要显示**：unclaimed === 0 的奖励就别返回了，不然页面上会有一堆零值，看着乱。

5. **外部数据要验证**：Merkl API 返回的数据要用 Zod validator 验证一下格式对不对，不要直接用。

### 有什么限制条件

1. **必须有 account**：用户钱包地址是必须的，没有的话就不发请求。

2. **默认查所有支持的链**：如果不指定 chainIds，就会一次性查所有 Merkl 支持的链。

3. **依赖 Merkl API**：这个 Hook 硬编码了 Merkl 的 API 地址 `https://api.merkl.xyz/v4/users/${account}/rewards`，换地址就不行了。

4. **领取需要特殊的参数格式**：返回的 claimArgs 是 Merkl 合约规定的格式，不是普通的交易参数。

5. **依赖 validator**：必须用 `merklRewardsValidator` 来验证返回的数据格式。

### 状态怎么管理

| 状态 | 怎么管 | 什么意思 |
|------|--------|---------|
| 奖励数据 | React Query 缓存 | 按 account 作为缓存的键 |
| 多链数据 | Record 结构 | `{ [chainId]: { rewardAmounts, claimArgs } }` |
| 加载/错误 | React Query 标准 | 自动处理 |
| 定时刷新 | staleTime: 15s | 15秒后数据过期，会重新拉 |
| 启用开关 | enabled 参数 | 要同时满足 account && enabled 才会查 |

---

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useClaimableRewards } from '@sushiswap/react-query'
import { useAccount } from 'wagmi'

function RewardsPanel() {
  const { address } = useAccount()
  const { data: rewards, isLoading } = useClaimableRewards({ account: address })

  if (isLoading) return <div>Loading...</div>
  if (!rewards) return <div>No rewards data</div>

  return (
    <div>
      {Object.entries(rewards).map(([chainId, chainRewards]) => (
        <ChainRewardsCard key={chainId} chainId={chainId} rewards={chainRewards} />
      ))}
    </div>
  )
}

function ChainRewardsCard({ chainId, rewards }) {
  return (
    <Card>
      <CardHeader>Chain {chainId}</CardHeader>
      <CardBody>
        <div>Total USD: ${rewards.totalRewardsUSD.toFixed(2)}</div>
        <div>Tokens: {Object.keys(rewards.rewardAmounts).length}</div>
        <ClaimButton chainId={chainId} claimArgs={rewards.claimArgs} />
      </CardBody>
    </Card>
  )
}
```

### 常见使用场景

**场景1：奖励页面总览**
```tsx
function RewardsOverview() {
  const { address } = useAccount()
  const { data: rewardsByChain } = useClaimableRewards({ account: address })

  const totalUSD = Object.values(rewardsByChain ?? {}).reduce(
    (sum, chain) => sum + chain.totalRewardsUSD,
    0
  )

  const chainsWithRewards = Object.entries(rewardsByChain ?? {})
    .filter(([_, chain]) => chain.totalRewardsUSD > 0)
    .map(([chainId]) => Number(chainId))

  return (
    <div>
      <h1>Total Rewards: ${totalUSD.toFixed(2)}</h1>
      <h2>Active on {chainsWithRewards.length} chains</h2>
    </div>
  )
}
```

**场景2：单个链的奖励详情和领取**
```tsx
function ChainRewardsDetail({ chainId }: { chainId: number }) {
  const { address } = useAccount()
  const { data: rewards, isLoading } = useClaimableRewards({
    account: address,
    chainIds: [chainId],
  })

  const chainData = rewards?.[chainId]
  if (!chainData) return null

  return (
    <div>
      <h2>Chain {chainId} Rewards</h2>
      {Object.entries(chainData.rewardAmounts).map(([tokenAddress, amount]) => (
        <RewardRow
          key={tokenAddress}
          tokenAddress={tokenAddress}
          amount={amount}
          usdValue={chainData.rewardAmountsUSD[tokenAddress]}
        />
      ))}

      {chainData.totalRewardsUSD > 0 && (
        <ClaimAllButton claimArgs={chainData.claimArgs} totalUSD={chainData.totalRewardsUSD} />
      )}
    </div>
  )
}
```

### Dos and Don'ts

**Do（推荐做法）：**
- ✅ 使用 `account` 检查确保用户已连接钱包
- ✅ 使用 `Object.entries()` 遍历多链数据
- ✅ 使用 `totalRewardsUSD` 显示总价值，避免自己累加
- ✅ 使用 `claimArgs` 直接构建领取交易

**Don't（避免做法）：**
- ❌ 不要在没有 account 时调用这个 hook，应该先检查 address
- ❌ 不要自己计算 totalUSD，应该使用 `totalRewardsUSD` 字段
- ❌ 不要假设每条链都有奖励，应该检查 `totalRewardsUSD > 0`
- ❌ 不要在 `claimArgs` 为空时调用领取交易，应该检查长度

### 注意事项

1. **返回结构是 Record<chainId, ...>**：要遍历数据应该用 `Object.entries()` 而不是直接 map

2. **claimArgs 可以直接用于交易**：`claimArgs` 的格式是 `[Address[], Address[], bigint[], Hex[][]]`，可以直接传给合约的 claim 方法

3. **15秒缓存**：数据会在15秒后变 stale，期间不会重新请求

4. **多链聚合**：默认会查询所有 Merkl 支持的链，如果只关心特定链可以用 `chainIds` 参数限制
