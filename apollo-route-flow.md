# Apollo 缓存与路由衔接分析

本文档梳理 Photoview 前端中 Apollo Client 缓存与 React Router 的衔接机制，涵盖**数据刷新、页面跳转、状态复用**三条核心处理路径。

---

## 一、整体架构总览

### 1.1 Provider 嵌套结构

文件：`ui/src/index.tsx`

```
ApolloProvider (client)
  └── BrowserRouter (basename)
        └── SidebarProvider
              └── App
                    ├── Routes        ← 路由系统
                    └── Messages      ← 订阅通知
```

**设计要点**：
- ApolloProvider 包裹 Router，确保所有路由组件均可访问 `useQuery/useMutation`
- Router 在外侧意味着路由变化不会重新创建 Apollo Client，**缓存跨路由持久化**
- 全局单例 `client` 贯穿整个应用生命周期

---

## 二、Apollo 缓存层深度分析

文件：`ui/src/apolloClient.ts`

### 2.1 Link 管道

```
ApolloLink.from([
  linkError,    ← 错误拦截：未授权清 Cookie、消息通知
  split(
    isSubscription ? wsLink : httpLink
  )
])
```

- **httpLink**: `credentials: 'include'`（携带 Cookie），URI `/api/graphql`
- **wsLink**: 懒加载 + 自动重连，`connectionParams` 注入 Bearer Token
- **linkError**: 捕获 `unauthorized` 错误 → 清除 Cookie → 触发重新登录流程

### 2.2 InMemoryCache 类型策略

缓存策略是 Apollo 与路由协作的核心——**路由切换时数据复用的基础**。

| 类型/字段 | keyArgs | 合并策略 | 说明 |
|----------|---------|---------|------|
| `SiteInfo` | - | `merge: true` | 全局单例，永远合并 |
| `MediaURL` | `['url']` | 默认 | 以 URL 作为缓存主键 |
| `Album.media` | `['onlyFavorites', 'order']` | `paginateCache` | 按筛选/排序分页合并 |
| `FaceGroup.imageFaces` | `[]` | `paginateCache` | 全参数分页合并 |
| `Query.myTimeline` | `['onlyFavorites']` | `paginateCache` | 仅收藏标记区分 |
| `Query.myFaceGroups` | `[]` | `paginateCache` | 全参数分页合并 |

### 2.3 `paginateCache` 自定义合并函数

```typescript
merge(existing, incoming, { args, fieldName }) {
  const merged = existing ? existing.slice(0) : []
  const { offset = 0 } = args.paginate
  for (let i = 0; i < incoming.length; ++i) {
    merged[offset + i] = incoming[i]
  }
  return merged
}
```

**与路由的关系**：
- 当路由切换至已加载过的页面时（如 Timeline → Album → Timeline），`existing` 不为空，Apollo 直接复用缓存数据，不发网络请求
- `keyArgs` 决定了**哪些参数变化会创建新的缓存条目**。例如 `onlyFavorites=true` 和 `onlyFavorites=false` 的数据彼此隔离，但切换到不同排序时缓存可能冲突

---

## 三、路由层架构

文件：`ui/src/components/routes/Routes.tsx`

### 3.1 路由表结构

```
/                    → IndexPage (根据 token 重定向 /timeline 或 /login)
/login               → LoginPage
/logout              → LogoutPage (清 Cookie → navigate /)
/initialSetup        → InitialSetupPage
/share/:token/*      → SharePageTokenRoute (无授权保护)
┌── 授权路由 (AuthorizedRoute) ──┐
│ /albums            → AlbumsPage
│ /album/:id         → AlbumPage (useParams: id)
│ /timeline          → TimelinePage
│ /places            → PlacesPage
│ /settings          → SettingsPage
│ /people            → PeoplePage
│   └── /people/:person → PersonPage (嵌套 Outlet)
└─────────────────────────────────┘
/photos              → Navigate → /timeline (兼容旧链接)
/*                   → NotFoundPage
```

### 3.2 懒加载与 Suspense

所有页面均通过 `React.lazy()` 异步加载，配合 `React.Suspense` 显示全局 Loader。

