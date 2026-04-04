> 源代码路径: `apps/web/src/lib/hooks/react-query/twap/useTwapOrders.ts`

# useTwapOrders Hook Tutorial

## 1. 大白话讲讲这个hook的作用

`useTwapOrders` 是用来获取用户 TWAP（时间加权平均价格）订单列表的 Hook。它不仅查询链上的订单，还管理本地存储（localStorage）中未上链的订单（如用户刚创建但交易还在 pending 的订单），并把所有订单按状态分类（Open/Completed/Expired/Canceled）。

简单来说：**就是帮你查"我这个钱包地址在 SushiSwap TWAP 上挂了多少单"，包括已成交的、还没成交的、已取消的。**

---

## 2. 为什么需要封装该hook

### 原始问题
1. **双重数据源**：链上订单（TwapSDK.getOrders）和本地订单（localStorage）需要合并
2. **状态同步复杂**：需要对比链上订单和本地已取消订单列表，更新订单状态
3. **新订单添加**：用户创建新订单时，需要同步到 localStorage 和 React Query cache
4. **分类汇总**：返回的数据需要按 `OrderStatus` *(TWAP订单的状态枚举，包含Open、Completed、Expired、Canceled)* 分组（Open、Completed 等）
5. **persist 逻辑**：localStorage 中的订单需要持久化，不应该丢失

### 封装带来的好处
1. **统一的订单视图**：链上订单和本地订单合并后的完整列表
2. **状态自动更新**：取消的订单自动标记为 Canceled
3. **分类数据**：直接获取 `orders.ALL`、`orders.Open` 等分类好的数据
4. **缓存同步**：新订单创建后自动 invalidateQueries *(React Query中用于使缓存失效并触发重新查询的方法)* 更新缓存
5. **20 秒轮询**：定期同步链上状态

---

## 3. 执行逻辑和数据流向

### 输入 (Params)
```typescript
{
  chainId: TwapSupportedChainId   // 链ID
  account: EvmAddress | undefined  // 钱包地址
  enabled?: boolean = true         // 是否启用
}
```

### 输出 (Return)
```typescript
{
  data: {
    ALL: TwapOrder[]                    // 所有订单
    [OrderStatus.Open]: TwapOrder[]    // 进行中
    [OrderStatus.Completed]: TwapOrder[] // 已完成
    [OrderStatus.Expired]: TwapOrder[]  // 已过期
    [OrderStatus.Canceled]: TwapOrder[] // 已取消
  }
  isLoading: boolean
  isError: boolean
  // ...
}
```

### 执行流程

```
1. useTwapOrders({ chainId, account, enabled })
       |
       v
2. 调用 useTwapOrdersQuery 获取基础数据
       |
       v
3. useTwapOrdersQuery 内部:
   a) 检查 account && config
   b) 获取 SDK 订单: TwapSDK.onNetwork(chainId).getOrders(account)
   c) 获取本地订单: getCreatedOrders()
   d) 合并订单（本地未上链的加进去，SDK 已有的移除本地）
   e) 获取已取消 ID: getCancelledOrderIds()
   f) 更新状态: 取消列表中的订单标记为 Canceled
   g) 计算 fillDelayMs 和 progress
       |
       v
4. useMemo 分类汇总:
   - ALL: 所有订单
   - Open: 过滤 status === Open 并按时间排序
   - Completed: 过滤 status === Completed 并排序
   - Expired: 过滤 status === Expired 并排序
   - Canceled: 过滤 status === Canceled 并排序
       |
       v
5. 返回分类后的数据
```

### 本地订单管理 (usePersistedOrdersStore)

```typescript
// 存储键
ordersKey = `orders-${chainId}:${account}`
cancelledOrderIdsKey = `cancelled-orders-${chainId}:${account}`

// 添加新订单到 localStorage 和 cache
addCreatedOrder(orderId, txHash, params, srcToken, dstToken)

// 添加取消 ID 到 localStorage 和 cache
addCancelledOrderId(orderId)

// 删除本地订单
deleteCreatedOrder(id)
deleteCancelledOrderId(orderId)
```

