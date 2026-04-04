> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/pool/use-calculate-dependent-amount.ts`

# useCalculateDependentAmount Hook Tutorial

## 大白话讲讲这个hook的作用

`useCalculateDependentAmount` *(一个React hook，用于计算流动性池中配对代币的数量，当用户输入一种代币数量时计算需要投入的另一种代币数量)* 是一个用于计算流动性池中"配对代币数量"的 hook。当你往一个集中流动性池添加资产时：

- 如果你输入了 Token0 的数量，系统需要计算你需要投入多少 Token1
- 如果你输入了 Token1 的数量，系统需要计算你需要投入多少 Token0

这个 hook 根据当前池子的价格和你的 tick 范围，计算出配对的代币数量。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **复杂的数学计算**：需要调用多个 Soroban 合约函数（`getCurrentSqrtPrice` *(获取池子当前价格)*、tickToSqrtPrice *(将tick转换为价格)*、calculateLiquidityFromAmount0/1 等）

2. **价格范围检查**：计算时需要判断当前价格是否在指定的 tick 范围内

3. **多种状态处理**：返回 idle、below-range、above-range、within-range、error 等状态

4. **React Query 缓存**：计算结果可以缓存一段时间避免重复计算

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
poolAddress: string | null      // 池子地址
amount: string                  // 输入的代币数量（人类可读格式）
independentField: 'token0' | 'token1'  // 用户输入的是哪个代币
tickLower: number               // 价格范围下限 tick
tickUpper: number               // 价格范围上限 tick
independentDecimals: number     // 输入代币的小数位数
dependentDecimals: number       // 配对代币的小数位数
independentTokenCode?: string   // 输入代币的代码（用于错误提示）
dependentTokenCode?: string     // 配对代币的代码（用于错误提示）
```

### 输出（返回值）
```typescript
{
  data: {
    amount: string                    // 配对代币数量（人类可读格式）
    status: 'idle' | 'below-range' | 'above-range' | 'within-range' | 'error'
    error?: string                    // 错误信息
  }
}
```

### 核心执行逻辑

1. **获取当前价格**：调用 `getCurrentSqrtPrice(poolAddress)` *(Soroban合约方法，获取池子当前价格)* 获取池子当前 sqrtPriceX96
2. **计算边界价格**：使用 `tickToSqrtPrice()` *(将tick索引转换为对应的价格)* 计算 tickLower 和 tickUpper 对应的价格
3. **范围检查**：
   - 如果当前价格 < tickLower 价格区间：below-range
   - 如果当前价格 >= tickUpper 价格区间：above-range
4. **计算配对数量**：根据输入的代币数量和 tick 范围，计算需要的配对代币数量
5. **格式化返回**：将结果从最小单位转换为人类可读格式

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个计算配对代币数量的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useCalculateDependentAmount的hook，用来计算流动性池中配对的代币数量。然后明确几个关键点。第一，参数要包括池子地址、输入的代币数量、用户输入的是哪个代币、tick范围上下限、输入代币的小数位数、输出代币的小数位数，还可以加上代币代码用于错误提示。第二，调用链上合约获取当前价格，然后计算需要配对的代币数量。第三，要处理价格范围外的边界情况，比如当前价格低于选择的范围上限或者高于选择的下限。第四，返回配对的代币数量和一个状态标识，状态包括idle表示还没开始算、below-range表示价格太低、above-range表示价格太高、within-range表示正常、error表示出错了。第五，数量用人类能看懂的格式，不是最小单位。第六，用React Query缓存计算结果。第七，错误处理要做好。

### 这里面有几个地方特别容易出错

这个hook依赖于池子初始化状态，池子没初始化的话算出来的数据是不准确的。输入和输出的代币可能有不同的小数位数，这个要分别处理，不能混为一谈。idle状态和正常计算的状态要区分开，idle是说参数还不完整不能算，within-range才是真的算出来了。价格随时都在变，所以staleTime要设短一点，比如10秒，超过10秒就要重新算了。

### 数据刷新这里有讲究

staleTime设成10秒比较合理，因为价格变化比较频繁。enabled条件要同时满足池子地址存在且池子已经初始化，两个条件缺一不可。这个hook依赖usePoolInitialized的数据，所以要确保那个hook的数据已经准备好了才能用这个。数据流是：先查池子初始化状态 -> 再查当前价格 -> 最后算配对数量，每一步都不能少。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useCalculateDependentAmount } from '@sushiswap/hooks'

function AddLiquidityForm() {
  const [token0Amount, setToken0Amount] = useState('100')
  const [tickLower, setTickLower] = useState(-1000)
  const [tickUpper, setTickUpper] = useState(1000)

  const { data, isLoading } = useCalculateDependentAmount({
    poolAddress: pool?.address,
    amount: token0Amount,
    independentField: 'token0',
    tickLower,
    tickUpper,
    independentDecimals: token0.decimals,
    dependentDecimals: token1.decimals,
    independentTokenCode: token0.symbol,
    dependentTokenCode: token1.symbol,
  })

  return (
    <div>
      <input value={token0Amount} onChange={(e) => setToken0Amount(e.target.value)} />
      <p>需要 Token1 数量: {data?.amount}</p>
      <p>状态: {data?.status}</p>
    </div>
  )
}
```

### 常见使用场景

1. **添加流动性表单**：显示配对代币数量
   ```tsx
   const { data: dependentAmount } = useCalculateDependentAmount({
     poolAddress,
    amount: inputAmount,
    independentField: inputToken === 'token0' ? 'token0' : 'token1',
     tickLower,
     tickUpper,
     // ...
   })
   ```

2. **价格范围外提示**：告诉用户当前价格不在选择范围内
   ```tsx
   if (data?.status === 'below-range') {
     toast.warning('当前价格低于您的范围下限')
   } else if (data?.status === 'above-range') {
     toast.warning('当前价格高于您的范围上限')
   }
   ```

3. **余额检查**：检查用户是否有足够的配对代币
   ```tsx
   const hasEnoughToken1 = BigInt(dependentAmount?.amount ?? 0) <= token1Balance
   ```

### Dos and Don'ts

**Dos:**
- ✅ 配合 `useMaxPairedAmount` *(计算最大可添加流动性的hook)* 使用，获取用户最大可添加数量
- ✅ 处理所有状态（idle/below-range/above-range/within-range/error）
- ✅ 设置合理的 `staleTime` 因为价格随时变化

**Don'ts:**
- ❌ 不要在池子未初始化时调用此 hook
- ❌ 不要忽略 decimals 差异，直接比较数量
- ❌ 不要假设 status 总是 within-range，要处理边界情况
