> 源代码路径: `apps/web/src/lib/hooks/react-query/rewards/useRewardTokens.ts`

# useRewardTokens Hook Tutorial

## 1. 大白话讲讲这个hook的作用

`useRewardTokens` 是用来查询 Merkl 协议支持的所有奖励代币列表的 Hook。它返回某条链上可以被用作流动性激励的代币信息，包括每个代币的最小每小时奖励数量（minimumAmountPerHour）。

简单来说：**这个 Hook 告诉你"这条链上，池子运营者可以拿哪些代币来奖励流动性提供者"——它是 Merkl 活动创建页面需要的核心数据。**

---

## 2. 为什么需要封装该hook

### 原始问题
1. **API 返回原始数据**：Merkl API 返回的代币数据是扁平的结构，需要转换封装成 `EvmToken` *(sushi库中的ERC20代币类型)* 对象
2. **过滤测试代币**：API 可能返回测试用的代币（如 `aglaMerkl`），需要过滤掉
3. **decimals 校验**：某些代币的 decimals 可能为 null/undefined，需要过滤掉
4. **Amount 类型转换**：`minimumAmountPerHour` 需要封装成 `Amount` *(sushi库中的代币金额类型，同时包含数值和代币信息)* 类型

### 封装带来的好处
1. **开箱即用的代币数据**：返回的 `token` 字段是完整的 `EvmToken` 对象，可以直接用于交易
2. **干净的代币列表**：自动过滤测试代币和无效 decimals 的代币
3. **Amount 类型**：奖励数量已经封装好，可以直接用于 UI 显示或交易构建

---

## 3. 执行逻辑和数据流向

### 输入 (Params)
```typescript
{
  chainId: EvmChainId   // 链ID（必需参数）
}
```

### 输出 (Return)
```typescript
{
  minimumAmountPerHour: Amount<EvmToken>  // 最小每小时奖励量
  token: EvmToken                         // 代币对象
}[]
```

### 执行流程

```
1. useRewardTokens({ chainId })
       |
       v
2. 构建 URL: https://api.merkl.xyz/v4/tokens/reward/${chainId}
       |
       v
3. 调用 API 并用 merklRewardsTokensValidator 验证数据
       |
       v
4. 过滤数据:
   a) 过滤 decimals 为 null/undefined 的代币
   b) 过滤 symbol === 'aglaMerkl' 的测试代币
   c) 过滤 isTest === true 的代币
       |
       v
5. 转换数据:
   a) 构建 EvmToken: new EvmToken({ chainId, address, symbol, decimals })
   b) 转换 minimumAmountPerHour: new Amount(token, el.minimumAmountPerHour)
       |
       v
6. 返回数组
```

---

## 4. 怎么用 AI 写这个 Hook

### 写代码时要注意什么

1. **必需参数不要用 enabled 来控制**：chainId 是必须有的，没有就直接报错，不发请求，而不是用一个 enabled 开关来跳过。

2. **过滤逻辑要写清楚**：为什么要过滤某个代币，注释里要说明白，方便以后的人维护。

3. **EvmToken 构造要完整**：symbol 和 name 可以是空字符串，但 decimals 必须有值且有效。

4. **返回的数据要能直接用**：token 是完整的 EvmToken 对象，minimumAmountPerHour 是 Amount 类型，外面直接拿来用就行。

### 有什么限制条件

1. **chainId 是必传的**：不是可选项，这个 Hook 和 useClaimableRewards 不一样，没有就是没有，不能跳过。

2. **依赖 Merkl API**：API 地址是 `https://api.merkl.xyz/v4/tokens/reward/${chainId}`。

3. **依赖 validator**：必须用 `merklRewardsTokensValidator` 验证数据格式。

4. **decimals 不能是空的**：如果 decimals 是 falsy 值（null、undefined、0），就要过滤掉这个代币。

### 状态怎么管理

| 状态 | 怎么管 | 什么意思 |
|------|--------|---------|
| 代币列表 | React Query 缓存 | 按 chainId 缓存 |
| 加载/错误 | React Query 标准 | 自动处理 |
| 过滤状态 | 内存过滤 | 在内存里把测试代币和无效代币过滤掉 |

---

### 完整AI提示词模板

```
你是一个 React Query + DeFi 专家。请为以下场景编写 Hook:

【场景描述】
需要查询 Merkl 协议在某条链上支持的奖励代币列表。
这个列表用于 Merkl 活动创建页面，让池子运营者选择用什么代币来奖励流动性提供者。

【技术要求】
1. 使用 @tanstack/react-query useQuery
2. 调用 Merkl API: https://api.merkl.xyz/v4/tokens/reward/${chainId}
3. 使用 merklRewardsTokensValidator 验证数据
4. 过滤条件:
   - decimals 必须存在
   - symbol !== 'aglaMerkl' (测试代币)
   - isTest !== true

【数据转换】
1. 构建 EvmToken: new EvmToken({ chainId, address, symbol, name: '', decimals })
2. 转换 minimumAmountPerHour: new Amount(token, el.minimumAmountPerHour)

【参数】
interface UseAngleRewardTokensParams {
  chainId: EvmChainId  // 必需参数
}

【返回】
{
  minimumAmountPerHour: Amount<EvmToken>  // 最小每小时奖励量
  token: EvmToken                         // 代币对象
}[]

【注意】
- chainId 是必需参数
- 返回的 token 可以直接用于交易构建
- 过滤掉无效/测试代币保证数据质量

请输出完整代码。
```