---

## 4. 怎么用 AI 写这个 Hook

### 写代码时要注意什么

1. **localStorage 和 React Query 要配合着用**：这是管理前端状态的常见套路，链上数据用 React Query 缓存，本地新建的订单用 localStorage 持久化。

2. **mutation 回调里直接更新缓存**：不要等下次 fetch 完再更新，要用 `queryClient.setQueryData` 直接改缓存，这样界面反应更快。

3. **SDK 订单和本地订单合并逻辑要清晰**：已经在链上的订单要从 localStorage 里删掉，还没上链的要加进列表。

4. **按状态分类的派生数据要用 useMemo**：这样可以避免每次渲染都重新算一遍。

5. **轮询间隔要合理**：TWAP 订单状态不会变来变去，20 秒查一次就够了。

### 有什么限制条件

1. **依赖 TwapSDK**：这是 TWAP SDK 提供的接口，没有这个 SDK 就跑不了。

2. **必须有 account**：enabled 检查里有 `!!account`，没有钱包地址就不查。

3. **需要 wagmi config**：因为 SDK 底层需要 provider 来跟区块链交互。

4. **localStorage 持久化**：用户关掉浏览器再打开，之前创建的订单不会丢。

### 状态怎么管理

| 状态 | 怎么管 | 什么意思 |
|------|--------|---------|
| 链上订单 | React Query 缓存 | 每 20 秒轮询一次 |
| 本地新订单 | localStorage + queryClient.setQueryData | 持久化存着，同时马上更新界面 |
| 已取消 ID | localStorage | 持久化 |
| 订单状态 | 计算属性 | 根据 SDK 订单和取消列表来算 |
| 分类数据 | useMemo | 缓存计算结果，不用每次都重算 |

---

### 完整AI提示词模板

```
你是一个 React Query + Web3 订单管理专家。请为以下场景编写 Hook:

【场景描述】
需要管理用户的 TWAP（时间加权平均价格）订单列表。
TWAP 订单会分成多个 chunk 在链上执行，需要跟踪每个订单的状态。

【技术要求】
1. 使用 @tanstack/react-query useQuery
2. 使用 TwapSDK.onNetwork(chainId).getOrders(account) 获取链上订单
3. 使用 localStorage 存储本地新建但还未上链的订单
4. 合并 SDK 订单和本地订单
5. 按订单状态分类: Open, Completed, Expired, Canceled

【本地订单管理】
需要 usePersistedOrdersStore:
- getCreatedOrders(): 从 localStorage 获取本地订单
- getCancelledOrderIds(): 获取已取消订单 ID 列表
- addCreatedOrder(): 添加本地订单到 localStorage 和 cache
- addCancelledOrderId(): 添加取消 ID 到 localStorage 和 cache

【合并逻辑】
1. SDK 订单中已有的本地订单 -> 从 localStorage 删除
2. SDK 订单中没有的本地订单 -> 合并到列表
3. 用 getCancelledOrderIds() 标记已取消的订单

【订单状态更新】
if (canceledOrders.has(order.id) && status !== 'Canceled') {
  status = 'Canceled'
}

【分类派生状态】
使用 useMemo 按 status 分类:
- ALL: 所有订单
- [OrderStatus.Open]: filter + sort by createdAt desc
- [OrderStatus.Completed]: 同上
- [OrderStatus.Expired]: 同上
- [OrderStatus.Canceled]: 同上

【TwapSDK 集成】
import { TwapSDK } from 'src/lib/swap/twap'
import { Order, OrderStatus, buildOrder } from '@orbs-network/twap-sdk'

【TwapOrder 类型】
type TwapOrder = Order & {
  status: OrderStatus
  fillDelayMs: number
}

【参数】
{
  chainId: TwapSupportedChainId
  account: EvmAddress | undefined
  enabled?: boolean
}

【缓存配置】
- refetchInterval: 20000 (20秒)
- enabled: Boolean(enabled && account && config)

【最佳实践】
- localStorage 持久化本地订单
- queryClient.setQueryData 直接更新缓存
- SDK 已有订单从本地移除
- useMemo 派生分类数据
- 20 秒轮询保持同步

请输出完整代码。
```

