> 源代码路径: `apps/web/src/lib/hooks/react-query/prices/useAllPrices.ts`

# useAllPrices Hook Tutorial

## 1. 大白话讲讲这个hook的作用

`useAllPrices` 是一个用来获取所有代币价格的 React Query Hook。它从 `/api/price` 接口一次性获取整个应用中所有需要代币的价格数据，然后把这些原始数据转换成一种特殊的数据结构（Map），方便快速查询某个特定链上、特定代币地址对应的价格。

简单来说：**它就是应用启动时或刷新时获取全局代币价格表的地方，所有需要显示价格的地方都从这里拿数据。**

---

## 2. 为什么需要封装该hook

### 原始问题
- 每次需要某个代币价格时，单独发请求去获取是极其低效的，会导致大量重复请求
- 原始 API 返回的价格数据格式是扁平的 `Record<Address, number>`，不够结构化，查询不便

### 封装带来的好处
1. **缓存与复用**：通过 React Query 的 `staleTime: 60s` 和 `gcTime: 1h`，同一个价格数据可以被多个组件共享，避免重复请求
2. **数据结构转换**：`hydrate` 函数将扁平的链地址映射转换成 `Map<ChainId, LowercaseMap<Address, Fraction>>` 嵌套 Map 结构，查询效率从 O(n) 降到 O(1)
3. **启用/禁用控制**：通过 `enabled` 参数，可以在不需要时关闭查询，节省资源
4. **数据格式化**：使用 `Fraction` 类型保证价格精度，使用 `withoutScientificNotation` 避免科学计数法导致精度丢失
5. **窗口焦点重取**：关闭 `refetchOnWindowFocus` 避免用户切换标签页时频繁刷新价格

---

## 3. 执行逻辑和数据流向

### 输入 (Params)
```typescript
{
  enabled?: boolean  // 是否启用查询，默认 true
}
```

### 输出 (Return)
```typescript
{
  data: Map<ChainId, LowercaseMap<Address, Fraction>>  // 嵌套Map：链ID -> 代币地址 -> 价格(Fraction)
  isLoading: boolean
  isError: boolean
  // ... 其他 React Query 标准返回字段
}
```

### 执行流程

```
1. 组件调用 useAllPrices({ enabled })
       |
       v
2. React Query 检查 enabled && queryKey 决定是否发起请求
       |
       v
3. 发起 fetch('/api/price') 请求原始价格数据
       |
       v
4. 原始数据格式:
   {
     "1": { "0x...": 1800.5, "0x...": 1.0 },   // ChainId 1 (Ethereum)
     "56": { "0x...": 300.2 },                  // ChainId 56 (BSC)
     ...
   }
       |
       v
5. hydrate(data) 转换:
   - 遍历每个 chainId 和代币地址
   - 将数字价格转换为 Fraction 高精度分数
   - 返回嵌套 Map 结构
       |
       v
6. React Query 缓存结果（staleTime: 60s, gcTime: 1h）
       |
       v
7. 组件通过 data?.get(chainId)?.get(address) 查询价格
```

### 关键转换逻辑 (hydrate)
```typescript
// 原始: { "1": { "0x123...": 1800.5 } }
// 转换后: Map(1) -> LowercaseMap("0x123...") -> Fraction(1800.5)
```

---

## 4. 怎么用 AI 写这个 Hook

### 写代码时要注意什么

1. **queryKey 要设计好**：用一个简洁稳定的键名就行，比如 `['/api/price']`，不要把内部实现细节暴露出去。

2. **数据转换要纯净**：hydrate 函数只负责把原始数据转成好用的格式，不要在这个函数里发请求。这样方便单独测试，出了也好排查。

3. **钱相关的数字要精确**：价格这种数据别用普通的 number，要用 `Fraction` 类型来处理，不然精度丢了就麻烦了。

4. **缓存时间要合理**：
   - `staleTime: 60s` - 价格数据 60 秒内算是新鲜的，不用重新拉
   - `gcTime: 1h` - 缓存保留 1 小时后自动清理

