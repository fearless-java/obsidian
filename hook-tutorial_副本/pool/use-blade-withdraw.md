> 源代码路径: `apps/web/src/lib/pool/blade/useBladeWithdraw/useBladeWithdraw.ts`

# useBladeWithdraw (useBladeWithdrawTransaction) Tutorial

## 1. 大白话讲讲这个hook的作用

`useBladeWithdrawTransaction` *(一个React hook，用于执行Blade池子赎回交易)* 是用于执行Blade池子赎回（Withdraw）交易的hook。

简单来说：
- 用户想要从Blade池子取出流动性
- 需要先通过 `useBladeWithdrawRequest` *(一个React hook，用于向Blade RFQ系统发起提款请求并获取签名)* 获取提款签名
- 这个hook负责构造并发送提款交易
- 根据不同的池子ABI类型选择不同的交易构造方式
- 追踪交易的Pending和Confirm状态

这是一个写操作hook，与存款类似，但是用于赎回资产。

## 2. 讲讲为什么需要封装该hook

与存款类似，提款交易也涉及多种池子类型和复杂的交易构造：

1. **ABI多样性**：不同池子类型（Blade vs Clipper各版本）使用不同的提款方法
2. **交易构造差异**：需要根据pool.abi *(池子的Application Binary Interface类型，决定了如何构造交易)* 选择正确的提款函数
3. **状态追踪**：isPending、isConfirming、isConfirmed
4. **错误处理**：交易失败需要正确处理

封装成hook后：
- 隐藏不同ABI的交易构造差异
- 提供统一的提款接口
- 自动处理Pending/Confirm状态
- 与wagmi集成

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  pool: BladePool *(包含池子地址、链ID和ABI类型的池子对象)*                    // 池子信息，包含abi类型
  onSuccess?: (hash: Hex) => void    // 交易成功回调
  onError?: (error: Error) => void   // 交易失败回调
}
```

### 输出（Return Value）
```typescript
{
  mutate: (args: Omit<WithdrawVariablesGetterArgs, 'poolAddress'>) => void
  mutateAsync: (args: Omit<WithdrawVariablesGetterArgs, 'poolAddress'>) => Promise<Hex>
  writeContract: (args: Omit<WithdrawVariablesGetterArgs, 'poolAddress'>) => void
  hash: Hex | undefined *(一个十六进制字符串类型，用于表示交易哈希)*
  isWritePending: boolean
  isConfirming: boolean
  isConfirmed: boolean
  isPending: boolean
  error: Error | null
  reset: () => void
}
```

### WithdrawVariablesGetterArgs
```typescript
interface WithdrawVariablesGetterArgs {
  poolAddress: string              // 池子地址（自动填充）
  tokenHolderAddress: string
  poolTokenAmountToBurn: string
  assetSymbol: string
  // ... 其他参数
}
```

### 执行逻辑
1. 使用useWriteContract *(一个wagmi hook，用于发送写合约交易)* 获取写合约方法
2. 使用useWaitForTransactionReceipt *(一个wagmi hook，用于等待和追踪交易确认状态)* 追踪确认状态
3. 根据pool.abi选择对应的提款交易构造函数：
   ```typescript
   const withdrawWriteVariablesMap = {
     'ClipperDirectExchangeV0': clipperV0Withdraw,
     'ClipperDirectExchangeV1': clipperWithdraw,
     'ClipperVerifiedExchange': clipperWithdraw,
     // ...
     'BladeVerifiedExchange': bladeWithdraw,
     'BladeApproximateExchange': bladeWithdraw,
   }
   ```
4. mutationFn中补充poolAddress并调用选中的函数
5. 使用writeContractAsync发送交易
6. 合并状态返回

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于执行Blade池子的提款交易。它根据池子的ABI类型选择正确的交易构造方式，发送交易并追踪状态。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合wagmi的useWriteContract发送交易，用useWaitForTransactionReceipt追踪确认状态，使用@tanstack/react-query管理mutation状态。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数包括池子对象（包含地址、链ID和ABI类型）、成功回调和错误回调。输出应该包含mutate和mutateAsync方法、交易哈希、各种pending和confirm状态，以及错误状态。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先从useWriteContract获取交易发送相关的方法和状态，然后从useWaitForTransactionReceipt获取确认状态，接着创建一个映射表来存储不同ABI类型对应的提款交易构造函数，然后创建mutation来处理交易发送和状态追踪，在mutationFn中补充poolAddress，最后合并所有状态并返回。

### 需要特别注意的约束条件

**根据ABI类型选择正确的交易构造函数**

与存款类似，不同的ABI类型需要调用不同的提款函数。比如ClipperDirectExchangeV0需要调用clipperV0Withdraw，而BladeVerifiedExchange需要调用bladeWithdraw。这个映射关系必须正确。

**poolAddress的自动补充**

提款函数接收的参数中不包含poolAddress（因为使用了Omit类型），poolAddress需要在mutationFn中从pool.address自动补充进去。

**状态获取的来源**

交易哈希（hash）只能从useWriteContract的data获取。isConfirming需要使用useWaitForTransactionReceipt的isLoading。isConfirmed需要使用useWaitForTransactionReceipt的isSuccess。

**错误合并**

错误可能来自三个地方：writeContract的错误、waitForTransactionReceipt的错误、以及mutation本身的错误。这三种错误需要合并处理。

**reset要重置所有状态**

reset函数应该同时重置writeContract的状态和withdrawTransaction的状态。

### 状态管理的要点

**useWriteContract管理发送状态**

useWriteContract负责管理交易发送的相关状态。

**useWaitForTransactionReceipt管理确认状态**

这个hook负责追踪交易是否被打包进区块。

**isPending的计算**

isPending是isWritePending和isConfirming的逻辑或，表示交易正在处理中（包括发送和确认两个阶段）。

**保存调用参数**

mutation.variables保存了最后一次调用的参数，可以用于调试或者重试。

### 完整的生产级提示词示例

```
请帮我创建一个React Mutation Hook，用于执行Blade池子的提款交易。

