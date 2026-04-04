> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/token/use-custom-tokens.ts`

# useCustomTokens Hook Tutorial

## 大白话讲讲这个hook的作用

`useCustomTokens` *(一个React hook，用于管理用户在Stellar网络上自定义添加的代币列表)* 是一个用于管理用户在 Stellar 网络上自定义添加的代币的 hook。它就像一个"我的自定义代币收藏夹"，允许用户：
- 添加自己发现或信任的代币到本地存储
- 删除不再需要的自定义代币
- 检查某个代币是否已经在自定义列表中

这个 hook 使用浏览器的 `localStorage` *(浏览器的本地存储机制，可以持久化保存字符串数据，刷新页面后数据不丢失)* 来持久化存储自定义代币数据，所以用户刷新页面后自定义代币列表不会丢失。

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **localStorage 操作的复杂性**：直接操作 `localStorage` *(浏览器本地存储API，用于持久化存储键值对)* 需要处理 JSON 序列化/反序列化和空值处理，封装后提供简洁的 API

2. **代币地址大小写问题**：Stellar 网络上合约地址是大小写不敏感的，hook 内部将地址统一转为大写存储和比较，避免大小写不一致导致的查找失败

3. **immer 式的不可变更新**：通过函数式方式更新状态，简化了添加/删除操作

4. **类型安全**：提供了完整的 TypeScript 类型定义

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
- hook 不接受任何参数，内部直接读取 localStorage

### 输出（返回值）
```typescript
{
  data: Record<string, Token>      // 自定义代币映射表，key 为合约地址(大写)
  mutate: (type: 'add' | 'remove', currency: Token[]) => void
  hasToken: (currency: Token | string) => boolean
}
```

### 核心执行逻辑

1. **初始化**：从 localStorage 读取 `sushi.stellar.custom-tokens` 键的值
2. **添加代币**：`mutate('add', [token])` 将代币的合约地址转为大写后存入
3. **删除代币**：`mutate('remove', [token])` 过滤掉匹配地址的代币
4. **检查存在**：`hasToken(currency)` 返回代币是否在列表中

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个管理自定义代币列表的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useCustomTokens的hook，用来管理某个区块链网络上的自定义代币列表。然后明确几个关键点。第一，用户添加的代币要存在localStorage里，这样刷新页面也不会丢。第二，代币地址要统一转成大写存储，因为区块链的地址比较起来很容易大小写不一致。第三，要有添加代币和删除代币的方法。第四，要能检查某个代币是否已经在列表里。第五，用专门的useLocalStorage hook来管理状态，而不是普通的useState，这样才能保证数据持久化。第六，返回的结构要包含数据本身、操作方法和检查方法。

### 这里面有几个地方特别容易出错

地址一定要转大写存储，这是Stellar合约地址的格式要求，不转的话查找的时候会出问题。初始化的时候要处理localStorage里没数据的情况，没数据就返回空对象，别让程序崩了。更新数据的时候要用展开运算符创建新对象，不要直接修改原来的对象，因为React依赖不可变数据来检测变化。

### 状态管理要注意什么

用useLocalStorage而不是useState，这样才能在刷新页面后保留用户的数据。操作方法要用useCallback包一下，避免每次渲染都创建新的函数引用。结果数据要用useMemo缓存一下，防止不必要的重复计算。状态管理看似繁琐，但做好了能让应用流畅很多。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useCustomTokens } from '@sushiswap/hooks'

function MyComponent() {
  const { data: customTokens, mutate, hasToken } = useCustomTokens()

  // 添加自定义代币
  const addToken = (token: Token) => {
    mutate('add', [token])
  }

  // 删除自定义代币
  const removeToken = (token: Token) => {
    mutate('remove', [token])
  }

  // 检查代币是否存在
  const exists = hasToken(tokenAddress)

  return (
    <div>
      <p>自定义代币数量: {Object.keys(customTokens).length}</p>
      <button onClick={() => addToken(myToken)}>添加代币</button>
    </div>
  )
}
```

### 常见使用场景

1. **代币搜索优先显示**：在代币列表中，优先显示用户自定义添加的代币
   ```tsx
   const allTokens = { ...commonTokens, ...customTokens }
   ```

2. **信任代币管理**：用户可以将信任的代币添加到自定义列表，方便快速访问

3. **代币信息缓存**：对于链上查询不到的代币，用户可以手动添加代币信息

### Dos and Don'ts

**Dos:**
- ✅ 使用 `hasToken` 检查代币是否存在后再添加，避免重复
- ✅ 添加代币时确保代币信息完整（address、symbol、name、decimals）
- ✅ 地址比较时使用 `toUpperCase()` 规范化

**Don'ts:**
- ❌ 不要直接修改 `data` 对象，应该使用 `mutate` 方法
- ❌ 不要存储敏感信息到 customTokens（本地存储不安全）
- ❌ 添加前不要忘记检查代币地址是否有效
