> 源代码路径: `apps/web/src/lib/hooks/react-query/strapi/useActiveBanners.ts`

# useActiveBanners Hook Tutorial

## 1. 大白话讲讲这个hook的作用

`useActiveBanners` 是用来获取当前时间段内有效的营销 Banner 列表的 Hook。这些 Banner 是 SushiSwap CMS（内容管理系统）配置的活动横幅广告，会根据时间自动上下线。

简单来说：**就是获取"现在应该显示哪些首页横幅广告"——它自动过滤掉还没开始或已过期的 Banner，只返回当前时间段内有效的。**

---

## 2. 为什么需要封装该hook

### 原始问题
1. **时间过滤逻辑复杂**：需要把当前时间格式化为 ISO 字符串，然后构造 Strapi 的日期过滤查询参数
2. **多层嵌套数据结构**：Strapi CMS 返回的数据结构是 `{ data: [{ id, attributes: {...} }] }`，需要解嵌套
3. **图片字段也是嵌套的**：image 字段也是一个嵌套的 `data.attributes` *(Strapi的关联数据结构格式)* 结构，需要额外解一层
4. **缓存策略特殊**：Banner 数据变更不频繁，但不应该无限期缓存，需要合理的 gcTime

### 封装带来的好处
1. **时间自动过滤**：只用当前时间构造查询参数，API 自动返回符合条件的 Banner
2. **数据解嵌套**：把 Strapi 的多层嵌套结构拍平，返回直接可用的字段
3. **配置缓存**：1 小时 staleTime + 4 小时 gcTime，适合 Banner 这种低频变更内容
4. **关闭窗口焦点刷新**：Banner 不需要实时刷新，关闭 refetchOnWindowFocus 节省资源

---

## 3. 执行逻辑和数据流向

### 输入 (Params)
```typescript
// 无需传入参数，这是一个无参数的查询 hook
```

### 输出 (Return)
```typescript
{
  id: number
  dateFrom: string
  dateTo: string
  link?: string
  createdAt: string
  image: {
    name: string
    alternativeText: string
    url: string
    width: number
    height: number
  }
}[]
```

### 执行流程

```
1. useActiveBanners()
       |
       v
2. 构造当前时间 ISO 字符串: new Date().toISOString()
       |
       v
3. 构建 URL:
   https://sushi-strapi-cms.herokuapp.com/api/banners
   ?filters[dateTo][$gte]=${date}
   &filters[dateFrom][$lte]=${date}
   &populate=*
       |
       v
4. 调用 Strapi API 获取原始数据
       |
       v
5. 解析并转换数据:
   data.data.map(entry => ({
     ...entry.attributes,           // 解一层
     image: entry.attributes.image.data.attributes  // 再解一层
   }))
       |
       v
6. 返回拍平后的 Banner 数组
```

### Strapi 数据结构转换

```typescript
// 原始 API 返回:
{
  data: [{
    id: 1,
    attributes: {
      dateFrom: "2024-01-01",
      dateTo: "2024-12-31",
      image: {
        data: {
          attributes: {
            url: "https://...",
            name: "banner.jpg"
          }
        }
      }
    }
  }]
}

// 转换后:
[{
  dateFrom: "2024-01-01",
  dateTo: "2024-12-31",
  image: {
    url: "https://...",
    name: "banner.jpg"
  }
}]
```

---

## 4. 怎么用 AI 写这个 Hook

### 写代码时要注意什么

1. **无参数 Hook 的 queryKey 要包含端点**：既然没有任何传入参数，那 queryKey 里就要带上 CMS 的 URL 作为唯一标识。

2. **时间要用 ISO 格式**：Strapi 的日期过滤只认 ISO 8601 格式的时间字符串。

3. **用 populate=* 把关联数据一起拉回来**：image 这些关联字段要一次性加载，不然还要再发请求。

4. **CMS 数据嵌套深，要拍平**：Strapi 返回的数据结构套了好几层，Hook 负责把它转成外面能直接用的格式。

5. **Banner 不常变，缓存要长**：Banner 又不会每秒换一张，缓存时间设长一点合理。

### 有什么限制条件

1. **这是一个无参数的查询**：外面不用传任何参数，过滤逻辑全部在内部处理。

2. **CMS URL 是写死的**：`https://sushi-strapi-cms.herokuapp.com/api/banners`，改不了。

3. **依赖 Strapi 的返回格式**：假设 Strapi 返回的就是那种 `{ data: [{ id, attributes: {...} }] }` 的结构。

4. **时间过滤是在服务端做的**：API 会自动只返回符合时间条件的 Banner，不用前端再过滤一遍。

### 状态怎么管理

| 状态 | 怎么管 | 什么意思 |
|------|--------|---------|
| Banner 数据 | React Query 缓存 | 按 queryKey 缓存 |
| 加载/错误 | React Query 标准 | 自动处理 |
| 定时刷新 | staleTime: 1h | Banner 不常变，1小时后才过期 |
| 缓存清除 | gcTime: 4h | 4小时后才彻底清理 |
| 窗口焦点刷新 | refetchOnWindowFocus: false | 用户切回标签页不会重新请求 |

