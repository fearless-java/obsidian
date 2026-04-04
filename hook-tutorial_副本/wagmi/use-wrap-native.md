> 源代码路径: `apps/web/src/lib/wagmi/hooks/wnative/useWrapNative.ts`

# useWrapNative

## 大白话讲讲这个hook的作用

`useWrapNative` 是一个用于将原生币（ETH、MATIC 等）包装成 WETH、WMATIC 等 Wrapped 代币的 Hook。

大白话：就是"把原生币换成它的 Wrapped 版本"。在以太坊上 ETH 和 WETH 可以 1:1 互换，有些场景（如 DEX 交易）需要 WETH 而不是 ETH。

它的核心操作是调用 WETH9 合约的 `deposit()` 函数，将原生币存入并铸造等量的 Wrapped 代币。

## 讲讲为什么需要封装该hook

1. **原生币交互**：直接与 WETH 合约交互需要处理 value 字段。

2. **模拟验证**：使用 `useSimulateContract` 确保交易会成功。

3. **用户体验**：封装 Toast 通知、Pending 状态。

4. **金额处理**：需要把 `Amount` 的 `amount` 和 `value` 正确传递给合约。

5. **事件追踪**：发送分析事件记录 wrap 操作。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseWrapNativeParams {
  amount: Amount<EvmCurrency> | undefined  // 包装数量
  enabled?: boolean
}
```

### 输出 (Outputs)

```typescript
{
  write: () => Promise<void>,   // 包装函数
  isPending: boolean,
  isError: boolean,
}
```

### 执行逻辑详解

#### 1. 模拟调用验证

```typescript
const { data: simulation } = useSimulateContract({
  chainId: amount?.currency.chainId,
  address: amount?.currency.wrap().address,  // WETH 地址
  abi: weth9Abi_deposit,
  functionName: 'deposit',
  value: amount?.amount,  // 发送的 ETH 数量
  query: {
    enabled: Boolean(enabled && amount && amount.currency.type === 'native'),
  },
})
```

#### 2. 发送交易

```typescript
const write = useMemo(() => {
  if (!simulation) return undefined

  return async () => {
    try {
      await writeContractAsync(simulation.request)
    } catch (error) {
      if (isUserRejectedError(error)) return
      logger.error(error, { location: 'useWrapNative', action: 'sendTransaction' })
    }
  }
}, [simulation, writeContractAsync])
```

#### 3. 成功回调

```typescript
const onSuccess = async (data) => {
  sendAnalyticsEvent(InterfaceEventName.WRAP_TOKEN_TXN_SUBMITTED, {
    chain_id: amount.currency.chainId,
    token_address: amount.currency.wrap().address,
    token_symbol: amount.currency.symbol,
  })

  const receiptPromise = client.waitForTransactionReceipt({ hash: data })

  createToast({
    account: address,
    type: 'swap',
    chainId: amount.currency.chainId,
    txHash: data,
    promise: receiptPromise,
    summary: {
      pending: `Wrapping ${amount.currency.symbol}`,
      completed: `Successfully wrapped ${amount.currency.symbol}`,
      failed: `Something went wrong wrapping ${amount.currency.symbol}`,
    },
  })

  await receiptPromise
}
```

### 数据流向图

```
输入: amount (原生币数量)
         ↓
    enabled 检查: amount && currency.type === 'native'
         ↓
    useSimulateContract 模拟
    contract.deposit(value: amount.amount)
         ↓
    ┌────────────────────────────────────┐
    │  useWriteContract 发送交易          │
    │  msg.value = amount.amount        │
    └────────────────────────────────────┘
         ↓
    onSuccess:
    1. sendAnalyticsEvent
    2. waitForTransactionReceipt
    3. createToast
         ↓
    返回 { write, isPending, isError }
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：只处理原生币**。这个hook只能处理ETH、MATIC这样的原生币。如果你传一个ERC20代币进去，比如USDC，enabled检查会确保不会发起查询。原生币和ERC20的交互方式完全不同。

