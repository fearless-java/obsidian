> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/liquidity/use-add-liquidity.ts`

# useAddLiquidity Hook Tutorial

## 大白话讲讲这个hook的作用

`useAddLiquidity` *(一个React hook，用于向流动性池添加流动性，封装了完整的添加交易流程包括签名和通知)* 是一个用于向流动性池添加流动性的 mutation hook。它：

- 将 Token0 和 Token1 添加到指定池子
- 创建流动性头寸（position）
- 支持指定 tick 范围

## 讲讲为什么需要封装该hook

封装这个 hook 的原因：

1. **交易封装**：封装完整的添加流动性交易流程
2. **Toast 通知**：显示交易进度和结果
3. **缓存失效**：成功后刷新余额、池子信息、头寸数据

## 讲讲该hook的执行逻辑和数据流向

### 输入（参数）
```typescript
{
  userAddress: string                 // 用户地址
  poolAddress: string                // 池子地址
  token0Amount: string               // Token0 数量
  token1Amount: string              // Token1 数量
  token0Decimals: number            // Token0 小数位
  token1Decimals: number            // Token1 小数位
  tickLower: number                 // 价格范围下限
  tickUpper: number                 // 价格范围上限
  recipient?: string                // 接收地址，默认用户自己
  deadline?: number                 // 截止时间
  signTransaction: (xdr: string) => Promise<string>
  signAuthEntry: (entryPreimageXdr: string) => Promise<string>
}
```

### 输出（返回值）
```typescript
{
  mutate: () => void
  mutateAsync: () => Promise<{ result, params }>
  isPending: boolean
}
```

### 核心执行逻辑

1. **显示 info toast**：开始添加时显示
2. **构建参数**：转换金额、设置截止时间
3. **调用服务**：执行 `addLiquidity` *(添加流动性的合约方法)*
4. **显示 success toast**：成功后显示
5. **刷新缓存**：invalidate *(React Query方法，使查询缓存失效)* 相关查询

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个向流动性池添加流动性的hook时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useAddLiquidity的hook，用来向某个区块链上的流动性池添加流动性。然后明确几个关键点。第一，参数包括池子地址、两种代币的数量和精度、tick范围上下限、用户地址，还可以设置接收地址和截止时间。第二，封装添加流动性的合约调用，这个操作比较复杂。第三，支持签名函数，Stellar上有特殊的auth entry需要额外签名。第四，提供Toast通知，让用户知道操作进度和结果。第五，成功后要刷新相关查询，包括池子余额、池子信息和用户头寸，不然UI显示的还是旧数据。第六，用React Query的useMutation来处理。

### 这里面有几个地方特别容易出错

签名函数需要两个，signTransaction签主交易，signAuthEntry签授权条目，Stellar上的操作比较特殊，两个都不能少。交易要有截止时间，防止交易卡住无限制等待。输入的数量是人类能看懂的格式，要转换成最小单位才能传给合约。

### 数据刷新这里有讲究

用useMutation处理写操作，能知道当前是否在处理中。成功后要立即invalidate多个相关查询：pool balances、pool info、positions，这些数据都可能因为添加流动性而变化。这个hook是LP的核心操作之一，做好了用户就能顺利添加流动性获得收益。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import { useAddLiquidity } from '@sushiswap/hooks'
import { useStellarWallet } from './wallet'
import { parseUnits } from 'ethers'

function AddLiquidityForm({ pool }: { pool: PoolInfo }) {
  const { address, signTransaction, signAuthEntry } = useStellarWallet()
  const [amount0, setAmount0] = useState('')
  const [amount1, setAmount1] = useState('')
  const [tickLower, setTickLower] = useState(-1000)
  const [tickUpper, setTickUpper] = useState(1000)

  const { mutate: addLiquidity, isPending } = useAddLiquidity({
    onSuccess: () => {
      toast.success('流动性添加成功！')
      queryClient.invalidateQueries(['positions'])
      queryClient.invalidateQueries(['pool-balances'])
    },
    onError: (error) => {
      toast.error(`添加失败: ${error.message}`)
    }
  })

  const handleAdd = () => {
    addLiquidity({
      userAddress: address,
      poolAddress: pool.address,
      token0Amount: parseUnits(amount0, pool.token0.decimals).toString(),
      token1Amount: parseUnits(amount1, pool.token1.decimals).toString(),
      token0Decimals: pool.token0.decimals,
      token1Decimals: pool.token1.decimals,
      tickLower,
      tickUpper,
      signTransaction,
      signAuthEntry,
    })
  }

  return (
    <div>
      <input value={amount0} onChange={(e) => setAmount0(e.target.value)} placeholder="Token0 数量" />
      <input value={amount1} onChange={(e) => setAmount1(e.target.value)} placeholder="Token1 数量" />
      <button onClick={handleAdd} disabled={isPending}>
        {isPending ? '添加中...' : '添加流动性'}
      </button>
    </div>
  )
}
```

### 常见使用场景

1. **完整添加流动性流程**：带余额检查和授权
   ```tsx
   const { mutate: add } = useAddLiquidity({
     onSuccess: () => {
       queryClient.invalidateQueries(['positions'])
       queryClient.invalidateQueries(['balances'])
     }
   })
   ```

2. **添加后跳转**：添加成功后查看头寸
   ```tsx
   const { mutateAsync } = useAddLiquidity({
     onSuccess: (result) => result.tokenId
   })
   const { tokenId } = await mutateAsync(params)
   navigate(`/position/${tokenId}`)
   ```

3. **批量添加**：为多个头寸添加流动性
   ```tsx
   positions.forEach(pos => {
     add({ ...pos, token0Amount, token1Amount })
   })
   ```

### Dos and Don'ts

**Dos:**
- ✅ 使用 `parseUnits` *(ethers.js工具函数，将人类可读数量转换为最小单位)* 转换数量
- ✅ 检查余额和授权是否足够
- ✅ 成功后刷新相关查询

**Don'ts:**
- ❌ 不要忽略签名函数
- ❌ 不要使用人类可读格式的数量
- ❌ 不要在 isPending 时允许重复提交
