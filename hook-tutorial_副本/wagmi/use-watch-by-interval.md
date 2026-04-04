> 源代码路径: `apps/web/src/lib/wagmi/hooks/watch/useWatchByInterval.ts`

# useWatchByInterval

## 大白话讲讲这个hook的作用

`useWatchByInterval` 是一个基于时间间隔来刷新 React Query 缓存数据的 Hook。

大白话：就是"每隔 N 毫秒，自动刷新这些数据"。这是传统的轮询方式，比区块监听更简单、更广泛适用。

与 `useWatchByBlock` 不同，它不依赖区块链浏览器/节点支持 websocket 订阅，只要有 RPC 就能工作。

## 讲讲为什么需要封装该hook

1. **简单可靠**：基于 setInterval，不依赖特殊 API。

2. **跨链兼容**：不需要链支持特定的事件订阅，任何链都能用。

3. **精确控制**：`interval` 参数完全控制刷新频率。

4. **批量刷新**：可以同时刷新多个 query key。

5. **自动清理**：`setInterval` 在组件卸载时自动清理。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
type UseWatchByIntervalKey = {
  key: QueryKey        // 单个 query key
  keys?: never
}

type UseWatchByIntervalKeys = {
  key?: never
  keys: QueryKey[]     // 多个 query keys
}

type UseWatchByInterval = {
  interval: number     // 刷新间隔（毫秒）
} & (UseWatchByIntervalKey | UseWatchByIntervalKeys)
```

### 执行逻辑详解

#### 1. 规范化 Keys

```typescript
const keys = useMemo(() => {
  if (key) return [key]
  if (_keys) return _keys
  return []
}, [_keys, key])
```

#### 2. 设置定时刷新

```typescript
useEffect(() => {
  const int = setInterval(() => {
    keys.forEach((key) => {
      queryClient.invalidateQueries({ queryKey: key })
    })
  }, interval)

  return () => clearInterval(int)
}, [keys, queryClient, interval])
```

### 数据流向图

```
输入: key(s), interval (ms)
         ↓
    setInterval(interval)
         ↓
    每隔 interval 毫秒:
         ↓
    keys.forEach invalidateQueries
         ↓
    React Query 重新获取数据
         ↓
    组件卸载时 clearInterval
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：余额10秒价格5秒**。余额数据变化不那么频繁，10秒刷新一次够了。价格变化更敏感，可以设5秒左右。但都不要设太短。

**第二点：间隔太短会限流**。RPC节点对请求频率有限制。如果间隔设成1秒，大量查询会让节点限流。建议最少5秒，10秒更稳妥。

**第三点：后台不刷新**。标签页在后台时，setInterval其实还在跑，白白消耗资源。配合refetchIntervalInBackground: false可以让页面不可见时不刷新。

**第四点：interval和block组合**。interval是轮询，block是事件触发。两者同时用，interval保证基本刷新，block在新区块产生时立即触发。

**第五点：自动清理**。组件卸载时useEffect的return会执行clearInterval。不需要手动写清理代码，React会自动处理。

### 约束条件要记住

**第一，interval必须大于0**。如果设为0或负数，setInterval会立即执行或者报错。代码里会检查这个。

**第二，太短会触发限流**。RPC节点对请求频率有限制。设成1秒基本会触发限流。建议最少5秒。

**第三，多个key全刷新**。传入多个key的时候，每个key都会被刷新。不是只刷新一个。

**第四，卸载时清理**。组件从DOM移除时，clearInterval会被调用。不会有内存泄漏。

### 状态管理要清楚

触发条件是setInterval定时器。每隔指定的毫秒数，回调函数就会执行一次。回调里遍历所有key，对每个key调用invalidateQueries触发刷新。

清理是useEffect的return。每次interval变化或者组件卸载时，先清除旧的interval，再设置新的。

### 常见错误要避开

**第一个坑：间隔选择**。余额设10秒左右就够了，实时性要求高的设3-5秒。不要设1秒，肯定会被RPC限流。

**第二个坑：后台刷新浪费**。setInterval在页面后台时还会继续跑，这会无谓消耗资源。最好配合visibility API或者设置refetchIntervalInBackground: false。

**第三个坑：CPU占用**。即使页面不可见，setInterval也在执行。如果你的应用需要很多这种定时器，CPU占用会比较高。

**第四个坑：Strict Mode的影响**。React 18的开发模式会故意执行两次useEffect。这会导致interval被创建两次又清理两次。这是开发模式的特性，生产环境不会有这个问题。

**第五个坑：时间漂移**。setInterval不保证精确时间。如果系统繁忙，可能下一次会稍微延迟。长时间运行后，误差会累积。不过一般应用场景不需要校正。

### 提示词模板

```markdown
帮我创建一个基于时间间隔刷新数据的hook。

需求：
- 每隔一定时间自动刷新数据
- 支持自定义间隔（毫秒）
- 支持单个或多个数据同时刷新
- 组件卸载时自动清理

技术栈：
- React + TypeScript
- @tanstack/react-query

输入：
- interval：刷新间隔（毫秒）
- key：单个查询key
- keys：多个查询key数组

使用示例：
- 余额数据：10秒刷新
- 价格数据：5秒刷新
- 多个数据：同时刷新

实现要点：
1. 用setInterval定时触发
2. 遍历所有key调用invalidateQueries
3. useEffect返回clearInterval清理
4. 组件卸载时自动停止

注意：
- 间隔不能太短，建议>=5秒
- 后台标签页可能需要暂停
- 多个key全都会刷新
- React 18开发模式会执行两次
```

