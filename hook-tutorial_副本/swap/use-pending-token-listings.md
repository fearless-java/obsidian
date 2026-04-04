> 源代码路径: `apps/web/src/lib/hooks/api/usePendingTokenListings.ts`

# usePendingTokenListings

## 1. 大白话讲讲这个hook的作用

`usePendingTokenListings` *(一个React hook，用于获取SushiSwap待审核的代币列表，通常用于管理后台)* 帮你获取"待审核的代币列表"。

SushiSwap代币上线流程：
1. 社区成员提交代币申请
2. 代币进入"待审核"状态
3. 管理员/社区投票决定是否批准

这个hook返回所有当前处于"待审核"状态的代币，供管理界面使用。

## 2. 讲讲为什么需要封装该hook

封装提供：
- 统一的API调用封装
- 自动处理加载状态
- 条件获取支持

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
shouldFetch?: boolean  // 默认true
```

**输出：**
```typescript
{
  data: PendingTokens,  // 待审核代币
  isLoading: boolean,
  error: Error | null
}
```

**执行逻辑：**
1. 调用 `getPendingTokens()` *(获取待审核代币列表的API函数)*
2. 返回结果

## 4. 怎么给这个hook写AI提示词

这个hook是给管理后台用的，获取那些还在等待审核的代币列表。普通用户界面用不上这个，只有管理员或者审核人员才需要看。

### 写提示词的小技巧

**第一，用shouldFetch控制获取。** 不是管理员的话根本没必要发这个请求，用开关控制住。

**第二，空结果要能正常显示。** 没有待审核的代币时，列表就是空的，UI要能handle住这个情况，显示个"暂无待审核"之类的提示就行。

### 提示词模板

```
帮我写一个React hook，功能是获取待审核的代币列表。

具体需求：
1. 调用 getPendingTokens 函数获取数据
2. 支持 shouldFetch 开关

返回：标准的react-query结果
```

### 实际用的例子

```typescript
const { data: pendingTokens } = usePendingTokenListings()

// 管理界面显示待审核代币
```

简单明了，就是拿列表然后显示。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 基础调用
const { data: pendingTokens, isLoading, error } = usePendingTokenListings()

// 条件获取
const { data: pendingTokens } = usePendingTokenListings(shouldFetch)
```

### 常见使用场景

**场景1：代币审核管理界面**

```typescript
const { data: pendingTokens, isLoading } = usePendingTokenListings()

return (
  <AdminPanel title="待审核代币">
    {isLoading ? (
      <LoadingSpinner />
    ) : pendingTokens?.length === 0 ? (
      <EmptyState message="暂无待审核代币" />
    ) : (
      pendingTokens?.map(token => (
        <ReviewCard
          key={token.address}
          token={token}
          onApprove={() => handleApprove(token.address)}
          onReject={() => handleReject(token.address)}
        />
      ))
    )}
  </AdminPanel>
)
```

**场景2：显示待审核数量**

```typescript
const { data: pendingTokens } = usePendingTokenListings()

// 在导航栏显示待审核数量
return (
  <Navbar>
    <NavItem>代币管理</NavItem>
    {pendingTokens?.length > 0 && (
      <Badge count={pendingTokens.length} />
    )}
  </Navbar>
)
```

**场景3：批量操作**

```typescript
const { data: pendingTokens } = usePendingTokenListings()

// 批量批准
const handleBatchApprove = (addresses: string[]) => {
  addresses.forEach(address => {
    approveToken(address)
  })
}

// 批量拒绝
const handleBatchReject = (addresses: string[]) => {
  addresses.forEach(address => {
    rejectToken(address)
  })
}
```

### Dos and Don'ts

**✅ Do:**
- 使用 `shouldFetch` 在不需要时禁用获取
- 处理空状态，当没有待审核代币时显示友好提示
- 在管理界面使用，控制权限

**❌ Don't:**
- 不要在普通用户界面显示待审核代币，这些是内部数据
- 不要忽略 `error` 处理
- 不要缓存太久，审核状态可能随时变化
- 不要在非管理页面调用这个 hook
