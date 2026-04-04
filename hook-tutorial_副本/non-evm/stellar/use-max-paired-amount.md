> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/pool/use-max-paired-amount.ts`

# useMaxPairedAmount Hook Tutorial

## 大白话讲讲这个hook的作用

`useMaxPairedAmount` *(一个React hook，用于计算用户最大可添加流动性数量，根据用户余额和配对计算结果智能取最小值)* 是一个用于计算"最大可添加流动性数量"的 hook。它根据用户的钱包余额计算：

- 如果用户只有 Token0，最多能添加多少流动性（受限于 Token1 余额）
- 如果用户只有 Token1，最多能添加多少流动性（受限于 Token0 余额）
- 如果两种代币余额都足够，返回用户能添加的最大数量

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **多数据源组合**：需要组合用户的 Token0 余额、Token1 余额、以及 `useCalculatePairedAmount` *(计算配对代币数量的hook)* 的计算结果

2. **智能取最小值**：根据配对计算结果和用户实际余额，取能够添加的最大数量

3. **范围状态处理**：根据 `useCalculatePairedAmount` 的状态返回不同的最大数量

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
poolAddress: string | null      // 池子地址
token0Balance: string           // 用户 Token0 余额
token1Balance: string           // 用户 Token1 余额
tickLower: number | null        // 价格范围下限
tickUpper: number | null        // 价格范围上限
token0Decimals: number          // Token0 小数位数
token1Decimals: number          // Token1 小数位数
```

### 输出（返回值）
```typescript
{
  data: {
    maxToken0Amount: string   // 最大可添加的 Token0 数量
    maxToken1Amount: string   // 最大可添加的 Token1 数量
  }
}
```

### 核心执行逻辑

1. **获取配对数量**：调用 `useCalculatePairedAmount` 获取用完所有 Token0 需要多少 Token1
2. **范围状态处理**：
   - below-range：只能添加 Token0，返回全部 token0Balance
   - above-range：只能添加 Token1，返回全部 token1Balance
   - within-range：根据配对数量和余额计算最大值
3. **取最小值**：如果配对需要的 Token1 < 用户 Token1 余额，使用 token0Balance；否则按比例计算

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个计算最大可添加流动性数量的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useMaxPairedAmount的hook，用来计算用户最大可以添加多少流动性。然后明确几个关键点。第一，参数要包括池子地址、用户两种代币的余额、tick范围上下限、两种代币的小数位数。第二，依赖useCalculatePairedAmount来获取配对数量的计算结果。第三，根据配对数量和用户实际的余额，取一个能添加的最大值。第四，要处理不同状态下的情况，比如below-range时只能添加Token0、above-range时只能添加Token1、within-range时根据两者余额综合计算。第五，返回最大可添加的Token0数量和最大可添加的Token1数量。第六，用React Query管理数据。

### 这里面有几个地方特别容易出错

依赖useCalculatePairedAmount的计算结果，所以要确保那个hook的数据已经拿到了才能用这个。token0Balance和token1Balance是字符串格式的bigint，不是普通数字，传参的时候不要搞错。当两种余额都不够的时候，要按比例计算，不能简单地返回0。

### 数据刷新这里有讲究

依赖usePoolInitialized和useCalculatePairedAmount这两个hook的数据，任何一个没准备好都不能启用这个hook。价格和余额都可能快速变化，所以staleTime要设短一点，比如10秒。这个hook做的事情听起来简单，就是取最小值，但实际逻辑比较复杂，因为涉及到状态判断和比例计算。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useMaxPairedAmount } from '@sushiswap/hooks'

function MaxAmountDisplay() {
  const { data: maxAmounts } = useMaxPairedAmount({
    poolAddress: pool?.address,
    token0Balance: userToken0Balance,
    token1Balance: userToken1Balance,
    tickLower,
    tickUpper,
    token0Decimals: 6,
    token1Decimals: 6,
  })

  return (
    <div>
      <p>最大可添加 Token0: {maxAmounts?.maxToken0Amount}</p>
      <p>最大可添加 Token1: {maxAmounts?.maxToken1Amount}</p>
      <button onClick={() => setAmount(maxAmounts?.maxToken0Amount)}>
        设置为最大值
      </button>
    </div>
  )
}
```

### 常见使用场景

1. **一键最大化**：帮助用户快速设置最大可添加数量
   ```tsx
   <Button onClick={() => setAmount0(maxAmounts?.maxToken0Amount)}>
     MAX
   </Button>
   ```

2. **余额不足提示**：当用户余额不足时显示实际可添加数量
   ```tsx
   if (Number(maxAmounts?.maxToken0Amount) === 0) {
     toast.warning('Token1 余额不足，无法添加流动性')
   }
   ```

3. **范围外警告**：告知用户当前价格下的限制
   ```tsx
   // below-range 时只能添加 Token0
   // above-range 时只能添加 Token1
   ```

### Dos and Don'ts

**Dos:**
- ✅ 依赖 `useCalculatePairedAmount` 的状态处理
- ✅ 传入正确的余额数据（从 `useTokenBalances` 获取）
- ✅ 设置 `enabled` 确保所有依赖数据可用

**Don'ts:**
- ❌ 不要忽略 tick 范围状态，不同状态下的最大数量不同
- ❌ 不要直接用余额作为最大数量，要考虑配对需求
- ❌ 余额参数要是字符串格式的 bigint
