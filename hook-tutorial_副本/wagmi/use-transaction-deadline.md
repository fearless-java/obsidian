> 源代码路径: `apps/web/src/lib/wagmi/hooks/utils/hooks/useTransactionDeadline.ts`

# useTransactionDeadline

## 大白话讲讲这个hook的作用

`useTransactionDeadline` 是一个计算交易截止时间（Deadline）的 Hook。

大白话：就是"给这笔交易设置一个过期时间"。如果交易在这个时间之前没有被执行，就失效了，不能再执行。

它会：
1. 获取当前区块时间戳
2. 加上用户设置的 TTL（Time To Live）分钟数
3. 返回一个未来的区块时间戳作为 deadline

这样可以防止交易被恶意延后执行（MEV 攻击等）。

## 讲讲为什么需要封装该hook

1. **动态 TTL**：用户可以自定义 TTL 值，封装后统一处理。

2. **链差异**：L2 链的 TTL 较短（5 分钟 vs 30 分钟）。

3. **自动刷新**：每 60 秒刷新一次，避免使用过期的 deadline。

4. **默认 TTL**：如果用户没有设置，使用链的默认值。

5. **存储偏好**：TTL 值存在 localStorage，用户偏好持久化。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseTransactionDeadline {
  chainId: EvmChainId
  enabled?: boolean
  storageKey: TTLStorageKey  // TTL 偏好的存储 key
}
```

### 输出 (Outputs)

```typescript
{
  data: bigint | undefined  // deadline 时间戳
  isLoading: boolean,
  isError: boolean,
}
```

### 执行逻辑详解

#### 1. 常量定义

```typescript
const L2_TTL = 5n   // L2 链 5 分钟
const TTL = 30n    // L1 链 30 分钟

export const getDefaultTTL = (chainId: EvmChainId) => {
  return getEvmChainById(chainId).parentChainId ? L2_TTL : TTL
}
```

#### 2. 获取区块时间戳

```typescript
const { data: currentBlockTimestampQuery } = useCurrentBlockTimestamp(
  chainId,
  enabled,
)
```

#### 3. 获取用户 TTL

```typescript
const [_ttl] = useTTL(storageKey)  // 从 localStorage 读取
```

#### 4. 计算 Deadline

```typescript
return useQuery({
  queryKey: ['useTransactionDeadline', _ttl],
  queryFn: () => {
    const blockTimestamp = currentBlockTimestampQuery
    let data = undefined

    const ttl = _ttl > 0 ? BigInt(_ttl) : getDefaultTTL(chainId)

    if (blockTimestamp) {
      data = blockTimestamp + ttl * 60n  // ttl 分钟转换为秒
    }

    return data
  },
  refetchInterval: 60_000,  // 60 秒刷新
  staleTime: 60_000,
  enabled: Boolean(enabled && currentBlockTimestampQuery),
})
```

### 数据流向图

```
输入: chainId, storageKey
         ↓
    ┌────────────────────────────────────┐
    │  useCurrentBlockTimestamp          │
    │  (获取当前区块时间戳)                │
    └────────────────────────────────────┘
         ↓
    ┌────────────────────────────────────┐
    │  useTTL(storageKey)                │
    │  (获取用户偏好的 TTL 分钟数)        │
    └────────────────────────────────────┘
         ↓
    计算: blockTimestamp + ttl * 60n
         ↓
    ┌────────────────────────────────────┐
    │  refetchInterval: 60000           │
    │  (每 60 秒刷新)                     │
    └────────────────────────────────────┘
         ↓
    返回 deadline (bigint)
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：TTL是分钟单位**。用户设置偏好的时候是按分钟算的，比如设置30分钟。但以太坊合约用的是秒单位。所以计算时要乘以60，把分钟转成秒。

**第二点：L2链有不同判断**。L2链（如Arbitrum、Optimism）和L1链（如以太坊主网）的默认TTL不同。怎么判断？L2链有parentChainId这个属性，通过它可以区分。

