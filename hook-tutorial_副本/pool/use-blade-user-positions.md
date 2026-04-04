> 源代码路径: `apps/web/src/lib/pool/blade/useBladeUserPositions.ts`

# useBladeUserPositions Tutorial

## 1. 大白话讲讲这个hook的作用

`useBladeUserPositions` *(一个React hook，用于获取用户在某个链上的所有Blade仓位)* 是用来获取用户在某个链上的所有Blade仓位（positions）的hook。

简单来说：
- 查询指定用户在Blade协议上的所有流动性仓位
- 返回每个仓位的详细信息，包括：
  - 存入的代币和数量
  - 池子地址
  - 当前的权益代币数量
  - 锁定期限等

这是一个只读的查询hook，用于展示用户现有的Blade仓位信息。

## 2. 讲讲为什么需要封装该hook

获取Blade仓位信息涉及多个复杂性：

1. **API调用**：需要调用Graph Client的data-api *(SushiSwap的GraphQL数据查询客户端)*
2. **类型安全**：返回的BladePositions类型需要正确定义
3. **查询控制**：需要支持enabled参数控制是否执行
4. **React Query集成**：需要管理loading/error状态和缓存

封装成hook后：
- 统一的API调用逻辑
- 类型安全的返回数据
- 与React Query集成，支持缓存和刷新
- 简洁的参数接口

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  chainId: BladeChainId *(Blade支持的链ID枚举)*              // 链ID
  user?: EvmAddress *(以太坊虚拟机兼容的地址类型)*                   // 用户地址
  enabled?: boolean                   // 是否启用，默认true
}
```

### 输出（Return Value）
```typescript
{
  data: BladePositions | undefined    // 用户仓位数据
  isLoading: boolean
  isError: boolean
  error: Error | null
  // ... 其他useQuery属性
}
```

### BladePositions类型（推断）
```typescript
interface BladePosition {
  id: string
  pool: {
    address: string
    token0: { address: string, symbol: string }
    token1: { address: string, symbol: string }
  }
  token0Balance: string
  token1Balance: string
  poolTokenBalance: string
  lockedUntil?: number
  // ... 其他字段
}

type BladePositions = BladePosition[]
```

### 执行逻辑
1. 检查enabled、chainId、user是否都存在
2. 如果user存在，调用getBladePositions API
3. 如果user不存在，返回skipToken *(一个React Query的特殊标记，用于跳过本次查询)*（跳过查询）
4. 返回React Query标准结果

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于获取指定用户在Blade协议上的所有流动性仓位信息。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合@tanstack/react-query的useQuery来管理数据获取，使用@sushiswap/graph-client来获取数据。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数包括链ID、用户地址和启用开关。输出应该是一个仓位数组，每个仓位包含池子信息、两个代币的余额、池子代币余额和锁定时间等信息。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先从graph-client导入获取仓位数据的函数，然后实现useQuery，在queryFn中调用API，配置queryKey和enabled条件。

### 需要特别注意的约束条件

**用户地址必须存在**

必须检查user是否存在才能发送请求。如果user为空，应该跳过查询而不是发送无效请求。

**enabled与其他参数的关系**

enabled默认是true，但查询是否真正执行还需要chainId和user都存在。所以enabled的检查应该是enabled && chainId && user，三者同时为真才执行查询。

**queryKey要包含所有相关参数**

queryKey应该包含chainId和user，这样每个不同的查询都有唯一的标识。格式可以是['blade', 'positions', { chainId, user }]。

**使用skipToken跳过无效查询**

当user为空时，应该使用skipToken（React Query提供的一个特殊标记）来跳过本次查询，而不是返回undefined或者其他默认值。

### 状态管理的要点

**使用React Query管理状态**

React Query会自动管理API响应的状态，包括loading、error和data。可以直接在UI中使用isLoading、isError等状态。

**设置合理的缓存时间**

可以设置staleTime和gcTime来控制缓存策略，这样可以在数据时效性和性能之间取得平衡。

**queryKey的重要性**

queryKey包含所有相关参数，确保每个不同的查询都有唯一的缓存标识。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于获取指定用户在Blade协议上的所有流动性仓位信息。

需要使用@tanstack/react-query的useQuery来管理数据获取，使用@sushiswap/graph-client来获取数据。

输入参数包括链ID、用户地址（可以是undefined）和启用开关（默认true）。

输出是一个仓位数组，每个仓位包含ID、池子信息（地址、链ID、两种代币的信息）、各代币余额、池子代币余额和锁定时间等。

实现步骤：
1. 从@sushiswap/graph-client/data-api导入getBladePositions函数
2. 从@tanstack/react-query导入useQuery和skipToken
3. 解构参数：chainId、user、enabled = true
4. 实现useQuery：
   - queryKey使用['blade', 'positions', { chainId, user }]
   - queryFn调用getBladePositions({ chainId, user })
   - enabled检查三者是否都存在
   - 当user为空时返回skipToken

重要约束：
- user必须存在才能执行查询
- enabled应该与user和chainId同时检查
- 当user为空时使用skipToken而非返回undefined
- queryKey包含所有相关参数

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useBladeUserPositions } from '@sushiswap/hooks'
import { useConnection } from 'wagmi'

function MyPositions() {
  const { address } = useConnection()
  const { data: positions, isLoading, error } = useBladeUserPositions({
    chainId: 1,
    user: address,
  })

  if (isLoading) return <div>加载中...</div>
  if (error) return <div>加载失败</div>
  if (!positions || positions.length === 0) return <div>暂无仓位</div>

  return (
    <div>
      <h3>我的Blade仓位 ({positions.length})</h3>
      {positions.map((position) => (
        <div key={position.id}>
          <p>池子: {position.pool.token0.symbol}/{position.pool.token1.symbol}</p>
          <p>Token0余额: {position.token0Balance}</p>
          <p>Token1余额: {position.token1Balance}</p>
        </div>
      ))}
    </div>
  )
}
```

### 常见使用场景

1. **用户持仓页面**
   - 展示用户在所有Blade池子的仓位
   - 显示每个仓位的价值和锁定状态

2. **跨链仓位查询**
   - 在不同链上查询同一用户的仓位
   - 汇总展示用户的整体Blade敞口

3. **仓位详情页面**
   - 获取特定池子的用户仓位
   - 用于管理或赎回操作

### Dos and Don'ts

**Dos：**
- ✅ 确保user地址存在后再调用
- ✅ 使用enabled参数控制是否启用
- ✅ 处理好空仓位的情况（返回空数组）
- ✅ 考虑设置合理的缓存时间

**Don'ts：**
- ❌ 不要在user不存在时传入undefined，应该依赖skipToken
- ❌ 不要忽略isLoading状态
- ❌ 不要直接渲染大数字，需要格式化
- ❌ 不要频繁轮询，应该依赖React Query的缓存策略
