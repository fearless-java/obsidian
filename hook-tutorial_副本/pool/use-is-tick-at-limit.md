> 源代码路径: `apps/web/src/lib/pool/v3/use-is-tick-at-limit.ts`

# useIsTickAtLimit Tutorial

## 1. 大白话讲讲这个hook的作用

`useIsTickAtLimit` *(一个React hook，用于判断Uniswap V3价格区间是否达到合约允许的最小/最大tick)* 是一个判断用户设置的价格区间是否达到合约允许的最大范围的hook。

简单来说：
- 检查用户输入的tickLower（lower price bound） *(用户设置的价格区间下限tick)* 是否等于该池子的最小可设置tick
- 检查用户输入的tickUpper（upper price bound） *(用户设置的价格区间上限tick)* 是否等于该池子的最大可设置tick
- 返回 `{ [Bound.LOWER]: boolean | undefined, [Bound.UPPER]: boolean | undefined }`

这有什么用呢？当用户想把流动性覆盖整个价格范围时，UI需要知道这一点，从而禁用某些按钮或提示用户"已覆盖全部范围"。

## 2. 讲讲为什么需要封装该hook

Uniswap V3的tick范围是有限制的，不是想设多少就设多少：

1. **合约限制**：TickMath *(Uniswap V3的数学工具库，定义了tick相关的数学计算)* 定义了MIN_TICK和MAX_TICK
2. **tickSpacing影响**：不同fee amount的池子有不同的tick间距（tickSpacing）*(tick间距决定了价格的最小跳动单位，不同费率池有不同的间距)*，所以最小/最大可用tick也不同
3. **nearestUsableTick转换**：合约层面会把你输入的tick rounding到最近的有效tick

直接比较 `tickLower === TickMath.MIN_TICK` 是不对的，因为要考虑tickSpacing的转换。

封装成hook后：
- 自动处理nearestUsableTick *(一个函数，用于将tick转换为给定tickSpacing下的有效tick)* 的转换
- 只需传入原始的tickLower/tickUpper和feeAmount
- 返回简单易懂的boolean值

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  feeAmount: SushiSwapV3FeeAmount | undefined    // 费率
  tickLower: number | undefined                  // 用户设置的lower tick
  tickUpper: number | undefined                  // 用户设置的upper tick
}
```

### 输出（Return Value）
```typescript
{
  [Bound.LOWER]: boolean | undefined   // lower bound是否达到最小tick
  [Bound.UPPER]: boolean | undefined   // upper bound是否达到最大tick
}
```

### 执行逻辑
1. 如果 `feeAmount` 未定义，返回 `undefined`
2. 如果 `tickLower` 已定义：
   - 计算 `nearestUsableTick(TickMath.MIN_TICK, TICK_SPACINGS[feeAmount])`
   - 比较 `tickLower === 计算结果`
3. 如果 `tickUpper` 已定义：
   - 计算 `nearestUsableTick(TickMath.MAX_TICK, TICK_SPACINGS[feeAmount])`
   - 比较 `tickUpper === 计算结果`
4. 返回包含 [Bound.LOWER] 和 [Bound.UPPER] 的对象

### TickMath 常量
```typescript
TickMath.MIN_TICK = -887272  // -200 * 60 * 60 * 60 / 2 (约)
TickMath.MAX_TICK = 887272

