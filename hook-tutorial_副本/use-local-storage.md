> 源代码路径: `packages/hooks/src/useLocalStorage.ts`

# useLocalStorage

## 1. 大白话讲讲这个 hook 的作用

`useLocalStorage` *(一个React hook，将数据持久化到浏览器LocalStorage，支持跨标签页同步，用法类似useState)* 是一个让你在 React 组件中使用 LocalStorage（本地存储）的 hook。它的工作是：

1. **读取**：从浏览器 LocalStorage 中读取某个 key 对应的值
2. **写入**：更新 state 时自动同步到 LocalStorage
3. **监听**：当同一个 key 在其他标签页被修改时，自动更新当前组件的 state
4. **类型安全**：支持泛型，TypeScript 知道存储的值的类型

简单来说：它把 LocalStorage 变成了一个"响应式"的状态，用法和 `useState` 几乎一样，但数据会持久保存。

## 2. 讲讲为什么需要封装该 hook

### 原生 LocalStorage 的问题

```javascript
// 原生写法
const [value, setValue] = useState(JSON.parse(localStorage.getItem('key') || 'null'))

// 更新
localStorage.setItem('key', JSON.stringify(newValue))

// 问题：
// 1. 每次都要手动 JSON.parse/JSON.stringify *(JSON序列化/反序列化，将JS对象转字符串和还原)*
// 2. SSR 时 localStorage 不存在会报错
// 3. 多个标签页之间不会同步更新
// 4. 错误处理需要重复写
```

### 需要处理好的细节

1. **SSR 安全**：服务端没有 window.localStorage *(浏览器本地存储，容量约5-10MB，可跨会话持久保存)*，需要返回默认值
2. **JSON 序列化**：自动处理 JS 对象和 JSON 的转换
3. **错误处理**：读取/写入出错时不能崩溃
4. **跨标签页同步**：一个标签页改了，其他标签页也要更新 *(通过自定义事件实现)*
5. **函数式更新**：支持 `setValue(prev => ...)` 用法
6. **事件触发**：需要手动触发 `StorageEvent` *(浏览器事件，当localStorage变化时在其他标签页触发)* 来同步

### 封装的好处

- **API 一致**：和 useState 用法几乎一样
- **开箱即用**：JSON 序列化自动处理
- **跨标签页同步**：通过 StorageEvent 监听
- **SSR 安全**：服务端返回初始值

## 3. 讲讲该 hook 的执行逻辑和数据流向

### 输入（参数）

| 参数 | 类型 | 说明 |
|------|------|------|
| `key` | `string` | LocalStorage 的 key *(用于标识存储值的键名)* |
| `initialValue` | `T` | 默认值（当 key 不存在或读取失败时使用） |

### 输出

| 返回值 | 类型 | 说明 |
|--------|------|------|
| `[storedValue, setValue]` | `[T, Dispatch<SetStateAction<T>>]` | 和 useState 返回值一样 |

### 执行逻辑

```
初始化:
    ↓
readValue() *(读取LocalStorage值的函数)*
    ↓
window 不存在？ → 返回 initialValue *(SSR时返回默认值)*
    ↓
window.localStorage.getItem(key) *(从本地存储获取值)*
    ↓
成功？ → JSON.parse(item) *(将JSON字符串转JS对象)* → 返回
    ↓
失败？ → 返回 initialValue *(读取失败时返回默认值)*
    ↓
useState(readValue) → storedValue *(初始化state)*

更新 setValue(newValue):
    ↓
value instanceof Function？ → value(storedValue) → valueToStore *(支持函数式更新)*
    ↓
不是函数 → newValue → valueToStore
    ↓
setStoredValue(valueToStore)
    ↓
window.localStorage.setItem(key, JSON.stringify(valueToStore)) *(写入本地存储)*
    ↓
window.dispatchEvent(new Event(key)) → 触发其他标签页更新 *(通过自定义事件通知其他标签页)*

其他标签页修改:
    ↓
window 监听 key 事件 *(监听其他标签页的修改)*
    ↓
listener → localStorage.getItem(key) → JSON.parse *(获取新值并解析)*
    ↓
setStoredValue(parsedValue) *(更新状态)*
    ↓
组件重渲染
```

### 数据流

```
LocalStorage (key → JSON string) *(浏览器本地存储的键值对)*
    ↓
readValue → JSON.parse → storedValue state *(读取并解析)*
    ↓
setValue → JSON.stringify → LocalStorage *(序列化并写入)*
    ↓
dispatchEvent → 其他标签页监听 → setStoredValue *(跨标签页同步)*
```

## 四、AI 提示词编写教学

你正在做一个有"记忆"的小工具，数据会保存到浏览器本地，下一次用户打开页面的时候还能读出来。就像 useState 一样好用，但数据不会在刷新页面后消失。

先确定一件事：现在是不是在浏览器里跑。服务端渲染的时候可没有本地存储，得先判断一下，没有的话就乖乖返回默认值。

