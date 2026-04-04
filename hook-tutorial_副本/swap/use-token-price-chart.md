> 源代码路径: `apps/web/src/lib/hooks/api/useTokenPriceChart.ts`

# useTokenPriceChart

## 1. 大白话讲讲这个hook的作用

`useTokenPriceChart` *(一个React hook，用于获取代币的价格历史数据，用于绘制价格走势图)* 帮你获取代币的价格历史数据，用于画价格走势图。

可以获取：
- 任意时间段的价格数据（24小时、7天、30天等）
- OHLC（开盘、最高、最低、收盘）数据
- 交易量数据

## 2. 讲讲为什么需要封装该hook

封装提供：
- 简洁的API调用
- 条件获取支持
- 标准react-query接口

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
args: GetTokenPriceChart
shouldFetch?: boolean
```

**输出：**
```typescript
{
  data: TokenPriceChart,
  isLoading: boolean,
  error: Error | null
}
```

**执行逻辑：**
1. 调用 `getTokenPriceChart(args)` *(获取代币价格图表数据的API函数)*
2. enabled = shouldFetch && args 有效

## 4. 怎么给这个hook写AI提示词

这个hook拿来查代币的价格历史，可以查24小时的、7天的、30天的、1年的。返回的数据包含OHLC（开盘、最高、最低、收盘）还有交易量，足够画一根K线出来。

### 写提示词的小技巧

**第一，参数要全。** chainId、address、timeframe 这三个必须有。chainId知道去哪查，address知道查谁，timeframe知道查哪段时间。

**第二，shouldFetch当开关。** 参数不完整的时候（比如用户还没输入地址）就不发请求。

### 提示词模板

```
帮我写一个React hook，功能是获取代币的价格历史图表数据。

具体需求：
1. 调用 getTokenPriceChart 函数
2. 支持 shouldFetch 开关

参数类型大概包含：chainId、address、timeframe（时间范围）

返回：{ data: TokenPriceChart, isLoading, error }
```

### 实际用的例子

```typescript
const { data: priceChart } = useTokenPriceChart({
  chainId: ChainId.ETHEREUM,
  address: WETH_ADDRESS,
  timeframe: '7d',
})

// 显示价格图表
<Chart data={priceChart} />
```

timeframe 可以是 '24h'、'7d'、'30d'、'1y' 这些选项。

### Production-Ready Example

```typescript
const { data: priceChart } = useTokenPriceChart({
  chainId: ChainId.ETHEREUM,
  address: WETH_ADDRESS,
  timeframe: '7d',
})

// 显示价格图表
<Chart data={priceChart} />
```

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 基础调用
const { data: priceChart, isLoading, error } = useTokenPriceChart({
  chainId: ChainId.ETHEREUM,
  address: WETH_ADDRESS,
  timeframe: '7d', // 时间范围: '24h', '7d', '30d', '1y'
})

// priceChart 包含 OHLC 数据
console.log(priceChart?.data) // OHLC数据数组
```

### 常见使用场景

**场景1：价格走势图**

```typescript
const [timeframe, setTimeframe] = useState('7d')

const { data: priceChart } = useTokenPriceChart({
  chainId,
  address: tokenAddress,
  timeframe,
})

// 转换为图表数据
const chartData = priceChart?.data?.map(candle => ({
  time: candle.timestamp,
  open: candle.open,
  high: candle.high,
  low: candle.low,
  close: candle.close,
}))

return <LineChart data={chartData} />
```

**场景2：时间范围切换**

```typescript
const [timeframe, setTimeframe] = useState('7d')

const { data: priceChart, isLoading } = useTokenPriceChart({
  chainId,
  address: tokenAddress,
  timeframe,
})

// 时间范围选项
const timeframes = [
  { label: '24小时', value: '24h' },
  { label: '7天', value: '7d' },
  { label: '30天', value: '30d' },
  { label: '1年', value: '1y' },
]

return (
  <div>
    <TimeframeSelector
      options={timeframes}
      value={timeframe}
      onChange={setTimeframe}
    />
    {isLoading ? <LoadingSpinner /> : <Chart data={priceChart} />}
  </div>
)
```

**场景3：OHLC蜡烛图**

```typescript
const { data: priceChart } = useTokenPriceChart({...})

// 使用OHLC数据绘制蜡烛图
return (
  <CandlestickChart>
    {priceChart?.data?.map(candle => (
      <Candle
        key={candle.timestamp}
        open={candle.open}
        high={candle.high}
        low={candle.low}
        close={candle.close}
        volume={candle.volume}
      />
    ))}
  </CandlestickChart>
)
```

### Dos and Don'ts

**✅ Do:**
- 使用合适的 timeframe 参数（24h, 7d, 30d, 1y）
- 使用 `shouldFetch` 在不需要时禁用获取
- 处理 `isLoading` 状态显示加载指示
- 根据 timeframe 调整图表刷新频率

**❌ Don't:**
- 不要使用过大的 timeframe 组合导致数据量过大
- 不要忽略 `error` 处理
- 不要在组件卸载后仍然更新状态
- 不要假设数据总是按时间顺序返回