**路由切换时的行为**：
1. 点击导航 → React Router 切换匹配的路由
2. 懒加载组件开始 fetch 模块（首次访问）
3. Suspense 显示 Loading 页
4. 组件挂载 → 执行内部的 `useQuery`
5. Apollo 根据 `fetchPolicy`（默认 `cache-first`）决定走缓存或发请求
6. 数据到达 → 重新渲染

### 3.3 授权路由 AuthorizedRoute

文件：`ui/src/components/routes/AuthorizedRoute.tsx`

```typescript
const AuthorizedRoute = ({ children }) => {
  const token = authToken()  // 读 Cookie
  if (!token) return <Navigate to="/" />  // 无 token → 跳首页
  return <>{children}</>
}
```

**关键机制**：使用 **cookie 存储 token**（非 localStorage），配合 `credentials: 'include'` 实现 Http 请求自动携带。登录/登出直接操作 Cookie，不经过 React 状态。

---

## 四、页面跳转的处理路径

### 4.1 登录 → 首页 (全量刷新)

文件：`ui/src/Pages/LoginPage/loginUtilities.tsx:13-16`

```typescript
export function login(token: string) {
  saveTokenCookie(token)
  window.location.href = `${import.meta.env.BASE_URL}`  // 硬刷新！
}
```

**路径说明**：
```
登录成功 → saveTokenCookie (js-cookie 写入 14 天)
        → window.location.href = '/'
        → 浏览器整页重载
        → 新的 Apollo Client 实例创建 (无旧缓存)
        → IndexPage 检测到 token → Navigate /timeline
        → TimelinePage useQuery → 全新 HTTP 请求 (带 Cookie)
```

**设计选择**：使用硬刷新而非 `navigate()`，确保：
- Apollo 缓存彻底清空（避免旧用户数据泄漏）
- WebSocket 连接重新建立（带新 Token）
- Service Worker 等全局状态重置

### 4.2 登出 → 首页

文件：`ui/src/components/routes/Routes.tsx:151-155`

```typescript
const LogoutPage = ({ navigate }) => {
  clearTokenCookie()
  navigate('/')
  return null
}
```

**路径说明**：
```
点击 /logout → LogoutPage 挂载
            → clearTokenCookie() (Cookies.remove)
            → navigate('/') (SPA 路由，不刷新)
            → IndexPage 检测无 token → Navigate /login
            → LoginPage useQuery (INITIAL_SETUP_QUERY)
```

**注意**：登出使用 SPA 路由切换（非硬刷新），Apollo 缓存**仍然保留**在内存中。但 `linkError` 会在请求返回 `unauthorized` 时清 Cookie。

### 4.3 页面间 SPA 导航 (软跳转)

例：Timeline → Albums → Album 详情

```
点击 <NavLink to="/albums"> → React Router 匹配 AlbumsPage
                           → useQuery(getMyAlbums) 
                           → Apollo 查缓存：首次 miss → 发请求
                           → 缓存命中时直接返回
                           
点击 <Link to="/album/:id"> → useParams 读取 id
                           → useQuery(albumQuery, { id })
                           → keyArgs 匹配：{onlyFavorites, order}
                           → 缓存有则复用，无则请求
```

**缓存复用场景**：
- AlbumPage → PeoplePage → AlbumPage（同一 album id）：**缓存命中，不请求**
- AlbumPage 切换排序 → order 参数变化 → keyArgs 不匹配 → **新缓存条目 + 请求**
- AlbumPage 切换 `favorites=1` → onlyFavorites 变化 → **新缓存条目 + 请求**

---

## 五、数据刷新机制详解

### 5.1 无限滚动加载 (Scroll Pagination)

文件：`ui/src/hooks/useScrollPagination.ts`

这是跨多个页面复用的核心 Hook，与 Apollo `fetchMore` 深度绑定。

```
IntersectionObserver (rootMargin: '-100%')
        │
        ▼
检测到容器未进入视口 (isIntersecting == false)
        │
        ▼
调用 fetchMore({ variables: { offset: itemCount } })
        │
        ▼
Apollo 执行分页查询
        │
        ▼
paginateCache merge 函数按 offset 拼接到 existing
        │
        ▼
返回新数据 → data 变化 → observer 重新配置
```

