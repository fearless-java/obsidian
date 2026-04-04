> 源代码路径: `apps/web/src/lib/pool/blade/useBladeAllowDeposit.ts`

# useBladeAllowDeposit Tutorial

## 1. 大白话讲讲这个hook的作用

`useBladeAllowDeposit` *(一个React hook，用于检查当前用户是否被允许向某个Blade池子进行存款)* 是用来检查当前用户是否被允许向某个Blade池子进行存款的hook。

简单来说：
- Blade是一个使用RFQ（Request for Quote）*(一种请求报价的交易模式，用户先请求报价再执行交易)* 系统的链上流动性协议
- 在存款之前，需要先向Blade API发送请求，检查用户的存款权限
- 这个hook会返回一个对象，包含：
  - `allow`: 是否允许存款
  - `usd_limit`: 允许存款的USD额度上限
  - `feature_single_asset_deposit`: 是否支持单资产存款
  - `min_lock_time` 或 `min_days_to_lock`: 最少锁定时间

这是一个权限检查类hook，用于在用户操作前进行条件判断。

## 2. 讲讲为什么需要封装该hook

Blade的存款权限检查涉及多个复杂性：

1. **API交互**：需要向Blade API *(Blade协议的后端API服务，用于处理RFQ请求)* 发送POST请求，而不是简单的合约调用
2. **签名验证**：请求需要包含用户的地址，API会验证签名
3. **Zod验证**：API返回的数据需要用Zod schema *(一个TypeScript优先的模式验证库)* 进行类型验证
4. **状态管理**：需要使用React Query *(一个React数据获取和缓存库)* 管理loading/error/数据状态
5. **查询键管理**：需要设计合理的queryKey用于缓存和刷新

封装成hook后：
- 统一的API调用逻辑
- 自动处理Zod schema验证
- 与React Query集成，支持缓存和自动刷新
- 错误处理标准化

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  chainId: BladeChainId *(Blade支持的链ID枚举)*              // 链ID
  poolAddress: string               // 池子地址
  enabled?: boolean                 // 是否启用，默认true
}
```

### 输出（Return Value）
```typescript
{
  data: {
    allow: boolean                              // 是否允许存款
    usd_limit: number                           // USD额度限制
    feature_single_asset_deposit: boolean       // 是否支持单资产存款
  } & (
    { min_lock_time: number } |
    { min_days_to_lock: number }
  ),
  isLoading: boolean,
  isError: boolean,
  error: Error | null,
  // ... 其他useQuery属性
}
```

### 执行逻辑
1. 从wagmi的 `useConnection` *(一个wagmi hook，用于获取当前连接的区块链钱包信息)* 获取当前用户地址
2. 检查 enabled、address、chainId、poolAddress 是否都存在
3. 构建请求体：
   ```typescript
   {
     chain_id: chainId,
     sender: address,
     pool_address: poolAddress,
   }
   ```
4. 向 `${BLADE_API_HOST}/rfq/v2/allow-deposit` 发送POST请求
5. 使用Zod schema验证返回数据
6. 返回useQuery的标准结果

### API Schema
```typescript
const rfqAllowDepositResponseBaseSchema = z.object({
  allow: z.boolean(),
  usd_limit: z.number(),
  feature_single_asset_deposit: z.boolean(),
})

