> 源代码路径: `packages/hooks/src/useSlippageTolerance.ts`

# useSlippageTolerance

## 1. 大白话讲讲这个 hook 的作用

`useSlippageTolerance` *(一个React hook，用于管理DEX交易中的滑点容差设置，基于LocalStorage持久化)* 是一个管理"滑点 tolerance（容差）"的 hook。在 DEX（去中心化交易所）中，滑点是指你预期的交易价格和实际成交价格之间的差异。

这个 hook 负责：
1. **读取**：从 LocalStorage 读取用户设置的滑点
2. **写入**：保存用户修改的滑点设置
3. **默认值**：提供 DEFAULT_SLIPPAGE 作为初始值

简单来说：它帮你记住用户在交易时愿意接受的滑点比例。

## 2. 讲讲为什么需要封装该 hook

### DEX 交易的需求

在 SushiSwap 进行交易时：
- 用户设置滑点 tolerance（比如 0.5%）
- 如果实际成交价格比预期差 0.5% 以内，交易会执行
- 如果超过 0.5%，交易会失败（防止frontrunning *(一种MEV策略，通过在交易前后操纵价格来获利)*）

### 需要处理好的细节

1. **持久化**：用户设置的滑点不能每次刷新都重置
2. **不同场景**：Swap、添加流动性、移除流动性可能有不同的滑点设置
3. **默认值**：新用户使用默认滑点
4. **类型安全**：滑点是一个字符串（百分比格式）

### 封装的好处

- **场景隔离**：不同场景（swap/添加流动性/移除流动性）用不同的 key
- **持久化**：通过 useLocalStorage 自动保存
- **API 简洁**：返回 `[value, setValue]` 和 useState 用法一样
- **默认配置**：使用 DEFAULT_SLIPPAGE 常量

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `key` | `SlippageToleranceStorageKey` | `SlippageToleranceStorageKey.Swap` | 存储的 key *(用于区分不同场景)* |
| `defaultValue` | `string` | `DEFAULT_SLIPPAGE` | 默认滑点值 |

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| `[value, setValue]` | `[string, Dispatch<SetStateAction<string>>]` | 和 useState 一样的用法 |

### SlippageToleranceStorageKey 枚举

```typescript
export enum SlippageToleranceStorageKey {
  Swap = 'slippage-swap', *(Swap交易的滑点key)*
  AddLiquidity = 'slippage-add-liquidity', *(添加流动性的滑点key)*
  RemoveLiquidity = 'slippage-remove-liquidity', *(移除流动性的滑点key)*
}
```

### 执行逻辑

```
useSlippageTolerance()
    ↓
调用 useLocalStorage('slippage-swap', DEFAULT_SLIPPAGE) *(基于useLocalStorage实现持久化)*
    ↓
返回 [storedValue, setValue] *(返回数组，类似useState)*
    ↓
用户修改: setValue('1.0') *(设置新的滑点值)*
    ↓
写入 LocalStorage key = 'slippage-swap'
    ↓
触发 dispatchEvent 同步其他标签页 *(跨标签页同步)*
```

### 数据流

```
DEFAULT_SLIPPAGE (默认 '0.5')
    ↓
useLocalStorage(key, defaultValue) *(从LocalStorage读取或使用默认值)*
    ↓
LocalStorage.getItem('slippage-swap') *(尝试读取已保存的值)*
    ↓
storedValue (用户保存的值或默认值)
    ↓
setValue → localStorage.setItem *(写入新值到本地存储)*
```

## 四、AI 提示词编写教学

你正在做一个滑点容差设置的小工具。在去中心化交易所交易的时候，实际成交价格可能和预期价格有点差异（这就是滑点）。用户可以设置自己愿意接受的滑点比例，比如 0.5% 或者 1%。这个工具帮用户记住这个设置，刷新页面后不会重置。

不同的操作要用不同的存储 key。Swap 交易、添加流动性、移除流动性，这三种场景的滑点设置是分开存的，互不影响。每个场景用一个唯一的 key 来标记，比如"slippage-swap"、"slippage-add-liquidity"等。

滑点值用字符串格式存储，比如"0.5"表示百分之零点五。存之前要有一个默认值，新用户就用这个默认值。

核心其实是复用了一个本地存储的 hook，来帮你把数据持久化。只要把存储用的 key 和默认值传进去，它就会自动处理读取、保存、还有跨标签页同步这些事情。返回值和 use state 一样，是一个状态值和一个修改函数，用起来很熟悉。