**应用页面**：
| 页面 | 查询 | getItems |
|------|------|----------|
| AlbumPage | albumQuery | `data.album.media` |
| TimelineGallery | myTimeline | `data.myTimeline` |
| PeoplePage | myFaces | `data.myFaceGroups` |
| SingleFaceGroup | singleFaceGroup | `data.faceGroup.imageFaces` |

### 5.2 URL 参数驱动的查询刷新

文件：`ui/src/hooks/useURLParameters.ts`

```typescript
const useURLParameters = () => {
  const [urlString, setUrlString] = useState(document.location.href)
  const url = new URL(urlString)
  const params = new URLSearchParams(url.search)

  const setParam = (key, value) => {
    // 更新 URLSearchParams
    history.replaceState({}, '', url.pathname + '?' + params.toString())
    setUrlString(document.location.href)  // 触发组件重渲染
  }
}
```

**工作链路**（以 AlbumPage 切换收藏为例）：

```
用户点击"仅显示收藏"
        │
        ▼
toggleFavorites(true/false)
        │
        ├─ 检查 refetchNeeded 标志
        │       ├─ 需要刷新 → refetch({ onlyFavorites }) → 网络请求 → setParam
        │       └─ 无需刷新 → 直接 setParam
        │
        ▼
setParam('favorites', '1')
        │
        ▼
history.replaceState → 更新 URL
        │
        ▼
setUrlString → useState 更新 → 组件重渲染
        │
        ▼
useQuery variables 变化 (onlyFavorites 从 false → true)
        │
        ▼
Apollo keyArgs 匹配 (['onlyFavorites', 'order'])
        │
        ├─ 缓存命中 → 直接返回
        └─ 缓存 miss → 新请求 + 新缓存条目
```

### 5.3 强制刷新策略

#### 5.3.1 Timeline 日期筛选：resetStore

文件：`ui/src/components/timelineGallery/TimelineGallery.tsx:125-137`

```typescript
useEffect(() => {
  ;(async () => {
    await client.resetStore()           // 清空全部缓存！
    await refetch({                     // 重新拉取
      onlyFavorites,
      fromDate: `${parseInt(filterDate) + 1}-01-01T00:00:00Z`,
      offset: 0, limit: 200,
    })
  })()
}, [filterDate])
```

**为何用 resetStore**：
- `fromDate` 参数不在 `keyArgs` 中，Apollo 无法自动区分不同年份的缓存
- 旧年份的 `myTimeline` 数据不应干扰新年份的分页合并
- resetStore 触发所有活跃 query 重新执行（副作用：其他页面缓存也清空）

#### 5.3.2 Album 收藏切换：手动 refetchNeeded 标记

文件：`ui/src/Pages/AlbumPage/AlbumPage.tsx:33-91`

```typescript
let refetchNeededAll = false       // 模块级变量
let refetchNeededFavorites = false

// 收藏按钮点击后：
onFavorite={() => (refetchNeededAll = refetchNeededFavorites = true)}

// 切换收藏显示时：
toggleFavorites = (onlyFavorites) => {
  if ((refetchNeededAll && !onlyFavorites) || 
      (refetchNeededFavorites && onlyFavorites)) {
    refetch({ id: albumId, onlyFavorites })  // 强制刷新
      .then(() => { 标记复位; setParam })
  } else {
    setParam(onlyFavorites)  // 走缓存
  }
}
```

**设计意图**：
- 收藏操作使用 `optimisticResponse` 乐观更新缓存（即时 UI）
- 但乐观更新只更新当前视图的 Media 条目，不重新筛选「仅显示收藏」列表
- 因此切换列表视图时需要重新拉取以保证正确性

#### 5.3.3 Places 地图：cache-first 策略

文件：`ui/src/Pages/PlacesPage/PlacesPage.tsx:34-36`

```typescript
useQuery<mediaGeoJson>(MAPBOX_DATA_QUERY, {
  fetchPolicy: 'cache-first',   // 显式声明
})
```

地图 GeoJSON 数据相对静态，使用最强缓存策略：**一次加载，永久复用**（除非 resetStore）。

### 5.4 Mutation 后的缓存刷新

#### 5.4.1 乐观响应 (Optimistic Response)

文件：`ui/src/components/photoGallery/photoGalleryMutations.ts`

