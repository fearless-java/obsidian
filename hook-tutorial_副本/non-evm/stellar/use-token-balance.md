> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/token/use-token-balance.ts`

# useTokenBalance Hook Tutorial

## 大白话讲讲这个hook的作用

`useTokenBalance` *(一个React hook，用于查询用户在Stellar网络上持有特定代币的余额，基于Stellar Soroban的getTokenBalance方法)* 是一个用于查询用户在 Stellar 网络上持有特定代币余额的 hook。它就像"查银行账户余额"：

- 输入：钱包地址 + 代币合约地址
- 输出：该钱包持有的该代币数量

这个 hook 有多个变体：
- `useTokenBalance`：查询单个代币余额
- `useTokenBalanceFromToken`：从 Token 对象查询余额
- `useTokenBalances`：批量查询多个代币余额
- `useTokenBalancesMap`：批量查询并返回 Map 格式

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **简化 SDK 调用**：Stellar SDK 的余额查询需要处理 Soroban RPC 调用，封装后简化使用

2. **类型转换**：从链上返回的余额可能是 bigint，需要转换和处理

3. **缓存优化**：使用 React Query 缓存余额数据，避免频繁查询

4. **空值处理**：优雅处理未连接钱包或无效代币地址的情况

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）

**useTokenBalance:**
```typescript
address: StellarAddress | undefined  // 钱包地址
tokenContractId: string | null       // 代币合约地址
```

**useTokenBalances:**
```typescript
address: StellarAddress | undefined   // 钱包地址
tokens: Token[]                      // 代币数组
```

**useTokenBalancesMap:**
```typescript
address: StellarAddress | undefined   // 钱包地址
contracts: string[]                  // 代币合约地址数组
```

### 输出（返回值）

**useTokenBalance:**
```typescript
{
  data: bigint | null   // 代币余额（最小单位）
  ...
}
```

**useTokenBalances:**
```typescript
{
  data: TokenWithBalance[]  // 代币数组，每个包含余额
  ...
}
```

**useTokenBalancesMap:**
```typescript
{
  data: Record<string, string>  // { 合约地址: 余额字符串 }
  ...
}
```

### 核心执行逻辑

1. **参数校验**：检查地址和代币合约 ID 都有效
2. **循环查询**：对于批量查询，遍历每个代币调用 `getTokenBalance` *(Stellar Soroban SDK方法，用于获取账户的代币余额)*
3. **错误处理**：单个代币查询失败不影响其他代币，使用 try-catch 包裹
4. **格式化返回**：Map 版本将 bigint 转为字符串避免序列化问题

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个查询代币余额的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useTokenBalance的hook，用来查询某个区块链网络上的代币余额。然后明确几个关键点。第一，要能查单个代币的余额，也要能批量查多个代币的余额。第二，用React Query来管理数据的获取和缓存，这样能省掉很多重复的工作。第三，查询的时候调用链SDK的getTokenBalance方法就行。第四，钱包没连接或者代币地址无效的时候要给一个合理的默认值，别让程序崩了。第五，批量查询的时候要处理部分失败的情况，查到一半有哪个失败了不能把整个都取消掉。第六，余额要用bigint类型，但React Query可能序列化不了bigint，所以也可以转成字符串。

### 这里面有几个地方特别容易出错

bigint序列化的问题要特别注意，React Query有时候没办法正确处理bigint，如果发现数据不对试试转成字符串。批量查询的时候单个失败了不要影响其他的，要把每个代币的查询单独用try-catch包起来。enabled条件要设置好，没有钱包地址或者没有代币合约ID的时候就不要发起查询，查了也是白查还浪费资源。

### 数据刷新这里有讲究

余额变化其实挺频繁的，用户做了转账之类的操作余额就变了，所以staleTime可以设短一点，比如10到30秒。用户切换窗口回来的时候要刷新一下，用户可能在别的应用里做了操作。钱包地址变化了要清除旧的数据，不然会显示别人钱包的余额。这些细节都注意到了，体验会好很多。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useTokenBalance, useTokenBalances } from '@sushiswap/hooks'
import { useStellarWallet } from './wallet'
import { formatUnits } from 'ethers'

function BalanceDisplay() {
  const { address } = useStellarWallet()

  // 单个代币余额
  const { data: usdcBalance } = useTokenBalance({
    address,
    tokenContractId: usdcContractAddress,
  })

  // 批量代币余额
  const { data: tokenBalances } = useTokenBalances({
    address,
    tokens: [usdcToken, wethToken, wbtcToken],
  })

  return (
    <div>
      <p>USDC 余额: {usdcBalance ? formatUnits(usdcBalance, 6) : '0'}</p>
    </div>
  )
}
```

### 常见使用场景

1. **钱包首页展示**：显示用户所有代币余额
   ```tsx
   const { data: balances } = useTokenBalances({
     address: connectedAddress,
     tokens: supportedTokens,
   })
   ```

2. **Swap 界面余额显示**：显示用户可用于 swap 的代币数量
   ```tsx
   const { data: balance } = useTokenBalance({
     address,
     tokenContractId: tokenIn.address,
   })
   ```

3. **余额不足提示**：当用户余额不足时显示提示
   ```tsx
   const hasInsufficientBalance = balance < amountToSend
   if (hasInsufficientBalance) {
     toast.error('余额不足')
   }
   ```

### Dos and Don'ts

**Dos:**
- ✅ 使用 `formatUnits` *(ethers.js的工具函数，将最小单位的代币数量转换为人类可读的格式)* 将 bigint 转换为可读格式显示给用户
- ✅ 批量查询时使用 `useTokenBalances` 减少 hook 调用次数
- ✅ 设置较短的 `staleTime` 因为余额会频繁变化

**Don'ts:**
- ❌ 不要直接显示 bigint 给用户（数字太大无法阅读）
- ❌ 钱包未连接时不要发起查询，浪费资源
- ❌ 批量查询不要一次性查询太多代币，可能影响性能
