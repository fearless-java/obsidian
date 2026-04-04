> 源代码路径: `apps/web/src/lib/hooks/useTypedSearchParams.ts`

# useTypedSearchParams

## 1. 大白话讲讲这个hook的作用

`useTypedSearchParams` *(一个React hook，用于从URL查询参数中读取数据并使用Zod schema进行类型验证，实现类型安全的参数解析)* 是一个帮你从URL查询参数中读取数据并进行类型验证的hook。

比如你的URL是 `https://sushi.com/swap?tokenIn=0x...&tokenOut=0x...&amount=1000`，这个hook可以：
- 自动解析这些查询参数
- 使用 Zod schema *(一个TypeScript优先的模式声明和验证库，用于定义输入数据的结构和类型)* 验证数据类型
- 返回类型安全的解析结果

简单说，它就是URL参数和类型安全代码之间的桥梁。

## 2. 讲讲为什么需要封装该hook

直接使用 `useSearchParams` *(Next.js提供的hook，用于访问URL查询参数，返回StringStringSearchParams对象)* 的问题：
- 返回的都是字符串，需要手动转换类型
- 没有验证，错误的参数会直接导致bug
- 复杂的解析逻辑（如处理数组、嵌套对象）散落在组件中
- URL参数来源不可控，可能被用户篡改

封装后：
- 用 Zod schema 定义参数结构，一次定义多处复用
- 自动类型转换（字符串 "123" -> 数字 123）
- 验证失败时抛出清晰的错误
- 返回类型由 schema 推断，无需手动声明

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
schema: z.ZodSchema  // Zod验证schema
```

**输出：**
```typescript
z.infer<typeof schema>  // 根据schema推断的类型
```

**执行逻辑：**
1. 调用 Next.js 的 `useSearchParams()` 获取原始查询参数
2. 将 URLSearchParams 转换为普通对象：`Object.fromEntries(searchParams.entries())`
3. 使用传入的 Zod schema 解析这个对象
4. 返回解析后的类型化数据

**数据流：**
```
URL: ?tokenIn=0x...&amount=1000
         |
         v
useSearchParams() --> { tokenIn: "0x...", amount: "1000" }
         |
         v
schema.parse() --> { tokenIn: Address, amount: bigint } (类型安全)
         |
         v
组件使用
```

## 4. 怎么给这个hook写AI提示词

这个hook干的事儿简单说就是：从URL里读参数，然后用Zod这个工具给参数做"体检"，确保类型安全。URL里的参数都是字符串，但它能帮你转成数字、布尔值这些正经类型。

### 写提示词的小技巧

**第一，schema要抽到组件外面去定义。** 每个组件里都重新定义一遍schema太浪费了，而且也不好复用。在文件顶部或者单独的文件夹里定义好，多个组件都能用。

**第二，可选参数一定要给默认值。** 用 `.default()` 可以给可选参数一个默认值，这样后面代码处理起来就省心很多，不用老去判断是不是 undefined。

**第三，字符串转数字用 `z.coerce`。** URL里 "123" 是字符串，但代码里你可能需要数字。`z.coerce.number()` 会自动把这个 "123" 转成 123。类似的还有 `z.coerce.boolean()`、`z.coerce.bigint()`。

**第四，验证失败要有应对方案。** `schema.parse()` 验证失败会抛错。可以考虑用 `z.safeParse()` 处理，它不会抛错，而是返回一个对象告诉你成没成功。

### 写提示词时要注意的条条框框

**URL参数本质上都是字符串：** 不管你看到的URL是 "?amount=100" 还是 "?active=true"，从 useSearchParams 拿到的都是字符串。所以数字、布尔这些类型必须显式转换。

**特殊类型需要额外验证：** 比如EVM地址，光是字符串不够，还要验证格式对不对（是不是0x开头、是不是40位十六进制）。可以用 `.regex()` 或者自定义验证函数。

**这个hook本身不维护状态：** 它只是读取URL，然后通过schema验证和转换。每次URL变了，它就会重新渲染一次，返回新的解析结果。

### 提示词模板

```
帮我写一个React hook，功能是用Zod验证schema来解析URL里的查询参数。

