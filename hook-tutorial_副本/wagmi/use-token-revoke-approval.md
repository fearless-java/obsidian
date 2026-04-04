> 源代码路径: `apps/web/src/lib/wagmi/hooks/approvals/hooks/useTokenRevokeApproval.ts`

# useTokenRevokeApproval

## 大白话讲讲这个hook的作用

`useTokenRevokeApproval` 是一个用于撤销 ERC20 代币授权的 Hook。简单来说，它帮你把之前"授权给别人的钱"收回来。

大白话：当你不信任某个合约了，或者不再需要用它了，可以用这个 Hook 把授权撤销掉（设置授权额度为 0）。

它的核心逻辑是：调用 ERC20 的 `approve(spender, 0)` 来撤销授权。

## 讲讲为什么需要封装该hook

1. **安全性考虑**：用户授权给未知或不再使用的合约存在安全风险，需要提供便捷的撤销功能。

2. **与 Approval 统一**：授权和撤销是配套操作，统一封装便于管理和维护。

3. **Fallback 机制**：某些旧代币的 approve 行为不同，需要 fallback 兼容。

4. **用户体验**：封装 Toast 通知和错误处理，让撤销操作有明确的反馈。

5. **Pending 状态**：撤销操作也需要处理 pending 状态，防止重复提交。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseTokenRevokeApproval {
  account: Address | undefined        // 钱包地址
  spender: Address | undefined        // 要撤销的授权对象
  token: Omit<EvmToken, 'wrapped'> | undefined  // 代币
  enabled?: boolean
}
```

### 输出 (Outputs)

```typescript
{
  write: undefined | (() => Promise<void>)  // 撤销函数
  isPending: boolean                          // 是否等待中
  // ... 其他 writeContract 状态
}
```

### 执行逻辑详解

#### 1. 模拟调用验证

```typescript
const standardSimulation = useSimulateContract({
  address: token?.address,
  abi: erc20Abi_approve,
  functionName: 'approve',
  args: [spender, 0n],  // 撤销设为 0
  scopeKey: 'revoke-std',
  query: {
    enabled: simulationEnabled && !fallback,
    retry: (failureCount, error) => {
      // 检测是否需要 fallback
      if (error instanceof ContractFunctionZeroDataError) {
        setFallback(true)
        return false
      }
      return failureCount < 2
    }
  }
})
```

#### 2. Fallback 切换

```typescript
// 某些旧代币可能需要旧版 ABI
const fallbackSimulation = useSimulateContract({
  abi: old_erc20Abi_approve,
  // ...
})
```

#### 3. 发送撤销交易

```typescript
const write = async () => {
  if (!simulation?.request) return
  await writeContractAsync(simulation.request as any)
}
```

#### 4. 成功回调

```typescript
const onSuccess = async (data) => {
  const receiptPromise = client.waitForTransactionReceipt({ hash: data })

  createToast({
    account,
    type: 'swap',
    chainId: token.chainId,
    txHash: data,
    promise: receiptPromise,
    summary: {
      pending: `Revoking approval for ${token.symbol}`,
      completed: `Successfully revoked approval for ${token.symbol}`,
      failed: `Failed to revoke approval for ${token.symbol}`,
    }
  })

  await receiptPromise
}
```

### 数据流向图

```
输入参数 (account, spender, token)
         ↓
    simulationEnabled 检查
         ↓
    ┌────────────────────────────────────┐
    │  useSimulateContract 模拟调用       │
    │  args: [spender, 0n] 撤销授权      │
    └────────────────────────────────────┘
         ↓
    错误检测 → 是否需要 fallback
         ↓
    ┌────────────────────────────────────┐
    │  useWriteContract 发送交易         │
    │  approve(spender, 0)              │
    └────────────────────────────────────┘
         ↓
    onSuccess: Toast 通知
         ↓
    返回 { write, isPending }
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：撤销就是设成0**。撤销授权的核心就是调用approve函数，但把授权额度设成0。这就是为什么代码里写的是args: [spender, 0n]。0n就代表把授权额度清零，也就是撤销授权。

**第二点：撤销前要有确认**。这个操作是不可逆的，一旦撤销就真的撤销了。如果用户误操作把一个还在用的合约授权撤销了，可能会造成资产损失。所以最好在UI层面加一个确认弹窗，让用户二次确认。

