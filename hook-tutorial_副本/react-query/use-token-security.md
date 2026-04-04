> 源代码路径: `apps/web/src/lib/hooks/react-query/tokens/useTokenSecurity.ts`

# useTokenSecurity Hook Tutorial

## 1. 大白话讲讲这个hook的作用

`useTokenSecurity` 是用来查询某个代币安全信息的 Hook。它从两个数据源（GoPlus 和 DeFi Scanner）获取代币的安全数据，包括：是否开源合约、是否有买卖税费、是否是 honeypot、是否可以买卖等安全相关信息。

简单来说：**就是帮你判断一个币"安不安全"——查这个币的合约有没有问题、能不能买卖、是不是貔貅（只能买不能卖）、有没有隐藏权限等。**

---

## 2. 为什么需要封装该hook

### 原始问题
1. **多数据源聚合**：需要同时查询 GoPlus API 和 DeFi Scanner API，然后合并结果
2. **EVM 和 Solana 差异**：两种链的代币安全数据结构和 API 不同，需要分别处理
3. **复杂的 Zod 转换**：GoPlus 返回的数据需要多种 transform（字符串 '0'/'1' 转 boolean，空白字符串处理等）
4. **并发请求和降级**：两个数据源可能一个成功一个失败，需要 Promise.allSettled *(一个Promise方法，用于并行执行多个Promise，即使部分失败也不会导致整体失败)* 处理
5. **安全问题判断**：有预设的 `isTokenSecurityIssue` 规则来判断哪些值是"有问题"的

### 封装带来的好处
1. **统一返回格式**：无论数据来自哪个源，返回的结构是一致的
2. **并发查询**：两个数据源并行查询，提高响应速度
3. **部分降级**：一个数据源失败时，另一个仍能提供数据
4. **预定义问题标签**：`isTokenSecurityIssue` 提供了哪些字段代表有问题的判断函数
5. **人类可读消息**：`TokenSecurityMessage` 提供了每个字段的中文解释

---

## 3. 执行逻辑和数据流向

### 输入 (Params)
```typescript
{
  currency: EvmToken | SvmToken | undefined  // 代币对象
  enabled?: boolean = true                    // 是否启用
}
```

### 输出 (Return)
```typescript
{
  data: {
    [K in keyof TokenSecurity]: {
      goPlus?: boolean   // GoPlus 数据源的值
      deFi?: boolean    // DeFi Scanner 数据源的值
    }
  }
  isHoneypot: boolean    // 汇总：是否是貔貅
  isFoT: boolean         // 汇总：是否有手续费代币
  isRisky: boolean       // 汇总：是否有任何安全问题
}
```

### 执行流程

```
1. useTokenSecurity({ currency, enabled })
       |
       v
2. 检查 enabled && currency
       |
       v
3. 判断链类型:
   a) 如果是 SVM 链 -> 调用 fetchGoPlusSolanaResponse
   b) 如果是 EVM 支持链 -> 调用 fetchGoPlusResponse + fetchDeFiResponse
   c) 其他链 -> 返回 undefined
       |
       v
4. 并行请求 (Promise.allSettled):
   - goPlusResponsePromise
   - deFiResponsePromise
       |
       v
5. 合并结果:
   data[field] = { goPlus: goPlusResponse?[field], deFi: deFiResponse?[field] }
       |
       v
6. 计算汇总字段:
   isHoneypot = data.is_honeypot.goPlus || data.is_honeypot.deFi
   isFoT = data.buy_tax.goPlus || data.buy_tax.deFi || ...
   isRisky = 任何字段符合 isTokenSecurityIssue 规则
       |
       v
7. 返回完整结果
```

### TokenSecurity 字段说明

| 字段 | 含义 |
|------|------|
| is_open_source | 合约是否开源 |
| is_proxy | 是否有代理合约 |
| is_mintable | 是否可铸造 |
| buy_tax / sell_tax | 买/卖税费 |
| is_honeypot | 是否貔貅 |
| cannot_buy / cannot_sell_all | 是否可买/可卖完全部 |
| is_blacklisted / is_whitelisted | 是否有黑/白名单 |
| is_anti_whale | 是否有反 whale 机制 |

---

## 4. 怎么用 AI 写这个 Hook

### 写代码时要注意什么

1. **多数据源要并发查**：用 `Promise.allSettled` 而不是 `Promise.all`，这样如果有一个数据源挂了，其他的正常返回，不会全挂。

2. **要有降级方案**：就算一个数据源失败了，另一个还能正常提供数据，不至于完全查不了。

