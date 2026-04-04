> 源代码路径: `apps/web/src/lib/wagmi/hooks/approvals/hooks/useTokenApproval.ts`

# useTokenApproval

## 大白话讲讲这个hook的作用

`useTokenApproval` 是一个用于管理 ERC20 代币授权状态的 Hook。它的核心作用是：检查当前授权额度是否足够，如果不够就发起授权交易。

大白话就是：这个 Hook 帮你处理"我授权 Uniswap 可以花我的 USDC"这件事。它会先检查"之前授权了多少，还够不够用"，如果不够就"帮用户发起授权请求"。

整个授权流程都被封装好了，包括：授权状态判断、模拟调用、交易发送、错误处理、Toast 通知等。

## 讲讲为什么需要封装该hook

1. **授权状态复杂**：授权状态有多种（LOADING、UNKNOWN、NOT_APPROVED、PENDING、APPROVED），直接用 wagmi 实现容易遗漏边界情况。

2. **兼容旧版合约**：有些代币的 approve 函数签名不同，需要 fallback 机制兼容。

3. **模拟调用**：在发送真实交易前先模拟调用，确保交易能成功，避免用户付 gas 却失败。

4. **用户体验**：封装了 Toast 通知、pending 状态、错误处理等细节。

5. **与 allowance 联动**：需要结合 `useTokenAllowance` 查询当前授权额度。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseTokenApprovalParams {
  spender: EvmAddress | undefined     // 授权给谁（如合约地址）
  amount: Amount<EvmCurrency> | undefined  // 要花费的金额
  approveMax?: boolean                // 是否授权最大值（maxUint256）
  enabled?: boolean                   // 是否启用
}
```

### 输出 (Outputs)

```typescript
// 返回 [ApprovalState, { write: () => void }]
type ApprovalState =
  | 'LOADING'      // 正在检查授权额度
  | 'UNKNOWN'      // 未知状态
  | 'NOT_APPROVED' // 未授权或授权额度不足
  | 'PENDING'      // 交易等待中
  | 'APPROVED'     // 已授权且额度足够

// 使用示例
const [approvalState, { write }] = useTokenApproval({...})
if (approvalState === ApprovalState.NOT_APPROVED) {
  write?.()
}
```

### 执行逻辑详解

#### 1. 查询当前授权额度

```typescript
const { data: allowance } = useTokenAllowance({
  token: amount?.currency?.wrap(),
  owner: address,
  spender,
  chainId: amount?.currency.chainId,
})
```

#### 2. 判断授权状态

```typescript
let state = ApprovalState.UNKNOWN
if (amount?.currency.type === 'native') state = ApprovalState.APPROVED
else if (allowance && amount && allowance.gt(amount)) state = ApprovalState.APPROVED
else if (allowance && amount && allowance.eq(amount)) state = ApprovalState.APPROVED
else if (pending) state = ApprovalState.PENDING
else if (isAllowanceLoading) state = ApprovalState.LOADING
else if (allowance && amount && allowance.lt(amount)) state = ApprovalState.NOT_APPROVED
```

#### 3. 模拟调用（Simulation）

使用 `useSimulateContract` 在发送交易前验证：

```typescript
const standardSimulation = useSimulateContract({
  abi: erc20Abi_approve,
  functionName: 'approve',
  args: [spender, approveMax ? maxUint256 : amount.amount],
  query: { enabled: simulationEnabled && !fallback }
})
```

#### 4. Fallback 机制

某些旧代币合约的 approve 函数返回值不同，需要切换到旧版 ABI：

```typescript
if (error instanceof ContractFunctionZeroDataError) {
  setFallback(true)  // 切换到 old_erc20Abi_approve
}
```

#### 5. 发送交易

```typescript
const { mutate } = useWriteContract({
  mutation: { onSuccess, onError }
})

const write = () => mutate(simulation.request)
```

### 数据流向图

```
输入参数 (spender, amount, approveMax)
         ↓
    ┌────────────────────────────────────┐
    │  useTokenAllowance 查询当前授权额度  │
    └────────────────────────────────────┘
         ↓
    判断 ApprovalState
    ├── APPROVED → 直接可用，无需操作
    ├── NOT_APPROVED → 需要调用 write()
    ├── PENDING → 交易进行中
    └── LOADING → 正在加载
         ↓
    write() 被调用
         ↓
    ┌────────────────────────────────────┐
    │  useSimulateContract 模拟调用        │
    │  (检测是否需要 fallback)            │
    └────────────────────────────────────┘
         ↓
    ┌────────────────────────────────────┐
    │  useWriteContract 发送交易          │
    └────────────────────────────────────┘
         ↓
    onSuccess: Toast通知 + refetch allowance
    onError: 错误处理 + Toast通知
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：approveMax选项要看场景用**。如果你做的是DEX交易，每次都要授权不同的金额，那建议用approveMax等于true，直接给个最大值。这样用户就不用每次交易都要重新授权了。但要注意，这样做有安全风险，授权额度太大会让用户担心。所以如果是信任度不那么高的合约，最好还是传具体的授权金额。

