> 源代码路径: `apps/web/src/lib/wagmi/hooks/approvals/hooks/useTokenPermit.ts`

# useTokenPermit

## 大白话讲讲这个hook的作用

`useTokenPermit` 是一个支持 ERC20 Permit（许可）机制的 Hook。Permit 是一种无需预授权（no-pre-approval）就能转移代币的方式。

大白话解释：传统授权是你先说"允许别人花我的钱"，然后才能转账。Permit 不一样，你可以直接签名一份"许可证"，连授权那一步都省了，对用户更方便（省一笔 gas）。

这个 Hook 会：
1. 获取用户的 nonce（签名唯一性保证）
2. 获取交易截止时间 deadline
3. 构造符合 EIP-712 标准的签名数据
4. 调用 `signTypedDataAsync` 让用户签名
5. 存储签名结果供后续使用

## 讲讲为什么需要封装该hook

1. **EIP-712 签名复杂**：Permit 签名涉及复杂的 EIP-712 类型数据和 domain separator，需要正确处理。

2. **多种 Permit 类型**：有的用 `Permit` (EIP-2612)，有的用 `PermitAllowed`，需要适配不同标准。

3. **与 Approval 状态统一**：需要和 `useTokenApproval` 返回相同的状态格式，方便 UI 统一处理。

4. **签名存储**：签名结果需要存储（通过 `useSignature`），供其他组件使用。

5. **有效期验证**：签名有 deadline/expiry，需要验证签名是否过期。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseTokenPermitParams {
  spender: Address | undefined              // 授权给谁
  amount: Amount<EvmCurrency> | undefined   // 授权金额
  enabled?: boolean
  permitInfo: PermitInfo                    // Permit 配置信息
  ttlStorageKey: TTLStorageKey              // TTL 存储 key
  tag: string                               // 签名标签
}

interface PermitInfo {
  type: PermitType          // AMOUNT 或 ALLOWED
  name: string              // 代币名称
  version?: string           // 可选，代币版本
}
```

### Permit 类型

```typescript
enum PermitType {
  AMOUNT = 1,    // EIP-2612 标准
  ALLOWED = 2,  // 旧版 PermitAllowed
}
```

### 输出 (Outputs)

```typescript
// 返回格式与 useTokenApproval 一致
[ApprovalState, { write: () => void }]
```

### 执行逻辑详解

#### 1. 获取 Nonce

```typescript
const { data: nonce } = useReadContract({
  address: amount?.currency.wrap().address,
  abi: eip2612Abi_nonces,
  functionName: 'nonces',
  args: [address]
})
```

#### 2. 获取 Transaction Deadline

```typescript
const { data: transactionDeadline } = useTransactionDeadline({
  chainId,
  storageKey: ttlStorageKey,
  enabled
})
```

#### 3. 构造 EIP-712 签名数据

```typescript
// 根据 permit 类型选择不同的消息格式
const types = {
  EIP712Domain: permitInfo.version
    ? EIP712_DOMAIN_TYPE
    : EIP712_DOMAIN_TYPE_NO_VERSION,
  Permit: allowed ? PERMIT_ALLOWED_TYPE : EIP2612_TYPE,
}

const domain = {
  name: permitInfo.name,
  version: permitInfo.version,  // 可选
  chainId,
  verifyingContract: tokenAddress,
}

const message = allowed ? {
  holder: address,
  spender,
  allowed: true,
  nonce,
  expiry: transactionDeadline,
} : {
  owner: address,
  spender,
  value: amount.amount,
  nonce,
  deadline: transactionDeadline,
}
```

#### 4. 签名并存储

```typescript
const signedData = await signTypedDataAsync({
  account: address,
  primaryType: 'Permit',
  types,
  domain,
  message,
})

setSignature({ ...hexToSignature(signedData), message, domain })
```

#### 5. 签名有效性验证

```typescript
const isSignatureDataValid =
  transactionDeadline &&
  amount &&
  signature?.domain?.verifyingContract === tokenAddress &&
  signature?.message?.owner === address &&
  signature?.message?.nonce === nonce &&
  signature?.message?.spender === spender &&
  // 金额匹配或 allowed 模式
  (signature.message.value === amount.amount || 'allowed' in signature.message) &&
  // 未过期
  (signature.message.deadline >= now || signature.message.expiry >= now)
```

### 数据流向图

```
输入参数
    ↓
