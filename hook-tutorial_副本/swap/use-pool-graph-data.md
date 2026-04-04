> 源代码路径: `apps/web/src/lib/hooks/api/usePoolGraphData.ts`

# usePoolGraphData

## 1. 大白话讲讲这个hook的作用

`usePoolGraphData` *(一个React hook，用于获取SushiSwap池子的历史数据（价格、流动性、交易量），用于绘制图表)* 帮你获取池子的历史数据（用于画图表）。

池子图数据包含：
- dayBuckets：按天聚合的数据（价格、流动性、交易量等）
- hourBuckets：按小时聚合的数据

这些数据用于在池子详情页展示：
- 价格走势图
- 流动性变化图
- 交易量历史

## 2. 讲讲为什么需要封装该hook

不同协议（V2/V3/Blade）有不同的数据获取接口：
- 需要根据 protocol 和 chainId 选择正确的API
- 数据结构不同但返回格式统一
- 需要优雅处理不支持的协议组合

封装后：
- 自动选择正确的API
- 统一返回格式
- 处理不支持的情况

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
{
  poolAddress: EvmAddress       // 池子地址
  chainId: SushiSwapV2ChainId | SushiSwapV3ChainId | BladeChainId
  protocol: SushiSwapProtocol  // SUSHISWAP_V2 | SUSHISWAP_V3 | BLADE
  enabled?: boolean             // 默认true
}
```

**输出：**
```typescript
{
  data: {
    dayBuckets: Bucket[]   // 日数据
    hourBuckets: Bucket[] // 小时数据
  },
  isLoading: boolean,
  error: Error | null
}
```

**执行逻辑：**
1. 根据 protocol 判断调用哪个函数：
   - SUSHISWAP_V2 -> `getV2PoolBuckets()`
   - SUSHISWAP_V3 -> `getV3PoolBuckets()`
   - BLADE -> `getBladePoolBuckets()`
2. 传入 chainId 和 address
3. 不支持的协议组合返回空数组
4. 使用 `keepPreviousData` *(react-query的选项，用于在请求新数据时保持显示旧数据，避免闪烁)* 保持旧数据

**数据流：**
```
{protocol, chainId, poolAddress}
         |
         v
    协议判断
    SUSHISWAP_V2 --> getV2PoolBuckets
    SUSHISWAP_V3 --> getV3PoolBuckets
    BLADE --> getBladePoolBuckets
         |
         v
    { dayBuckets: [], hourBuckets: [] }
```

## 4. 怎么给这个hook写AI提示词

这个hook是拿来画图表的——获取池子的历史数据（价格、流动性、交易量等），然后喂给图表组件显示。数据分两种：按天聚合的（dayBuckets）和按小时聚合的（hourBuckets）。

### 写提示词的小技巧

**第一，切换池子时不要闪烁。** 用 `keepPreviousData` 选项，可以让在加载新数据的时候继续显示旧数据，等新数据来了再替换。用户看起来就不会有一闪而过的空白。

**第二，数据要新鲜。** staleTime 设成 0，表示数据永远是新鲜的，不存在"缓存过期"这个概念。图表数据必须实时。

**第三，缓存设成一小时比较合适。** gcTime 设成 3600 秒（1小时），意思是数据可以缓存这么久。图表数据变化不那么快，一小时内的重复请求都走缓存。

### 写提示词时要注意的条条框框

**协议和链要匹配。** V2链用V2的API，V3链用V3的API，Blade链用Blade的API。不匹配的话会返回空数组，不会报错。

**poolAddress和chainId必须同时有。** 光有一个没用，必须两个都有才会发请求。

### 提示词模板

```
帮我写一个React hook，功能是获取池子的历史图表数据。

具体需求：
1. 根据协议类型选择调哪个API：
   - SUSHISWAP_V2 -> getV2PoolBuckets
   - SUSHISWAP_V3 -> getV3PoolBuckets
   - BLADE -> getBladePoolBuckets
2. 支持 enabled 开关
3. 如果协议和链不匹配，返回空数组，不报错
4. 用 keepPreviousData 保持切换时的显示

大概的参数类型：
{
  poolAddress: EvmAddress
  chainId: SushiSwapV2ChainId | SushiSwapV3ChainId | BladeChainId
  protocol: SushiSwapProtocol
  enabled?: boolean
}

返回：{ dayBuckets: Bucket[], hourBuckets: Bucket[] }
```

### 实际用的例子

```typescript
const { data, isLoading } = usePoolGraphData({
  poolAddress: pool.address,
  chainId: pool.chainId,
  protocol: SushiSwapProtocol.SUSHISWAP_V3,
})

// 传给图表组件
<PriceChart buckets={data?.hourBuckets} />
```

hourBuckets 是小时级别的数据，适合看短期趋势；dayBuckets 是日数据，适合看长期走势。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 获取池子历史数据
const { data, isLoading, error } = usePoolGraphData({
  poolAddress: '0x...',
  chainId: ChainId.ETHEREUM,
  protocol: SushiSwapProtocol.SUSHISWAP_V3,
})

// data 包含 dayBuckets 和 hourBuckets
console.log(data?.hourBuckets) // 小时数据数组
console.log(data?.dayBuckets) // 日数据数组
```

### 常见使用场景

**场景1：价格走势图**

```typescript
const { data } = usePoolGraphData({...})

// 提取价格数据用于图表
const priceData = data?.hourBuckets?.map(bucket => ({
  time: bucket.timestamp,
  price: bucket.open, // 或 close, high, low
}))
```

**场景2：流动性变化图**

```typescript
const { data } = usePoolGraphData({...})

// 提取流动性数据
const liquidityData = data?.dayBuckets?.map(bucket => ({
  date: bucket.date,
  liquidity: bucket.liquidityUSD, // 流动性USD价值
}))
```

**场景3：交易量历史**

```typescript
const { data } = usePoolGraphData({...})

// 提取交易量数据
const volumeData = data?.dayBuckets?.map(bucket => ({
  date: bucket.date,
  volume: bucket.volumeUSD, // 交易量USD
}))
```

### Dos and Don'ts

**✅ Do:**
- 使用 `keepPreviousData` 避免切换池子时的闪烁
- 设置合理的 `gcTime`（如1小时）缓存数据
- 根据需要选择 dayBuckets 或 hourBuckets
- 处理 `isLoading` 状态显示加载指示

**❌ Don't:**
- 不要在不存在的协议/链组合上调用，会返回空数据
- 不要忽略 `enabled` 控制，在不需要时禁用获取
- 不要缓存太久不刷新，市场数据需要相对实时
- 不要假设所有 bucket 都有完整数据，检查字段是否存在
