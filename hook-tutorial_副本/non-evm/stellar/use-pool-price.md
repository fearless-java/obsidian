> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/pool/use-pool-price.ts`

# usePoolPrice Hook Tutorial

## 大白话讲讲这个hook的作用

`usePoolPrice` *(一个React hook，用于获取池子的当前价格，将sqrtPriceX96转换为人类可读的Token0/Token1价格)* 是一个用于获取池子当前价格的 hook。它返回池子中 Token0/Token1 的即时价格（以 Token1/Token0 计价）。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **合约调用封装**：需要调用 `getCurrentSqrtPrice` *(获取池子当前价格)* 和 `calculatePriceFromSqrtPrice` *(将sqrtPriceX96转换为人类可读价格)*
2. **React Query 集成**：管理数据获取和缓存

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
address: string | null         // 池子地址
```

### 输出（返回值）
```typescript
{
  data: number | null          // 当前价格
}
```

### 核心执行逻辑

1. **获取 sqrtPrice**：调用 `getCurrentSqrtPrice(address)` *(Soroban合约方法，获取池子的当前价格)*
2. **计算价格**：使用 `calculatePriceFromSqrtPrice(sqrtPriceX96)` *(数学函数，将价格转换为Token0/Token1格式)* 转换为人类可读价格
3. **返回结果**：返回 Token0/Token1 的价格

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个获取池子当前价格的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似usePoolPrice的hook，用来获取某个区块链上池子的当前价格。然后明确几个关键点。第一，参数就是池子地址。第二，调用链上合约获取sqrtPrice，这是价格的一种数学表示方式。第三，把sqrtPrice转换成人类能看懂的价格格式。第四，返回价格数值。第五，用React Query管理数据。

### 这里面有几个地方特别容易出错

价格随时都在变，所以staleTime要设短一点，不然显示的可能已经是旧价格了。要明确价格的方向，是token0除以token1还是反过来，这两个是不同的，通常池子会有一个固定的表示方式。

### 数据刷新这里有讲究

价格变化很频繁，staleTime设成10秒比较合适，设太长用户看到的价格就不准了。这个hook虽然简单，但价格是交易的核心数据，准确性和及时性都很重要。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { usePoolPrice } from '@sushiswap/hooks'
import { formatPrice } from '@sushiswap/utilities'

function PoolPriceDisplay({ poolAddress }: { poolAddress: string }) {
  const { data: price } = usePoolPrice(poolAddress)

  return (
    <div>
      <p>当前价格: {price ? formatPrice(price, 6) : '0'}</p>
      <p>1 Token0 = {price} Token1</p>
    </div>
  )
}
```

### 常见使用场景

1. **Swap 界面价格显示**：显示当前汇率
   ```tsx
   const { data: price } = usePoolPrice(poolAddress)
   const token0Out = amountIn * price
   ```

2. **Tick 范围选择参考**：基于当前价格推荐 tick 范围
   ```tsx
   const currentTick = priceToTick(price)
   const recommendedLower = currentTick - 1000
   const recommendedUpper = currentTick + 1000
   ```

3. **价格变化提示**：实时监控价格变动
   ```tsx
   const [price, setPrice] = useState()
   const { data: newPrice } = usePoolPrice(poolAddress)

   useEffect(() => {
     if (newPrice !== price) {
       setPrice(newPrice)
       // 价格变化超过阈值时提示
     }
   }, [newPrice])
   ```

### Dos and Don'ts

**Dos:**
- ✅ 使用 `formatPrice` 格式化价格显示
- ✅ 设置较短的 `staleTime` 因为价格随时变化
- ✅ 明确价格的方向（token0/token1）

**Don'ts:**
- ❌ 不要忽略 null 检查，池子可能不存在
- ❌ 不要直接显示原始数字，使用格式化函数
- ❌ 不要假设价格不变，始终视为可变的
