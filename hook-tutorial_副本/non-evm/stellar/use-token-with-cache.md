> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/token/use-token-with-cache.ts`

# useTokenWithCache Hook Tutorial

## 大白话讲讲这个hook的作用

`useTokenWithCache` *(一个React hook，带有缓存层的代币信息查询，优先使用本地缓存减少链上查询)* 是一个带有缓存层的代币信息查询 hook。它的工作方式：

1. 首先检查本地缓存（customTokens）中是否有这个代币
2. 如果有，直接返回缓存数据
3. 如果没有，通过代币合约地址从链上查询代币信息（name、symbol、decimals 等）

这个 hook 就像一个"代币信息速查器"，优先使用本地缓存减少不必要的链上查询。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **缓存优先策略**：自定义代币列表是用户添加的可信代币，优先使用本地缓存

2. **减少 RPC 调用**：链上查询比本地缓存慢很多，缓存命中时直接返回

3. **保持数据一致性**：如果链上数据更新了，可以通过清除缓存或强制刷新获取最新数据

4. **placeholderData 支持**：保持之前的数据直到收到新数据，减少 UI 闪烁

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  address: string           // 代币合约地址
  enabled?: boolean         // 是否启用查询，默认 true
  keepPreviousData?: boolean // 是否保持旧数据，默认 true
}
```

### 输出（返回值）
```typescript
{
  data: Token | undefined   // 代币信息，包含 name、symbol、decimals、contract 等
  isLoading: boolean
  isPending: boolean
  error: Error | null
}
```

### 核心执行逻辑

1. **检查缓存**：调用 `useCustomTokens()` *(获取本地缓存的自定义代币列表)* 获取本地缓存的自定义代币
2. **缓存命中判断**：如果 `customTokens[address.toUpperCase()]` 存在，直接返回
3. **链上查询**：缓存未命中时，调用 `getTokenByContract(address)` *(从链上通过合约地址获取代币详细信息)* 从链上获取
4. **返回数据**：使用 React Query 的 `placeholderData` *(React Query选项，用于在加载时保持之前的数据)* 保持之前的数据直到新数据到达

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个带缓存的代币信息查询hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useTokenWithCache的hook，带缓存层的代币信息查询。然后明确几个关键点。第一，参数就是代币地址，很简单。第二，先查本地缓存，如果缓存里已经有这个代币了就直接返回，不用再跑去链上查。第三，如果缓存里没有，再通过代币合约地址从链上查询详细信息。第四，用useCustomTokens来获取本地缓存的数据。第五，支持keepPreviousData选项，这样在加载新数据的时候页面不会闪烁。第六，地址比较的时候要忽略大小写，因为区块链地址大小写不一致很常见。第七，用React Query管理数据。

### 这里面有几个地方特别容易出错

地址要用大写作为缓存的key，Stellar的合约地址格式就是这样规定的，不转大写的话查找会出问题。缓存依赖要先调用useCustomTokens获取缓存数据，没这个数据后面的缓存命中判断就没法做。链上查询可能失败，代币可能不存在或者合约地址无效，这种情况下返回undefined就行。缓存命中的场景下不要重试，重试只会让用户等更久。

### 数据刷新这里有讲究

窗口获得焦点的时候不需要刷新缓存，缓存里的数据本来就是本地的，没必要每次都去链上查。keepPreviousData选项要利用好，新数据还在加载的时候就显示旧数据，用户体验会好很多。缓存命中的时候不要重试，这是常识，重试只会浪费时间。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useTokenWithCache } from '@sushiswap/hooks'

function TokenInfo({ address }: { address: string }) {
  const { data: token, isLoading } = useTokenWithCache({
    address,
    keepPreviousData: true,
  })

  if (isLoading) return <p>加载中...</p>

  return (
    <div>
      <p>代币名称: {token?.name}</p>
      <p>代币符号: {token?.symbol}</p>
      <p>小数位数: {token?.decimals}</p>
    </div>
  )
}
```

### 常见使用场景

1. **代币选择器展示**：显示代币的图标和名称
   ```tsx
   function TokenOption({ tokenAddress }) {
     const { data: token } = useTokenWithCache({ address: tokenAddress })

     return (
       <div>
         <img src={token?.logoURI} alt={token?.symbol} />
         <span>{token?.symbol}</span>
       </div>
     )
   }
   ```

2. **Swap 界面代币信息**：显示输入/输出代币的详细信息
   ```tsx
   const { data: tokenIn } = useTokenWithCache({ address: tokenInAddress })
   const { data: tokenOut } = useTokenWithCache({ address: tokenOutAddress })
   ```

3. **池子代币信息**：获取池子的 Token0 和 Token1 信息
   ```tsx
   const { data: token0 } = useTokenWithCache({ address: pool.token0Address })
   const { data: token1 } = useTokenWithCache({ address: pool.token1Address })
   ```

### Dos and Don'ts

**Dos:**
- ✅ 配合 `useCustomTokens` 使用，优先显示用户自定义添加的代币
- ✅ 设置 `keepPreviousData: true` 避免输入切换时 UI 闪烁
- ✅ 使用 `useMemo` 缓存合并后的代币列表

**Don'ts:**
- ❌ 不要在每次渲染时传入新建的对象作为参数
- ❌ 不要忽略 `isPending` 状态（数据可能来自缓存但仍在加载）
- ❌ 对于批量代币，不要逐个调用 useTokenWithCache，使用批量查询接口
