> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/token/use-sorted-token-list.ts`

# useSortedTokenList Hook Tutorial

## 大白话讲讲这个hook的作用

`useSortedTokenList` *(一个React hook，用于对代币列表进行搜索过滤和排序，基于React Query的缓存机制提高性能)* 是一个用于对代币列表进行搜索和排序的 hook。它的功能类似于一个"智能代币搜索器"：
- 根据用户输入的文字（query）过滤代币列表
- 按照相关性对代币排序
- 如果提供了余额数据（balanceMap），会优先显示余额不为零的代币

这个 hook 使用了 250ms 的防抖处理，避免用户快速输入时频繁触发搜索。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **搜索逻辑复杂性**：代币搜索需要处理 symbol 匹配、name 匹配、地址匹配等多种场景，封装后简化使用

2. **排序优先级**：需要按照"余额 > 相关性 > 字母顺序"的优先级排序

3. **防抖优化**：输入搜索时不需要每次按键都触发搜索，防抖可以减少不必要的计算

4. **缓存优化**：使用 React Query 缓存搜索结果

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  query: string                    // 用户搜索框输入的文字
  tokenMap?: Record<string, Token> // 代币映射表
  balanceMap?: Record<string, string> // 代币余额映射表
}
```

### 输出（返回值）
```typescript
// React Query 返回值，包含：
{
  data: Token[]                    // 排序和过滤后的代币列表
  isLoading: boolean
  isPending: boolean
  ...
}
```

### 核心执行逻辑

1. **防抖**：将 query 防抖 250ms 后使用
2. **过滤**：调用 `filterTokens()` *(一个过滤函数，用于根据搜索词匹配代币的symbol、name或address)* 过滤代币
3. **排序**：调用 `getSortedTokensByQuery()` *(一个排序函数，根据搜索相关性对代币排序)* 按相关性排序
4. **余额排序**：如果有 balanceMap，将代币按余额从大到小排序

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个搜索和排序代币列表的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useSortedTokenList的hook，用来在代币列表里搜索和排序。然后明确几个关键点。第一，参数要包括搜索词、代币列表的映射表和余额的映射表这三个。第二，搜索词要经过防抖处理再使用，不然用户每次敲字都触发一次搜索会很卡，推荐250毫秒的防抖间隔。第三，搜索的时候要能匹配代币的符号、名称和地址，而且要忽略大小写，用户搜大写小写都应该能查到。第四，如果有余额数据，要优先显示余额大于零的代币，用户有钱的代币当然要排前面。第五，用React Query管理缓存。第六，返回排序好了的代币列表。

### 这里面有几个地方特别容易出错

防抖一定要做，不做的话用户输入的时候会很卡，体验很差。搜索匹配的时候要忽略大小写，不然用户搜"usdc"找不到"USDC"就很烦。tokenMap或者balanceMap是空的时候也要能正常工作，不能一空就报错。

### 数据刷新这里有讲究

防抖用useDebounce这个hook来处理就行，很方便。queryKey要包含所有的依赖项，这样缓存才能正确工作。刷新的时候要保持旧数据，不然列表会闪烁，用户体验会很差。这些细节都注意到了，搜索的体验就会很流畅。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useSortedTokenList } from '@sushiswap/hooks'
import { useCustomTokens, useCommonTokens } from './hooks'

function TokenSearch() {
  const [query, setQuery] = useState('')
  const { data: customTokens } = useCustomTokens()
  const { data: commonTokens } = useCommonTokens()

  // 合并代币列表
  const allTokens = { ...commonTokens, ...customTokens }

  // 使用hook搜索和排序
  const { data: sortedTokens, isLoading } = useSortedTokenList({
    query,
    tokenMap: allTokens,
    balanceMap: userBalances,
  })

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {isLoading ? (
        <p>搜索中...</p>
      ) : (
        <ul>
          {sortedTokens?.map((token) => (
            <li key={token.contract}>{token.symbol}</li>
          ))}
        </ul>
      )}
    </div>
  )
}
```

### 常见使用场景

1. **代币选择器**：在 swap 界面让用户搜索选择代币
   ```tsx
   const { data: tokens } = useSortedTokenList({
     query: searchQuery,
     tokenMap: allTokens,
   })
   ```

2. **钱包代币列表排序**：优先显示用户有余额的代币
   ```tsx
   const { data: tokens } = useSortedTokenList({
     query: '',
     tokenMap: allTokens,
     balanceMap: balances,
   })
   ```

### Dos and Don'ts

**Dos:**
- ✅ 合并多个代币来源（commonTokens + customTokens）后再传入
- ✅ 配合防抖的输入组件使用，提升用户体验
- ✅ 设置合理的 `staleTime` 避免频繁重新获取

**Don'ts:**
- ❌ 不要在每次渲染时都传入新的 tokenMap 对象，用 `useMemo` 缓存
- ❌ 不要忽略 `isPending` 状态，应该在搜索时显示加载指示器
- ❌ 不要传入过大的代币列表没有分页，会影响性能
