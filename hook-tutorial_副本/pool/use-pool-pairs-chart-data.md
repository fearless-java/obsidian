> 源代码路径: `apps/web/src/lib/pool/blade/usePoolPairsChartData.tsx`

# usePoolPairsChartData Tutorial

## 1. 大白话讲讲这个hook的作用

`usePoolPairsChartData` *(一个React hook，用于获取Blade池子交易对的图表数据)* 是用来获取Blade池子交易对图表数据的hook。

简单来说：
- 获取指定时间段内（如24小时、一周、一个月或全部）池子的交易数据
- 返回交易量、价格变化等数据点
- 用于在UI上渲染池子的交易图表

这是一个只读查询hook，用于展示池子的历史交易数据。

## 2. 讲讲为什么需要封装该hook

获取图表数据涉及多个复杂性：

1. **API调用**：需要调用Graph Client的data-api *(SushiSwap的GraphQL数据查询客户端)*
2. **时间范围**：需要支持DAY、WEEK、MONTH、ALL四种时间范围
3. **查询参数**：poolAddress和chainId
4. **React Query集成**：需要管理loading/error状态和缓存

封装成hook后：
- 统一的API调用逻辑
- 支持灵活的时间范围切换
- 与React Query集成，支持缓存和刷新
- 简化UI组件的数据获取

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  poolAddress: EvmAddress *(以太坊虚拟机兼容的地址类型)*            // 池子地址
  chainId: BladeChainId *(Blade支持的链ID枚举)*              // 链ID
  duration?: 'DAY' | 'WEEK' | 'MONTH' | 'ALL'  // 时间范围，默认DAY
  enabled?: boolean                 // 是否启用，默认true
}
```

### 输出（Return Value）
```typescript
{
  data: BladePoolPairsChart | undefined   // 图表数据
  isLoading: boolean
  isError: boolean
  error: Error | null
  // ... 其他useQuery属性
}
```

### BladePoolPairsChart类型（推断）
```typescript
interface BladePoolPairsChart {
  data: {
    timestamp: number
    token0Price: string
    token1Price: string
    volume: string
  }[]
  // ... 其他元数据
}
```

### 执行逻辑
1. 检查enabled、poolAddress、chainId是否都存在
2. 调用getBladePoolPairsChart API
3. 传递poolAddress、chainId、duration参数
4. 使用keepPreviousData *(一个React Query辅助函数，用于在加载新数据时保持旧数据)* 保持旧数据直到新数据加载完成
5. 设置staleTime和gcTime

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于获取Blade池子的交易对图表数据，支持不同时间范围的查询。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合@tanstack/react-query的useQuery来管理数据获取，使用@sushiswap/graph-client来获取图表数据。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数包括池子地址、链ID、时间范围（默认是一天）和启用开关。输出包含数据点数组（每个点有时间戳、两种代币的价格和交易量）以及可选的元数据（起始价、结束价、最高价、最低价和总交易量）。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先从graph-client导入获取图表数据的函数，然后实现useQuery，在queryFn中调用API，配置queryKey、缓存时间和enabled条件，使用keepPreviousData来在加载新数据时保留旧数据。

### 需要特别注意的约束条件

**duration参数有四种选择**

duration参数可以是DAY、WEEK、MONTH或ALL，分别代表24小时、一周、一个月和全部时间。默认值应该是DAY。

**enabled要检查所有必要参数**

enabled应该与poolAddress和chainId同时检查，三者都存在时才执行查询。

**queryKey要包含所有查询参数**

queryKey应该包含poolAddress、chainId和duration，这样每个不同的查询都有唯一的标识。格式可以是['blade', 'pool', `${chainId}:${poolAddress}`, 'pairs', duration]。

**使用keepPreviousData避免空白**

在加载新数据时，应该使用keepPreviousData来保持旧数据，这样用户就不会看到空白或者闪烁的界面。

**设置合理的缓存时间**

可以设置staleTime为60000毫秒（1分钟），gcTime为300000毫秒（5分钟），这样可以在数据时效性和性能之间取得平衡。

### 状态管理的要点

**使用React Query管理状态**

React Query会自动管理API响应的状态。可以直接在UI中使用isLoading、isError等状态。

**keepPreviousData的作用**

keepPreviousData是一个辅助函数，用于在加载新数据时保留旧数据。这对于图表类数据特别有用，可以避免用户看到空白界面。

**缓存策略**

合理的staleTime和gcTime设置可以减少不必要的请求，同时保证数据的时效性。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于获取Blade池子的交易对图表数据，支持不同时间范围的查询。

需要使用@tanstack/react-query的useQuery来管理数据获取，使用@sushiswap/graph-client来获取数据。

输入参数包括池子地址、链ID、时间范围（默认DAY）和启用开关（默认true）。

输出包含数据点数组（每个点包含时间戳、两种代币的价格和交易量）以及可选的元数据（起始价、结束价、最高价、最低价和总交易量）。

实现步骤：
1. 从@sushiswap/graph-client/data-api导入getBladePoolPairsChart函数
2. 从@tanstack/react-query导入useQuery和keepPreviousData
3. 解构参数，设置默认值：duration = 'DAY', enabled = true
4. 实现useQuery：
   - queryKey使用['blade', 'pool', 'chainId:poolAddress', 'pairs', duration]
   - queryFn调用getBladePoolPairsChart({ address: poolAddress, chainId, duration })
   - 使用placeholderData: keepPreviousData
   - 设置staleTime为60000，gcTime为300000
   - enabled检查所有必要参数是否都存在

重要约束：
- duration参数有四种选择，默认值是DAY
- enabled应该与poolAddress和chainId同时检查
- queryKey包含所有查询参数
- 使用keepPreviousData避免加载时闪烁
- 设置合理的缓存时间

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { usePoolPairsChartData } from '@sushiswap/hooks'

function PoolChart({ poolAddress, chainId }) {
  const [duration, setDuration] = useState<'DAY' | 'WEEK' | 'MONTH' | 'ALL'>('DAY')

  const { data: chartData, isLoading } = usePoolPairsChartData({
    poolAddress,
    chainId,
    duration,
  })

  return (
    <div>
      <div>
        <button onClick={() => setDuration('DAY')}>24小时</button>
        <button onClick={() => setDuration('WEEK')}>一周</button>
        <button onClick={() => setDuration('MONTH')}>一个月</button>
        <button onClick={() => setDuration('ALL')}>全部</button>
      </div>
      {isLoading ? (
        <div>加载中...</div>
      ) : (
        <div>
          {/* 渲染图表 */}
          {chartData?.data.map((point) => (
            <div key={point.timestamp}>
              {new Date(point.timestamp).toLocaleDateString()}: {point.volume}
            </div>
          ))}
        </div>
      )}
    </div>
  )
}
```

### 常见使用场景

1. **交易对价格图表**
   - 显示价格随时间的变化
   - 帮助用户分析价格趋势

2. **交易量统计**
   - 展示不同时间段的总交易量
   - 帮助用户了解市场活跃度

3. **时间范围切换**
   - 支持多种时间维度的数据查看
   - 提升用户体验

### Dos and Don'ts

**Dos：**
- ✅ 合理设置duration的默认值（通常为DAY）
- ✅ 使用keepPreviousData避免图表闪烁
- ✅ 在时间范围切换时保持之前的数据可见
- ✅ 使用staleTime和gcTime优化缓存

**Don'ts：**
- ❌ 不要在duration变化时立即显示loading状态遮挡住旧数据
- ❌ 不要忽略isLoading状态，应该给用户反馈
- ❌ 不要频繁切换duration导致大量请求
- ❌ 不要直接渲染原始数据，应该先格式化（如价格保留小数位）
