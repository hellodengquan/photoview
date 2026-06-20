# Photoview 分享链接生命周期与匿名下载授权协作机制

## 一、核心数据模型

### ShareToken 模型定义
文件：`api/graphql/models/share_token.go:7-22`

```go
type ShareToken struct {
    Model                    // 包含 ID, CreatedAt, UpdatedAt
    Value    string          // token 值，主键索引
    OwnerID  int             // 创建者用户 ID
    Owner    User            // 关联用户 (CASCADE 删除)
    Expire   *time.Time      // 过期时间（可为空 = 永不过期）
    Password *string         // bcrypt 哈希后的密码（可为空）
    AlbumID  *int            // 关联相册 ID
    Album    *Album          // 关联相册 (CASCADE 删除)
    MediaID  *int            // 关联媒体 ID
    Media    *Media          // 关联媒体 (CASCADE 删除)
}
```

**关键设计要点：**
- `AlbumID` 与 `MediaID` 互斥存在，token 只能绑定其中一种资源
- `Expire` 和 `Password` 均可为空，分别表示「永不过期」和「无密码」
- 所有关联对象均设置 `CASCADE` 删除，保证数据一致性

---

## 二、分享 Token 生命周期管理

### 2.1 创建流程

后端通过两个 GraphQL mutation 创建分享：
- `shareMedia(mediaId, expire, password)` → 绑定单张媒体
- `shareAlbum(albumId, expire, password)` → 绑定整个相册

**创建流程（以 AddMediaShare 为例）：**
文件：`api/graphql/models/actions/share_token_actions.go:15-59`

```
1. 权限校验：验证当前用户是否拥有该媒体所在相册的所有权
   └─ 通过 user_albums 关联表检查
2. 密码处理：若设置了密码，用 bcrypt(12) 生成哈希
3. 生成 Token：utils.GenerateToken() 生成随机值
4. 构造对象：设置 Value, OwnerID, Expire, Password, MediaID/AlbumID
5. 写入数据库：db.Create(&shareToken)
6. 返回对象：包含完整 token 信息给前端
```

### 2.2 Token 修改操作

文件：`api/graphql/models/actions/share_token_actions.go`

| 操作 | 函数 | 说明 |
|------|------|------|
| 删除 | `DeleteShareToken` | 仅 owner 或 admin 可删除 |
| 设置密码 | `ProtectShareToken` | 传 null 可清除密码 |
| 设置过期 | `SetExpireShareToken` | 传 null 可设为永不过期 |

**owner 校验逻辑** (`getUserToken` 函数，第 167-184 行)：
```go
query = "Owner.id = ? OR Owner.admin = TRUE"
```
即 token 创建者本人或系统管理员均可操作。

### 2.3 过期记录的清理机制（无后台定期任务）

**结论：share_tokens 表中的过期记录**不会被后台定期任务自动清理。

**代码证据链：**

1. **定期扫描器只做媒体扫描**  
   文件：`api/scanner/periodic_scanner/periodic_scanner.go:159-163`
   ```go
   case <-ticker.C:
       log.Info(nil, "Scan interval runner: Starting periodic scan")
       if err := ps.scannerQueue.AddAllToQueue(); err != nil {
           // ...
       }
   ```
   periodic_scanner 的唯一职责是触发媒体扫描队列，与 ShareToken 无关。

2. **扫描任务列表不含 Token 清理**  
   文件：`api/scanner/scanner_tasks/scanner_tasks.go:15-27`
   ```go
   var allTasks []scanner_task.ScannerTask = []scanner_task.ScannerTask{
       NotificationTask{},
       IgnorefileTask{},
       processing_tasks.CounterpartFilesTask{},
       processing_tasks.SidecarTask{},
       processing_tasks.ProcessPhotoTask{},
       processing_tasks.ProcessVideoTask{},
       FaceDetectionTask{},
       BlurhashTask{},
       ExifTask{},
       VideoMetadataTask{},
       cleanup_tasks.MediaCleanupTask{},  // 仅清理磁盘缺失的媒体
   }
   ```
   `MediaCleanupTask` 只负责清理「文件系统中已删除的媒体」，不涉及 ShareToken。

