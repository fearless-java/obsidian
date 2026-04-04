> 源代码路径: `apps/web/src/lib/hooks/react-query/tokens/useCoinGeckoTokenInfo.ts`

# useCoinGeckoTokenInfo Hook Tutorial

## 1. 大白话讲讲这个hook的作用

`useCoinGeckoTokenInfo` 是用来获取单个代币在 CoinGecko 上的市场信息的 Hook。它获取的数据包括：当前价格、市值排名、24小时交易量、市值、历史最高价（ATH）、历史最低价（ATL）、流通量和总供应量。

简单来说：**就是查一个币在 CoinGecko 上的"档案信息"——价格、市值、排名、ATH/ATL 等一手数据。**

---

## 2. 为什么需要封装该hook

### 原始问题
1. **CoinGecko API 链ID映射复杂**：CoinGecko 使用自己的一套 chain ID（如 'eth', 'bsc', 'polygon_pos'），需要和 SushiSwap 的 ChainId 映射
2. **响应数据结构复杂**：CoinGecko 返回嵌套很深的 JSON，需要用 Zod schema *(一个TypeScript优先的数据验证库，用于验证和转换API返回的数据)* 提取关键字段
3. **不支持的链需要跳过**：如果链 ID 不在映射表中，应该优雅地抛出错误
4. **缓存 CoinGecko 数据**：CoinGecko 限制请求频率，需要较长的缓存时间

### 封装带来的好处
1. **自动链 ID 映射**：内部处理 ChainId -> CoinGecko ID 的转换
2. **数据验证和提取**：用 Zod schema 验证并提取关键字段
3. **长缓存时间**：15分钟 staleTime 避免频繁请求 CoinGecko
4. **placeholderData**：切换代币时保持旧数据不闪烁

---

## 3. 执行逻辑和数据流向

### 输入 (Params)
```typescript
{
  token?: Token         // 代币对象，包含 chainId 和 address
  enabled?: boolean     // 是否启用，默认 true
}
```

### 输出 (Return)
```typescript
{
  price: number          // 当前 USD 价格
  rank: number | null   // 市值排名
  volume24h: number      // 24小时交易量（USD）
  marketCap: number      // 市值（USD）
  ath: number            // 历史最高价
  atl: number            // 历史最低价
  circulatingSupply: number   // 流通量
  totalSupply: number | null   // 总供应量
}
```

### 执行流程

```
1. useCoinGeckoTokenInfo({ token, enabled })
       |
       v
2. 检查 enabled && token
       |
       v
3. 检查 token.chainId 是否在 COINGECKO_CHAIN_ID_BY_NAME 映射中
   - 不支持则抛出 Error('Invalid token')
       |
       v
4. 构建 URL:
   https://api.coingecko.com/api/v3/coins/${coingeckoChainId}/contract/${tokenAddress}
       |
       v
5. 调用 CoinGecko API
       |
       v
6. 用 coinGeckoSchema 验证返回数据
       |
       v
7. 提取关键字段:
   {
     price: parsed.market_data.current_price.usd,
     rank: parsed.market_cap_rank,
     volume24h: parsed.market_data.total_volume.usd,
     marketCap: parsed.market_data.market_cap.usd,
     ath: parsed.market_data.ath.usd,
     atl: parsed.market_data.atl.usd,
     circulatingSupply: parsed.market_data.circulating_supply,
     totalSupply: parsed.market_data.total_supply
   }
       |
       v
8. 返回提取后的数据
```

### ChainId 映射表

```typescript
COINGECKO_CHAIN_ID_BY_NAME = {
  1: 'eth',           // Ethereum
  56: 'bsc',          // BSC
  137: 'polygon_pos', // Polygon
  43114: 'avax',      // Avalanche
  25: 'cro',          // Cronos
  // ... 更多链
}
```

---

## 4. 怎么用 AI 写这个 Hook

### 写代码时要注意什么

1. **链 ID 映射表要用 const 对象**：这样 TypeScript 编译的时候能检查，IDE 也能给出正确的自动补全。

2. **类型守卫函数要做好**：`isCoinGeckoChainId` 这样的函数用来做类型收窄，让 TypeScript 知道这个 chainId 是合法的。

3. **缓存时间要长**：CoinGecko 对请求频率有限制，15 分钟的缓存是比较安全的，别设太短。

4. **切换代币时保留旧数据**：用 `placeholderData: keepPreviousData` 可以让界面在切换代币时不会闪一下。

5. **外部数据要用 Zod schema 验证**：保证从 CoinGecko 拿到的数据格式是对的。

### 有什么限制条件

1. **token 必须存在**：enabled 检查里面要有 `!!token`，token 不存在就不查。

2. **只支持映射表里有的链**：如果链 ID 不在映射表中，直接抛错误，不会发请求。

3. **依赖 CoinGecko API**：这是第三方的 API，不是自己写的内部接口。

4. **不支持 Solana**：这个 Hook 只处理 EVM 链，Solana 链需要另外的处理方式。

### 状态怎么管理

| 状态 | 怎么管 | 什么意思 |
|------|--------|---------|
| 代币信息 | React Query 缓存 | 按 token.id 缓存 |
| 加载状态 | React Query isLoading | 自动处理 |
| 切换代币 | placeholderData | keepPreviousData 让旧数据保留，不闪 |
| 缓存时间 | staleTime: 15min, gcTime: 24h | 缓存时间很长，15分钟才过期 |
| 启用开关 | enabled && !!token | 两个条件都要满足 |

---

### 完整AI提示词模板