---

### 完整AI提示词模板

```
你是一个 React Query + CMS 集成专家。请为以下场景编写 Hook:

【场景描述】
需要从 Strapi CMS 获取当前时间段内有效的营销 Banner 列表。
Banner 有 dateFrom 和 dateTo 字段，只有当前时间在这两个日期之间时才应该显示。

【技术要求】
1. 使用 @tanstack/react-query useQuery
2. 调用 Strapi API: https://sushi-strapi-cms.herokuapp.com/api/banners
3. 构造时间过滤参数:
   ?filters[dateTo][$gte]=${当前ISO时间}
   &filters[dateFrom][$lte]=${当前ISO时间}
   &populate=*
4. 将嵌套的 Strapi 数据结构拍平

【数据结构转换】
原始: { data: [{ id, attributes: { dateFrom, dateTo, image: { data: { attributes: {...} } } } }] }
转换后: [{ dateFrom, dateTo, image: { url, name, ... } }]

【缓存配置】
- staleTime: 3600000 (1小时)
- gcTime: 14400000 (4小时)
- refetchOnWindowFocus: false

【无参数设计】
这个 Hook 不需要任何参数，因为是获取"当前有效"的 Banner

【最佳实践】
- queryKey 使用 CMS 端点 URL
- 时间使用 toISOString()
- populate=* 确保关联数据被加载
- 数据结构拍平在 Hook 内部完成

请输出完整代码。
```

---

## 5. 该 hook 的用法教学

### 基本使用示例

```tsx
import { useActiveBanners } from '@sushiswap/react-query'

function HomePageBanners() {
  const { data: banners, isLoading } = useActiveBanners()

  if (isLoading) return <BannerSkeleton />
  if (!banners?.length) return null

  return (
    <div className="banner-carousel">
      {banners.map((banner) => (
        <Banner key={banner.id} banner={banner} />
      ))}
    </div>
  )
}

function Banner({ banner }) {
  return (
    <a href={banner.link} target="_blank" rel="noopener noreferrer">
      <img
        src={banner.image.url}
        alt={banner.image.alternativeText || banner.image.name}
        width={banner.image.width}
        height={banner.image.height}
      />
    </a>
  )
}
```

### 常见使用场景

**场景1：首页横幅轮播**
```tsx
function BannerCarousel() {
  const { data: banners } = useActiveBanners()

  if (!banners?.length) return null

  return (
    <Carousel>
      {banners.map((banner, index) => (
        <Carousel.Item key={banner.id} activeIndex={index}>
          <BannerCard banner={banner} />
        </Carousel.Item>
      ))}
    </Carousel>
  )
}

function BannerCard({ banner }) {
  return (
    <div className="relative">
      <img src={banner.image.url} alt={banner.image.alternativeText} />
      {banner.link && (
        <a
          href={banner.link}
          className="absolute inset-0"
          target="_blank"
          rel="noopener noreferrer"
        />
      )}
    </div>
  )
}
```

**场景2：促销公告Banner**
```tsx
function PromoBanner() {
  const { data: banners } = useActiveBanners()

  // 只显示有链接的 banners（通常是促销类）
  const promoBanners = banners?.filter((b) => b.link)

  if (!promoBanners?.length) return null

  return (
    <AnnouncementBar>
      {promoBanners.map((banner) => (
        <PromoLink
          key={banner.id}
          href={banner.link}
          target="_blank"
          rel="noopener noreferrer"
        >
          <img
            src={banner.image.url}
            alt={banner.image.alternativeText}
            className="h-8 w-auto"
          />
        </PromoLink>
      ))}
    </AnnouncementBar>
  )
}
```

### Dos and Don'ts

**Do（推荐做法）：**
- ✅ 使用 `banner.image.url` 显示图片，而不是直接用原始数据结构
- ✅ 使用 `banner.image.alternativeText` 作为 img 的 alt 属性
- ✅ 检查 `banner.link` 是否存在，有链接的才可点击
- ✅ 使用返回的 `id` 作为 React key

**Don't（避免做法）：**
- ❌ 不要直接使用 Strapi 返回的原始嵌套结构
- ❌ 不要假设每个 banner 都有 link，有些可能只是展示
- ❌ 不要忽略 image 的 width/height，可以用于优化布局
- ❌ 不要频繁刷新这个数据，Banner 不会频繁变化

### 注意事项

1. **无参数设计**：这个 hook 不需要任何参数，因为它总是获取"当前有效"的 Banner

2. **返回数据已拍平**：不需要再处理 Strapi 的 `{ data: [{ attributes: {...} }] }` 嵌套结构

3. **4小时缓存**：Banner 数据会缓存4小时，不需要担心过时

4. **时间过滤在服务端**：只请求符合时间条件的 Banner，API 已经做好了过滤

5. **image 对象结构**：`{ name, alternativeText, url, width, height }` 都是直接可用的字段