```typescript
toggleFavoriteAction = ({ media, markFavorite }) => {
  return markFavorite({
    variables: { mediaId: media.id, favorite: !media.favorite },
    optimisticResponse: {
      favoriteMedia: {
        id: media.id,
        favorite: !media.favorite,
        __typename: 'Media',
      },
    },
  })
}
```

**路径**：
```
点击收藏 → markFavorite mutation
        → 立即写入 optimisticResponse 到缓存
        → UI 即时反馈（星标切换）
        → 服务器响应返回：
             ├─ 成功：Apollo 合并真实结果（通常与乐观一致）
             └─ 失败：Apollo 回滚乐观更新
```

由于 `Media` 类型按 `id` 标准化存储，乐观更新会**同步更新所有引用该 Media 的查询**（Album/Timeline/People 等页面共享）。

#### 5.4.2 refetchQueries 显式刷新

在需要**重新拉取整个列表**的场景（不仅仅是单条更新），使用 `refetchQueries`：

| 文件 | 操作 | 刷新查询 |
|------|------|---------|
| `Sharing.tsx` | 添加/删除/保护分享 | `SHARE_ALBUM_QUERY` / `SHARE_PHOTO_QUERY` |
| `MergeFaceGroupsModal.tsx` | 合并人脸组 | 外部传入 `refetchQueries` |
| `MoveImageFacesModal.tsx` | 移动人脸图片 | `SINGLE_FACE_GROUP` + `MY_FACES_QUERY` |
| `DetachImageFacesModal.tsx` | 分离人脸 | 同上 |
| `MediaSidebarPeople.tsx` | 人脸操作 | 同上 |
| `EditUserRowRootPaths.tsx` | 用户路径编辑 | `settingsUsersQuery` |

---

## 六、Present Mode（全屏看图）与浏览器历史

这是 Apollo + 路由衔接中最精巧的部分——**不改变 URL path，仅使用 history state**。

### 6.1 核心 Hook：urlPresentModeSetupHook

文件：`ui/src/components/photoGallery/mediaGalleryReducer.tsx:76-100`

```typescript
export const urlPresentModeSetupHook = ({ dispatchMedia, openPresentMode }) => {
  useEffect(() => {
    // 1. 监听浏览器后退/前进
    const urlChangeListener = (event: MediaGalleryPopStateEvent) => {
      if (event.state.presenting === true) {
        openPresentMode(event)
      } else {
        dispatchMedia({ type: 'closePresentMode' })
      }
    }
    window.addEventListener('popstate', urlChangeListener)
    
    // 2. 初始化 state
    history.replaceState({ presenting: false }, '')

    return () => window.removeEventListener('popstate', urlChangeListener)
  }, [])
}
```

### 6.2 打开 Present Mode

```typescript
export const openPresentModeAction = ({ dispatchMedia, activeIndex }) => {
  dispatchMedia({ type: 'openPresentMode', activeIndex })
  history.pushState({ presenting: true, activeIndex }, '')  // 推入历史
}
```

### 6.3 关闭 Present Mode

```typescript
export const closePresentModeAction = ({ dispatchMedia }) => {
  dispatchMedia({ type: 'closePresentMode' })
  history.back()  // 弹出历史，触发 popstate → 确认关闭
}
```

### 6.4 完整交互链路

```
用户点击缩略图
    │
    ▼
openPresentModeAction
    ├─ dispatch: presenting=true, activeIndex=5
    └─ history.pushState({ presenting: true, activeIndex: 5 })
    │
    ▼
PresentView 渲染（全屏黑色遮罩）
    │
    ├─ 用户按 ESC 或点击 ✕
    │       │
    │       ▼
    │   closePresentModeAction
    │       ├─ dispatch: presenting=false
    │       └─ history.back() → 触发 popstate
    │            │
    │            ▼
    │        urlChangeListener 检测 state.presenting = false
    │            └─ dispatch: closePresentMode (幂等确认)
    │
    └─ 用户点击浏览器后退
            │
            ▼
        popstate 事件 → state.presenting = false
            └─ dispatch: closePresentMode → PresentView 卸载
```

**与 Apollo 的协同**：
- PresentView 内切换上下张仅操作本地 reducer state，不触发 Apollo 查询
- 但 `activeMedia` 对象来自 Apollo 缓存的引用（非深拷贝）
- 因此当 Media 的 `favorite` 被乐观更新后，PresentView 中显示的星标状态**自动同步**

