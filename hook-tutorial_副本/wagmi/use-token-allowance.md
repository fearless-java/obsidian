> 源代码路径: `apps/web/src/lib/wagmi/hooks/approvals/hooks/useTokenAllowance.ts`

# useTokenAllowance

## 大白话讲讲这个hook的作用

`useTokenAllowance` 是一个用来查询 ERC20 Token 授权额度的 Hook。简单来说，它会告诉你在某个钱包地址授权给某个 Spender（消费方）可以使用的代币数量。

举个例子：当你授权 Uniswap 可以使用你的 USDC 进行交易时，这个 Hook 就能查到 "你的地址授权给 Uniswap 地址的 USDC 额度是多少"。

它不仅会读取当前区块的授权数据，还会监听区块变化自动刷新数据，确保你看到的授权额度是最新的。

## 讲讲为什么需要封装该hook

虽然 wagmi 提供了 `useReadContract` 可以直接读取 ERC20 的 `allowance` 方法，但直接使用会有几个问题：

1. **类型安全**：原生的 `useReadContract` 返回的是原始的 bigint，而封装后会转换成 `Amount` 类型，方便进行大小比较和格式化。

2. **自动刷新**：需要监听区块变化来自动刷新授权数据，直接用的话需要自己写 `useBlockNumber` + `invalidateQueries` 的逻辑。

3. **状态管理**：授权状态需要在多个组件间共享，封装可以统一管理这些状态。

4. **数据转换**：原始 bigint 数据需要根据 token 的 decimals 转换成人类可读的形式，封装后直接得到 `Amount` 实例。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseTokenAllowance {
  token?: EvmToken           // 要查询的代币
  chainId: EvmChainId | undefined  // 链ID
  owner: Address | undefined       // 授权者地址（钱包地址）
  spender: Address | undefined     // 被授权方地址（如合约地址）
  enabled?: boolean                // 是否启用查询
}
```

### 执行逻辑

1. **基础查询**：使用 `useReadContract` 调用 ERC20 的 `allowance(owner, spender)` 方法

2. **数据转换**：通过 `select` 选项将原始 bigint 转换为 `Amount<EvmToken>` 类型

3. **区块监听**：使用 `useBlockNumber({ watch: true })` 监听新区块

4. **自动刷新**：当区块号变化时，自动 `invalidateQueries` 刷新授权数据

### 输出 (Outputs)

```typescript
// 返回 wagmi 的 UseQueryResult，data 被转换为 Amount 类型
{
  data: Amount<EvmToken>,  // 授权额度
  isLoading: boolean,
  isError: boolean,
  refetch: () => void,
  // ... 其他 wagmi query 属性
}
```

### 数据流向图

```
输入参数 (owner, spender, token, chainId)
         ↓
    useReadContract
         ↓
    ERC20.allowance() 调用
         ↓
    select 转换: bigint → Amount<EvmToken>
         ↓
    返回授权额度数据
         ↑
    useBlockNumber 监听区块变化
         ↓
    invalidateQueries 触发刷新
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：参数校验要做好**。在开启查询之前，一定要检查所有必要参数是否都传入了。比如token地址、owner地址、spender地址，这些都不能少。如果随便传个空值进去，查询肯定会失败。可以用一个布尔表达式一次性检查所有参数：enabled应该同时判断token、owner、spender和chainId都存在。

**第二点：善用Amount类型**。查询返回的授权额度是Amount对象，这个对象有很多好用的方法。比如你想判断授权额度够不够用，可以直接用allowance.gt(amount)来判断授权额度是否大于要花费的金额。这样比直接比较两个数字要安全得多，而且Amount对象还会自动处理精度问题。

**第三点：原生币要特殊处理**。像ETH这样的原生币本身不需要授权，所以在代码里要判断一下币的类型。如果是原生币，直接标记为已授权状态就行，不用真的去查询授权额度。

**第四点：注意查询性能**。这个hook会监听区块变化来自动刷新数据，意思是每次新区块产生的时候，它都会重新查询一次授权额度。这对于需要实时显示授权状态的场景很有用，但也会增加RPC调用次数。如果你同时查询很多个代币的授权，就要考虑会不会被rate limit限制了。

### 状态管理要清楚

这个hook返回的状态主要分三种。第一种是查询状态，通过isLoading、isError、isFetching这几个字段来跟踪，分别表示首次加载中、出错了、正在后台刷新。第二种是授权额度本身，通过data字段返回，已经是Amount类型了，可以直接拿来用。第三种是刷新控制，除了依赖自动的区块刷新，还可以通过refetch方法手动触发一次查询。

### 常见错误要避开

**第一个坑：连环刷新问题**。如果你在useEffect里调用invalidateQueries来手动刷新，要记得加上cancelRefetch: false这个选项。否则可能会出现重复请求的情况，白白浪费RPC调用。

**第二个坑：QueryKey要设计好**。查询的key必须包含所有会影响结果的变量，比如chainId、owner地址、spender地址。如果key设计得不对，可能会导致缓存失效或者返回错误的数据。

