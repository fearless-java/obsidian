> 源代码路径: `apps/web/src/lib/wagmi/hooks/bar/useBarDeposit.ts`

# useBarDeposit

## 大白话讲讲这个hook的作用

`useBarDeposit` 是一个用于在 SushiSwap Bar（XSUSHI 质押池）存入 SUSHI 代币的 Hook。

大白话：就是"质押 SUSHI，换成 XSUSHI"。你把 SUSHI 存进去，1:1 换 xsUSHI，xsUSHI 可以赚取质押奖励，同时保留对 SUSHI 的敞口。

它的核心操作是调用 XSUSHI 合约的 `enter()` 函数，将 SUSHI 质押并铸造等量的 XSUSHI。

## 讲讲为什么需要封装该hook

1. **合约交互封装**：直接调用合约的 `enter` 函数需要处理 ABI、参数编码等，封装后只需传入 Amount。

2. **模拟验证**：使用 `useSimulateContract` 在发送前验证交易是否会成功。

3. **用户体验**：封装了 Toast 通知、Pending 状态、错误处理等。

4. **余额刷新**：交易成功后自动触发 `refetchBalances` 刷新钱包余额。

5. **静默错误处理**：用户拒绝签名时不显示错误 Toast。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseBarDepositParams {
  amount?: Amount<EvmToken>    // 存入数量（必须是 SUSHI）
  enabled?: boolean
}
```

### 输出 (Outputs)

```typescript
{
  write: () => Promise<void>,   // 存款函数
  isPending: boolean,
  isError: boolean,
  error: Error | null,
}
```

### 执行逻辑详解

#### 1. 模拟调用验证

```typescript
const { data: simulation } = useSimulateContract({
  address: XSUSHI_ADDRESS[EvmChainId.ETHEREUM],
  abi: xsushiAbi_enter,
  functionName: 'enter',
  chainId: EvmChainId.ETHEREUM,
  args: amount ? [amount.amount] : undefined,
  query: { enabled },
})
```

#### 2. 发送交易

```typescript
const { mutateAsync: writeContractAsync, ...rest } = useWriteContract({
  mutation: { onSuccess, onError },
})

const write = useMemo(() => {
  if (!writeContractAsync || !simulation) return undefined

  return async () => {
    try {
      await writeContractAsync(simulation.request)
    } catch {}  // 错误在 onError 中处理
  }
}, [writeContractAsync, simulation])
```

#### 3. 成功回调

```typescript
const onSuccess = (data) => {
  if (!amount) return

  const receipt = client.waitForTransactionReceipt({ hash: data })
  receipt.then(() => {
    refetchBalances(EvmChainId.ETHEREUM)  // 刷新余额
  })

  createToast({
    account: address,
    type: 'enterBar',
    chainId: EvmChainId.ETHEREUM,
    txHash: data,
    promise: receipt,
    summary: {
      pending: `Staking ${amount.toSignificant(6)} SUSHI`,
      completed: 'Successfully staked SUSHI',
      failed: 'Something went wrong when staking SUSHI',
    },
  })
}
```

#### 4. 错误处理

```typescript
const onError = (e: Error) => {
  if (isUserRejectedError(e)) return  // 用户拒绝不显示
  logger.error(e, { location: 'useBarDeposit', action: 'mutationError' })
  createErrorToast(e?.message, true)
}
```

### 数据流向图

```
输入: amount (SUSHI 数量)
         ↓
    useSimulateContract 模拟调用
    contract.enter(amount)
         ↓
    ┌────────────────────────────────────┐
    │  useWriteContract 发送交易          │
    └────────────────────────────────────┘
         ↓
    ┌────────────────────────────────────┐
    │  onSuccess:                        │
    │  1. waitForTransactionReceipt      │
    │  2. refetchBalances 刷新余额        │
    │  3. createToast 通知               │
    └────────────────────────────────────┘
         ↓
    返回 { write, isPending, isError }
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：金额要校验**。在调用存款函数之前，必须确保amount存在且大于0。如果amount是0或者undefined，就不应该发起交易，否则只会浪费用户的gas。好的做法是在调用write之前先检查一下，或者用enabled参数控制。

