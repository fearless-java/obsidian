> 源代码路径: `apps/web/src/lib/steer/hooks/use-steer-account-positions-extended.ts`

# useSteerAccountPositionsExtended Tutorial

## 1. 大白话讲讲这个hook的作用

`useSteerAccountPositionsExtended` *(一个React hook，用于获取用户的Steer仓位数据并计算USD价值)* 是用来获取用户的Steer仓位数据并计算USD价值的扩展版本。

简单来说：
- 在 `useSteerAccountPositions` *(一个React hook，用于查询用户的所有Steer仓位)* 的基础上增加价格计算
- 获取所有Vault的代币价格
- 将用户的token0/token1余额转换为USD价值
- 计算总仓位的USD价值

这是一个数据增强hook，用于展示用户的完整仓位价值信息。

## 2. 讲讲为什么需要封装该hook

计算USD价值涉及多个数据源：

1. **价格获取**：需要调用 `usePrices` *(一个React hook，用于获取代币价格数据)* 获取代币价格
2. **池子信息**：需要调用 `useSmartPools` *(一个React hook，用于获取SushiSwap的Smart Pools列表)* 获取Vault配置
3. **余额计算**：需要将余额转换为Amount对象 *(一个表示带代币精度金额的类)*
4. **数据聚合**：需要合并多个数据源

封装成hook后：
- 统一管理多个数据源
- 自动计算USD价值
- 提供完整的位置信息
- 简化UI组件

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  account: Address | undefined *(钱包地址类型)*       // 用户地址
  chainId: SmartPoolChainId *(支持Smart Pool的链ID枚举)*         // 链ID
  enabled?: boolean                  // 是否启用，默认true
}
```

### 输出（Return Value）
```typescript
{
  data: SteerAccountPositionExtended[] | undefined   // 扩展后的仓位列表
  isLoading: boolean
}

