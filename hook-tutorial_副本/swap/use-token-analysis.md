> 源代码路径: `apps/web/src/lib/hooks/api/useTokenAnalysis.ts`

# useTokenAnalysis

## 1. 大白话讲讲这个hook的作用

`useTokenAnalysis` *(一个React hook，用于获取代币的安全分析数据，包括审计状态、是否套娃代币、流动性等)* 帮你获取代币的"分析数据"。

这个分析数据包括：
- 代币的安全审计状态
- 是否是 honeypot（貔貅合约）
- 流动性情况
- 持有者分布
- 其他风险指标

注意：只用于代币上线页面，不要在其他地方用这个接口。

## 2. 讲讲为什么需要封装该hook

封装提供：
- 地址验证（必须是有效的EVM地址）
- 条件获取（必须有 chainId 和 address）
- 类型安全

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
```typescript
args: Partial<GetTokenAnalysis>
shouldFetch?: boolean
```

**输出：**
```typescript
{
  data: TokenAnalysis,
  isLoading: boolean,
  error: Error | null
}
```

**执行逻辑：**
1. 验证地址是有效EVM地址：`isEvmAddress(args.address)` *(检查字符串是否为有效EVM地址格式的函数)*
2. enabled = shouldFetch && chainId && address && isValidAddress
3. 调用 `getTokenAnalysis(args as GetTokenAnalysis)` *(获取代币分析数据的API函数)*

## 4. 怎么给这个hook写AI提示词

这个hook用来查一个代币的安全分析数据——是不是貔貅合约、有没有审计过、流动性怎么样、持有者分布如何。主要用在代币上线页面，让用户能判断这个代币靠不靠谱。

### 写提示词的小技巧

**第一，只在代币上线页面用。** 这个接口是专门给上币流程准备的，别的地方随便调用会有不必要的服务器压力。

**第二，地址必须是个正经的EVM地址。** 传个乱七八糟的字符串进去查不出东西，还可能出问题。先用 isEvmAddress 验证一下格式对不对。

**第三，shouldFetch当开关用。** 比如用户还没输入地址，或者地址格式不对，就别发请求了。

### 提示词模板

```
帮我写一个React hook，功能是获取代币的安全分析数据。

具体需求：
1. 调用 getTokenAnalysis 函数
2. 地址必须是有效的EVM地址格式，先验证再查
3. 支持 shouldFetch 开关

注意：这个hook专门给代币上线页面用的

返回：{ data: TokenAnalysis, isLoading, error }
```

### 实际用的例子

```typescript
const { data: analysis } = useTokenAnalysis({
  chainId: ChainId.ETHEREUM,
  address: '0x...',
})

// 显示风险分析
<RiskPanel analysis={analysis} />
```

拿到分析数据之后，想怎么展示就怎么展示，比如标红警告"貔貅合约"，或者标绿显示"已审计"。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 基础调用
const { data: analysis, isLoading, error } = useTokenAnalysis({
  chainId: ChainId.ETHEREUM,
  address: tokenAddress,
})

// analysis 包含代币分析数据
console.log(analysis?.isHoneypot) // 是否是貔貅
console.log(analysis?.auditStatus) // 审计状态
console.log(analysis?.liquidity) // 流动性
```

### 常见使用场景

**场景1：代币上线页面的风险提示**

```typescript
const { data: analysis } = useTokenAnalysis({
  chainId,
  address: newTokenAddress,
})

return (
  <TokenAnalysisPanel>
    {analysis?.isHoneypot && (
      <Warning type="danger" message="该代币可能是貔貅合约" />
    )}
    {analysis?.auditStatus === 'audited' ? (
      <Badge type="success" message="已审计" />
    ) : (
      <Warning type="warning" message="未审计" />
    )}
    <LiquidityInfo liquidity={analysis?.liquidity} />
    <HolderDistribution distribution={analysis?.holders} />
  </TokenAnalysisPanel>
)
```

**场景2：阻止高风险代币交易**

```typescript
const { data: analysis } = useTokenAnalysis({...})

// 如果检测到高风险，禁用交易按钮
const canTrade = analysis &&
  !analysis.isHoneypot &&
  analysis.liquidity > MIN_LIQUIDITY &&
  analysis.holderCount > MIN_HOLDERS

return (
  <Button disabled={!canTrade}>
    {canTrade ? '交易' : '该代币存在风险'}
  </Button>
)
```

**场景3：详细的分析报告**

```typescript
const { data: analysis } = useTokenAnalysis({...})

// 显示完整分析
return (
  <AnalysisReport>
    <Section title="基本信息">
      <div>名称: {analysis?.name}</div>
      <div>符号: {analysis?.symbol}</div>
      <div>总供给: {analysis?.totalSupply}</div>
    </Section>

    <Section title="风险评估">
      <RiskItem label="貔貅检测" status={analysis?.isHoneypot ? 'danger' : 'safe'} />
      <RiskItem label="审计状态" status={analysis?.auditStatus} />
      <RiskItem label="流动性" status={analysis?.liquidity > 100000 ? 'safe' : 'warning'} />
    </Section>

    <Section title="持有者分布">
      <HolderChart data={analysis?.holders} />
    </Section>
  </AnalysisReport>
)
```

### Dos and Don'ts

**✅ Do:**
- 只在代币上线/交易页面使用这个 hook
- 验证地址有效性后再调用
- 使用 `shouldFetch` 控制何时获取
- 根据分析结果向用户展示风险提示

**❌ Don't:**
- 不要在其他页面（非代币上线页面）使用这个接口
- 不要忽略地址验证，EVM 地址格式必须正确
- 不要假设分析数据总是存在，处理 undefined 情况
- 不要将这个 hook 用于交易决策的唯一依据，分析仅供参考