**第二点：状态判断有顺序**。判断当前是什么授权状态的时候，一定要按顺序来。首先要判断是不是原生币，因为原生币根本不需要授权，直接就是APPROVED状态。然后才判断授权额度够不够。如果顺序搞反了，可能会对原生币做一些没必要的查询。

**第三点：Pending状态要处理好**。用户点击授权按钮后，交易提交到链上需要时间。这个等待的过程要给用户明显的反馈，比如按钮变灰显示"授权中..."。而且还要防止用户在交易还没确认的时候又点了一次，导致重复授权。

**第四点：成功后要刷新状态**。授权交易成功后，链上的授权额度已经变了，但本地的数据可能还是旧的。所以需要在成功的回调里主动调用refetch去重新查询授权额度，这样UI才能及时更新。

### 约束条件要记住

**第一，amount必须是Amount类型**。不能直接传一个bigint数字进去，否则类型不匹配可能会出问题。Amount类型是sushi库提供的，它会把数字和代币信息打包在一起，方便后续处理。

**第二，spender地址要验证**。传给hook的合约地址必须是用isAddress验证过的有效以太坊地址。如果传个空值或者格式不对，整个授权流程都会失败。

**第三，原生币直接过**。如果你传入的token是ETH这种原生币，hook会直接返回APPROVED状态，根本不会去查询链上数据。这是因为原生币不存在授权这个概念。

**第四，approveMax的用法**。当你设置为true的时候，实际授权的金额是maxUint256，这是一个接近2的256次方的巨大数字。理论上这个数字大到不可能被花完，所以用户以后就不用再授权了。

### 状态转换要清晰

这个hook使用了一个状态机来管理授权流程。初始状态是UNKNOWN，表示参数还没准备好或者正在检查。然后根据不同的情况切换到不同状态：如果正在查询授权额度就是LOADING状态，额度不够就变成NOT_APPROVED，额度够或者原生币就直接是APPROVED。用户调用write方法后会进入PENDING状态，等交易确认后根据结果变成APPROVED或者NOT_APPROVED。

状态之间的转换是单向的：UNKNOWN可以变成LOADING，LOADING根据额度情况变成APPROVED或NOT_APPROVED，NOT_APPROVED用户点击授权后变成PENDING，PENDING最后变成APPROVED或NOT_APPROVED。

### 常见错误要避开

**第一个坑：金额为0的情况**。如果用户要授权的金额是0，不应该真的去发起授权交易。这种情况下应该直接返回APPROVED，因为0额度本来就不需要授权。

**第二个坑：判断条件的边界**。判断授权额度是否足够的时候，应该用gte（大于等于）而不是gt（大于），因为等于的情况也算足够。比如用户之前授权了100，现在要花100，这个授权额度是够用的。

**第三个坑：fallback状态不会自动重置**。有些旧代币的approve函数返回值不同，需要切换到旧版ABI。这个fallback状态是通过useState管理的，一旦切换成true就不会自动变回false。所以下次再用这个hook的时候，如果参数变了但状态还是fallback，可能会出问题。

**第四个坑：并发重复提交**。用户可能在授权交易进行中又点了一次，这样会同时发起多笔授权交易。解决方案是在UI上做好防护，比如按钮禁用，或者在代码里加锁。

### 提示词模板

```markdown
帮我创建一个管理ERC20代币授权的hook。

需求：
- 查询当前授权额度够不够用
- 不够的话帮用户发起授权交易
- 交易完成后显示结果提示

技术栈：
- React + TypeScript
- wagmi v2 + viem
- @tanstack/react-query v5
- sushi库的Amount类型
- Toast通知

输入：
- spender：授权给哪个合约地址
- amount：要花费的金额
- approveMax：是否授权最大值
- enabled：是否启用

输出：
- ApprovalState：当前授权状态
- write：发起授权的函数

状态流程：
1. 原生币直接APPROVED
2. 额度足够也是APPROVED
3. 额度不够是NOT_APPROVED
4. 交易提交中是PENDING
5. 交易确认后根据结果变APPROVED或NOT_APPROVED

实现要点：
1. 先查当前授权额度
2. 判断是否需要授权
3. 发起授权交易前先模拟调用验证
4. 交易结果要Toast通知
5. 成功后要刷新授权额度数据

注意：
- amount为0时不发起交易
- 用户取消授权不要显示错误
- approveMax会授权一个巨大的数字
- 旧代币合约可能需要兼容处理
```