3. **Zod transform 要处理各种边界值**：比如 '0'/'1' 这种字符串要转成 boolean，空字符串要处理成 undefined。

4. **汇总字段要设计好**：`isHoneypot`、`isRisky` 这些字段让 UI 可以快速判断一个币有没有问题，不用自己一个个字段去判断。

5. **TypeScript 类型要安全**：用 `TokenSecurity` 类型定义好所有安全字段，类型检查不能少。

### 有什么限制条件

1. **currency 不能为空**：enabled 检查里面要有 `Boolean(currency)`，没有就不查。

2. **EVM 和 Solana 要分开处理**：这两个生态的 API 和数据结构完全不一样，不能用同一套逻辑。

3. **依赖两个外部 API**：GoPlus 和 DeFi Scanner 都是第三方 API，要确保这两个都能访问。

4. **地址要用小写**：GoPlus API 要求 `currency.address.toLowerCase()`，传地址之前要先转小写。

### 状态怎么管理

| 状态 | 怎么管 | 什么意思 |
|------|--------|---------|
| 安全数据 | React Query 缓存 | 按 currency.id 缓存 |
| 加载/错误 | React Query 标准 | 自动处理 |
| 多数据源 | Promise.allSettled | 同时查，失败不影响其他的 |
| 汇总判断 | 派生计算 | isHoneypot/isRisky 这些是算出来的 |
| 缓存时间 | staleTime: 15min, gcTime: 24h | 安全数据不用实时，缓存设长一点 |

---

### 完整AI提示词模板

```
你是一个 React Query + DeFi 安全分析专家。请为以下场景编写 Hook:

【场景描述】
需要查询代币的安全信息，帮助用户判断一个币是否有风险。
需要从两个数据源获取：GoPlus 和 DeFi Scanner。

【数据源】
1. GoPlus API (EVM):
   https://api.gopluslabs.io/api/v1/token_security/${chainId}?contract_addresses=${address}

2. GoPlus API (Solana):
   https://api.gopluslabs.io/api/v1/solana/token_security?contract_addresses=${address}

3. DeFi Scanner (部分 EVM 链):
   /api/token-scanner?chainId=${chainId}&address=${address}

【链类型处理】
- SVM 链 (Solana): 只用 GoPlus Solana API
- EVM 支持链: 并行查询 GoPlus + DeFi Scanner
- 其他链: 返回 undefined

【Zod 数据转换】
GoPlus 数据中:
- '0'/'1'/'true'/'false' 字符串 -> boolean
- '' (空字符串) -> undefined
- 无法解析 -> undefined

【安全问题判断】
isTokenSecurityIssue 规则:
- is_open_source: false = 问题
- is_proxy: true = 问题
- is_mintable: true = 问题
- buy_tax: true = 问题
- is_honeypot: true = 问题
- ... 更多规则

【汇总字段】
isHoneypot = goPlus.is_honeypot || deFi.is_honeypot
isFoT = buy_tax || sell_tax 任一为 true
isRisky = 任何字段符合问题规则

【参数】
{
  currency: EvmToken | SvmToken | undefined
  enabled?: boolean
}

【返回】
{
  data: {
    [K in keyof TokenSecurity]: { goPlus?: boolean; deFi?: boolean }
  }
  isHoneypot: boolean
  isFoT: boolean
  isRisky: boolean
}

【缓存配置】
- staleTime: 900000 (15分钟)
- gcTime: 86400000 (24小时)
- placeholderData: keepPreviousData

【最佳实践】
- Promise.allSettled 处理部分失败
- 数据源分别查询再合并
- Zod transform 处理各种边界值
- 汇总字段方便 UI 使用

请输出完整代码。
```

---

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useTokenSecurity } from '@sushiswap/react-query'
import { useToken } from '@sushiswap/token'

function TokenSecurityBadge({ tokenAddress, chainId }: { tokenAddress: string; chainId: number }) {
  const token = useToken({ address: tokenAddress, chainId })
  const { data: security, isLoading } = useTokenSecurity({ currency: token })

  if (isLoading) return <SecurityBadgeSkeleton />
  if (!security) return <SecurityBadge status="unknown" />

  if (security.isHoneypot) {
    return <SecurityBadge status="danger" label="Honeypot Detected" />
  }

  if (security.isRisky) {
    return <SecurityBadge status="warning" label="Potential Risks" />
  }

  return <SecurityBadge status="safe" label="No Issues Found" />
}

