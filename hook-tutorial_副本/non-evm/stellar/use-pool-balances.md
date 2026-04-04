> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/pool/use-pool-balances.ts`

# usePoolBalances Hook Tutorial

## 大白话讲讲这个hook的作用

`usePoolBalances` *(一个React hook，用于查询用户在流动性池中的份额余额，返回用户作为流动性提供者存放的代币数量)* 是一个用于查询流动性池中用户份额的 hook。它返回：

- 用户在该池子中的 Token0 和 Token1 余额（作为流动性提供者）
- 这些是用户存放在池子中的代币数量

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **池子合约交互**：需要调用池子合约获取用户余额
2. **双地址参数**：需要合约地址和用户地址两个参数
3. **React Query 集成**：管理数据获取和缓存

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
address: string | null          // 池子合约地址
connectedAddress: StellarAddress | undefined  // 用户钱包地址
```

### 输出（返回值）
```typescript
{
  data: PoolBalance | null      // 用户在池子中的余额
  isLoading: boolean
  error: Error | null
}
```

### 核心执行逻辑

1. **参数校验**：确保池子地址和钱包地址都存在
2. **调用合约**：使用 `getPoolBalances(address, connectedAddress)` *(Soroban合约方法，获取用户在池子中的流动性份额)* 获取数据
3. **返回结果**：返回用户的池子代币余额

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个查询用户在流动性池中余额的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似usePoolBalances的hook，用来查询用户在某个区块链流动性池中的余额。然后明确几个关键点。第一，参数要包括池子合约地址和用户钱包地址，两个缺一不可。第二，调用链上的池子合约来获取用户作为流动性提供者存放的代币数量。第三，返回用户在池子中的代币数量。第四，用React Query管理数据。第五，如果任何一个参数无效就不要发起查询，浪费资源。

### 这里面有几个地方特别容易出错

两个参数都要校验，池子地址和用户地址，任意一个无效都不能发起查询。这个hook需要和池子合约交互，可能还要调用其他相关的合约，逻辑比单纯查代币余额要复杂一点。

### 数据刷新这里有讲究

用户余额可能因为其他操作而变化，比如添加流动性、移除流动性或者交易，所以staleTime不能设太长，10秒比较合适。这个数据相对简单，但涉及多个合约调用，所以还是要谨慎一点。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { usePoolBalances } from '@sushiswap/hooks'

function PoolPosition() {
  const { address } = useStellarWallet()

  const { data: balances, isLoading } = usePoolBalances({
    address: poolAddress,
    connectedAddress: address,
  })

  return (
    <div>
      <p>池子 Token0 余额: {balances?.token0}</p>
      <p>池子 Token1 余额: {balances?.token1}</p>
    </div>
  )
}
```

### 常见使用场景

1. **显示流动性头寸**：用户查看自己在某池子的份额
   ```tsx
   const { data: poolBalances } = usePoolBalances({
     address: poolAddress,
     connectedAddress: userAddress,
   })
   ```

2. **移除流动性预览**：计算移除后能获得多少代币
   ```tsx
   const { data: balances } = usePoolBalances(...)
   const shareRatio = liquidityToRemove / totalLiquidity
   const token0ToReceive = balances?.token0 * shareRatio
   ```

3. **计算头寸占比**：用户 liquidity 占池子总 liquidity 的比例
   ```tsx
   const positionRatio = balances?.liquidity / pool?.totalLiquidity
   ```

### Dos and Don'ts

**Dos:**
- ✅ 配合 `usePoolInfo` *(获取池子详细信息的hook)* 获取池子数据
- ✅ 设置较短的 `staleTime` 因为余额会变化
- ✅ 检查池子是否存在再查询

**Don'ts:**
- ❌ 不要忽略任一参数为空的情况
- ❌ 不要直接显示原始余额，应该格式化后显示
- ❌ 不要在组件卸载后更新 state