---

### 生产级提示词示例

```
请帮我创建一个完整的 TWAP 订单管理 Hook 系统。

【需求】
1. 查询用户在 SushiSwap TWAP 上的所有订单
2. 管理本地存储的新订单（未上链的）
3. 管理已取消订单列表
4. 按状态分类返回订单

【组件结构】
1. useTwapOrders - 主 Hook，返回分类后的订单
2. useTwapOrdersQuery - 内部 useQuery，获取和合并订单
3. usePersistedOrdersStore - localStorage + cache 管理

【usePersistedOrdersStore 实现】
存储键格式:
- orders-${chainId}:${account}
- cancelled-orders-${chainId}:${account}

方法:
- getCreatedOrders(): Order[]
- getCancelledOrderIds(): number[]
- addCreatedOrder(orderId, txHash, params, srcToken, dstToken)
- addCancelledOrderId(orderId)
- deleteCreatedOrder(id)
- deleteCancelledOrderId(orderId)

【合并逻辑伪代码】
sdkOrders = await TwapSDK.onNetwork(chainId).getOrders(account)

localOrders = getCreatedOrders()
sdkOrderIds = sdkOrders.map(o => o.id)

localOrders.forEach(localOrder => {
  if (sdkOrderIds.includes(localOrder.id)) {
    deleteCreatedOrder(localOrder.id) // 已上链，从本地删除
  } else {
    sdkOrders.unshift(localOrder) // 未上链，加入列表
  }
})

canceledIds = new Set(getCancelledOrderIds())
orders = sdkOrders.map(order => {
  if (canceledIds.has(order.id)) {
    return { ...order, status: OrderStatus.Canceled }
  }
  return order
})

【返回结构】
{
  data: {
    ALL: TwapOrder[]
    [OrderStatus.Open]: TwapOrder[]
    [OrderStatus.Completed]: TwapOrder[]
    [OrderStatus.Expired]: TwapOrder[]
    [OrderStatus.Canceled]: TwapOrder[]
  }
}

【依赖】
- @tanstack/react-query
- @orbs-network/twap-sdk (Order, OrderStatus, buildOrder, getOrderFillDelayMillis)
- wagmi useConfig
- src/lib/swap/twap (TwapSDK)

请输出完整代码，包括 usePersistedOrdersStore 和 useTwapOrders。
```

---

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useTwapOrders } from '@sushiswap/react-query'
import { OrderStatus } from '@orbs-network/twap-sdk'

function MyTwapOrders({ chainId, account }) {
  const { data: orders, isLoading } = useTwapOrders({ chainId, account })

  if (isLoading) return <OrdersSkeleton />
  if (!orders?.ALL?.length) return <EmptyOrders>No TWAP orders</EmptyOrders>

  return (
    <div>
      <OrdersTabs orders={orders} />
      <OrdersList orders={orders.ALL} />
    </div>
  )
}