function SecurityBadge({ status, label }) {
  const colors = {
    safe: 'bg-green-100 text-green-800',
    warning: 'bg-yellow-100 text-yellow-800',
    danger: 'bg-red-100 text-red-800',
    unknown: 'bg-gray-100 text-gray-800',
  }

  return (
    <span className={`px-2 py-1 rounded text-sm ${colors[status]}`}>
      {label}
    </span>
  )
}
```

### 常见使用场景

**场景1：交易前的代币安全检查**
```tsx
function SwapConfirmation({ tokenIn, tokenOut }) {
  const { data: securityIn } = useTokenSecurity({ currency: tokenIn })
  const { data: securityOut } = useTokenSecurity({ currency: tokenOut })

  const hasIssue = securityIn?.isRisky || securityOut?.isRisky

  if (hasIssue) {
    return (
      <WarningBox>
        <WarningTitle>Security Warning</WarningTitle>
        {securityIn?.isRisky && <WarningItem token={tokenIn} security={securityIn} />}
        {securityOut?.isRisky && <WarningItem token={tokenOut} security={securityOut} />}
      </WarningBox>
    )
  }

  return <SwapSummary tokenIn={tokenIn} tokenOut={tokenOut} />
}
```

**场景2：代币详情页的安全信息展示**
```tsx
function TokenSecurityPanel({ token }: { token: EvmToken }) {
  const { data: security, isLoading } = useTokenSecurity({ currency: token })

  if (isLoading) return <SecurityPanelSkeleton />
  if (!security) return null

  return (
    <Panel title="Security Information">
      <SecurityCheckRow
        label="Contract Open Source"
        value={security.data.is_open_source?.goPlus}
      />
      <SecurityCheckRow
        label="Honeypot"
        value={!security.isHoneypot}
        isFlag={security.isHoneypot}
      />
      <SecurityCheckRow
        label="Buy Tax"
        value={security.data.buy_tax?.goPlus}
        isFlag={security.data.buy_tax?.goPlus}
      />
      <SecurityCheckRow
        label="Sell Tax"
        value={security.data.sell_tax?.goPlus}
        isFlag={security.data.sell_tax?.goPlus}
      />
      <SecurityCheckRow
        label="Cannot Buy"
        value={!security.data.cannot_buy?.goPlus}
        isFlag={security.data.cannot_buy?.goPlus}
      />
      <SecurityCheckRow
        label="Cannot Sell All"
        value={!security.data.cannot_sell_all?.goPlus}
        isFlag={security.data.cannot_sell_all?.goPlus}
      />
      <SecurityCheckRow
        label="Anti-Whale"
        value={security.data.is_anti_whale?.goPlus}
      />
      <SecurityCheckRow
        label="Blacklisted"
        value={!security.data.is_blacklisted?.goPlus}
        isFlag={security.data.is_blacklisted?.goPlus}
      />

      {security.isHoneypot && (
        <Alert type="danger" message="This token may be a honeypot. Do not buy!" />
      )}
      {security.isFoT && (
        <Alert type="warning" message="This token has transfer fees." />
      )}
    </Panel>
  )
}

function SecurityCheckRow({ label, value, isFlag }) {
  return (
    <div className="flex justify-between items-center py-2 border-b">
      <span>{label}</span>
      {value === true && <CheckIcon className="text-green-500" />}
      {value === false && <XIcon className={isFlag ? 'text-red-500' : 'text-gray-400'} />}
      {value === undefined && <MinusIcon className="text-gray-400" />}
    </div>
  )
}
```

### Dos and Don'ts

**Do（推荐做法）：**
- ✅ 使用 `isHoneypot` 和 `isRisky` 汇总字段快速判断
- ✅ 使用 `isLoading` 显示加载状态
- ✅ 检查 `security` 是否存在再访问属性
- ✅ 使用 GoPlus 和 DeFi 两个数据源综合判断

**Don't（避免做法）：**
- ❌ 不要只看单一数据源，两个源都有参考价值
- ❌ 不要忽略 `isLoading` 状态，应该显示加载中
- ❌ 不要假设所有字段都有值，有些可能是 undefined
- ❌ 不要在安全数据没加载完时就显示"安全"，应该等待或显示未知

### 注意事项

1. **双数据源**：GoPlus 和 DeFi Scanner 的数据可能不一致，应该综合考虑

2. **15分钟缓存**：安全数据不需要频繁刷新

3. **EVM 和 Solana 处理不同**：Solana 链只使用 GoPlus Solana API

4. **汇总字段**：isHoneypot、isFoT、isRisky 是综合判断的结果，方便快速筛选

5. **字段可能是 undefined**：如果某个数据源没有返回该字段，值可能是 undefined
