> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/pool/use-calculate-paired-amount.ts`

# useCalculatePairedAmount Hook Tutorial

## 大白话讲讲这个hook的作用

`useCalculatePairedAmount` *(一个React hook，专门用于从Token0输入计算Token1配对数量，是useCalculateDependentAmount的特化版本)* 是专门用于计算"基于 Token0 输入计算 Token1 配对数量"的 hook。它是 `useCalculateDependentAmount` *(计算配对代币数量的通用hook，支持token0或token1输入)* 的一个特化版本。

当你确定要投入的 Token0 数量后，这个 hook 计算在当前价格和指定 tick 范围内需要投入多少 Token1。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **简化常见场景**：swap 和添加流动性时经常需要从 Token0 计算 Token1
2. **特定业务逻辑**：针对 Token0 输入的场景优化
3. **范围检查**：返回 below-range、above-range 等状态帮助 UI 显示

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
poolAddress: string | null      // 池子地址
token0Amount: string            // Token0 输入数量
tickLower: number | null        // 价格范围下限
tickUpper: number | null        // 价格范围上限
decimals: number | null         // 代币小数位数
token0Code?: string             // 代币代码（用于错误提示）
```

### 输出（返回值）
```typescript
{
  data: {
    token1Amount: string         // 配对的 Token1 数量
    status: 'idle' | 'below-range' | 'above-range' | 'within-range' | 'error'
    error?: string
  }
}
```

### 核心执行逻辑

1. **参数校验**：确保所有必要参数有效
2. **获取当前价格**：调用 `getCurrentSqrtPrice(poolAddress)` *(Soroban合约方法，获取池子当前价格)*
3. **范围判断**：根据当前价格与 tick 范围的关系返回状态
4. **计算配对**：如果价格在范围内，计算需要的 Token1 数量
5. **格式化**：将 bigint 转为人类可读格式

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个从Token0数量计算Token1配对数量的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useCalculatePairedAmount的hook，用来从Token0的数量计算出需要配对的Token1数量。然后明确几个关键点。第一，参数要包括池子地址、Token0的数量、tick范围上下限、decimals，可能还要加上Token0的代码用于错误提示。第二，先检查当前价格是不是在用户选择的tick范围内。第三，如果价格在范围内，就算出需要多少Token1。第四，返回配对的Token1数量和状态，状态有idle表示还没开始算、below-range表示价格太低、above-range表示价格太高、within-range表示正常、error表示出错了。第五，用React Query缓存结果。

### 这里面有几个地方特别容易出错

这个hook是专门针对用户输入Token0的场景，别把它用于反向计算从Token1算Token0。依赖池子初始化状态，池子没初始化的话数据不准。要假设token0和token1有相同的decimals，虽然这不是绝对的，但这个hook是这样设计的。

### 数据刷新这里有讲究

依赖usePoolInitialized确保池子可用，池子没初始化的话不能算。价格随时都在变，所以staleTime要设短一点，比如10秒，超过10秒就要重新算了。这个hook本质上是从通用版本特化出来的，专注于Token0输入这个常见场景，所以使用起来会更简单一点。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useCalculatePairedAmount } from '@sushiswap/hooks'

function LiquidityPanel() {
  const [token0Amount, setToken0Amount] = useState('100')
  const [tickLower, setTickLower] = useState(-1000)
  const [tickUpper, setTickUpper] = useState(1000)

  const { data: pairedAmount } = useCalculatePairedAmount({
    poolAddress: pool?.address,
    token0Amount,
    tickLower,
    tickUpper,
    decimals: 6,
    token0Code: 'USDC',
  })

  return (
    <div>
      <input
        type="number"
        value={token0Amount}
        onChange={(e) => setToken0Amount(e.target.value)}
        placeholder="输入 USDC 数量"
      />
      <p>需要 USDC.e 数量: {pairedAmount?.token1Amount ?? '0'}</p>
      <RangeStatus status={pairedAmount?.status} />
    </div>
  )
}
```

### 常见使用场景

1. **添加流动性预览**：显示添加 Token0 时需要搭配的 Token1 数量
   ```tsx
   const { data: paired } = useCalculatePairedAmount({
     poolAddress: selectedPool.address,
     token0Amount: amount0,
     tickLower,
     tickUpper,
     decimals,
   })
   ```

2. **范围状态提示**：帮助用户理解为什么配对数量为 0
   ```tsx
   if (paired?.status === 'below-range') {
     message.info('当前价格低于范围下限，只能添加 Token0')
   }
   ```

3. **Swap 金额计算**：计算需要多少 Token1 来 swap
   ```tsx
   const { data: token1Needed } = useCalculatePairedAmount({
     // 用于 swap 场景
   })
   ```

### Dos and Don'ts

**Dos:**
- ✅ 假设输入的是 Token0，计算 Token1 输出
- ✅ 检查池子初始化状态后再调用
- ✅ 处理所有状态类型，特别是 below-range 和 above-range

**Don'ts:**
- ❌ 不要用于反向计算（从 Token1 计算 Token0），使用 useCalculateDependentAmount
- ❌ 不要忽略 decimals 参数
- ❌ 不要假设池子一定存在