function OrdersTabs({ orders }) {
  return (
    <div className="flex gap-4">
      <Tab count={orders.ALL.length}>All</Tab>
      <Tab count={orders[OrderStatus.Open]?.length}>Open</Tab>
      <Tab count={orders[OrderStatus.Completed]?.length}>Completed</Tab>
      <Tab count={orders[OrderStatus.Expired]?.length}>Expired</Tab>
      <Tab count={orders[OrderStatus.Canceled]?.length}>Canceled</Tab>
    </div>
  )
}
```

### 常见使用场景

**场景1：TWAP 订单列表页面**
```tsx
function TwapOrdersPage({ chainId, account }) {
  const { data: orders, isLoading } = useTwapOrders({ chainId, account })
  const [activeTab, setActiveTab] = useState(OrderStatus.Open)

  const displayOrders = activeTab === 'ALL' ? orders?.ALL : orders?.[activeTab]

  return (
    <Page>
      <PageHeader title="My TWAP Orders" />

      <TabGroup activeIndex={activeTab} onChange={setActiveTab}>
        <Tab name="All" count={orders?.ALL?.length ?? 0} />
        <Tab name="Open" count={orders?.[OrderStatus.Open]?.length ?? 0} />
        <Tab name="Completed" count={orders?.[OrderStatus.Completed]?.length ?? 0} />
        <Tab name="Expired" count={orders?.[OrderStatus.Expired]?.length ?? 0} />
        <Tab name="Canceled" count={orders?.[OrderStatus.Canceled]?.length ?? 0} />
      </TabGroup>

      <OrdersTable orders={displayOrders} />
    </Page>
  )
}
```

**场景2：订单状态徽章**
```tsx
function OrderStatusBadge({ order }) {
  const statusColors = {
    [OrderStatus.Open]: 'bg-blue-100 text-blue-800',
    [OrderStatus.Completed]: 'bg-green-100 text-green-800',
    [OrderStatus.Expired]: 'bg-gray-100 text-gray-800',
    [OrderStatus.Canceled]: 'bg-red-100 text-red-800',
  }

  const statusLabels = {
    [OrderStatus.Open]: 'Open',
    [OrderStatus.Completed]: 'Completed',
    [OrderStatus.Expired]: 'Expired',
    [OrderStatus.Canceled]: 'Canceled',
  }

  return (
    <span className={`px-2 py-1 rounded text-xs ${statusColors[order.status]}`}>
      {statusLabels[order.status]}
    </span>
  )
}

function TwapOrderRow({ order }) {
  return (
    <tr>
      <td><OrderStatusBadge order={order} /></td>
      <td>{order.srcToken.symbol} → {order.dstToken.symbol}</td>
      <td>{order.amount.toSignificant(6)}</td>
      <td>{order.filledAmount?.toSignificant(6) ?? '0'}</td>
      <td>
        <ProgressBar progress={order.progress} />
      </td>
      <td><OrderActions order={order} /></td>
    </tr>
  )
}
```

**场景3：取消订单**
```tsx
function CancelOrderButton({ order, onSuccess }) {
  const { cancelOrder, isPending } = useCancelTwapOrder()

  const handleCancel = async () => {
    try {
      await cancelOrder(order)
      onSuccess?.()
    } catch (error) {
      console.error('Failed to cancel order:', error)
    }
  }

  return (
    <Button
      variant="outline"
      size="sm"
      onClick={handleCancel}
      disabled={isPending || order.status !== OrderStatus.Open}
    >
      {isPending ? 'Canceling...' : 'Cancel'}
    </Button>
  )
}
```

### Dos and Don'ts

**Do（推荐做法）：**
- ✅ 使用 `orders?.ALL` 或 `orders?.[OrderStatus.Open]` 等分类访问数据
- ✅ 使用 `OrderStatus` 枚举判断订单状态
- ✅ 使用 `order.progress` 或 `order.filledAmount` 显示完成进度
- ✅ 检查 account 存在后再调用 hook

**Don't（避免做法）：**
- ❌ 不要假设订单一定存在，应该检查 `orders?.ALL?.length`
- ❌ 不要忽略 `isLoading` 状态，应该显示加载状态
- ❌ 不要直接使用 OrderStatus 的数字，应该使用枚举
- ❌ 不要在 account 为空时调用，应该先检查

### 注意事项

1. **双重数据源**：返回的订单包含链上订单和本地新建但未上链的订单

2. **自动状态同步**：取消的订单会自动标记为 Canceled

3. **20秒轮询**：每20秒自动同步一次链上状态

4. **localStorage 持久化**：新建的订单会保存到 localStorage，刷新页面不会丢失

5. **分类派生状态**：orders.ALL、orders.Open 等是通过 useMemo 计算的，直接使用不需要再次过滤