**第三点：用户偏好优先**。如果用户设置了自定义的TTL，就用用户的设置。如果用户没设置（值为0），才用链的默认值。

**第四点：60秒刷新一次**。区块时间戳每60秒刷新一次。这个间隔既能保证数据不会太旧，又不会对RPC造成太大压力。

**第五点：链ID变化会自动刷新**。queryKey里包含chainId，所以切换链的时候queryKey会变，React Query会自动重新获取新链的数据。

### 约束条件要记住

**第一，只支持EVM链**。这个hook只能在以太坊、Polygon、BSC这些EVM兼容链上用。非EVM链不支持。

**第二，TTL有最大值**。不能设置一个非常大的TTL值，这可能是为了防止用户误操作或者安全考虑。

**第三，L2和L1的TTL不同**。L2默认5分钟，L1默认30分钟。因为L2处理交易更快，可以用更短的TTL。

**第四，返回的是区块时间戳**。这不是普通的Unix时间戳，是区块链的区块时间戳。两者格式可能不同，合约会按区块时间戳来处理。

### 状态管理要清楚

数据有三个来源。currentBlockTimestamp是当前区块的时间戳。_ttl是用户设置的分钟数，存在localStorage里。deadline是计算结果，用blockTimestamp加上ttl对应的秒数。

刷新策略是每60秒刷新一次。因为staleTime也设置为60秒，所以数据一旦超过60秒就会被认为是过期的，需要重新获取。

### 常见错误要避开

**第一个坑：单位搞混**。TTL是分钟，合约用秒。一定要乘以60n进行转换。如果忘记乘以60，deadline会变得巨大。

**第二个坑：L2判断错误**。不能用chainId本身来判断是不是L2。必须用parentChainId这个属性。L2链的parentChainId指向主链，而L1链没有这个属性。

**第三个坑：用户设为0的情况**。如果用户没有设置过TTL，存储的值可能是0。这时候要用默认值而不是0。

**第四个坑：区块时间和Unix时间的区别**。返回的不是我们平常用的Unix时间戳。如果你的合约期望的是Unix时间戳，需要转换一下。

**第五个坑：60秒后的准确性问题**。deadline是基于获取时刻的区块时间戳计算的。60秒后区块时间已经往前走了，如果这时候才用这个deadline，可能会有几秒的误差。

### 提示词模板

```markdown
帮我创建一个计算交易截止时间的hook。

需求：
- 获取当前区块时间戳
- 结合用户偏好的TTL计算deadline
- L2链用更短的TTL
- 定时刷新保持数据新鲜

技术栈：
- React + TypeScript
- wagmi v2 + viem
- @tanstack/react-query v5
- sushi库hooks

常量：
- L1默认TTL：30分钟
- L2默认TTL：5分钟
- 用parentChainId判断是否为L2

输入：
- chainId：链ID
- enabled：是否启用
- storageKey：用户TTL偏好的存储key

输出：
- data：deadline区块时间戳
- isLoading：是否加载中
- isError：是否出错

计算逻辑：
1. 获取当前区块时间戳
2. 获取用户设置的TTL（分钟）
3. 如果TTL为0，用默认值
4. 用blockTimestamp + ttl * 60n得到deadline

注意：
- TTL是分钟单位，转秒要乘60
- 用parentChainId判断L2，不是chainId
- 用户没设置时用默认值
- 返回的是区块时间戳不是Unix时间
```

### 实际避坑指南

第一个，60的乘法。ttl是分钟，合约要秒。一定要ttl * 60n，忘记这点的后果是deadline变成几十年的有效期。

第二个，parentChainId判断L2。别用chainId直接判断，Polygon的chainId不是以太坊的，但都不是L2。正确的方式是检查parentChainId属性。

第三个，用户偏好设置。用户的偏好存在localStorage里。代码里用useTTL hook来读取这个偏好。如果偏好不存在或者为0，就用默认值。

第四个，刷新策略。每60秒刷新一次staleTime也是60秒，这意味着超过60秒的数据会被标记为过期。但实际的deadline值是基于获取时计算的，60秒后使用可能有误差。

