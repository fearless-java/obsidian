> 源代码路径: `apps/web/src/lib/wagmi/hooks/wallet/useConnect.ts`

# useConnect

## 大白话讲讲这个hook的作用

`useConnect` 是一个连接钱包的 Hook。它封装了 wagmi 的 `useConnect` 方法，增加了：
1. Pending 状态管理
2. 钱包连接分析事件（telemetry）
3. 错误日志记录

大白话：就是"连接钱包"的按钮。用户点击后，会弹出钱包选择框，让用户选择并连接钱包。

## 讲讲为什么需要封装该hook

1. **用户体验**：封装了连接过程中的 pending 状态，UI 可以显示加载。

2. **分析事件**：连接成功或失败时发送分析事件，便于产品分析。

3. **错误处理**：统一处理连接错误，记录日志。

4. **统一接口**：封装后提供统一的 `connect` / `connectAsync` 接口。

5. **钱包信息**：通过 `useConnection` 获取当前连接的钱包信息。

## 讲讲该hook的执行逻辑和数据流向

### 输入参数 (Inputs)

```typescript
// 透传 wagmi 的 useConnect 参数
props?: Parameters<typeof useWagmiConnect>[0]
```

### 输出 (Outputs)

```typescript
{
  connect: (...args: Parameters<typeof connectAsync>) => Promise<any>,
  connectAsync: (...args: Parameters<typeof connectAsync>) => Promise<any>,  // 带分析
  pending: boolean,  // 连接中状态
  // ... 其他 wagmi connect 状态
}
```

### 执行逻辑详解

#### 1. 获取 wagmi Connect

```typescript
const { mutateAsync: connectAsync, ...rest } = useWagmiConnect(props)
const { connector } = useConnection()
```

#### 2. 封装连接方法

```typescript
const _connectAsync = async (...args: Parameters<typeof connectAsync>) => {
  let result

  try {
    setPending(true)
    result = await connectAsync(...args)

    // 发送成功分析事件
    sendAnalyticsEvent(InterfaceEventName.WALLET_CONNECTED, {
      result: WalletConnectionResult.SUCCEEDED,
      wallet_address: result.accounts[0],
      wallet_type: args[0].connector.name,
      page: window.location,
    })

    return result
  } catch (e) {
    // 发送失败分析事件
    sendAnalyticsEvent(InterfaceEventName.WALLET_CONNECTED, {
      result: WalletConnectionResult.FAILED,
      wallet_type: connector?.name,
      page: window.location,
      error: e instanceof Error ? e.message : undefined,
    })
    console.error(e)
    throw e
  } finally {
    setPending(false)
  }
}
```

### 数据流向图

```
用户点击连接
         ↓
    setPending(true)
         ↓
    connectAsync(args)
         ↓
    ┌────────────────────────────────────┐
    │  成功:                             │
    │  1. sendAnalyticsEvent SUCCEEDED   │
    │  2. 返回 result                    │
    │  3. setPending(false)              │
    └────────────────────────────────────┘
         ↓
    ┌────────────────────────────────────┐
    │  失败:                             │
    │  1. sendAnalyticsEvent FAILED     │
    │  2. console.error                  │
    │  3. throw error                   │
    │  4. setPending(false)              │
    └────────────────────────────────────┘
```

## 四、AI提示词编写教程

### 为什么要写好提示词

写提示词就像给AI一个详细的工作说明书。你要把需求描述得越清楚，AI就越能帮你生成高质量的代码。一个好的提示词应该包含：技术栈说明、输入输出定义、关键实现逻辑、以及容易出错的地方提醒。

### 核心要点

**第一点：pending状态要显示**。连接钱包可能需要好几秒甚至更久。用户点了连接按钮后，必须有明确的loading状态显示，比如按钮变灰显示"连接中..."。否则用户可能以为没点到，又点了一次。

**第二点：用户拒绝不算错误**。当用户弹窗出来选择拒绝的时候，这不是程序的错误。代码里如果把这个当作错误处理并显示给用户，用户会很困惑。应该静默处理。

