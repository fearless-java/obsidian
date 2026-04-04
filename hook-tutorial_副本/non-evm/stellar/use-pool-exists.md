> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/factory/use-pool-exists.ts`

# usePoolExists Hook Tutorial

## 大白话讲讲这个hook的作用

`usePoolExists` *(一个React hook，用于检查池子是否已存在，通过代币对和手续费率查询)* 是一个用于检查池子是否已存在的 hook。它通过：

- Token A 和 Token B 地址
- Fee tier（手续费率）

检查对应的池子是否已创建。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **前置检查**：添加流动性前需要确认池子存在
2. **工厂合约调用**：封装 `poolExists` *(工厂合约方法，检查池子是否已创建)* 函数

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
  data: boolean                   // true = 存在，false = 不存在
}
```

### 核心执行逻辑

1. **调用合约**：使用 `poolExists(params)` *(工厂合约方法，检查指定代币对和手续费的池子是否已创建)* 检查
2. **返回布尔值**：池子是否存在

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个检查池子是否存在的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似usePoolExists的hook，用来检查某个区块链上的池子是否存在。然后明确几个关键点。第一，参数包括代币A地址、代币B地址和手续费率。第二，调用工厂合约的poolExists函数来检查。第三，返回布尔值，true表示池子存在，false表示不存在。第四，用React Query管理数据。

### 这里面有几个地方特别容易出错

参数必须完整才能查询，任何一个是null都不能发起请求，无效的参数查了也是白查。这个hook看起来简单，但它是很多操作的前置条件，比如添加流动性前要先确认池子存在。

### 数据刷新这里有讲究

enabled条件要设成参数都不为null才行。这个hook返回的数据变化不频繁，池子创建和存在状态不会每秒都在变，所以不需要频繁刷新。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { usePoolExists } from '@sushiswap/hooks'

function AddLiquidityButton({ tokenA, tokenB, fee }: Props) {
  const { data: exists, isLoading } = usePoolExists({ tokenA, tokenB, fee })

  if (isLoading) return <Button disabled>检查中...</Button>

  return exists ? (
    <Button>添加流动性</Button>
  ) : (
    <Button>创建池子</Button>
  )
}
```

### 常见使用场景

1. **添加流动性前置检查**：确认池子存在再显示添加按钮
   ```tsx
   const { data: poolExists } = usePoolExists({ tokenA, tokenB, fee })

   return poolExists ? (
     <AddLiquidityForm />
   ) : (
     <CreatePoolPrompt />
   )
   ```

2. **Swap 前置检查**：Swap 前确认池子存在
   ```tsx
   const { data: canSwap } = usePoolExists({ tokenA, tokenB, fee })

   return canSwap ? (
     <SwapForm />
   ) : (
     <Message type="warning">该交易对暂无池子</Message>
   )
   ```

3. **条件渲染**：根据池子是否存在显示不同 UI
   ```tsx
   {poolExists ? (
     <>
       <PriceDisplay />
       <ActionButton />
     </>
   ) : (
     <CreatePoolCard />
   )}
   ```

### Dos and Don'ts

**Dos:**
- ✅ 配合 `useGetPool` *(获取池子地址的hook)* 使用
- ✅ 代币地址顺序要一致（tokenA < tokenB）
- ✅ 检查 enabled 条件

**Don'ts:**
- ❌ 不要忽略参数完整性检查
- ❌ 不要假设不同 fee tier 的池子互通
- ❌ 不要在参数为空时发起查询