**第二点：只在以太坊主网操作**。这个hook是专门给SushiSwap Bar设计的，而Bar合约只部署在以太坊主网上。所以chainId被硬编码为以太坊主网的ID，不支持其他链。这是一个重要的约束，不要尝试在其他链上使用这个hook。

**第三点：用户拒绝要静默处理**。当用户弹窗拒绝签名的时候，不应该显示任何错误提示。因为这是用户的正常操作，不是什么错误。代码里用isUserRejectedError来判断是否是用户拒绝，如果是就直接返回，不做任何处理。

**第四点：金额格式化要友好**。在显示存款金额给用户看的时候，要用toSignificant(6)这样的方法来格式化。这样可以保留6位有效数字，既不会太长也不会丢失重要精度。

**第五点：Promise要正确处理**。waitForTransactionReceipt返回的是一个Promise，必须正确await它。如果不等待就直接返回，Toast可能显示"交易成功"但实际上交易还在pending。receipt.then()里的回调会在交易确认后执行，这时候才应该刷新余额数据。

### 约束条件要记住

**第一，只能在以太坊主网用**。Bar合约只在以太坊主网上有，所以这个hook只能在以太坊主网使用。如果你在其他链上调用，永远不会成功。

**第二，只能存SUSHI**。这个操作是专门给SUSHI代币设计的。你只能存入SUSHI，然后收到等量的XSUSHI。不能存入其他代币。

**第三，需要先授权**。在存款之前，必须先授权XSUSHI合约可以动用你的SUSHI代币。这是一个两步流程，先approve再deposit，不能跳过。

**第四，交易可能失败**。虽然模拟调用已经验证过了，但真实交易还是可能因为各种原因失败。比如合约被暂停，或者链上拥堵导致超时等。

### 状态管理要清楚

交易状态有三个关键字段。write是执行存款的函数，调用它会发起交易。isPending表示交易是否在等待确认中。isError表示是否有错误发生。error对象包含具体的错误信息。

刷新的触发是在交易成功后自动进行的。代码里用了receipt.then()来监听交易确认，确认后才调用refetchBalances刷新余额数据。这样可以确保余额数据是最新的。

### 常见错误要避开

**第一个坑：chainId是硬编码的**。代码里用 EvmChainId.ETHEREUM 硬编码了链ID，这意味着不管你传什么chainId进去，实际使用的都是以太坊主网。如果你在其他链上调用，比如Polygon，会发现交易永远发不出去。

**第二个坑：先approve再deposit**。存款操作之前必须先授权XSUSHI合约动用你的SUSHI。没有这个授权，deposit调用会失败。这两个步骤不能合并，必须分开执行。

**第三个坑：1:1的比例**。存入多少SUSHI就铸造多少XSUSHI，比例是固定1:1的。不会有手续费或者汇率波动。但这也意味着XSUSHI的价值完全跟着SUSHI走。

**第四个坑：receiptPromise的处理**。waitForTransactionReceipt必须await，否则代码会继续执行而不是等待交易确认。UI可能显示成功但实际上交易还没确认。receipt.then()里的回调是在交易确认后才执行的。

**第五个坑：empty catch的问题**。write函数里的catch是空的，看起来像bug但其实是故意的。因为真正的错误处理在onError回调里做。如果catch了，onError就不会收到了。

### 提示词模板

```markdown
帮我创建一个在SushiSwap Bar质押SUSHI的hook。

需求：
- 把SUSHI存入换取XSUSHI
- 1:1兑换，没有手续费
- 交易确认后要刷新余额

技术栈：
- React + TypeScript
- wagmi v2 + viem
- sushi库
- Toast通知
- 刷新余额的工具

输入：
- amount：存款数量，Amount类型
- enabled：是否启用

输出：
- write：执行存款的函数
- isPending：是否在等待中
- isError：是否出错

实现要点：
1. 调用XSUSHI合约的enter函数
2. 先模拟调用验证交易会不会成功
3. 交易确认后才刷新余额
4. 用户拒绝签名不要显示错误
5. 金额显示要格式化

常量：
- XSUSHI地址：以太坊主网上的合约地址

注意：
- 只能在以太坊主网使用
- 需要先授权才能存款
- 存款是1:1的
- 用户拒绝签名不算错误
```

