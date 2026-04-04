> 源代码路径: `apps/web/src/lib/hooks/react-query/tokenlist/useApplyForTokenList.ts`

# useApplyForTokenList Hook Tutorial

## 1. 大白话讲讲这个hook的作用

`useApplyForTokenList` 是一个用于申请将代币添加到 SushiSwap 代币列表（Token List）的 mutation Hook。用户（或项目方）可以提交代币的详细信息和 logo，申请将代币列入 SushiSwap 的官方代币列表。

简单来说：**这是一个"我要上币"的申请表单提交 Hook —— 它把用户的申请数据发送到后端 API，完成代币列表的申请流程。**

---

## 2. 为什么需要封装该hook

### 原始问题
1. **这是一个写操作**：不是查询数据，而是提交数据到服务器，需要用 `useMutation` *(React Query中用于处理写操作的hook，如POST、PUT、DELETE)* 而不是 `useQuery`
2. **请求体构造复杂**：需要把用户的表单数据构造成特定的 JSON 格式
3. **错误处理特殊**：API 可能返回错误信息（如"代币已存在"），需要把这些错误信息转换为异常
4. **没有轮询/缓存**：mutation 不需要缓存数据，每次调用都是全新的请求

### 封装带来的好处
1. **开箱即用的 mutation**：直接返回 React Mutation 的所有状态和方法
2. **类型安全**：参数和返回都有完整的 TypeScript 类型
3. **自动 JSON 序列化**：自动设置正确的 Content-Type 和 Accept 头
4. **友好的错误处理**：API 返回非 200 状态码时抛出有意义的错误信息

---

## 3. 执行逻辑和数据流向

### 输入 (Params)
```typescript
{
  chainId: number           // 链ID
  listType: string          // 代币列表类型
  logoFile: string          // logo 文件（base64 或 URL）
  tokenDecimals: number     // 代币精度
  tokenAddress: string      // 代币地址
  tokenSymbol: string       // 代币符号
  tokenName?: string        // 代币名称（可选）
}
```

### 输出 (Return)
```typescript
{
  mutate: (params) => void
  mutateAsync: (params) => Promise<any>  // 可 await 的版本
  isPending: boolean
  isError: boolean
  error: Error | null
  data: any  // API 返回的成功数据
}
```

### 执行流程

```
1. useApplyForTokenList()
       |
       v
2. 返回 useMutation 对象
       |
       v
3. 用户调用 mutate({ chainId, listType, ... })
       |
       v
4. 构造请求:
   POST /tokenlist-request/api/submit
   Content-Type: application/json
   Body: {
     tokenAddress, tokenSymbol, tokenName,
     tokenDecimals, logoFile, chainId, listType
   }
       |
       v
5. 检查响应状态:
   - 如果 status !== 200: throw new Error(json.error)
   - 如果 status === 200: return json
       |
       v
6. React Query 更新 mutation 状态
```

---

## 4. 怎么用 AI 写这个 Hook

### 写代码时要注意什么

1. **要用 useMutation**：这是提交数据的操作，不是查数据，必须用 useMutation，不能用 useQuery。

2. **错误要转换成抛出异常**：API 返回错误状态码的时候，要用 `throw new Error` 抛出去，不要自己返回一个错误状态。

3. **参数要解构**：mutationFn 的参数要解构，这样外面调用的时候更方便。

4. **参数简单就好**：不需要什么 enabled、staleTime 之类的配置，就是一个纯粹的提交流程。

### 有什么限制条件

1. **这是 mutation 不是 query**：是往服务器提交数据的操作，不是从服务器拉数据。

2. **依赖内部后端 API**：要调用 `/tokenlist-request/api/submit` 这个内部接口。

3. **错误处理是同步抛异常**：用 `throw new Error` 的方式处理错误，不是返回个错误码。

4. **不会自动执行**：mutation 必须手动调用 `mutate()` 才会真正执行，不会自动触发。

### 状态怎么管理

| 状态 | 怎么管 | 什么意思 |
|------|--------|---------|
| 提交状态 | React Mutation isPending | 正在提交中 |
| 错误状态 | React Mutation isError | 提交失败了 |
| 返回数据 | React Mutation data | API 返回的成功数据 |
| 缓存 | 没有 | mutation 不走缓存 |

---

### 完整AI提示词模板

```
你是一个 React Query 专家。请为以下场景编写 Hook:

【场景描述】
需要创建一个用于申请将代币添加到 SushiSwap 代币列表的 mutation Hook。
这是一个写操作，需要 POST 数据到后端 API。

【技术要求】
1. 使用 @tanstack/react-query useMutation
2. POST /tokenlist-request/api/submit
3. Content-Type: application/json
4. 请求体包含所有代币信息

【参数】
interface UseApplyForTokenListParams {
  chainId: number
  listType: string
  logoFile: string
  tokenDecimals: number
  tokenAddress: string
  tokenSymbol: string
  tokenName?: string
}

【mutationFn 实现】
1. 构造 JSON 请求体
2. fetch POST 请求
3. 检查 response.status
4. 非 200 状态: throw new Error(json.error)
5. 200 状态: return json

【返回】
useMutation 的标准返回:
- mutate(params)
- mutateAsync(params)
- isPending
- isError
- error
- data

【最佳实践】
- 使用 useMutation 而非 useQuery
- 错误转换为 throw Error
- 保持参数解构简洁
- 不需要缓存配置

请输出完整代码。
```

