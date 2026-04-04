> 源代码路径: `apps/web/src/lib/wagmi/hooks/master-chef/use-master-chef.ts`

# useMasterChef

## 大白话讲讲这个hook的作用

`useMasterChef` 是一个查询 MasterChef 质押池信息的 Hook。MasterChef 是 SushiSwap 的流动性挖矿合约，用户质押 LP 代币可以获得 SUSHI 奖励。

大白话：就是查询"我在哪个池子质押了多少，能领多少奖励"。

它会批量查询：
1. 用户在池子中的质押数量 (balance)
2. 待领取的 SUSHI 奖励 (pendingSushi)
3. SUSHI 在 MasterChef 合约中的余额

同时支持 MasterChef V1、V2 和 MiniChef 三种合约版本。

## 讲讲为什么需要封装该hook

1. **多版本兼容**：MasterChef 有 V1/V2/MiniChef 三个版本，接口略有不同，需要统一封装。

2. **批量查询**：使用 `useReadContracts` 一次查询多个数据，提高效率。

3. **自动刷新**：监听新区块自动刷新 pendingSushi（奖励会随着时间累积）。

4. **收割封装**：内置 `harvest` 函数，可以一键领取奖励。

5. **复杂合约交互**：不同版本的 harvest 逻辑不同（有的需要先从 V1 转），封装后统一接口。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
interface UseMasterChefParams {
  chainId: EvmChainId        // 链 ID
  chef: ChefType              // MasterChef 版本 (V1/V2/MiniChef)
  pid: number                 // Pool ID
  token: EvmToken | undefined  // 池子对应的 LP 代币
  enabled?: boolean
  watch?: boolean              // 是否监听区块变化
}
```

### ChefType 枚举

```typescript
enum ChefType {
  MasterChefV1 = 0,
  MasterChefV2 = 1,
  MiniChef = 2,
}
```

### 输出 (Outputs)

```typescript
interface UseMasterChefReturn {
  balance: Amount<EvmToken> | undefined      // 质押数量
  harvest: undefined | (() => void)          // 领取奖励函数
  pendingSushi: Amount<EvmToken> | undefined  // 待领取奖励
  isLoading: boolean
  isError: boolean
  isWritePending: boolean
  isWriteError: boolean
}
```

### 执行逻辑详解

#### 1. 获取合约配置

```typescript
const contract = useMasterChefContract(chainId, chef)
```

#### 2. 构建批量查询

```typescript
const contracts = useMemo(() => {
  if (!chainId || !enabled || !address) return []

  if (chainId === EvmChainId.ETHEREUM) {
    // V1/V2 查询
    return [
      // SUSHI 余额
      { abi: erc20Abi_balanceOf, functionName: 'balanceOf', args: [masterChefAddress] },
      // 用户质押信息
      { abi: userInfoAbi, functionName: 'userInfo', args: [pid, address] },
      // 待领取奖励
      { abi: pendingSushiAbi, functionName: 'pendingSushi', args: [pid, address] },
    ]
  }
  // MiniChef 查询...
}, [address, chainId, chef, enabled, pid])
```

#### 3. 监听区块变化

```typescript
const { data: blockNumber } = useBlockNumber({ chainId, watch: true })

useEffect(() => {
  if (watch && blockNumber) {
    queryClient.invalidateQueries({ queryKey }, { cancelRefetch: false })
  }
}, [blockNumber, watch, queryClient, queryKey])
```

#### 4. 构造 Harvest 交易

```typescript
const prepare = useMemo(() => {
  if (!address || !chainId || !data || !contract) return

  switch (chef) {
    case ChefType.MasterChefV1:
      return {
        to: contract.address,
        data: encodeFunctionData({
          abi: masterChefV1Abi_deposit,
          functionName: 'deposit',
          args: [BigInt(pid), 0n],  // 0n 表示只领取不解锁
        }),
      }
    case ChefType.MasterChefV2:
      // 可能需要 batch harvest
      if (pendingSushi > sushiBalance) {
        // 先从 V1 转过来再收割
      }
      return { /* harvest */ }
  }
}, [address, chainId, chef, contract, data, pendingSushi, sushiBalance])
```

### 数据流向图

```
输入: chainId, chef, pid, token
         ↓
    useMasterChefContract 获取合约
         ↓
    useReadContracts 批量查询
    ┌────────────────────────────────────┐
    │  1. SUSHI.balanceOf(masterChef)    │
    │  2. userInfo(pid, user)           │
    │  3. pendingSushi(pid, user)       │
    └────────────────────────────────────┘
         ↓
    转换: bigint → Amount<token>
         ↓
    返回 { balance, pendingSushi, harvest }
         ↑
    useBlockNumber 监听区块 → invalidateQueries
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：PID必须正确**。PID就是Pool ID，每个流动性池子都有一个唯一的编号。这个编号必须跟质押池配置对应上，错了就查不到正确的数据。PID不是随便设的，要从池子配置里读取。

