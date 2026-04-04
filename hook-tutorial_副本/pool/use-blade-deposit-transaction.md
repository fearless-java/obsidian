> 源代码路径: `apps/web/src/lib/pool/blade/useBladeDepositTransaction/useBladeDepositTransaction.ts`

# useBladeDepositTransaction Tutorial

## 1. 大白话讲讲这个hook的作用

`useBladeDepositTransaction` *(一个React hook，用于执行Blade池子存款交易)* 是用于执行Blade池子存款交易的hook。

简单来说：
- 在获取了Blade RFQ签名后，需要实际执行链上交易完成存款
- 这个hook负责构造并发送存款交易
- 根据不同的池子ABI类型（Blade、Clipper等），选择不同的交易构造方式 *(不同版本的池子合约有不同的存款方法实现)*
- 同时追踪交易Pending和Confirm状态

这是一个写操作hook，用于最终执行用户的存款动作。

## 2. 讲讲为什么需要封装该hook

Blade池子有多种实现（Blade、Clipper的不同版本），它们的存款交易构造方式不同：

1. **ABI多样性**：不同池子类型使用不同的合约ABI *(Application Binary Interface，合约的接口定义)*
2. **交易构造差异**：Blade使用Packed方式，Clipper有不同版本
3. **状态追踪**：需要追踪isPending、isConfirming、isConfirmed
4. **错误处理**：交易失败需要正确处理

封装成hook后：
- 隐藏不同ABI的交易构造差异
- 提供统一的接口
- 自动处理Pending/Confirm状态
- 与wagmi的useWriteContract和useWaitForTransactionReceipt集成

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  pool: BladePool *(包含池子地址、链ID和ABI类型的池子对象)*                    // 池子信息，包含abi类型
  onSuccess?: (hash: Hex) => void    // 交易成功回调
  onError?: (error: Error) => void   // 交易失败回调
}
```

### BladePool类型
```typescript
interface BladePool {
  address: string
  chainId: number
  abi: 'BladeVerifiedExchange' | 'BladeApproximateExchange' |
       'ClipperDirectExchangeV0' | 'ClipperDirectExchangeV1' |
       'ClipperVerifiedExchange' | 'ClipperCaravelExchange' |
       'ClipperVerifiedCaravelExchange' | 'ClipperApproximateCaravelExchange' |
       'ClipperPackedVerifiedExchange' | 'ClipperPackedExchange' |
       'ClipperPackedOracleVerifiedExchange'
}
```

### 输出（Return Value）
```typescript
{
  mutate: (args: DepositVariablesGetterArgs) => void
  mutateAsync: (args: DepositVariablesGetterArgs) => Promise<Hex>
  writeContract: (args: DepositVariablesGetterArgs) => void
  hash: Hex | undefined *(一个十六进制字符串类型，用于表示交易哈希)*
  isWritePending: boolean
  isConfirming: boolean
  isConfirmed: boolean
  isPending: boolean              // isWritePending || isConfirming
  error: Error | null
  reset: () => void
}
```

### DepositVariablesGetterArgs
```typescript
interface DepositVariablesGetterArgs {
  depositSignature: {
    sender: string
    pool_tokens: string
    good_until: number
    signature: { v: number, r: string, s: string }
    // ... 其他签名数据
  }
  // ... 其他参数
}
```

### 执行逻辑
1. 使用useWriteContract *(一个wagmi hook，用于发送写合约交易)* 获取写合约方法
2. 使用useWaitForTransactionReceipt *(一个wagmi hook，用于等待和追踪交易确认状态)* 追踪确认状态
3. 根据pool.abi选择对应的交易构造函数：
   ```typescript
   const depositWriteVariablesMap = {
     'ClipperDirectExchangeV0': clipperV0TransmitAndDeposit,
     'ClipperDirectExchangeV1': clipperTransmitAndDeposit,
     'ClipperVerifiedExchange': clipperTransmitAndDeposit,
     // ...
     'BladeVerifiedExchange': bladePackedTransmitAndDeposit,
     'BladeApproximateExchange': bladeTransmitAndDeposit,
   }
   ```
4. 在mutationFn中调用选中的函数构造交易变量
5. 使用writeContractAsync发送交易
6. 合并isWritePending和isConfirming为isPending

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于执行Blade池子的存款交易。它根据池子的ABI类型选择正确的交易构造方式，发送交易并追踪状态。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合wagmi的useWriteContract发送交易，用useWaitForTransactionReceipt追踪确认状态，使用@tanstack/react-query管理mutation状态。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数包括池子对象（包含地址、链ID和ABI类型）、成功回调和错误回调。输出应该包含mutate和mutateAsync方法、交易哈希、各种pending和confirm状态，以及错误状态。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先从useWriteContract获取交易发送相关的方法和状态，然后从useWaitForTransactionReceipt获取确认状态，接着创建一个映射表来存储不同ABI类型对应的交易构造函数，然后创建mutation来处理交易发送和状态追踪，最后合并所有状态并返回。

### 需要特别注意的约束条件

**根据ABI类型选择正确的交易构造函数**

Blade池子有多种实现，不同的ABI类型需要调用不同的交易构造函数。比如ClipperDirectExchangeV0需要调用clipperV0TransmitAndDeposit，而BladeVerifiedExchange需要调用bladePackedTransmitAndDeposit。这个映射关系必须正确。

**状态获取的来源**

交易哈希（hash）只能从useWriteContract的data获取。isConfirming（是否正在确认）需要使用useWaitForTransactionReceipt的isLoading。isConfirmed（是否已确认）需要使用useWaitForTransactionReceipt的isSuccess。

**错误合并**

错误可能来自三个地方：writeContract的错误、waitForTransactionReceipt的错误、以及mutation本身的错误。这三种错误需要合并处理，格式是writeError || confirmError || depositTransaction.error。

**reset要重置所有状态**

reset函数应该同时重置writeContract的状态和depositTransaction的状态，而不是只重置一个。

**映射表要完整**

depositWriteVariablesMap应该是一个Record类型，映射所有支持的ABI类型到对应的交易构造函数。

### 状态管理的要点

**useWriteContract管理发送状态**

useWriteContract负责管理交易发送的相关状态，包括isPending、error等。

**useWaitForTransactionReceipt管理确认状态**

这个hook负责追踪交易是否被打包进区块，状态包括isLoading（对应isConfirming）和isSuccess（对应isConfirmed）。

**isPending的计算**

isPending是isWritePending和isConfirming的逻辑或，表示交易正在处理中（包括发送和确认两个阶段）。

**onSuccess回调的时机**

onSuccess回调应该在交易确认后执行，而不是在交易发送后就执行。

**保存调用参数**

mutation.variables保存了最后一次调用的参数，可以用于调试或者重试。

### 完整的生产级提示词示例

```
请帮我创建一个React Mutation Hook，用于执行Blade池子的存款交易。

