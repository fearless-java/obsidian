> 源代码路径: `apps/web/src/lib/steer/hooks/use-steer-account-position.ts`

# useSteerAccountPosition Tutorial

## 1. 大白话讲讲这个hook的作用

`useSteerAccountPosition` *(一个React hook，用于查询用户在某个特定Steer Vault中的单个仓位)* 是用来查询用户在某个特定Steer Vault（智能策略金库）中的仓位的hook。

简单来说：
- Steer是提供自动化策略的协议
- 用户在策略池中存入流动性获得收益
- 这个hook查询用户在指定Vault中的头寸信息
- 包括：steer代币余额、token0/token1的余额和储量

这是Steer系列中最基础的单个仓位查询hook。

## 2. 讲讲为什么需要封装该hook

查询Steer仓位涉及复杂的合约交互：

1. **批量读取**：需要同时读取多个合约的多个方法
2. **数据聚合**：需要从不同来源聚合数据
3. **比率计算**：需要计算用户占总供应量的比例
4. **自动刷新**：需要定期刷新数据

封装成hook后：
- 隐藏复杂的合约调用逻辑
- 提供简洁的仓位数据接口
- 自动计算用户权益
- 支持自动刷新

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  account: Address | undefined *(钱包地址类型)*       // 用户地址
  vaultId: EvmID | undefined *(实体ID类型，用于标识Vault)*         // Vault ID
  enabled?: boolean                  // 是否启用，默认true
}
```

### 输出（Return Value）
```typescript
{
  data: SteerAccountPosition | undefined   // 单个仓位数据
  isLoading: boolean
  // ... 其他状态
}

interface SteerAccountPosition {
  id: string                         // vaultId
  steerTokenBalance: bigint *(一个大整数类型，用于表示区块链上的大数字)*          // 用户持有的steer代币数量
  token0Balance: bigint               // 用户token0余额
  token1Balance: bigint               // 用户token1余额
  // ... 其他计算后的数据
}
```

### 执行逻辑
1. 调用 `useSteerAccountPositions` *(一个React hook，用于查询用户的所有Steer仓位)* 获取所有仓位
2. 过滤出指定的vaultId
3. 返回第一个匹配的结果
4. 内部使用useMemo合并数据

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于查询用户在某个特定Steer Vault中的单个仓位。它是对useSteerAccountPositions的封装，返回单个Vault的仓位数据而不是数组。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合wagmi和@tanstack/react-query。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数包括用户地址、 vaultId和启用开关。输出是单个仓位数据，包含steer代币余额、两种代币的余额等信息。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先调用useSteerAccountPositions获取所有仓位，注意vaultIds参数需要是数组形式（即使只有一个id），然后使用useMemo提取data[0]并返回。

### 需要特别注意的约束条件

**内部调用另一个hook**

这个hook内部调用useSteerAccountPositions来获取所有仓位数据，不需要自己实现复杂的逻辑。

**vaultIds必须是数组**

传给useSteerAccountPositions的vaultIds必须是数组形式，即使用户只查询一个vault，也需要用[vaultId]而不是直接传vaultId。

**返回data[0]而非完整数组**

useSteerAccountPositions返回的是仓位数组，但这个hook需要返回单个仓位，所以应该提取data[0]。

**使用useMemo包裹返回结果**

返回结果应该用useMemo包裹，这样可以避免不必要的引用创建。

### 状态管理的要点

**内部hook的状态管理**

useSteerAccountPositions会自动管理数据获取的状态，这个hook直接使用其返回的状态。

**isLoading状态**

isLoading等状态直接透传自内部hook，不需要额外处理。

**enabled影响内部hook**

enabled参数直接影响内部hook的查询是否执行。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于查询用户在某个特定Steer Vault中的单个仓位。

这个hook是对useSteerAccountPositions的封装，返回单个Vault的仓位数据而不是数组。

需要使用React和TypeScript，配合wagmi和@tanstack/react-query。

输入参数包括用户地址（可以是undefined）、vaultId（可以是undefined）和启用开关（默认true）。

输出是单个仓位数据，包含ID、steer代币余额、两种代币的余额等信息。

实现步骤：
1. 调用useSteerAccountPositions：
   - vaultIds使用vaultId ? [vaultId] : undefined（注意必须是数组）
   - 传入account和enabled参数
2. 使用useMemo返回结果：
   - 展开query的所有属性
   - data使用query.data?.[0]（取第一个元素）
   - 依赖数组包含query

重要约束：
- 内部调用useSteerAccountPositions
- vaultIds必须是数组，即使只有一个id
- 返回data[0]而非完整数组
- 使用useMemo包裹返回结果

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useSteerAccountPosition } from '@sushiswap/hooks'
import { useConnection } from 'wagmi'

function VaultPosition({ vaultId }) {
  const { address } = useConnection()
  const { data: position, isLoading } = useSteerAccountPosition({
    account: address,
    vaultId,
  })

  if (isLoading) return <div>加载中...</div>
  if (!position) return <div>没有仓位</div>

  return (
    <div>
      <h3>Vault仓位</h3>
      <p>Steer代币余额: {position.steerTokenBalance.toString()}</p>
      <p>Token0余额: {position.token0Balance.toString()}</p>
      <p>Token1余额: {position.token1Balance.toString()}</p>
    </div>
  )
}
```

### 常见使用场景

1. **单个Vault仓位展示**
   - 查看用户在特定Vault的仓位
   - 显示详细的代币余额

2. **仓位详情页**
   - 用于Vault详情页面
   - 显示用户的完整仓位信息

3. **操作前置检查**
   - 检查用户是否有仓位
   - 决定是否显示操作按钮

### Dos and Don'ts

**Dos：**
- ✅ 使用vaultId确保获取正确的仓位
- ✅ 确保account地址存在
- ✅ 处理undefined情况（无仓位）
- ✅ 格式化bigint显示

**Don'ts：**
- ❌ 不要在vaultId为空时调用
- ❌ 不要忽略无仓位的情况
- ❌ 不要直接渲染bigint
- ❌ 不要忽略isLoading状态