**第三点：多签钱包要额外注意**。如果用户用的是多签钱包，那撤销授权可能需要多个签名者都确认才能执行。这种情况下，交易的处理时间会比普通钱包长很多，UI要给用户明确的提示。

**第四点：撤销后要刷新状态**。撤销操作完成后，链上的授权额度已经变成0了，但本地的数据可能还是旧的。最好在交易成功后主动调用refetch刷新一下授权额度数据，这样UI能及时更新。

### 约束条件要记住

**第一，地址要有效**。spender地址必须是经过验证的合法以太坊地址。如果传个空值或者无效地址，交易肯定会失败。

**第二，代币要合规**。token必须是有效的ERC20代币地址。如果传入的是一个原生币或者无效地址，查询和交易都会出问题。

**第三，撤销需要付gas**。虽然撤销授权不涉及任何代币转移，只是修改一个状态变量，但还是要付gas的。而且如果合约有特殊的逻辑，gas消耗可能会比预期的高。

**第四，授权额度为0也可以撤销**。有些时候授权额度本来就是0，但用户可能还是想调用一次撤销操作。这可能是为了重置某些状态，或者用户只是想确认一下当前的授权情况。这种情况是可以执行的，不会报错。

### 状态管理要清楚

状态流转很简单：用户调用write函数后，isPending变成true，表示交易正在等待确认。交易确认后isPending变回false。如果交易失败，错误会在onError回调里处理。

整个撤销流程是：调用write() -> isPending变为true -> 等待交易确认 -> 交易确认后isPending变为false -> 需要手动刷新授权额度数据。

### 常见错误要避开

**第一个坑：旧代币兼容问题**。有些老版本的代币合约，approve函数的返回值格式不一样。比如有些代币在成功时返回true，有些返回数据长度而不是具体值。如果遇到这种合约，代码里的fallback机制会自动切换到旧版ABI来处理。

**第二个坑：多签钱包的复杂性**。如果用户用的是多签钱包，那撤销操作需要多个签名者分别确认。这意味着交易提交的只是一个待执行的状态，不是立即生效的。UI要给用户说明这个情况。

**第三个坑：gas估算可能不准**。普通的撤销操作gas消耗一般比较低。但如果有合约有特殊的验证逻辑，或者撤销的是一个大额的授权，gas消耗可能会高很多。最好在交易发送前有个预估。

**第四个坑：事件监听更新UI**。撤销成功会触发Approval事件，合约内部会记录授权额度从大变小。如果你的应用监听这个事件，可以用来实时更新UI状态，不用等主动查询。

**第五个坑：批量撤销**。如果用户有很多个授权要撤销，需要逐个调用。如果有permit batch功能，可以一次撤销多个，会省很多gas。

### 提示词模板

```markdown
帮我创建一个撤销ERC20代币授权的hook。

需求：
- 把某个合约的授权额度设成0
- 撤销前最好有用户确认
- 交易结果要有提示

技术栈：
- React + TypeScript
- wagmi v2 + viem
- @tanstack/react-query v5
- sushi库
- Toast通知

输入：
- account：钱包地址
- spender：要撤销授权的合约地址
- token：要撤销授权的代币
- enabled：是否启用

输出：
- write：执行撤销的函数
- isPending：是否在等待中

实现要点：
1. 调用approve(spender, 0)来撤销
2. 先模拟调用验证交易会不会成功
3. 有些旧代币需要用不同的ABI处理
4. 撤销完成后要刷新授权额度数据
5. 操作不可逆，要让用户确认

注意：
- 撤销操作没有代币费用，只有gas费用
- 操作不可逆，确认要谨慎
- 有些代币撤销后可能要等一个区块才能重新授权
- 某些合约可能不支持撤销
```

### 实际避坑指南

第一个，旧代币的问题。有些老代币的approve函数返回值格式跟标准不一样，比如某些稳定币。如果代码里没有做兼容处理，遇到这种代币就会失败。解决方案是加一个fallback机制，遇到特殊返回值时切换到旧版ABI。

第二个，多签钱包执行撤销需要多个签名。如果你的应用支持多签钱包，那撤销操作可能需要等待一段时间才能完成。UI要给用户说明情况，不能假设交易立即生效。