---

## 七、跨页面状态复用的具体场景

### 7.1 Media 对象跨页面共享

Apollo 缓存按 `__typename:id` 标准化存储 `Media` 对象。以下场景所有页面共享同一份数据：

```
Timeline (myTimeline query)
    └─ Media{id: 42, favorite: false}
           ↑ 通过缓存引用共享 ↓
Album /album/7 (albumQuery)
    └─ Media{id: 42, favorite: false}
           ↑ 通过缓存引用共享 ↓
People /people/3 (singleFaceGroup → imageFaces[].media)
    └─ Media{id: 42, favorite: false}
```

**效果**：在任意页面收藏该图片，其他页面的 UI 自动更新（无需手动刷新）。

### 7.2 Album 对象跨页面访问

- AlbumsPage (`getMyAlbums`) 加载的 Album 缩略图信息
- AlbumPage (`albumQuery`) 加载该 Album 的详细信息
- 两者通过 `Album:id` 标准化合并，第二次访问可部分复用

### 7.3 SiteInfo 全局单例

`typePolicies` 中 `SiteInfo: { merge: true }` 确保：
- LoginPage 的 `INITIAL_SETUP_QUERY` 拉取的 `initialSetup` 字段
- Layout 子查询的 `faceDetectionEnabled` 字段
- 主菜单查询的其他字段

**所有查询写入同一个 SiteInfo 缓存对象，字段逐步合并，互不覆盖**。

---

## 八、Sidebar 延迟加载与缓存联动

### 8.1 SidebarContext 架构

文件：`ui/src/components/sidebar/Sidebar.tsx`

Sidebar 是一个**全局单例浮层**，通过 Context 跨组件树共享状态：

```typescript
SidebarContext = {
  updateSidebar(content)    // 设置内容，null 则关闭
  setPinned(boolean)        // 固定/取消固定
  content: ReactNode        // 当前内容
  pinned: boolean           // 固定状态
}
```

Sidebar 内容与路由无关——它不随 URL 变化而自动重置，仅在 `updateSidebar(null)` 时关闭。

### 8.2 MediaSidebar 的两级数据加载

文件：`ui/src/components/sidebar/MediaSidebar/MediaSidebar.tsx`

MediaSidebar 展示了 Apollo 缓存与按需查询的精巧配合：

```
用户点击缩略图 (MediaGallery)
        │
        ▼
selectImage(index) + updateSidebar(<MediaSidebar media={mediaState.media[index]} />)
        │
        ▼
MediaSidebar 接收 media 对象 (来自列表查询的缓存引用)
        │
        ├─ 无 authToken → 直接用列表数据渲染 <SidebarContent> (基础信息)
        │
        └─ 有 authToken → useLazyQuery(SIDEBAR_MEDIA_QUERY)
                │
                ├─ loading/data==null → 先用列表数据渲染
                │                       (缩略图/标题等字段已在缓存中)
                └─ data 到达 → 用完整数据渲染
                    (exif/faces/coordinates/videoMetadata 等)
```

**设计要点**：
- 列表查询只拉取 `id, type, blurhash, thumbnail, highRes, videoWeb, favorite`（`MediaGalleryFields` fragment）
- Sidebar 查询拉取完整详情（exif、faces、album path、downloads 等）
- 两次查询的结果通过 `Media:id` 标准化**自动合并到同一缓存对象**
- 因此列表页的其他组件（如 PresentView）也能读取到 Sidebar 查询写入的额外字段

### 8.3 AlbumSidebar 与路由跳转联动

文件：`ui/src/components/sidebar/AlbumSidebar.tsx`、`ui/src/components/sidebar/MediaSidebar/MediaSidebar.tsx:184-200`

当用户在 MediaSidebar 的 album path 中点击相册链接时：

```
<Link to={`/album/${album.id}`} onClick={() => updateSidebar(null)}>
    {album.title}
</Link>
```

先关闭 Sidebar，再由 React Router 处理路由跳转。这里 `updateSidebar(null)` 是**手动清理**，因为 Sidebar 不会随路由自动重置。

---

## 九、Share 页面独立数据流

文件：`ui/src/Pages/SharePage/SharePage.tsx`

