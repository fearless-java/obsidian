> 源代码路径: `apps/web/src/lib/wagmi/hooks/pools/hooks/useSushiSwapV2Pools.ts`

# useSushiSwapV2Pools

## 大白话讲讲这个hook的作用

`useSushiSwapV2Pools` 是一个批量查询 SushiSwap V2 池子准备金的 Hook。

大白话：就是"查询这些交易对的 V2 池子里各有多少钱"。V2 的池子是标准的恒定乘积池 (x*y=k)，查询结果包括两种代币的实时数量。

它使用 `computeSushiSwapV2PoolAddress` 计算池子地址，然后用 `getReserves` 获取准备金数据。

## 讲讲为什么需要封装该hook

1. **批量查询**：使用 `useReadContracts` 一次查询多个池子。

2. **地址计算**：封装了池子地址计算逻辑。

3. **缓存优化**：React Query 缓存结果。

4. **区块监听**：新区块时自动刷新数据。

5. **状态管理**：封装了池子存在性判断（EXISTS/NOT_EXISTS/LOADING）。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseSushiSwapV2PoolsParams {
  chainId: SushiSwapV2ChainId | undefined
  currencies: [EvmCurrency | undefined, EvmCurrency | undefined][]
  config?: Omit<UseReadContractsParameters, 'contracts'>
}
```

### 池子状态枚举

```typescript
enum SushiSwapV2PoolState {
  LOADING = 'Loading',
  NOT_EXISTS = 'Not Exists',
  EXISTS = 'Exists',
  INVALID = 'Invalid',
}
```

### 输出 (Outputs)

```typescript
interface UseSushiSwapV2PoolsReturn {
  isLoading: boolean
  isError: boolean
  data: [SushiSwapV2PoolState, SushiSwapV2Pool | null][]
}
```

### 执行逻辑详解

#### 1. 计算池子地址

```typescript
function getSushiSwapV2Pools(chainId, currencies) {
  // 过滤无效对
  const filtered = currencies.filter(
    (currencies) => Boolean(
      currencyA && currencyB &&
      currencyA.chainId === currencyB.chainId &&
      !currencyA.wrap().isSame(currencyB.wrap()) &&
      isSushiSwapV2ChainId(currencyA.chainId)
    )
  )

  // 计算每个池子的地址
  const contracts = filtered.map(([currencyA, currencyB]) => ({
    address: computeSushiSwapV2PoolAddress({
      factoryAddress: SUSHISWAP_V2_FACTORY_ADDRESS[currencyA.chainId],
      tokenA: currencyA.wrap(),
      tokenB: currencyB.wrap(),
    }),
    abi: sushiSwapV2PairAbi_getReserves,
    functionName: 'getReserves',
  }))
}
```

#### 2. 批量查询

```typescript
const { data, isLoading, isError } = useReadContracts({
  contracts,
  query: {
    enabled: contracts.length > 0,
    select: (results) => results.map((r) => r.result),
  },
})
```

#### 3. 区块监听刷新

```typescript
const { data: blockNumber } = useBlockNumber({ chainId, watch: true })

useEffect(() => {
  if (blockNumber) {
    queryClient.invalidateQueries({ queryKey }, { cancelRefetch: false })
  }
}, [blockNumber, queryClient, queryKey])
```

#### 4. 数据转换

```typescript
return {
  data: data.map((result, i) => {
    if (!result) return [SushiSwapV2PoolState.NOT_EXISTS, null]

    const [reserve0, reserve1] = result
    const [token0, token1] = tokenA.sortsBefore(tokenB)
      ? [tokenA, tokenB]
      : [tokenB, tokenA]

    return [
      SushiSwapV2PoolState.EXISTS,
      new SushiSwapV2Pool(
        new Amount(token0, reserve0.toString()),
        new Amount(token1, reserve1.toString()),
      ),
    ]
  }),
}
```

### 数据流向图

```
输入: currencies: [[tokenA, tokenB], [tokenC, tokenD], ...]
         ↓
    computeSushiSwapV2PoolAddress 计算每个池子地址
         ↓
    useReadContracts 批量查询 getReserves
         ↓
    useBlockNumber 监听区块变化
         ↓
    invalidateQueries 触发刷新
         ↓
    转换结果: [state, SushiSwapV2Pool | null]
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：过滤无效交易对**。在查询之前，代码会过滤掉那些明显无效的交易对。比如两个相同的代币不能组成交易对，不同链上的代币也不能配对，无效的地址也会被过滤掉。这些无效数据如果不过滤，查询会失败或者返回错误结果。