5. **用 select 选项做转换**：React Query 的 select 可以帮你在数据返回之前就做好预处理，查询层就完成了。

### 有什么限制条件

1. **只能查 SushiSwap 支持的链**：不是所有链的价格都能查，只支持 SushiSwap 自己支持的那些链。

2. **用的是内部接口**：这个 Hook 调的是 `/api/price`，不是直接调第三方价格 API。

3. **关了就不请求**：如果 `enabled: false`，那就什么都不干，不会发请求。

4. **返回的是 Map 结构**：查价格的时候要用 `.get(chainId).get(address)` 这种方式，不是普通的对象访问。

### 状态怎么管理

| 状态 | 怎么管 | 什么意思 |
|------|--------|---------|
| 价格数据 | React Query 缓存 | 用 queryKey `['/api/price']` 作为缓存的键 |
| 加载状态 | React Query isLoading | 自动处理，不用手动管 |
| 错误状态 | React Query isError | 自动处理 |
| 启用开关 | enabled 参数 | 传 false 就关掉，不发请求 |
| 定时刷新 | staleTime | 60秒后数据过期，后台会悄悄刷新 |

---

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useAllPrices } from '@sushiswap/react-query'

function TokenPriceDisplay({ chainId, tokenAddress }) {
  const { data: prices } = useAllPrices()

  // 查询价格
  const price = prices?.get(chainId)?.get(tokenAddress.toLowerCase())

  if (!price) return <span>Loading...</span>

  return <span>{price.toSignificant(6)} USD</span>
}
```

### 常见使用场景

**场景1：在交易页面显示代币价格**
```tsx
function SwapPage() {
  const { data: prices } = useAllPrices()

  const ethPrice = prices?.get(1)?.get('0xC02aa...'.toLowerCase())
  const usdcPrice = prices?.get(1)?.get('0xA0b8...'.toLowerCase())

  return (
    <div>
      ETH价格: {ethPrice?.toSignificant(6)}
      USDC价格: {ethPrice?.toSignificant(6)}
    </div>
  )
}
```

**场景2：在池子页面显示 TVL 和手续费收益**
```tsx
function PoolStats({ pool }) {
  const { data: prices } = useAllPrices()

  const token0Price = prices?.get(pool.chainId)?.get(pool.token0.address.toLowerCase())
  const token1Price = prices?.get(pool.chainId)?.get(pool.token1.address.toLowerCase())

  const tvlUSD = pool.tvl * token0Price + pool.tvl * token1Price

  return <div>TVL: ${tvlUSD?.toFixed(2)}</div>
}
```

### Dos and Don'ts

**Do（推荐做法）：**
- ✅ 使用 `.toLowerCase()` 处理代币地址，因为 Map 的 key 是小写的
- ✅ 使用 `?.` 可选链访问数据，因为初始加载时 data 可能是 undefined
- ✅ 使用 `toSignificant()` 或 `toFixed()` 格式化价格显示
- ✅ 在应用启动时启用查询，全局共享价格数据

**Don't（避免做法）：**
- ❌ 不要对每个代币单独发起请求，应该使用这个全局价格查询
- ❌ 不要直接使用 `data[chainId][address]` 访问，因为 data 是 Map 不是普通对象
- ❌ 不要忽略价格为 0 或不存在的情况，应该显示合适的 UI
- ❌ 不要设置过短的 staleTime，价格不需要实时刷新

### 注意事项

1. **数据结构是 Map**：返回的 `data` 是 `Map<ChainId, LowercaseMap<Address, Fraction>>`，不是普通对象，访问方式是用 `.get()` 方法

2. **Fraction 类型**：价格是 `Fraction` 类型，需要用 `.toSignificant()` 或 `.toFixed()` 方法转换为可读字符串

3. **缓存全局共享**：同一个 `queryKey` 的查询会被全局共享，多个组件使用 `useAllPrices` 只会发起一次请求

4. **初始值是 undefined**：在数据加载完成前，`data` 是 undefined，需要使用可选链或条件渲染处理