**第二点：value字段是关键**。WETH9的deposit函数没有参数，但你不能像普通函数调用那样传参数。原生币的金额要通过msg.value字段发送，这是以太坊的特殊机制，普通合约调用做不到。

**第三点：金额必须匹配**。你调用deposit时msg.value里放的金额，必须等于你想要wrap的数量。不是多一点点少一点的问题，必须完全相等。

**第四点：Toast反馈要清晰**。wrap操作涉及真实资产转换，需要给用户明确的反馈。pending状态显示"包装中..."，completed显示"成功包装"，failed显示"包装失败"。让用户随时知道进展。

**第五点：用户拒绝要静默**。如果用户看到钱包弹窗点拒绝了，不应该显示错误提示。这是用户的正常操作，取消不等于出错。

### 约束条件要记住

**第一，只处理原生币类型**。ETH、MATIC、BNB这些都是原生币。WETH、WMATIC这些已经是包装过的ERC20了，不是原生币。

**第二，只能wrap自己的余额**。你只能wrap自己钱包里的原生币。不能wrap别人的，也不能wrap比余额更多的数量。

**第三，合约要有足够WETH**。WETH9合约必须有足够的WETH余额才能完成mint。不过正常情况下合约余额都很充足，不用担心。

**第四，用户余额要够**。用户钱包里的原生币余额必须大于等于要wrap的数量。如果余额不足交易会失败。

### 状态管理要清楚

交易状态有三个字段。write是执行wrap的函数，调用它发起交易。isPending表示交易是否在pending中。isError表示是否有错误。

回调有两个。onSuccess在交易成功时触发，做Toast通知和数据分析。onError在交易失败时触发，记录错误日志。

### 常见错误要避开

**第一个坑：value字段而不是参数**。WETH9的deposit函数比较特殊，函数签名里没有参数。金额要通过msg.value发送，这是以太坊的特性，不是普通的合约调用方式。

**第二个坑：1:1没有手续费**。Wrap操作是1:1兑换的。1个ETH换1个WETH，不收任何手续费。只有正常的gas费用。

**第三个坑：Unwrap是另一个操作**。Unwrap是反向操作，调用的是WETH9的withdraw函数。需要单独封装，不能和wrap混用。

**第四个坑：Wrap需要gas**。虽然Wrap不收手续费，但交易本身需要gas。而且因为是合约调用，gas消耗比普通转账略高。

**第五个坑：合约余额问题**。正常情况下WETH9合约余额充足。但如果遇到极端情况合约被抽干，mint可能会失败。不过这种概率极低。

### 提示词模板

```markdown
帮我创建一个将原生币包装为Wrapped代币的hook。

需求：
- 把原生币（如ETH）换成WETH
- 1:1兑换，不收手续费
- 交易结果要明确反馈

技术栈：
- React + TypeScript
- wagmi v2 + viem
- sushi库
- Toast通知
- 分析事件

关键点：
- deposit函数没有参数
- 金额通过msg.value发送
- 1:1兑换没有手续费

输入：
- amount：wrap的数量，必须是原生币类型
- enabled：是否启用

输出：
- write：执行wrap的函数
- isPending：是否等待中
- isError：是否出错

实现要点：
1. 检查是原生币类型
2. 调用WETH9的deposit函数
3. 通过msg.value发送金额
4. 模拟验证确保交易能成功
5. Toast通知结果

注意：
- 只处理原生币
- 金额通过msg.value发送
- 1:1兑换没有手续费
- 用户取消不显示错误
```

### 实际避坑指南

第一个，确认原生币类型。传进去的amount必须是原生币类型。如何判断？看currency.type是否等于'native'。

第二个，msg.value的发送方式。代码里通过value参数传入，这会在交易层面发送对应数量的原生币给合约。不是普通函数参数。

第三个，1:1没有额外费用。Wrap操作除了gas没有其他费用。用户兑换多少就得多少。

第四个，Unwrap要单独处理。Wrap和Unwrap是不同函数，Unwrap调用withdraw。代码里只封装了Wrap。

2. **1:1 互换**：Wrap 是 1:1 的，没有手续费。

