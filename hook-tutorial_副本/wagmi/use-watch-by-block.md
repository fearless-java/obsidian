> 源代码路径: `apps/web/src/lib/wagmi/hooks/watch/useWatchByBlock.ts`

# useWatchByBlock

## 大白话讲讲这个hook的作用

`useWatchByBlock` 是一个基于区块变化来刷新 React Query 缓存数据的 Hook。

大白话：就是"每当新区块产生时，自动刷新这些数据"。它可以帮助你保持数据的实时性，而不需要用户手动刷新页面。

你传入一个或多个 query key，每当指定的区块数产生时（可配置 modulo），就会自动 invalidate 这些查询，触发数据重新获取。

## 讲讲为什么需要封装该hook

1. **数据新鲜度**：对于需要实时更新的数据（如余额、价格），区块变化时刷新很有用。

2. **Modulo 选项**：可以设置"每 N 个区块刷新一次"，避免过于频繁。

3. **批量刷新**：可以同时监听多个 query key。

4. **避免重复设置**：不需要在每个需要刷新的组件里单独写 useBlockNumber + useEffect。

5. **灵活控制**：支持单个 key 或多个 key，支持 modulo 控制频率。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
type UseWatchByBlockKey = {
  key: QueryKey        // 单个 query key
  keys?: never
}

type UseWatchByBlockKeys = {
  key?: never
  keys: QueryKey[]     // 多个 query keys
}

type UseWatchByBlock = {
  chainId: EvmChainId | undefined
  modulo?: number       // 每多少个区块刷新一次
} & (UseWatchByBlockKey | UseWatchByBlockKeys)
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

#### 2. 监听区块

```typescript
const { data: blockNumber } = useBlockNumber({ chainId, watch: true })
```

#### 3. 区块变化时刷新

```typescript
useEffect(() => {
  if (!blockNumber || !keys.length) return

  // 如果设置了 modulo，只有 blockNumber % modulo === 0 时才刷新
  if (modulo && blockNumber % BigInt(modulo) !== 0n) return

  keys.forEach((key) => {
    queryClient.invalidateQueries({ queryKey: key })
  })
}, [blockNumber, keys, queryClient, modulo])
```

### 数据流向图

```
输入: key(s), chainId, modulo
         ↓
    useBlockNumber 监听区块
         ↓
    区块变化时检查:
    ┌────────────────────────────────────┐
    │  modulo 检查:                      │
    │  blockNumber % modulo === 0 ?      │
    └────────────────────────────────────┘
         ↓
    keys.forEach invalidateQueries
         ↓
    React Query 重新获取数据
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：不要滥用**。区块监听是通过websocket订阅实现的，每个订阅都会持续占用资源。如果对所有数据都设置区块监听，RPC负担会很大。只对真正需要实时的数据开启，比如余额、持仓等。

**第二点：modulo减少频率**。不是所有数据都需要每分钟刷新很多次。用modulo可以设置每几个区块才刷新一次。比如modulo设为10，就是每10个区块（约2分钟）才刷新一次。这能大大减少RPC调用。

**第三点：key格式灵活**。可以传入完整的queryKey数组，也可以只传queryKey属性。两种方式效果一样，代码会自动规范化处理。

**第四点：和interval组合用**。interval是基于时间的轮询，block是基于新区块的监听。两者可以同时用，interval作为兜底，block保证实时性。

**第五点：自动清理**。useEffect会在组件卸载时自动清理副作用。不需要手动写清理逻辑，框架已经帮你处理好了。

### 约束条件要记住

**第一，需要有效的chainId**。必须传入正确的链ID才能建立正确的订阅。如果chainId无效，监听可能失败。

**第二，区块监听增加RPC负担**。websocket订阅虽然比轮询省资源，但仍然会增加节点负担。要节制使用。

**第三，modulo必须大于0**。如果modulo设为0或负数，逻辑会出错。代码里会检查这个条件。

**第四，多个key是AND关系**。如果你传了多个key，那每个key都会刷新。不是只刷新其中一个。

### 状态管理要清楚

触发条件有两个。第一是区块号必须变化，监听的是新区块产生事件。第二是modulo检查通过，比如你设的modulo是10，那就每10个区块才真正触发刷新。

操作是调用queryClient.invalidateQueries，参数是queryKey数组。这会让React Query重新获取这些查询的数据。

### 常见错误要避开

**第一个坑：BigInt计算modulo**。modulo计算用的是BigInt，不是普通数字。因为区块号是非常大的数字，普通数字可能会溢出。

**第二个坑：空key列表**。如果传入的keys是空数组，invalidate什么都不会发生。代码里有检查，keys为空时直接return，不做无效操作。

**第三个坑：切换链的问题**。用户切换链后，websocket订阅需要重新建立。wagmi会处理这个，但如果你的queryKey没有包含chainId，可能会缓存错乱。

**第四个坑：性能影响**。watch: true会建立websocket订阅。如果订阅太多，内存和性能都会受影响。长期开着不用的话记得关闭。

**第五个坑：cancelRefetch配置**。invalidateQueries时可以配置cancelRefetch。如果设为true，会取消正在进行的请求。如果不想取消，设为false。

### 提示词模板

```markdown
帮我创建一个基于区块变化刷新数据的hook。

需求：
- 监听新区块产生
- 自动刷新指定的数据
- 可以设置每几个区块刷新一次
- 支持单个或多个数据同时刷新

