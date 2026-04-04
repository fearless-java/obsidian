> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/aptos/swap/lib/use-swap.ts`

# useSwap Hook Tutorial

## 大白话讲讲这个hook的作用

`useSwap` 是一个用于获取 swap 路由的 hook。它根据：

- 输入代币和数量
- 输出代币
- 池子列表

计算最优 swap 路径。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **路由计算**：封装路由计算逻辑
2. **多池子支持**：支持多个池子的路由

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
- 从 `useSimpleSwapState` 获取状态

### 输出（返回值）
```typescript
{
  data: SwapRoute | undefined   // Swap 路由
  isLoading: boolean
}
```

### 核心执行逻辑

1. **获取状态**：从 context 获取 amount、token0、token1
2. **计算路由**：调用 `getSwapRoute` 计算最优路径

## 教你怎么用AI写这个Hook的提示词

### 怎么组织提示词

用AI来帮你写hook的时候，最重要的是把需求描述清楚。下面的模板展示了怎么组织你的提示词：

```
我需要创建一个类似 useSwap 的 React hook，用来获取最优的交换路径。

核心需求：
1. 从交换状态里获取输入金额、输入代币、输出代币
2. 调用路径计算函数找出最优的交换路线
3. 支持经过多个池子的复杂路径
4. 用 React Query 来管理数据请求
5. 要能定时刷新数据

请帮我写出完整的代码实现。
```

### 你需要注意的关键点

**依赖池子数据**：计算最优路径需要先有池子列表的数据，所以你的代码要确保池子数据已经准备好了，才能正常计算出路由。

### 这个Hook怎么管理状态

建议设置每10秒自动刷新一次，过期时间设为60秒。这样路由数据能保持新鲜，但又不会太频繁地去重新计算。

## 5. 该 hook 的用法教学

### 基本用法

```tsx
import { useSwap } from '@sushiswap/aptos'

function SwapRoute() {
  // 使用 useSwap 获取 swap 路由
  const { data: route, isLoading } = useSwap()

  if (isLoading) return <div>计算路由中...</div>

  if (!route) return <div>无法找到路由</div>

  return (
    <div>
      <p>输入: {route.amountIn}</p>
      <p>输出: {route.amountOut}</p>
      <p>路径: {route.path.map((t) => t.symbol).join(' → ')}</p>
    </div>
  )
}
```

### 常见用法

#### 1. Swap 界面

```tsx
import { useSwap, useSwapNetworkFee } from '@sushiswap/aptos'

function SwapInterface() {
  const { data: route, isLoading: routeLoading } = useSwap()
  const { data: networkFee, isLoading: feeLoading } = useSwapNetworkFee()

  const isLoading = routeLoading || feeLoading

  return (
    <div>
      <div className="swap-input">
        <label>输入</label>
        <input placeholder="0.0" />
        <span>{route?.tokenIn?.symbol}</span>
      </div>

      <div className="swap-output">
        <label>输出</label>
        {isLoading ? (
          <span>计算中...</span>
        ) : route ? (
          <>
            <span>{route.amountOut}</span>
            <span>{route.tokenOut?.symbol}</span>
          </>
        ) : (
          <span>无法交换</span>
        )}
      </div>

      {networkFee && (
        <div className="network-fee">
          网络费用: {networkFee} APT
        </div>
      )}

      <button disabled={!route || isLoading}>
        {isLoading ? '计算中...' : 'Swap'}
      </button>
    </div>
  )
}
```

#### 2. 多路径显示

```tsx
import { useSwap } from '@sushiswap/aptos'

function SwapPathDisplay() {
  const { data: route } = useSwap()

  if (!route?.path || route.path.length === 0) {
    return <div>无路由</div>
  }

  return (
    <div className="swap-path">
      <h4>交换路径</h4>
      <div className="path-steps">
        {route.path.map((token, i) => (
          <div key={i} className="path-step">
            <img src={token.logoURI} />
            <span>{token.symbol}</span>
            {i < route.path.length - 1 && <span>→</span>}
          </div>
        ))}
      </div>
      <div className="path-info">
        <p>中间池数量: {route.path.length - 2}</p>
        <p>预估输出: {route.amountOut} {route.tokenOut?.symbol}</p>
      </div>
    </div>
  )
}
```

#### 3. 价格比较

```tsx
import { useSwap } from '@sushiswap/aptos'
import { useStablePrice } from '@sushiswap/aptos'

function PriceComparison() {
  const { data: route } = useSwap()
  const { data: inputPrice } = useStablePrice({ currency: route?.tokenIn })
  const { data: outputPrice } = useStablePrice({ currency: route?.tokenOut })

  const marketPrice = inputPrice && outputPrice ? outputPrice / inputPrice : 0
  const routePrice = route ? Number(route.amountOut) / Number(route.amountIn) : 0
  const priceImpact = marketPrice > 0 ? ((marketPrice - routePrice) / marketPrice) * 100 : 0

  return (
    <div>
      <p>市场价格: {marketPrice.toFixed(6)}</p>
      <p>路由价格: {routePrice.toFixed(6)}</p>
      <p>价格影响: {priceImpact.toFixed(2)}%</p>
    </div>
  )
}
```

### Do（推荐做法）

- **组合 useSwapNetworkFee 获取费用**：计算总成本
- **显示加载状态**：路由计算需要时间
- **处理无路由情况**：显示友好提示
- **定期刷新**：设置 refetchInterval

### Don't（不推荐做法）

- **不要在路由未准备好时执行 swap**：需要检查 route 存在
- **不要忽略价格影响**：过高影响应该警告
- **不要假设路由存在**：某些交易对可能无法交换

### 相关的其他 hooks

- `useSwapNetworkFee` *(一个React hook，用于计算swap交易网络费用，构建swap交易并模拟获取gas估算)*：计算网络费用
- `usePools` *(一个React hook，用于获取网络中所有流动性池，从合约资源中读取所有交易对信息)*：获取池子列表
- `getSwapRoute` *(一个API方法/函数，根据输入输出代币和池子列表计算最优swap路径，支持多跳)*：计算最优路由
- `SwapRoute` *(一个TypeScript类型/配置，表示swap路由的数据结构，包含amountIn、amountOut、path等字段)*：路由类型