3. **服务启动时无清理初始化**  
   文件：`api/server.go:62-68`
   ```go
   if err := scanner_queue.InitializeScannerQueue(db); err != nil { ... }
   if err := periodic_scanner.InitializePeriodicScanner(db); err != nil { ... }
   if err := face_detection.InitializeFaceDetector(db); err != nil { ... }
   ```
   server.go 初始化的三个后台组件均与 ShareToken 清理无关。

4. **无任何 cron / scheduler / 定时清理代码**  
   全库搜索 `cron`、`scheduler`、`expired.*delete` 等关键词，均无 ShareToken 相关清理逻辑。

**过期记录的删除仅在以下场景发生：**
- 用户主动调用 `deleteShareToken` mutation
- 关联的 Media 或 Album 被删除（CASCADE 级联删除）
- 关联的 Owner 用户被删除（CASCADE 级联删除）

---

## 三、续期接口的权限校验深度剖析

### 3.1 setExpireShareToken 的完整权限链

`setExpireShareToken` mutation 有**两层权限校验**，不需要 owner 重新登录确认，只要当前会话有效即可操作。

**校验链：**

```
GraphQL 请求 → @isAuthorized 指令（第一层）
    ↓
resolver 函数 SetExpireShareToken（第二层）
    ↓
actions.SetExpireShareToken()
    ↓
getUserToken() → "Owner.id = ? OR Owner.admin = TRUE"
```

### 3.2 第一层：@isAuthorized 指令（登录态校验）

文件：`api/graphql/directive.go:20-27`

```go
func IsAuthorized(ctx context.Context, obj interface{}, next graphql.Resolver) (res interface{}, err error) {
    user := auth.UserFromContext(ctx)
    if user == nil {
        return nil, auth.ErrUnauthorized
    }
    return next(ctx)
}
```

- 作用：确保请求方是已登录用户
- 校验方式：从 HTTP 请求的 `auth-token` Cookie 中读取 token，通过 dataloader 查用户
- 只要 Cookie 中的 auth-token 有效，就能通过这一层
- **不需要重新输入密码或二次确认**

### 3.3 第二层：getUserToken 所有权校验

文件：`api/graphql/models/actions/share_token_actions.go:167-184`

```go
func getUserToken(db *gorm.DB, userID int, tokenValue string) (*models.ShareToken, error) {
    var query string
    if drivers.POSTGRES.MatchDatabase(db) {
        query = "\"Owner\".id = ? OR \"Owner\".admin = TRUE"
    } else {
        query = "Owner.id = ? OR Owner.admin = TRUE"
    }

    var token models.ShareToken
    err := db.Where("share_tokens.value = ?", tokenValue).Joins("Owner").Where(query, userID).First(&token).Error
    // ...
}
```

**两个分支的权限：**

| 分支 | 条件 | 能否操作 |
|------|------|----------|
| **Owner 分支** | 当前用户 ID == token 的 OwnerID | ✅ 可以（token 创建者本人） |
| **Admin 分支** | 当前用户的 admin = TRUE | ✅ 可以（系统管理员越权操作） |

### 3.4 续期操作的完整调用栈

```
前端侧边栏 Sharing.tsx
  MorePopoverSectionExpiration 组件
    用户选择新日期 → 点击确认
      ↓
    useMutation(SET_EXPIRE_MUTATION)
      variables: { token, expire: dayjs(date).endOf('day').format()+'Z' }
      ↓
    GraphQL 请求（携带 auth-token Cookie）
      ↓
后端
  @isAuthorized 指令 → 确认已登录
  SetExpireShareToken resolver → 调 actions.SetExpireShareToken
    getUserToken → 确认是 owner 或 admin
    token.Expire = expire
    db.Save(&token) → 更新数据库
      ↓
    refetchQueries → 刷新侧边栏 shares 列表
```