**第二点：批次大小要控制**。虽然这个hook支持批量查询多个池子，但不能一次查太多。以太坊的批量调用有gas和数据大小限制，查太多会失败。建议一次查不超过20个池子，如果需要查更多就分批。

**第三点：缓存key要包含所有变量**。查询的缓存key必须包含所有会影响查询结果的变量，比如chainId和代币地址。如果key设计不对，缓存可能会返回错误的数据。

**第四点：LOADING状态要处理**。查询进行中的时候，状态是LOADING。UI要根据这个状态显示加载中，不能假设数据已经准备好了。

**第五点：代币要按地址排序**。V2池子合约里token0和token1是按地址排序存储的。你的代码也要按同样的规则排序，确保查询时用的顺序和合约内部存储的顺序一致。

### 约束条件要记住

**第一，只能查同链代币**。两个代币必须在同一条链上。如果一个是以太坊上的USDC，一个是Polygon上的USDC，不能配对成一个池子。

**第二，不能是同一个币**。交易对的两边必须是不同的代币。不能查USDC/USDC这样的组合。

**第三，批量查询有上限**。虽然代码可以接收很多交易对作为参数，但实际能成功查询的数量有限制以太坊的批量调用有gas限制。

**第四，池子可能不存在**。不是所有代币对都有对应的池子。如果没有池子，状态会返回NOT_EXISTS而不是报错。

### 状态管理要清楚

池子状态有四种。LOADING表示正在查询，NOT_EXISTS表示没有池子，EXISTS表示池子存在并返回了数据，INVALID表示输入无效。

返回数据是一个数组，每个元素对应一个交易对。数组里每个元素又是一个元组，第一个是状态，第二个是池子数据或者null。

刷新方式有两种。一种是新区块产生时自动刷新，另一种是通过refetchInterval配置定时刷新。

### 常见错误要避开

**第一个坑：Factory地址要对应**。不同链上的Factory合约地址不同。代码里有个映射表，根据chainId查找对应的Factory地址。你要确保传入的chainId是正确的。

**第二个坑：排序要一致**。合约内部按代币地址排序存储token0和token1。你的代码也必须按同样的规则排序，否则查到的结果可能不对。地址小的排前面。

**第三个坑：批量有上限**。不是想查多少就能查多少。以太坊的批量调用有参数大小限制。超过一定数量交易会失败。如果需要查很多池子，要分批次。

**第四个坑：NOT_EXISTS不等于错误**。如果一个交易对没有池子，返回的状态是NOT_EXISTS，不是错误。这是正常情况，调用方检查状态即可。

**第五个坑：小数位处理**。不同代币有不同的小数位数。Amount类型会处理这个转换，不用担心精度问题。

### 提示词模板

```markdown
帮我创建一个批量查询SushiSwap V2池子准备金的hook。

需求：
- 一次查询多个池子的准备金数据
- 计算池子地址
- 批量调用getReserves
- 新区块产生时自动刷新

技术栈：
- React + TypeScript
- wagmi v2 + viem
- @tanstack/react-query v5
- sushi库

池子状态：
- LOADING：正在查询
- NOT_EXISTS：池子不存在
- EXISTS：池子存在
- INVALID：输入无效

输入：
- chainId：链ID
- currencies：代币对数组，如[[tokenA, tokenB], [tokenC, tokenD]]

输出：
- isLoading：是否加载中
- isError：是否出错
- data：数组，每个元素是[状态, 池子数据]

实现要点：
1. 计算每个交易对的池子地址
2. 批量调用getReserves
3. 按地址排序代币确保一致
4. 过滤无效交易对
5. 监听新区块刷新数据

注意：
- 代币对必须在同一链上
- 不能是同一个代币
- 批量查询有数量限制
- 池子不存在是正常情况
```

### 实际避坑指南

第一个，Factory地址映射表。以太坊、Polygon、BSC等链上的Factory地址都不同。代码里有映射表，根据chainId自动查找对应的地址。

第二个，分批查询。如果需要查超过20个池子，分成多批来查。每批之间可以稍微延迟一下，防止触发RPC限制。

第三个，排序的重要性。传入的代币对要按地址排序。如果你自己写排序逻辑，确保排序规则和合约一致：地址小的在前。

第四个，空池子正常处理。NOT_EXISTS状态很常见，不代表出错了。用户看到"暂无池子"的提示是正常的。

3. **批量限制**：单次 batch 调用有 gas 限制，太多池子会失败。

4. **NOT_EXISTS 判断**：读取失败的 result 不一定是池子不存在，也可能是 RPC 问题。

