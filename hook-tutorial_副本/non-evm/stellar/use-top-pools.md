> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/pool/use-top-pools.ts`

# useTopPools Hook Tutorial

## 大白话讲讲这个hook的作用

`useTopPools` *(一个React hook，用于获取Stellar网络上热门流动性池列表，从SushiSwap数据API获取交易量最高的池子)* 是一个用于获取 Stellar 网络上热门流动性池列表的 hook。它从 SushiSwap 的数据 API 获取：

- 交易量最高的池子列表
- 每个池子的基本信息（代币对、地址、TVL、交易量等）

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **数据 API 封装**：封装 `@sushiswap/graph-client/data-api` *(SushiSwap的图数据API客户端)* 的调用
2. **代币信息补全**：从池子数据获取后，再通过 `getTokenByContract` *(通过合约地址获取代币详细信息)* 获取完整代币信息
3. **合并处理**：将池子数据与代币数据合并

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
- 无参数，使用默认配置

### 输出（返回值）
```typescript
{
  data: TopPool[]           // 热门池子列表
  isLoading: boolean
  isPending: boolean
}
```

### TopPool 结构
```typescript
{
  address: string           // 池子地址
  token0: Token            // Token0 信息
  token1: Token            // Token1 信息
  volumeUSD1d: number       // 日交易量
  totalApr1d: number        // 日收益率
  // ... 其他数据
}
```

### 核心执行逻辑

1. **获取热门池子**：调用 `getTopNonEvmPools({ chainId: ChainId.STELLAR })` *(SushiSwap数据API方法，获取指定链的热门池子)*
2. **获取代币信息**：对每个池子调用 `getTokenByContract` *(通过合约地址获取代币详细信息)* 获取代币详情
3. **合并返回**：返回带有完整代币信息的池子列表

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个获取热门流动性池列表的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useTopPools的hook，用来获取某个区块链网络上热门的流动性池列表。然后明确几个关键点。第一，从数据API获取热门池子列表，这个列表按交易量排序。第二，每个池子只有基础的代币地址，要调用getTokenByContract补全每个代币的详细信息，包括名称、符号、小数位数等。第三，把池子数据和代币数据合并在一起返回。第四，返回完整的热门池子列表。第五，用React Query管理数据。

### 这里面有几个地方特别容易出错

依赖代币数据，所以要等useCommonTokens的数据加载好了才能用这个hook。获取代币信息的时候要用Promise.all并行处理，不要一个个顺序来，这样太慢了。即使某些代币信息获取失败了，也要返回能返回的部分数据，不要因为一个代币查不到就把整个列表都丢了。

### 数据刷新这里有讲究

enabled条件要设成代币数据已加载才行，依赖的数据没准备好就不能发起查询。用isLoadingTokens和isPendingTokens来控制加载状态，这样能准确反映当前是不是在加载。这个hook涉及API调用和批量数据处理，逻辑比单纯一个查询要复杂一些。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useTopPools } from '@sushiswap/hooks'

function PoolList() {
  const { data: pools, isLoading } = useTopPools()

  if (isLoading) return <p>加载中...</p>

  return (
    <div>
      {pools?.map((pool) => (
        <PoolCard key={pool.address} pool={pool} />
      ))}
    </div>
  )
}
```

### 常见使用场景

1. **热门池子展示**：显示交易量最高的池子
   ```tsx
   const { data: topPools } = useTopPools()
   const sortedByVolume = topPools?.sort((a, b) => b.volumeUSD1d - a.volumeUSD1d)
   ```

2. **池子搜索**：从热门池子列表中搜索
   ```tsx
   const { data: pools } = useTopPools()
   const filtered = pools?.filter(p =>
     p.token0.symbol.includes(search) || p.token1.symbol.includes(search)
   )
   ```

3. **获取单个池子**：通过代币对获取池子
   ```tsx
   const pool = pools?.find(p =>
     (p.token0.address === tokenA && p.token1.address === tokenB) ||
     (p.token0.address === tokenB && p.token1.address === tokenA)
   )
   ```

### Dos and Don'ts

**Dos:**
- ✅ 使用 `keepPreviousData` 避免切换时闪烁
- ✅ 设置较长的 `staleTime` 减少 API 调用
- ✅ 与 `useCommonTokens` 配合获取代币信息

**Don'ts:**
- ❌ 不要直接修改返回的池子数据
- ❌ 不要假设池子列表总是存在
- ❌ 不要忽略加载状态