**关键结论：**
- ✗ **不需要**重新登录确认
- ✗ **不需要**输入密码二次验证
- ✓ 只要 auth-token Cookie 有效且是 owner/admin，即可直接续期
- ✓ 续期后原 token 值不变，分享链接不变，匿名用户刷新页面即可用新过期时间

---

## 四、分享页面访问流程（前端路由）

### 4.1 路由入口
文件：`ui/src/components/routes/Routes.tsx:77-80`

```
/share/:token/*  →  TokenRoute (懒加载)
```

### 4.2 两阶段验证机制

文件：`ui/src/Pages/SharePage/SharePage.tsx`

**阶段一：Token 密码有效性预检（TokenRoute 组件）**
```
TokenRoute
├─ 调用 VALIDATE_TOKEN_PASSWORD_QUERY
│   ├─ 参数：{ token, password: 从 cookie 读取 }
│   └─ 后端: shareTokenValidatePassword resolver
│
├─ 返回值处理：
│   ├─ error = "share not found" → 显示「分享不存在」
│   ├─ data = false → 显示密码输入页 (PasswordProtectedShare)
│   └─ data = true → 进入 AuthorizedTokenRoute
│
└─ PasswordProtectedShare 组件
    ├─ 用户输入密码
    ├─ saveSharePassword(token, password) → 存 cookie
    └─ refetch() → 用新密码重新验证
```

**阶段二：加载分享内容（AuthorizedTokenRoute 组件）**
```
AuthorizedTokenRoute
├─ 调用 SHARE_TOKEN_QUERY
│   └─ 参数：{ token, password }
│
├─ 返回 shareToken 对象，包含 album 或 media
│
├─ 若有 album → 渲染 AlbumSharePage
│   └─ 支持子相册路由：/share/:token/:subAlbum
│
└─ 若有 media → 渲染 MediaSharePage
```

---

## 五、匿名下载授权机制

### 5.1 三条下载/访问路由

| 路由 | 文件 | 用途 |
|------|------|------|
| `GET /photo/{name}` | `api/routes/photos.go` | 单张图片加载（缩略图、高清图） |
| `GET /video/{name}` | `api/routes/videos.go` | 视频播放（含转码后 web 格式） |
| `GET /download/album/{id}/{purpose}` | `api/routes/downloads.go` | 批量打包下载相册为 ZIP |

### 5.2 双轨认证架构

文件：`api/routes/authenticate_routes.go:19-69`

```
authenticateMedia / authenticateAlbum
│
├─ 【已登录用户路径】auth.UserFromContext(ctx) != nil
│   └─ 检查 user.OwnsAlbum(db, album)
│       ├─ true → 通过
│       └─ false → 403 Forbidden
│
└─ 【匿名用户路径】进入 shareTokenFromRequest()
```

### 5.3 ShareToken 请求认证核心逻辑

文件：`api/routes/authenticate_routes.go:71-155`

```
shareTokenFromRequest(db, r, mediaID, albumID)
│
├─ Step 1: 从 URL Query 提取 token 参数
│   token := r.URL.Query().Get("token")
│   为空 → 403 "share token not provided"
│
├─ Step 2: 数据库查找 token 记录
│   db.Where("value = ?", token).First(&shareToken)
│   找不到 → 403 "invalid share token"
│
├─ Step 3: 过期时间检查 ★（下载路由侧）
│   if shareToken.Expire != nil && time.Now().UTC().After(expire.UTC())
│   过期 → 403 "invalid share token"
│
├─ Step 4: 密码 Cookie 校验
│   if shareToken.Password != nil
│   ├─ 读取 Cookie: share-token-pw-{token}
│   │   不存在 → 403
│   └─ bcrypt.CompareHashAndPassword(hash, cookie值)
│       不匹配 → 403
│
├─ Step 5: 资源绑定匹配
│   ├─ Album 型 token
│   │   ├─ 精确匹配：*albumID == *shareToken.AlbumID
│   │   └─ 递归子相册检查：WITH RECURSIVE child_albums ...
│   │       即分享父相册可访问所有子相册
│   │
│   └─ Media 型 token
│       └─ 精确匹配：*mediaID == *shareToken.MediaID
│
└─ 全部通过 → 返回 true
```