---

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useApplyForTokenList } from '@sushiswap/react-query'

function TokenApplicationForm() {
  const {
    mutate: submitToken,
    isPending,
    isError,
    error,
    isSuccess,
  } = useApplyForTokenList()

  const handleSubmit = (formData) => {
    submitToken({
      chainId: formData.chainId,
      listType: 'default', // 或 'community'
      logoFile: formData.logoBase64,
      tokenDecimals: formData.decimals,
      tokenAddress: formData.address,
      tokenSymbol: formData.symbol,
      tokenName: formData.name,
    })
  }

  if (isSuccess) return <SuccessMessage>Token application submitted!</SuccessMessage>
  if (isError) return <ErrorMessage>{error?.message}</ErrorMessage>

  return (
    <Form onSubmit={handleSubmit}>
      <FormSubmitButton disabled={isPending}>
        {isPending ? 'Submitting...' : 'Submit Application'}
      </FormSubmitButton>
    </Form>
  )
}
```

### 常见使用场景

**场景1：项目方申请上币**
```tsx
function TokenListApplication() {
  const [formData, setFormData] = useState({
    chainId: 1,
    listType: 'default',
    tokenAddress: '',
    tokenSymbol: '',
    tokenName: '',
    tokenDecimals: 18,
    logoFile: '',
  })

  const { mutate: submit, isPending, isError, error, isSuccess } = useApplyForTokenList()

  const handleLogoUpload = (file) => {
    // 转换为 base64
    const reader = new FileReader()
    reader.onload = () => {
      setFormData({ ...formData, logoFile: reader.result })
    }
    reader.readAsDataURL(file)
  }

  const handleSubmit = (e) => {
    e.preventDefault()
    submit(formData)
  }

  return (
    <form onSubmit={handleSubmit}>
      {/* 表单字段 */}
      <input
        type="text"
        placeholder="Token Address"
        value={formData.tokenAddress}
        onChange={(e) => setFormData({ ...formData, tokenAddress: e.target.value })}
      />

      <input
        type="text"
        placeholder="Symbol (e.g. SUSHI)"
        value={formData.tokenSymbol}
        onChange={(e) => setFormData({ ...formData, tokenSymbol: e.target.value })}
      />

      <input
        type="file"
        accept="image/*"
        onChange={(e) => handleLogoUpload(e.target.files[0])}
      />

      <button type="submit" disabled={isPending}>
        {isPending ? 'Submitting...' : 'Apply for Token List'}
      </button>

      {isSuccess && <p className="text-green-600">Application submitted successfully!</p>}
      {isError && <p className="text-red-600">Error: {error?.message}</p>}
    </form>
  )
}
```

**场景2：异步提交并跳转**
```tsx
function TokenApplyWithNavigation() {
  const router = useRouter()
  const { mutate: submitApplication, isPending } = useApplyForTokenList()

  const handleApply = async (formData) => {
    try {
      await submitApplication(formData)
      // 提交成功后跳转
      router.push('/token-list/pending')
    } catch (err) {
      console.error('Failed to submit:', err)
    }
  }

  return (
    <Button onClick={() => handleApply(formData)} disabled={isPending}>
      {isPending ? 'Submitting...' : 'Submit and View Status'}
    </Button>
  )
}
```

### Dos and Don'ts

**Do（推荐做法）：**
- ✅ 使用 `isPending` 显示提交中的 loading 状态
- ✅ 使用 `isSuccess` 判断提交成功并显示成功消息
- ✅ 使用 `error?.message` 显示友好的错误信息
- ✅ 使用 `mutateAsync` 如果需要 await 结果

**Don't（避免做法）：**
- ❌ 不要在 mutation 中使用 `enabled` 参数，mutation 不支持
- ❌ 不要忽略 `isError` 状态，提交失败应该告知用户
- ❌ 不要在 `mutate` 调用时直接使用 async/await，用 `mutateAsync` 代替
- ❌ 不要假设提交一定成功，应该处理错误情况

### 注意事项

1. **这是 useMutation 不是 useQuery**：这是提交操作，不是查询，所以不会缓存数据

2. **手动触发**：必须调用 `mutate()` 或 `mutateAsync()` 才会执行提交

3. **isPending vs isLoading**：`isPending` 是 React Query v5 中的命名，v4 中叫 `isLoading`

4. **错误处理**：API 返回非200状态时会 throw Error，需要用 try/catch 或 isError 状态处理

5. **无缓存**：mutation 的结果不会被 React Query 缓存
