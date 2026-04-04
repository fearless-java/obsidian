> 源代码路径: `apps/web/src/app/(networks)/(non-evm)/stellar/_common/lib/hooks/trustline/use-trustline.ts`

# useTrustline Hook Tutorial

## 大白话讲讲这个hook的作用

`useTrustline` *(一个React hook集合，用于管理Stellar网络的信任线，包括检查、创建信任线和动态查找issuer)* 是一个用于管理 Stellar 信任线的 hook 集合。Stellar 上的非原生资产需要建立信任线才能持有。包含：

- **useHasTrustline**：检查用户是否有特定资产的信任线
- **useUserTrustlines**：获取用户所有信任线
- **useCreateTrustline**：创建信任线
- **useNeedsTrustline**：检查是否需要信任线
- **useNeedsTrustlines**：批量检查多个代币

## 讲讲为什么需要封装该hook

封装这些 hooks 的原因：

1. **信任线必要性**：Stellar 的 Soroban 合约代币需要信任线
2. **动态 issuer 查找**：可以从 Horizon *(Stellar的API服务节点，用于查询账户和交易信息)* 动态查找 issuer
3. **批量检查**：支持批量检查多个代币的信任线状态

## 讲讲该hook的执行逻辑和数据流向

### useHasTrustline

**输入：**
```typescript
assetCode: string              // 资产代码
assetIssuer: string            // 资产发行者
```

**输出：**
```typescript
{
  data: boolean                // true = 有信任线
}
```

### useCreateTrustline

**输入：**
```typescript
{
  assetCode: string
  assetIssuer: string
  limit?: string              // 可选的信任额度
}
```

**输出：**
```typescript
{
  mutate: () => void
  isPending: boolean
}
```

### useNeedsTrustline

**输入：**
```typescript
assetCode: string
assetContract: string
assetIssuer?: string
```

**输出：**
```typescript
{
  needsTrustline: boolean     // 是否需要创建信任线
  hasTrustline: boolean      // 是否已有信任线
  issuer: string | null      // 发行者地址
}
```

### 核心执行逻辑

1. **检查信任线**：调用 `hasTrustline` *(检查信任线是否存在的合约方法)* 检查
2. **动态查找**：如果 issuer 未知，从 Horizon 查找
3. **创建信任线**：调用 `createTrustline` *(创建信任线的合约方法)* 创建

## 四、用这个hook来教你写AI提示词

好，现在我们来聊聊怎么用这个 hook 作为一个例子，让AI帮你写出类似功能的代码。这其实是一种很好的学习方式——通过已经理解的东西来理解新的东西。

### 怎么组织你的问题

当你让AI帮你写一个管理信任线的hook集合时，你可以这样描述你的需求：

首先告诉AI你想要什么：我想创建一个类似useTrustline的hook集合，用来管理某个区块链上的信任线。Stellar上的非原生资产需要建立信任线才能持有，这个hook就是管理这个的。然后明确几个关键点。第一，useHasTrustline检查用户是否有特定资产的信任线。第二，useCreateTrustline创建信任线。第三，useNeedsTrustline检查是否需要信任线。第四，useNeedsTrustlines批量检查多个代币。第五，支持动态issuer查找，因为某些代币的issuer可能需要从别的地方查。第六，提供Toast通知。第七，创建信任线后要invalidate相关查询。第八，用React Query管理数据。

### 这里面有几个地方特别容易出错

信任线机制是Stellar特有的，和其他区块链不一样，需要信任资产的发行者。某些代币的issuer可能不知道在哪，需要动态从Horizon查找。某些合约包装的经典资产可能不需要信任线，比如SAC代币，这种情况要特殊处理。

### 数据刷新这里有讲究

信任线检查可以设较长的staleTime，因为信任线状态变化不频繁。创建信任线后要立即invalidate trustline相关查询，让UI能刷新。这个hook集合涉及Stellar的特殊机制，做好了能大大简化用户操作。

## 5. 该 hook 的用法教学

### 基础用法

```tsx
import {
  useHasTrustline,
  useCreateTrustline,
  useNeedsTrustline,
} from '@sushiswap/hooks'

function TrustlineManager({ token }: { token: Token }) {
  const { address } = useStellarWallet()

  // 检查是否有信任线
  const { data: hasTrustline } = useHasTrustline({
    assetCode: token.symbol,
    assetIssuer: token.issuer,
  })

  // 创建信任线
  const { mutate: createTrustline, isPending } = useCreateTrustline({
    onSuccess: () => {
      toast.success('信任线创建成功！')
      queryClient.invalidateQueries(['trustline'])
    }
  })

  if (hasTrustline) {
    return <p>已有信任线</p>
  }

  return (
    <button onClick={() => createTrustline({
      assetCode: token.symbol,
      assetIssuer: token.issuer,
    })} disabled={isPending}>
      {isPending ? '创建中...' : '创建信任线'}
    </button>
  )
}
```

### 常见使用场景

1. **代币持有前检查**：检查用户是否能持有某代币
   ```tsx
   const { needsTrustline } = useNeedsTrustline({
     assetCode: token.symbol,
     assetContract: token.contract,
   })

   if (needsTrustline) {
     return <TrustlinePrompt />
   }
   ```

2. **批量检查多个代币**：检查用户是否需要为多个代币创建信任线
   ```tsx
   const { data: trustlineStatus } = useNeedsTrustlines({
     assetCodes: tokens.map(t => t.symbol),
     assetContracts: tokens.map(t => t.contract),
   })
   ```

3. **信任线引导**：引导用户创建必要的信任线
   ```tsx
   const missingTrustlines = tokens.filter(t => !trustlineStatus[t.contract])

   if (missingTrustlines.length > 0) {
     return (
       <div>
         <p>请先创建以下信任线：</p>
         {missingTrustlines.map(t => (
           <CreateTrustlineButton key={t.contract} token={t} />
         ))}
       </div>
     )
   }
   ```

### Dos and Don'ts

**Dos:**
- ✅ 在用户尝试持有代币前检查信任线
- ✅ 使用 `useNeedsTrustline` 简化检查逻辑
- ✅ 创建后刷新信任线状态

**Don'ts:**
- ❌ 不要假设所有代币都需要信任线（SAC 代币可能不需要）
- ❌ 不要忽略 issuer 查找
- ❌ 不要在 isPending 时允许重复提交