### 5.4 前端 URL Token 注入机制

文件：`ui/src/components/photoGallery/ProtectedMedia.tsx:13-25`

```javascript
const getProtectedUrl = (url?: string) => {
  if (url == undefined) return undefined
  const imgUrl = new URL(url, location.origin)
  
  // 从当前页面路径正则提取 token
  const tokenRegex = location.pathname.match(/^\/share\/([\d\w]+)(\/?.*)$/)
  if (tokenRegex) {
    const token = tokenRegex[1]
    imgUrl.searchParams.set('token', token)  // 附加 ?token=xxx
  }
  return imgUrl.href
}
```

**注入时机：**
- `ProtectedImage` 组件：所有图片 src 属性（缩略图、高清图）
- `ProtectedVideo` 组件：video poster 和 source src
- AlbumSharePage 中相册内所有媒体 URL
- 下载链接（downloads 字段中的 url）

**跨域凭据：**
```
crossOrigin="use-credentials"  // 确保 Cookie 随请求发送
```

---

## 六、密码 Cookie 机制

### 6.1 Cookie 存储（前端）

文件：`ui/src/helpers/authentication.ts:31-47`

```javascript
SHARE_TOKEN_COOKIE_NAME = `share-token-pw-${shareToken}`

saveSharePassword(token, password)
  → Cookies.set(cookieName, password, { path: '/', sameSite: 'Lax' })
  // 注意：未设置 expires，即 session cookie，浏览器关闭失效

getSharePassword(token)    // 读取
clearSharePassword(token)  // 删除
```

### 6.2 Cookie 读取（后端）

文件：`api/routes/authenticate_routes.go:95-113`

```go
cookieName := fmt.Sprintf("share-token-pw-%s", shareToken.Value)
tokenPasswordCookie, err := r.Cookie(cookieName)
tokenPassword := tokenPasswordCookie.Value

bcrypt.CompareHashAndPassword(
    []byte(*shareToken.Password),  // 数据库存的哈希
    []byte(tokenPassword),         // Cookie 中的明文
)
```

**安全设计：**
- 数据库存 bcrypt 哈希（cost=12）
- Cookie 存明文密码（仅 HTTPS 环境下可接受）
- sameSite=Lax 防止 CSRF

---

## 七、Token 过期检查的「双向触发」机制

### 7.1 两处过期检查点

| 检查位置 | 文件 | 场景 | 时间处理 |
|----------|------|------|----------|
| **GraphQL Resolver** | `api/graphql/resolvers/share_token.go:84-98` | 页面访问时获取 shareToken 元数据 | **构造 fakeTime**：截断纳秒，UTC，客户端本地时间视为 UTC |
| **HTTP 路由** | `api/routes/authenticate_routes.go:89-92` | 图片/视频/下载请求时 | **直接 UTC 比较**：time.Now().UTC() vs expire.UTC() |

### 7.2 为什么有两处检查？——协作流程

```
用户访问 /share/abc123
    │
    ▼
[GraphQL 路径：过期检查点 A]
shareToken(credentials) resolver
    ├─ token 已过期 → 返回 "share expired" 错误
    ├─ 前端显示「分享已过期/删除」提示页
    └─ 页面渲染终止，**不会发起任何媒体下载请求**
    │
    ▼ （通过检查才继续）
[ProtectedMedia 注入 ?token=abc123]
    │
    ▼
浏览器发起 GET /photo/thumb_xxx.jpg?token=abc123
    │
    ▼
[HTTP 路由路径：过期检查点 B]
shareTokenFromRequest()
    ├─ token 已过期 → 403 Forbidden
    ├─ 图片无法加载（显示破图图标）
    └─ 防止用户手动构造 URL 绕过页面层检查
```

