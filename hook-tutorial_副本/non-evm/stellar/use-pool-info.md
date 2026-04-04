> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/pool/use-pool-info.ts`

# usePoolInfo Hook Tutorial

## 大白话讲讲这个hook的作用

`usePoolInfo` *(一个React hook，用于获取流动性池的完整信息，包括代币地址、当前tick、手续费率、流动性总量和价格)* 是一个用于获取流动性池完整信息的 hook。它返回池子的核心数据：

- Token0 和 Token1 的地址
- 当前 tick
- Fee tier（手续费率）
- 流动性总量
- sqrtPriceX96（价格）

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **合约调用封装**：需要调用多个 Soroban 合约获取池子信息
2. **错误处理**：池子不存在时返回 null 并触发重试
3. **React Query 集成**：管理数据获取、重试和缓存

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
address: string | null         // 池子合约地址
```

### 输出（返回值）
```typescript
{
  data: PoolInfo | null       // 池子详细信息
  isLoading: boolean
  error: Error | null
}
```

### 核心执行逻辑

1. **查询池子信息**：调用 `getPoolInfo(address)` *(Soroban合约方法，获取池子的完整信息)* 获取链上数据
2. **null 检查**：如果返回 null，抛出错误触发重试
3. **重试机制**：最多重试 3 次，使用指数退避

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个获取流动性池详细信息的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似usePoolInfo的hook，用来获取某个区块链上流动性池的详细信息。然后明确几个关键点。第一，参数就是池子地址。第二，调用链上合约获取池子的各种信息，包括代币地址、当前tick、手续费率、流动性总量、价格等。第三，如果池子不存在就返回null。第四，用React Query管理数据，而且要支持重试机制，因为RPC调用可能会临时失败。第五，重试策略要用指数退避，就是第一次等1秒，第二次等2秒，第三次等4秒这样。

### 这里面有几个地方特别容易出错

RPC可能会临时失败，所以重试机制必须有，不然用户可能会看到莫名其妙的失败。重试要区分是暂时性错误还是池子真的不存在，所以返回null的时候还是要触发重试，而不是直接就返回了。指数退避的重试策略比较好，避免在网络忙的时候疯狂重试把系统打爆。

### 数据刷新这里有讲究

最多重试3次，3次都失败的话基本上就是真的有问题了。重试间隔用指数退避，这样比较合理。staleTime可以设成10秒，池子信息变化频率不高，但也别设太长，万一池子有大的变化用户看不到。这个hook返回的数据结构比较复杂，包含了很多字段。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { usePoolInfo } from '@sushiswap/hooks'

function PoolDetails({ poolAddress }: { poolAddress: string }) {
  const { data: poolInfo, isLoading } = usePoolInfo(poolAddress)

  if (isLoading) return <p>加载中...</p>

  return (
    <div>
      <p>Token0: {poolInfo?.token0}</p>
      <p>Token1: {poolInfo?.token1}</p>
      <p>当前 Tick: {poolInfo?.tick}</p>
      <p>手续费率: {poolInfo?.fee / 10000}%</p>
      <p>流动性: {poolInfo?.liquidity}</p>
    </div>
  )
}
```

### 常见使用场景

1. **添加流动性表单**：获取池子的基本信息
   ```tsx
   const { data: pool } = usePoolInfo(poolAddress)
   const currentPrice = pool?.sqrtPriceX96
   ```

2. **价格显示**：将 sqrtPriceX96 转换为可读价格
   ```tsx
   const price = calculatePriceFromSqrtPrice(pool?.sqrtPriceX96)
   ```

3. **Tick 范围建议**：基于当前 tick 提供默认范围
   ```tsx
   const defaultLower = pool?.tick - 1000
   const defaultUpper = pool?.tick + 1000
   ```

### Dos and Don'ts

**Dos:**
- ✅ 配合 `usePoolInitialized` *(检查池子是否已初始化的hook)* 确保池子可用
- ✅ 使用重试机制处理临时 RPC 错误
- ✅ 缓存结果因为池子信息不经常变化

**Don'ts:**
- ❌ 不要在池子未初始化时使用部分数据
- ❌ 不要忽略 null 检查，池子可能不存在
- ❌ 不要设置过短的 staleTime，池子信息不频繁变化