这个hook需要根据池子的ABI类型选择正确的交易构造方式，发送交易并追踪状态。

需要使用wagmi的useWriteContract发送交易，用useWaitForTransactionReceipt追踪确认状态，使用@tanstack/react-query管理mutation。

输入参数包括池子对象（包含地址、链ID和ABI类型）、成功回调和错误回调。

输出包含mutate和mutateAsync方法、交易哈希、各种pending和confirm状态，以及错误状态。

实现步骤：
1. 从useWriteContract获取mutate、mutateAsync、data（hash）、isPending、error、reset
2. 从useWaitForTransactionReceipt获取isLoading（对应isConfirming）、isSuccess（对应isConfirmed）、error（对应confirmError）
3. 创建一个映射表，键是ABI类型字符串，值是对应的提款交易构造函数
4. 创建withdrawTransaction mutation：
   - 在mutationFn中，首先补充poolAddress（从pool.address获取）
   - 然后调用对应ABI的交易构造函数获取交易参数
   - 最后调用writeContractAsync发送交易
5. 合并返回所有状态

重要约束：
- 必须根据pool.abi选择正确的交易构造函数
- poolAddress在mutationFn中自动补充
- hash只能从useWriteContract获取
- isConfirming需要使用isLoading，isConfirmed需要使用isSuccess
- 错误合并要考虑三种来源
- reset应该同时重置所有状态

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useBladeWithdrawTransaction } from '@sushiswap/hooks'
import { useBladeWithdrawRequest } from '@sushiswap/hooks'

function WithdrawForm({ pool }) {
  const { mutate: requestMutate, data: signatureData } = useBladeWithdrawRequest(...)
  const {
    mutate,
    hash,
    isPending,
    isConfirmed,
  } = useBladeWithdrawTransaction({
    pool,
    onSuccess: (hash) => {
      console.log('提款成功:', hash)
    },
  })

  const handleWithdraw = () => {
    // 先请求签名
    requestMutate({ ... })
  }

  const handleExecute = () => {
    // 用签名执行交易
    if (signatureData) {
      mutate({
        tokenHolderAddress: address,
        poolTokenAmountToBurn: '1000000000000000000',
        assetSymbol: 'WETH',
      })
    }
  }

  return (
    <div>
      <button onClick={handleWithdraw} disabled={isPending}>
        {isPending ? '处理中...' : '提款'}
      </button>
      {hash && <p>交易哈希: {hash}</p>}
      {isConfirmed && <p>交易已确认!</p>}
    </div>
  )
}
```

### 常见使用场景

1. **提款流程第二步**
   - 在 `useBladeWithdrawRequest` 获取签名后执行
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
- ✅ 先调用 `useBladeWithdrawRequest` 获取签名
- ✅ 传入完整的pool对象，包含abi类型
- ✅ 使用isPending状态禁用按钮防止重复提交
- ✅ 使用onSuccess回调处理交易完成后的逻辑

**Don'ts：**
- ❌ 不要在签名过期后使用，应该重新请求
- ❌ 不要忽略isPending状态，可能导致重复交易
- ❌ 不要在onError中直接throw，这会被mutation捕获
- ❌ 不要使用过期的pool.abi，新池子可能有新的ABI类型
