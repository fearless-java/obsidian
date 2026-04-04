> 源代码路径: `packages/hooks/src/useMediaQuery.ts`

# useMediaQuery

## 1. 大白话讲讲这个 hook 的作用

`useMediaQuery` *(一个React hook，用于执行CSS媒体查询，基于window.matchMedia实现，返回布尔值表示查询是否匹配)* 是一个执行 CSS 媒体查询的 hook。你传入一个 CSS 媒体查询字符串，它会告诉你这个查询当前是否匹配（true/false）。

举个例子：
- `(min-width: 768px)` - 屏幕宽度 >= 768px 时返回 true
- `(max-width: 768px)` - 屏幕宽度 <= 768px 时返回 true
- `(prefers-color-scheme: dark)` - 用户喜欢深色主题时返回 true

简单来说：它让你的 React 代码知道"用户设备的状态"，比如屏幕大小、颜色主题等。

## 2. 讲讲为什么需要封装该 hook

### 原生 window.matchMedia 的问题

```javascript
// 原生写法
const [matches, setMatches] = useState(
  window.matchMedia('(min-width: 768px)').matches
)

useEffect(() => {
  const mediaQuery = window.matchMedia('(min-width: 768px)')
  const handler = (e) => setMatches(e.matches)
  mediaQuery.addEventListener('change', handler)
  return () => mediaQuery.removeEventListener('change', handler)
}, [])
```

### 需要处理好的细节

1. **SSR 安全**：服务端没有 window.matchMedia *(浏览器API，用于检测媒体查询是否匹配)*
2. **事件监听**：媒体查询结果变化时要更新状态
3. **清理工作**：组件卸载时要移除监听
4. **初始化**：初始值要正确获取
5. **兼容性**：旧版浏览器可能需要 addListener/removeListener *(已废弃的API，新版浏览器使用addEventListener)*

### 封装的好处

- **开箱即用**：不需要关心底层实现
- **SSR 安全**：服务端渲染不会报错
- **自动监听**：matches 变化时自动更新组件状态
- **响应式**：返回布尔值，直接用于条件渲染

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

| 参数 | 类型 | 说明 |
|------|------|------|
| `query` | `string \| MediaQueryQuery` | CSS 媒体查询字符串 *(如'(min-width: 768px)','(prefers-color-scheme: dark)')* |

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| `boolean` | `true/false` | 媒体查询是否匹配 |

### 执行逻辑

```
组件初始化:
    ↓
window.matchMedia(query) 存在？ *(检查浏览器是否支持matchMedia API)*
    ↓
yes → matches = mediaQuery.matches *(mediaQuery.matches是布尔值，表示查询是否匹配)*
    ↓
no → matches = false (SSR 或不支持 *(Server-Side Rendering，服务端渲染)*)

组件挂载 useEffect:
    ↓
创建 MediaQueryList 对象 *(matchMedia返回的对象，包含matches属性和change事件)*
    ↓
添加 change 事件监听 *(监听媒体查询结果变化)*
    ↓
事件触发 → setMatches(mediaQuery.matches) *(matches变化时更新状态)*
    ↓
组件卸载: removeEventListener *(移除事件监听，防止内存泄漏)*
```

### 数据流

```
query ('(min-width: 768px)')
    ↓
window.matchMedia(query) → MediaQueryList *(浏览器API对象)*
    ↓
mediaQuery.matches → boolean
    ↓
state → matches *(React状态)*
    ↓
返回 matches
```

## 四、AI 提示词编写教学

你正在做一个媒体查询的小工具。媒体查询就是 CSS 里用来判断"现在的环境是什么样的"的一种方式，比如屏幕够不够宽、用户喜不喜欢深色主题、是横屏还是竖屏等。

传给这个工具的是一个 CSS 媒体查询的字符串，比如"(min-width: 768px)"意思是"宽度至少 768 像素"，或者"(prefers-color-scheme: dark)"意思是"用户喜欢深色主题"。它会返回 true 或 false，告诉你这个条件现在满不满足。

浏览器有一个原生的 API 叫 matchMedia 就是干这个的。先检查一下这个 API 存不存在（服务端渲染的时候可没有），不存在的话就默认返回 false。然后用这个 API 创建一个查询对象，它的 matches 属性会告诉你当前条件成不成立。