### 实际避坑指南

第一个，合适的间隔。余额10秒，价格5秒，实时性要求很高的3秒。不要设1秒，肯定会被RPC限流。

第二个，后台暂停。如果不希望后台标签页继续刷新，可以用document.hidden判断，在页面不可见时暂停interval。

第三个，组合使用。interval和useWatchByBlock可以同时用。interval保证基本刷新间隔，block在新区块时立即触发。两者的刷新逻辑要区分开，避免重复刷新。

第四个，不要滥用。不是所有数据都需要定时刷新。静态配置数据根本不需要。开启定时刷新的应该是真正需要实时性的数据。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useWatchByInterval` 用于定时刷新数据。基本用法如下：

```typescript
import { useWatchByInterval } from '@sushiswap/wag'

function BalanceComponent({ account }) {
  const { data: balance } = useBalance({
    address: account,
    query: {
      queryKey: ['balance', account],
      refetchInterval: false, // 禁用自动刷新，由 useWatchByInterval 控制
    },
  })

  // 每 10 秒刷新一次余额
  useWatchByInterval({
    key: ['balance', account],
    interval: 10000, // 10 秒
  })

  return <div>余额: {balance?.toString()}</div>
}
```

### 5.2 常见使用场景

#### 场景一：余额定时刷新

```typescript
function WalletBalances({ account, tokens }) {
  const { data: balances } = useBalances({
    account,
    currencies: tokens,
    query: {
      queryKey: ['balances', account],
      refetchInterval: false, // 禁用内置刷新
    },
  })

  // 每 10 秒刷新
  useWatchByInterval({
    key: ['balances', account],
    interval: 10000,
  })

  return <BalanceList balances={balances} />
}
```

#### 场景二：价格定时刷新

```typescript
function TokenPrice({ token }) {
  const { data: price } = usePrice({
    token,
    query: {
      queryKey: ['price', token.address],
      refetchInterval: false,
    },
  })

  // 价格需要更频繁刷新（5 秒）
  useWatchByInterval({
    key: ['price', token.address],
    interval: 5000,
  })

  return <div>价格: ${price?.toSignificant(6)}</div>
}
```

#### 场景三：组合 useWatchByBlock

```typescript
function RealTimeData({ pool }) {
  const { data: reserves } = usePoolReserves({
    pool,
    query: {
      queryKey: ['pool-reserves', pool.address],
      refetchInterval: false,
    },
  })

  // interval 作为兜底，block 作为实时补充
  useWatchByInterval({
    key: ['pool-reserves', pool.address],
    interval: 30000, // 30 秒轮询
  })

  useWatchByBlock({
    chainId,
    key: ['pool-reserves', pool.address],
    modulo: 3, // 每 3 个区块刷新
  })

  return <ReserveDisplay reserves={reserves} />
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **选择合适的间隔**
   ```typescript
   // 余额等相对静态的数据
   interval: 10000 // 10 秒

   // 价格等动态数据
   interval: 5000 // 5 秒

   // 非常动态的数据
   interval: 3000 // 3 秒
   ```

2. **在组件卸载时停止刷新**
   ```typescript
   // useWatchByInterval 会自动清理
   // 但如果需要在后台暂停，可以结合 visibility API
   useEffect(() => {
     if (document.hidden) return // 标签页不可见时不刷新
     // ...
   }, [])
   ```

3. **禁用 query 的内置刷新**
   ```typescript
   // 避免双重刷新
   const { data } = useQuery({
     queryKey,
     refetchInterval: false, // 禁用内置刷新
   })

   useWatchByInterval({ key: queryKey, interval: 10000 })
   ```

#### ❌ Don'ts

1. **不要使用过短的间隔**
   ```typescript
   // 错误：1 秒间隔太短
   interval: 1000

   // 正确：至少 3-5 秒
   interval: 5000
   ```

2. **不要对不需要刷新的数据使用**
   ```typescript
   // 错误：静态配置不需要定时刷新
   useWatchByInterval({ key: ['config'], interval: 1000 })

   // 正确：只在需要实时数据时使用
   ```

3. **不要忘记禁用 query 的内置刷新**
   ```typescript
   // 可能导致双重刷新
   const { data } = useQuery({
     queryKey,
     refetchInterval: 10000, // 内置刷新
   })

   useWatchByInterval({
     key: queryKey,
     interval: 10000, // 重复刷新
   })

   // 正确：禁用其中一个
   const { data } = useQuery({
     queryKey,
     refetchInterval: false, // 禁用内置
   })
   ```

4. **不要在生产环境使用过长的 key 列表**
   ```typescript
   // 错误：每个 token 都单独监听
   tokens.map(token => useWatchByInterval({
     key: ['balance', token.address],
     interval: 10000,
   }))

   // 正确：批量刷新
   useWatchByInterval({
     keys: tokens.map(t => ['balance', t.address]),
     interval: 10000,
   })
   ```
