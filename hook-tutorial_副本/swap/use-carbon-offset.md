> 源代码路径: `apps/web/src/lib/swap/useCarbonOffset.ts`

# useCarbonOffset

## 1. 大白话讲讲这个hook的作用

`useCarbonOffset` *(一个React hook，用于管理碳中和交易的开关状态，数据持久化到localStorage)* 是一个简单的开关hook，用于记录用户是否选择"碳中和"交易。

当开启时：
- 交易会额外包含一笔碳信用合约调用
- 用于支持环保项目

## 2. 讲讲为什么需要封装该hook

封装提供：
- 持久化存储（localStorage）
- 默认值为 false
- 统一的读取接口

## 3. 讲讲该hook的执行逻辑和数据流向

**输入：**
无

**输出：**
```typescript
[
  boolean,                    // 当前值
  (value: boolean) => void    // 设置函数
]
```

**执行逻辑：**
1. 使用 `useLocalStorage('carbonOffset', false)` *(React hook，封装localStorage，存储键值对数据，支持任意类型的自动序列化/反序列化)* 读取
2. 返回 [值, setter]

## 4. 怎么给这个hook写AI提示词

这是一个简简单单的开关hook，决定用户要不要选择"碳中和"交易。开启的话，每笔交易会额外包含一笔碳信用合约调用，支持环保项目。用户的偏好会保存到localStorage里，刷新页面也还在。

### 写提示词的小技巧

**第一，就是个localStorage封装。** 没什么复杂的逻辑，就是存个布尔值。key是'carbonOffset'，默认是false。

**第二，返回格式是[值, setter]。** 和普通的useState一样用，tuple形式。

### 提示词模板

```
帮我写一个React hook，功能是管理"碳中和交易"的开关状态。

具体需求：
1. 用 useLocalStorage 实现
2. key 是 'carbonOffset'
3. 默认值 false
4. 返回 [boolean, setter] 的格式

返回：[boolean, (value: boolean) => void]
```

### 实际用的例子

```typescript
const [carbonOffset, setCarbonOffset] = useCarbonOffset()

<Switch
  checked={carbonOffset}
  onChange={setCarbonOffset}
  label="碳中和交易"
/>
```

用户点开关就会自动保存到localStorage，下次进来还是开的。

### Production-Ready Example

```typescript
const [carbonOffset, setCarbonOffset] = useCarbonOffset()

<Switch
  checked={carbonOffset}
  onChange={setCarbonOffset}
  label="碳中和交易"
/>
```

## 5. 该 hook 的用法教学

### 基本用法

```typescript
// 基础调用
const [carbonOffset, setCarbonOffset] = useCarbonOffset()

// 读取当前值
console.log(carbonOffset) // false

// 设置为开启
setCarbonOffset(true)

// 设置为关闭
setCarbonOffset(false)

// 切换
setCarbonOffset(prev => !prev)
```

### 常见使用场景

**场景1：碳中和交易开关**

```typescript
const [carbonOffset, setCarbonOffset] = useCarbonOffset()

return (
  <div>
    <label>
      <input
        type="checkbox"
        checked={carbonOffset}
        onChange={(e) => setCarbonOffset(e.target.checked)}
      />
      启用碳中和交易
    </label>
    {carbonOffset && (
      <p>每笔交易将额外投入0.1%支持环保项目</p>
    )}
  </div>
)
```

**场景2：交易确认对话框显示**

```typescript
const [carbonOffset, setCarbonOffset] = useCarbonOffset()

// 在交易确认中显示碳中和选项
return (
  <ConfirmDialog>
    <div>确认交易 {inputAmount} {fromToken.symbol} → {outputAmount} {toToken.symbol}</div>

    <CarbonOffsetOption>
      <Checkbox
        checked={carbonOffset}
        onChange={setCarbonOffset}
      />
      <div>
        <div>碳中和交易</div>
        <div className="description">额外投入0.1%支持碳信用项目</div>
      </div>
    </CarbonOffsetOption>
  </ConfirmDialog>
)
```

**场景3：用户偏好设置页面**

```typescript
const [carbonOffset, setCarbonOffset] = useCarbonOffset()

// 用户设置页面
return (
  <Settings>
    <SettingsSection title="交易偏好">
      <SettingsItem
        label="碳中和交易"
        description="默认启用碳中和交易选项"
        trailing={
          <Switch
            checked={carbonOffset}
            onChange={setCarbonOffset}
          />
        }
      />
    </SettingsSection>
  </Settings>
)
```

### Dos and Don'ts

**✅ Do:**
- 使用 `useLocalStorage` 自动持久化，用户刷新页面后设置保留
- 使用布尔值直接判断，不需要 `=== true`
- 在 UI 中清晰展示碳中和选项的含义
- 将用户的碳中和选择包含在交易参数中

**❌ Don't:**
- 不要假设默认值是 false，虽然当前是 false
- 不要在不需要时频繁调用 setCarbonOffset
- 不要忽略类型检查，setCarbonOffset 期望接收 boolean
- 不要将这个 hook 用于其他持久化需求，它专门用于碳中和开关