### 7.3 双向触发的含义

```
方向 1：Token 过期 → 阻止下载（正向拦截）
  GraphQL 检查先拦，HTTP 检查兜底
  形成双重保险

方向 2：下载续期 → 延长 Token（反向操作，用户主动触发）
  在侧边栏 Sharing 面板：
  MorePopoverSectionExpiration 组件
  ├─ 勾选「Expiration date」
  ├─ 选择新日期
  └─ 调用 setExpireShareToken mutation
      → 更新 DB 中 Expire 字段
      → 后续下载请求自动放行
```

### 7.4 时间处理的「坑」——fakeTime 逻辑

文件：`api/graphql/resolvers/share_token.go:84-94` 与 `113-133`

```go
now := time.Now()
fakeTime := time.Date(
    now.Year(), now.Month(), now.Day(),
    now.Hour(), now.Minute(), now.Second(),
    0,        // 纳秒清零
    time.UTC, // 强制 UTC
)
```

**为什么这样做？**
- 前端 DatePicker 选择日期时，`dayjs(date).endOf('day').format('YYYY-MM-DDTHH:mm:ss')+'Z'`
- 即把**用户本地时间的当天结束时刻**，直接加 `Z` 后缀当作 UTC 存
- 后端 resolver 校验时，也把**服务器当前时间**截断并强制 UTC
- 这样「对表」：即使跨时区，用户选的「当天结束」也能正确匹配

**注意：** HTTP 路由路径的过期检查**没有 fakeTime 处理**，用真实 UTC 比较
→ 意味着 resolver 层可能通过（差几秒），但实际下载请求被拒
→ 这是一个细微的不一致点

### 7.5 前端缓存：过期 Token 的隐形窗口期

**结论：Apollo Client 的 InMemoryCache 可能导致「token 已过期但页面仍显示内容」的窗口期。**

#### 7.5.1 Apollo 缓存配置

文件：`ui/src/apolloClient.ts:161-194`

```javascript
const memoryCache = new InMemoryCache({
  typePolicies: {
    SiteInfo: { merge: true },
    MediaURL: { keyFields: ['url'] },
    Album: { fields: { media: paginateCache(...) } },
    // 注意：没有 ShareToken 的特殊 typePolicy
  },
})

const client = new ApolloClient({
  link: ApolloLink.from([linkError, link]),
  cache: memoryCache,
  // 未设置 defaultOptions，fetchPolicy 使用默认值 cache-first
})
```

**关键事实：**
- 默认 `fetchPolicy: 'cache-first'`（Apollo Client 默认值）
- ShareToken 查询未设置任何特殊缓存策略
- 缓存 key 由 query 名称 + 变量共同决定

#### 7.5.2 窗口期产生的条件

```
场景：用户第一次访问分享链接（token 未过期）
    │
    ├─ SHARE_TOKEN_QUERY 发起网络请求
    ├─ 后端返回 shareToken 数据（包含 album/media）
    └─ Apollo 将结果写入 InMemoryCache（以 token+password 为 key）
    │
    ▼
用户停留在页面上，期间 token 过期了
    │
    └─ 页面上的图片/视频会陆续 403（因为 HTTP 路由每次都查 DB）
    └─ 但页面元数据（标题、结构）仍然显示（因为走缓存）

场景：用户在同一会话中重新访问同一分享链接
    │
    ├─ useQuery 检测到缓存中有匹配结果
    ├─ cache-first 策略：直接返回缓存数据，不发网络请求
    └─ 页面显示「正常」，但图片全是破图
```

