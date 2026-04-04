> 源代码路径: `apps/web/src/lib/steer/hooks/use-steer-vault-reserves.ts`

# useSteerVaultReserves Tutorial

## 1. 大白话讲讲这个hook的作用

`useSteerVaultReserves` *(一个React hook，用于查询Steer Vault合约的储备数据)* 是用来查询Steer Vault合约储备量的hook。

简单来说：
- 直接从区块链读取Vault合约的储备数据
- 返回Vault中token0和token1的储备量
- 用于计算份额价值、显示池子规模等

这是一个只读查询hook，提供Vault的基础储备数据。

## 2. 讲讲为什么需要封装该hook

查询Vault储备涉及：

1. **批量读取**：需要使用useReadContracts *(一个wagmi hook，用于批量执行多个合约只读调用)* 读取多个Vault
2. **数据转换**：需要从原始结果提取储备数据
3. **自动刷新**：需要定期刷新数据
4. **格式统一**：需要统一的输出格式

封装成hook后：
- 简化多Vault查询
- 自动处理数据转换
- 设置合理的刷新间隔
- 返回格式化数据

## 3. 讲讲该hook的执行逻辑和数据流向

### 输入（Parameters）
```typescript
{
  vaultIds: EvmID[] | undefined *(实体ID数组，用于标识多个Vault)*      // Vault ID列表
  enabled?: boolean                  // 是否启用，默认true
}
```

### 输出（Return Value）
```typescript
{
  data: VaultReserve[] | undefined   // 储备数据列表
  isLoading: boolean
  // ... 其他useReadContracts属性
}

interface VaultReserve {
  id: string                         // vaultId
  reserve0: bigint *(一个大整数类型，用于表示区块链上的大数字)*                   // token0储备
  reserve1: bigint                   // token1储备
}
```

### 执行逻辑
1. 使用useMemo *(一个React hook，用于缓存计算结果避免不必要的重算)* 构建contract reads
2. 调用getVaultsReservesContracts生成批量读取配置
3. 使用useReadContracts执行批量读取
4. 使用select转换数据
5. 使用setInterval定期刷新（4秒）

## 4. 对该hook进行AI提示词编写教学

当你需要让AI帮你写一个类似的hook时，这里有一些实用的指导和需要注意的要点。简单来说，你需要清晰地描述这个hook要做什么、用什么技术栈、有哪些具体要求，以及一些关键的约束条件。

### 编写提示词的最佳实践

想让AI写出好用的代码，提示词需要包含以下几个核心部分：

**1. 清晰的功能描述**

首先告诉AI这个hook是用来做什么的。比如这个hook是用于查询Steer Vault合约的储备数据。它批量读取多个Vault的储备量并自动刷新。

**2. 明确的技术栈**

指定使用什么技术来构建这个hook。这里推荐使用React和TypeScript作为基础，配合wagmi的useReadContracts进行批量读取，使用@tanstack/react-query来管理状态。

**3. 输入输出要定义清楚**

告诉AI这个hook需要接收什么参数，以及返回什么格式的数据。输入参数是vaultIds数组和启用开关。输出是储备数据列表，每个元素包含ID和两种代币的储备量。

**4. 实现步骤要分清楚**

AI需要知道具体的实现步骤。首先导入需要的函数和hook，然后使用useMemo构建contracts配置，使用useReadContracts执行批量读取，在select中转换数据，使用useEffect设置定时刷新，最后清理定时器。

### 需要特别注意的约束条件

**contracts必须用useMemo缓存**

contracts配置需要用useMemo缓存，这样可以避免每次渲染时都重新创建配置对象。

**select中使用flatMap处理结果**

select中需要使用flatMap处理批量读取的结果，因为结果是一个数组。

**处理undefined结果**

select中需要检查result是否存在，如果不存在则返回空数组。

**4秒自动刷新**

使用setInterval在useEffect中实现4秒自动刷新，使用queryClient.invalidateQueries来刷新数据。

**cancelRefetch设为false**

invalidateQueries时cancelRefetch设为false，这样可以避免取消正在进行的请求。

**清理函数必须清除interval**

useEffect的清理函数必须清除interval，避免内存泄漏。

### 状态管理的要点

**contracts使用useMemo缓存**

这样可以确保配置对象引用稳定，只有当vaultIds变化时才重新创建。

**select处理数据转换**

select中需要将原始的批量读取结果转换为格式化后的数据。

**useQueryClient获取queryClient**

使用useQueryClient来获取queryClient实例，用于手动刷新数据。

**useEffect中设置定时刷新**

在useEffect中设置setInterval，依赖数组包含queryClient和query.queryKey。

### 完整的生产级提示词示例

```
请帮我创建一个React Hook，用于查询Steer Vault合约的储备数据。它批量读取多个Vault的储备量并自动刷新。

需要使用wagmi的useReadContracts进行批量读取，使用@tanstack/react-query来管理状态。

输入参数是vaultIds数组（可以是undefined）和启用开关（默认true）。

输出是储备数据列表，每个元素包含ID、第一个代币的储备量和第二个代币的储备量。

实现步骤：
1. 导入getVaultsReservesContracts和getVaultsReservesSelect函数
2. 导入useQueryClient
3. 使用useMemo构建contracts：
   - 检查vaultIds是否存在
   - 如果存在则调用getVaultsReservesContracts获取配置
4. 使用useReadContracts执行批量读取：
   - enabled检查enabled && vaultIds
   - select中使用flatMap处理结果，处理undefined情况
5. 使用useEffect设置定时刷新：
   - 使用setInterval每4秒刷新一次
   - 使用queryClient.invalidateQueries刷新数据
   - cancelRefetch设为false
   - 清理函数清除interval

重要约束：
- contracts必须使用useMemo缓存
- select中使用flatMap处理结果
- 处理undefined结果
- 4秒刷新间隔
- cancelRefetch设为false
- 清理函数必须清除interval

请生成完整的、可直接使用的代码实现。
```

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useSteerVaultReserves } from '@sushiswap/hooks'

function VaultReservesList({ vaultIds }) {
  const { data: reserves, isLoading } = useSteerVaultReserves({
    vaultIds,
  })

  if (isLoading) return <div>加载中...</div>
  if (!reserves || reserves.length === 0) return <div>暂无储备数据</div>

  return (
    <div>
      <h3>Vault储备数据</h3>
      {reserves.map((reserve) => (
        <div key={reserve.id}>
          <p>Vault: {reserve.id}</p>
          <p>Reserve0: {reserve.reserve0.toString()}</p>
          <p>Reserve1: {reserve.reserve1.toString()}</p>
        </div>
      ))}
    </div>
  )
}
```

### 常见使用场景

1. **Vault统计展示**
   - 显示Vault的流动性规模
   - 帮助用户了解池子深度

2. **份额价值计算**
   - 结合总供应量计算份额价值
   - 辅助投资决策

3. **实时监控**
   - 利用4秒自动刷新
   - 监控储备变化

### Dos and Don'ts

**Dos：**
- ✅ 使用useMemo缓存contracts配置
- ✅ 处理undefined结果
- ✅ 格式化bigint显示
- ✅ 在useEffect中清理interval

**Don'ts：**
- ❌ 不要在vaultIds为空时调用
- ❌ 不要忽略select中的undefined处理
- ❌ 不要直接渲染bigint
- ❌ 不要忘记清理setInterval
