> 源代码路径: `apps/web/src/lib/wagmi/hooks/bar/useBarWithdraw.ts`

# useBarWithdraw

## 大白话讲讲这个hook的作用

`useBarWithdraw` 是一个用于从 SushiSwap Bar（XSUSHI 质押池）提取 SUSHI 代币的 Hook。

大白话：就是"把 XSUSHI 换回 SUSHI"。你烧掉你的 xsUSHI，按 1:1 比例换回 SUSHI。

它的核心操作是调用 XSUSHI 合约的 `leave()` 函数，销毁指定数量的 XSUSHI 并返还等量的 SUSHI。

## 讲讲为什么需要封装该hook

1. **合约交互封装**：封装了 `leave()` 函数的调用细节。

2. **模拟验证**：使用 `useSimulateContract` 预验证交易。

3. **用户体验**：封装了 Toast 通知、Pending 状态、错误处理。

4. **余额刷新**：成功后自动刷新链上余额。

5. **静默拒绝**：用户拒绝签名时不显示无意义的错误提示。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseBarWithdrawParams {
  amount?: Amount<EvmToken>    // 提取数量（XSUSHI）
  enabled?: boolean
}
```

### 输出 (Outputs)

```typescript
{
  write: () => Promise<void>,   // 提取函数
  isPending: boolean,
  isError: boolean,
}
```

### 执行逻辑详解

#### 1. 模拟调用验证

```typescript
const { data: simulation } = useSimulateContract({
  address: XSUSHI_ADDRESS[EvmChainId.ETHEREUM],
  abi: xsushiAbi_leave,
  functionName: 'leave',
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
  if (!writeContractAsync || !simulation) return

  return async () => {
    try {
      await writeContractAsync(simulation.request)
    } catch {}
}, [writeContractAsync, simulation])
```

#### 3. 成功回调

```typescript
const onSuccess = (data) => {
  if (!amount) return

  const receipt = client.waitForTransactionReceipt({ hash: data })
  receipt.then(() => {
    refetchBalances(EvmChainId.ETHEREUM)
  })

  createToast({
    account: address,
    type: 'leaveBar',
    chainId: EvmChainId.ETHEREUM,
    txHash: data,
    promise: receipt,
    summary: {
      pending: `Unstaking ${amount.toSignificant(6)} XSUSHI`,
      completed: 'Successfully unstaked XSUSHI',
      failed: 'Something went wrong when unstaking XSUSHI',
    },
  })
}
```

#### 4. 错误处理

```typescript
const onError = (e: Error) => {
  if (isUserRejectedError(e)) return
  logger.error(e, { location: 'useBarWithdraw', action: 'mutationError' })
  createErrorToast(e?.message, true)
}
```

### 数据流向图

```
输入: amount (XSUSHI 数量)
         ↓
    useSimulateContract 模拟调用
    contract.leave(amount)
         ↓
    ┌────────────────────────────────────┐
    │  useWriteContract 发送交易          │
    └────────────────────────────────────┘
         ↓
    onSuccess:
    1. waitForTransactionReceipt
    2. refetchBalances
    3. createToast
         ↓
    返回 { write, isPending, isError }
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：余额检查很重要**。在用户点击提取之前，最好先检查一下他持有的XSUSHI是否足够。如果用户的XSUSHI余额为0或者小于要提取的数量，提取交易肯定会失败。可以在UI上做个判断，余额不足时禁用提取按钮。

**第二点：只能在以太坊主网操作**。这个hook同样是硬编码为以太坊主网的。SushiSwap Bar合约只在以太坊主网上有，测试网或者其他链上都没有。这个约束要记住，不能在别的链上使用这个功能。

**第三点：金额显示要格式化**。在Toast提示或者UI上显示提取金额的时候，要用toSignificant(6)这样的方法。这样可以保证数字既易读又不会太长。用户体验会好很多。

**第四点：用户取消要静默处理**。如果用户看到签名弹窗后点取消了，不要显示任何错误提示。这不算错误，是用户的正常操作。代码里用isUserRejectedError来判断，只有真正的错误才需要显示给用户。

**第五点：销毁是不可逆的**。当你调用leave函数提取XSUSHI时，这些XSUSHI会被永久销毁换成SUSHI。这个操作是不可逆的，用户一旦提取就拿不回原来的XSUSHI了。最好在UI上给用户一个确认提示。

### 约束条件要记住

**第一，只在以太坊主网操作**。跟存款一样，提取也只能在以太坊主网上进行。这是Bar合约部署的位置决定的。

**第二，只能XSUSHI换SUSHI**。这个操作的方向是确定的：输入XSUSHI，输出SUSHI。不能反向操作，也不能换其他代币。

**第三，销毁无法恢复**。XSUSHI被销毁后是找不回来的。所以用户在提取前要确认清楚，确实想要换回SUSHI。

**第四，需要持有足够的XSUSHI**。这是显而易见的，但如果余额不足就发起交易，会导致交易失败，浪费gas。

### 状态管理要清楚

交易状态主要看三个字段。write是执行提取的函数。isPending表示交易是否在pending中。isError表示是否有错误。成功后会自动触发余额刷新，通过receipt.then()监听交易确认，确认后才刷新数据。

### 常见错误要避开

**第一个坑：销毁不可逆**。leave函数会永久销毁XSUSHI。一旦执行，即使发现操作有误，也无法回滚了。这不像有些操作有宽限期或者可以取消。所以最好在执行前让用户二次确认。

**第二个坑：没有手续费但有gas**。提取操作本身不收任何费用，只有正常的gas消耗。但如果合约逻辑复杂，gas可能比预期高一些。

**第三个坑：奖励发放时间**。SushiSwap Bar的SUSHI奖励发放可能跟某些时区或者区块时间有关。如果遇到奖励没有及时到账的情况，可能是这个原因。

**第四个坑：大批量提取**。如果你需要提取非常大量的XSUSHI，可能需要分批操作。这取决于合约的具体限制，代码里没有处理这种情况。

**第五个坑：可能有冷却期**。某些版本的Bar合约可能有提取冷却期，提取后要过一段时间才能再次提取。这个要查阅具体合约的逻辑。

### 提示词模板

```markdown
帮我创建一个从SushiSwap Bar提取SUSHI的hook。

需求：
- 把XSUSHI换回SUSHI
- 提取后XSUSHI会被销毁
- 交易确认后要刷新余额

技术栈：
- React + TypeScript
- wagmi v2 + viem
- sushi库
- Toast通知
- 刷新余额的工具

输入：
- amount：提取数量，Amount类型
- enabled：是否启用

输出：
- write：执行提取的函数
- isPending：是否在等待中
- isError：是否出错

实现要点：
1. 调用XSUSHI合约的leave函数
2. leave会销毁XSUSHI
3. 先模拟调用验证交易会不会成功
4. 交易确认后才刷新余额
5. 用户取消签名不要显示错误

常量：
- XSUSHI地址：以太坊主网上的合约地址

注意：
- 只能在以太坊主网使用
- XSUSHI会被销毁，无法恢复
- 1:1换回SUSHI
- 最好让用户确认后再执行
```

### 实际避坑指南

第一个，确认提示。由于销毁不可逆，UI上最好有一个确认步骤。比如弹窗显示"你确定要把XSUSHI换回SUSHI吗？这个操作无法撤销"。

第二个，余额检查。在发起提取交易前，先检查用户余额是否足够。如果余额不足，应该禁用按钮并提示用户。

第三个，先模拟再发送。代码里已经做了模拟调用验证，这可以避免很多交易失败的情况。不要跳过这一步直接发送。

第四个，等待确认。一定要等待交易确认后再刷新余额数据。如果不等确认就刷新，可能刷新到的是旧数据。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useBarWithdraw` 用于从 SushiSwap Bar 提取 SUSHI。基本用法如下：

```typescript
import { useBarWithdraw } from '@sushiswap/wag'

function WithdrawForm({ amount }) {
  const { write, isPending, isError } = useBarWithdraw({
    amount,
    enabled: Boolean(amount && amount.gt(0)),
  })

  return (
    <button onClick={write} disabled={isPending || !write}>
      {isPending ? '提取中...' : '提取 SUSHI'}
    </button>
  )
}
```

### 5.2 常见使用场景

#### 场景一：提取全部 XSUSHI

```typescript
function WithdrawAllButton({ xSushiBalance }) {
  const { write, isPending } = useBarWithdraw({
    amount: xSushiBalance, // 使用全部余额
    enabled: Boolean(xSushiBalance && xSushiBalance.gt(0)),
  })

  return (
    <button onClick={write} disabled={isPending}>
      提取全部 XSUSHI
    </button>
  )
}
```

#### 场景二：提取并显示比例

```typescript
function WithdrawForm({ amount, sushiValue }) {
  const { write, isPending } = useBarWithdraw({
    amount,
    enabled: Boolean(amount && amount.gt(0)),
  })

  return (
    <div>
      <div>将提取 {amount.toSignificant(6)} XSUSHI</div>
      <div>获得约 {sushiValue.toSignificant(6)} SUSHI</div>
      <button onClick={write} disabled={isPending}>
        确认提取
      </button>
    </div>
  )
}
```

#### 场景三：提取后重新质押（循环套娃）

```typescript
function HarvestXSushi({ xSushiBalance }) {
  // 提取全部 XSUSHI
  const { write: withdraw, isPending: withdrawing } = useBarWithdraw({
    amount: xSushiBalance,
  })

  // 或者只领取奖励（提取 0）
  const { write: harvest, isPending: harvesting } = useBarWithdraw({
    amount: 0n, // 提取 0 = 只领取奖励
  })

  return (
    <div>
      <button onClick={harvest} disabled={harvesting}>
        领取 SUSHI 奖励
      </button>
      <button onClick={withdraw} disabled={withdrawing}>
        提取全部
      </button>
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **检查 XSUSHI 余额**
   ```typescript
   const { data: xSushiBalance } = useBalanceWeb3({
     currency: XSUSHI,
     account,
   })

   enabled: Boolean(xSushiBalance && xSushiBalance.gt(0))
   ```

2. **提 0 只领取奖励**
   ```typescript
   // 如果只想领取奖励而不提取本金
   const { write } = useBarWithdraw({ amount: 0n })
   ```

3. **确认是不可逆操作**
   ```typescript
   // 销毁 XSUSHI 无法恢复，需要用户确认
   if (!confirm('确定提取？XSUSHI 将被销毁。')) return
   write()
   ```

#### ❌ Don'ts

1. **不要提取超过余额的数量**
   ```typescript
   // 错误
   amount: xSushiBalance.add(1n) // 会失败

   // 正确
   amount: xSushiBalance
   ```

2. **不要假设实时价格**
   ```typescript
   // XSUSHI 和 SUSHI 的比例可能不等于 1:1
   // 因为 XSUSHI 会累积质押奖励
   // 所以提取的 SUSHI 数量 >= 存入的 SUSHI 数量
   ```

3. **不要忽略 Pending 状态**
   ```typescript
   <button disabled={isPending}>提取中...</button>
   ```

4. **不要在交易确认前关闭弹窗**
   ```typescript
   // 错误：用户可能在交易还没确认时就关闭了弹窗
   onSuccess: (hash) => {
     closeModal()
     refetchBalances()
   }

   // 正确：等待交易确认后再关闭
   onSuccess: async (hash) => {
     await waitForTransactionReceipt(hash)
     refetchBalances()
     closeModal()
   }
   ```