存进去的数据是什么格式都行，数字、字符串、对象、数组都可以，但本地存储只认字符串。所以每次写入的时候要把数据转成 JSON 字符串，取出来的时候再转回 JavaScript 能认识的样子。如果转的时候出错了（比如存的 JSON 格式本身就有问题），也要能 handle 住，返回默认值而不是直接崩溃。

更新数据的时候要支持"函数式更新"，就是 setValue(prev => 新值) 这种用法，意思基于之前的值来计算新的值。

更新完之后还要通知一下其他标签页。同一个应用可能在多个标签页里开着，在一个标签页改了设置，其他标签页应该立刻生效，而不是要刷新才知道。用自定义事件的方式通知比较靠谱，比浏览器原生的 StorageEvent 更可靠。

读取数据的时候也是，如果其他标签页改了，要能监听到变化并更新。所以要监听前面说的那个自定义事件，事件来了就去重新读一遍本地存储，然后更新状态。

组件不用了的时候把监听清理掉，防止内存泄漏。错误处理方面，如果出问题了只 log 一下就好，不要让应用崩溃。

## 五、该 hook 的用法教学

### 基本用法

```typescript
import { useLocalStorage } from './useLocalStorage'

function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage('theme', 'light')

  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      Current: {theme}
    </button>
  )
}
```

### 常见使用场景

🟩 **主题切换**

```typescript
function ThemeProvider({ children }) {
  const [theme, setTheme] = useLocalStorage('sushi-theme', 'light')

  const toggleTheme = () => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light')
  }

  return (
    <ThemeContext.Provider value={{ theme, setTheme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}
```

🟩 **表单草稿保存**

```typescript
function DraftForm() {
  const [formData, setFormData] = useLocalStorage('draft-form', {
    name: '',
    email: '',
    message: ''
  })

  const handleSubmit = (e) => {
    e.preventDefault()
    submitForm(formData)
    setFormData({ name: '', email: '', message: '' }) // 清空草稿
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={formData.name}
        onChange={e => setFormData(prev => ({ ...prev, name: e.target.value }))}
      />
      {/* ... */}
    </form>
  )
}
```

🟩 **用户偏好设置**

```typescript
function UserPreferences() {
  const [preferences, setPreferences] = useLocalStorage('user-prefs', {
    language: 'en',
    notifications: true,
    autoPlay: false
  })

  return (
    <div>
      <label>
        <input
          type="checkbox"
          checked={preferences.notifications}
          onChange={e => setPreferences(prev => ({
            ...prev,
            notifications: e.target.checked
          }))}
        />
        通知
      </label>
    </div>
  )
}
```

### Dos and Don'ts

✅ **Do: 使用函数式更新**

```typescript
// 正确：基于前一个值更新
setTheme(prev => prev === 'light' ? 'dark' : 'light')

// 正确：展开更新对象
setFormData(prev => ({ ...prev, name: newName }))
```

❌ **Don't: 直接修改对象**

```typescript
// 错误：不要直接修改
storedValue.name = newName
setStoredValue(storedValue)

// 正确：创建新对象
setStoredValue({ ...storedValue, name: newName })
```

✅ **Do: 使用合适的初始值类型**

```typescript
// 正确：指定泛型类型
const [user, setUser] = useLocalStorage<User | null>('user', null)

// 正确：对象初始值
const [settings, setSettings] = useLocalStorage('settings', {
  theme: 'light',
  language: 'en'
})
```

❌ **Don't: 存储非序列化数据**

```typescript
// 错误：函数、undefined 等无法 JSON 序列化
const [data, setData] = useLocalStorage('data', {
  complex: () => {}, // 无法序列化！
  symbol: Symbol('test'), // 无法序列化
})

// 正确：只存储可序列化的数据
const [data, setData] = useLocalStorage('data', {
  name: 'test',
  timestamp: Date.now()
})
```

✅ **Do: 跨标签页同步自动工作**

```typescript
// 正确：在标签页A修改，标签页B会自动更新
// 标签页A:
setTheme('dark')

// 标签页B:
// 自动收到更新，无需额外处理
```

❌ **Don't: 在 SSR 环境访问 storedValue 的特定格式**

```typescript
// 错误：SSR 时 storedValue 是 initialValue
function ServerComponent() {
  const [data, setData] = useLocalStorage('key', { type: 'default' })
  // SSR 时 data 是 initialValue
  // hydration *(水合作用，SSR后客户端接管)* 后才会变成真实值
}
```

✅ **Do: 错误处理**

```typescript
// 正确：useLocalStorage 内部已经处理了错误
// 读取失败时返回 initialValue
// 写入失败时不会崩溃，只是 console.log
```

❌ **Don't: 存储敏感信息**

```typescript
// 错误：LocalStorage 不加密，敏感信息不应存在这里
const [token, setToken] = useLocalStorage('auth-token', '')
// 用户可以看到，XSS *(跨站脚本攻击)* 可以读取
```