技术栈：
- React + TypeScript
- wagmi v2
- @tanstack/react-query

输入：
- chainId：链ID
- key：单个查询key
- keys：多个查询key数组
- modulo：每多少个区块刷新一次

使用方式：
- 传入key刷新单个数据
- 传入keys刷新多个数据
- 设置modulo减少刷新频率

实现要点：
1. 用useBlockNumber监听区块，watch设为true
2. 规范化key格式，支持单个或多个
3. modulo检查通过才刷新
4. 调用invalidateQueries重新获取数据
5. useEffect自动清理副作用

注意：
- 不要对所有数据都用，会增加负担
- modulo设为1就是每个区块都刷新
- keys为空时不执行
- 切换链要重新监听
```

### 实际避坑指南

第一个，合理使用modulo。如果数据不需要秒级更新，可以设modulo大一点。比如10，约2分钟刷新一次就够了。

第二个，避免重复订阅。如果多个组件都在监听同一个数据源的区块变化，会建立多个订阅。最好在父组件统一管理。

第三个，清理问题。组件卸载时订阅会自动清理。但如果在组件内部多次调用这个hook（比如依赖变化时重新设置），要确保前一个订阅被清理了。

第四个，和interval的区别。interval是固定时间轮询，不管有没有新区块。block是事件驱动，有新区块才触发。两者可以组合用，block保证实时性，interval作为备用。

2. **空 Keys**：keys 为空数组时不执行 invalidate，避免无效操作。

3. **ChainId 变化**：切换链时需要重新监听，wagmi 会自动处理。

4. **性能影响**：`watch: true` 会建立 websocket 订阅，长期开会有性能影响。

5. **Cancel Refetch**：可以配置 `cancelRefetch: false` 避免取消正在进行的请求。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useWatchByBlock` 用于在区块变化时刷新数据。基本用法如下：

```typescript
import { useWatchByBlock } from '@sushiswap/wag'

function BalanceComponent({ account }) {
  const { data: balance } = useBalance({
    address: account,
    query: {
      queryKey: ['balance', account],
    },
  })

  // 当新区块产生时自动刷新余额
  useWatchByBlock({
    chainId,
    key: ['balance', account],
  })

  return <div>余额: {balance?.toString()}</div>
}
```

### 5.2 常见使用场景

#### 场景一：刷新账户余额

```typescript
function AccountBalances({ account, tokens }) {
  const { data: balances } = useBalances({
    account,
    currencies: tokens,
    query: {
      queryKey: ['balances', account, tokens.map(t => t.address)],
    },
  })

  // 区块变化时刷新余额
  useWatchByBlock({
    chainId,
    key: ['balances', account],
  })

  return <BalanceList balances={balances} />
}
```

#### 场景二：刷新池子准备金

```typescript
function PoolReserves({ pool }) {
  const { data: reserves } = useSushiSwapV2Pool({
    pair: [tokenA, tokenB],
    query: {
      queryKey: ['pool', tokenA.address, tokenB.address],
    },
  })

  // 每 5 个区块刷新一次（不要太频繁）
  useWatchByBlock({
    chainId,
    key: ['pool', tokenA.address, tokenB.address],
    modulo: 5,
  })

  return <ReserveDisplay reserves={reserves} />
}
```

#### 场景三：批量刷新多个查询

```typescript
function Dashboard() {
  const queryClient = useQueryClient()

  // 同时刷新多个相关数据
  useWatchByBlock({
    chainId,
    keys: [
      ['balances', account],
      ['pools', account],
      ['positions', account],
    ],
  })

  return <DashboardContent />
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **使用 modulo 减少刷新频率**
   ```typescript
   // 不是所有数据都需要实时刷新
   useWatchByBlock({
     chainId,
     key: queryKey,
     modulo: 10, // 每 10 个区块刷新一次
   })
   ```

2. **组合 useWatchByInterval 使用**
   ```typescript
   // useWatchByBlock 处理新区块
   // useWatchByInterval 处理轮询
   // 两者可以同时使用
   useWatchByBlock({ chainId, key: queryKey })
   useWatchByInterval({ key: queryKey, interval: 30000 })
   ```

3. **只对需要实时的数据使用**
   ```typescript
   // 适合：余额、价格、持仓等需要实时更新的数据
   // 不适合：历史数据、静态配置等
   ```

#### ❌ Don'ts

1. **不要对太多数据使用**
   ```typescript
   // 错误：每个组件都监听区块
   function Component1() { useWatchByBlock({ key: ['a'] }) }
   function Component2() { useWatchByBlock({ key: ['b'] }) }
   function Component3() { useWatchByBlock({ key: ['c'] }) }

   // 正确：只对需要的数据使用
   // 考虑在父组件统一管理
   ```

2. **不要忘记传入 chainId**
   ```typescript
   // 错误
   useWatchByBlock({ key: queryKey }) // chainId 缺失

   // 正确
   useWatchByBlock({ chainId, key: queryKey })
   ```

3. **不要使用过短的 modulo**
   ```typescript
   // 错误：每个区块都刷新，太频繁
   modulo: 1

   // 正确：至少几个区块
   modulo: 5 // 或更长
   ```

4. **不要在后台标签页持续消耗资源**
   ```typescript
   // websocket 订阅在后台标签页仍会运行
   // 如果不需要实时数据，考虑用 useWatchByInterval 替代
   // 并设置 refetchIntervalInBackground: false
   ```
