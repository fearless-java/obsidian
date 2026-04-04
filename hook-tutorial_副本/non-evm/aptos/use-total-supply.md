> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/_common/lib/common/use-total-supply.ts`

# useTotalSupply Hook Tutorial

## 大白话讲讲这个hook的作用

`useTotalSupply` 是一个用于查询 LP 代币总供应量的 hook。它返回：

- 流动性池 LP 代币的总供应量

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **API 调用封装**：封装 Aptos API 的 LP 代币供应量查询

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
tokenAddress: string | undefined   // LP 代币地址
```

### 输出（返回值）
```typescript
{
  data: CoinInfo | undefined      // 代币信息（含供应量）
}
```

### 核心执行逻辑

1. **构建资源类型**：`0x1::coin::CoinInfo<LPToken>`
2. **调用 API**：获取 CoinInfo 资源

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useTotalSupply 的 React hook，用来查询某个代币的总发行量。

核心需求：
1. 接收一个代币地址作为参数
2. 去链上查询这个代币的 CoinInfo 资源，从里面获取总供应量
3. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**资源类型要写对**：Aptos链上每个资源都有自己固定的类型标识。查询的时候要写对这个类型，不然查不到数据。CoinInfo资源的类型格式是固定的，你的代码要能正确构建出这个类型字符串。

### 这个Hook怎么管理状态

总供应量这种数据变化不频繁，所以可以把缓存时间设置得长一些。因为没必要频繁地去链上查询，链上数据变化没那么快。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useTotalSupply } from '@sushiswap/aptos'

function LPTokenSupply({ lpTokenAddress }: { lpTokenAddress: string }) {
  // 使用 useTotalSupply 查询 LP 代币总供应量
  const { data: coinInfo } = useTotalSupply(lpTokenAddress)

  return (
    <div>
      <p>LP 代币总供应量</p>
      <p>{coinInfo?.supply}</p>
    </div>
  )
}
```

### 常见用法

#### 1. 计算用户份额比例

```tsx
import { useTotalSupply, useTokenBalance } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function UserShareRatio({ lpTokenAddress }: { lpTokenAddress: string }) {
  const { account } = useAccount()

  // 查询 LP 代币总供应量
  const { data: coinInfo } = useTotalSupply(lpTokenAddress)
  // 查询用户持有的 LP 数量
  const { data: userBalance } = useTokenBalance({
    account: account?.address,
    currency: lpTokenAddress
  })

  // 计算用户份额比例
  const totalSupply = Number(coinInfo?.supply ?? 0)
  const userBalanceNum = Number(userBalance ?? 0)

  if (totalSupply === 0) {
    return <div>池子暂无流动性</div>
  }

  const shareRatio = (userBalanceNum / totalSupply) * 100

  return (
    <div>
      <p>你的份额: {shareRatio.toFixed(2)}%</p>
      <p>你的 LP: {userBalanceNum}</p>
      <p>总供应: {totalSupply}</p>
    </div>
  )
}
```

#### 2. 计算池子总价值（TVL）

```tsx
import { useTotalSupply, usePool, useStablePrices } from '@sushiswap/aptos'

function PoolTVL({ poolAddress }: { poolAddress: string }) {
  // 获取池子信息
  const { data: pool } = usePool({ poolAddress })
  // 查询 LP 总供应量
  const { data: coinInfo } = useTotalSupply(pool?.lpTokenAddress)
  // 获取代币价格
  const { data: prices } = useStablePrices({
    currencies: [pool?.token0, pool?.token1].filter(Boolean)
  })

  const totalSupply = Number(coinInfo?.supply ?? 0)
  if (totalSupply === 0) return <div>池子为空</div>

  // 每个 LP 的价值 = (reserve0 * price0 + reserve1 * price1) / totalSupply
  const price0 = prices?.[pool?.token0?.address ?? ''] ?? 0
  const price1 = prices?.[pool?.token1?.address ?? ''] ?? 0
  const lpValue = (Number(pool?.reserve0) * price0 + Number(pool?.reserve1) * price1) / totalSupply

  return (
    <div>
      <p>每个 LP 价值: ${lpValue.toFixed(2)}</p>
      <p>总供应: {totalSupply}</p>
    </div>
  )
}
```

#### 3. 检测池子是否为空

```tsx
import { useTotalSupply } from '@sushiswap/aptos'

function PoolEmptyCheck({ lpTokenAddress }: { lpTokenAddress: string }) {
  // 查询 LP 代币总供应量
  const { data: coinInfo } = useTotalSupply(lpTokenAddress)

  const totalSupply = Number(coinInfo?.supply ?? 0)
  const isEmpty = totalSupply === 0
  const isNotEmpty = totalSupply > 0

  return (
    <div>
      {isEmpty && <div className="warning">池子为空</div>}
      {isNotEmpty && <div>池子有流动性</div>}
    </div>
  )
}
```

### Do（推荐做法）

- **设置较长的 staleTime**：供应量变化不频繁
- **组合 useTokenBalance 计算份额**：计算用户在池子中的份额
- **处理 supply 为字符串**：区块链数字可能是字符串格式
- **组合价格计算 TVL**：结合价格计算总锁仓价值

### Don't（不推荐做法）

- **不要直接除法没检查零**：除以零会导致错误
- **不要忽略 decimals**：供应量可能有精度需要处理
- **不要假设供应量是数字**：可能是 BigInt 或字符串

### 相关的其他 hooks

- `useTokenBalance` *(一个React hook，用于查询单个代币余额，接受account、currency、enabled等参数)*：查询代币余额
- `usePool` *(一个React hook，用于获取单个流动性池信息，包括代币对、储备量、手续费率等)*：获取池子信息
- `useStablePrices` *(一个React hook，批量获取多个代币USD价格，是useStablePrice的批量版本)*：批量获取价格
- `CoinInfo` *(一个TypeScript类型/配置，表示Aptos链上的CoinInfo资源结构，包含supply、decimals等字段)*：Aptos 代币信息类型