## 五、该 hook 的用法教学

### 基本用法

```typescript
import { useSlippageTolerance } from './useSlippageTolerance'

function SlippageSettings() {
  const [slippage, setSlippage] = useSlippageTolerance()

  return (
    <div>
      <p>当前滑点: {slippage}%</p>
      <button onClick={() => setSlippage('0.5')}>0.5%</button>
      <button onClick={() => setSlippage('1.0')}>1.0%</button>
    </div>
  )
}
```

### 常见使用场景

🟩 **Swap 交易滑点设置**

```typescript
function SwapSettings() {
  const [slippage, setSlippage] = useSlippageTolerance()

  const handleSlippageChange = (value) => {
    setSlippage(value)
  }

  return (
    <div>
      <span>滑点容忍度</span>
      <button onClick={() => handleSlippageChange('0.1')}>0.1%</button>
      <button onClick={() => handleSlippageChange('0.5')}>0.5%</button>
      <button onClick={() => handleSlippageChange('1.0')}>1.0%</button>
      <CustomSlippageInput onChange={handleSlippageChange} value={slippage} />
    </div>
  )
}
```

🟩 **添加流动性时的滑点设置**

```typescript
function AddLiquiditySettings() {
  const [slippage, setSlippage] = useSlippageTolerance(
    SlippageToleranceStorageKey.AddLiquidity
  )

  return (
    <div>
      <p>添加流动性滑点: {slippage}%</p>
      <button onClick={() => setSlippage('0.5')}>0.5%</button>
      <button onClick={() => setSlippage('1.0')}>1.0%</button>
    </div>
  )
}
```

🟩 **显示当前滑点值**

```typescript
function TradeSummary() {
  const [slippage] = useSlippageTolerance()

  return (
    <div className="summary">
      <div>滑点容忍度: {slippage}%</div>
      <div>预计成交价: {estimatedPrice}</div>
      <div>最小成交价: {minPrice}</div>
    </div>
  )
}
```

### Dos and Don'ts

✅ **Do: 使用场景对应的 key**

```typescript
// 正确：Swap 使用默认 key
const [slippage, setSlippage] = useSlippageTolerance()

// 正确：添加流动性使用对应 key
const [slippage, setSlippage] = useSlippageTolerance(
  SlippageToleranceStorageKey.AddLiquidity
)

// 正确：移除流动性使用对应 key
const [slippage, setSlippage] = useSlippageTolerance(
  SlippageToleranceStorageKey.RemoveLiquidity
)
```

❌ **Don't: 混用不同场景的滑点设置**

```typescript
// 错误：添加流动性时使用了 Swap 的滑点
const [addLiquiditySlippage] = useSlippageTolerance() // 默认是 Swap
```

✅ **Do: 使用字符串格式的值**

```typescript
// 正确：滑点值是字符串
setSlippage('0.5')
setSlippage('1.0')

// 使用时转换为数字
const slippagePercent = parseFloat(slippage)
const slippageDecimal = slippagePercent / 100
```

❌ **Don't: 使用数字格式设置滑点**

```typescript
// 错误：滑点值是字符串格式
setSlippage(0.5) // 不要传数字
setSlippage('0.5') // 应该传字符串
```

✅ **Do: 处理滑点验证**

```typescript
// 正确：验证滑点值的合理性
const handleSlippageChange = (value) => {
  const num = parseFloat(value)
  if (num > 0 && num <= 50) {
    setSlippage(value)
  }
}
```

❌ **Don't: 允许极端的滑点值**

```typescript
// 错误：50% 的滑点太高了
setSlippage('50') // 风险太大

// 正确：限制在合理范围
const handleSlippageChange = (value) => {
  const num = parseFloat(value)
  if (num > 0 && num <= 5) { // 最多 5%
    setSlippage(value)
  }
}
```

✅ **Do: 跨标签页自动同步**

```typescript
// 正确：在标签页A修改，标签页B会自动更新
// 标签页A:
setSlippage('1.0')

// 标签页B:
// 自动收到更新，无需额外处理
```

❌ **Don't: 忽略滑点变化的影响**

```typescript
// 错误：滑点变化会影响交易结果，应该通知用户
useEffect(() => {
  if (parseFloat(slippage) > 2) {
    showHighSlippageWarning()
  }
}, [slippage])
```
