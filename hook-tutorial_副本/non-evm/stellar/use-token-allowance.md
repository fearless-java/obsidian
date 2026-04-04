> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/token/use-token-allowance.ts`

# useTokenAllowance Hook Tutorial

## 大白话讲讲这个hook的作用

`useTokenAllowance` *(一个React hook，用于查询代币授权额度，通过Stellar Soroban合约的getTokenAllowance方法获取)* 是一个用于查询 Stellar 代币授权额度的 hook。你可以把它想象成"检查信用卡预授权额度"：

- **owner**：持卡人（你的钱包地址）
- **spender**：商户（合约地址，如某个 DApp）
- **tokenAddress**：币种（代币合约地址）

这个 hook 返回 owner 授权给 spender 使用多少数量的某个代币。比如你授权了一个 DEX 合约使用你的 1000 USDC，这个 hook 就会返回 1000（以最小单位表示）。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **桥接 SDK 与 React**：Stellar SDK 的 `getTokenAllowance` *(Stellar Soroban SDK的方法，用于查询代币授权额度)* 是异步函数，封装成 hook 后可以方便地在 React 组件中使用

2. **React Query 集成**：自动处理加载状态、错误状态和缓存

3. **参数校验**：在查询函数内部检查参数有效性，避免无效的 RPC 调用

4. **空值安全**：当 owner/spender/tokenAddress 任一为 null 时，返回 null 而不是发起无效查询

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
owner: string | null         // 代币持有人地址
spender: string | null       // 被授权使用代币的地址
tokenAddress: string | null  // 代币合约地址
```

### 输出（返回值）
```typescript
// React Query 返回值
{
  data: bigint | null         // 授权额度（最小单位），null 表示参数无效或查询中
  isLoading: boolean
  isPending: boolean
  error: Error | null
  ...
}
```

### 核心执行逻辑

1. **参数校验**：如果 owner/spender/tokenAddress 任一为 null，直接返回 null
2. **查询**：调用 `getTokenAllowance(owner, spender, tokenAddress)` *(调用链上合约查询授权额度)* 从链上获取授权额度
3. **缓存**：使用 React Query 自动缓存结果，默认 5 分钟（可配置）

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个查询代币授权额度的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useTokenAllowance的hook，用来查询某个区块链网络上的代币授权额度。然后明确几个关键点。第一，参数要包括代币持有人地址、被授权方地址和代币合约地址这三个。第二，底层封装链上的getTokenAllowance查询函数，不用每次都写那些复杂的合约调用。第三，用React Query管理数据的获取和缓存，这样能省掉很多重复的工作。第四，如果任何一个参数是null，就不要发起查询，enabled要设成false，查了也是白查还浪费资源。第五，queryKey要包含owner、spender和tokenAddress，这样缓存才能正确区分不同的查询。第六，返回的授权额度是bigint类型。

### 这里面有几个地方特别容易出错

参数校验一定要做好，所有参数都有效才能发起查询，任何一个是null都不应该发起请求。enabled控制要用`Boolean(owner && spender && tokenAddress)`这种方式，确保所有参数都存在才启用查询。参数无效的时候要返回null而不是undefined，undefined和null在TypeScript里是两回事，返回null能保持类型一致，程序处理起来不容易出错。

### 数据刷新这里有讲究

授权额度变化其实不频繁，所以staleTime可以设长一点，比如一分钟，减少不必要的链上查询。可以监听`['stellar', 'token', 'allowance']`相关的query key，在授权变化后刷新UI显示。这个hook看起来简单，但涉及到钱，所以还是要谨慎一点。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useTokenAllowance } from '@sushiswap/hooks'
import { useStellarWallet } from './wallet'

function AllowanceCheck() {
  const { address } = useStellarWallet()
  const spender = '0x1234...5678' // 要授权的合约地址
  const tokenAddress = '0xABCD...EFGH' // 代币合约地址

  const { data: allowance, isLoading } = useTokenAllowance({
    owner: address,
    spender: spender,
    tokenAddress: tokenAddress,
  })

  return (
    <div>
      <p>当前授权额度: {allowance?.toString() ?? '0'}</p>
      <p>状态: {isLoading ? '加载中' : '已加载'}</p>
    </div>
  )
}
```

### 常见使用场景

1. **检查是否需要授权**：在执行 swap 前检查授权额度是否足够
   ```tsx
   const { data: allowance } = useTokenAllowance({
     owner: address,
     spender: poolAddress,
     tokenAddress: tokenA,
   })

   const needsApproval = allowance === 0n || allowance < amountToSwap
   ```

2. **显示授权进度**：向用户显示已授权额度占总余额的比例

3. **批量检查多个代币**：对多个代币批量检查授权状态

### Dos and Don'ts

**Dos:**
- ✅ 结合 `useHasSufficientAllowance` *(一个检查授权额度是否足够的hook)* 使用，判断是否可以执行操作
- ✅ 授权变化后调用 `queryClient.invalidateQueries` 刷新 UI
- ✅ 设置较长的 `staleTime` 减少不必要的链上查询

**Don'ts:**
- ❌ 不要在每次渲染时都重新创建 queryKey，要保持稳定
- ❌ 不要忽略 `enabled` 条件，确保参数完整才查询
- ❌ 不要直接比较 bigint 和普通数字，使用 `bigint` 字面量或转换
