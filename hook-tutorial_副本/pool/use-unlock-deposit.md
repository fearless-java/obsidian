> 源代码路径: `apps/web/src/lib/pool/blade/useUnlockDeposit.ts`

# useUnlockDeposit Tutorial

## 1. 大白话讲讲这个hook的作用

`useUnlockDeposit` *(一个React hook，用于执行Blade池子解锁存款交易)* 是用于执行Blade池子"解锁存款"交易的hook。

简单来说：
- Blade池子有锁定期机制 *(用户存款后需要锁定一段时间才能完全提取)*
- 用户存款后，在锁定期结束前不能完全提取
- 这个hook用于执行"解锁"操作，解除存款的锁定状态
- 与普通存款不同，这是一个特殊的解锁交易

这是一个写操作hook，用于处理锁定期结束后的解锁操作。

## 2. 讲讲为什么需要封装该hook

解锁交易涉及多个复杂性：

1. **交易模拟**：使用useSimulateContract *(一个wagmi hook，用于在发送交易前模拟交易以确保能成功)* 先模拟交易确保能成功
2. **状态追踪**：isPending、isConfirming等
3. **Toast通知**：成功或失败时显示通知
4. **错误处理**：用户拒绝错误的特殊处理

封装成hook后：
- 统一的交易发送逻辑
- 自动处理模拟调用
- Toast通知集成
- 错误处理标准化

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  pool: Pick<BladePool, 'address' | 'chainId'> *(只包含地址和链ID的池子对象)*  // 池子信息
  enabled?: boolean                              // 是否启用
  onSuccess?: () => void                         // 成功回调
}
```

### 输出（Return Value）
```typescript
{
  write?: () => Promise<void>                    // 发送交易的方法
  isPending: boolean                            // 交易是否 pending
  simulation: SimulateContractReturnType | undefined  // 模拟结果
  // ... 其他属性
}
```

### 执行逻辑
1. 从wagmi获取address
2. 使用useSimulateContract模拟交易
3. 创建write函数：
   - 检查simulation?.request存在
   - 调用writeContractAsync(simulation.request)
4. onSuccessCallback中：
   - 设置Pending状态
   - 创建Toast通知 *(一个UI通知组件，用于显示交易状态)*
   - 等待交易确认
   - 调用onSuccess回调
5. onError中处理用户拒绝和其他错误

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于执行Blade池子的解锁存款交易。它先模拟交易确保能成功执行，然后发送实际交易并追踪状态。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合wagmi的useSimulateContract和useWriteContract来处理交易模拟和发送，使用@sushiswap/notifications来显示Toast通知。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数包括池子对象（只需要地址和链ID）、启用开关和成功回调。输出包含write方法和isPending状态，以及simulation结果。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先从wagmi导入需要的hook，从notifications导入Toast相关函数，从错误处理导入用户拒绝错误判断函数。解构池子的地址和链ID，然后使用useState管理本地isPending状态。接着实现 useSimulateContract来模拟交易，实现onSuccessCallback和onError回调函数，实现useWriteContract来处理交易发送，创建 write函数，最后合并返回所有状态。

### 需要特别注意的约束条件

**模拟交易的前置检查**

simulationEnabled需要检查enabled、address和pool?.address是否都存在，只有都存在时才进行模拟。

**write函数的检查**

write函数在调用前需要检查simulation?.request是否存在，只有存在时才执行交易发送。

**避免double error**

writeContractAsync的调用在try-catch中，但catch块为空，这样可以让错误在onError中统一处理，避免重复处理。

**isPending的合并**

isPending是由本地setPending状态和rest.isPending合并得到的，这样无论交易是在发送阶段还是确认阶段，都能正确反映状态。

**使用useCallback包裹回调**

onSuccessCallback和onError都需要使用useCallback包裹，这样可以确保回调函数引用稳定，避免不必要的重渲染。

**错误日志记录**

对于非用户拒绝的错误，应该使用logger.error记录日志，这样可以在出现问题时进行调试。

### 状态管理的要点

**本地isPending状态**

使用useState管理本地的isPending状态，用于追踪交易处理中状态。

**onSuccessCallback的处理流程**

onSuccessCallback中需要：设置Pending为true、创建Toast通知、等待交易确认、调用外部onSuccess回调、设置Pending为false。

**错误处理**

onError中需要判断是否是用户拒绝的错误，如果是用户拒绝就不需要特殊处理，如果不是用户拒绝则记录错误并显示错误Toast。

**isPending的计算**

isPending是本地isPending和rest.isPending的逻辑或，表示交易正在处理中。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于执行Blade池子的解锁存款交易。

这个hook先模拟交易确保能成功执行，然后发送实际交易并追踪状态。

需要使用wagmi的useSimulateContract和useWriteContract来处理交易，使用@sushiswap/notifications来显示通知。

输入参数包括池子对象（只需要地址和链ID）、启用开关（默认true）和成功回调。

输出包含write方法、isPending状态和simulation结果。

实现步骤：
1. 从wagmi导入useConnection、usePublicClient、useSimulateContract、useWriteContract
2. 从notifications导入createToast、createErrorToast
3. 从错误处理导入isUserRejectedError判断函数
4. 从logger导入logger
5. 解构池子的address和chainId
6. 使用useState管理本地isPending状态
7. 实现useSimulateContract：
   - enabled检查enabled && address && pool?.address
   - 设置chainId、address、abi和functionName为'unlockDeposit'
8. 实现onSuccessCallback（使用useCallback）：
   - 设置Pending为true
   - 创建Toast通知
   - 等待交易确认
   - 调用外部onSuccess回调
   - 设置Pending为false
9. 实现onError（使用useCallback）：
   - 如果不是用户拒绝错误，记录错误并显示错误Toast
10. 实现useWriteContract：
    - 设置onSuccess为onSuccessCallback，onError为onError
11. 创建write函数（使用useMemo）：
    - 检查simulation?.request存在
    - 调用writeContractAsync发送交易
    - try-catch捕获错误但不处理
12. 返回合并结果

重要约束：
- simulationEnabled必须检查所有必要参数
- write函数需要检查simulation结果
- isPending合并本地状态和rest状态
- 使用useCallback包裹回调函数
- 错误处理使用logger.error记录日志

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useUnlockDeposit } from '@sushiswap/hooks'

function UnlockButton({ pool }) {
  const { write, isPending, simulation } = useUnlockDeposit({
    pool: {
      address: pool.address,
      chainId: pool.chainId,
    },
    onSuccess: () => {
      console.log('解锁成功!')
      // 刷新用户仓位数据
    },
  })

  const handleUnlock = () => {
    if (simulation?.request) {
      write?.()
    }
  }

  return (
    <div>
      <button onClick={handleUnlock} disabled={isPending || !simulation?.request}>
        {isPending ? '解锁中...' : '解锁存款'}
      </button>
      {!simulation?.request && <p>当前无法解锁</p>}
    </div>
  )
}
```

### 常见使用场景

1. **锁定期结束后的解锁**
   - 用户存款锁定到期后解锁
   - 解锁后可以正常提取

2. **交易模拟预检**
   - 在用户点击解锁前先模拟交易
   - 确保交易能够成功执行

3. **状态追踪和通知**
   - 显示Pending状态
   - Toast通知交易结果

### Dos and Don'ts

**Dos：**
- ✅ 先检查simulation?.request是否存在再调用write
- ✅ 使用onSuccess回调刷新相关数据
- ✅ 处理isPending状态禁用按钮防止重复点击
- ✅ 正确处理用户拒绝错误

**Don'ts：**
- ❌ 不要在simulation不存在时调用write
- ❌ 不要忽略isPending状态
- ❌ 不要在onError中直接throw
- ❌ 不要忘记在onSuccess中刷新数据