```
你是一个 React Query + 第三方 API 集成专家。请为以下场景编写 Hook:

【场景描述】
需要获取单个代币在 CoinGecko 上的市场数据。
CoinGecko 是一个流行的加密货币数据 API。

【背景知识】
1. CoinGecko API: https://api.coingecko.com/api/v3/coins/{chainId}/contract/{address}
2. CoinGecko 使用自己的 chain ID 命名（如 'eth', 'bsc', 'polygon_pos'）
3. SushiSwap 使用 ChainId 枚举，需要映射到 CoinGecko 的 ID

【链 ID 映射要求】
需要支持至少以下链:
- Ethereum -> 'eth'
- BSC -> 'bsc'
- Polygon -> 'polygon_pos'
- Avalanche -> 'avax'
- Arbitrum -> 'arbitrum'
- Optimism -> 'optimism'
- 其他 EVM 链...

【技术要求】
1. 使用 @tanstack/react-query useQuery
2. 使用 Zod schema 验证 CoinGecko 返回数据
3. 类型守卫检查 chainId 是否支持
4. 提取字段: price, rank, volume24h, marketCap, ath, atl, circulatingSupply, totalSupply

【参数】
{
  token?: Token  // 包含 chainId 和 address 的代币对象
  enabled?: boolean
}

【返回】
{
  price: number
  rank: number | null
  volume24h: number
  marketCap: number
  ath: number
  atl: number
  circulatingSupply: number
  totalSupply: number | null
}

【缓存配置】
- staleTime: 900000 (15分钟)
- gcTime: 86400000 (24小时)
- placeholderData: keepPreviousData (切换时保持旧数据)
- enabled: !!token && enabled

【最佳实践】
- 使用 const 对象存储链 ID 映射
- 类型守卫函数用于类型收窄
- Zod schema 验证外部 API 数据
- 较长缓存避免触发 rate limit

请输出完整代码。
```

---

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useCoinGeckoTokenInfo } from '@sushiswap/react-query'
import { useToken } from '@sushiswap/token'

function TokenMarketInfo({ tokenAddress, chainId }: { tokenAddress: string; chainId: number }) {
  const token = useToken({ address: tokenAddress, chainId })
  const { data: info, isLoading } = useCoinGeckoTokenInfo({ token })

  if (isLoading) return <div>Loading market data...</div>
  if (!info) return <div>Market data unavailable</div>

  return (
    <div>
      <div>Price: ${info.price.toLocaleString()}</div>
      <div>Rank: #{info.rank ?? 'N/A'}</div>
      <div>24h Volume: ${info.volume24h.toLocaleString()}</div>
      <div>Market Cap: ${info.marketCap.toLocaleString()}</div>
      <div>ATH: ${info.ath.toLocaleString()}</div>
      <div>ATL: ${info.atl.toLocaleString()}</div>
    </div>
  )
}
```

### 常见使用场景

**场景1：代币详情页的市场数据**
```tsx
function TokenDetailPage({ token }: { token: EvmToken }) {
  const { data: info, isLoading } = useCoinGeckoTokenInfo({ token })

  return (
    <Page>
      <TokenHeader token={token} price={info?.price} rank={info?.rank} />

      <div className="grid grid-cols-2 gap-4">
        <MarketStat label="Price" value={info?.price} format={(v) => `$${v?.toFixed(2)}`} />
        <MarketStat label="24h Volume" value={info?.volume24h} format={(v) => formatLargeNumber(v)} />
        <MarketStat label="Market Cap" value={info?.marketCap} format={(v) => formatLargeNumber(v)} />
        <MarketStat label="Circulating Supply" value={info?.circulatingSupply} />
      </div>

      <div className="mt-4">
        <ATHAndATLDisplay ath={info?.ath} atl={info?.atl} currentPrice={info?.price} />
      </div>
    </Page>
  )
}

function MarketStat({ label, value, format }) {
  if (value === undefined) return <Skeleton />
  return (
    <div>
      <div className="text-gray-500">{label}</div>
      <div className="text-xl font-bold">{format ? format(value) : value}</div>
    </div>
  )
}
```

**场景2：价格变化百分比显示**
```tsx
function PriceChangeIndicator({ token }: { token: EvmToken }) {
  // 注意：当前 hook 不直接返回 priceChange24h，需要额外计算或从其他源获取
  const { data: info } = useCoinGeckoTokenInfo({ token })

  return (
    <div className="flex items-center gap-2">
      <span className="text-gray-500">Current Price</span>
      <span className="text-xl font-semibold">
        ${info?.price?.toFixed(token.decimals > 6 ? 2 : token.decimals)}
      </span>
    </div>
  )
}
```

### Dos and Don'ts

**Do（推荐做法）：**
- ✅ 使用 `info?.rank ?? 'N/A'` 处理可能的 null 值
- ✅ 使用 `toLocaleString()` 格式化大数字显示
- ✅ 使用 `formatLargeNumber()` 辅助函数处理市值等大数字
- ✅ 检查 `info` 是否存在再显示数据

**Don't（避免做法）：**
- ❌ 不要在 token 不存在时调用 hook，应该先检查 token
- ❌ 不要假设 rank 总是存在，有些币可能没有排名
- ❌ 不要频繁请求，15分钟缓存是CoinGecko建议的
- ❌ 不要直接显示原始数字，应该格式化（toLocaleString、toFixed等）

### 注意事项

1. **只支持映射表中的链**：如果链不在 COINGECKO_CHAIN_ID_BY_NAME 映射中，会抛出错误

2. **15分钟缓存**：这是 CoinGecko 的 rate limit 限制，不要设置更短的刷新时间

3. **切换代币时保持旧数据**：使用 `placeholderData: keepPreviousData` 避免闪烁

4. **rank 可能是 null**：有些币没有市值排名

5. **EVM only**：这个 hook 只支持 EVM 链，Solana 等其他链需要其他处理