---

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useRewardTokens } from '@sushiswap/react-query'

function RewardTokenSelector({ chainId, selectedToken, onSelect }) {
  const { data: rewardTokens, isLoading } = useRewardTokens({ chainId })

  if (isLoading) return <div>Loading reward tokens...</div>
  if (!rewardTokens?.length) return <div>No reward tokens available</div>

  return (
    <select
      value={selectedToken?.address}
      onChange={(e) => {
        const token = rewardTokens.find((t) => t.token.address === e.target.value)
        onSelect(token)
      }}
    >
      <option value="">Select reward token</option>
      {rewardTokens.map(({ token, minimumAmountPerHour }) => (
        <option key={token.address} value={token.address}>
          {token.symbol} (min: {minimumAmountPerHour.toSignificant(4)}/hr)
        </option>
      ))}
    </select>
  )
}
```

### 常见使用场景

**场景1：Merkl 活动创建表单**
```tsx
function CreateIncentiveForm({ poolAddress, chainId }) {
  const { data: rewardTokens } = useRewardTokens({ chainId })
  const [selectedRewardToken, setSelectedRewardToken] = useState(null)
  const [rewardAmount, setRewardAmount] = useState('')

  const handleCreate = async () => {
    if (!selectedRewardToken || !rewardAmount) return

    // 使用 EvmToken 直接构建交易
    const token = selectedRewardToken.token
    const amount = parseUnits(rewardAmount, token.decimals)

    // 创建活动逻辑...
  }

  return (
    <Form>
      <FormField label="Reward Token">
        <RewardTokenSelector
          chainId={chainId}
          selectedToken={selectedRewardToken}
          onSelect={setSelectedRewardToken}
        />
      </FormField>

      <FormField label="Reward Amount">
        <Input
          value={rewardAmount}
          onChange={(e) => setRewardAmount(e.target.value)}
          placeholder="Enter amount"
        />
        <span className="text-xs text-gray-500">
          Min: {selectedRewardToken?.minimumAmountPerHour.toSignificant(4)}/hr
        </span>
      </FormField>

      <Button onClick={handleCreate}>Create Campaign</Button>
    </Form>
  )
}
```

**场景2：展示可用的奖励代币列表**
```tsx
function AvailableRewardTokens({ chainId }: { chainId: number }) {
  const { data: tokens } = useRewardTokens({ chainId })

  return (
    <div>
      <h3>Available Reward Tokens on Chain {chainId}</h3>
      <div className="grid grid-cols-3 gap-4">
        {tokens?.map(({ token, minimumAmountPerHour }) => (
          <TokenCard key={token.address} token={token} minReward={minimumAmountPerHour} />
        ))}
      </div>
    </div>
  )
}

function TokenCard({ token, minReward }) {
  return (
    <Card>
      <CardHeader>
        <img src={token.logoUri} alt={token.symbol} className="w-8 h-8" />
        <span>{token.symbol}</span>
      </CardHeader>
      <CardBody>
        <div className="text-sm text-gray-500">
          Min hourly reward: {minReward.toSignificant(6)}
        </div>
        <div className="text-xs text-gray-400">
          Decimals: {token.decimals}
        </div>
      </CardBody>
    </Card>
  )
}
```

### Dos and Don'ts

**Do（推荐做法）：**
- ✅ 使用返回的 `token` 对象的完整信息（symbol, logoUri, decimals 等）
- ✅ 展示 `minimumAmountPerHour` 告诉用户最小奖励量
- ✅ 使用 `rewardTokens` 数组的解构 `{ token, minimumAmountPerHour }`
- ✅ 直接使用 `token` 构建交易，因为它是完整的 EvmToken

**Don't（避免做法）：**
- ❌ 不要直接使用 API 返回的原始数据，要使用经过转换的 EvmToken
- ❌ 不要假设所有代币都有 logo，logoUri 可能是 undefined
- ❌ 不要忽略 decimals，过滤后的代币 decimals 一定存在但仍需使用
- ❌ 不要在没有数据时显示空，应该显示 "No reward tokens available"

### 注意事项

1. **chainId 是必需参数**：这个 hook 没有 `enabled` 控制，必须传入有效的 chainId

2. **返回数组结构**：每个元素是 `{ token: EvmToken, minimumAmountPerHour: Amount<EvmToken> }`

3. **自动过滤**：测试代币（aglaMerkl, isTest=true）和无效 decimals 的代币已被过滤

4. **可直接用于交易**：返回的 `token` 是完整的 EvmToken，可以直接用于构建交易

5. **没有缓存时间配置**：这个 hook 使用默认的 React Query 缓存策略