**第三点：分析事件很重要**。连接成功还是失败、用的什么钱包、用户在哪里发起的连接，这些数据对产品优化很有价值。应该把这些信息发送到分析服务。

**第四点：多钱包要支持**。用户的浏览器里可能安装了多个钱包插件。好的设计应该让用户选择用哪个钱包连接，而不是只能用一个。

**第五点：页面位置要记录**。用户可能在不同页面发起连接。记录下window.location对于分析用户行为很有帮助，能知道哪个页面的连接转化率更高。

### 约束条件要记住

**第一，需要connector配置**。wagmi需要配置好支持哪些钱包连接器。如果没配置，可能连不上。

**第二，钱包需要授权**。连接过程需要用户钱包授权。如果用户钱包是锁定状态，会先弹窗要求解锁。

**第三，用户可能拒绝**。用户看到钱包弹窗后，可能选择取消。这是正常操作，不是错误。

**第四，不同钱包格式可能不同**。不同钱包返回账户地址的格式可能略有差异，不过一般都以太坊地址格式是标准的。

### 状态管理要清楚

连接状态有三种。pending表示正在连接中。connect是同步连接方法。connectAsync是异步版本，带分析事件功能。

分析事件会记录连接结果：SUCCEEDED表示成功，FAILED表示失败。成功时记录钱包地址，失败时记录错误原因。

### 常见错误要避开

**第一个坑：finally保证清理**。setPending(false)放在finally里，这样不管连接成功还是失败，pending状态都会被重置。不会卡在loading状态。

**第二个坑：钱包拒绝要友好**。用户拒绝钱包授权不应该显示错误toast。用户取消操作是正常的，不需要告诉他"连接失败"。

**第三个坑：获取钱包名称**。通过args[0].connector.name可以获取用户选择的钱包名称。这个信息很有用，可以记录下来分析用户偏好。

**第四个坑：SPA的location问题**。在单页应用中，window.location不会像传统网页那样刷新。如果你需要在路由变化时更新location信息，要自己处理。

**第五个坑：快速点击防护**。用户可能快速多次点击连接按钮。最好在UI层面防止重复点击，比如点一次后禁用按钮。

### 提示词模板

```markdown
帮我创建一个封装钱包连接的hook。

需求：
- 调用方能显示连接中状态
- 记录连接成功失败到分析服务
- 统一处理用户拒绝等异常情况
- 支持多种钱包

技术栈：
- React + TypeScript
- wagmi v2
- 分析服务

输出：
- connect：同步连接方法
- connectAsync：异步连接方法，带分析
- pending：是否连接中
- 其他wagmi原有状态

连接结果分析：
- 成功时记录：钱包地址、钱包类型、页面位置
- 失败时记录：错误原因、钱包类型、页面位置

实现要点：
1. 包装wagmi的useConnect
2. 添加pending状态管理
3. 连接前后发送分析事件
4. 捕获异常但区分用户拒绝和真正错误
5. 返回统一格式给调用方

注意：
- pending状态要让UI有反馈
- 用户取消不显示错误
- 记录钱包类型有利于分析
- 防止快速重复点击
```

### 实际避坑指南

第一个，pending的清理。setPending(false)一定要放在finally里，这样连接过程中途出错也能重置状态。不会卡死。

第二个，用户拒绝的处理。判断是否是用户拒绝可以用isUserRejectedError这个函数。如果是用户拒绝，可以什么都不显示，或者显示一个友好的提示。

第三个，分析数据的价值。记录下用户用什么钱包、在哪个页面发起连接，这些数据能帮助产品团队优化连接流程。比如发现大部分用户在Swap页面连接，就可以优化那个页面的连接入口。

第四个，多钱包支持。wagmi支持配置多个connector，用户可以有多个钱包选择。这要求UI上显示钱包选择列表。

## 5. 该 hook 的用法教学

### 5.1 基础用法

