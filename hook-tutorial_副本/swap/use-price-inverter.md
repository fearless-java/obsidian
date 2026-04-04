> 源代码路径: `apps/web/src/lib/hooks/usePriceInverter.ts`

# usePriceInverter

## 1. 大白话讲讲这个hook的作用

`usePriceInverter` *(一个React hook，用于翻转价格显示方向，如将 ETH/USDT 转为 USDT/ETH)* 是一个帮你"翻转"价格显示的hook。

在交易中，价格有两种表示方式：
- ETH/USDT = 2000（1 ETH = 2000 USDT）
- USDT/ETH = 0.0005（1 USDT = 0.0005 ETH）

这个hook可以让你在需要的时候把价格方向反转，比如在显示市价价格时用 ETH/USDT，但用户界面可能需要显示 USDT/ETH。

## 2. 讲讲为什么需要封装该hook

价格翻转逻辑简单但容易出错：
- 每次都要调用 `.invert()` 方法
- 容易忘记同时翻转 base 和 quote
- 条件判断（是否需要翻转）散落在各处

封装后：
- 统一的翻转逻辑
- base/quote同时处理
- 简洁的接口

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
{
  priceLower?: Price<TCurrency, TCurrency>  // 价格下限
  priceUpper?: Price<TCurrency, TCurrency>  // 价格上限
  quote?: TCurrency                         // 报价货币
  base?: TCurrency                          // 基础货币
  invert?: boolean                          // 是否翻转
}
```

**输出：**
```typescript
{
  priceLower?: Price<TCurrency, TCurrency>
  priceUpper?: Price<TCurrency, TCurrency>
  quote?: TCurrency
  base?: TCurrency
}
```

**执行逻辑：**
1. 如果 `invert === true`：
   - `priceUpper = priceLower?.invert()`
   - `priceLower = priceUpper?.invert()`（注意这里的priceUpper是原始值）
   - `quote = base`
   - `base = quote`
2. 如果 `invert === false/falsy`：
   - 保持原值

**数据流：**
```
{ priceLower, priceUpper, quote, base, invert }
         |
         v
    invert ? 反转 : 保持
         |
         v
    { priceLower, priceUpper, quote, base }
```

## 4. 怎么给这个hook写AI提示词

这个hook很简单，就是翻转价格的显示方向。比如 ETH/USDT = 2000，翻转后变成 USDT/ETH = 0.0005。数值变了但实际价值没变，只是显示给用户看的方式不一样。

### 写提示词的小技巧

**第一，用useMemo缓存结果。** 这个hook干的是纯计算活儿，每次渲染都重新算一遍太浪费。priceLower、priceUpper、quote、base、invert 这些依赖只要没变，就用缓存的结果。

**第二，invert逻辑要简单。** 就是 if-else，当它是 true 的时候做翻转，false 的时候保持原样。别搞太复杂。

**第三，.invert()不修改原对象。** Price对象的 invert() 方法会返回一个新的Price对象，原来的那个不变。这是好事儿，意味着你不用担心数据被意外修改。

### 写提示词时要注意的条条框框

**Price类型两个参数要一样。** Price<TCurrency, TCurrency>，前后都是同一个类型。这不是bug，是这个hook设计如此。

**翻转只影响显示，不影响价值。** 1000 USDT 换 0.5 ETH，翻转后 0.5 ETH 换 1000 USDT，本质上没区别。只是用户界面可能需要展示成其中一种。

**保持类型安全。** TypeScript会帮你确保类型正确，别用 any 绕过去。

### 提示词模板

```
帮我写一个React hook，功能是翻转价格的显示方向。

具体需求：
1. 输入：priceLower、priceUpper、quote、base、invert
2. 当 invert 为 true 的时候，翻转价格方向（ETH/USDT 变成 USDT/ETH）
3. 用 Price 对象的 invert() 方法来做翻转
4. 用 useMemo 缓存计算结果

大概的类型：
function usePriceInverter<T extends Currency>({
  priceLower?: Price<T, T>
  priceUpper?: Price<T, T>
  quote?: T
  base?: T
  invert?: boolean
}): { priceLower?, priceUpper?, quote?, base? }

注意事项：
- Price.invert() 会创建新对象，不会修改原来的
- invert 为 true 的时候要交换 quote 和 base
- 用 useMemo 避免每次渲染都重新计算
```

### 实际用的例子

```typescript
const { priceLower, priceUpper, quote, base } = usePriceInverter({
  priceLower,
  priceUpper,
  quote,
  base,
  invert: showInverted,
})

// 当 showInverted 为 true 时，价格方向会翻转
```

用户可以切换按钮来选择看 ETH/USDT 还是 USDT/ETH。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 基础使用
const { priceLower, priceUpper, quote, base } = usePriceInverter({
  priceLower: currentPrice,
  quote: USDT,
  base: ETH,
  invert: false,
})

// 当 invert 为 true 时，价格方向会翻转
// ETH/USDT 变成 USDT/ETH
```

### 常见使用场景

**场景1：交易对价格显示切换**

```typescript
const [showInverted, setShowInverted] = useState(false)

const { priceLower, priceUpper, quote, base } = usePriceInverter({
  priceLower: poolPrice,
  quote: token0,
  base: token1,
  invert: showInverted,
})

// 显示价格（根据invert状态自动切换方向）
<span>
  {showInverted ? '1 ' + base?.symbol : quote?.symbol + '/'} {base?.symbol}
  = {priceLower?.toSignificant()}
</span>

// 切换按钮
<Button onClick={() => setShowInverted(!showInverted)}>
  翻转价格显示
</Button>
```

**场景2：价格区间显示**

```typescript
const { priceLower, priceUpper, quote, base } = usePriceInverter({
  priceLower: tickLowerPrice,
  priceUpper: tickUpperPrice,
  quote: token0,
  base: token1,
  invert: isInverted,
})

// 显示价格范围
<span>
  {priceLower?.toSignificant()} - {priceUpper?.toSignificant()}
  {isInverted ? token0.symbol : token1.symbol}
</span>
```

**场景3：深度聚合价格**

```typescript
// 在计算深度数据时使用
const { priceLower, priceUpper } = usePriceInverter({
  priceLower: depthData.priceLower,
  priceUpper: depthData.priceUpper,
  quote: depthData.quote,
  base: depthData.base,
  invert: shouldInvert,
})
```

### Dos and Don'ts

**✅ Do:**
- 使用 `useMemo` 缓存结果，避免重复计算
- 保持 invert 状态独立管理，不要混淆业务逻辑和显示逻辑
- 当 invert 变化时，确保同时翻转 quote 和 base
- Price.invert() 创建新对象，原对象不受影响

**❌ Don't:**
- 不要直接修改 Price 对象，应该用 `.invert()` 返回新对象
- 不要忘记处理 `invert` 为 false/undefined 的情况
- 不要在每次渲染时重新创建 Price 对象，应该复用
- 不要将 invert 用于业务逻辑计算，只用于显示目的