TICK_SPACINGS = {
  100: 200,
  500: 10,
  3000: 60,
  10000: 200,
}
```

### 实际计算示例
假设 feeAmount = 3000（0.3%费率），tickSpacing = 60：
- MIN_TICK = -887272
- nearestUsableTick(MIN_TICK, 60) = -887160
- MAX_TICK = 887272
- nearestUsableTick(MAX_TICK, 60) = 887160

如果用户设置的tick范围是 [-887160, 887160]，则返回 `{ [Bound.LOWER]: true, [Bound.UPPER]: true }`

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于判断Uniswap V3集中流动性池的价格区间是否达到了合约允许的最小或最大tick。当用户设置的tick下限等于最小可用tick，或者tick上限等于最大可用tick时，就返回true。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，使用wagmi或viem来获取TickMath相关的常量。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数包括费率数量、下限tick和上限tick这三个数字类型的值（都可能是undefined）。输出应该是一个对象，包含下限边界是否达到极限和上限边界是否达到极限这两个属性，值是boolean或者undefined。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先从相关库导入TickMath的最小和最大tick常量，导入费率到tick间距的映射表，导入nearestUsableTick函数。然后用useMemo包裹整个计算逻辑。对于下限边界的判断，如果费率或下限tick是undefined，就返回undefined；否则计算在给定费率下最小tick经过nearestUsableTick转换后的值，然后比较用户输入的下限tick是否等于这个计算结果。对于上限边界也是类似的逻辑，只是用的是最大tick。最后返回包含两个边界判断结果的对象。

### 需要特别注意的约束条件

**理解MIN_TICK和MAX_TICK的来源**

TickMath合约定义了最小的tick是-887272，最大的tick是887272。这些是Uniswap V3合约的常量，不能修改或随意设置。

**费率与tick间距的对应关系**

不同的费率对应不同的tick间距。具体来说，费率100对应tick间距200，费率500对应10，费率3000对应60，费率10000对应200。这个映射关系需要正确使用。

**nearestUsableTick的重要性**

用户输入的tick不能直接与MIN_TICK或MAX_TICK比较，因为还要考虑tick间距的转换。必须先用nearestUsableTick函数把MIN_TICK或MAX_TICK转换成在给定tick间距下的有效tick值，然后再与用户输入的tick比较。

**undefined和false的含义完全不同**

这是一个非常容易出错的地方。当参数未定义（undefined）时，返回值也应该是undefined，而不是false。false表示参数都定义了但是没有达到边界，而undefined表示无法判断（比如因为参数缺失）。这两个状态的含义完全不同，UI层需要区别处理。

**这是一个纯计算hook**

这个hook不需要进行任何数据获取、不需要调用合约、不需要useQuery或任何异步操作。它只是一个同步的计算逻辑，所有的计算都应该在useMemo中进行。

### 状态管理的要点

**纯同步hook不需要状态管理**

这是一个纯同步的hook，不涉及任何异步操作，所以不需要管理loading或error状态。所有的状态其实就是输入参数和计算结果。

**使用useMemo包裹计算逻辑**

为了避免不必要的重算，整个计算逻辑都应该用useMemo包裹。依赖数组包含feeAmount、tickLower和tickUpper这三个参数。只有当这些参数变化时，才会重新计算。

**计算结果应该是稳定的**

由于是纯计算逻辑，只要输入参数不变，计算结果就应该保持不变。这样的行为可以让UI层更容易处理，避免不必要的渲染。

**Bound枚举的使用**

Bound.LOWER和Bound.UPPER是枚举常量，定义在constants文件中。返回的对象应该使用这些枚举作为key，而不是直接使用字符串'LOWER'和'UPPER'。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于判断Uniswap V3集中流动性池的价格区间是否达到了合约允许的最小或最大tick。

这个hook需要返回每个边界是否达到极限的状态。

需要使用React和TypeScript，使用wagmi或viem来获取TickMath相关的常量。

输入参数包括费率数量、下限tick和上限tick（都可能是undefined）。输出应该是一个对象，包含两个属性：下限边界是否达到极限和上限边界是否达到极限。

实现步骤：
1. 导入TickMath的最小tick(-887272)和最大tick(887272)
2. 导入费率到tick间距的映射表
3. 导入nearestUsableTick函数
4. 用useMemo包裹整个计算逻辑
5. 对于下限边界：如果费率或下限tick是undefined就返回undefined，否则计算nearestUsableTick(MIN_TICK, 对应tick间距)，然后比较用户输入是否等于这个值
6. 对于上限边界：同样的逻辑，只是用的是MAX_TICK
7. 返回包含两个判断结果的对象

重要约束：
- MIN_TICK和MAX_TICK是Uniswap V3 TickMath合约的常量，不能修改
- tick间距必须根据费率正确映射
- 当参数为undefined时必须返回undefined，而不是false
- 这是一个纯计算hook，不涉及任何异步操作或合约调用
- 使用useMemo优化性能，避免不必要的重算

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useIsTickAtLimit, Bound } from '@sushiswap/hooks'
import { useRange hooks } from '@sushiswap/ui'

function PriceRangeSelector() {
  const { tickLower, tickUpper, feeAmount } = useRange()

  const isAtLimit = useIsTickAtLimit({
    feeAmount,
    tickLower,
    tickUpper,
  })

  return (
    <div>
      {isAtLimit[Bound.LOWER] && (
        <span>已选择最低价格范围</span>
      )}
      {isAtLimit[Bound.UPPER] && (
        <span>已选择最高价格范围</span>
      )}
      {!isAtLimit[Bound.LOWER] && !isAtLimit[Bound.UPPER] && (
        <span>选择自定义价格范围</span>
      )}
    </div>
  )
}
```

### 常见使用场景

1. **无限范围按钮状态**
   - 当用户点击"Max"按钮设置全范围时，检测并提示
   - 禁用重复的"设置全范围"按钮

2. **价格范围覆盖提示**
   - 告诉用户当前选择是否覆盖了全部价格范围
   - 帮助用户理解他们的流动性覆盖范围

3. **条件渲染UI元素**
   - 根据是否达到边界来显示/隐藏某些选项
   - 调整价格范围选择器的可用性

### Dos and Don'ts

**Dos：**
- ✅ 使用useMemo包裹计算逻辑，避免不必要的重算
- ✅ 正确处理undefined情况，undefined表示"无法判断"
- ✅ 在feeAmount未定义时优雅地返回undefined
- ✅ 配合Bound枚举使用，保持代码可读性

**Don'ts：**
- ❌ 不要直接比较tickLower === MIN_TICK，这没有考虑tickSpacing
- ❌ 不要忽略undefined的含义，undefined表示参数不完整
- ❌ 不要在这个hook中使用任何异步操作，它是一个纯计算hook
- ❌ 不要在渲染时直接使用返回的undefined进行比较，应该先检查
