> 源代码路径: `apps/web/src/lib/pool/blade/useBladeWithdrawRequest.ts`

# useBladeWithdrawRequest Tutorial

## 1. 大白话讲讲这个hook的作用

`useBladeWithdrawRequest` *(一个React hook，用于向Blade RFQ系统发起提款请求并获取签名)* 是用于向Blade RFQ系统发起提款请求并获取签名的hook。

简单来说：
- 类似于存款请求，提款也需要先获取RFQ签名（Request for Quote）*(一种请求报价的交易模式，用户先请求报价，API返回一个签名用于后续链上交易)*
- 这个hook发送提款请求给Blade API
- API返回一个签名（tokenHolderAddress、assetAddress、assetAmount、signature等）
- 这个签名用于后续的链上交易执行提款

特点：
- 这是一个useMutation *(一个React hook，用于处理写操作，如API请求或状态更新)*（写入操作）
- 支持自动刷新：如果签名即将过期，会自动重新请求
- 与存款请求类似，但是请求的端点不同

## 2. 讲讲为什么需要封装该hook

与存款请求类似：

1. **签名时效性**：Blade的签名有有效期（good_until），过期后需要重新请求
2. **自动刷新机制**：使用useTimeout *(一个React hook，用于设置超时后执行回调)* 在签名快过期时自动重新请求
3. **Zod验证**：响应schema需要正确验证
4. **状态管理**：需要追踪refreshDelay和mutation状态

封装成hook后：
- 隐藏自动刷新逻辑
- 提供简洁的mutation接口
- 统一错误处理
- 与React Query集成

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  onError?: (error: Error) => void     // 错误回调
}
```

### 调用mutation时传入
```typescript
{
  chain_id: number                     // 链ID
  pool_token_amount_to_burn: string   // 要燃烧的池子代币数量
  asset_symbol: string                 // 资产符号
  token_holder_address: string        // 代币持有者地址
  pool_address: string                // 池子地址
}
```

### 输出（Return Value）
```typescript
{
  mutate: (payload: RfqWithdrawPayload) => void
  mutateAsync: (payload: RfqWithdrawPayload) => Promise<RfqWithdrawResponse>
  data: RfqWithdrawResponse | undefined
  isPending: boolean
  error: Error | null
  reset: () => void
}

interface RfqWithdrawResponse {
  token_holder_address: string
  pool_token_amount_to_burn: string
  asset_address: string
  asset_amount: string
  good_until: number
  signature: { v: number, r: string, s: string }
  extra_data?: string
}
```

### 执行逻辑
1. 用户调用mutation时，发送POST请求到 `${BLADE_API_HOST}/rfq/v2/withdraw`
2. API返回签名数据
3. 在onSuccess回调中计算签名剩余有效期
4. 如果remainingTime > 0，设置refreshDelay
5. useTimeout在refreshDelay后自动触发重新请求
6. 返回mutation结果

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于向Blade RFQ系统发起提款请求，获取用于执行链上提款交易的签名。签名有过期时间，hook会在签名快过期时自动重新请求。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合@tanstack/react-query的useMutation来处理写操作，使用zod来验证响应，使用@sushiswap/hooks中的useTimeout来实现自动刷新机制。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数主要是错误回调函数。调用mutation时需要传入链ID、要燃烧的池子代币数量、资产符号、代币持有者地址和池子地址。输出是标准的mutation对象。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先用useState管理refreshDelay状态，然后创建Zod schema来验证响应。接着实现useMutation，在mutationFn中发送POST请求到/rfq/v2/withdraw（注意这个端点与存款不同）并验证响应。在onMutate回调中重置refreshDelay为null，在onSuccess回调中计算签名剩余有效期并设置refreshDelay，在onError回调中调用外部的错误处理函数。最后使用useTimeout在refreshDelay到期后触发重新请求。

### 需要特别注意的约束条件

**时间戳转换**

good_until是Unix时间戳，单位是秒。但是在JavaScript中Date.now()和setTimeout使用的是毫秒，所以需要乘以1000转换为毫秒。

**使用正确的API端点**

提款请求的端点是/rfq/v2/withdraw，这与存款请求的/rfq/v2/deposit不同。确保使用正确的端点。

**API配置从constants导入**

BLADE_API_HOST和BLADE_API_KEY需要从constants导入。请求头需要设置Content-Type为application/json，如果有API_KEY还要添加X-Api-Key头。

**自动刷新机制**

自动刷新使用useTimeout实现。当签名快过期时，会自动触发重新请求。重新请求时应该使用mutation.mutate而不是mutation.mutateAsync。

### 状态管理的要点

**refreshDelay状态**

使用useState来管理refreshDelay状态。这个状态保存的是距离签名过期还有多少毫秒。

**mutation状态的自动管理**

useMutation会自动管理mutation的状态。

**生命周期流程**

当触发mutation时（onMutate），refreshDelay被重置为null。当请求成功时（onSuccess），计算签名剩余的有效时间，如果大于0就设置refreshDelay。当refreshDelay大于0时，useTimeout会触发新的mutation。

**保存调用参数**

mutation.variables保存了上次请求的参数，这些参数用于重新请求时使用。

### 完整的生产级提示词示例

```
请帮我创建一个React Mutation Hook，用于向Blade RFQ系统发起提款请求并获取签名。

