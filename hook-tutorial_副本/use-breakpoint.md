> 源代码路径: `packages/hooks/src/useBreakpoint.ts`

# useBreakpoint

## 1. 大白话讲讲这个 hook 的作用

`useBreakpoint` *(一个React hook，用于检测当前屏幕宽度是否达到某个断点（breakpoint），基于CSS媒体查询实现，返回布尔值表示是否匹配)* 是一个用于检测当前屏幕宽度是否达到某个断点（breakpoint）的 hook。它基于 Tailwind CSS *(一个Utility-first的CSS框架，提供了预定义的屏幕断点配置，如sm:640px, md:768px等)* 的默认断点配置来判断屏幕尺寸，比如 `sm`（640px）、`md`（768px）、`lg`（1024px）等。当屏幕宽度满足或超过指定的断点时，返回 `true`，否则返回 `false`。

简单来说：它告诉你"现在的屏幕够不够大"。

## 2. 讲讲为什么需要封装该 hook

在实际项目中，我们经常需要根据不同的屏幕尺寸来展示不同的 UI。比如：
- 在大屏幕上显示表格，在小屏幕上显示卡片
- 在大屏幕上显示侧边栏，在小屏幕上隐藏或显示汉堡菜单

原生实现需要：
1. 手动监听 `window.resize` 事件 *(浏览器窗口大小改变时触发的事件，用于响应式开发)*
2. 维护一个 state 来存储当前屏幕宽度
3. 手动比较宽度与断点值
4. 处理 SSR（服务端渲染）场景，因为服务端没有 `window` 对象 *(Server-Side Rendering，服务端渲染，指在服务器上生成HTML，Next.js默认使用)*

封装成 hook 的好处：
- **代码复用**：项目多处需要检测断点时，无需重复编写监听逻辑
- **统一配置**：使用 Tailwind 默认断点，保持设计语言一致
- **SSR 友好**：内部处理了服务端渲染的场景
- **类型安全**：通过泛型约束，只能传入有效的 Tailwind 断点 key
- **返回值直观**：返回 `{ isSm: true }` 这样的对象，直接使用

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

| 参数 | 类型 | 说明 |
|------|------|------|
| `breakpointKey` | `keyof typeof defaultTheme.screens` | Tailwind 断点键名，如 `'sm'`、`'md'`、`'lg'` 等 |

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| `Record<Key, boolean>` | 对象 | Key 格式为 `is${Capitalize<K>}`，如 `isSm`、`isMd` |

### 执行逻辑

```
1. 接收 breakpointKey（如 'sm'）
2. 构建媒体查询字符串：'(min-width: 640px)'（从 Tailwind 配置获取对应宽度）
3. 调用 useMediaQuery hook *(执行CSS媒体查询的hook，基于window.matchMedia实现)* 执行媒体查询
4. 将结果包装成 { isSm: boolean } 这样的对象返回
5. 动态生成 Key 类型：`is${Capitalize<K>}`
```

### 数据流

```
breakpointKey ('sm')
    ↓
defaultTheme.screens['sm'] → '640px'
    ↓
useMediaQuery({ query: '(min-width: 640px)' }) *(基于window.matchMedia的hook，返回布尔值表示查询是否匹配)*
    ↓
boolean (true/false)
    ↓
{ isSm: boolean }
```

## 四、AI 提示词编写教学

想象你正在做一个响应式小工具，需要知道用户的屏幕到底够不够大。

先看看现在是什么环境。如果是服务器在跑代码（SSR），就别去碰窗口这类浏览器才有的东西，不然会报错。

查一下 Tailwind CSS 帮我们定义好的屏幕断点配置，它已经告诉了我们什么时候算"大屏"、什么时候算"小屏"：比如 640px 以上算 sm，768px 以上算 md，1024px 以上算 lg。这样你就不用自己记这些数字。

拿到断点对应的像素值后，把它拼成一个标准的 CSS 媒体查询字符串，比如"(min-width: 768px)"，意思就是"宽度至少 768 像素"。

把这个查询扔给专门处理媒体查询的小工具，让它帮你盯着屏幕大小有没有变化。窗口被拖大拖小的时候，它会自动告诉你结果变了。

最后把查询结果包装成方便使用的格式返回。比如查的是 sm 断点，就返回 `{ isSm: true/false }` 这样的对象，名字清晰明了，一看就知道是查的哪个断点。

要注意的是：服务器渲染时默认返回 false；断点的像素值要从配置里读，不要自己写死；返回的 key 要用驼峰格式。

## 五、该 hook 的用法教学

### 基本用法

```typescript
import { useBreakpoint } from './useBreakpoint'

function App() {
  // 检测是否是小型及以上屏幕 (>= 640px)
  const { isSm } = useBreakpoint('sm')

  // 检测是否是中型及以上屏幕 (>= 768px)
  const { isMd } = useBreakpoint('md')

  // 检测是否是大屏 (>= 1024px)
  const { isLg } = useBreakpoint('lg')

  return (
    <div>
      {isMd ? <DesktopLayout /> : <MobileLayout />}
    </div>
  )
}
```

### 常见使用场景

🟩 **响应式布局切换**

```typescript
function ResponsiveNav() {
  const { isLg } = useBreakpoint('lg')

  return isLg ? <DesktopNav /> : <MobileDrawerNav />
}
```

🟩 **条件渲染不同组件**

```typescript
function ProductList() {
  const { isSm, isMd, isLg } = useBreakpoint('sm')

  if (isLg) return <GridView columns={4} />
  if (isMd) return <GridView columns={3} />
  return <ListView />
}
```

🟩 **动态样式**

```typescript
function Card() {
  const { isSm } = useBreakpoint('sm')

  const cardStyle = {
    width: isSm ? '300px' : '100%',
    padding: isSm ? '20px' : '10px',
  }

  return <div style={cardStyle}>Content</div>
}
```

### Dos and Don'ts

✅ **Do: 使用 Tailwind 断点名称**

```typescript
// 正确：使用 Tailwind 默认断点名称
const { isSm } = useBreakpoint('sm')
const { isMd } = useBreakpoint('md')
const { isLg } = useBreakpoint('lg')
```

❌ **Don't: 硬编码断点值**

```typescript
// 错误：不应该硬编码
if (width >= 768) { ... }
```

✅ **Do: 结合条件渲染**

```typescript
// 正确：条件渲染不同布局
const { isMd } = useBreakpoint('md')
return isMd ? <DesktopLayout /> : <MobileLayout />
```

❌ **Don't: 在渲染初期依赖精确宽度**

```typescript
// 错误：不要依赖精确像素值做关键判断
if (width === 1024) { ... }
```

✅ **Do: SSR 安全使用**

```typescript
// 正确：SSR 时返回 false，不会有问题
function ServerComponent() {
  const { isMd } = useBreakpoint('md')
  // 服务端渲染时 isMd 为 false
  return isMd ? <DesktopNav /> : <MobileNav />
}
```

❌ **Don't: 在 hook 外使用断点值**

```typescript
// 错误：断点值应该用于条件渲染，不应该做复杂计算
const breakpoint = useBreakpoint('md')
const calculatedWidth = breakpoint.isMd ? containerWidth * 0.8 : containerWidth * 0.95
```
