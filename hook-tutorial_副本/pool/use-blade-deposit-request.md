> 源代码路径: `apps/web/src/lib/pool/blade/useBladeDepositRequest.ts`

# useBladeDepositRequest Tutorial

## 1. 大白话讲讲这个hook的作用

`useBladeDepositRequest` *(一个React hook，用于向Blade RFQ系统发起存款请求并获取签名)* 是用于向Blade RFQ系统发起存款请求并获取签名的hook。

简单来说：
- Blade使用RFQ（Request for Quote）*(一种请求报价的交易模式，用户先请求报价，API返回一个签名用于后续链上交易)* 模式，用户先请求报价
- 这个hook发送存款请求给Blade API
- API返回一个签名（sender、pool_tokens、good_until、signature等）
- 这个签名用于后续的区块链交易执行存款

特点：
- 这是一个useMutation *(一个React hook，用于处理写操作，如API请求或状态更新)*（写入操作）
- 支持自动刷新：如果签名即将过期（good_until），会自动重新请求
- 返回的签名数据用于构造最终的链上交易

## 2. 讲讲为什么需要封装该hook

RFQ存款请求涉及多个复杂性：

1. **签名时效性**：Blade的签名有有效期（good_until），过期后需要重新请求
2. **自动刷新机制**：使用useTimeout *(一个React hook，用于设置超时后执行回调)* 在签名快过期时自动重新请求
3. **Zod验证**：复杂的响应schema需要正确验证
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
  onError?: (e: Error) => void      // 错误回调
  enabled?: boolean                 // 是否启用自动刷新，默认false
}
```

### 调用mutation时传入
```typescript
{
  sender: string                    // 用户地址
  deposit: { [address: string]: string }  // 存款金额，key是代币地址
  chain_id: number                  // 链ID
  single_asset?: boolean            // 是否单资产存款
  pool_address: string              // 池子地址
  lock_time?: number                // 锁定时间
  days_to_lock?: number             // 锁定天数
}
```

### 输出（Return Value）
```typescript
{
  mutate: (payload: RfqDepositPayload) => void
  mutateAsync: (payload: RfqDepositPayload) => Promise<RfqDepositResponse>
  data: RfqDepositResponse | undefined
  isPending: boolean
  error: Error | null
  reset: () => void
  // ... 其他mutation属性
}

interface RfqDepositResponse {
  sender: string
  pool_tokens: string
  good_until: number
  signature: { v: number, r: string, s: string }
  clipper_exchange_address: string
  extra_data?: string
  // deposit_amounts或amount+token
  deposit_amounts?: string[]
  amount?: string
  token?: string
  // lock_time或n_days
  lock_time?: number
  n_days?: number
}
```

### 执行逻辑
1. 用户调用mutation时，发送POST请求到 `${BLADE_API_HOST}/rfq/v2/deposit`
2. API返回签名数据
3. 在onSuccess回调中计算签名剩余有效期：
   ```typescript
   const remainingTime = data.good_until * 1000 - Date.now()
   ```
4. 如果remainingTime > 0，设置refreshDelay
5. useTimeout在refreshDelay后自动触发重新请求
6. 返回mutation结果

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于向Blade RFQ系统发起存款请求，获取用于执行链上存款交易的签名。签名有过期时间，hook会在签名快过期时自动重新请求。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合@tanstack/react-query的useMutation来处理写操作，使用zod来验证复杂的响应结构，使用@sushiswap/hooks中的useTimeout来实现自动刷新机制。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数包括错误回调函数和启用开关（控制是否自动刷新）。调用mutation时需要传入发送者地址、存款金额（代币地址到金额的映射）、链ID、池子地址，以及可选的单资产存款标识和锁定参数。输出是标准的mutation对象，包含mutate和mutateAsync方法，以及data、isPending、error等状态。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先用useState管理refreshDelay状态，然后创建Zod schema来验证响应。接着实现useMutation，在mutationFn中发送POST请求到Blade API并验证响应。在onMutate回调中重置refreshDelay为null，在onSuccess回调中计算签名剩余有效期并设置refreshDelay，在onError回调中调用外部的错误处理函数。最后使用useTimeout在refreshDelay到期后触发重新请求。

### 需要特别注意的约束条件

**时间戳转换很重要**

good_until是Unix时间戳，单位是秒。但是在JavaScript中Date.now()和setTimeout使用的是毫秒，所以需要乘以1000转换为毫秒。这个转换很容易忘记，会导致时间比较出错。

**自动刷新由enabled控制**

自动刷新机制只在enabled为true时才生效。默认应该是false，这样用户可以先获取签名然后再决定是否启用自动刷新。

**使用正确的刷新方法**

当签名快过期需要重新请求时，应该使用mutation.mutate而不是mutation.mutateAsync。因为mutate是同步触发请求的方法，而mutateAsync返回Promise。

**每次mutation时重置refreshDelay**

在onMutate回调中应该重置refreshDelay为null，这样可以避免在新的请求还在进行中时触发了旧的timeout。

**Zod schema要正确定义**

请求体和响应体都有复杂的结构。响应可能有单资产或多资产的存款方式，也可能有lock_time或days_to_lock两种锁定时间的表示方式。需要正确处理这四种组合。

**API配置从constants导入**

BLADE_API_HOST和BLADE_API_KEY需要从constants导入，而不是硬编码。请求头需要设置Content-Type为application/json，如果有API_KEY还要添加X-Api-Key头。

### 状态管理的要点

**refreshDelay状态**

使用useState来管理refreshDelay状态。这个状态保存的是距离签名过期还有多少毫秒。

**mutation状态的自动管理**

useMutation会自动管理mutation的状态，包括isPending、error等。不需要手动管理这些状态。

**生命周期流程**

当触发mutation时（onMutate），refreshDelay被重置为null。当请求成功时（onSuccess），计算签名剩余的有效时间，如果大于0就设置refreshDelay。当refreshDelay大于0且enabled为true时，useTimeout会触发新的mutation。

**refreshDelay为null时不触发timeout**

当refreshDelay为null时，不应该触发timeout。这个检查在useTimeout的实现中应该体现。

**保存上次请求参数**

mutation.variables保存了上次请求的参数，这些参数用于重新请求时使用。

### 完整的生产级提示词示例

```
请帮我创建一个React Mutation Hook，用于向Blade RFQ系统发起存款请求并获取签名。

