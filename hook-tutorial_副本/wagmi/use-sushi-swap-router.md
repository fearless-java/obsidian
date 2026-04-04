> 源代码路径: `apps/web/src/lib/wagmi/hooks/contracts/useSushiSwapRouter.ts`

# useSushiSwapRouter

## 大白话讲讲这个hook的作用

`useSushiSwapRouter` 是一个获取 SushiSwap V2 路由合约实例的 Hook。它返回的 contract 对象可以直接调用 SushiSwap 的 swap、addLiquidity、removeLiquidity 等功能。

大白话：就是"拿到 SushiSwap 路由合约的操作柄"，有了它你就可以跟 SushiSwap 合约交互了。

它会根据链 ID 返回对应链上的 Router 地址，并同时获取 Public Client（只读）和 Wallet Client（签名操作）。

## 讲讲为什么需要封装该hook

1. **多链支持**：不同链上有不同的 Router 地址，封装后统一接口。

2. **Client 选择**：根据操作类型选择 Public Client（只读）或 Wallet Client（签名）。

3. **类型安全**：返回完整的 contract 实例，有完整的类型提示。

4. **懒加载**：只在有需要时才创建合约实例。

5. **ABIF 封装**：封装了 SushiSwap V2 的 ABI，方便调用。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
chainId: SushiSwapV2ChainId | undefined
```

### 输出 (Outputs)

```typescript
{
  // SushiSwap Router Contract 实例
  // 包含所有 ABI 方法的调用接口
} | null
```

### 执行逻辑详解

#### 1. 获取 Clients

```typescript
const publicClient = usePublicClient({ chainId }) as PublicClient
const { data: walletClient } = useWalletClient({ chainId })
```

#### 2. 创建合约实例

```typescript
return useMemo(() => {
  if (!chainId || (!publicClient && !walletClient)) return null

  return getContract({
    address: SUSHISWAP_V2_ROUTER_ADDRESS[chainId],
    abi: sushiSwapV2RouterAbi,
    client: walletClient || publicClient!,
  })
}, [chainId, publicClient, walletClient])
```

#### 3. Client 选择逻辑

```typescript
// walletClient 存在时使用（可签名）
// 否则使用 publicClient（只读）
client: walletClient || publicClient!
```

### 数据流向图

```
输入: chainId
         ↓
    ┌────────────────────────────────────┐
    │  usePublicClient 获取公开客户端      │
    │  useWalletClient 获取钱包客户端      │
    └────────────────────────────────────┘
         ↓
    getContract 创建合约实例
    - address: SUSHISWAP_V2_ROUTER_ADDRESS[chainId]
    - abi: sushiSwapV2RouterAbi
    - client: walletClient || publicClient
         ↓
    返回 Contract 实例或 null
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：链ID必须有效**。传入的chainId必须是SushiSwap V2支持的链才行。SushiSwap V2不是所有链都部署了，如果传入一个不支持的链，hook会返回null。这个要提前检查清楚。

**第二点：选择正确的Client**。这个hook会根据情况选择用哪个client。如果是只读操作，比如查询价格或者准备金，用publicClient就行。如果是需要签名的操作，比如 swap 或者添加流动性，要用 walletClient。代码里用walletClient优先，如果没连接才fallback到publicClient。

**第三点：null的情况要处理**。当chainId无效或者client都不可用的时候，hook会返回null。调用方必须处理这种情况，最简单的就是在router为null的时候显示一个提示或者禁用相关按钮。

**第四点：类型提示很完整**。返回的contract实例带有完整的类型信息，你可以直接调用contract.read.方法名或者contract.write.方法名，IDE会给你自动补全和类型检查。这比你自己拼凑参数要安全得多。

**第五点：实例被缓存了**。合约实例是用useMemo缓存的，不会每次render都重新创建。这样既避免了重复创建的开销，也保证了实例的稳定性。

### 约束条件要记住

**第一，只支持V2的链**。不是所有链都有SushiSwap V2的Router合约。代码里有个列表标记了支持哪些链，只有在这些链上才能正常使用。

**第二，需要有client可用**。必须至少有一个client可用才能创建合约实例。walletClient可能还没连接，publicClient理论上总是可用的。

**第三，null时不能调用**。如果hook返回null，说明初始化失败了。这时候不能尝试调用任何合约方法，否则会报错。

**第四，不能跨链操作**。chainId必须匹配合约部署的链。如果你传入的chainId和合约地址对应的链不一致，交易肯定会失败。

### 状态管理要清楚

状态来自三个地方。publicClient来自wagmi的provider，这个是始终可用的。walletClient来自用户连接的钱包，可能还没连接或者断开了。合约实例是用useMemo缓存的，只有依赖变化时才会重新创建。

返回值可能是合约实例，也可能是null。调用方需要自己判断，如果是null就应该提示用户或者禁用功能。

### 常见错误要避开

**第一个坑：首次挂载时client未连接**。组件刚挂载的时候，walletClient可能还没有连接上。代码用||操作符来处理这种情况，优先用walletClient，没有就用publicClient。这样可以保证总是能创建合约实例。

**第二个坑：链ID类型要正确**。传入的chainId必须是SushiSwapV2ChainId类型，不是随便一个数字都行。如果你传错了类型，代码会找不到对应的合约地址。

**第三个坑：ABI版本要匹配**。不同版本的Router合约，ABI可能略有不同。如果ABI和合约地址不匹配，调用会失败。要确保用的是对应版本的ABI。

**第四个坑：签名操作要估算gas**。如果用walletClient发送交易，最好先估算一下gas，避免交易因为gas不足而失败。不过这个一般 wagmi 会帮你处理。