**第三个坑：大额数字的处理**。有些代币的总量非常大，用bigint存储的时候要注意精度问题。不过用Amount类型来处理的话，一般不会有这个问题，因为Amount已经帮你处理好了。

**第四个坑：链ID和token要匹配**。传入的token和chainId必须是对应的。比如一个在Polygon上的USDC，你不能用以太坊的chainId去查询，那样查出来的结果肯定不对。

### 提示词模板

```markdown
帮我创建一个查询ERC20代币授权额度的hook。

需求：
- 查询某个地址授权给某个合约的代币额度
- 返回值用Amount类型，方便比较大小
- 监听区块变化自动刷新数据

技术栈：
- React + TypeScript
- wagmi v2 + viem
- @tanstack/react-query v5
- sushi库的Amount类型

输入参数：
- token：要查询的代币
- chainId：链ID
- owner：授权者的钱包地址
- spender：被授权的合约地址
- enabled：是否启用查询

输出：
- data：授权额度（Amount类型）
- isLoading：是否在加载
- isError：是否有错误
- refetch：手动刷新函数

实现要点：
1. 用erc20的allowance方法查询
2. 把返回的bigint转成Amount类型
3. 用useBlockNumber监听新区块触发刷新
4. enabled要检查所有必要参数都不为空

注意：
- 原生币不用查授权
- 地址要验证有效性
- 查询key要包含chainId和地址
```

### 实际避坑指南

第一，连环invalidate的问题。在useEffect里调用invalidateQueries时，如果连续触发了多次，可能会重复请求。解决方案是加上{ cancelRefetch: false }，让请求不会互相取消。

第二，queryKey的唯一性。queryKey就像查询的身份证号，必须唯一且包含所有变量。比如你的查询依赖chainId和地址，那queryKey里这两个都要有。如果漏掉了，React Query的缓存就会出问题。

第三，大额代币的精度。大额数字如果精度处理不对，可能会出现数字失真。但用Amount类型就基本不会有这个问题，因为它内部已经处理好了精度转换。

第四，链ID和token不匹配。这种情况最容易出bug。比如你在Polygon上查一个只在以太坊上的token，肯定查不到。所以每次查询前最好验证一下token是否在对应的链上。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useTokenAllowance` 用于查询 ERC20 代币的授权额度。基本用法如下：

```typescript
import { useTokenAllowance } from '@sushiswap/wag'

function ApprovalCheck() {
  const { data: allowance } = useTokenAllowance({
    token: USDC,
    chainId: EvmChainId.ETHEREUM,
    owner: '0x...',       // 你的钱包地址
    spender: '0x...',     // 要授权给谁（如 Uniswap V2 Router）
  })

  if (allowance && allowance.gt(0)) {
    return <div>已授权 {allowance.toSignificant(6)} USDC</div>
  }
  return <div>未授权</div>
}
```

### 5.2 常见使用场景

#### 场景一：检查是否需要授权

在执行 DEX 交易前，检查用户是否已授权足够额度：

```typescript
const { data: allowance } = useTokenAllowance({
  token: inputCurrency,
  chainId,
  owner: account,
  spender: SUSHISWAP_V2_ROUTER_ADDRESS[chainId],
})

// 判断是否需要授权
const needsApproval = !allowance || allowance.lt(amount)
```

#### 场景二：显示剩余可用额度

```typescript
const { data: allowance } = useTokenAllowance({ ... })

// 显示已授权但未使用的额度
const remaining = allowance.sub(amount)
```

#### 场景三：监听授权变化

```typescript
// 配合 useTokenApproval 使用
const [approvalState, approve] = useTokenApproval({
  spender: routerAddress,
  amount: amountToSpend,
})

// allowance 自动监听区块刷新
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **使用 Amount 类型进行比较**
   ```typescript
   // 正确：使用 Amount 的方法
   if (allowance.gt(amount)) { ... }
   ```

2. **先检查参数有效性**
   ```typescript
   enabled: Boolean(token && owner && spender && chainId)
   ```

3. **结合 useTokenApproval 使用**
   ```typescript
   const { data: allowance } = useTokenAllowance({ ... })
   const [state, { write }] = useTokenApproval({ ... })
   ```

#### ❌ Don'ts

1. **不要直接比较 bigint**
   ```typescript
   // 错误
   if (allowance > amount.amount) { ... }
   ```

2. **不要忽略 enabled 状态**
   ```typescript
   // 错误：可能导致无效查询
   enabled: true

   // 正确
   enabled: Boolean(token && owner && spender)
   ```

3. **不要在循环中使用**
   ```typescript
   // 错误：每个 allowance 查询都是独立的 RPC 调用
   tokens.map(token => useTokenAllowance({ token, ... }))
   ```

4. **不要忽略原生币**
   ```typescript
   // 错误：原生币（如 ETH）不需要授权
   if (currency.type === 'native') return ApprovalState.APPROVED
   ```
