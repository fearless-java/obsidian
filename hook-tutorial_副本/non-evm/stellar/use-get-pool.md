> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/factory/use-get-pool.ts`

# useGetPool Hook Tutorial

## 大白话讲讲这个hook的作用

`useGetPool` *(一个React hook，通过代币对和手续费率查询池子地址，是池子发现的核心hook)* 是一个用于通过代币对和手续费率查询池子地址的 hook。给定：

- Token A 和 Token B
- Fee tier（手续费率）

返回对应的池子合约地址。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **工厂合约封装**：调用 DEX 工厂合约 *(去中心化交易所的工厂合约，负责创建和管理池子)* 获取池子地址
2. **React Query 集成**：管理查询和缓存

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  tokenA: string                    // 代币 A 地址
  tokenB: string                   // 代币 B 地址
  fee: number                      // 手续费率
}
```

### 输出（返回值）
```typescript
{
  data: string | null              // 池子地址
}
```

### 核心执行逻辑

1. **调用工厂**：使用 `getPoolDirectSDK(params)` *(工厂合约SDK方法，通过代币对和手续费率获取池子地址)* 查询
2. **返回结果**：返回池子地址或 null

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个通过代币对查询池子地址的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useGetPool的hook，通过代币对和手续费率来查询某个区块链上的池子地址。然后明确几个关键点。第一，参数包括代币A地址、代币B地址和手续费率，三个缺一不可。第二，调用工厂合约获取池子地址，工厂合约是专门管理池子创建和查询的。第三，返回池子地址，如果池子不存在就返回null。第四，用React Query管理数据。

### 这里面有几个地方特别容易出错

返回null不一定就是错误，也可能是池子真的还没创建，所以不要把null当作错误来处理。参数任何一个是null都不能发起查询，无效的参数查了也是白查。

### 数据刷新这里有讲究

参数为null时不发起查询，这个enabled条件要设好。这个hook是池子发现的核心，用户选了一个代币对之后要靠这个hook来找到池子地址，找到了才能做后续操作。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useGetPool } from '@sushiswap/hooks'

function PoolFinder({ tokenA, tokenB, fee }: Props) {
  const { data: poolAddress, isLoading } = useGetPool({
    tokenA,
    tokenB,
    fee,
  })

  return (
    <div>
      {isLoading ? (
        <p>查找池子中...</p>
      ) : poolAddress ? (
        <p>池子地址: {poolAddress}</p>
      ) : (
        <p>池子不存在</p>
      )}
    </div>
  )
}
```

### 常见使用场景

1. **Swap 页面**：查找选定的代币对的池子
   ```tsx
   const { data: poolAddress } = useGetPool({
     tokenA: tokenIn.address,
     tokenB: tokenOut.address,
     fee: selectedFeeTier,
   })
   ```

2. **添加流动性**：先查找池子是否存在
   ```tsx
   const { data: pool } = useGetPool({ tokenA, tokenB, fee })

   if (pool) {
     // 池子存在，添加流动性
   } else {
     // 池子不存在，需要先创建
     showCreatePoolButton()
   }
   ```

3. **检查池子是否存在**：用于条件渲染
   ```tsx
   const { data: exists } = useGetPool({ tokenA, tokenB, fee })
   const poolExists = Boolean(exists)
   ```

### Dos and Don'ts

**Dos:**
- ✅ 传入排序后的代币地址（tokenA < tokenB）
- ✅ 使用 `enabled` 条件避免无效查询
- ✅ 配合 `usePoolExists` *(检查池子是否存在的hook)* 使用

**Don'ts:**
- ❌ 不要假设池子一定存在
- ❌ 不要忽略 fee 参数，不同 fee 的池子不同
- ❌ 代币地址顺序要一致
