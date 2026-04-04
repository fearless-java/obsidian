> 源代码路径: `apps/web/src/lib/hooks/api/useTokens.ts`

# useTokens

## 1. 大白话讲讲这个hook的作用

`useTokens` *(一个React hook，用于获取代币列表数据，支持搜索和过滤)* 帮你获取代币列表数据。

用于：
- 探索页面展示代币列表
- 搜索功能
- 代币排行榜

## 2. 讲讲为什么需要封装该hook

封装提供：
- 统一的数据获取接口
- 条件获取支持
- 类型安全

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
args: GetTokens
shouldFetch?: boolean
```

**输出：**
```typescript
{
  data: TokensResponse,
  isLoading: boolean,
  error: Error | null
}
```

## 4. 怎么给这个hook写AI提示词

这个hook用来查代币列表，可以指定链ID来查某条链上的代币，也可以加搜索关键字来过滤。

### 写提示词的小技巧

**第一，用shouldFetch控制请求。** 如果不需要这个数据（比如用户在看别的页面），就别发请求了。

**第二，支持搜索和过滤。** 可以传搜索关键字、排序方式、分页参数这些。

### 提示词模板

```
帮我写一个React hook，功能是获取代币列表。

具体需求：
1. 调用 getTokens 函数
2. 支持 shouldFetch 开关

返回：{ data: TokensResponse, isLoading, error }
```

### 实际用的例子

```typescript
const { data: tokens, isLoading, error } = useTokens({
  chainId: ChainId.ETHEREUM,
})
```

就是查列表，然后遍历显示。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 基础调用
const { data: tokens, isLoading, error } = useTokens({
  chainId: ChainId.ETHEREUM,
})

// tokens 是代币列表
console.log(tokens?.data) // 代币数组
console.log(tokens?.total) // 总数
```

### 常见使用场景

**场景1：代币搜索**

```typescript
const [searchQuery, setSearchQuery] = useState('')

const { data: tokens } = useTokens({
  chainId,
  search: searchQuery,
})

// 搜索过滤
const searchResults = tokens?.data?.filter(token =>
  token.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
  token.symbol.toLowerCase().includes(searchQuery.toLowerCase())
)
```

**场景2：代币排行榜**

```typescript
const { data: tokens } = useTokens({
  chainId,
  sortBy: 'volume', // 按交易量排序
  order: 'desc',
})

// 显示top代币
const topTokens = tokens?.data?.slice(0, 10)
```

**场景3：分页加载**

```typescript
const { data: tokens, fetchNextPage, hasNextPage } = useTokens({
  chainId,
  page: 1,
  limit: 50,
})
```

### Dos and Don'ts

**✅ Do:**
- 使用 `shouldFetch` 控制何时获取
- 处理 `isLoading` 和 `error` 状态
- 根据需要添加搜索和排序参数

**❌ Don't:**
- 不要在不需要时频繁调用
- 不要忽略错误处理
- 不要假设数据总是存在
