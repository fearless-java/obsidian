> 源代码路径: `apps/web/src/lib/wagmi/hooks/master-chef/use-master-chef-withdraw.ts`

# useMasterChefWithdraw

## 大白话讲讲这个hook的作用

`useMasterChefWithdraw` 是一个用于从 MasterChef 池子提取（取款）的 Hook。

大白话：就是"把存进去的 LP 代币拿回来，同时领取奖励"。你调用 withdraw，合约会把你质押的 LP 退给你，同时把累积的 SUSHI 奖励也发给你。

它的核心操作是调用 MasterChef 合约的 `withdraw()` 或 `withdrawAndHarvest()` 函数。

## 讲讲为什么需要封装该hook

1. **多版本支持**：V1 用 withdraw，V2/MiniChef 用 withdrawAndHarvest（提取+收割一步完成）。

2. **模拟验证**：使用 `useSimulateContract` 预验证。

3. **用户体验**：封装 Toast 通知、Pending 状态。

4. **奖励自动领取**：V2/MiniChef 的 withdraw 会自动带上 harvest，无需单独操作。

5. **确认回调**：支持传入 confirm 回调用于 UI 更新。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseMasterChefWithdrawParams {
  chainId: EvmChainId
  chef: ChefType
  pid: number
  amount?: Amount<Token>    // 提取数量（不是全部质押量）
  enabled?: boolean
}
```

### 输出 (Outputs)

```typescript
{
  write: (confirm?: () => void) => Promise<void>,  // 提取函数
  isPending: boolean,
  isError: boolean,
}
```

### 执行逻辑详解

#### 1. 构造 Withdraw 数据

```typescript
const prepare = useMemo(() => {
  if (!address || !chainId || !amount || !chef) return {}

  let data
  switch (chef) {
    case ChefType.MasterChefV1:
      data = {
        abi: masterChefV1Abi_withdraw,
        functionName: 'withdraw',
        args: [BigInt(pid), BigInt(amount.amount.toString())],
      }
      break
    case ChefType.MasterChefV2:
      data = {
        abi: masterChefV2Abi_withdraw,
        functionName: 'withdraw',
        args: [BigInt(pid), BigInt(amount.amount.toString()), address],
      }
      break
    case ChefType.MiniChef:
      data = {
        abi: miniChefV2Abi_withdrawAndHarvest,
        functionName: 'withdrawAndHarvest',
        args: [BigInt(pid), BigInt(amount.amount.toString()), address],
      }
  }

  return {
    account: address,
    address: getMasterChefContractConfig(chainId, chef)?.address,
    ...data,
  }
}, [address, amount, chainId, chef, pid])
```

#### 2. 模拟 + 发送

```typescript
const { data: simulation } = useSimulateContract({ ...prepare, chainId })

const write = useMemo(() => {
  if (!simulation?.request) return

  return async (confirm?: () => void) => {
    try {
      await writeContractAsync(simulation.request as any)
      confirm?.()  // 成功后调用确认回调
    } catch {}
  }
}, [simulation?.request, writeContractAsync])
```

#### 3. 成功回调

```typescript
const onSuccess = (data) => {
  createToast({
    account: address,
    type: 'burn',
    chainId,
    summary: {
      pending: `Unstaking ${amount.toSignificant(6)} ${amount.currency.symbol} tokens`,
      completed: `Successfully unstaked ${amount.toSignificant(6)} ${amount.currency.symbol} tokens`,
    },
  })
}
```

### 数据流向图

```
输入: chainId, chef, pid, amount
         ↓
    getMasterChefContractConfig 获取地址
         ↓
    prepare: 构造 withdraw 数据
    ┌────────────────────────────────────┐
    │  V1: withdraw(pid, amount)        │
    │  V2: withdraw(pid, amount, user)  │
    │  MiniChef: withdrawAndHarvest    │
    └────────────────────────────────────┘
         ↓
    useSimulateContract 模拟
         ↓
    useWriteContract 发送
         ↓
    onSuccess: Toast
         ↓
    返回 { write(confirm?), isPending }
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：提取全部就是传全部质押量**。想把存进去的LP全部拿回来？很简单，把用户的全部质押量作为amount传进去就行。合约会解锁全部的LP并返还给你。