Share 页面是整个应用中**唯一不经过 AuthorizedRoute** 的功能页面，拥有独立的数据鉴权链路。

### 9.1 三层数据获取

```
TokenRoute (入口)
    │
    ▼ useQuery(VALIDATE_TOKEN_PASSWORD_QUERY, { token, password })
    │
    ├─ shareTokenValidatePassword == false → PasswordProtectedShare
    │                                         用户输入密码 → refetch({ password })
    │
    ├─ loading → 显示 Loading
    │
    └─ 验证通过 → AuthorizedTokenRoute
                      │
                      ▼ useQuery(SHARE_TOKEN_QUERY, { token, password })
                      │
                      ├─ shareToken.album 存在 → 嵌套 Routes
                      │     ├─ /share/:token/:subAlbum → AlbumSharePage(albumID=subAlbum)
                      │     └─ /share/:token (index)    → AlbumSharePage(albumID=album.id)
                      │
                      └─ shareToken.media 存在 → MediaSharePage
```

### 9.2 Share 页面与主应用的缓存隔离

**关键区别**：

| 维度 | 主应用页面 | Share 页面 |
|------|-----------|-----------|
| 鉴权方式 | Cookie Bearer Token | URL 参数 token + password |
| 查询参数 | 无需额外凭据 | 每个查询传入 `tokenCredentials` 或 `token/password` |
| 图片鉴权 | `credentials: 'include'` (Cookie) | URL 注入 `?token=xxx` 参数 |
| 路由保护 | AuthorizedRoute (Cookie 检查) | 无保护，验证在 GraphQL 层 |

### 9.3 ProtectedMedia 的 URL 重写

文件：`ui/src/components/photoGallery/ProtectedMedia.tsx:13-25`

```typescript
const getProtectedUrl = (url) => {
  const imgUrl = new URL(url, location.origin)
  const tokenRegex = location.pathname.match(/^\/share\/([\d\w]+)(\/?.*)$/)
  if (tokenRegex) {
    imgUrl.searchParams.set('token', tokenRegex[1])  // 注入 token
  }
  return imgUrl.href
}
```

这是 Share 页面鉴权的**最后一环**：后端返回的图片 URL 本身不带鉴权参数，前端根据当前路径是否匹配 `/share/:token`，动态注入 token 作为查询参数，使 `<img>` 和 `<video>` 标签能通过后端的 token 校验。

---

## 十、订阅通知 (WebSocket) 与缓存

文件：`ui/src/components/messages/SubscriptionsHook.ts`

```
WebSocketLink (全局单例)
    │
    ▼
NOTIFICATION_SUBSCRIPTION 监听
    │
    ├─ 扫描进度通知 → 更新 Messages 组件 state (与 Apollo 缓存无关)
    ├─ 扫描完成 Close 通知 → 移除消息
    └─ 通知类型不直接修改 Apollo 缓存（由用户手动刷新或下次进入页面触发）
```

**注意**：当前实现中，后端扫描完成后**不会主动刷新 Apollo 缓存中的媒体/相册列表**。用户需要手动刷新页面或重新进入路由才能看到新扫描的照片。

---

## 十一、路由切换时的全局副作用

文件：`ui/src/App.tsx:15-19`

```typescript
useEffect(() => {
  window.scrollTo(0, 0)                    // 回到顶部
  if (document.activeElement != document.body)
    (document.activeElement as HTMLInputElement).blur()  // 释放焦点
}, [pathname])
```

这段代码监听路由 pathname 变化，确保：
1. 每次路由切换后页面滚回顶部（避免无限滚动页面保留旧滚动位置）
2. 释放输入框焦点（避免 SearchBar 在切换页面后仍处于聚焦状态）

**与 Apollo 的间接关系**：滚动重置会影响 `useScrollPagination` 的 `IntersectionObserver` 重新触发，导致首页数据可能需要重新检查是否已加载完毕。

---

## 十二、SearchBar 的延迟查询

文件：`ui/src/components/header/Searchbar.tsx`

```
用户输入 → debounce(250ms) → useLazyQuery(SEARCH_QUERY)
                                  │
                                  ▼
                          搜索结果 (albums + media)
                                  │
                  ┌───────────────┼───────────────┐
                  ▼                               ▼
          AlbumRow → <NavLink to="/album/:id">   PhotoRow → <NavLink to="/album/:albumId">
```

