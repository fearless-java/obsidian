> 源代码路径: `apps/web/src/lib/steer/hooks/use-vaults.ts`

# useVaults Tutorial

## 1. 大白话讲讲这个hook的作用

`useVaults` *(一个React hook，用于获取SushiSwap Vault列表)* 是用来获取SushiSwap Vault列表的hook。

简单来说：
- Vault是SushiSwap的收益策略池
- 这个hook获取指定池子的所有Vault列表
- 返回每个Vault的配置信息（策略、代币对、费率等）

这是一个只读查询hook，用于展示可用的Vault列表。

## 2. 讲讲为什么需要封装该hook

获取Vault列表涉及：

1. **API调用**：需要调用Graph Client的data-api *(SushiSwap的GraphQL数据查询客户端)*
2. **参数处理**：需要传入chainId和poolAddress
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
  poolAddress: string                 // 池子地址
  shouldFetch?: boolean               // 是否获取，默认true
}
```

### 输出（Return Value）
```typescript
{
  data: VaultV1[] | null | undefined  // Vault列表
  isLoading: boolean
  isError: boolean
  error: Error | null
  // ... 其他useQuery属性
}
```

### VaultV1类型（推断）
```typescript
interface VaultV1 {
  id: string                         // Vault ID
  address: string                    // Vault地址
  strategy: string                   // 策略地址
  token0: { address: string, symbol: string }
  token1: { address: string, symbol: string }
  fee: number                        // 费率
  // ... 其他配置
}
```

### 执行逻辑
1. 检查chainId、poolAddress存在且shouldFetch为true
2. 调用getVaults API
3. 返回React Query标准结果

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于获取指定池子的Vault列表。Vault是SushiSwap提供的收益策略池子。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合@tanstack/react-query的useQuery来管理数据获取，使用@sushiswap/graph-client来获取数据。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数是一个包含chainId和poolAddress的对象，以及shouldFetch开关。输出是Vault列表，每个Vault包含ID、地址、策略地址、两种代币的信息和费率等。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先从graph-client导入获取Vaults的函数和类型，然后实现 useQuery，在queryFn中调用API，配置queryKey和enabled条件。

### 需要特别注意的约束条件

**enabled检查**

enabled检查应该是shouldFetch && chainId && poolAddress，三者都满足时才执行查询。shouldFetch默认为true。

**args参数传递**

args直接作为queryKey的一部分，这样可以确保参数变化时自动触发新的查询。

**返回可能是null**

当poolAddress不支持vault时，返回可能是null而不是空数组，UI层需要处理这种情况。

**queryKey的设计**

queryKey应该包含完整的args对象，格式可以是['vaults', args]。

### 状态管理的要点

**使用React Query管理状态**

React Query会自动管理API响应的状态。可以直接在UI中使用isLoading、isError等状态。

**shouldFetch的作用**

shouldFetch是一个控制是否执行查询的开关，默认为true。当设为false时，不会发送请求。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于获取指定池子的Vault列表。Vault是SushiSwap提供的收益策略池子。

需要使用@tanstack/react-query的useQuery来管理数据获取，使用@sushiswap/graph-client来获取数据。

输入参数是一个包含chainId和poolAddress的对象，以及shouldFetch开关（默认true）。

输出是Vault列表，每个Vault包含ID、地址、策略地址、两种代币的信息、费率和分配器地址等。

实现步骤：
1. 从@sushiswap/graph-client/data-api导入getVaults函数和VaultV1类型
2. 从@tanstack/react-query导入useQuery
3. 解构参数，设置shouldFetch默认值为true
4. 实现 useQuery：
   - queryKey使用['vaults', args]
   - queryFn调用getVaults(args as GetVaults)
   - enabled检查shouldFetch && args.chainId && args.poolAddress

重要约束：
- shouldFetch默认为true
- enabled检查shouldFetch && chainId && poolAddress
- queryKey包含完整的args对象
- 返回可能是null（当poolAddress不支持vault时）

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useVaults } from '@sushiswap/hooks'

function VaultList({ poolAddress, chainId }) {
  const { data: vaults, isLoading, error } = useVaults({
    args: { chainId, poolAddress },
  })

  if (isLoading) return <div>加载中...</div>
  if (error) return <div>加载失败</div>
  if (!vaults || vaults.length === 0) return <div>暂无Vault</div>

  return (
    <div>
      <h3>可用Vault ({vaults.length})</h3>
      {vaults.map((vault) => (
        <div key={vault.id}>
          <p>Vault: {vault.token0.symbol}/{vault.token1.symbol}</p>
          <p>费率: {vault.fee / 10000}%</p>
          <p>策略: {vault.strategy}</p>
        </div>
      ))}
    </div>
  )
}
```

### 常见使用场景

1. **Vault列表展示**
   - 显示指定池子的所有Vault
   - 展示策略和费率信息

2. **策略选择**
   - 用户选择使用的策略
   - 用于存款操作

3. **收益对比**
   - 对比不同策略的收益
   - 辅助决策

### Dos and Don'ts

**Dos：**
- ✅ 使用shouldFetch控制是否获取
- ✅ 处理好null情况（不支持Vault的池子）
- ✅ 传入完整的args对象
- ✅ 设计合理的queryKey

**Don'ts：**
- ❌ 不要在poolAddress不支持时期望有数据返回
- ❌ 不要忽略isLoading和error状态
- ❌ 不要频繁调用，设置合理的缓存
- ❌ 不要假设数据一定存在
