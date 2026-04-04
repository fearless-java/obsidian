> 源代码路径: `apps/web/src/lib/pool/blade/useVestingDeposit.ts`

# useVestingDeposit Tutorial

## 1. 大白话讲讲这个hook的作用

`useVestingDeposit` *(一个React hook，用于查询用户在Blade池子中的归属存款信息)* 是用来查询用户在Blade池子中的归属中存款（Vesting Deposit）的hook。

简单来说：
- 查询用户锁定的池子代币数量
- 查询锁定的到期时间
- 用于显示用户的归属中资产

这是一个只读查询hook，用于展示用户的归属存款信息。

## 2. 讲讲为什么需要封装该hook

查询归属存款涉及：

1. **合约调用**：需要使用wagmi的useReadContract *(一个wagmi hook，用于调用合约的只读方法)*
2. **参数处理**：需要传入用户地址
3. **数据转换**：返回的bigint需要转换为可用格式
4. **时间处理**：lockedUntil需要从Unix时间戳转换为Date

封装成hook后：
- 简化调用方式
- 自动处理数据转换
- 设置合理的自动刷新间隔
- 返回标准化的数据格式

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  pool: Pick<BladePool, 'address' | 'chainId'> *(只包含地址和链ID的池子对象)*   // 池子信息
  address: Address | undefined *(钱包地址类型)*                   // 用户地址
  enabled?: boolean                              // 是否启用，默认true
}
```

### 输出（Return Value）
```typescript
{
  data: {
    balance: bigint *(一个大整数类型，用于表示区块链上的大数字)*        // 锁定的池子代币数量
    lockedUntil?: Date     // 解锁时间（如果为0则未锁定）
  } | undefined
  isLoading: boolean
  isError: boolean
  error: Error | null
}
```

### 执行逻辑
1. 解构pool的chainId和address
2. 使用useReadContract调用合约：
   - address: pool.address
   - abi: bladeCommonExchangeAbi *(Blade池子的通用ABI定义)*
   - functionName: 'vestingDeposits'
   - args: address ? [address] : undefined
3. 使用select转换数据：
   - 从readonly [bigint, bigint]提取
   - 第一个是lockedUntil，第二个是poolTokenAmount
   - lockedUntil !== 0n时转换为Date，否则undefined
4. 设置refetchInterval为10000ms
5. enabled检查：address && enabled

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于查询用户在Blade池子中的归属存款信息，包括锁定的代币数量和锁定到期时间。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合wagmi的useReadContract来调用合约的只读方法。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数包括池子对象（只需要地址和链ID）、用户地址和启用开关。输出包含锁定的池子代币数量和解锁时间（如果为0则未锁定）。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先导入正确的ABI，解构池子的链ID和地址，然后使用 useReadContract配置合约调用参数，在select中对返回的元组数据进行转换，将lockedUntil从时间戳转换为Date格式，设置合理的自动刷新间隔和enabled条件。

### 需要特别注意的约束条件

**返回值是元组**

vestingDeposits返回一个元组[lockedUntil, poolTokenAmount]，第一个是锁定到期时间，第二个是池子代币数量。

**address为空时不调用合约**

如果address为空，应该使用undefined作为args，而不是发送无效的请求。

**select中使用useCallback**

select函数需要使用useCallback包裹，这样可以避免不必要的重算。

**lockedUntil为0表示未锁定**

lockedUntil如果是0n（大数字的0），表示没有锁定，应该返回undefined而不是返回1970年那个日期。转换时需要先转成Number再乘以1000得到毫秒。

**设置自动刷新**

refetchInterval设置为10000毫秒（10秒），可以定期刷新数据。

### 状态管理的要点

**使用useReadContract管理状态**

useReadContract会自动管理合约调用的状态，包括loading、error和data。

**select的数据转换**

select函数在数据返回时进行转换，返回格式化后的对象，而不是原始的元组。

**enabled检查**

enabled检查是Boolean(address && enabled)，只有当地址存在且启用时才执行查询。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于查询用户在Blade池子中的归属存款信息，包括锁定的代币数量和锁定到期时间。

需要使用wagmi的useReadContract来调用合约的只读方法。

输入参数包括池子对象（只需要地址和链ID）、用户地址（可以是undefined）和启用开关（默认true）。

输出包含锁定的池子代币数量和解锁时间（如果为0则未锁定）。

实现步骤：
1. 从正确的位置导入bladeCommonExchangeAbi
2. 解构池子的chainId和address
3. 使用useReadContract：
   - chainId使用pool.chainId
   - address使用pool.address
   - abi使用bladeCommonExchangeAbi
   - functionName使用'vestingDeposits'
   - args使用address ? [address] : undefined
   - query中设置refetchInterval为10000
   - enabled检查Boolean(address && enabled)
   - select中使用useCallback处理数据转换：
     - 解构返回的元组[lockedUntil, poolTokenAmount]
     - 返回格式化对象：balance是poolTokenAmount，lockedUntil根据lockedUntil是否为0n来决定是Date还是undefined
     - 注意：lockedUntil需要乘以1000转换秒为毫秒

重要约束：
- 返回值是元组[lockedUntil, poolTokenAmount]
- address为空时使用undefined作为args
- select需要使用useCallback
- lockedUntil为0n时返回undefined而非1970年
- 需要乘以1000转换秒为毫秒

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useVestingDeposit } from '@sushiswap/hooks'
import { useConnection } from 'wagmi'

function VestingInfo({ pool }) {
  const { address } = useConnection()
  const { data, isLoading } = useVestingDeposit({
    pool: {
      address: pool.address,
      chainId: pool.chainId,
    },
    address,
  })

  if (isLoading) return <div>加载中...</div>

  return (
    <div>
      <h3>归属存款信息</h3>
      {data?.balance > 0n ? (
        <div>
          <p>锁定余额: {data.balance.toString()}</p>
          {data.lockedUntil && (
            <p>解锁时间: {data.lockedUntil.toLocaleDateString()}</p>
          )}
          {data.lockedUntil && new Date() < data.lockedUntil && (
            <p>距离解锁: {getTimeRemaining(data.lockedUntil)}</p>
          )}
        </div>
      ) : (
        <p>没有归属中的存款</p>
      )}
    </div>
  )
}
```

### 常见使用场景

1. **显示归属存款**
   - 展示用户锁定的存款金额
   - 显示锁定到期时间

2. **解锁判断**
   - 检查是否已解锁（lockedUntil为undefined或已过）
   - 决定是否显示解锁按钮

3. **倒计时显示**
   - 计算距离解锁的剩余时间
   - 实时更新

### Dos and Don'ts

**Dos：**
- ✅ 确保address存在后再调用
- ✅ 处理lockedUntil为undefined的情况（已解锁）
- ✅ 格式化大数字和日期显示
- ✅ 使用refetchInterval自动刷新

**Don'ts：**
- ❌ 不要在address为空时调用，应该检查
- ❌ 不要直接渲染bigint，需要格式化
- ❌ 不要直接使用lockedUntil为0的时间戳
- ❌ 不要忽略isLoading状态
