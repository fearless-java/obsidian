> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/token/use-token-utilities.ts`

# useTokenUtilities Hook Tutorial

## 大白话讲讲这个hook的作用

`useTokenUtilities` *(一个React hook集合，包含多个代币状态检查工具，用于检查余额和授权是否足够)* 是一个工具类 hook 集合，提供代币相关的检查功能：

- **useHasSufficientBalance** *(检查钱包余额是否足够的hook)*：检查余额是否足够
- **useHasSufficientAllowance** *(检查授权额度是否足够的hook)*：检查授权额度是否足够
- **useMultipleTokenBalances** *(批量查询代币余额的hook)*：批量查询代币余额
- **useMultipleTokenAllowances** *(批量查询授权额度的hook)*：批量查询授权额度

它就像一个"代币状态检查器"，帮助 DApp 在执行操作前检查用户是否有足够的余额或授权。

## 讲讲为什么需要封装该hook

封装这些 utility hooks 的原因：

1. **操作前置检查**：在用户发起 swap、添加流动性等操作前，检查余额/授权是否足够

2. **批量查询优化**：一次性查询多个代币的余额或授权，减少 RPC 调用次数

3. **UI 状态控制**：根据余额/授权状态决定是否显示"授权"按钮或"执行"按钮

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）

**useHasSufficientBalance:**
```typescript
address: string | null        // 钱包地址
tokenAddress: string | null   // 代币地址
amount: bigint | null         // 需要检查的金额
```

**useHasSufficientAllowance:**
```typescript
owner: string | null          // 持有人地址
spender: string | null        // 被授权方地址
tokenAddress: string | null   // 代币地址
amount: bigint | null         // 需要检查的授权额度
```

**useMultipleTokenBalances:**
```typescript
address: string | null        // 钱包地址
tokenAddresses: string[]       // 代币地址数组
```

**useMultipleTokenAllowances:**
```typescript
owner: string | null          // 持有人地址
spender: string | null        // 被授权方地址
tokenAddresses: string[]       // 代币地址数组
```

### 输出（返回值）

**useHasSufficientBalance / useHasSufficientAllowance:**
```typescript
{
  data: boolean | null   // true = 足够, false = 不足, null = 参数无效
}
```

**useMultipleTokenBalances / useMultipleTokenAllowances:**
```typescript
{
  data: Record<string, bigint | string> | null   // { 代币地址: 数量 }
}
```

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个检查代币余额是否足够的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useHasSufficientBalance的hook，用来检查用户的代币余额是否足够完成某个操作。然后明确几个关键点。第一，参数要包括钱包地址、代币地址和要检查的金额这三个。第二，底层调用链上的getBalance和hasSufficientBalance函数来判断。第三，返回一个布尔值，true表示够了，false表示不够。第四，用React Query来管理数据，这样能省掉很多重复的工作。第五，如果参数无效，比如地址是空的，要返回null而不是false，因为null表示还不知道，false表示确认不够，这是两个完全不同的状态。

### 这里面有几个地方特别容易出错

金额一定要传最小单位，这个已经强调过很多次了，金额单位搞错了判断肯定不准。参数无效的时候要返回null，这个很重要，null和false的含义完全不同，null是说还没办法判断，false是确认判断过了不够。临界值检查的时候要用大于等于而不是大于，等于的情况也要算够。

### 数据刷新这里有讲究

这类检查的hook数据变化频率其实不太高，余额和授权不会每秒都在变，所以staleTime可以设长一点，减少不必要的链上查询。授权变化之后要刷新一下allowance的检查，因为用户可能刚刚授权了一笔钱出去。如果用到了多个相关的hook，要注意它们之间的依赖关系，合理安排刷新顺序。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import {
  useHasSufficientBalance,
  useHasSufficientAllowance,
  useMultipleTokenBalances,
} from '@sushiswap/hooks'
import { useStellarWallet } from './wallet'

function SwapButton() {
  const { address } = useStellarWallet()
  const [amount, setAmount] = useState('100')
  const amountIn最小单位 = parseUnits(amount, 6)

  // 检查余额是否足够
  const { data: hasEnoughBalance } = useHasSufficientBalance({
    address,
    tokenAddress: tokenInContract,
    amount: amountIn最小单位,
  })

  // 检查授权是否足够
  const { data: hasEnoughAllowance } = useHasSufficientAllowance({
    owner: address,
    spender: poolAddress,
    tokenAddress: tokenInContract,
    amount: amountIn最小单位,
  })

  const canSwap = hasEnoughBalance && hasEnoughAllowance

  return (
    <button disabled={!canSwap}>
      {!hasEnoughBalance ? '余额不足' : !hasEnoughAllowance ? '需要授权' : 'Swap'}
    </button>
  )
}
```

### 常见使用场景

1. **Swap 按钮状态控制**：根据余额和授权状态决定按钮文案
   ```tsx
   const getButtonState = () => {
     if (hasEnoughBalance === null) return '加载中'
     if (!hasEnoughBalance) return '余额不足'
     if (!hasEnoughAllowance) return '需要授权'
     return 'Swap'
   }
   ```

2. **批量检查多个代币**：一次性检查多个代币状态
   ```tsx
   const { data: balances } = useMultipleTokenBalances({
     address,
     tokenAddresses: [usdc, weth, wbtc],
   })
   ```

3. **授权/执行流程控制**：引导用户完成授权后再执行
   ```tsx
   {needsApproval ? (
     <ApproveButton onClick={approve} />
   ) : (
     <ExecuteButton onClick={execute} disabled={!hasEnoughBalance} />
   )}
   ```

### Dos and Don'ts

**Dos:**
- ✅ 使用 `useHasSufficientBalance` 和 `useHasSufficientAllowance` 组合判断
- ✅ 设置较长的 `staleTime` 减少链上查询
- ✅ 在操作执行前进行余额/授权检查

**Don'ts:**
- ❌ 不要假设余额或授权状态，always 检查
- ❌ 不要忽略 null 状态，应该显示加载中
- ❌ 不要使用人类可读的金额比较，必须是最小单位