**窗口期的边界：**

| 操作 | 是否命中缓存 | 是否能检测过期 |
|------|-------------|---------------|
| 首次访问分享页 | ❌ 无缓存 | ✅ 网络请求，后端检查 |
| 同一会话内再次访问 | ✅ 命中缓存 | ❌ 不发请求，无法知道过期 |
| 刷新页面（F5） | ❌ 内存缓存重置 | ✅ 重新请求 |
| 关闭浏览器再打开 | ❌ Cookie 和缓存都重置 | ✅ 重新请求 |
| 密码变更后访问 | ❌ 变量变了，缓存 key 变 | ✅ 重新请求 |
| 主动调用 refetch() | ❌ 强制网络请求 | ✅ 重新请求 |

#### 7.5.3 两道防线的分工

```
过期检测三道防线：
┌─────────────────────────────────────────────────────────┐
│ 第 1 道：GraphQL Resolver 过期检查（fakeTime 方式）       │
│   作用：页面加载时判断是否显示内容                        │
│   问题：受 Apollo 缓存影响，可能不触发                    │
├─────────────────────────────────────────────────────────┤
│ 第 2 道：HTTP 路由过期检查（真实 UTC 比较）               │
│   作用：每次媒体请求都检查，防止直链绕过                  │
│   特点：无缓存，每次都查 DB，最可靠                       │
├─────────────────────────────────────────────────────────┤
│ 第 3 道：图片加载失败的视觉提示                           │
│   作用：用户看到破图，间接感知 token 失效                 │
│   特点：被动感知，不是主动拦截                            │
└─────────────────────────────────────────────────────────┘
```

**实际效果：**
- 缓存窗口期内，页面**结构和文字**可能正常显示
- 但**所有媒体资源**（图片、视频、下载）都会因 HTTP 路由检查而失败
- 用户体验是「页面能打开，但图全挂了」
- 这是「过期 → 拦截下载」双向触发机制中，HTTP 路由层兜底作用的体现

#### 7.5.4 测试环境的特殊配置

文件：`ui/src/Pages/SharePage/SharePage.test.tsx:104-106`

```javascript
// disable cache, required to make fragments work
watchQuery: { fetchPolicy: 'no-cache' },
query: { fetchPolicy: 'no-cache' },
```

测试中显式禁用了缓存，因此测试用例不会遇到缓存窗口期问题。生产环境使用默认的 `cache-first` 策略。

---

## 八、完整协作流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      分享 Token 全生命周期协作图                              │
└─────────────────────────────────────────────────────────────────────────────┘

  【创建阶段】
  登录用户 → 侧边栏 Sharing → 点击 Add shares
      │
      ▼
  shareMedia / shareAlbum (GraphQL Mutation)
      │
      ├─ 校验相册所有权
      ├─ bcrypt 密码哈希（可选）
      ├─ GenerateToken() 生成 Value
      └─ 写入 share_tokens 表
      │
      ▼
  返回 { token } → 前端复制链接：/share/{token}

