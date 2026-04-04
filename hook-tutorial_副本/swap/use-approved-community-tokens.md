> 源代码路径: `apps/web/src/lib/hooks/api/useApprovedCommunityTokens.ts`

# useApprovedCommunityTokens

## 1. 大白话讲讲这个hook的作用

`useApprovedCommunityTokens` *(一个React hook，用于获取SushiSwap已批准的社区代币列表)* 帮你获取"已批准的社区代币列表"。

SushiSwap有一个社区代币上币流程：
- 任何人可以提交代币申请
- 社区投票决定是否通过
- 通过的代币会进入"已批准列表"

这个hook就是获取这个列表，返回所有已批准的社区代币信息。

## 2. 讲讲为什么需要封装该hook

直接调用图数据库API很繁琐：
- 需要处理分页
- 需要处理加载/错误状态
- 接口URL和参数可能变化

封装后：
- 简洁的hook接口
- 自动处理请求状态
- 错误处理和loading状态

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
shouldFetch?: boolean  // 是否应该获取，默认true
```

**输出：**
```typescript
{
  data: ApprovedCommunityTokens,  // 已批准代币数据
  isLoading: boolean,
  error: Error | null
}
```

**执行逻辑：**
1. 调用 `getApprovedCommunityTokens()` *(获取已批准社区代币的API函数)* 获取数据
2. 除非 `shouldFetch === false`，否则禁用查询
3. 返回标准react-query结果

**数据流：**
```
shouldFetch
    |
    v
getApprovedCommunityTokens()
    |
    v
图数据库 --> ApprovedCommunityTokens
    |
    v
组件使用
```

## 4. 怎么给这个hook写AI提示词

这个hook干的事儿很简单，就是从后端拿到SushiSwap已经批准的社区代币列表。比如有些代币社区投票通过了，它们就会出现在这个列表里。

### 写提示词的小技巧

**第一，用shouldFetch控制要不要获取。** 不是什么页面都需要这个列表，所以加个开关。比如用户根本没在浏览上币页面，就没必要发请求。

**第二，用标准的react-query接口。** 这样用起来顺手，isLoading、error这些属性都是现成的，UI也好处理。

**第三，错误要传上去但别卡住UI。** 请求失败了，error要有，但别因为这个把整个页面搞崩了。

### 写提示词时要注意的条条框框

**shouldFetch默认是true。** 大部分场景下是需要获取的，所以默认开着用。

**不需要传参数。** 这个hook不需要输入任何参数，它就是个简单的列表获取。

### 提示词模板

```
帮我写一个React hook，功能是获取SushiSwap已批准的社区代币列表。

具体需求：
1. 调用 getApprovedCommunityTokens 函数获取数据
2. 支持 shouldFetch 开关来控制要不要发请求
3. 返回标准的react-query结果（data、isLoading、error这些）

返回类型：
{
  data: ApprovedCommunityTokens
  isLoading: boolean
  error: Error | null
}

注意事项：
- shouldFetch 默认为 true
- 错误正常传播
```

### 实际用的例子

```typescript
const { data: approvedTokens, isLoading } = useApprovedCommunityTokens()

// 显示已批准代币列表
{approvedTokens?.map(token => (
  <TokenRow key={token.address} token={token} />
))}
```

没什么花头，就是拿到列表然后遍历显示。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 基础调用
const { data: approvedTokens, isLoading, error } = useApprovedCommunityTokens()

// 有条件地获取
const { data: approvedTokens } = useApprovedCommunityTokens(shouldFetch)
```

### 常见使用场景

**场景1：代币申请页面的白名单验证**

```typescript
const { data: approvedTokens } = useApprovedCommunityTokens()

// 检查代币是否在白名单中
const isApproved = (tokenAddress: string) => {
  return approvedTokens?.some(token =>
    token.address.toLowerCase() === tokenAddress.toLowerCase()
  )
}

// 使用
if (!isApproved(newToken.address)) {
  showWarning('该代币尚未被批准，请先申请上币')
}
```

**场景2：显示已批准代币列表**

```typescript
const { data: approvedTokens, isLoading } = useApprovedCommunityTokens()

return (
  <div>
    {isLoading ? (
      <LoadingSpinner />
    ) : (
      approvedTokens?.map(token => (
        <TokenCard
          key={token.address}
          name={token.name}
          symbol={token.symbol}
          address={token.address}
        />
      ))
    )}
  </div>
)
```

**场景3：搜索和过滤**

```typescript
const { data: approvedTokens } = useApprovedCommunityTokens()
const [searchQuery, setSearchQuery] = useState('')

// 过滤代币
const filteredTokens = approvedTokens?.filter(token =>
  token.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
  token.symbol.toLowerCase().includes(searchQuery.toLowerCase())
)
```

### Dos and Don'ts

**✅ Do:**
- 使用 `shouldFetch` 控制何时获取数据，避免不必要的请求
- 处理 `isLoading` 状态，给用户加载反馈
- 处理 `error` 状态，展示错误信息

**❌ Don't:**
- 不要在没有检查 `shouldFetch` 的情况下频繁调用
- 不要假设数据总是存在，检查 `data?.length`
- 不要忽略错误处理，API 可能失败
- 不要在每次渲染时都调用，应该根据页面需求控制
