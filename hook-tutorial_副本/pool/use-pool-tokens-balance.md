> 源代码路径: `apps/web/src/lib/pool/blade/usePoolTokensBalance.ts`

# usePoolTokensBalance Tutorial

## 1. 大白话讲讲这个hook的作用

`usePoolTokensBalance` *(一个React hook，用于获取Blade池子合约中所有代币的余额)* 是用来获取Blade池子合约中所有代币余额的hook。

简单来说：
- 直接从区块链读取Blade池子合约的 `allTokensBalance` 函数
- 返回该池子中所有代币的当前余额
- 用于显示池子的流动性总量等数据

这是一个只读查询hook，直接从合约读取数据。

## 2. 讲讲为什么需要封装该hook

读取池子代币余额涉及：

1. **合约调用**：需要使用wagmi的useReadContract *(一个wagmi hook，用于调用合约的只读方法)*
2. **参数简单化**：只需传入pool信息
3. **自动刷新**：设置refetchInterval定期刷新数据
4. **ABI处理**：需要正确的ABI和functionName

封装成hook后：
- 简化调用方式，只需传入pool对象
- 自动处理chainId和address
- 设置合理的自动刷新间隔
- 返回标准化的wagmi查询结果

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  pool: Pick<BladePool, 'address' | 'chainId'> *(只包含地址和链ID的池子对象类型)*   // 池子信息
  enabled?: boolean                               // 是否启用，默认true
}
```

### BladePool类型
```typescript
interface BladePool {
  address: string
  chainId: number
  // ... 其他字段
}
```

### 输出（Return Value）
```typescript
{
  data: readonly [bigint, bigint, ...] | undefined  // 代币余额数组
  isLoading: boolean
  isError: boolean
  error: Error | null
  // ... 其他useReadContract属性
}
```

### 执行逻辑
1. 解构pool的chainId和address
2. 使用useReadContract调用合约：
   - address: pool.address
   - abi: bladeCommonExchangeAbi *(Blade池子的通用ABI定义)*
   - functionName: 'allTokensBalance'
   - args: []
3. 设置refetchInterval为10000ms（10秒）
4. enabled参数控制是否执行

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于获取Blade池子合约中所有代币的余额。它直接从区块链读取allTokensBalance函数的返回值。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合wagmi的useReadContract来调用合约的只读方法。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数是一个池子对象，只需要包含地址和链ID（使用Pick提取需要的字段）。输出是代币余额数组，是readonly类型的大数字数组。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先导入正确的ABI，然后解构池子对象的地址和链ID，最后使用useReadContract配置合约调用参数，包括链ID、地址、ABI、函数名、空参数数组，以及查询配置。

### 需要特别注意的约束条件

**只传入必要的池子字段**

池子对象只需要地址和链ID两个字段，使用Pick来提取而不是传入整个池子对象。这样可以减少不必要的数据传递。

**使用正确的ABI和函数名**

ABI使用bladeCommonExchangeAbi，函数名是'allTokensBalance'。args是空数组，因为这个函数不需要任何参数。

**设置合理的自动刷新间隔**

refetchInterval设置为10000毫秒（10秒），这样可以定期刷新数据，让用户看到相对实时的余额信息。

**enabled检查**

enabled默认是true，但查询是否真正执行还需要池子的地址和链ID都存在。所以enabled的检查应该是enabled && pool.address && pool.chainId。

### 状态管理的要点

**使用useReadContract管理状态**

useReadContract会自动管理合约调用的状态，包括loading、error和data。不需要手动管理queryKey，wagmi会自动处理。

**返回的数据是readonly**

返回的代币余额数组是readonly类型，这意味着一旦返回就不应该被修改。

**自动刷新**

通过设置refetchInterval，可以实现自动刷新数据的功能，不需要用户手动刷新。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于获取Blade池子合约中所有代币的余额。

这个hook直接从区块链读取allTokensBalance函数的返回值。

需要使用wagmi的useReadContract来调用合约的只读方法。

输入参数是一个池子对象，只需要地址和链ID两个字段（使用Pick提取）。

输出是代币余额数组，是readonly类型的大数字数组。

实现步骤：
1. 从正确的位置导入bladeCommonExchangeAbi
2. 解构池子对象的chainId和address
3. 使用useReadContract：
   - chainId使用pool.chainId
   - address使用pool.address
   - abi使用bladeCommonExchangeAbi
   - functionName使用'allTokensBalance'
   - args使用空数组
   - query中设置enabled检查和refetchInterval为10000

重要约束：
- 池子对象只需要地址和链ID，使用Pick提取
- abi使用bladeCommonExchangeAbi
- functionName是'allTokensBalance'
- args是空数组
- refetchInterval设置为10000毫秒
- enabled检查所有必要参数

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { usePoolTokensBalance } from '@sushiswap/hooks'

function PoolStats({ pool }) {
  const { data: balances, isLoading, refetch } = usePoolTokensBalance({
    pool: {
      address: pool.address,
      chainId: pool.chainId,
    },
  })

  if (isLoading) return <div>加载中...</div>

  return (
    <div>
      <h3>池子代币余额</h3>
      {balances && (
        <div>
          <p>Token0余额: {balances[0].toString()}</p>
          <p>Token1余额: {balances[1].toString()}</p>
        </div>
      )}
      <button onClick={() => refetch()}>刷新</button>
    </div>
  )
}
```

### 常见使用场景

1. **池子统计展示**
   - 显示池子的总流动性
   - 计算各代币占比

2. **实时监控**
   - 利用refetchInterval自动刷新
   - 监控池子余额变化

3. **交易对分析**
   - 结合价格计算交易对价值
   - 辅助流动性分析

### Dos and Don'ts

**Dos：**
- ✅ 使用Pick只传入需要的字段
- ✅ 设置合理的refetchInterval
- ✅ 处理bigint类型时转换为可显示格式
- ✅ 使用enabled参数控制查询启用

**Don'ts：**
- ❌ 不要传入整个pool对象，只传address和chainId
- ❌ 不要设置过短的refetchInterval，会导致频繁请求
- ❌ 不要直接渲染bigint，需要格式化
- ❌ 不要忽略isLoading和error状态