═══════════════════════════════════════════════════════════════════════════════

  【访问阶段：匿名用户打开分享链接】
  浏览器访问 /share/abc123
      │
      ▼
  ┌─ TokenRoute (前端路由层) ────────────────────────────────────────────┐
  │                                                                      │
  │  Step 1: 预检密码有效性                                              │
  │  VALIDATE_TOKEN_PASSWORD_QUERY → shareTokenValidatePassword         │
  │      ├─ 查 share_tokens 表                                           │
  │      ├─ ★ 过期检查（fakeTime 方式）                                  │
  │      │   └─ 过期 → error "share expired" → 显示过期提示页            │
  │      ├─ 无密码 → return true                                        │
  │      ├─ 有密码 + Cookie 正确 → return true                          │
  │      └─ 有密码 + Cookie 缺失/错误 → return false → 密码输入页        │
  │                                                                      │
  │  Step 2: 密码输入（如需要）                                          │
  │  PasswordProtectedShare → 保存 Cookie: share-token-pw-abc123         │
  │                                                                      │
  └──────────────────────────────────────────────────────────────────────┘
      │
      ▼ （验证通过）
  ┌─ AuthorizedTokenRoute ──────────────────────────────────────────────┐
  │                                                                      │
  │  SHARE_TOKEN_QUERY → shareToken resolver                            │
  │      ├─ ★ 过期检查（fakeTime 方式）                                  │
  │      ├─ ★ 密码校验（bcrypt）                                         │
  │      └─ 返回 album 或 media 对象（含 downloads/thumbnail/... URL）   │
  │                                                                      │
  └──────────────────────────────────────────────────────────────────────┘
      │
      ▼
  ┌─ 媒体渲染层 ─────────────────────────────────────────────────────────┐
  │                                                                      │
  │  ProtectedImage / ProtectedVideo                                    │
  │      └─ getProtectedUrl()                                           │
  │          ├─ 正则匹配 /share/{token}                                 │
  │          └─ 所有媒体 URL 附加 ?token=abc123                          │
  │                                                                      │
  │  AlbumSharePage / MediaSharePage 渲染                                │
  │                                                                      │
  └──────────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════════

  【下载阶段：匿名用户加载/下载媒体】
  浏览器发起 GET /photo/thumb.jpg?token=abc123
      │
      ▼
  ┌─ HTTP 路由层 ───────────────────────────────────────────────────────┐
  │                                                                      │
  │  RegisterPhotoRoutes → authenticateMedia()                          │
  │  RegisterVideoRoutes → authenticateMedia()                          │
  │  RegisterDownloadRoutes → authenticateAlbum()                       │
  │                                                                      │
  │  shareTokenFromRequest()                                            │
  │      ├─ 从 URL 取 ?token= 参数                                       │
  │      ├─ 查 share_tokens 表                                           │
  │      ├─ ★ 过期检查（真实 UTC 比较）← 第 2 次检查                     │
  │      │   └─ 过期 → 403 Forbidden                                    │
  │      ├─ ★ 密码 Cookie 校验（读取 Cookie + bcrypt）                  │
  │      ├─ 资源匹配检查                                                 │
  │      │   ├─ Album token：精确匹配 + 递归子相册                       │
  │      │   └─ Media token：精确 ID 匹配                               │
  │      └─ 全部通过 → 放行文件                                          │
  │                                                                      │
  └──────────────────────────────────────────────────────────────────────┘
      │
      ▼
  响应文件内容 / ZIP 压缩包

═══════════════════════════════════════════════════════════════════════════════

  【续期阶段：所有者延长/取消过期】
  登录用户 → 侧边栏 Sharing → More → Expiration date
      │
      ▼
  MorePopoverSectionExpiration 组件
      ├─ 勾选复选框：启用过期
      ├─ 取消勾选：expire=null（永不过期）
      ├─ DatePicker 选择日期
      │   └─ dayjs(date).endOf('day').UTC 格式处理
      └─ 提交：setExpireShareToken mutation
          │
          ▼
      actions.SetExpireShareToken()
          └─ UPDATE share_tokens SET expire = ? WHERE value = ?
          │
          ▼
      后续匿名用户访问 → 过期检查使用新时间
