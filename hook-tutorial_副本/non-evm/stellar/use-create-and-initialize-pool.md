> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/factory/use-create-and-initialize-pool.ts`

# useCreateAndInitializePool Hook Tutorial

## 大白话讲讲这个hook的作用

`useCreateAndInitializePool` *(一个React hook，用于创建并初始化新的集中流动性池，封装了完整的创建交易流程和通知)* 是一个用于创建并初始化流动性池的 mutation hook。它：

- 创建一个新的集中流动性池
- 设置初始价格

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **交易封装**：封装创建池子的完整交易流程
2. **Toast 通知**：创建过程中和成功后显示通知
3. **缓存失效**：创建成功后刷新池子列表

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  userAddress: string                 // 用户地址
  tokenA: string                    // 代币 A 地址
  tokenB: string                    // 代币 B 地址
  fee: number                       // 手续费率
  sqrtPriceX96: bigint              // 初始价格（sqrt price * 2^96）
  signTransaction: (xdr: string) => Promise<string>  // 签名函数
}
```

### 输出（返回值）
```typescript
{
  mutate: () => void
  mutateAsync: () => Promise<{ result, params }>
  isPending: boolean
}
```

### 核心执行逻辑

1. **显示 info toast**：创建开始时显示通知
2. **调用合约**：执行 `createAndInitializePool` *(创建并初始化池子的合约方法)*
3. **显示 success toast**：创建成功后显示通知
4. **刷新缓存**：invalidate *(React Query方法，使查询缓存失效)* pool 相关查询

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个创建并初始化流动性池的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useCreateAndInitializePool的hook，用来创建并初始化一个流动性池。然后明确几个关键点。第一，参数包括两种代币的地址、手续费率、初始价格sqrtPriceX96，还有用户地址。第二，封装创建池子的合约调用，这个操作比较复杂，需要处理很多事情。第三，提供Toast通知，让用户知道当前在处理中、成功了还是失败了。第四，创建成功后要刷新池子列表查询，不然新池子创建了但列表里看不到。第五，用React Query的useMutation来处理，因为这是写操作不是读操作。

### 这里面有几个地方特别容易出错

签名函数必须正确实现，创建池子需要签名，没签好交易就发不出去。池子可能已经存在了，要优雅处理这种情况，不能直接报错，要给用户一个清晰的提示。错误处理要做好，显示友好的错误信息，不要让用户看到一堆技术术语。

### 数据刷新这里有讲究

用useMutation处理写操作，组件里能知道当前是不是在处理中。成功后要立即invalidate pool相关查询，让列表能刷新显示新池子。Toast通知要做好，让用户清楚知道操作结果。这个hook是工厂类的核心操作，做好了用户就能创建新的交易对。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useCreateAndInitializePool } from '@sushiswap/hooks'
import { useStellarWallet } from './wallet'
import { useQueryClient } from '@tanstack/react-query'

function CreatePoolForm() {
  const { address, signTransaction } = useStellarWallet()
  const queryClient = useQueryClient()

  const { mutate: createPool, isPending } = useCreateAndInitializePool({
    onSuccess: (result) => {
      toast.success('池子创建成功！')
      queryClient.invalidateQueries(['pools'])
    },
    onError: (error) => {
      toast.error(`创建失败: ${error.message}`)
    }
  })

  const handleCreate = () => {
    createPool({
      userAddress: address,
      tokenA: usdcContract,
      tokenB: wethContract,
      fee: 3000, // 0.3%
      sqrtPriceX96: parsePrice('1'), // 初始价格
      signTransaction,
    })
  }

  return (
    <button onClick={handleCreate} disabled={isPending}>
      {isPending ? '创建中...' : '创建池子'}
    </button>
  )
}
```

### 常见使用场景

1. **DEX 池子创建页面**：让用户创建新的交易池
   ```tsx
   const { mutate: create } = useCreateAndInitializePool({
     onSuccess: () => navigateToPool(newPoolAddress)
   })
   ```

2. **批量创建**：为多个交易对创建池子
   ```tsx
   for (const pair of pairs) {
     create({ ...pair, signTransaction })
   }
   ```

3. **创建后跳转**：自动跳转到新池子
   ```tsx
   const { mutateAsync: createAsync } = useCreateAndInitializePool({
     onSuccess: (result) => result.poolAddress
   })
   const result = await createAsync(params)
   navigate(`/pool/${result.poolAddress}`)
   ```

### Dos and Don'ts

**Dos:**
- ✅ 签名函数必须正确实现
- ✅ 创建成功后刷新池子列表
- ✅ 使用 toast 通知用户状态变化

**Don'ts:**
- ❌ 不要重复点击（使用 isPending 防重）
- ❌ 不要忽略池子已存在的错误
- ❌ 初始价格必须是有效值