具体需求：
1. 用Next.js的useSearchParams拿到URL参数
2. 用传入的Zod schema解析这些参数
3. 自动把字符串转成需要的类型（比如数字、布尔值）
4. 验证失败的参数要报错或者给默认值
5. 返回的类型由schema自动推断出来，不用手写

大概长这样：
function useTypedSearchParams<T extends z.Schema>(schema: T): z.infer<T>

使用例子：
const schema = z.object({
  tokenIn: z.string().optional(),
  amount: z.coerce.number().default(0),
  chainId: z.coerce.number(),
})

const params = useTypedSearchParams(schema)
// params 的类型自动推断为 { tokenIn?: string, amount: number, chainId: number }

注意事项：
- URL参数都是字符串，必须用z.coerce处理需要转类型的字段
- 用.default()给可选参数提供备选值
- schema最好定义在组件外面复用
```

### 实际用的例子

```typescript
// 先定义好schema，写在组件外面方便复用
const swapParamsSchema = z.object({
  tokenIn: z.string().optional(),
  tokenOut: z.string().optional(),
  amount: z.coerce.bigint().optional(),
  chainId: z.coerce.number(),
  slippage: z.coerce.number().min(0.01).max(50).default(0.5),
})

// 在组件里用
const params = useTypedSearchParams(swapParamsSchema)

// 在组件中使用
useEffect(() => {
  if (params.amount) {
    // do something with amount
  }
}, [params.amount])
```

这里用 `.min(0.01).max(50)` 对滑点做了范围限制，用户如果输入超过50就会验证失败。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 1. 定义schema（推荐在单独文件中定义复用）
import { z } from 'zod'

const swapSchema = z.object({
  tokenIn: z.string().optional(),
  tokenOut: z.string().optional(),
  amount: z.coerce.bigint().optional(),
  chainId: z.coerce.number(),
  slippage: z.coerce.number().min(0.01).max(50).default(0.5),
})

// 2. 在组件中使用
const params = useTypedSearchParams(swapSchema)

// 3. params已经是类型安全的对象
console.log(params.chainId) // number类型，不需要手动转换
```

### 常见使用场景

**场景1：Swap页面的参数管理**

```typescript
// 定义完整的swap参数schema
const swapSchema = z.object({
  tokenIn: z.string().optional(),
  tokenOut: z.string().optional(),
  amount: z.coerce.bigint().optional(),
  chainId: z.coerce.number(),
  slippage: z.coerce.number().min(0.01).max(50).default(0.5),
})

// 使用
const params = useTypedSearchParams(swapSchema)

// 用于获取代币数据
const tokenIn = useToken(params.tokenIn)
const tokenOut = useToken(params.tokenOut)
```

**场景2：带验证的地址类型**

```typescript
// 自定义地址类型验证
const addressSchema = z.string().refine(
  (val) => /^0x[a-fA-F0-9]{40}$/.test(val),
  { message: '无效的EVM地址格式' }
)

const poolSchema = z.object({
  token0: addressSchema,
  token1: addressSchema,
})

const params = useTypedSearchParams(poolSchema)
```

**场景3：分页参数**

```typescript
const listSchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  sort: z.enum(['asc', 'desc']).optional(),
})

const params = useTypedSearchParams(listSchema)

// URL: ?page=2&limit=50
// params = { page: 2, limit: 50, sort: undefined }
```

### Dos and Don'ts

**✅ Do:**
- 在应用顶层或单独文件中定义共享的schema，便于多个组件复用
- 使用 `z.coerce` 自动处理字符串到数字/布尔/bigint的转换
- 使用 `.default()` 为可选参数提供合理的默认值
- 使用 `.min()` 和 `.max()` 添加数值范围验证

**❌ Don't:**
- 不要在每次组件渲染时重新定义schema，应该在外部定义复用
- 不要假设参数一定存在，使用可选链 `?.` 或默认值处理
- 不要忽略验证错误，应该使用 `z.safeParse` 替代 `z.parse` 来优雅处理失败
- 不要在schema中包含敏感信息，URL参数对用户可见