// 响应可以是两种之一
export const rfqAllowDepositResponseSchema = z.union([
  rfqAllowDepositResponseBaseSchema.extend({ min_lock_time: z.number() }),
  rfqAllowDepositResponseSchema.extend({ min_days_to_lock: z.number() }),
])
```

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于检查当前连接的钱包地址是否有权限向指定的Blade池子进行存款。它通过调用Blade的RFQ API来获取存款权限信息。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合@tanstack/react-query的useQuery来管理数据获取，使用wagmi来获取当前连接的钱包信息，使用zod来验证API响应。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数包括链ID、池子地址和启用开关。输出应该包含是否允许存款、USD额度限制、是否支持单资产存款，以及最少锁定时间或最少锁定天数。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先从constants导入API主机地址和密钥，然后使用wagmi的useConnection获取当前用户地址，接着定义Zod schema来验证API响应，创建queryKey生成函数，最后实现 useQuery来发送 POST 请求并验证响应。

### 需要特别注意的约束条件

**地址验证是必须的**

必须验证address存在，否则不应该发送请求。这是一个安全相关的检查，避免向API发送无效的地址。

**API配置要正确**

BLADE_API_HOST和BLADE_API_KEY需要从constants导入，而不是硬编码。请求头需要设置Content-Type为application/json。如果有API_KEY，还需要添加X-Api-Key头。

**错误处理要完善**

当response.ok不为true时应该抛出错误。响应验证失败时zod会抛出错误，这些错误都应该被正确处理。

**queryKey要包含所有相关参数**

queryKey应该包含chainId、poolAddress和address，这样每个不同的查询都有唯一的标识，支持细粒度的缓存。格式可以是['blade', 'pool', `${chainId}:${poolAddress}`, 'allow-deposit', address]。

**enabled要检查所有必要参数**

enabled应该检查address、chainId和poolAddress是否都存在，只有都存在时才执行查询。

### 状态管理的要点

**使用React Query管理状态**

React Query会自动管理API响应的状态，包括loading、error和data。可以直接在UI中使用isLoading、isError等状态。

**设置合理的缓存时间**

可以设置staleTime为60000毫秒（1分钟），gcTime为300000毫秒（5分钟），这样可以避免过于频繁的请求，同时保证数据的时效性。

**响应数据可能有两种形状**

返回的数据可能有两种不同的形状，比如一种是min_lock_time，另一种是min_days_to_lock。这两种形状是互斥的，需要在类型层面正确处理。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于检查当前连接的钱包地址是否有权限向指定的Blade池子进行存款。

这个hook通过调用Blade的RFQ API来获取存款权限信息。

需要使用@tanstack/react-query的useQuery来管理数据获取，使用wagmi来获取钱包连接信息，使用zod来验证API响应。

输入参数包括链ID、池子地址和启用开关。

输出包含是否允许存款、USD额度限制、是否支持单资产存款，以及最少锁定时间或天数。

实现步骤：
1. 从constants导入API主机地址和密钥
2. 使用wagmi的useConnection获取当前用户地址
3. 定义Zod schema验证响应，包括基本字段和两种互斥的锁定时间字段
4. 创建queryKey生成函数，格式为['blade', 'pool', 'chainId:poolAddress', 'allow-deposit', address]
5. 实现useQuery：
   - 在queryFn中构建请求体，包含chain_id、sender和pool_address
   - 设置请求头Content-Type为application/json，如果有API_KEY还要加X-Api-Key
   - 发送POST请求到BLADE_API_HOST/rfq/v2/allow-deposit
   - 使用schema.parse验证响应
   - enabled检查所有必要参数是否都存在

重要约束：
- address必须存在才能发送请求
- 请求应该使用fetch而非axios
- 错误处理：response.ok不为true时抛出错误
- enabled参数应该检查所有必要参数

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useBladeAllowDeposit } from '@sushiswap/hooks'

function DepositButton({ poolAddress, chainId }) {
  const { data: allowance, isLoading, error } = useBladeAllowDeposit({
    chainId,
    poolAddress,
  })

  if (isLoading) return <button disabled>检查中...</button>
  if (error) return <button disabled>检查失败</button>
  if (!allowance?.allow) return <button disabled>不允许存款</button>

  return (
    <div>
      <button>存款</button>
      <p>存款限额: ${allowance.usd_limit.toLocaleString()}</p>
      {allowance.min_days_to_lock && (
        <p>最少锁定: {allowance.min_days_to_lock} 天</p>
      )}
    </div>
  )
}
```

### 常见使用场景

1. **存款按钮状态控制**
   - 根据allow字段控制存款按钮的启用/禁用
   - 显示不允许存款的原因

2. **存款限额提示**
   - 向用户展示当前池子的存款USD限额
   - 超过限额时提示用户

3. **锁定条件展示**
   - 显示最少锁定时间要求
   - 帮助用户了解存款条件

### Dos and Don'ts

**Dos：**
- ✅ 在存款操作前调用此hook检查权限
- ✅ 处理好isLoading状态，避免用户看到闪烁的UI
- ✅ 使用data?.allow || data?.allow === undefined的情况
- ✅ 合理设置staleTime减少API调用

**Don'ts：**
- ❌ 不要在每次渲染时都调用，应该依赖React Query的缓存
- ❌ 不要忽略address为空的情况，此时不应该发送请求
- ❌ 不要直接使用返回的数字类型展示大数字，需要格式化
- ❌ 不要在error发生时简单地显示"error"，应该给用户更友好的提示