5. **小数位处理**：不同代币有不同 decimals，需要在 Amount 中正确处理。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useSushiSwapV2Pools` 用于批量查询 V2 池子准备金。基本用法如下：

```typescript
import { useSushiSwapV2Pools, SushiSwapV2PoolState } from '@sushiswap/wag'

function PoolList({ pairs }) {
  const { data: pools, isLoading } = useSushiSwapV2Pools({
    chainId,
    currencies: pairs, // [[tokenA, tokenB], [tokenC, tokenD], ...]
  })

  if (isLoading) return <div>加载中...</div>

  return (
    <div>
      {pools?.map(([state, pool], i) => (
        <div key={i}>
          {state === SushiSwapV2PoolState.EXISTS ? (
            <div>池子: {pool.reserve0.toSignificant(6)}</div>
          ) : (
            <div>池子不存在</div>
          )}
        </div>
      ))}
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景一：查询交易对的准备金

```typescript
function SwapPoolInfo({ tokenA, tokenB }) {
  const { data: pools } = useSushiSwapV2Pools({
    chainId,
    currencies: [[tokenA, tokenB]],
  })

  const [state, pool] = pools?.[0] || []

  if (state !== SushiSwapV2PoolState.EXISTS) {
    return <div>无池子</div>
  }

  return (
    <div>
      <div>准备金 {tokenA.symbol}: {pool.reserve0.toSignificant(6)}</div>
      <div>准备金 {tokenB.symbol}: {pool.reserve1.toSignificant(6)}</div>
    </div>
  )
}
```

#### 场景二：批量查询多个交易对

```typescript
function MultiplePools({ tokenList }) {
  // 创建所有可能的交易对
  const pairs = tokenList.flatMap((tokenA, i) =>
    tokenList.slice(i + 1).map(tokenB => [tokenA, tokenB] as [Currency, Currency])
  )

  const { data: pools } = useSushiSwapV2Pools({
    chainId,
    currencies: pairs,
  })

  return (
    <div>
      {pairs.map(([tokenA, tokenB], i) => {
        const [state, pool] = pools?.[i] || []
        return (
          <div key={`${tokenA.address}-${tokenB.address}`}>
            {tokenA.symbol}/{tokenB.symbol}:
            {state === SushiSwapV2PoolState.EXISTS
              ? pool.reserve0.toSignificant(4)
              : '无池子'}
          </div>
        )
      })}
    </div>
  )
}
```

#### 场景三：获取最优交易池

```typescript
function BestPool({ tokenA, tokenB }) {
  const { data: pools } = useSushiSwapV2Pools({
    chainId,
    currencies: [[tokenA, tokenB]],
  })

  const [state, pool] = pools?.[0] || []

  if (state !== SushiSwapV2PoolState.EXISTS) {
    return <div>无池子</div>
  }

  // 计算价格
  const price = pool.reserve1.toBigInt() * BigInt(1e18) / pool.reserve0.toBigInt()

  return (
    <div>
      <div>价格: 1 {tokenA.symbol} = {price.toString()} {tokenB.symbol}</div>
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **检查池子状态**
   ```typescript
   const [state, pool] = pools?.[0] || []

   if (state !== SushiSwapV2PoolState.EXISTS) {
     return <NoPool />
   }
   ```

2. **过滤无效交易对**
   ```typescript
   // useSushiSwapV2Pools 内部会过滤
   // 但传入之前确保格式正确
   currencies: [[tokenA, tokenB]] // 两个代币的数组
   ```

3. **批量查询注意限制**
   ```typescript
   // 一次不要查询太多
   // 建议不超过 20 个
   currencies: pairs.slice(0, 20)
   ```

#### ❌ Don'ts

1. **不要传入相同的代币**
   ```typescript
   // 错误
   currencies: [[USDC, USDC]]

   // 正确
   currencies: [[USDC, WETH]]
   ```

2. **不要跨链查询**
   ```typescript
   // 错误：两个代币必须在同一链
   currencies: [[ETH_ON_ETHEREUM, MATIC_ON_POLYGON]]

   // 正确：同链代币
   currencies: [[ETH, USDC]] // 都在以太坊
   ```

3. **不要假设池子一定存在**
   ```typescript
   // 错误
   const reserve = pools[0][1].reserve0

   // 正确：先检查状态
   if (pools[0][0] !== SushiSwapV2PoolState.EXISTS) return
   ```

4. **不要忽略 isLoading**
   ```typescript
   if (isLoading) return <Skeleton />
   ```
