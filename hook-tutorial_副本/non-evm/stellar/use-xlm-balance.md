> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/token/use-xlm-balance.ts`

# useXlmBalance Hook Tutorial

## 大白话讲讲这个hook的作用

`useXlmBalance` *(一个React hook，专门用于查询Stellar原生代币XLM的余额，XLM用于支付网络费用和最低余额要求)* 是一个专门用于查询 XLM（Stellar 原生代币）余额的 hook。XLM 是 Stellar 网络的原生货币，用于支付网络费用和最低余额要求。

不同于其他代币，XLM 不需要合约，直接存储在账户中，所以查询方式也不同。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **XLM 特殊性**：XLM 是原生资产，查询方式和 Soroban 合约代币不同
2. **格式化输出**：XLM 有 7 位小数，hook 提供了 `formattedBalance` 方便 UI 显示
3. **钱包集成**：从 `useStellarWallet` *(获取当前连接的Stellar钱包地址和连接状态)* 获取当前连接的钱包地址

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
- 无参数，内部从 `useStellarWallet` 获取 connectedAddress

### 输出（返回值）
```typescript
{
  data: {
    balance: bigint      // XLM 余额（最小单位，7 位小数）
    formattedBalance: string  // 格式化后的余额，如 "100.00"
  }
  isLoading: boolean
  isPending: boolean
  error: Error | null
}
```

### 核心执行逻辑

1. **获取钱包地址**：从 `useStellarWallet` 获取 `connectedAddress`
2. **查询余额**：调用 `getXlmBalance(connectedAddress)` *(Stellar SDK方法，用于获取账户的XLM原生余额)* 获取链上余额
3. **格式化**：使用 `formatUnits(balance, 7)` *(ethers.js的工具函数，将最小单位转换为人类可读格式，7位小数)* 转换为人类可读格式，保留两位小数
4. **enabled 控制**：只有钱包连接时才发起查询

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个查询原生代币余额的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useXlmBalance的hook，用来查询某个区块链的原生代币余额。然后明确几个关键点。第一，你需要从钱包 provider 获取当前连接的钱包地址，没有连接就没有可查询的对象。第二，调用链上的API来获取余额，原生代币的查询方式和普通合约代币不一样。第三，原生代币有特定的精度位数，比如XLM是7位小数，这个必须搞清楚。第四，余额查出来是一个很大的数字，需要转换成方便阅读的格式给用户看。第五，只有当钱包连接上了才应该发起查询，没连接的时候查了也是白查。最后，用React Query来管理数据获取的缓存和状态。

### 这里面有几个地方特别容易出错

精度位数千万不能搞错，XLM是7位小数，如果你用了其他的位数，余额显示出来就会差十万八千里。未连接钱包的时候要返回一个默认值 `{ balance: 0n, formattedBalance: '-' }`，这样UI处理起来会方便很多。XLM余额虽然平时变化不大，但用户做了交易之后余额是会变的，所以还是需要设置一个合理的缓存时间。

### 数据刷新这里有讲究

组件挂载的时候应该获取一次最新余额，因为用户可能从别的页面切回来。窗口重新获得焦点的时候也要刷新一下，用户可能在外面做了交易回来。钱包地址变化的时候自然要重新查询。这些都是很容易忽略但很重要的细节。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useXlmBalance } from '@sushiswap/hooks'

function XlmBalanceDisplay() {
  const { data, isLoading } = useXlmBalance()

  if (isLoading) return <p>加载中...</p>

  return (
    <div>
      <p>XLM 余额: {data?.formattedBalance ?? '-'}</p>
    </div>
  )
}
```

### 常见使用场景

1. **钱包首页显示**：显示钱包的 XLM 余额
   ```tsx
   function WalletDashboard() {
     const { data: xlmBalance } = useXlmBalance()

     return (
       <div className="balance-card">
         <span>XLM</span>
         <span>{xlmBalance?.formattedBalance}</span>
       </div>
     )
   }
   ```

2. **费用检查**：在进行操作前检查是否有足够的 XLM 支付费用
   ```tsx
   const { data: xlmBalance } = useXlmBalance()
   const hasEnoughForFees = xlmBalance && xlmBalance.balance >= minBalance
   ```

3. **余额不足提示**：XLM 余额不足时提示用户
   ```tsx
   if (xlmBalance && xlmBalance.balance < 1n) {
     toast.warning('XLM 余额不足，无法支付网络费用')
   }
   ```

### Dos and Don'ts

**Dos:**
- ✅ 使用 `data?.formattedBalance` 显示给用户，方便阅读
- ✅ 使用 `data?.balance` 进行金额比较（bigint）
- ✅ 检查余额是否足够支付最低要求（1 XLM）

**Don'ts:**
- ❌ 不要直接显示 `balance`（bigint 数字太大）
- ❌ 不要假设 decimals 总是 7，不同链的原生代币可能不同
- ❌ 未连接钱包时不要发起查询浪费资源