`useConnect` 用于连接钱包。基本用法如下：

```typescript
import { useConnect } from '@sushiswap/wag'

function ConnectButton() {
  const { connect, connectAsync, pending, ...rest } = useConnect()

  if (rest.isConnected) {
    return <div>已连接: {rest.account}</div>
  }

  if (rest.isConnecting) {
    return <div>连接中...</div>
  }

  return (
    <button onClick={() => connect({ connector: rest.connector })} disabled={pending}>
      {pending ? '连接中...' : '连接钱包'}
    </button>
  )
}
```

### 5.2 常见使用场景

#### 场景一：显示钱包选择器

```typescript
function WalletSelector() {
  const { connectAsync, pending } = useConnect()

  const { connectors, connect } = useConnect()

  return (
    <div>
      <h3>选择钱包</h3>
      {connectors.map((connector) => (
        <button
          key={connector.uid}
          onClick={() => connectAsync({ connector })}
          disabled={pending}
        >
          {pending ? '连接中...' : connector.name}
        </button>
      ))}
    </div>
  )
}
```

#### 场景二：连接成功后执行操作

```typescript
function ConnectAndSwap() {
  const { connectAsync, pending } = useConnect()

  const handleConnect = async () => {
    try {
      const result = await connectAsync()
      // 连接成功后可以执行其他操作
      console.log('已连接:', result.accounts[0])
    } catch (e) {
      // 用户拒绝等错误
      console.error('连接失败:', e)
    }
  }

  return (
    <button onClick={handleConnect} disabled={pending}>
      {pending ? '连接中...' : '连接并交易'}
    </button>
  )
}
```

#### 场景三：组合 useAccount 使用

```typescript
function WalletStatus() {
  const { account, isConnected } = useAccount()
  const { connectAsync, pending } = useConnect()

  if (isConnected) {
    return (
      <div>
        <div>已连接: {account}</div>
        <DisconnectButton />
      </div>
    )
  }

  return (
    <button onClick={() => connectAsync()} disabled={pending}>
      {pending ? '连接中...' : '连接钱包'}
    </button>
  )
}
```

### 5.3 Dos and Don'ts

#### ✅ Dos

1. **处理 pending 状态**
   ```typescript
   // 连接过程中禁用按钮
   <button disabled={pending || isConnecting}>
     {pending ? '连接中...' : '连接'}
   </button>
   ```

2. **处理用户拒绝**
   ```typescript
   try {
     await connectAsync()
   } catch (e) {
     // 用户拒绝不是错误，不需要显示 toast
     if (!isUserRejectedError(e)) {
       showErrorToast(e)
     }
   }
   ```

3. **使用 connectAsync 而不是 connect**
   ```typescript
   // connectAsync 返回 Promise，方便处理
   try {
     await connectAsync()
   } catch (e) {
     // 错误处理
   }
   ```

#### ❌ Don'ts

1. **不要在 render 时调用 connect**
   ```typescript
   // 错误：render 期间不应该有副作用
   if (!isConnected) {
     connect() // 不要这样做！
   }

   // 正确：用户点击按钮触发
   return <button onClick={() => connect()}>连接</button>
   ```

2. **不要忽略 pending 状态**
   ```typescript
   // 错误：用户可能不知道连接正在进行
   return <button onClick={() => connect()}>连接</button>

   // 正确：显示 pending 状态
   return <button disabled={pending} onClick={() => connect()}>
     {pending ? '连接中...' : '连接'}
   </button>
   ```

3. **不要假设连接立即成功**
   ```typescript
   // 错误
   connect()
   navigate('/swap') // 可能还没连接成功

   // 正确
   await connectAsync()
   navigate('/swap')
   ```

4. **不要忽略多钱包场景**
   ```typescript
   // 有些 DApp 支持多个钱包
   // 应该让用户选择
   {connectors.map(c => (
     <button key={c.uid} onClick={() => connectAsync({ connector: c })}>
       {c.name}
     </button>
   ))}
   ```