**第二点：版本选择要对**。MasterChef有三个版本，V1和V2只在以太坊主网，MiniChef在其他链比如Polygon、BSC。传入的chef类型必须跟实际部署的合约版本一致。如果版本选错了，调用合约方法会失败。

**第三点：watch选项看场景**。如果需要实时显示待领取的SUSHI奖励，可以开启watch。这样每次新区块产生都会自动刷新数据。但这样做会增加RPC调用次数，所以如果不是实时显示的需求，可以不开启。

**第四点：链和代币要匹配**。传入的token和chainId必须跟池子部署的链一致。比如一个池子在Polygon上部署，你就不能用以太坊的chainId去查。数据会查不到或者查错。

**第五点：harvest要有意义才调用**。harvest是领取奖励的函数。只有当pendingSushi大于0的时候调用才有意义。如果pendingSushi是0，调用harvest也不会有任何效果。

### 约束条件要记住

**第一，V1/V2只在以太坊主网**。这两个版本的MasterChef合约只部署在以太坊主网上。其他链要用MiniChef。

**第二，MiniChef在其他链**。Polygon、BSC等链上运行的是MiniChef合约。接口和V1/V2略有不同。

**第三，PID是池子唯一标识**。每个池子都有一个PID，这是合约里分配的编号。不同的池子PID不同，必须查正确的PID才能拿到正确的数据。

**第四，pendingSushi需要watch才能实时**。待领取的SUSHI数量是随着区块不断增加的。如果不开启watch，数据不会自动刷新，用户看到的可能是比较旧的数据。

### 状态管理要清楚

查询状态主要看三个值。isLoading表示初始加载中，isError表示查询出错了，data是实际返回的数据，包含[sushiBalance余额, balance质押量, pendingSushi待领取]三个值。

刷新触发有两种方式。一种是watch开启时，每隔一个区块会自动invalidateQueries刷新数据。另一种是harvest交易成功后会自动刷新。

harvest本身的交易状态也有两个字段。isWritePending表示领取交易是否在pending中，isWriteError表示交易是否失败了。

### 常见错误要避开

**第一个坑：V2的harvest逻辑复杂**。V2版本的harvest可能不是直接领取，而是需要先从V1批量转移SUSHI。这个逻辑比V1和MiniChef都复杂，代码里要处理这种情况。

**第二个坑：PID可能重复**。不同链上的PID编号是独立的。同一个PID号在不同链上可能对应完全不同的池子。所以查的时候一定要同时指定chainId和pid。

**第三个坑：精度问题**。奖励计算可能涉及多个精度级别的转换。用Amount类型来处理比较安全，自己算可能会丢精度。

**第四个坑：V2的batch harvest更省gas**。如果用户有多个池子的奖励要领，V2支持批量harvest，一次操作可以领取多个池子的奖励，比逐个领取省很多gas。

**第五个坑：harvest不需要授权**。harvest函数是任何人，都可以帮任何人领取的。不需要额外的授权。但有时候领取的时候合约可能还没拿到足够的SUSHI代币，这种情况会失败。

### 提示词模板

```markdown
帮我创建一个查询MasterChef质押池信息的hook。

需求：
- 查询用户在某个池子的质押数量
- 查询待领取的SUSHI奖励
- 提供领取奖励的函数
- 支持V1、V2、MiniChef三种版本

技术栈：
- React + TypeScript
- wagmi v2 + viem
- @tanstack/react-query v5
- sushi库

版本类型：
- MasterChefV1：以太坊主网旧版
- MasterChefV2：以太坊主网新版
- MiniChef：其他链如Polygon、BSC

输入：
- chainId：链ID
- chef：版本类型
- pid：池子ID
- token：池子对应的LP代币
- enabled：是否启用
- watch：是否监听区块刷新

输出：
- balance：质押数量
- pendingSushi：待领取奖励
- harvest：领取函数
- isLoading：是否加载中
- isError：是否出错
- isWritePending：领取交易是否等待中
- isWriteError：领取交易是否失败

实现要点：
1. 批量查询SUSHI余额、用户信息、待领取奖励
2. 根据chef类型选择正确的合约和函数
3. watch开启时监听新区块刷新数据
4. harvest成功后自动刷新

注意：
- PID必须与池子对应
- V1/V2只在以太坊主网
- MiniChef在其他链
- watch会增加RPC调用
- pendingSushi大于0才能harvest
```

