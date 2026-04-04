> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/position/use-positions.ts`

# usePositions Hook Tutorial

## 大白话讲讲这个hook的作用

`usePositions` *(一个React hook集合，提供流动性头寸的查询和操作功能，包括获取头寸列表、单个头寸、未收集费用和执行收集)* 是一个用于管理用户流动性头寸的核心 hook 集合，包含：

- **useUserPositions**：获取用户所有头寸
- **usePosition**：获取单个特定头寸
- **useUncollectedFees**：获取头寸的未收集费用
- **useCollectFees**：执行收集费用操作

## 讲讲为什么需要封装该hook

封装这个 hook 集合的原因：

1. **头寸生命周期管理**：提供完整的头寸查询和操作功能
2. **费用收集封装**：封装费用收集的交易签名和执行
3. **缓存失效**：收集费用后自动刷新相关查询

## 讲讲该hook的执行逻辑和数据流向

### useUserPositions

**输入：**
```typescript
{
  userAddress?: string          // 用户地址
  excludeDust?: boolean         // 是否排除小额头寸
}
```

**输出：**
```typescript
{
  data: PositionInfo[]          // 头寸列表
}
```

### useCollectFees

**输入：**
```typescript
{
  tokenId: number
  recipient: string
  amount0Max: bigint
  amount1Max: bigint
  signTransaction: (xdr: string) => Promise<string>  // 交易签名函数
  signAuthEntry: (entryPreimageXdr: string) => Promise<string>  // Auth Entry签名函数
}
```

**输出：**
```typescript
{
  mutate: () => void
  mutateAsync: () => Promise<CollectFeesResult>
  isPending: boolean
}
```

### 核心执行逻辑

1. **获取头寸**：调用 `positionService.getUserPositionsWithFees` *(获取用户所有头寸及费用的方法)* 获取头寸列表
2. **获取单个头寸**：调用 `positionService.getPosition` *(PositionService方法，获取单个头寸详情)* 获取头寸
3. **获取费用**：调用 `positionService.getUncollectedFees` *(获取头寸未收集费用的方法)* 获取费用
4. **收集费用**：调用 `positionService.collectFees` *(执行费用收集的合约方法)* 并等待交易确认

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个管理流动性头寸的hook集合时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似usePositions的hook集合，用来管理某个区块链上的流动性头寸。然后明确几个关键点。第一，useUserPositions用于获取用户所有的头寸。第二，usePosition用于获取单个特定头寸的详细信息。第三，useUncollectedFees用于获取某个头寸还没收集的费用。第四，useCollectFees用于执行费用收集操作，把头寸累计的费用真正拿到手。第五，收集费用这种写操作之后要自动清除相关的查询缓存，不然UI显示的还是旧数据。第六，费用收集需要签名授权，Stellar上有特殊的auth entry机制，需要单独签名。

### 这里面有几个地方特别容易出错

费用收集需要签名，而且不只是签一笔交易，可能要签多个auth entry，这个要处理妥当。缓存失效一定要做，收集完了但UI还显示着可以收集的费用，这种体验会很差。交易可能失败，所以错误处理要做好。

### 数据刷新这里有讲究

头寸数据相对稳定，staleTime可以设成1分钟。费用收集这种写操作完成后要立即invalidate positions相关的查询，让UI能立刻刷新显示。这个hook集合提供了完整的头寸生命周期管理，从查到操作都有了。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import {
  useUserPositions,
  usePosition,
  useUncollectedFees,
  useCollectFees,
} from '@sushiswap/hooks'

function PositionsPage() {
  const { address } = useStellarWallet()

  // 获取用户所有头寸
  const { data: positions } = useUserPositions({ userAddress: address })

  return (
    <div>
      {positions?.map((pos) => (
        <PositionCard key={pos.tokenId} position={pos} />
      ))}
    </div>
  )
}

// 收集费用示例
function CollectFeesButton({ tokenId }: { tokenId: number }) {
  const { mutate: collect, isPending } = useCollectFees({
    onSuccess: () => {
      toast.success('费用收集成功！')
      queryClient.invalidateQueries(['positions'])
    }
  })

  return (
    <button onClick={() => collect({
      tokenId,
      recipient: address,
      amount0Max: MaxUint256,
      amount1Max: MaxUint256,
      signTransaction,
      signAuthEntry,
    })} disabled={isPending}>
      {isPending ? '收集ing...' : '收集费用'}
    </button>
  )
}
```

### 常见使用场景

1. **头寸列表页面**：显示用户所有头寸
   ```tsx
   const { data: positions } = useUserPositions({ userAddress: address })
   ```

2. **单个头寸详情**：获取特定头寸的详细信息
   ```tsx
   const { data: position } = usePosition({ tokenId })
   ```

3. **批量收集费用**：一次性收集所有头寸的费用
   ```tsx
   positions?.forEach(pos => {
     if (pos.feesToken0 > 0n || pos.feesToken1 > 0n) {
       collect({ tokenId: pos.tokenId, ...params })
     }
   })
   ```

### Dos and Don'ts

**Dos:**
- ✅ 签名函数需要支持 Auth Entry 签名（Stellar Soroban 特性）
- ✅ 费用收集后调用 `queryClient.invalidateQueries` 刷新
- ✅ 使用 `useUncollectedFees` 显示可收集的费用金额

**Don'ts:**
- ❌ 不要假设用户一定有头寸或费用
- ❌ 不要忽略 isPending 状态
- ❌ 签名函数必须正确实现，否则交易会失败