3. **Unwrap 是反向**：unwrap 调用的是 `withdraw(uint256)`，需要单独封装。

4. **Gas 费用**：Wrap 操作需要支付 gas，但不需要 approve（原生币不需要）。

5. **合约余额**：WETH 合约必须有能力 mint，如果合约被损坏可能失败。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useWrapNative` 用于将原生币包装为 Wrapped 代币。基本用法如下：

```typescript
import { useWrapNative } from '@sushiswap/wag'
import { EvmNative } from '@sushiswap/currency'

function WrapForm({ amount }) {
  const { write, isPending, isError } = useWrapNative({
    amount,
    enabled: Boolean(amount && amount.currency.type === 'native'),
  })

  return (
    <button onClick={write} disabled={isPending || !write}>
      {isPending ? '包装中...' : `包装 ${amount?.currency.symbol}`}
    </button>
  )
}
```

### 5.2 常见使用场景

#### 场景一：Wrap 全部 ETH

```typescript
function WrapAllButton({ ethBalance }) {
  const { write, isPending } = useWrapNative({
    amount: ethBalance, // 使用全部 ETH 余额
    enabled: Boolean(ethBalance && ethBalance.gt(0)),
  })

  return (
    <button onClick={write} disabled={isPending}>
      包装全部 ETH → WETH
    </button>
  )
}
```

#### 场景二：DEX 交易前 Wrap

```typescript
function SwapInput({ tokenA, tokenB, amount }) {
  const isNativeA = tokenA.type === 'native'
  const isNativeB = tokenB.type === 'native'

  // 如果输入是原生币，可能需要先 Wrap
  if (isNativeA && !isWrapped) {
    const { write: wrap, isPending: wrapping } = useWrapNative({
      amount,
    })

    return (
      <div>
        <div>需要先将 {tokenA.symbol} 包装为 WETH</div>
        <button onClick={wrap} disabled={wrapping}>
          {wrapping ? '包装中...' : '包装并交易'}
        </button>
      </div>
    )
  }

  return <SwapButton />
}
```

#### 场景三：组合 Unwrap

```typescript
function UnwrapButton({ wethBalance }) {
  // useWrapNative 只处理 wrap
  // unwrap 需要使用 useUnwrapNative 或直接调用 WETH.withdraw

  const { write, isPending } = useUnwrapNative({
    amount: wethBalance,
    enabled: Boolean(wethBalance && wethBalance.gt(0)),
  })

  return (
    <button onClick={write} disabled={isPending}>
      解包 WETH → ETH
    </button>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **检查是原生币类型**
   ```typescript
   // 确保 amount 是原生币类型
   amount.currency.type === 'native'

   // ETH、MATIC 等都是 native 类型
   // Wrapped 版本（如 WETH）是 ERC20 类型
   ```

2. **使用 value 字段**
   ```typescript
   // WETH deposit 通过 msg.value 发送，不是参数
   // useWrapNative 已经处理了这个
   ```

3. **1:1 互换**
   ```typescript
   // Wrap 是 1:1 的
   // 1 ETH → 1 WETH
   // 1 WETH → 1 ETH (通过 unwrap)
   ```

#### ❌ Don'ts

1. **不要 Wrap 非原生币**
   ```typescript
   // 错误
   const amount = new Amount(USDC, '1000000') // USDC 不是原生币
   useWrapNative({ amount }) // 会导致错误

   // 正确
   const amount = new Amount(EvmNative.ETHEREUM, '1000000') // ETH 是原生币
   ```

2. **不要 Wrap 超过余额的数量**
   ```typescript
   // 错误
   amount: ethBalance.add(1n) // 会失败

   // 正确
   amount: ethBalance
   ```

3. **不要忘记检查 enabled**
   ```typescript
   // 错误
   enabled: true

   // 正确
   enabled: Boolean(amount && amount.currency.type === 'native')
   ```

4. **不要假设交易立即完成**
   ```typescript
   // 错误
   write()
   checkWethBalance() // WETH 余额可能还没更新

   // 正确：等待 refetch 或使用 onSuccess 回调
   ```
