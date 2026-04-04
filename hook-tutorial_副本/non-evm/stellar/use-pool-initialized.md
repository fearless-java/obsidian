> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/pool/use-pool-initialized.ts`

# usePoolInitialized Hook Tutorial

## 大白话讲讲这个hook的作用

`usePoolInitialized` *(一个React hook，用于检查流动性池是否已初始化，Stellar的集中流动性池需要先初始化价格区间才能正常操作)* 是一个用于检查池子是否已初始化的 hook。Stellar 的集中流动性池在创建后需要先初始化价格区间才能正常操作。这个 hook 检查：

- 池子是否存在
- 池子是否已完成初始化

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **状态前置检查**：很多操作（如添加流动性、swap）需要池子先初始化
2. **合约调用封装**：封装 `isPoolInitialized` *(Soroban合约方法，检查池子是否已初始化)* 函数调用
3. **提供失效方法**：导出 `invalidatePoolInitializedQuery` *(手动刷新池子初始化状态的工具函数)* 方便在池子初始化后刷新

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
address: string | null | undefined   // 池子地址
```

### 输出（返回值）
```typescript
{
  data: boolean           // true = 已初始化，false = 未初始化或不存在
}
```

### 额外导出
```typescript
invalidatePoolInitializedQuery(queryClient, address)  // 手动刷新池子初始化状态
```

### 核心执行逻辑

1. **调用合约**：使用 `isPoolInitialized(address)` 检查
2. **返回布尔值**：返回池子是否已初始化

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个检查池子是否已初始化的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似usePoolInitialized的hook，用来检查某个区块链上的池子是否已经初始化。然后明确几个关键点。第一，参数就是池子地址。第二，调用链上合约检查初始化状态。第三，返回布尔值，true表示已初始化，false表示未初始化或不存在。第四，导出一个手动刷新池子初始化状态的方法invalidateQuery，方便在其他操作（比如创建池子）完成后刷新状态。第五，用React Query管理数据，支持重试。

### 这里面有几个地方特别容易出错

返回false的时候不一定是池子不存在，也可能是池子存在但还没初始化，这两个情况要区分开其实挺难的。所以返回false之后还是要触发重试，通过重试次数来区分是暂时性问题还是真的不存在。导出的invalidateQuery方法很有用，创建池子之后要调用它刷新状态，不然UI可能还显示池子不存在。

### 数据刷新这里有讲究

初始化状态不常变化，所以staleTime可以设长一点，比如30秒，没必要频繁刷新。RPC错误时要重试，因为可能只是临时网络问题。池子是很多操作的前置条件，所以这个hook虽然简单但很重要。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { usePoolInitialized } from '@sushiswap/hooks'

function PoolStatus({ poolAddress }: { poolAddress: string }) {
  const { data: isInitialized, isLoading } = usePoolInitialized(poolAddress)

  if (isLoading) return <p>检查中...</p>

  return (
    <div>
      {isInitialized ? (
        <p>池子已初始化，可以添加流动性</p>
      ) : (
        <p>池子未初始化</p>
      )}
    </div>
  )
}
```

### 常见使用场景

1. **添加流动性前置检查**：确保池子已初始化
   ```tsx
   const { data: isInitialized } = usePoolInitialized(poolAddress)

   if (!isInitialized) {
     return <Message type="warning">池子未初始化，请先初始化</Message>
   }

   return <AddLiquidityForm />
   ```

2. **初始化按钮显示**：池子未初始化时显示初始化按钮
   ```tsx
   {!isInitialized && (
     <InitializePoolButton poolAddress={poolAddress} />
   )}
   ```

3. **创建池子后刷新**：创建池子后手动刷新状态
   ```tsx
   import { invalidatePoolInitializedQuery } from '@sushiswap/hooks'

   const { mutate: createPool } = useCreatePool({
     onSuccess: () => {
       queryClient.invalidatePoolInitializedQuery(poolAddress)
     }
   })
   ```

### Dos and Don'ts

**Dos:**
- ✅ 在执行任何池子操作前检查初始化状态
- ✅ 提供手动刷新方法，在池子状态变化后调用
- ✅ 设置较长的 `staleTime` 因为初始化状态不常变化

**Don'ts:**
- ❌ 不要假设池子存在就一定已初始化
- ❌ 返回 false 不一定代表池子不存在
- ❌ 不要忘记在池子创建成功后刷新查询