**与路由和缓存的协同**：
- `useLazyQuery` 不随组件挂载自动执行，仅在用户输入后触发
- 路由变化时（`useEffect([location])`）清空搜索框和结果
- 搜索结果的 Media/Album 对象会写入 Apollo 缓存（按 `__typename:id` 标准化）
- 用户点击搜索结果跳转到 AlbumPage 时，如果搜索结果中已包含该 Album 的部分数据，Apollo 会部分复用

---

## 十三、关键代码位置速查

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|-------|
| Apollo 客户端初始化 | `ui/src/apolloClient.ts` | 190-196 |
| 自定义分页缓存合并 | `ui/src/apolloClient.ts` | 144-159 |
| 类型策略配置 | `ui/src/apolloClient.ts` | 161-188 |
| 路由表 | `ui/src/components/routes/Routes.tsx` | 60-124 |
| 授权路由 | `ui/src/components/routes/AuthorizedRoute.tsx` | 35-43 |
| 登录硬刷新 | `ui/src/Pages/LoginPage/loginUtilities.tsx` | 13-16 |
| 无限滚动分页 Hook | `ui/src/hooks/useScrollPagination.ts` | 全文 |
| URL 参数 Hook | `ui/src/hooks/useURLParameters.ts` | 全文 |
| Present Mode 历史整合 | `ui/src/components/photoGallery/mediaGalleryReducer.tsx` | 76-127 |
| Timeline 强制 resetStore | `ui/src/components/timelineGallery/TimelineGallery.tsx` | 125-137 |
| Album 收藏标记手动刷新 | `ui/src/Pages/AlbumPage/AlbumPage.tsx` | 33-91 |
| 收藏乐观更新 | `ui/src/components/photoGallery/photoGalleryMutations.ts` | 23-43 |
| 错误处理（未授权清 Cookie） | `ui/src/apolloClient.ts` | 97-106 |
| Sidebar 全局 Context | `ui/src/components/sidebar/Sidebar.tsx` | 全文 |
| MediaSidebar 两级加载 | `ui/src/components/sidebar/MediaSidebar/MediaSidebar.tsx` | 277-306 |
| Sidebar 路由跳转清理 | `ui/src/components/sidebar/MediaSidebar/MediaSidebar.tsx` | 184-200 |
| Share 页面 Token 验证 | `ui/src/Pages/SharePage/SharePage.tsx` | 154-202 |
| ProtectedMedia URL 重写 | `ui/src/components/photoGallery/ProtectedMedia.tsx` | 13-25 |
| 路由切换滚动重置 | `ui/src/App.tsx` | 15-19 |
| SearchBar 延迟查询 | `ui/src/components/header/Searchbar.tsx` | 46-186 |

---

## 十四、潜在问题与设计权衡

| 现象 | 原因 | 影响 |
|------|------|------|
| 登出不清缓存 | `navigate()` 而非硬刷新，`client` 不重建 | 内存中残留旧数据，但 Cookie 清除后请求 401 会触发清 Cookie |
| Timeline 切年份清全部缓存 | `client.resetStore()` 是全局操作 | 其他页面已加载数据也被清空，需重新请求 |
| 扫描后列表不更新 | Subscription 不写缓存 | 用户需手动刷新页面 |
| Album 收藏切换手动标记 | 模块级变量 `refetchNeededXxx` | 多个 AlbumPage 实例间标记冲突（但实际同一时间只打开一个） |
| Present Mode 不写入 URL | 仅使用 history state | 刷新页面后 Present Mode 丢失，无法分享链接 |
| Sidebar 不随路由自动关闭 | SidebarContext 是全局状态 | 必须在每个跳转入口手动 `updateSidebar(null)`，遗漏会导致旧内容残留 |
| SearchBar 结果无清除缓存策略 | 搜索结果写入 Apollo 标准化缓存 | 低频搜索的 Album/Media 条目永久留在缓存中，可能占用内存 |
| Share 页面与主应用共享缓存 | 共用同一个 Apollo Client | Share 页面查询的 Album/Media 对象会与主应用查询合并，可能产生字段覆盖（如 Share 查询含 downloads 字段而主查询不含） |