光知道当前的还不够，得盯着点，因为条件可能会变。比如用户把浏览器窗口拉大了，或者系统主题从浅色切换到了深色。监听 change 事件，当条件变化的时候更新一下状态。

组件不用了要把监听去掉，防止内存泄漏。这个工具返回的就是一个简单的布尔值，true 就是条件匹配，false 就是不匹配，用起来很直接。

## 5. 该 hook 的用法教学

### 基本用法

```typescript
import { useMediaQuery } from './useMediaQuery'

function ResponsiveLayout() {
  const isMobile = useMediaQuery('(max-width: 768px)')

  return isMobile ? <MobileLayout /> : <DesktopLayout />
}
```

### 常见使用场景

🟩 **响应式布局切换**

```typescript
function App() {
  const isMobile = useMediaQuery('(max-width: 768px)')
  const isTablet = useMediaQuery('(min-width: 769px) and (max-width: 1024px)')
  const isDesktop = useMediaQuery('(min-width: 1025px)')

  return (
    <>
      {isMobile && <MobileNav />}
      {isTablet && <TabletNav />}
      {isDesktop && <DesktopNav />}
    </>
  )
}
```

🟩 **深色模式检测**

```typescript
function ThemeProvider({ children }) {
  const prefersDark = useMediaQuery('(prefers-color-scheme: dark)')
  const [theme, setTheme] = useState(prefersDark ? 'dark' : 'light')

  useEffect(() => {
    setTheme(prefersDark ? 'dark' : 'light')
  }, [prefersDark])

  return <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>
}
```

🟩 **减少动画偏好**

```typescript
function AnimationComponent() {
  const prefersReducedMotion = useMediaQuery('(prefers-reduced-motion: reduce)')

  return (
    <div className={prefersReducedMotion ? 'no-animation' : 'animate'}>
      <AnimatedContent />
    </div>
  )
}
```

### Dos and Don'ts

✅ **Do: 使用有效的 CSS 媒体查询字符串**

```typescript
// 正确：标准CSS媒体查询格式
const isWide = useMediaQuery('(min-width: 1200px)')
const isDark = useMediaQuery('(prefers-color-scheme: dark)')
const isPortrait = useMediaQuery('(orientation: portrait)')
```

❌ **Don't: 使用无效的查询语法**

```typescript
// 错误：不是有效的媒体查询
const isWide = useMediaQuery('width >= 1200') // 不是CSS语法

// 正确：CSS媒体查询格式
const isWide = useMediaQuery('(min-width: 1200px)')
```

✅ **Do: SSR 时考虑初始值**

```typescript
// 正确：SSR 时默认返回 false（matches 初始值）
function ServerComponent() {
  const isMobile = useMediaQuery('(max-width: 768px)')
  // SSR 时 isMobile 为 false
  return isMobile ? <Mobile /> : <Desktop />
}
```

❌ **Don't: 在 hook 外使用媒体查询结果**

```typescript
// 错误：媒体查询结果是响应式的，要在组件内使用
const isMobile = useMediaQuery('(max-width: 768px)')

// 错误：不能用在条件渲染之外的地方做计算
const width = isMobile ? 300 : 800
```

✅ **Do: 在组件挂载时获取准确值**

```typescript
// 正确：hook 在客户端会返回准确值
function Component() {
  const isMobile = useMediaQuery('(max-width: 768px)')

  useEffect(() => {
    // 这里的 isMobile 值和组件渲染时一致
    if (isMobile) {
      loadMobileResources()
    }
  }, [isMobile])
}
```

❌ **Don't: 假设值不会变化**

```typescript
// 错误：媒体查询结果会随窗口大小变化
// 不要在 useEffect 外部缓存这个值
const isMobile = useMediaQuery('(max-width: 768px)')
// 窗口resize后，isMobile 会自动更新
```

✅ **Do: 结合 useBreakpoint 使用更方便**

```typescript
// 正确：如果使用 Tailwind，可以用 useBreakpoint
const { isSm, isMd, isLg } = useBreakpoint('sm')
// 更语义化，不需要记像素值
```

❌ **Don't: 混用不同的检测方式**

```typescript
// 错误：不要同时用多种方式检测屏幕大小
const isMobile = useMediaQuery('(max-width: 768px)')
const { width } = useWindowSize() // 会冲突！

// 正确：只使用一种方式
const isMobile = useMediaQuery('(max-width: 768px)')
```
