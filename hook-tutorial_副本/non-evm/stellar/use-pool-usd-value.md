> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/pool/use-pool-usd-value.ts`

# useLPUsdValue Hook Tutorial

## 大白话讲讲这个hook的作用

`useLPUsdValue` *(一个React hook，用于计算流动性池总USD价值，通过组合两个代币的价格和储备量计算)* 是一个用于计算流动性池总 USD 价值的 hook。它计算池子中：

- Token0 储备量的 USD 价值
- Token1 储备量的 USD 价值
- 两者相加的总价值

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **价格组合**：需要组合两个代币的价格（通过 `useStablePrice` *(获取代币USD价格的hook)*）
2. **储备量数据**：需要池子的 reserve0 和 reserve1
3. **计算逻辑**：将代币数量乘以价格得到 USD 价值

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  token0: Token            // Token0 代币信息
  token1: Token           // Token1 代币信息
  reserve0: bigint        // Token0 储备量（最小单位）
  reserve1: bigint         // Token1 储备量（最小单位）
}
```

### 输出（返回值）
```typescript
{
  data: number             // 总 USD 价值
}
```

### 核心执行逻辑

1. **获取价格**：使用 `useStablePrice` 获取 token0 和 token1 的 USD 价格
2. **计算价值**：reserve * price / 10^decimals
3. **汇总返回**：token0USD + token1USD

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个计算池子USD总价值的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useLPUsdValue的hook，用来计算某个流动性池的美元总价值。然后明确几个关键点。第一，参数要包括两个代币的信息和两个代币的储备量。第二，获取这两个代币各自的美元价格，用useStablePrice来获取。第三，用储备量乘以价格再除以精度位数，得到每个代币的美元价值。第四，把两个代币的价值加起来就是总价值。

### 这里面有几个地方特别容易出错

依赖价格数据，所以要确保两个代币的价格数据都能拿到才能计算。储备量是最小单位，不是人类能看懂的数字，要除以10的decimals次方才能转换成实际数量。这个计算看起来就是简单的乘法和加法，但每一步都要注意单位转换。

### 数据刷新这里有讲究

enabled条件要设成两个价格数据都可用才行，只有其中一个的话算出来的结果是不完整的。这个hook做的是计算而不是查询，所以缓存策略跟查询类的hook不太一样。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useLPUsdValue } from '@sushiswap/hooks'

function PoolTVLDisplay({ pool }: { pool: PoolInfo }) {
  const { data: tvlUsd } = useLPUsdValue({
    token0: pool.token0,
    token1: pool.token1,
    reserve0: pool.reserve0,
    reserve1: pool.reserve1,
  })

  return (
    <div>
      <p>池子 TVL: ${tvlUsd?.toLocaleString() ?? '0'}</p>
    </div>
  )
}
```

### 常见使用场景

1. **池子 TVL 显示**：在池子列表中显示总锁仓量
   ```tsx
   const tvl = useLPUsdValue({ token0, token1, reserve0, reserve1 })
   return <span>TVL: ${formatUSD(tvl)}</span>
   ```

2. **收益率计算**：TVL 与日交易量的比率
   ```tsx
   const { data: tvl } = useLPUsdValue(...)
   const { data: dailyVolume } = usePoolDailyVolumeUsd(address)
   const apr = (Number(dailyVolume) * 365) / Number(tvl) * 100
   ```

3. **排序依据**：按 TVL 排序显示池子
   ```tsx
   pools.sort((a, b) => Number(getTVL(b)) - Number(getTVL(a)))
   ```

### Dos and Don'ts

**Dos:**
- ✅ 配合 `usePoolInfo` *(获取池子详细信息的hook)* 获取储备量数据
- ✅ 使用 `useStablePrice` *(获取代币USD价格的hook)* 获取价格
- ✅ 格式化显示带千分位

**Don'ts:**
- ❌ 不要在价格数据未加载时阻塞，显示加载状态
- ❌ 不要忽略 decimals 处理
- ❌ 不要设置过长的缓存时间，价格会变化