interface SteerAccountPositionExtended {
  // 基础数据（来自useSteerAccountPositions）
  id: string
  steerTokenBalance: bigint *(一个大整数类型，用于表示区块链上的大数字)*
  token0Balance: bigint
  token1Balance: bigint
  // 扩展数据
  vault: SmartPoolV1 *(Smart Pool的配置信息类型)*                // Vault配置
  token0: EvmToken *(以太坊虚拟机代币类型)*                   // 代币对象
  token1: EvmToken
  token0Amount: Amount<EvmToken> *(带精度的金额对象)*
  token1Amount: Amount<EvmToken>
  token0AmountUSD: number           // USD价值
  token1AmountUSD: number
  totalAmountUSD: number             // 总USD价值
}
```

### 执行逻辑
1. 调用 `usePrices` *(一个React hook，用于获取代币价格数据)* 获取价格数据
2. 调用 `useSmartPools` *(一个React hook，用于获取SushiSwap的Smart Pools列表)* 获取Vault列表
3. 调用 `useSteerAccountPositions` *(一个React hook，用于查询用户的所有Steer仓位)* 获取原始仓位
4. 使用useMemo合并所有数据：
   - 过滤出steerTokenBalance > 0的仓位
   - 查找对应的vault配置
   - 创建EvmToken对象
   - 获取代币价格
   - 创建Amount对象
   - 计算USD价值

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于获取用户的Steer仓位数据并计算USD价值。它合并仓位数据、Vault配置和价格数据，计算每个仓位的美元价值。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合wagmi、@tanstack/react-query、sushi库（用于Amount和Fraction）和@sushiswap/graph-client。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数包括用户地址、链ID和启用开关。输出是扩展后的仓位列表，每个仓位包含基础数据（ID、各种余额）、扩展数据（vault配置、两个代币对象、两个代币金额对象）以及USD价值（token0、token1和总计）。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先并行调用三个hook获取价格、Vault列表和原始仓位数据，然后在useMemo中合并这些数据，计算每个仓位的USD价值，过滤掉余额为0的仓位，最后合并所有loading状态并返回。

### 需要特别注意的约束条件

**并行调用三个hook**

这个hook需要同时调用usePrices、useSmartPools和useSteerAccountPositions来获取三种数据。

**条件获取**

只有当account存在时才获取smartPools和positions，所以useSmartPools的enabled应该是Boolean(enabled && account)。

**过滤条件**

需要过滤掉余额为0的仓位，过滤条件是position.steerTokenBalance > 0n。

**prices可能为undefined**

prices可能为空，需要提供默认值（如new Fraction(0)），否则计算会出错。

**USD计算**

使用Number转换，并且用toSignificant(8)保留有效数字。

### 状态管理的要点

**合并loading状态**

isLoading是三个hook的loading状态合并：isLoading || isVaultsLoading || isPricesLoading。

**useMemo中合并数据**

所有数据合并、过滤和计算都在useMemo中完成，避免不必要的重复计算。

**vault通过find查找**

需要通过find方法在smartPools中找到与position.id匹配的vault。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于获取用户的Steer仓位数据并计算USD价值。

这个hook合并仓位数据、Vault配置和价格数据，计算每个仓位的美元价值。

需要使用wagmi、@tanstack/react-query、sushi库（用于Amount和Fraction）和@sushiswap/graph-client。

输入参数包括用户地址（可以是undefined）、链ID和启用开关（默认true）。

输出是扩展后的仓位列表，每个仓位包含基础数据、vault配置、代币对象、代币金额和USD价值。

实现步骤：
1. 调用usePrices获取价格数据
2. 调用useSmartPools获取Vault列表（enabled检查Boolean(enabled && account)）
3. 从smartPools提取vaultIds数组
4. 调用useSteerAccountPositions获取原始仓位数据
5. 在useMemo中合并数据：
   - 检查smartPools和positions都存在
   - 使用flatMap遍历positions
   - 过滤掉余额为0的仓位
   - 使用find查找对应的vault
   - 创建EvmToken对象
   - 获取代币价格，创建Amount对象
   - 计算USD价值
6. 合并loading状态返回

重要约束：
- 只有当account存在时才获取smartPools和positions
- 过滤条件是position.steerTokenBalance > 0n
- prices可能为undefined，需要提供默认值
- USD计算使用Number转换

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useSteerAccountPositionsExtended } from '@sushiswap/hooks'
import { useConnection } from 'wagmi'

function MySteerPositions({ chainId }) {
  const { address } = useConnection()
  const { data: positions, isLoading } = useSteerAccountPositionsExtended({
    account: address,
    chainId,
  })

  if (isLoading) return <div>加载中...</div>
  if (!positions || positions.length === 0) return <div>暂无仓位</div>

  const totalValue = positions.reduce((sum, p) => sum + p.totalAmountUSD, 0)

  return (
    <div>
      <h3>我的Steer仓位</h3>
      <p>总价值: ${totalValue.toLocaleString()}</p>
      {positions.map((position) => (
        <div key={position.id}>
          <p>Vault: {position.vault.token0.symbol}/{position.vault.token1.symbol}</p>
          <p>Token0: {position.token0Amount.toString()} (${position.token0AmountUSD.toFixed(2)})</p>
          <p>Token1: {position.token1Amount.toString()} (${position.token1AmountUSD.toFixed(2)})</p>
          <p>总价值: ${position.totalAmountUSD.toFixed(2)}</p>
        </div>
      ))}
    </div>
  )
}
```

### 常见使用场景

1. **投资组合总览**
   - 展示用户在所有Steer Vault的仓位
   - 计算总USD价值

2. **收益分析**
   - 结合价格计算各仓位价值
   - 辅助投资决策

3. **仓位分布**
   - 分析仓位在不同Vault的分布
   - 帮助平衡投资组合

### Dos and Don'ts

**Dos：**
- ✅ 确保account地址存在
- ✅ 处理prices可能为undefined的情况
- ✅ 过滤掉余额为0的仓位
- ✅ 格式化USD显示

**Don'ts：**
- ❌ 不要在account不存在时调用
- ❌ 不要忽略prices为空的情况
- ❌ 不要直接渲染大数字，需要格式化
- ❌ 不要忽略isLoading状态