┌─────────────────────────────────────┐
│  useTransactionDeadline 获取截止时间   │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  useReadContract 获取 nonce          │
└─────────────────────────────────────┘
    ↓
构造 EIP-712 签名数据
    ↓
signTypedDataAsync 用户签名
    ↓
┌─────────────────────────────────────┐
│  setSignature 存储签名 (全局状态)      │
└─────────────────────────────────────┘
    ↓
验证签名有效性
    ↓
返回 [ApprovalState, { write }]
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：Permit类型选择有讲究**。ERC20 Permit有两种实现方式，现代代币大多用EIP-2612标准，也就是AMOUNT类型，这种方式更省gas。旧版的PermitAllowed虽然还能用，但已经不推荐了。所以如果你要集成Permit功能，优先选AMOUNT类型。

**第二点：版本字段要兼容**。有些代币（比如USDT）它们的Permit实现不包含version字段，这在构造签名数据的时候会有影响。你的代码要能正确处理这种情况，有version和没有version的代币要分别处理。

**第三点：签名存储要有唯一标识**。签名数据是通过全局状态来存储的，用tag作为唯一标识。tag的设计要保证全局唯一，建议用"spender地址-代币地址-链ID"这样的格式来组合，这样就不会跟其他授权场景冲突了。

**第四点：过期检查不能忘**。签名是有有效期的，过期了就失效了。在使用签名之前一定要检查当前时间是否超过了deadline或者expiry。如果过期了，用户需要重新签名。

**第五点：和Approval状态要统一**。这个hook的返回格式跟useTokenApproval是一样的，都是返回一个状态和一个write函数。这样的设计是为了方便UI层统一处理，授权方式不管是传统的approve还是Permit，组件代码改动最小。

### 约束条件要记住

**第一，只能用于支持Permit的代币**。不是所有ERC20代币都支持EIP-2612标准，所以在调用这个hook之前要先检查代币是否支持Permit。如果不支持，还是要回退到传统的授权方式。

**第二，钱包必须支持签名**。用户的钱包必须能签署EIP-712格式的签名数据。像MetaMask、WalletConnect这些主流钱包都支持，但如果用户用的是比较老的钱包，可能就不行了。

**第三，签名只能用一次**。每次成功执行Permit后，代币合约里的nonce都会增加。即使是同一份签名数据，第二次使用时合约会认为nonce不对而拒绝。所以每次授权都需要用户重新签名。

**第四，deadline是最后期限**。签名在deadline之前是有效的，超过deadline就失效了。这个deadline是区块时间戳，不是普通的Unix时间。

### 状态管理要清楚

状态流转是这样的：初始状态是UNKNOWN，然后开始加载nonce变成LOADING状态，如果还没签名或者签名已经过期就是NOT_APPROVED，正在签名中是PENDING，签名有效且未过期才是APPROVED。

签名数据存储在全局状态里，包含v、r、s三个签名分量，还有原始的message和domain。这些数据在合适的时机要用clearSignature清除掉，否则可能会用到过期的签名。

### 常见错误要避开

**第一个坑：链ID不匹配**。签名数据里包含了链ID信息，如果用户在某条链上签的名，却想在另一条链上用，签名会失败。所以一定要确保签名和使用的链是一致的。

**第二个坑：nonce会变化**。每次Permit成功后，链上的nonce就变了。如果你的代码里缓存了旧的签名再用，就会失败。因为合约会验证nonce，如果不对就会拒绝。

**第三个坑：Domain Separator要对应**。不同的代币合约有自己的domain separator，验证签名的时候必须用原始的token地址来校验，不能用包装后的地址。

**第四个坑：deadline和expiry的区别**。EIP-2612标准用deadline表示有效期，而旧版的PermitAllowed用expiry。这两个字段的意思差不多，但名字不一样，你的代码要分清楚用哪个。

**第五个坑：Permit其实不是真正无gas**。Permit的设计理念是用户签名后，由sponsor来支付gas执行交易。但实际落地的时候，还是需要有人付gas的，通常sponsor会补贴这笔费用。

### 提示词模板

