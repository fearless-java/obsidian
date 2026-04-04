> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/_common/lib/common/use-ledger-version.ts`

# useLedgerVersion Hook Tutorial

## 大白话讲讲这个hook的作用

`useLedgerVersion` 是一个用于获取 Aptos 账本版本（ledger version）的 hook。账本版本类似于以太坊的区块号，但它是递增的序列号：

- 可以获取"secondsAgo 秒前"的账本版本
- 用于历史数据查询

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **API 调用封装**：封装获取账本版本的 API
2. **历史查询支持**：支持查询历史状态的账本版本

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
secondsAgo: number           // 多少秒前的版本
```

### 输出（返回值）
```typescript
{
  data: number              // 账本版本号
}
```

### 核心执行逻辑

1. **调用 API**：调用 `/api/ledgerversion` 获取
2. **返回版本号**：返回历史账本版本

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useLedgerVersion 的 React hook，用来获取链上账本的版本号。

核心需求：
1. 要能获取"多久之前"的版本，比如获取1小时前的版本
2. 调用后端接口来获取数据
3. 返回的是账本的版本号数字
4. 用 React Query 来管理数据请求

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**依赖外部接口**：这个hook需要调用后端提供的接口来获取数据。所以你的代码要处理好接口调用失败的情况，比如网络问题或者服务器暂时不可用。

### 这个Hook怎么管理状态

因为账本版本号是不断增长的，而且查询"历史版本"这个操作本身不常用，所以建议把缓存时间设置得短一些。这样可以保证数据的时效性，同时也不会浪费存储空间。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useLedgerVersion } from '@sushiswap/aptos'

function CurrentLedgerVersion() {
  // 使用 useLedgerVersion 获取当前账本版本（secondsAgo = 0）
  const { data: version } = useLedgerVersion(0)

  return <div>当前账本版本: {version}</div>
}
```

### 常见用法

#### 1. 查询历史状态的账本版本

```tsx
import { useLedgerVersion } from '@sushiswap/aptos'

function HistoricalQuery() {
  // 获取 1 小时前的账本版本
  const { data: historicalVersion } = useLedgerVersion(3600)

  // 使用历史版本查询彼时的池子状态
  const { data: historicalPool } = usePool({
    poolAddress: '0x123...',
    ledgerVersion: historicalVersion
  })

  return (
    <div>
      <p>1小时前账本版本: {historicalVersion}</p>
      <p>当时池子储备: {historicalPool?.reserve0}</p>
    </div>
  )
}
```

#### 2. 用于价格历史查询

```tsx
import { useLedgerVersion, useStablePrice } from '@sushiswap/aptos'

function HistoricalPrice({ token }: { token: Token }) {
  // 获取 24 小时前的账本版本
  const { data: version } = useLedgerVersion(86400)

  // 查询历史价格
  const { data: price } = useStablePrice({
    currency: token,
    ledgerVersion: version
  })

  return (
    <div>
      <p>24小时前价格: ${price}</p>
    </div>
  )
}
```

#### 3. 组合多个历史版本查询

```tsx
import { useLedgerVersion } from '@sushiswap/aptos'

function MultiPeriodStats() {
  // 获取不同时间点的账本版本
  const { data: version1h } = useLedgerVersion(3600)    // 1小时前
  const { data: version1d } = useLedgerVersion(86400)    // 1天前
  const { data: version7d } = useLedgerVersion(604800)   // 7天前

  return (
    <div>
      <h3>历史账本版本</h3>
      <p>1小时前: {version1h}</p>
      <p>1天前: {version1d}</p>
      <p>7天前: {version7d}</p>
    </div>
  )
}
```

### Do（推荐做法）

- **用于历史数据查询**：需要查询过去某个时间点的链上状态时使用
- **设置较小的缓存时间**：账本版本随时变化，不需要长时间缓存
- **组合其他 hooks 使用**：与 usePool、useStablePrice 等组合查询历史状态
- **注意 secondsAgo 的范围**：过大的值可能导致 API 不支持

### Don't（不推荐做法）

- **不要用于实时数据查询**：实时数据应该使用 ledgerVersion = undefined 或 0
- **不要设置过长的缓存时间**：版本号会持续增长
- **不要忽略 API 限制**：后端可能不支持过旧的历史版本查询

### 相关的其他 hooks

- `usePool` *(一个React hook，用于获取单个流动性池信息，包括代币对、储备量、手续费率等)*：查询池子信息
- `useStablePrice` *(一个React hook，用于获取代币的USD价格，通过查找代币与稳定币的交易对并计算储备量比例得出价格)*：获取代币价格
- `usePoolsByTokens` *(一个React hook，通过代币对查询池子状态，返回每个代币对的池子是否存在)*：批量查询池子
- `React Query` *(一个React数据获取库，提供数据缓存、后台更新、错误处理等功能，useQuery是其核心hook之一)*：数据获取库
