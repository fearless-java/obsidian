> 源代码路径: `apps/web/src/lib/hooks/useSlippageTolerance.ts`

# useSlippageTolerance

## 1. 大白话讲讲这个hook的作用

`useSlippageTolerance` *(一个React hook，用于管理交易滑点设置，基于useLocalStorage实现持久化存储)* 是一个管理交易滑点设置的hook。简单来说，它帮你存储和读取用户设定的"能接受的最高滑点"。

滑点是什么？就是当你下单时，预期得到的价格和实际成交价格之间的差别。比如你想定价买入1 ETH，价格是2000 USDT，但实际成交可能是1998 USDT，这个2 USDT的差价就是滑点。

这个hook做了两件事：
- 读取用户之前保存的滑点设置（默认是0.5%）
- 提供一个 setter 函数，让用户可以修改滑点设置

## 2. 讲讲为什么需要封装该hook

直接使用 localStorage *(浏览器本地存储API，用于持久化保存键值对数据)* 存取滑点值会导致的问题：
- 没有类型安全，存取的都是原始字符串或数字
- 没有默认值处理，第一次使用可能是 undefined
- 业务逻辑（百分比转换、AUTO选项处理）散落在组件各处
- 无法统一管理不同的滑点key（swap、addLiquidity等不同场景用不同的key）

封装后：
- 返回的是标准的 `Percent` *(SushiSwap中的百分比类型，内部使用basis points（分母10000）来表示，如0.5% = new Percent({ numerator: 50, denominator: 10000 }))* 对象，可以直接用于各种计算
- 统一处理 "AUTO" 这种特殊值
- 通过 key 参数支持不同场景的独立配置
- 与 localStorage 解耦，便于测试和替换存储方式

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
- `key?: SlippageToleranceStorageKey` *(存储键的类型，用于区分不同场景的滑点设置，如'swap'、'addLiquidity'等)* - 可选，指定存储的key，默认是Swap场景

**输出：**
```typescript
[
  Percent,  // 转换后的滑点值，如 0.5% = new Percent({ numerator: 50, denominator: 10000 })
  {
    slippageTolerance: number | string,  // 原始存储值
    setSlippageTolerance: (value: number | string) => void  // 设置函数
  }
]
```

**执行逻辑：**
1. 调用 `useLocalStorage` *(一个React hook，封装了localStorage的读取和写入，支持任意类型的值自动序列化/反序列化)* 读取指定key的值（默认 `DEFAULT_SLIPPAGE` = 0.5）
2. 如果值是字符串 "AUTO"，使用默认滑点
3. 将数值乘以100，转成 `Percent` 对象（分母为10000，即basis points）
4. 返回 [Percent对象, {原始值, setter}]

**数据流：**
```
localStorage[key] ---> useLocalStorage --> 数值处理 --> Percent对象 --> 组件使用
用户修改 --> setSlippageTolerance --> useLocalStorage更新 --> localStorage[key]
```

## 4. 怎么给这个hook写AI提示词

这个hook用来管理用户的滑点设置。滑点就是交易时能接受的最高价格偏差，比如你设定0.5%，那实际成交价格比预期最多差0.5%。用户的设置会保存到localStorage，刷新页面也不会丢。

### 写提示词的小技巧

**第一，计算要用Percent对象，显示用原始值。** Percent对象是给人算账用的（做数学运算），但UI上显示的时候要给用户看"0.5"这种易懂的值，不是"50"（basis points）。

**第二，AUTO选项要特殊处理。** AUTO是智能滑点，会根据市场情况自动调整。代码里要能识别这个特殊值，不能把它当成普通数字处理。

**第三，不同场景用不同key。** swap（交易）和addLiquidity（加流动性）用的是不同的滑点设置，存的时候要分开存，用不同的key。

### 写提示词时要注意的条条框框

**滑点值有范围限制。** 一般是0.01%到50%之间。太低容易交易失败，太高容易被攻击。

**Percent内部用basis points。** 分母是10000，所以50就代表0.5%（50/10000）。numerator最大能到5000，也就是50%。

**localStorage要同步更新。** 用户改了设置，localStorage里存的那个值也要跟着改，不然刷新页面就丢了。

### 提示词模板

```
帮我写一个React hook，功能是管理用户的滑点设置。

具体需求：
1. 用 localStorage 做持久化存储
2. 支持不同场景用不同的key，比如swap和addLiquidity用不同的key
3. 把用户输入的数字（比如0.5）转成Percent对象（分母是10000）
4. 支持"AUTO"这个特殊值，代表智能滑点
5. 提供setter函数让用户能改设置
6. 默认滑点是0.5%

类型定义：
- 输入：key可选，默认'swap'
- 输出：[Percent, {slippageTolerance, setSlippageTolerance}]

注意事项：
- 滑点范围是 0.01% 到 50%
- Percent对象可以直接用在SushiSwap的交易计算里
```

### 实际用的例子

```typescript
// 使用示例
const [slippage, { slippageTolerance, setSlippageTolerance }] = useSlippageTolerance()

// 在交易中使用
const minAmountOut = amountOut.mul(new Percent(1).sub(slippage))

// UI显示
<input
  value={slippageTolerance}
  onChange={(e) => setSlippageTolerance(e.target.value)}
/>
```

这里 slippage 是 Percent 对象用来算数学，slippageTolerance 是给用户看和让用户输入的原始值。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 基础调用，获取当前滑点值和设置函数
const [slippage, { slippageTolerance, setSlippageTolerance }] = useSlippageTolerance()

// 指定场景key
const [slippage, { slippageTolerance, setSlippageTolerance }] = useSlippageTolerance('addLiquidity')
```

### 常见使用场景

**场景1：在交易组件中使用滑点计算最小输出金额**

```typescript
const [slippage, { slippageTolerance }] = useSlippageTolerance()

// 计算最小输出金额（考虑滑点）
const minAmountOut = amountOut.mul(new Percent(10000).sub(slippage))
// slippage是Percent对象，直接用于数学运算
```

**场景2：UI中显示和修改滑点**

```typescript
const [slippage, { slippageTolerance, setSlippageTolerance }] = useSlippageTolerance()

// 显示
<span>滑点: {slippageTolerance}%</span>

// 修改（用户输入）
<Input
  value={slippageTolerance}
  onChange={(e) => setSlippageTolerance(e.target.value)}
/>
```

**场景3：处理AUTO选项**

```typescript
const [slippage, { slippageTolerance, setSlippageTolerance }] = useSlippageTolerance()

// AUTO选项会让合约自动计算最优滑点
const handleAutoSlippage = () => {
  setSlippageTolerance('AUTO')
}
```

### Dos and Don'ts

**✅ Do:**
- 使用返回的 `Percent` 对象进行所有计算，不要直接用原始数值
- 在UI显示时使用 `slippageTolerance`（用户友好的格式）
- 始终设置合理的滑点范围验证（如0.01% - 50%）
- 为不同场景使用不同的key，保持配置独立性

**❌ Don't:**
- 不要直接使用 localStorage 读取滑点，应该用hook提供的接口
- 不要在计算中用 `slippageTolerance` 字符串直接运算，要用 `Percent` 对象
- 不要忽略AUTO选项的处理，用户可能选择智能滑点
- 不要设置过大的滑点（如>50%），这会导致用户资金风险
