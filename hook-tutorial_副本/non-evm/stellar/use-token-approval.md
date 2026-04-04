> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/token/use-token-approval.ts`

# useApproveToken Hook Tutorial

## 大白话讲讲这个hook的作用

`useApproveToken` *(一个React hook，用于授权代币给其他合约使用，封装了Stellar Soroban的approveToken交易)* 是一个用于授权代币给其他合约使用的 mutation hook。你可以把它想象成"给服务员授权刷卡额度"：

- 你（owner）授权给某个 DApp/合约（spender）使用你的一定数量的代币
- 这个操作需要签名交易

当用户要进行 swap、添加流动性等操作时，通常需要先授权合约使用他们的代币。这个 hook 封装了这个授权交易。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **交易签名封装**：Stellar 的授权操作需要签名，封装后简化调用

2. **自动缓存失效**：授权成功后自动使 `allowance`、`multipleAllowances`、`hasSufficientAllowance` 相关缓存失效，确保 UI 及时更新

3. **Toast 通知**：封装了成功/失败的 toast 提示

4. **错误处理**：统一的错误处理逻辑

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  spender: string       // 被授权使用代币的合约地址
  amount: bigint        // 授权数量（最小单位）
  tokenAddress: string  // 代币合约地址
}
```

### 输出（返回值）
```typescript
// React Query Mutation 返回值
{
  mutate: (params: ApproveTokenParams) => void
  mutateAsync: (params: ApproveTokenParams) => Promise<...>
  isPending: boolean
  error: Error | null
  ...
}
```

### 核心执行逻辑

1. **发起授权**：调用 `approveToken(spender, amount, tokenAddress)` *(Stellar Soroban合约的授权方法，授权spender使用指定数量的代币)* 发起链上授权交易
2. **等待确认**：等待交易被区块链确认
3. **缓存失效**：授权成功后，invalidate *(React Query方法，用于使查询缓存失效，触发重新获取)* 相关的 query key
4. **错误处理**：记录错误并显示 toast

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个代币授权的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useApproveToken的hook，用来授权代币给某个合约使用。然后明确几个关键点。第一，参数要包括被授权方的地址、授权的数量和代币地址这三个。第二，底层要封装链上的授权交易函数，不用每次都写那些复杂的合约调用。第三，用React Query的useMutation来处理，因为授权是写操作。第四，授权成功后要自动清除相关的查询缓存，比如授权额度查询和余额查询，不然UI上显示的还是旧数据。第五，提供onMutate、onSuccess、onError这些回调，方便在不同的阶段做不同的处理。第六，组件里要能展示pending状态和错误信息，让用户知道发生了什么。

### 这里面有几个地方特别容易出错

amount一定要是bigint类型的最小单位，不是人类能看懂的数字格式。缓存失效一定要做，授权完了但UI上显示的授权额度还是0，这种体验会很差。钱包必须已经连接好了才能授权，不然会失败。交易哈希要保存好，方便用户查这笔交易的状态。

### 状态管理要注意什么

授权是写操作，必须用useMutation，不能用useQuery，这个应该很好理解。isPending状态可以用来禁用按钮或者显示加载中的提示，防止用户重复点击。授权成功后一般要重新检查一下授权额度够不够，万一不够可能还要再授权一次。状态管理做得好，用户操作起来才会流畅，也不会出现一些奇怪的bug。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useApproveToken, useTokenAllowance } from '@sushiswap/hooks'
import { useStellarWallet } from './wallet'
import { parseUnits } from 'ethers'

function SwapComponent() {
  const { address } = useStellarWallet()
  const [amount, setAmount] = useState('100')

  const { mutate: approve, isPending: isApproving } = useApproveToken({
    onSuccess: () => {
      toast.success('授权成功！')
    },
    onError: (error) => {
      toast.error(`授权失败: ${error.message}`)
    }
  })

  const handleApprove = () => {
    const amountIn最小单位 = parseUnits(amount, 6) // USDC是6位小数
    approve({
      spender: poolAddress,
      amount: amountIn最小单位,
      tokenAddress: usdcContract,
    })
  }

  return (
    <button onClick={handleApprove} disabled={isApproving}>
      {isApproving ? '授权中...' : '授权'}
    </button>
  )
}
```

### 常见使用场景

1. **Swap 前授权**：用户 swap 前需要先授权代币给池子
   ```tsx
   const needsApproval = allowance < amountIn
   if (needsApproval) {
     approve({ spender: poolAddress, amount: MaxUint256, tokenAddress })
   }
   ```

2. **无限授权**：用户可以选择授权最大数量避免后续重复授权
   ```tsx
   mutate({ spender, amount: MaxUint256, tokenAddress })
   ```

3. **授权后自动执行**：授权成功后自动执行下一步操作
   ```tsx
   const { mutate: approve } = useApproveToken({
     onSuccess: () => executeSwap()
   })
   ```

### Dos and Don'ts

**Dos:**
- ✅ 使用 `useTokenAllowance` 检查当前授权额度后再决定是否需要授权
- ✅ 显示明确的授权状态给用户（已授权/未授权/授权中）
- ✅ 设置 `onSuccess` 回调刷新 allowance 查询

**Don'ts:**
- ❌ 不要在组件卸载后仍然更新 state（使用 isMounted 检查）
- ❌ 不要忘记处理授权失败的情况
- ❌ 授权数量不要使用人类可读的格式，必须转换为最小单位
