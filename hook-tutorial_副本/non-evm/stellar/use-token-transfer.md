> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/token/use-token-transfer.ts`

# useTransferToken / useTransferFromToken Hook Tutorial

## 大白话讲讲这个hook的作用

`useTransferToken` *(一个React hook，用于从当前钱包转账代币给其他人，封装了Stellar Soroban的transferToken交易)* 和 `useTransferFromToken` *(一个React hook，用于授权后从别人账户转账代币，封装了Stellar Soroban的transferFromToken交易)* 是用于转账代币的 mutation hooks。它们就像"银行转账功能"：

- **useTransferToken**：直接从你的钱包转账代币给其他人
- **useTransferFromToken**：授权后，从别人的账户转账（需要先授权）

两个 hook 都封装了 Stellar Soroban 合约的代币转账交易。

## 讲讲为什么需要封装该hook

封装这两个 hook 的原因：

1. **交易签名封装**：转账操作需要签名，封装后简化调用

2. **统一错误处理**：转账失败时的错误处理逻辑统一

3. **简化 API**：调用者只需要传入参数，不需要关心底层合约调用细节

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）

**TransferTokenParams:**
```typescript
{
  to: string       // 收款人地址
  amount: bigint  // 转账数量（最小单位）
  tokenAddress: string  // 代币合约地址
}
```

**TransferFromTokenParams:**
```typescript
{
  from: string     // 转出方地址
  to: string       // 收款人地址
  amount: bigint   // 转账数量（最小单位）
  tokenAddress: string  // 代币合约地址
}
```

### 输出（返回值）
```typescript
{
  mutate: (params: ...) => void
  mutateAsync: (params: ...) => Promise<...>
  isPending: boolean
  error: Error | null
}
```

### 核心执行逻辑

1. **校验参数**：确保所有必要参数都存在
2. **调用合约**：使用 SDK 的 `transferToken` *(Stellar Soroban合约方法，用于转账代币)* 或 `transferFromToken` *(Stellar Soroban合约方法，用于授权转账从别人账户转出)* 函数
3. **等待确认**：等待区块链确认交易
4. **错误处理**：捕获并记录错误

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个代币转账的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useTransferToken的hook，用来在某个区块链网络上转账代币。然后明确几个关键点。第一，参数要包括收款人地址、转账数量和代币地址这三个。第二，底层要调用链上的转账函数，封装好就不用每次都写那些复杂的合约调用了。第三，用React Query的useMutation来处理，因为转账是写操作不是读操作。第四，要提供pending状态让UI知道交易正在处理中，还要有错误处理，万一出错了得让用户知道。第五，数量要用bigint类型，而且是最小单位，不是我们平时看到的那种带小数的格式。

### 这里面有几个地方特别容易出错

金额一定要传最小单位，如果你传了人类能看懂的数字，比如100USDC，传进去之后可能就变成0.0001了，因为程序会以为那是最小单位。钱包必须已经连接好了才能转账，没连接就去转肯定会失败。错误处理要做好，转账失败了用户肯定想知道是怎么回事，不能就这么无声无息的过去了。

### 状态管理要注意什么

转账是写操作，必须用useMutation，不能用useQuery。转账成功后一般要刷新一下余额，让用户看到最新的金额。可以根据交易的状态来显示不同的UI，比如交易pending的时候就给用户一个提示。状态管理在这种地方特别重要，因为涉及到钱，用户会很敏感。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useTransferToken } from '@sushiswap/hooks'
import { parseUnits } from 'ethers'
import { useStellarWallet } from './wallet'

function TransferComponent() {
  const { address } = useStellarWallet()
  const [recipient, setRecipient] = useState('')
  const [amount, setAmount] = useState('')

  const { mutate: transfer, isPending, error } = useTransferToken({
    onSuccess: (result) => {
      toast.success(`转账成功！交易哈希: ${result.hash}`)
      // 刷新余额
      queryClient.invalidateQueries(['token-balance'])
    },
    onError: (error) => {
      toast.error(`转账失败: ${error.message}`)
    }
  })

  const handleTransfer = () => {
    const amountIn最小单位 = parseUnits(amount, 6) // USDC是6位小数
    transfer({
      to: recipient,
      amount: amountIn最小单位,
      tokenAddress: usdcContract,
    })
  }

  return (
    <div>
      <input
        value={recipient}
        onChange={(e) => setRecipient(e.target.value)}
        placeholder="收款人地址"
      />
      <input
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        placeholder="数量"
      />
      <button onClick={handleTransfer} disabled={isPending}>
        {isPending ? '转账中...' : '转账'}
      </button>
      {error && <p>错误: {error.message}</p>}
    </div>
  )
}
```

### 常见使用场景

1. **代币转账**：用户之间互相转账代币
   ```tsx
   mutate({
     to: recipientAddress,
     amount: parseUnits(amount, decimals),
     tokenAddress: tokenContract,
   })
   ```

2. **批量转账**：为多个收款人转账（循环调用）
   ```tsx
   for (const recipient of recipients) {
     mutate({ to: recipient, amount, tokenAddress })
   }
   ```

3. **转账后刷新余额**：转账成功后自动刷新显示最新余额
   ```tsx
   const { mutate: transfer } = useTransferToken({
     onSuccess: () => {
       queryClient.invalidateQueries(['token-balance', address, tokenAddress])
     }
   })
   ```

### Dos and Don'ts

**Dos:**
- ✅ 使用 `parseUnits` *(ethers.js的工具函数，将人类可读的代币数量转换为最小单位)* 将金额转为最小单位
- ✅ 转账前验证收款人地址格式是否正确
- ✅ 显示明确的 pending 状态防止重复点击

**Don'ts:**
- ❌ 不要使用人类可读的金额格式直接转账
- ❌ 不要在签名等待期间允许用户再次点击（使用 isPending）
- ❌ 转账后不要忘记处理错误情况
