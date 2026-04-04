> 源代码路径: `apps/web/src/lib/hooks/useTokenAmountDollarValues.ts`

# useTokenAmountDollarValues

## 1. 大白话讲讲这个hook的作用

`useTokenAmountDollarValues` *(一个React hook，用于将一组代币数量批量转换为对应的美元价值，支持多链)* 帮你把一组代币数量转换成对应的美元价值。

比如你有 [1000 USDC, 0.5 ETH, 200 USDT]，这个hook会：
1. 获取每个代币的当前价格
2. 计算每个代币的美元价值
3. 返回 [1000, 950, 200]（假设ETH=$1900）

## 2. 讲讲为什么需要封装该hook

手动计算美元价值很繁琐：
- 需要获取每个代币的价格
- 需要处理精度问题
- 需要处理价格不存在的情况
- 小额代币（<$0.000001）应该显示为0

封装后：
- 自动获取价格数据
- 处理各种边界情况
- 返回直接可用的美元数值

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
{
  chainId: TChainId                    // 链ID
  amounts: (Amount<Currency> | undefined)[] | null | undefined  // 代币数量数组
}
// TChainId extends EvmChainId | SvmChainId
```

**输出：**
```typescript
number[]  // 美元价值数组
```

**执行逻辑：**
1. 如果 amounts 为 null/undefined，返回空数组
2. 遍历每个 amount：
   a. 获取代币的包装地址（`.wrap().address`）
   b. 检查 `amount > 0` 且价格在 map 中存在
   c. 计算：`amount * price`
   d. 如果结果 < 0.000001，返回 0（避免dust）
3. 返回美元价值数组

**数据流：**
```
[amount0, amount1, amount2]
         |
         v
    获取价格 --> [price0, price1, price2]
         |
         v
    计算美元 --> [value0, value1, value2]
```

## 4. 怎么给这个hook写AI提示词

这个hook干的事儿很直接：给你一组代币数量，算出每个代币值多少美元。比如你有 [1000 USDC, 0.5 ETH]，它会返回 [1000, 950]（假设ETH=$1900）。

### 写提示词的小技巧

**第一，太小金额要返回0。** 如果计算出来不到一美分的千分之一（$0.000001），显示出来就是一堆零，没意义。直接返回0，用户看起来也清爽。

**第二，用useMemo缓存。** 金额和价格没变的话，美元价值也不该变。每次渲染都重新算一遍太浪费。

**第三，undefined/null要能兼容。** 传进来的 amounts 数组里可能有 undefined 的元素（比如还在loading），代码要能处理这种情况，而不是直接崩掉。

### 写提示词时要注意的条条框框

**金额必须大于0才算。** 如果 amount 小于等于 0，直接返回0，不用查价格。

**价格查不到的话价值就是0。** 有些新上线的代币可能没有价格数据，这种情况下只能显示 $0，而不是报错。

**精度处理要注意。** 计算结果用 toFixed(10) 然后转 number，这样能避免浮点数精度问题，最后显示的时候再用 toFixed(2) 保留两位小数。

### 提示词模板

```
帮我写一个React hook，功能是把一组代币数量转成对应的美元价值。

具体需求：
1. 输入链ID和代币数量数组
2. 用 usePrices 拿到每个代币的当前价格
3. 逐个计算美元价值：数量 × 价格
4. 如果算出来小于 $0.000001，返回 0（避免显示垃圾数字）
5. 价格查不到的话，那一项的价值就是 0

大概的类型：
function useTokenAmountDollarValues<T extends EvmChainId | SvmChainId>({
  chainId: T
  amounts: (Amount<CurrencyFor<T>> | undefined)[]
}): number[]

注意事项：
- amounts 数组里可能有 undefined 或 null，要能正常处理
- 用 useMemo 缓存结果，别每次渲染都重新算
- 金额必须大于 0 才参与计算
- 价格不存在的代币直接返回 0
- dust 值（<$0.000001）也返回 0
```

### 实际用的例子

```typescript
const amounts = [usdcBalance, ethBalance, usdtBalance]

const dollarValues = useTokenAmountDollarValues({
  chainId: ChainId.ETHEREUM,
  amounts,
})

const totalValue = dollarValues.reduce((sum, val) => sum + val, 0)
console.log(`总价值: $${totalValue.toFixed(2)}`)
```

这个例子算出了三个代币的总价值，最后用 toFixed(2) 显示成美元形式。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 获取用户的多个代币余额
const { data: usdcBalance } = useBalance(account, usdcToken)
const { data: ethBalance } = useBalance(account, ethToken)
const { data: usdtBalance } = useBalance(account, usdtToken)

// 转换为美元价值
const dollarValues = useTokenAmountDollarValues({
  chainId: ChainId.ETHEREUM,
  amounts: [usdcBalance, ethBalance, usdtBalance],
})

// dollarValues 是 number[] 数组
// [1000, 950, 200] 表示 USDC=$1000, ETH=$950, USDT=$200
```

### 常见使用场景

**场景1：计算投资组合总价值**

```typescript
// 获取投资组合中所有代币的价值
const positions = [usdc, eth, usdt, wbtc]
const { data: balances } = useBalances(account, positions)

const dollarValues = useTokenAmountDollarValues({
  chainId,
  amounts: balances,
})

// 计算总价值
const totalValue = dollarValues.reduce((sum, val) => sum + val, 0)
console.log(`投资组合总价值: $${totalValue.toFixed(2)}`)
```

**场景2：显示单个资产价值**

```typescript
const { data: ethBalance } = useBalance(account, ETH)
const dollarValues = useTokenAmountDollarValues({
  chainId,
  amounts: [ethBalance],
})

// 显示ETH的价值
<span>ETH: ${dollarValues[0]?.toFixed(2) ?? '0.00'}</span>
```

**场景3：在交易中使用**

```typescript
// 交易前显示各代币的美元价值
const { data: token0Balance } = useBalance(account, token0)
const { data: token1Balance } = useBalance(account, token1)

const values = useTokenAmountDollarValues({
  chainId,
  amounts: [token0Balance, token1Balance],
})

// 显示交易对的美元价值
console.log(`交易对总价值: $${(values[0] + values[1]).toFixed(2)}`)
```

### Dos and Don'ts

**✅ Do:**
- 使用 `useMemo` 缓存计算结果，避免重复计算
- 处理 amounts 为 undefined/null 的情况
- 小额代币（< $0.000001）会自动返回0，这是预期行为
- 返回值是 number 类型，可以直接用于显示和计算

**❌ Don't:**
- 不要假设返回值数组的每个元素都有效，可能返回 0
- 不要在每次渲染时重新创建 amounts 数组，应该用 `useMemo`
- 不要忽略价格不存在的情况，这种情况下该代币价值为0
- 不要对返回值进行 toFixed 后再参与计算，这会丢失精度，应该在最后显示时再格式化