这个hook需要发送POST请求获取存款签名，并且能够在签名快过期时自动重新请求。

需要使用@tanstack/react-query的useMutation来处理写操作，使用zod来验证响应，使用@sushiswap/hooks的useTimeout来实现自动刷新。

输入参数包括错误回调函数和启用开关（默认false）。

调用mutation时需要传入发送者地址、存款金额（代币地址到金额的映射）、链ID、池子地址，以及可选的单资产存款标识和锁定参数。

响应包含发送者、池子代币、有效期、签名等字段，其中签名有过期时间。

实现步骤：
1. 使用useState管理refreshDelay状态，初始值为null
2. 创建Zod schema验证响应，处理单资产/多资产和lock_time/days_to_lock的组合
3. 实现useMutation：
   - mutationFn发送POST请求到BLADE_API_HOST/rfq/v2/deposit，设置正确的请求头，使用schema.parse验证响应
   - onMutate中重置refreshDelay为null
   - onSuccess中计算签名剩余时间，如果大于0且enabled为true则设置refreshDelay，否则立即重新请求
   - onError中调用外部错误处理回调
4. 使用useTimeout在refreshDelay后触发重新请求
5. 返回完整的mutation对象

重要约束：
- good_until是秒，需要乘以1000转换为毫秒
- 自动刷新只在enabled为true时生效
- 重新请求时使用mutation.mutate而非mutation.mutateAsync
- 每次mutation时重置refreshDelay为null
- BLADE_API_HOST和BLADE_API_KEY从constants导入

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useBladeDepositRequest } from '@sushiswap/hooks'

function DepositForm({ poolAddress, chainId }) {
  const { mutate, data, isPending, error } = useBladeDepositRequest({
    onError: (e) => console.error('存款请求失败:', e),
    enabled: true, // 启用自动刷新
  })

  const handleDeposit = () => {
    mutate({
      sender: address,
      deposit: { [WETH_ADDRESS]: '1000000000000000000' }, // 1 WETH
      chain_id: chainId,
      pool_address: poolAddress,
      single_asset: true,
      days_to_lock: 7,
    })
  }

  return (
    <div>
      <button onClick={handleDeposit} disabled={isPending}>
        {isPending ? '请求中...' : '存款'}
      </button>
      {data && (
        <div>
          <p>签名有效期至: {new Date(data.good_until * 1000).toLocaleString()}</p>
          <p>池子代币: {data.pool_tokens}</p>
        </div>
      )}
    </div>
  )
}
```

### 常见使用场景

1. **存款流程第一步**
   - 在用户确认存款后，首先调用此hook获取签名
   - 签名获取后传递给 `useBladeDepositTransaction` 执行实际交易

2. **签名自动刷新**
   - 当enabled为true时，hook会自动在签名过期前刷新
   - 适合需要用户慢慢确认存款金额的场景

3. **错误处理和重试**
   - onError回调处理错误
   - mutation支持重试机制

### Dos and Don'ts

**Dos：**
- ✅ 在调用此hook前先检查 `useBladeAllowDeposit`
- ✅ 传入正确的chain_id和pool_address
- ✅ 使用onError回调处理可能的错误
- ✅ 签名获取后立即使用，避免过期

**Don'ts：**
- ❌ 不要在获取签名后等待太久才执行交易，签名有时效性
- ❌ 不要忽略enabled参数，不启用自动刷新可能导致签名过期
- ❌ 不要在onError中直接显示给用户，应该先记录日志
- ❌ 不要在签名过期后继续使用，应该重新请求
