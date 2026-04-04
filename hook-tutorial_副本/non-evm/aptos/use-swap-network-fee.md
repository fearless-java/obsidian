> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/swap/lib/use-swap-network-fee.ts`

# useSwapNetworkFee Hook Tutorial

## 大白话讲讲这个hook的作用

`useSwapNetworkFee` 是一个用于计算 swap 交易网络费用的 hook。它：

- 构建 swap 交易
- 模拟交易获取 gas 估算
- 计算网络费用

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **Gas 估算**：模拟交易获取 gas 使用量
2. **费用计算**：结合 gas price 计算费用

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
- 从 `useSimpleSwapState` 获取 bestRoutes、token0、token1 等

### 输出（返回值）
```typescript
{
  data: number   // 网络费用（APT 格式）
}
```

### 核心执行逻辑

1. **获取 gas price**：调用 Aptos API 获取
2. **构建交易**：构建 swap 交易
3. **模拟执行**：simulate 获取 gas 使用量
4. **计算费用**：gas_used * gas_price

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useSwapNetworkFee 的 React hook，用来计算一笔交换交易需要付多少网络费用。

核心需求：
1. 获取最优路由和代币信息
2. 构建出一笔完整的交换交易
3. 模拟执行这笔交易来估算会消耗多少gas
4. 结合gas价格计算出需要支付的总费用
5. 返回费用的数值

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**需要钱包连接**：这个功能要模拟交易执行才能计算出费用，所以用户必须先连接钱包。没有钱包连接就没法模拟，也就算不出费用。

### 这个Hook怎么管理状态

这个hook要等到账户和路由数据都准备好了才能启用。任何一个问题没准备好都没法正常计算费用。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useSwapNetworkFee } from '@sushiswap/aptos'

function NetworkFeeDisplay() {
  // 使用 useSwapNetworkFee 获取网络费用
  const { data: fee, isLoading } = useSwapNetworkFee()

  if (isLoading) return <div>计算费用中...</div>

  return (
    <div>
      <p>网络费用</p>
      <p>{fee} APT</p>
    </div>
  )
}
```

### 常见用法

#### 1. Swap 确认界面

```tsx
import { useSwap, useSwapNetworkFee } from '@sushiswap/aptos'

function SwapConfirmation() {
  const { data: route } = useSwap()
  const { data: networkFee, isLoading: feeLoading } = useSwapNetworkFee()

  const totalCost = networkFee ?? 0

  return (
    <div className="swap-confirmation">
      <h3>确认 Swap</h3>
      <div className="summary">
        <div>输入: {route?.amountIn} {route?.tokenIn?.symbol}</div>
        <div>输出: {route?.amountOut} {route?.tokenOut?.symbol}</div>
      </div>

      <div className="fees">
        {feeLoading ? (
          <div>计算网络费用...</div>
        ) : (
          <>
            <div>网络费用: {networkFee} APT</div>
            <div>总成本: {totalCost} APT</div>
          </>
        )}
      </div>

      <button disabled={!route || feeLoading}>确认 Swap</button>
    </div>
  )
}
```

#### 2. 费用比较

```tsx
import { useSwapNetworkFee } from '@sushiswap/aptos'
import { useNetwork } from '@sushiswap/aptos'

function FeeComparison() {
  const { data: networkFee } = useSwapNetworkFee()
  const { network } = useNetwork()

  // 不同网络的平均费用
  const avgFee = network === 'mainnet' ? 0.005 : 0.001

  return (
    <div>
      <h3>费用信息</h3>
      <p>当前交易费用: {networkFee} APT</p>
      <p>网络平均费用: ~{avgFee} APT</p>
      {networkFee && networkFee > avgFee * 5 && (
        <div className="warning">费用较高，请确认</div>
      )}
    </div>
  )
}
```

#### 3. 费用不足提示

```tsx
import { useSwapNetworkFee } from '@sushiswap/aptos'
import { useTokenBalance } from '@sushiswap/aptos'
import { useAccount } from '@aptos-labs/wallet-adapter-react'

function FeeSufficiencyCheck() {
  const { account } = useAccount()
  const { data: fee, isLoading: feeLoading } = useSwapNetworkFee()
  const { data: aptBalance } = useTokenBalance({
    account: account?.address,
    currency: '0x1::aptos_coin::AptosCoin'
  })

  const hasEnough = aptBalance && fee ? aptBalance > fee : false

  return (
    <div>
      {feeLoading ? (
        <div>计算费用...</div>
      ) : (
        <>
          <div>需要费用: {fee} APT</div>
          <div>你的余额: {aptBalance} APT</div>
          {!hasEnough && (
            <div className="error">APT 余额不足，无法支付网络费用</div>
          )}
        </>
      )}
    </div>
  )
}
```

### Do（推荐做法）

- **组合 useSwap 使用**：获取路由后计算费用
- **检查费用充足性**：确保钱包余额足够支付费用
- **显示加载状态**：费用计算需要模拟交易
- **处理费用过高情况**：提醒用户确认

### Don't（不推荐做法）

- **不要在没有路由时计算**：需要 route 准备好
- **不要忽略钱包连接**：模拟交易需要钱包
- **不要假设费用固定**：费用会根据交易复杂度变化

### 相关的其他 hooks

- `useSwap` *(一个React hook，用于获取swap路由，根据输入输出代币和池子列表计算最优swap路径)*：获取 swap 路由
- `useTokenBalance` *(一个React hook，用于查询单个代币余额，接受account、currency、enabled等参数)*：查询 APT 余额
- `useNetwork` *(一个React hook，用于获取当前链网络配置信息，返回网络名称、默认稳定币、合约地址等配置)*：获取网络配置
- `simulate` *(一个API方法，用于模拟交易执行获取gas使用量，不实际广播交易)*：交易模拟 API