```

---

## 九、协作机制总结

### 9.1 分享 Token 生命周期 × 匿名下载授权 的协作关系

| 生命周期阶段 | 涉及模块 | 授权影响 |
|-------------|----------|----------|
| **创建** | `AddMediaShare` / `AddAlbumShare` | 生成可用 token，匿名用户可访问 |
| **活跃** | 所有访问路径 | 双重过期检查 + 双重密码校验 |
| **过期（存储）** | share_tokens 表 | **无后台清理**，过期记录永久保留，仅 CASCADE 删除 |
| **过期（访问）** | GraphQL resolver + HTTP route + 前端缓存 | 三道防线：页面检查 → 媒体检查 → 视觉感知 |
| **续期** | `SetExpireShareToken` | 两层权限校验（登录 + owner/admin），原链接不变 |
| **加/解密** | `ProtectShareToken` | 增删密码，影响 Cookie 校验要求 |
| **删除** | `DeleteShareToken` | DB 记录消失，所有检查失败 |

### 9.2 双向触发的本质

```
「过期 → 拦截下载」：
  被动检查机制，三道防线
  第 1 道：GraphQL Resolver（可能被 Apollo 缓存跳过）
  第 2 道：HTTP 路由（每次请求都查 DB，最可靠）
  第 3 道：图片加载失败的视觉提示
  防止用户绕开前端直接访问媒体 URL

「续期 → 延长下载」：
  主动修改机制，两层权限校验
  第 1 层：@isAuthorized 确保已登录
  第 2 层：getUserToken 确保是 owner 或 admin
  一次 DB 更新即可生效，无需重新生成 token
```

### 9.3 三处深入发现的关键结论

| 问题 | 结论 | 关键代码 |
|------|------|----------|
| **过期记录清理** | ❌ 无后台 cron 任务，过期记录永久保留 | `periodic_scanner.go`、`scanner_tasks.go` |
| **续期权限校验** | ✅ 两层校验（登录 + owner/admin），无需重新登录 | `directive.go`、`share_token_actions.go:167` |
| **前端缓存窗口期** | ⚠️ Apollo cache-first 策略可能导致页面文字正常但图片全挂 | `apolloClient.ts`、`SharePage.tsx` |

### 9.4 关键文件索引

| 模块 | 文件路径 | 核心内容 |
|------|----------|----------|
| 数据模型 | `api/graphql/models/share_token.go` | ShareToken 结构体 |
| 业务动作 | `api/graphql/models/actions/share_token_actions.go` | CRUD + 权限校验（getUserToken）|
| GraphQL 解析 | `api/graphql/resolvers/share_token.go` | 过期检查（fakeTime）+ 密码校验 |
| GraphQL 指令 | `api/graphql/directive.go` | @isAuthorized / @isAdmin 权限指令 |
| HTTP 路由认证 | `api/routes/authenticate_routes.go` | 下载授权核心逻辑（过期+密码+资源匹配）|
| 图片路由 | `api/routes/photos.go` | 图片加载 authenticateMedia 调用 |
| 视频路由 | `api/routes/videos.go` | 视频播放 authenticateMedia 调用 |
| 下载路由 | `api/routes/downloads.go` | ZIP 打包下载 authenticateAlbum 调用 |
| 定期扫描器 | `api/scanner/periodic_scanner/periodic_scanner.go` | 仅媒体扫描，无 Token 清理 |
| 扫描任务列表 | `api/scanner/scanner_tasks/scanner_tasks.go` | allTasks 数组不含 Token 清理 |
| 服务入口 | `api/server.go` | 后台组件初始化，无 Token 清理 |
| Apollo 客户端 | `ui/src/apolloClient.ts` | InMemoryCache 配置、默认 cache-first |
| 前端路由 | `ui/src/components/routes/Routes.tsx` | /share/:token 路由定义 |
| 分享页面 | `ui/src/Pages/SharePage/SharePage.tsx` | 两阶段验证（预检+加载） |
| 密码输入页 | `ui/src/Pages/SharePage/PasswordProtectedShare.tsx` | 密码收集与 Cookie 保存 |
| Cookie 管理 | `ui/src/helpers/authentication.ts` | share-token-pw-{token} 读写 |
| URL 注入 | `ui/src/components/photoGallery/ProtectedMedia.tsx` | ?token= 查询参数附加 |
| 侧边栏管理 | `ui/src/components/sidebar/Sharing.tsx` | 创建/删除/加密码/设置过期 UI |
