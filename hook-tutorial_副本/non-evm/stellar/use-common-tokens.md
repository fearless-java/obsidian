> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/token/use-common-tokens.ts`

# useCommonTokens Hook Tutorial

## 大白话讲讲这个hook的作用

`useCommonTokens` *(一个React hook，用于获取Stellar网络上的常见/热门代币列表，从静态配置和外部API两个来源获取)* 是一个用于获取常见/热门代币列表的 hook。它从两个来源获取数据：

1. **硬编码的静态代币**：如 XLM、USDC 等核心资产
2. **StellarExpert API**：动态获取 Stellar 网络上热门的 50 个代币

这个 hook 就像一个"代币白名单"，帮助用户快速找到常用的代币。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **数据聚合**：将静态配置和动态 API 数据合并
2. **容错处理**：StellarExpert API 失败时使用硬编码代币作为后备
3. **过滤机制**：过滤掉已知的过时或废弃代币
4. **缓存优化**：热门代币列表变化不频繁，设置较长的缓存时间

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
- 无参数，使用默认配置

### 输出（返回值）
```typescript
{
  data: Record<string, Token>   // 代币映射表，key 为合约地址(大写)
  isLoading: boolean
  isPending: boolean
  error: Error | null
}
```

### 核心执行逻辑

1. **加载静态代币**：首先加载硬编码的 `staticTokens`
2. **获取动态代币**：从 StellarExpert API 获取热门代币
3. **合并去重**：合并两个来源，使用合约地址作为 key
4. **过滤过时代币**：根据 `OUTDATED_TOKENS` *(已废弃/过时的代币列表，用于过滤)* 集合过滤废弃代币
5. **统一大小写**：所有 key 转为大写保持一致性

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个获取常见代币列表的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useCommonTokens的hook，用来获取某个区块链网络上常见的代币列表。然后明确几个关键点。第一，代币要从两个地方来——一个是写死的基础代币列表，比如XLM、USDC这些核心资产，另一个是通过外部API获取的热门代币。第二，返回的数据格式应该是 `Record<string, Token>` 这样的映射表，地址作为key。第三，地址作为key的时候要统一转成大写，不然查来查去会乱套。第四，外部API可能会抽风，所以API查不到的时候要用硬编码的代币作为后备。第五，已经废弃的代币要过滤掉，不能让用户选到。第六，用React Query管理数据，而且因为热门代币不常变，缓存时间可以设长一点，比如一个小时。第七，刷新的时候要用keepPreviousData保持旧数据，不然列表会闪烁。

### 这里面有几个地方特别容易出错

容错机制一定要有，外部API不是百分百可靠的，万一它挂了你的应用不能跟着挂。缓存时间可以设长一点，因为热门代币确实变化不频繁，没必要反复去查。过期代币的列表要维护好，不然用户可能选到一个已经废弃的代币。地址作为key的时候统一转大写，这样查找的时候才不会因为大小写不一致而找不到。

### 数据刷新这里有讲究

刷新的时候要保持旧数据，这样用户看起来不会有一片空白的瞬间。缓存保留一天就够了，一天前的数据也没啥价值。API请求失败可以重试一次，但不要重试太多次，免得把API打爆。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useCommonTokens } from '@sushiswap/hooks'

function TokenList() {
  const { data: commonTokens, isLoading } = useCommonTokens()

  if (isLoading) return <p>加载中...</p>

  return (
    <div>
      <h2>常见代币</h2>
      <ul>
        {Object.values(commonTokens ?? {}).map((token) => (
          <li key={token.contract}>{token.symbol}</li>
        ))}
      </ul>
    </div>
  )
}
```

### 常见使用场景

1. **代币选择器基础数据**：为代币搜索提供基础列表
   ```tsx
   const { data: commonTokens } = useCommonTokens()
   const allTokens = { ...commonTokens, ...customTokens }
   ```

2. **显示热门代币**：展示用户最常用的代币
   ```tsx
   const popularTokens = Object.values(commonTokens).slice(0, 10)
   ```

3. **与自定义代币合并**：将常见代币和用户自定义代币合并使用
   ```tsx
   const { data: customTokens } = useCustomTokens()
   const mergedTokens = useMemo(() => ({
     ...commonTokens,
     ...customTokens,
   }), [commonTokens, customTokens])
   ```

### Dos and Don'ts

**Dos:**
- ✅ 与 `useCustomTokens` 配合使用，合并多个代币来源
- ✅ 设置较长的 `staleTime` 减少不必要的 API 调用
- ✅ 使用 `keepPreviousData` 避免列表闪烁

**Don'ts:**
- ❌ 不要直接修改返回的 data 对象
- ❌ 不要忽略 API 失败情况，要有后备方案
- ❌ 不要每次渲染时重新创建合并的代币对象，使用 `useMemo`