### 实际避坑指南

第一个，确认版本和链的对应关系。以太坊主网用V1或V2，其他链用MiniChef。不要搞混了。

第二个，PID要从配置里读取。PID不是随便设的数值，要从质押池的配置数据里获取。配置一般包括池子地址、对应LP代币等信息。

第三个，watch看需求开启。如果只是页面加载时查一次，不需要实时数据，就不要开启watch，可以节省RPC调用。如果是要实时显示奖励变化，那就开启。

第四个，batch harvest的优化。如果用户有多个池子在同一个V2合约上，可以考虑用batch功能一次领取，比逐个领取省很多gas。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useMasterChef` 用于查询 MasterChef 质押池信息。基本用法如下：

```typescript
import { useMasterChef, ChefType } from '@sushiswap/wag'

function PoolInfo({ chainId, pid, lpToken }) {
  const { balance, pendingSushi, harvest, isLoading } = useMasterChef({
    chainId,
    chef: ChefType.MasterChefV2, // 或 MiniChef
    pid,
    token: lpToken,
    enabled: Boolean(account && chainId),
    watch: true, // 开启区块监听
  })

  if (isLoading) return <div>加载中...</div>

  return (
    <div>
      <div>质押数量: {balance?.toSignificant(6)} LP</div>
      <div>待领取: {pendingSushi?.toSignificant(6)} SUSHI</div>
      {pendingSushi && pendingSushi.gt(0) && (
        <button onClick={harvest}>领取奖励</button>
      )}
    </div>
  )
}
```

### 5.2 常见使用场景

#### 场景一：显示池子信息

```typescript
function FarmCard({ farm }) {
  const { balance, pendingSushi, harvest } = useMasterChef({
    chainId: farm.chainId,
    chef: farm.chefType,
    pid: farm.pid,
    token: farm.lpToken,
  })

  return (
    <Card>
      <div>池子: {farm.lpToken.symbol}</div>
      <div>APY: {farm.apy}%</div>
      <div>你的质押: {balance?.toSignificant(6)}</div>
      <div>待领取: {pendingSushi?.toSignificant(6)} SUSHI</div>
      <HarvestButton harvest={harvest} pendingSushi={pendingSushi} />
    </Card>
  )
}
```

#### 场景二：批量显示多个池子

```typescript
function MyFarms({ farms }) {
  return (
    <div>
      {farms.map(farm => (
        <FarmCard key={`${farm.chainId}-${farm.pid}`} farm={farm} />
      ))}
    </div>
  )
}
```

#### 场景三：自动刷新奖励

```typescript
function LiveRewards({ chainId, pid, token }) {
  // watch: true 会监听新区块，自动刷新 pendingSushi
  const { pendingSushi, harvest } = useMasterChef({
    chainId,
    chef: ChefType.MiniChef,
    pid,
    token,
    watch: true, // 开启自动刷新
  })

  // pendingSushi 会随新区块自动更新
  return (
    <div>
      <div>实时奖励: {pendingSushi?.toSignificant(6)} SUSHI</div>
      <button onClick={harvest} disabled={!pendingSushi || pendingSushi.eq(0)}>
        收割
      </button>
    </div>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **正确选择 ChefType**
   ```typescript
   // 以太坊主网
   chainId === EvmChainId.ETHEREUM
     ? ChefType.MasterChefV2
     : ChefType.MiniChef
   ```

2. **开启 watch 实时更新**
   ```typescript
   // 需要实时显示 pendingSushi 时开启
   watch: true
   ```

3. **检查 pendingSushi 是否大于 0**
   ```typescript
   if (pendingSushi && pendingSushi.gt(0)) {
     return <button onClick={harvest}>领取</button>
   }
   ```

#### ❌ Don'ts

1. **不要用错误的 PID**
   ```typescript
   // 错误：PID 必须与实际池子对应
   pid: 999 // 可能不存在

   // 正确：从配置或合约获取正确 PID
   pid: farm.pid
   ```

2. **不要忘记 enabled 检查**
   ```typescript
   // 错误
   enabled: true

   // 正确
   enabled: Boolean(account && chainId && pid)
   ```

3. **不要在 watch 时不做限制**
   ```typescript
   // watch: true 会增加 RPC 调用
   // 不要在太多池子上同时开启
   // 只在用户查看的池子上开启
   ```

4. **不要忘记 harvest 后刷新**
   ```typescript
   // harvest 成功后数据会自动刷新
   // 因为 harvest 交易会触发 onSuccess 中的 refetch
   ```