这个hook需要根据池子的ABI类型选择正确的交易构造方式，发送交易并追踪状态。

需要使用wagmi的useWriteContract发送交易，用useWaitForTransactionReceipt追踪确认状态，使用@tanstack/react-query管理mutation。

输入参数包括池子对象（包含地址、链ID和ABI类型）、成功回调和错误回调。

输出包含mutate和mutateAsync方法、交易哈希、各种pending和confirm状态，以及错误状态。

实现步骤：
1. 从useWriteContract获取mutate、mutateAsync、data（hash）、isPending、error、reset
2. 从useWaitForTransactionReceipt获取isLoading（对应isConfirming）、isSuccess（对应isConfirmed）、error（对应confirmError）
3. 创建一个映射表，键是ABI类型字符串，值是对应的交易构造函数
4. 创建depositTransaction mutation：
   - 在mutationFn中调用对应ABI的交易构造函数获取交易参数
   - 然后调用writeContractAsync发送交易
5. 合并返回：
   - 包含所有mutation属性
   - hash从useWriteContract获取
   - isConfirming从useWaitForTransactionReceipt的isLoading获取
   - isConfirmed从useWaitForTransactionReceipt的isSuccess获取
   - isPending是isWritePending和isConfirming的逻辑或
   - error是三种错误的合并
   - reset同时重置writeContract和mutation

重要约束：
- 必须根据pool.abi选择正确的交易构造函数
- hash只能从useWriteContract获取
- isConfirming需要使用isLoading，isConfirmed需要使用isSuccess
- 错误合并要考虑三种来源
- reset应该同时重置所有状态

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useBladeDepositTransaction } from '@sushiswap/hooks'
import { useBladeDepositRequest } from '@sushiswap/hooks'

function DepositForm({ pool }) {
  const { mutate: requestMutate, data: signatureData } = useBladeDepositRequest(...)
  const {
    mutate,
    hash,
    isPending,
    isConfirmed,
  } = useBladeDepositTransaction({
    pool,
    onSuccess: (hash) => {
      console.log('存款成功:', hash)
    },
    onError: (error) => {
      console.error('存款失败:', error)
    },
  })

  const handleDeposit = () => {
    // 先请求签名
    requestMutate({ ... })
  }

  const handleExecute = () => {
    // 用签名执行交易
    if (signatureData) {
      mutate({ depositSignature: signatureData, ... })
    }
  }

  return (
    <div>
      <button onClick={handleDeposit} disabled={isPending}>
        {isPending ? '处理中...' : '存款'}
      </button>
      {hash && <p>交易哈希: {hash}</p>}
      {isConfirmed && <p>交易已确认!</p>}
    </div>
  )
}
```

### 常见使用场景

1. **存款流程第二步**
   - 在 `useBladeDepositRequest` 获取签名后执行
   - 完成从签名到链上交易的转换

2. **交易状态追踪**
   - isPending 显示交易是否在处理中
   - isConfirming 显示是否在等待确认
   - isConfirmed 显示交易是否完成

3. **多池子支持**
   - 自动适配不同ABI类型的池子
   - 统一接口简化UI开发

### Dos and Don'ts

**Dos：**
- ✅ 先调用 `useBladeDepositRequest` 获取签名
- ✅ 传入完整的pool对象，包含abi类型
- ✅ 使用isPending状态禁用按钮防止重复提交
- ✅ 使用onSuccess回调处理交易完成后的逻辑

**Don'ts：**
- ❌ 不要在签名过期后使用，应该重新请求
- ❌ 不要忽略isPending状态，可能导致重复交易
- ❌ 不要在onError中直接throw，这会被mutation捕获
- ❌ 不要使用过期的pool.abi，新池子可能有新的ABI类型
