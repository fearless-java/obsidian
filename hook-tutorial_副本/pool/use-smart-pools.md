> 源代码路径: `apps/web/src/lib/steer/hooks/use-smart-pools.ts`

# useSmartPools Tutorial

## 1. 大白话讲讲这个hook的作用

`useSmartPools` *(一个React hook，用于获取SushiSwap的Smart Pools列表)* 是用来获取SushiSwap的Smart Pools（智能池）列表的hook。

简单来说：
- Smart Pool是SushiSwap的一种自动化策略池子
- 由Steer协议 *(一个提供自动化流动性策略的协议)* 提供支持
- 这个hook获取链上所有可用的Smart Pool列表
- 返回每个池子的配置信息

这是一个只读查询hook，用于展示可用的Smart Pool列表。

## 2. 讲讲为什么需要封装该hook

获取Smart Pools列表涉及：

1. **API调用**：需要调用Graph Client的data-api *(SushiSwap的GraphQL数据查询客户端)*
2. **参数处理**：需要传入chainId
3. **条件获取**：shouldFetch控制是否获取
4. **React Query集成**：需要管理loading/error状态

封装成hook后：
- 统一的API调用逻辑
- 支持条件获取
- 与React Query集成，支持缓存和刷新
- 简化UI组件的数据获取

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  chainId: SmartPoolChainId *(支持Smart Pool的链ID枚举)*           // 链ID
  shouldFetch?: boolean               // 是否获取，默认true
}
```

### 输出（Return Value）
```typescript
{
  data: SmartPoolsV1 | undefined      // Smart Pool列表
  isLoading: boolean
  isError: boolean
  error: Error | null
  // ... 其他useQuery属性
}
```

### SmartPoolsV1类型（推断）
```typescript
interface SmartPoolV1 {
  id: string                         // 池子ID
  address: string                    // 池子地址
  token0: { address: string, symbol: string }
  token1: { address: string, symbol: string }
  fee: number                        // 费率
  // ... 其他配置
}
```

### 执行逻辑
1. 检查chainId存在且shouldFetch为true
2. 调用getSmartPools API
3. 返回React Query标准结果

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于获取SushiSwap的Smart Pools列表。Smart Pool是由Steer协议提供的自动化策略池子。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合@tanstack/react-query的useQuery来管理数据获取，使用@sushiswap/graph-client来获取数据。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数是一个包含chainId的对象和shouldFetch开关。输出是Smart Pool列表，每个池子包含ID、地址、两种代币的信息、费率和策略等。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先从graph-client导入获取Smart Pools的函数和类型，然后实现useQuery，在queryFn中调用API，配置queryKey和enabled条件。

### 需要特别注意的约束条件

**enabled检查**

enabled检查应该是shouldFetch && chainId，shouldFetch默认为true，但只有当chainId也存在时才真正执行查询。

**queryKey的设计**

queryKey应该包含完整的args对象，这样可以确保每个不同的查询都有唯一的标识。格式可以是['smart-pools', args]。

**参数传递方式**

args是一个包含chainId的对象，直接作为queryKey的一部分传入，这样参数变化时会自动触发新的查询。

### 状态管理的要点

**使用React Query管理状态**

React Query会自动管理API响应的状态。可以直接在UI中使用isLoading、isError等状态。

**shouldFetch的作用**

shouldFetch是一个控制是否执行查询的开关，默认为true。当设为false时，不会发送请求。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于获取SushiSwap的Smart Pools列表。Smart Pool是由Steer协议提供的自动化策略池子。

需要使用@tanstack/react-query的useQuery来管理数据获取，使用@sushiswap/graph-client来获取数据。

输入参数是一个包含chainId的对象和shouldFetch开关（默认true）。

输出是Smart Pool列表，每个池子包含ID、地址、两种代币的信息（地址、符号、小数位数）、费率和策略地址等。

实现步骤：
1. 从@sushiswap/graph-client/data-api导入getSmartPools函数和类型
2. 从@tanstack/react-query导入useQuery
3. 解构参数，设置shouldFetch默认值为true
4. 实现useQuery：
   - queryKey使用['smart-pools', args]
   - queryFn调用getSmartPools(args)
   - enabled检查Boolean(shouldFetch && args.chainId)

重要约束：
- shouldFetch默认为true
- enabled检查shouldFetch && chainId
- queryKey包含完整的args对象

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useSmartPools } from '@sushiswap/hooks'

function SmartPoolList({ chainId }) {
  const { data: pools, isLoading, error } = useSmartPools({
    args: { chainId },
  })

  if (isLoading) return <div>加载中...</div>
  if (error) return <div>加载失败</div>
  if (!pools || pools.length === 0) return <div>暂无Smart Pools</div>

  return (
    <div>
      <h3>Smart Pools ({pools.length})</h3>
      {pools.map((pool) => (
        <div key={pool.id}>
          <p>池子: {pool.token0.symbol}/{pool.token1.symbol}</p>
          <p>费率: {pool.fee / 10000}%</p>
          <p>地址: {pool.address}</p>
        </div>
      ))}
    </div>
  )
}
```

### 常见使用场景

1. **Smart Pool列表展示**
   - 显示链上所有可用的Smart Pool
   - 展示池子基本信息

2. **选择管理目标**
   - 用户选择要管理的Smart Pool
   - 用于后续的策略操作

3. **数据分析**
   - 结合其他数据展示完整信息
   - 辅助投资决策

### Dos and Don'ts

**Dos：**
- ✅ 合理使用shouldFetch控制是否获取
- ✅ 处理好空数组情况
- ✅ 使用正确的chainId
- ✅ 设计合理的queryKey

**Don'ts：**
- ❌ 不要在不支持的chainId上调用
- ❌ 不要忽略isLoading和error状态
- ❌ 不要频繁调用，设置合理的缓存
- ❌ 不要假设数据一定存在