**第二点：V1不会自动领奖励**。这是V1版本的一个特点。V1的withdraw只处理本金，奖励不会自动打给你。如果你有V1池子的质押，还需要单独调用一次harvest才能拿到奖励。

**第三点：MiniChef一步到位**。MiniChef的withdrawAndHarvest函数是提取和领取奖励一起的。一步操作同时拿到LP和SUSHI，更省gas也更方便。

**第四点：confirm回调很实用**。write函数接受一个可选的confirm回调参数。这个回调会在交易成功后执行，常用于关闭弹窗等UI操作。让交易确认后再关窗，用户体验更好。

**第五点：Toast类型用burn**。提取操作在Toast里应该标识为burn类型，这样图标和样式会和存入操作区分开，让用户一眼就看出这是在做提取操作。

### 约束条件要记住

**第一，提取量不能超过质押量**。这个显而易见，超过会失败。但要注意，如果有其他人在你操作的同时也在提取或存款，状态可能变化，导致你的提取失败。

**第二，V1需要单独harvest**。V1的withdraw只拿回本金，奖励要单独harvest。这比V2和MiniChef麻烦一些。

**第三，V2/MiniChef自动领奖励**。调用withdraw或withdrawAndHarvest的时候，合约会自动把该用户的SUSHI奖励一起打过去。不用额外操作。

**第四，滑点可能导致失败**。如果池子被大量提取导致流动性枯竭，或者价格剧烈波动，提取可能失败。不过这种情况比较少见。

### 状态管理要清楚

交易状态有三个。write是一个接受可选回调的函数，交易成功后会执行回调。isPending表示交易是否在pending中。isError表示是否出错。

回调有两个层面。onSuccess是交易成功后的标准回调，可以做Toast通知等。confirm是可选的回调参数，专门用于关闭弹窗等UI操作。

### 常见错误要避开

**第一个坑：V1不自动harvest**。如果你用的是V1池子，withdraw只会拿回LP代币，SUSHI奖励还在合约里。需要再调用一次harvest才能拿到。

**第二个坑：提取0的特殊用法**。传amount=0有一个妙用：可以只harvest不提取本金。这样你就能只领奖励而不动本金。

**第三个坑：池子被掏空**。如果在你提取之前有大量提取操作，池子可能被抽干。这种情况下你的提取可能失败。不过一般很快会恢复。

**第四个坑：并发竞态**。如果有人在同一时间也在操作同一个池子，状态会变化。可能需要重试。

**第五个坑：withdrawAndHarvest更省gas**。相比分别调用withdraw和harvest，用withdrawAndHarvest一步完成更高效。因为少了一次合约调用，省不少gas。

### 提示词模板

```markdown
帮我创建一个从MasterChef池子提取的hook。

需求：
- 提取质押的LP代币
- 支持V1、V2、MiniChef三个版本
- V2/MiniChef自动领取奖励
- 支持提取后回调关闭弹窗

技术栈：
- React + TypeScript
- wagmi v2 + viem
- sushi库
- Toast通知

版本差异：
- V1：withdraw(pid, amount)，只提取不领奖励
- V2：withdraw(pid, amount, to)，只提取不领奖励
- MiniChef：withdrawAndHarvest(pid, amount, to)，提取+领奖励

输入：
- chainId：链ID
- chef：版本类型
- pid：池子ID
- amount：提取数量
- enabled：是否启用

输出：
- write(confirm?)：提取函数，可选回调
- isPending：是否等待中
- isError：是否出错

实现要点：
1. 根据版本选择正确的函数
2. V1需要单独harvest
3. MiniChef用withdrawAndHarvest
4. 成功后执行confirm回调
5. Toast用burn类型标识

注意：
- 提取量不能超过质押量
- V1要单独harvest
- 其他人的操作可能影响你的提取
- withdrawAndHarvest比分别调用更省gas
```

### 实际避坑指南

第一个，V1的harvest问题。如果你用V1池子，提取后别忘了再harvest一次把奖励拿出来。V2和MiniChef就不用这么麻烦。

第二个，提取0的用法。传0不是报错，而是只harvest的技巧。适合只想领奖励不想动本金的时候用。