```markdown
帮我创建一个支持ERC20 Permit签名的授权hook。

需求：
- 用户签名授权而不是发起交易
- 支持EIP-2612标准
- 签名有过期时间要注意
- 签名数据全局存储供后续使用

技术栈：
- React + TypeScript
- wagmi v2 + viem
- @tanstack/react-query v5
- sushi库
- useTTL管理用户偏好

 Permit类型：
- AMOUNT类型：现代标准，更省gas
- ALLOWED类型：旧版，已不推荐

输入：
- spender：授权给哪个合约
- amount：授权金额
- permitInfo：代币的Permit配置信息
- tag：签名存储的唯一标识
- enabled：是否启用

输出：
- ApprovalState：当前授权状态
- write：发起签名的函数

签名验证要点：
1. 检查签名是否过期
2. 检查签名者是否正确
3. 检查授权对象和金额是否匹配
4. 检查nonce是否正确

注意：
- 代币必须支持EIP-2612
- 钱包必须能签TypedData
- 签名只能用一次
- 过期后要重新签
```

### 实际避坑指南

第一个，链ID要一致。签名时用的链ID和执行时用的链ID必须完全一致。如果用户在Polygon上签的名，却想在以太坊上用，是不行的。

第二个，nonce会递增。每次Permit执行成功后，nonce就变了。如果捕获了签名数据但没有立即使用，下次用的时候nonce已经变了，签名就失效了。所以签名后要尽快使用。

第三个，版本字段问题。有些代币的Permit实现不包含version字段，构造签名数据的时候要区分处理，不能假设每个代币都有version。

第四个，deadline和expiry的取舍。EIP-2612用deadline，旧版用expiry。如果你的代码要同时支持两种Permit类型，一定要分清楚用哪个字段来验证有效期。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useTokenPermit` 用于 ERC20 Permit 签名授权。基本用法如下：

```typescript
import { useTokenPermit, ApprovalState } from '@sushiswap/wag'

function PermitButton({ amount, currency, spender }) {
  const permitInfo = {
    type: PermitType.AMOUNT,
    name: 'USDC',
    version: '2',
  }

  const [permitState, { write }] = useTokenPermit({
    spender,
    amount,
    permitInfo,
    ttlStorageKey: 'permit-ttl',
    tag: `permit-${spender}-${currency.address}`,
    enabled: Boolean(amount && spender),
  })

  if (permitState === ApprovalState.NOT_APPROVED) {
    return <button onClick={write}>签名授权</button>
  }

  return <div>已签名授权</div>
}
```

### 5.2 常见使用场景

#### 场景一：EIP-2612 Permit (推荐)

现代代币如 UNI、COMP 使用 EIP-2612：

```typescript
const permitInfo = {
  type: PermitType.AMOUNT,  // EIP-2612
  name: 'UNI',
  version: '2',
}
```

#### 场景二：PermitAllowed (旧版)

某些旧代币使用 PermitAllowed：

```typescript
const permitInfo = {
  type: PermitType.ALLOWED,  // 旧版
  name: 'SomeToken',
  // 注意：某些代币没有 version 字段
}
```

#### 场景三：与 useTokenApproval 切换

UI 可以根据代币是否支持 Permit 来切换：

```typescript
const isSupported = checkPermitSupport(token)

// 根据代币特性选择授权方式
if (isSupported) {
  return <PermitApprove amount={amount} />
} else {
  return <TraditionalApprove amount={amount} />
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **验证签名有效性后再使用**
   ```typescript
   // 在使用签名前验证
   if (!isSignatureDataValid) {
     return <button onClick={write}>重新签名</button>
   }
   ```

2. **使用唯一的 tag**
   ```typescript
   // 确保每个授权场景有唯一的 tag
   tag: `swap-${spender}-${tokenAddress}-${chainId}`
   ```

3. **处理不支持 Permit 的代币**
   ```typescript
   // 检查代币是否支持 permit
   if (!token.supportedPermit) {
     return useTokenApproval({ ... })
   }
   ```

#### ❌ Don'ts

1. **不要重复使用签名**
   ```typescript
   // 错误：签名只能用一次
   // 正确：每次需要时让用户重新签名
   ```

2. **不要忽略过期检查**
   ```typescript
   // 错误
   if (!signature) { write() }

   // 正确：检查签名是否过期
   if (!signature || isExpired(signature)) { write() }
   ```

3. **不要在不支持的链上使用**
   ```typescript
   // Permit 需要正确的 chainId 与签名匹配
   // 确保 chainId 一致
   ```

4. **不要忘记处理用户拒绝**
   ```typescript
   // 用户取消签名不应该显示错误
   // permitState 不会变成 ERROR
   ```