### 实际避坑指南

第一个，金额为0的问题。有时候业务逻辑会传入0作为授权金额，这种时候其实不需要真的调用合约，直接返回APPROVED就行了。否则会浪费用户的gas。

第二个，allowance判断用gte。我们在判断授权额度是否足够的时候，一定要用allowance.gte(amount)而不是allowance.gt(amount)。因为刚好等于也是足够的，用户就是想花这个数量的币。

第三个，fallback状态的问题。有些老版本的代币合约，approve函数的返回值格式不一样，需要切换到旧版ABI。这个fallback状态一旦变成true就不会自动恢复，所以如果参数变了要手动重置。

第四个，并发问题。用户可能在交易pending的时候又点了一次，这样会发起多笔交易。可以在UI上禁用按钮，也可以在代码里防止重复提交。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useTokenApproval` 用于管理 ERC20 代币授权。基本用法如下：

```typescript
import { useTokenApproval, ApprovalState } from '@sushiswap/wag'

function SwapButton({ amount, currency, spender }) {
  const [approvalState, { write }] = useTokenApproval({
    spender,
    amount,
    enabled: Boolean(amount && spender),
  })

  if (approvalState === ApprovalState.NOT_APPROVED) {
    return <button onClick={write}>授权 {currency.symbol}</button>
  }

  if (approvalState === ApprovalState.PENDING) {
    return <button disabled>授权中...</button>
  }

  if (approvalState === ApprovalState.APPROVED) {
    return <button>交换</button>
  }

  return <button disabled>加载中...</button>
}
```

### 5.2 常见使用场景

#### 场景一：DEX 交易授权

在 SushiSwap 进行交易前授权：

```typescript
const amount = new Amount(USDC, '1000000') // 1 USDC (假设 6 decimals)

const [approvalState, { write }] = useTokenApproval({
  spender: SUSHISWAP_V2_ROUTER_ADDRESS[chainId],
  amount,
  approveMax: false, // 只授权需要的金额
})
```

#### 场景二：授权最大值（适合 DeFi 理财）

```typescript
// 授权 maxUint256，后续无需再次授权
const [approvalState, { write }] = useTokenApproval({
  spender: someDefiProtocol,
  amount,
  approveMax: true, // 授权全部额度
})
```

#### 场景三：组合 useTokenAllowance 使用

```typescript
const { data: allowance } = useTokenAllowance({
  token,
  owner: account,
  spender,
})

const [approvalState, { write }] = useTokenApproval({
  spender,
  amount: amountToSpend,
})

// 手动检查是否需要授权
const needsApproval = !allowance || allowance.lt(amountToSpend)
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **先检查余额**
   ```typescript
   // 确保用户有足够的代币
   if (balance.lt(amount)) {
     return <div>余额不足</div>
   }
   ```

2. **显示清晰的 UI 状态**
   ```typescript
   // 好的 UX
   switch (approvalState) {
     case ApprovalState.LOADING:
       return <Spinner>检查授权状态...</Spinner>
     case ApprovalState.NOT_APPROVED:
       return <Button onClick={write}>授权 ${symbol}</Button>
     case ApprovalState.PENDING:
       return <Button loading>授权中...</Button>
     case ApprovalState.APPROVED:
       return <Button>交易</Button>
   }
   ```

3. **正确处理原生币**
   ```typescript
   // 原生币（如 ETH）不需要授权
   if (currency.type === 'native') {
     return <Button>交易</Button>
   }
   ```

#### ❌ Don'ts

1. **不要在 render 中直接调用 write**
   ```typescript
   // 错误：render 期间不应该有副作用
   if (approvalState === NOT_APPROVED) {
     write() // 不要这样做！
   }

   // 正确：用户点击按钮触发
   return <Button onClick={write}>授权</Button>
   ```

2. **不要忽略用户拒绝**
   ```typescript
   // 错误
   onError: (e) => createErrorToast(e.message)

   // 正确：用户拒绝不算错误
   onError: (e) => {
     if (isUserRejectedError(e)) return
     createErrorToast(e.message)
   }
   ```

3. **不要忘记 enabled 检查**
   ```typescript
   // 错误
   enabled: true

   // 正确：确保必要参数都存在
   enabled: Boolean(amount && spender && amount.amount > 0n)
   ```

4. **approveMax 有安全风险**
   ```typescript
   // 警告：授权全部额度有风险
   approveMax: true // 只在必要时使用，并确保合约可信
   ```
