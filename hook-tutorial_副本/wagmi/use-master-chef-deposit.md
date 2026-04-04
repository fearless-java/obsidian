> 源代码路径: `apps/web/src/lib/wagmi/hooks/master-chef/use-master-chef-deposit.ts`

# useMasterChefDeposit

## 大白话讲讲这个hook的作用

`useMasterChefDeposit` 是一个用于在 MasterChef 池子中质押（存款）的 Hook。

大白话：就是"把 LP 代币存进去，开始赚 SUSHI 奖励"。你把 LP 代币质押到 MasterChef 合约，它会记录你的质押量并开始计算奖励。

它的核心操作是调用 MasterChef 合约的 `deposit()` 函数，将 LP 代币质押进去。

## 讲讲为什么需要封装该hook

1. **多版本兼容**：MasterChef V1、V2、 MiniChef 的 deposit 函数签名不同，需要分别处理。

2. **模拟验证**：使用 `useSimulateContract` 预验证交易。

3. **用户体验**：封装 Toast 通知、Pending 状态、错误处理。

4. **Amount 转换**：需要把 `Amount<Token>` 转换成 `bigint` 传入合约。

5. **返回值统一**：封装后返回格式与 withdraw 等操作统一。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseMasterChefDepositParams {
  chainId: EvmChainId | undefined  // 链 ID
  chef: ChefType                    // MasterChef 版本
  pid: number                       // Pool ID
  amount?: Amount<EvmToken>          // 质押数量
  enabled?: boolean
}
```

### 输出 (Outputs)

```typescript
{
  write: () => Promise<void>,   // 存款函数
  isPending: boolean,
  isError: boolean,
}
```

### 执行逻辑详解

#### 1. 获取合约

```typescript
const contract = useMasterChefContract(chainId, chef)
```

#### 2. 构造 Deposit 数据

```typescript
const prepare = useMemo(() => {
  if (!address || !chainId || !amount || !contract) return {}

  let data
  if (chef === ChefType.MasterChefV1) {
    data = {
      abi: masterChefV1Abi_deposit,
      functionName: 'deposit',
      args: [BigInt(pid), BigInt(amount.amount.toString())],
    }
  } else {
    data = {
      abi: masterChefV2Abi_deposit,
      functionName: 'deposit',
      args: [BigInt(pid), BigInt(amount.amount.toString()), address],
    }
  }

  return {
    account: address,
    address: contract.address,
    ...data,
  }
}, [address, amount, chainId, chef, contract, pid])
```

#### 3. 模拟调用

```typescript
const { data: simulation } = useSimulateContract({
  ...prepare,
  chainId,
  query: { enabled },
})
```

#### 4. 发送交易

```typescript
const write = useMemo(() => {
  if (!simulation) return undefined

  return async () => {
    try {
      await writeContractAsync(simulation.request as any)
    } catch {}
}, [simulation?.request, writeContractAsync])
```

#### 5. 成功回调

```typescript
const onSuccess = (data) => {
  createToast({
    account: address,
    type: 'mint',
    chainId,
    summary: {
      pending: `Staking ${amount.toSignificant(6)} ${amount.currency.symbol} tokens`,
      completed: `Successfully staked ${amount.toSignificant(6)} ${amount.currency.symbol} tokens`,
      failed: `Something went wrong when staking ${amount.currency.symbol} tokens`,
    },
  })
}
```

### 数据流向图

```
输入: chainId, chef, pid, amount
         ↓
    useMasterChefContract 获取合约
         ↓
    prepare: 构造 deposit 数据
    ┌────────────────────────────────────┐
    │  V1: deposit(pid, amount)        │
    │  V2: deposit(pid, amount, user)   │
    └────────────────────────────────────┘
         ↓
    useSimulateContract 模拟验证
         ↓
    useWriteContract 发送交易
         ↓
    onSuccess: Toast 通知
         ↓
    返回 { write, isPending, isError }
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：先approve再存款**。存款操作之前必须先授权MasterChef合约可以动用你的LP代币。这个授权只需要一次，以后就不用再授权了。但如果你换了一个新的池子，可能还需要再授权一次。

**第二点：金额要校验**。amount必须大于0，而且不能超过你钱包里实际的LP余额。如果超额了交易会失败，浪费gas。

**第三点：PID要正确**。PID是Pool ID，每个池子都有唯一的编号。如果PID错了，就把钱存到了错误的池子里。所以PID一定要和池子配置对应上。

**第四点：版本要选对**。MasterChef有三个版本：V1、V2、MiniChef。以太坊主网用V1或V2，其他链用MiniChef。版本选错了调用合约会失败。

**第五点：Toast要详细**。存款是真实资产操作，Toast通知要包含代币符号和数量。比如"质押 100 SUSHI-LP"，让用户确认自己操作的是正确的。

### 约束条件要记住

**第一，必须先approve LP**。没有授权，deposit调用会失败。授权和存款是独立的两步操作。

**第二，amount不能超过余额**。这个不用多解释，超额肯定失败。

**第三，PID必须对应池子**。PID和池子是一一对应的关系。错了就存错地方了。

**第四，只能操作支持的链**。V1和V2只在以太坊主网。Polygon、BSC等其他链要用MiniChef。

### 状态管理要清楚

交易状态有三个。write是执行存款的函数。isPending表示交易是否在pending中。isError表示是否出错。

准备状态也有两个。prepare是构造好的交易参数。simulation是模拟调用的结果，用来验证交易是否能成功。

### 常见错误要避开

**第一个坑：V1和V2的差异**。V1的deposit只能质押给自己，V2的deposit可以指定接收人地址参数。这个差异影响函数签名和参数个数。

**第二个坑：BigInt转换**。amount.amount返回的是bigint，但合约参数需要bigint类型。代码里用BigInt()包装确保类型正确。自己写的时候不要忘了转换。

**第三个坑：approve的时机**。通常的做法是先检查当前授权额度够不够，不够才发起授权。这样可以避免重复授权。

**第四个坑：LP地址的对应**。每个池子的LP代币地址都不同。你要存哪个池子，就要用哪个池子的LP代币地址去授权。

**第五个坑：余额检查**。建议在UI层面就做好余额检查，如果余额不足就直接禁用存款按钮，而不是让交易失败。

### 提示词模板

```markdown
帮我创建一个在MasterChef池子质押LP的hook。

需求：
- 把LP代币质押到MasterChef池子
- 支持V1、V2、MiniChef三个版本
- 交易前先模拟验证
- 显示详细的Toast信息

技术栈：
- React + TypeScript
- wagmi v2 + viem
- sushi库
- Toast通知
- useMasterChefContract

版本差异：
- V1：deposit(pid, amount)
- V2：deposit(pid, amount, to)
- MiniChef：deposit(pid, amount, to)

输入：
- chainId：链ID
- chef：版本类型
- pid：池子ID
- amount：质押数量
- enabled：是否启用

输出：
- write：执行存款函数
- isPending：是否等待中
- isError：是否出错

实现要点：
1. 根据版本选择正确的合约和函数
2. 构造deposit参数
3. 先模拟调用验证
4. 成功后Toast通知
5. Toast要包含代币信息

注意：
- 必须先approve才能存款
- 金额不能超过余额
- PID必须对应正确的池子
- 版本要选对
```

### 实际避坑指南

第一个，approve和deposit是分开的。两步操作不能合并，用户需要签两次名。这是合约设计的限制。

第二个，版本选择有讲究。以太坊主网用V1或V2，其他链用MiniChef。搞混了交易会失败。

第三个，PID要准确获取。PID不是随便设的，要从池子配置里读取。每个池子配置里有pid、lpToken地址等信息。

第四个，金额用toString()转换。amount.amount是BigInt类型，但合约需要字符串形式的数字。代码里会做这个转换。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useMasterChefDeposit` 用于在 MasterChef 池子质押 LP 代币。基本用法如下：

```typescript
import { useMasterChefDeposit, useMasterChef, ChefType } from '@sushiswap/wag'

function DepositForm({ chainId, chef, pid, lpToken, amount }) {
  const { write, isPending, isError } = useMasterChefDeposit({
    chainId,
    chef,
    pid,
    amount,
    enabled: Boolean(amount && amount.gt(0)),
  })

  return (
    <button onClick={write} disabled={isPending || !write}>
      {isPending ? '质押中...' : '质押 LP'}
    </button>
  )
}
```

### 5.2 常见使用场景

#### 场景一：质押 LP 并领取奖励

```typescript
function FarmStake({ farm, amount }) {
  // 质押
  const { write: deposit, isPending: depositing } = useMasterChefDeposit({
    chainId: farm.chainId,
    chef: farm.chefType,
    pid: farm.pid,
    amount,
  })

  // 领取奖励
  const { harvest, pendingSushi } = useMasterChef({
    chainId: farm.chainId,
    chef: farm.chefType,
    pid: farm.pid,
    token: farm.lpToken,
  })

  return (
    <div>
      <input value={amount.toString()} />
      <button onClick={deposit} disabled={depositing}>质押</button>
      <button onClick={harvest} disabled={!pendingSushi?.gt(0)}>
        领取 {pendingSushi?.toSignificant(6)} SUSHI
      </button>
    </div>
  )
}
```

#### 场景二：质押全部余额

```typescript
function MaxDepositButton({ lpBalance, farm }) {
  const { write, isPending } = useMasterChefDeposit({
    chainId: farm.chainId,
    chef: farm.chefType,
    pid: farm.pid,
    amount: lpBalance,
    enabled: Boolean(lpBalance && lpBalance.gt(0)),
  })

  return (
    <button onClick={write} disabled={isPending}>
      质押全部
    </button>
  )
}
```

#### 场景三：组合 Approval 检查

```typescript
function DepositWithApproval({ farm, amount }) {
  const masterChefAddress = useMasterChefContractAddress(farm.chainId, farm.chefType)

  const { data: allowance } = useTokenAllowance({
    token: farm.lpToken,
    owner: account,
    spender: masterChefAddress,
  })

  const needsApproval = !allowance || allowance.lt(amount)

  const { write: approve } = useTokenApproval({
    spender: masterChefAddress,
    amount,
  })

  const { write: deposit, isPending } = useMasterChefDeposit({
    chainId: farm.chainId,
    chef: farm.chefType,
    pid: farm.pid,
    amount,
    enabled: Boolean(!needsApproval && amount),
  })

  if (needsApproval) {
    return <button onClick={approve}>先授权</button>
  }

  return (
    <button onClick={deposit} disabled={isPending}>
      质押
    </button>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **先检查并执行 Approval**
   ```typescript
   // LP 代币需要先 approve 给 MasterChef
   const { write: approve } = useTokenApproval({ ... })
   ```

2. **检查 LP 余额**
   ```typescript
   const { data: lpBalance } = useBalanceWeb3({
     currency: farm.lpToken,
     account,
   })

   enabled: Boolean(amount && amount.gt(0) && amount.lte(lpBalance))
   ```

3. **使用正确的 ChefType**
   ```typescript
   // 根据链选择正确的版本
   chef: chainId === EvmChainId.ETHEREUM
     ? ChefType.MasterChefV2
     : ChefType.MiniChef
   ```

#### ❌ Don'ts

1. **不要在没有 Approval 的情况下存款**
   ```typescript
   // 错误：会失败
   const { write } = useMasterChefDeposit({ amount })

   // 正确：先检查 allowance
   if (needsApproval) return <ApproveButton />
   ```

2. **不要质押超过余额的数量**
   ```typescript
   // 错误
   amount: lpBalance.add(1n)

   // 正确
   amount: lpBalance
   ```

3. **不要忘记传入正确的 PID**
   ```typescript
   // PID 必须与池子对应
   // 可以从 farm 配置中获取
   pid: farm.pid
   ```

4. **不要在组件卸载后调用 write**
   ```typescript
   // 错误
   useEffect(() => {
     if (ready) write()
   }, [])

   // 正确：用户点击按钮触发
   return <button onClick={write}>质押</button>
   ```