### 实际避坑指南

第一个，检查chainId。虽然代码硬编码了以太坊主网，但你传入的参数最好也检查一下，防止误用。

第二个，approve和deposit的顺序。存款前必须先approve XSUSHI合约。这是一个独立的两步操作，用户需要签两次名。如果你在UI上把这两个合并了，要清楚背后的逻辑。

第三个，1:1没有手续费。存款的比例是固定的1:1，没有手续费或滑点。这点要跟用户说明清楚。

第四个，receiptPromise要await。等待交易确认是很重要的，不要假设交易发送出去就算成功了。一定要await waitForTransactionReceipt，等交易确认后再刷新数据。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useBarDeposit` 用于在 SushiSwap Bar 质押 SUSHI。基本用法如下：

```typescript
import { useBarDeposit } from '@sushiswap/wag'

function DepositForm({ amount }) {
  const { write, isPending, isError } = useBarDeposit({
    amount,
    enabled: Boolean(amount && amount.gt(0)),
  })

  return (
    <button onClick={write} disabled={isPending || !write}>
      {isPending ? '质押中...' : '质押 SUSHI'}
    </button>
  )
}
```

### 5.2 常见使用场景

#### 场景一：质押 SUSHI 获取 XSUSHI

```typescript
function StakeSUSHI({ amount }) {
  const { write, isPending } = useBarDeposit({ amount })

  return (
    <div>
      <div>将质押 {amount.toSignificant(6)} SUSHI</div>
      <div>获得等量 XSUSHI</div>
      <button onClick={write} disabled={isPending}>
        {isPending ? '处理中...' : '确认质押'}
      </button>
    </div>
  )
}
```

#### 场景二：组合 Balance 检查

```typescript
function MaxDepositButton({ balance }) {
  const { write, isPending } = useBarDeposit({
    amount: balance, // 使用全部余额
    enabled: Boolean(balance && balance.gt(0)),
  })

  return (
    <button onClick={write} disabled={isPending}>
      质押全部 (max)
    </button>
  )
}
```

#### 场景三：组合 Approval 检查

```typescript
function DepositWithApproval({ amount }) {
  const { data: allowance } = useTokenAllowance({
    token: SUSHI,
    owner: account,
    spender: XSUSHI_ADDRESS[EvmChainId.ETHEREUM],
  })

  const needsApproval = !allowance || allowance.lt(amount)

  const { write: approve } = useTokenApproval({
    spender: XSUSHI_ADDRESS[EvmChainId.ETHEREUM],
    amount,
  })

  const { write: deposit, isPending } = useBarDeposit({
    amount,
    enabled: Boolean(!needsApproval && amount),
  })

  if (needsApproval) {
    return <button onClick={approve}>先授权 XSUSHI</button>
  }

  return (
    <button onClick={deposit} disabled={isPending}>
      质押 SUSHI
    </button>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **先检查 Approval**
   ```typescript
   // SUSHI 需要先 approve 给 XSUSHI 合约
   const { write: approve } = useTokenApproval({ ... })
   ```

2. **检查余额**
   ```typescript
   const { data: sushibalance } = useBalanceWeb3({ currency: SUSHI, account })
   enabled: Boolean(balance && balance.gt(0))
   ```

3. **处理 Pending 状态**
   ```typescript
   <button disabled={isPending}>
     {isPending ? '质押中...' : '质押'}
   </button>
   ```

#### ❌ Don'ts

1. **不要在没有 Approval 的情况下直接存款**
   ```typescript
   // 错误
   const { write } = useBarDeposit({ amount })

   // 正确：先检查并执行 approval
   if (needsApproval) return <ApproveButton />
   ```

2. **不要传入 0 或负数**
   ```typescript
   // 错误
   amount: BigInt(0)

   // 正确
   enabled: Boolean(amount && amount.gt(0))
   ```

3. **不要假设交易立即成功**
   ```typescript
   // 错误：交易后立即检查余额
   await write()
   checkBalance() // 余额可能还没更新

   // 正确：等待 refetchBalances 触发
   ```

4. **不要忽略用户拒绝**
   ```typescript
   // 用户取消不应该显示错误
   // isUserRejectedError 会处理这个情况
   ```