第三个，事件监听更新UI。撤销成功后，链上会触发一个Approval事件。如果你的应用监听这个事件，可以不用主动查询就更新授权状态，这会让UI响应更快。

第四个，批量撤销的优化。如果用户有很多授权要撤销，可以考虑用permit2或者permit batch这样的功能，一次撤销多个，会比逐个撤销省很多gas。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useTokenRevokeApproval` 用于撤销 ERC20 代币授权。基本用法如下：

```typescript
import { useTokenRevokeApproval } from '@sushiswap/wag'

function RevokeButton({ token, spender, account }) {
  const { write, isPending } = useTokenRevokeApproval({
    account,
    spender,      // 要撤销授权的合约地址
    token,        // 要撤销授权的代币
    enabled: Boolean(account && spender && token),
  })

  return (
    <button onClick={write} disabled={isPending}>
      {isPending ? '撤销中...' : `撤销 ${token.symbol} 授权`}
    </button>
  )
}
```

### 5.2 常见使用场景

#### 场景一：授权管理界面

在用户设置页面展示已授权的合约，允许撤销：

```typescript
function ApprovedList({ approvals }) {
  return (
    <div>
      <h3>已授权的合约</h3>
      {approvals.map(({ token, spender, allowance }) => (
        <div key={`${token.address}-${spender}`}>
          <span>授权 {token.symbol} 给 {spender}</span>
          <span>额度: {allowance.toSignificant(6)}</span>
          <RevokeButton token={token} spender={spender} />
        </div>
      ))}
    </div>
  )
}
```

#### 场景二：撤销前确认

```typescript
function RevokeButton({ token, spender }) {
  const [showConfirm, setShowConfirm] = useState(false)

  const { write, isPending } = useTokenRevokeApproval({
    account,
    spender,
    token,
  })

  const handleRevoke = () => {
    // 撤销前二次确认
    if (confirm(`确定要撤销对 ${spender} 的 ${token.symbol} 授权吗？`)) {
      write()
    }
  }

  return (
    <>
      <button onClick={handleRevoke} disabled={isPending}>
        撤销授权
      </button>
    </>
  )
}
```

#### 场景三：撤销后刷新 Allowance

```typescript
function RevokeAndRefresh({ token, spender }) {
  const { write, isPending } = useTokenRevokeApproval({
    account,
    spender,
    token,
  })

  const { refetch } = useTokenAllowance({
    token,
    owner: account,
    spender,
  })

  const onSuccess = async (data) => {
    await refetch() // 撤销后刷新授权额度
  }

  // ... success callback 处理
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **撤销前显示确认**
   ```typescript
   // 撤销是危险操作，需要用户确认
   if (!confirm('确定撤销授权？')) return
   write()
   ```

2. **撤销后刷新数据**
   ```typescript
   // 撤销成功后刷新授权状态
   const { refetch } = useTokenAllowance({ ... })
   onSuccess: () => refetch()
   ```

3. **处理 Pending 状态**
   ```typescript
   // 防止重复提交
   <button disabled={isPending}>撤销中...</button>
   ```

#### ❌ Don'ts

1. **不要撤销不认识的合约**
   ```typescript
   // 错误：不要盲目撤销
   <button onClick={() => revokeAll()}>撤销所有授权</button>

   // 正确：让用户逐个确认
   approvals.map(a => <RevokeButton token={a.token} spender={a.spender} />)
   ```

2. **不要假设撤销立即生效**
   ```typescript
   // 错误：撤销后立即检查
   write()
   setTimeout(() => checkAllowance(), 1000) // 太早了

   // 正确：等待交易确认
   onSuccess: async (hash) => {
     await waitForTransactionReceipt(hash)
     refetch()
   }
   ```

3. **不要撤销正在进行交易的合约授权**
   ```typescript
   // 如果用户正在使用某合约的交易，不要撤销
   if (isUsingProtocol(spender)) {
     return <div>该合约正在使用中</div>
   }
   ```

4. **不要忽略授权额度为 0 的情况**
   ```typescript
   // 授权额度为 0 时也可以撤销，但没必要
   if (allowance?.eq(0n)) {
     return <div>无需撤销</div>
   }
   ```