第三个，确认回调的时机。confirm回调是在交易成功后才执行的，这样可以确保交易真正确认了再关弹窗。避免用户以为交易成功了结果其实还在pending。

第四个，gas优化。如果你要同时提取和harvest（针对V1），可以考虑用批量操作一次完成，减少交互次数。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useMasterChefWithdraw` 用于从 MasterChef 池子提取。基本用法如下：

```typescript
import { useMasterChefWithdraw, useMasterChef, ChefType } from '@sushiswap/wag'

function WithdrawForm({ farm, amount }) {
  const { write, isPending } = useMasterChefWithdraw({
    chainId: farm.chainId,
    chef: farm.chefType,
    pid: farm.pid,
    amount,
    enabled: Boolean(amount && amount.gt(0)),
  })

  return (
    <button onClick={write} disabled={isPending || !write}>
      {isPending ? '提取中...' : '提取 LP'}
    </button>
  )
}
```

### 5.2 常见使用场景

#### 场景一：提取全部质押

```typescript
function WithdrawAllButton({ farm }) {
  const { balance } = useMasterChef({
    chainId: farm.chainId,
    chef: farm.chefType,
    pid: farm.pid,
    token: farm.lpToken,
  })

  const { write, isPending } = useMasterChefWithdraw({
    chainId: farm.chainId,
    chef: farm.chefType,
    pid: farm.pid,
    amount: balance,
    enabled: Boolean(balance && balance.gt(0)),
  })

  return (
    <button onClick={write} disabled={isPending}>
      提取全部 ({balance?.toSignificant(4)} LP)
    </button>
  )
}
```

#### 场景二：只领取奖励（不提取本金）

```typescript
function HarvestOnlyButton({ farm }) {
  // V2/MiniChef 可以传入 amount = 0 来只领取奖励
  const { write, isPending } = useMasterChefWithdraw({
    chainId: farm.chainId,
    chef: farm.chefType,
    pid: farm.pid,
    amount: 0n, // 只领取，不提取本金
  })

  return (
    <button onClick={write} disabled={isPending}>
      收割奖励
    </button>
  )
}
```

#### 场景三：提取并关闭弹窗

```typescript
function WithdrawModal({ farm, amount, onClose }) {
  const { write, isPending } = useMasterChefWithdraw({
    chainId: farm.chainId,
    chef: farm.chefType,
    pid: farm.pid,
    amount,
  })

  const handleWithdraw = async () => {
    await write(() => {
      // 交易成功后关闭弹窗
      onClose()
    })
  }

  return (
    <Modal>
      <div>提取 {amount.toSignificant(6)} LP</div>
      <button onClick={handleWithdraw} disabled={isPending}>
        确认提取
      </button>
    </Modal>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **V1 需要单独 harvest**
   ```typescript
   // V1 withdraw 不自动领取奖励
   // 需要额外调用 harvest
   if (farm.chefType === ChefType.MasterChefV1) {
     const { harvest } = useMasterChef({ ... })
     // 先 withdraw，再 harvest
   }
   ```

2. **使用 confirm 回调关闭弹窗**
   ```typescript
   const handleWithdraw = () => {
     write(() => {
       onClose() // 成功后关闭
     })
   }
   ```

3. **提取 0 = 只收割**
   ```typescript
   // MiniChef/V2 可以只领取奖励
   amount: 0n
   ```

#### ❌ Don'ts

1. **不要提取超过质押总量**
   ```typescript
   // 错误
   amount: balance.add(1n)

   // 正确
   amount: balance
   ```

2. **V1 不要忘记 harvest**
   ```typescript
   // 错误：V1 只调用 withdraw 不领取奖励
   write()

   // 正确：V1 需要先 withdraw 再 harvest
   withdraw(() => harvest())
   ```

3. **不要在提取时关闭弹窗太快**
   ```typescript
   // 错误：交易还没确认就关闭
   write()
   onClose()

   // 正确：等待交易确认
   write(() => onClose())
   ```

4. **不要传入负数**
   ```typescript
   // 错误
   amount: -100n

   // 正确
   amount: 0n // 只收割
   amount: balance // 全部提取
   ```