这个hook需要发送POST请求获取提款签名，并且能够在签名快过期时自动重新请求。

需要使用@tanstack/react-query的useMutation来处理写操作，使用zod来验证响应，使用@sushiswap/hooks的useTimeout来实现自动刷新。

输入参数主要是错误回调函数。

调用mutation时需要传入链ID、要燃烧的池子代币数量、资产符号、代币持有者地址和池子地址。

响应包含代币持有者地址、池子代币数量、资产地址、资产数量、有效期和签名等字段。

实现步骤：
1. 使用useState管理refreshDelay状态，初始值为null
2. 创建Zod schema验证响应
3. 实现useMutation：
   - mutationFn发送POST请求到BLADE_API_HOST/rfq/v2/withdraw（注意不是deposit），设置正确的请求头，使用schema.parse验证响应
   - onMutate中重置refreshDelay为null
   - onSuccess中计算签名剩余时间，如果大于0则设置refreshDelay，否则立即重新请求
   - onError中调用外部错误处理回调
4. 使用useTimeout在refreshDelay后触发重新请求
5. 返回完整的mutation对象

重要约束：
- good_until是秒，需要乘以1000转换为毫秒
- 请求端点是/rfq/v2/withdraw，不是/deposit
- BLADE_API_HOST和BLADE_API_KEY从constants导入
- 重新请求时使用mutation.mutate而非mutation.mutateAsync

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useBladeWithdrawRequest } from '@sushiswap/hooks'

function WithdrawForm({ poolAddress, chainId }) {
  const { mutate, data, isPending, error } = useBladeWithdrawRequest({
    onError: (e) => console.error('提款请求失败:', e),
  })

  const handleWithdraw = () => {
    mutate({
      chain_id: chainId,
      pool_token_amount_to_burn: '1000000000000000000', // 1池子代币
      asset_symbol: 'WETH',
      token_holder_address: address,
      pool_address: poolAddress,
    })
  }

  return (
    <div>
      <button onClick={handleWithdraw} disabled={isPending}>
        {isPending ? '请求中...' : '提款'}
      </button>
      {data && (
        <div>
          <p>签名有效期至: {new Date(data.good_until * 1000).toLocaleString()}</p>
          <p>资产金额: {data.asset_amount}</p>
        </div>
      )}
    </div>
  )
}
```

### 常见使用场景

1. **提款流程第一步**
   - 在用户确认提款后，首先调用此hook获取签名
   - 签名获取后传递给 `useBladeWithdrawTransaction` 执行实际交易

2. **签名自动刷新**
   - 当用户需要慢慢确认提款金额时，hook会自动刷新签名
   - 适合需要用户确认的场景

3. **错误处理和重试**
   - onError回调处理错误
   - mutation支持重试机制

### Dos and Don'ts

**Dos：**
- ✅ 在调用此hook前确保用户有有效的仓位可以提取
- ✅ 传入正确的chain_id和pool_address
- ✅ 使用onError回调处理可能的错误
- ✅ 签名获取后立即使用，避免过期

**Don'ts：**
- ❌ 不要在获取签名后等待太久才执行交易，签名有时效性
- ❌ 不要忽略enabled参数，自动刷新应该在需要时启用
- ❌ 不要在onError中直接显示给用户，应该先记录日志
- ❌ 不要在签名过期后继续使用，应该重新请求
