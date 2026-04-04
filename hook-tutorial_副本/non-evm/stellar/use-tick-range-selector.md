> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/tick/use-tick-range-selector.ts`

# useTickRangeSelector Hook Tutorial

## 大白话讲讲这个hook的作用

`useTickRangeSelector` *(一个React hook，用于管理流动性头寸的价格范围选择，支持动态跟随当前价格和预设范围)* 是一个用于管理价格范围选择的 hook。它帮助用户：

- 设置流动性的 tickLower 和 tickUpper
- 根据当前价格动态调整范围
- 应用预设范围或自定义范围

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **状态管理复杂**：需要管理多个相关状态（tick 值、是否动态、预设等）
2. **动态计算**：需要根据当前价格自动调整 tick 值
3. **tick 对齐**：tick 必须与 tickSpacing *(tick间距，由手续费率决定)* 对齐

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
fee: number                   // 手续费率（决定 tickSpacing）
currentPrice: number         // 当前价格
```

### 输出（返回值）
```typescript
{
  currentTick: number        // 当前 tick
  tickLower: number         // 下限 tick
  tickUpper: number         // 上限 tick
  ticksAligned: boolean     // tick 是否已对齐
  tickSpacing: number       // tick 间距
  isTickRangeValid: boolean // 范围是否有效
  defaultLower: number      // 默认下限
  defaultUpper: number      // 默认上限
  setTickLower: Dispatch    // 设置 tickLower
  setTickUpper: Dispatch    // 设置 tickUpper
  isDynamic: boolean       // 是否动态模式
  setIsDynamic: Dispatch    // 设置动态模式
  applyPresetRange: (lower: number, upper: number) => void  // 应用预设范围
  dynamicOffsets: { lower: number; upper: number } | null  // 动态偏移量
}
```

### 核心执行逻辑

1. **计算 tickSpacing**：根据 fee tier 确定 tick 间距
2. **计算当前 tick**：从当前价格计算对应的 tick 并对齐
3. **设置默认范围**：currentTick +/- 预设偏移量并对齐
4. **动态更新**：isDynamic 为 true 时，自动跟随当前价格调整
5. **对齐检查**：确保 tick 与 tickSpacing 对齐

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个管理价格范围选择的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useTickRangeSelector的hook，用来管理流动性头寸的价格范围选择。然后明确几个关键点。第一，参数是手续费率和当前价格，手续费率决定了tick的间距。第二，根据手续费率计算tickSpacing，手续费越高tickSpacing越大。第三，从当前价格计算出当前tick值。第四，提供tickLower和tickUpper两个状态，表示价格范围的下限和上限。第五，支持动态模式，就是范围跟着当前价格自动调整，用户不用手动改。第六，也支持预设范围，用户可以一键设置常用的范围。第七，确保tick与tickSpacing对齐，因为池子只认对齐后的tick值。第八，返回各种状态和设置函数，方便组件调用。

### 这里面有几个地方特别容易出错

tick必须对齐，池子只接受对齐后的tick值，用alignTick函数处理一下。动态模式下要保持与currentTick的偏移量，不然价格一变范围就乱跑了。tickLower一定要小于tickUpper，这个要校验。

### 数据刷新这里有讲究

用useState管理状态，这些状态会触发组件重渲染。用useEffect响应tickSpacing和currentPrice的变化，当这两个变了要重新计算相关值。用useRef保存偏移量，因为偏移量变了不需要触发重渲染，只是一个内部参考值。这个hook状态比较多，逻辑也比较复杂，做好了能大大简化添加流动性表单的代码。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useTickRangeSelector, usePoolPrice } from '@sushiswap/hooks'

function RangeSelector({ pool }: { pool: PoolInfo }) {
  const { data: currentPrice } = usePoolPrice(pool.address)

  const {
    tickLower,
    tickUpper,
    tickSpacing,
    isDynamic,
    setIsDynamic,
    setTickLower,
    setTickUpper,
    applyPresetRange,
  } = useTickRangeSelector({
    fee: pool.fee,
    currentPrice,
  })

  return (
    <div>
      <div>
        <label>
          <input
            type="checkbox"
            checked={isDynamic}
            onChange={(e) => setIsDynamic(e.target.checked)}
          />
          动态跟随价格
        </label>
      </div>

      <div>
        <p>Tick 间距: {tickSpacing}</p>
        <p>当前 Tick: {Math.floor(currentTick)}</p>
      </div>

      <RangeInputs
        tickLower={tickLower}
        tickUpper={tickUpper}
        onTickLowerChange={setTickLower}
        onTickUpperChange={setTickUpper}
      />

      <div>
        <button onClick={() => applyPresetRange(-1000, 1000)}>
          ±1000 范围
        </button>
        <button onClick={() => applyPresetRange(-5000, 5000)}>
          ±5000 范围
        </button>
      </div>
    </div>
  )
}
```

### 常见使用场景

1. **添加流动性表单**：用户选择价格范围
   ```tsx
   const { tickLower, tickUpper } = useTickRangeSelector({ fee, currentPrice })
   // 传给添加流动性 hook
   addLiquidity({ tickLower, tickUpper, ... })
   ```

2. **动态范围模式**：自动跟随价格调整范围
   ```tsx
   const [isDynamic, setIsDynamic] = useState(false)
   const { tickLower, tickUpper } = useTickRangeSelector({
     fee,
     currentPrice,
   })

   useEffect(() => {
     if (isDynamic) {
       // 自动更新范围
     }
   }, [currentPrice, isDynamic])
   ```

3. **预设范围按钮**：一键设置常用范围
   ```tsx
   const presets = [
     { label: '保守', lower: -500, upper: 500 },
     { label: '适中', lower: -2500, upper: 2500 },
     { label: '激进', lower: -10000, upper: 10000 },
   ]
   ```

### Dos and Don'ts

**Dos:**
- ✅ 配合 `usePoolPrice` *(获取池子当前价格的hook)* 获取当前价格
- ✅ tick 必须与 tickSpacing 对齐后使用
- ✅ 使用预设范围提升用户体验

**Don'ts:**
- ❌ 不要设置 tickLower >= tickUpper 的范围
- ❌ 不要忽略 tickSpacing 变化
- ❌ 不要在动态模式下让用户手动修改范围（会被覆盖）
