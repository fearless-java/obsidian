> 源代码路径: `apps/web/src/lib/hooks/api/usePoolsInfinite.ts`

# usePoolsInfinite

## 1. 大白话讲讲这个hook的作用

`usePoolsInfinite` *(一个React hook，用于无限滚动获取池子列表，自动管理分页和加载更多)* 帮你"无限滚动"获取池子列表。

特点：
- 类似于分页，但用户滚动到底部自动加载更多
- 返回的是分页后的数据页数组
- 提供 `fetchNextPage` 函数手动触发加载

## 2. 讲讲为什么需要封装该hook

无限滚动分页很复杂：
- 需要管理 pageParam
- 需要合并多页数据
- 需要判断是否有更多页

封装后：
- 使用 react-query 的 `useInfiniteQuery` *(React Query的hook，用于无限滚动等分页场景，自动管理pageParam和多页数据合并)*
- 自动管理分页状态
- 提供简洁的接口

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
Omit<GetPools, 'page'>  // 去掉page参数的查询参数
```

**输出：**
```typescript
{
  data: {
    pages: GetPoolsResult[]  // 数据页数组
    pageParams: number[]    // 页码数组
  },
  fetchNextPage: () => void
  hasNextPage: boolean
  isLoading: boolean
  error: Error | null
}
```

**执行逻辑：**
1. 使用 `useInfiniteQuery` 包装
2. `queryFn` 接收 `{ pageParam: page }`，调用 `getPools({ ...args, page })`
3. `initialPageParam: 1`
4. `getNextPageParam` 返回 `pages.length + 1`
5. 自动缓存和分页

**数据流：**
```
滚动触发
    |
    v
fetchNextPage()
    |
    v
getPools({ ...args, page: nextPage })
    |
    v
pages.push(newPage)
    |
    v
hasNextPage = pages.length + 1 < totalPages
```

## 4. 怎么给这个hook写AI提示词

无限滚动的核心就是：用户滚到底部了，触发加载更多。这个hook帮你管理分页逻辑——初始页是1，每次加载下一页自动加1，直到没有更多数据为止。

### 写提示词的小技巧

**第一，数据的结构是 pages 数组。** 每一页是一个对象，里面有这页的数据和下一页的参数。别想着 data 就是个数组，它是个有嵌套结构的东东。

**第二，合并多页数据用 flatMap。** `data?.pages.flatMap(page => page.pools)` 这样能拿到所有页的池子列表。打平了才能直接遍历。

**第三，fetchNextPage要手动调用。** 不是自动的，得在用户滚动到底部的时候（比如 onScroll 或者 IntersectionObserver 检测到可见元素）才调用。

**第四，hasNextPage要先检查。** 还没到最后一页才调用 fetchNextPage，不然会发空请求。

### 写提示词时要注意的条条框框

**初始页码是1，不是0。** 这个很重要，很多人数数数着数着就忘了。getNextPageParam 要返回下一页的页码，是当前页数加1。

**getNextPageParam决定还有没有下一页。** 返回 undefined 就表示没了，返回数字就还有。

### 提示词模板

```
帮我写一个React hook，功能是无限滚动获取池子列表。

具体需求：
1. 用 react-query 的 useInfiniteQuery 来做
2. 调用 getPools 函数，自动加上 page 参数
3. 提供 fetchNextPage（手动触发加载更多）和 hasNextPage（还有没有更多）
4. 数据结构是 pages 数组，每页有自己的池子列表

使用例子：
const { data, fetchNextPage, hasNextPage } = usePoolsInfinite({
  chainId: 1,
  protocol: SushiSwapProtocol.SUSHISWAP_V3,
})

// 合并所有页的池子
const allPools = data?.pages.flatMap(page => page.pools) ?? []
```

### 实际用的例子

```typescript
const { data, fetchNextPage, hasNextPage } = usePoolsInfinite({
  chainId: ChainId.ETHEREUM,
  protocol: SushiSwapProtocol.SUSHISWAP_V3,
})

// 无限滚动触发器
<div onScroll={handleScroll}>
  {data?.pages.flatMap(p => p.pools).map(pool => (
    <PoolRow key={pool.id} pool={pool} />
  ))}
  {hasNextPage && <LoadingSpinner onInView={fetchNextPage} />}
</div>
```

IntersectionObserver 是个好东西，可以监听一个元素是否进入视口。进入视口了就调用 fetchNextPage，比 onScroll 性能好。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 基础调用
const {
  data,
  fetchNextPage,
  hasNextPage,
  isLoading,
  isFetchingNextPage,
  error,
} = usePoolsInfinite({
  chainId: ChainId.ETHEREUM,
  protocol: SushiSwapProtocol.SUSHISWAP_V3,
})

// 合并所有页面的池子
const allPools = data?.pages.flatMap(page => page.pools) ?? []
```

### 常见使用场景

**场景1：无限滚动池子列表**

```typescript
const { data, fetchNextPage, hasNextPage } = usePoolsInfinite({...})

// 滚动加载更多
const handleScroll = (e) => {
  const { scrollTop, scrollHeight, clientHeight } = e.target
  if (scrollHeight - scrollTop <= clientHeight * 1.5) {
    if (hasNextPage && !isFetchingNextPage) {
      fetchNextPage()
    }
  }
}

return (
  <div onScroll={handleScroll}>
    {data?.pages.flatMap(p => p.pools).map(pool => (
      <PoolCard key={pool.id} pool={pool} />
    ))}
    {isFetchingNextPage && <LoadingSpinner />}
  </div>
)
```

**场景2：Intersection Observer 触发加载**

```typescript
const { data, fetchNextPage, hasNextPage } = usePoolsInfinite({...})

// 使用 IntersectionObserver 自动触发加载
useEffect(() => {
  const observer = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting && hasNextPage) {
      fetchNextPage()
    }
  }, { threshold: 0.1 })

  if (loadingRef.current) {
    observer.observe(loadingRef.current)
  }

  return () => observer.disconnect()
}, [hasNextPage, fetchNextPage])

return (
  <div>
    {data?.pages.flatMap(p => p.pools).map(pool => (
      <PoolCard key={pool.id} pool={pool} />
    ))}
    <div ref={loadingRef}>
      {hasNextPage && <LoadingSpinner />}
    </div>
  </div>
)
```

**场景3：下拉刷新**

```typescript
const {
  data,
  fetchNextPage,
  hasNextPage,
  refetch,
  isRefetching,
} = usePoolsInfinite({...})

// 下拉刷新
const handleRefresh = () => {
  refetch()
}
```

### Dos and Don'ts

**✅ Do:**
- 使用 `flatMap` 正确合并多页数据
- 在触发加载前检查 `hasNextPage`
- 使用 IntersectionObserver 或 scroll 事件触发 `fetchNextPage`
- 处理 `isFetchingNextPage` 状态显示加载指示

**❌ Don't:**
- 不要直接访问 `data[0]`，应该用 `data?.pages[0]`
- 不要忘记 `hasNextPage` 检查，可能导致无限请求
- 不要在组件卸载后调用 `fetchNextPage`
- 不要忽略 `error` 处理