**第五个坑：多链部署地址不同**。SushiSwap V2在不同链上部署的Router合约地址是不同的。代码里用Record来存储这个映射，根据chainId查找对应的地址。

### 提示词模板

```markdown
帮我创建一个获取SushiSwap V2 Router合约实例的hook。

需求：
- 根据链ID返回对应链的Router地址
- 同时获取只读client和钱包client
- 创建好的合约实例可以调用各种方法
- 支持查询和签名两种操作

技术栈：
- React + TypeScript
- wagmi v2 + viem
- sushi库

常量：
- Router地址：不同链有不同的地址

输入：
- chainId：链ID，必须是V2支持的链

输出：
- Contract实例，有完整的类型提示
- 如果无效就返回null

实现要点：
1. 获取publicClient和walletClient
2. 根据chainId找到Router地址
3. 创建合约实例并缓存
4. 优先用walletClient，没有就用publicClient

ABI包含：
- swapExactTokensForTokens
- addLiquidity
- removeLiquidity
- 等方法

注意：
- 链ID必须有效
- walletClient可能还没连接
- 返回null时要处理
- 不要跨链使用
```

### 实际避坑指南

第一个，检查chainId有效性。在调用这个hook之前，最好检查一下传入的chainId是否是SushiSwap V2支持的链。如果不是，应该提示用户切换到支持的链。

第二个，null要处理。当hook返回null的时候，调用方应该显示一个提示，比如"当前链不支持此操作"。不要尝试在null的合约上调用方法。

第三个，Client选择策略。代码优先用walletClient，这是对的。但如果你确定是只读操作，可以直接用publicClient，这样不需要用户连接钱包也能工作。

第四个，类型提示很方便。返回的合约实例有完整的类型，可以直接用IDE的自动补全。这比查文档或者自己拼凑参数要方便得多，也更安全。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useSushiSwapRouter` 用于获取 SushiSwap Router 合约实例。基本用法如下：

```typescript
import { useSushiSwapRouter } from '@sushiswap/wag'

function SwapComponent({ chainId }) {
  const router = useSushiSwapRouter({ chainId })

  if (!router) {
    return <div> Router 不可用</div>
  }

  // router 包含完整的 ABI 方法类型
  // router.read.* 用于只读调用
  // router.write.* 用于交易
  return <div>Router 地址: {router.address}</div>
}
```

### 5.2 常见使用场景

#### 场景一：只读查询（如获取预期输出）

```typescript
function GetSwapOutput({ tokenIn, tokenOut, amountIn }) {
  const router = useSushiSwapRouter({ chainId })

  const { data: outputAmount } = useReadContract({
    // @ts-ignore
    address: router?.address,
    // @ts-ignore
    abi: router?.abi,
    functionName: 'getAmountsOut',
    args: [amountIn.amount, [tokenIn.address, tokenOut.address]],
  })

  return <div>预计获得: {outputAmount?.[1]?.toString()}</div>
}
```

#### 场景二：签名交易（需要钱包）

```typescript
async function executeSwap({ tokenIn, tokenOut, amountIn, amountOutMin }) {
  const router = useSushiSwapRouter({ chainId })
  const { writeContract } = useWriteContract()

  // @ts-ignore
  writeContract({
    address: router.address,
    abi: router.abi,
    functionName: 'swapExactTokensForTokens',
    args: [
      amountIn.amount,
      amountOutMin.amount,
      [tokenIn.address, tokenOut.address],
      account,
      deadline,
    ],
  })
}
```

#### 场景三：添加流动性

```typescript
async function addLiquidity({ tokenA, tokenB, amountA, amountB }) {
  const router = useSushiSwapRouter({ chainId })
  const { writeContract } = useWriteContract()

  // @ts-ignore
  writeContract({
    address: router.address,
    abi: router.abi,
    functionName: 'addLiquidity',
    args: [
      tokenA.address,
      tokenB.address,
      amountA.amount,
      amountB.amount,
      0n, // 最小 amount A
      0n, // 最小 amount B
      account,
      deadline,
    ],
  })
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **检查 router 是否为 null**
   ```typescript
   // router 可能返回 null（无效 chainId 或未连接钱包）
   if (!router) {
     return <div>请连接钱包</div>
   }
   ```

2. **使用 useWriteContract 发送交易**
   ```typescript
   // 不要直接调用 router.write
   const { writeContract } = useWriteContract()
   // 使用 writeContract 配合 simulation
   ```

3. **理解 Client 区别**
   ```typescript
   // 只读操作用 publicClient
   // 需要签名的操作用 walletClient
   // useSushiSwapRouter 会自动选择
   ```

#### ❌ Don'ts

1. **不要在每次 render 时创建合约**
   ```typescript
   // 错误
   const contract = getContract({ ... })

   // 正确：使用 useSushiSwapRouter，它有 useMemo 缓存
   const router = useSushiSwapRouter({ chainId })
   ```

2. **不要忽略链 ID 检查**
   ```typescript
   // 错误：传入任意 chainId
   useSushiSwapRouter({ chainId: 99999 })

   // 正确：使用有效的 SushiSwapV2ChainId
   useSushiSwapRouter({ chainId: EvmChainId.ETHEREUM })
   ```

3. **不要直接调用 contract 函数而不做模拟**
   ```typescript
   // 错误：直接发送可能失败
   router.write.swapExactTokensForTokens(...)

   // 正确：先用 useSimulateContract 模拟
   const { data: simulation } = useSimulateContract({ ... })
   ```

4. **不要忘记 deadline**
   ```typescript
   // SushiSwap 交易需要 deadline 防止交易被恶意延后
   const deadline = BigInt(Math.floor(Date.now() / 1000) + 60 * 20) // 20 minutes
   ```