2. **L2 判断**：必须用 `parentChainId` 判断，不是 chainId 本身。

3. **0 值处理**：用户 TTL 为 0 时使用默认值。

4. **区块时间 vs Unix 时间**：返回的是区块时间戳，不是标准的 Unix timestamp。

5. **过期风险**：如果 60 秒后才用这个 deadline，可能已经不太准确了。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useTransactionDeadline` 用于获取交易截止时间。基本用法如下：

```typescript
import { useTransactionDeadline } from '@sushiswap/wag'

function SwapSettings() {
  const { data: deadline } = useTransactionDeadline({
    chainId,
    storageKey: 'swap-ttl',
    enabled: Boolean(chainId),
  })

  return (
    <div>
      交易截止时间: {deadline ? `${Number(deadline)}` : '加载中...'}
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景一：在交易时传入 deadline

```typescript
function SwapButton({ tokenIn, tokenOut, amountIn }) {
  const { data: deadline } = useTransactionDeadline({
    chainId,
    storageKey: 'swap-ttl',
  })

  const { write } = useWriteContract({ ... })

  const executeSwap = () => {
    write({
      // ... swap 参数
      deadline, // 传入 deadline
    })
  }

  return <button onClick={executeSwap}>交换</button>
}
```

#### 场景二：显示用户友好的剩余时间

```typescript
function DeadlineDisplay({ deadline }) {
  // deadline 是区块时间戳
  const { data: currentBlock } = useCurrentBlockTimestamp(chainId)

  if (!deadline || !currentBlock) {
    return <div>计算中...</div>
  }

  const remaining = Number(deadline - currentBlock) // 秒
  const minutes = Math.floor(remaining / 60)

  return (
    <div>
      交易将在 {minutes} 分钟后过期
    </div>
  )
}
```

#### 场景三：自定义 TTL 设置

```typescript
function TTLSettings() {
  const [ttl, setTtl] = useState(30) // 默认 30 分钟

  // 用户可以在设置界面调整 TTL
  return (
    <div>
      <label>交易有效期 (分钟)</label>
      <select value={ttl} onChange={(e) => setTtl(Number(e.target.value))}>
        <option value={5}>5 分钟</option>
        <option value={15}>15 分钟</option>
        <option value={30}>30 分钟</option>
        <option value={60}>60 分钟</option>
      </select>
      {/* 保存到 localStorage */}
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **在交易时传入 deadline**
   ```typescript
   // DEX 交易应该使用 deadline 防止 MEV
   const { write } = useWriteContract({ ... })

   writeContract({
     // ... 其他参数
     deadline: deadline, // 防止交易被恶意延后
   })
   ```

2. **L2 使用较短 TTL**
   ```typescript
   // L2 链处理更快，可以使用较短 TTL
   // L1 使用较长 TTL
   // getDefaultTTL 会自动处理
   ```

3. **等待加载完成**
   ```typescript
   const { data: deadline } = useTransactionDeadline({ ... })

   if (!deadline) {
     return <div>计算中...</div>
   }

   // 使用 deadline
   ```

#### ❌ Don'ts

1. **不要使用过期的 deadline**
   ```typescript
   // 错误：使用 0 或硬编码值
   deadline: 0n

   // 正确：使用动态计算的 deadline
   deadline: deadlineFromHook
   ```

2. **不要对 L1/L2 使用相同的 TTL**
   ```typescript
   // 错误：L2 应该用更短的 TTL
   const ttl = 30n // 对 Arbitrum 来说太长了

   // 正确：使用 getDefaultTTL 自动判断
   const ttl = getDefaultTTL(chainId)
   ```

3. **不要忽略刷新**
   ```typescript
   // deadline 会随区块时间变化
   // 60 秒后应该重新获取
   ```

4. **不要在同一个区块内多次使用同一个 deadline**
   ```typescript
   // 每个交易应该用新的 deadline
   // 或者确保在同一个区块内发送
   ```
