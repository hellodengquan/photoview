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

---

## 三、分享页面访问流程（前端路由）

### 3.1 路由入口
文件：`ui/src/components/routes/Routes.tsx:77-80`

```
/share/:token/*  →  TokenRoute (懒加载)
```

### 3.2 两阶段验证机制

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

## 四、匿名下载授权机制

### 4.1 三条下载/访问路由

| 路由 | 文件 | 用途 |
|------|------|------|
| `GET /photo/{name}` | `api/routes/photos.go` | 单张图片加载（缩略图、高清图） |
| `GET /video/{name}` | `api/routes/videos.go` | 视频播放（含转码后 web 格式） |
| `GET /download/album/{id}/{purpose}` | `api/routes/downloads.go` | 批量打包下载相册为 ZIP |

### 4.2 双轨认证架构

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

### 4.3 ShareToken 请求认证核心逻辑

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

### 4.4 前端 URL Token 注入机制

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

## 五、密码 Cookie 机制

### 5.1 Cookie 存储（前端）

文件：`ui/src/helpers/authentication.ts:31-47`

```javascript
SHARE_TOKEN_COOKIE_NAME = `share-token-pw-${shareToken}`

saveSharePassword(token, password)
  → Cookies.set(cookieName, password, { path: '/', sameSite: 'Lax' })
  // 注意：未设置 expires，即 session cookie，浏览器关闭失效

getSharePassword(token)    // 读取
clearSharePassword(token)  // 删除
```

### 5.2 Cookie 读取（后端）

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

## 六、Token 过期检查的「双向触发」机制

### 6.1 两处过期检查点

| 检查位置 | 文件 | 场景 | 时间处理 |
|----------|------|------|----------|
| **GraphQL Resolver** | `api/graphql/resolvers/share_token.go:84-98` | 页面访问时获取 shareToken 元数据 | **构造 fakeTime**：截断纳秒，UTC，客户端本地时间视为 UTC |
| **HTTP 路由** | `api/routes/authenticate_routes.go:89-92` | 图片/视频/下载请求时 | **直接 UTC 比较**：time.Now().UTC() vs expire.UTC() |

### 6.2 为什么有两处检查？——协作流程

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

### 6.3 双向触发的含义

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

### 6.4 时间处理的「坑」——fakeTime 逻辑

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

---

## 七、完整协作流程图

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

## 八、协作机制总结

### 8.1 分享 Token 生命周期 × 匿名下载授权 的协作关系

| 生命周期阶段 | 涉及模块 | 授权影响 |
|-------------|----------|----------|
| **创建** | `AddMediaShare` / `AddAlbumShare` | 生成可用 token，匿名用户可访问 |
| **活跃** | 所有访问路径 | 双重过期检查 + 双重密码校验 |
| **过期** | GraphQL resolver + HTTP route | 双向拦截：页面层和下载层都拒绝 |
| **续期** | `SetExpireShareToken` | 更新 DB，后续请求自动放行 |
| **加/解密** | `ProtectShareToken` | 增删密码，影响 Cookie 校验要求 |
| **删除** | `DeleteShareToken` | DB 记录消失，所有检查失败 |

### 8.2 双向触发的本质

```
「过期 → 拦截下载」：
  被动检查机制，两道防线（GraphQL + HTTP）
  防止用户绕开前端直接访问媒体 URL

「续期 → 延长下载」：
  主动修改机制，一次 DB 更新即可生效
  无需重新生成 token，原链接保持可用
```

### 8.3 关键文件索引

| 模块 | 文件路径 | 核心内容 |
|------|----------|----------|
| 数据模型 | `api/graphql/models/share_token.go` | ShareToken 结构体 |
| 业务动作 | `api/graphql/models/actions/share_token_actions.go` | CRUD + 权限校验 |
| GraphQL 解析 | `api/graphql/resolvers/share_token.go` | 过期检查（fakeTime）+ 密码校验 |
| HTTP 路由认证 | `api/routes/authenticate_routes.go` | 下载授权核心逻辑（过期+密码+资源匹配）|
| 图片路由 | `api/routes/photos.go` | 图片加载 authenticateMedia 调用 |
| 视频路由 | `api/routes/videos.go` | 视频播放 authenticateMedia 调用 |
| 下载路由 | `api/routes/downloads.go` | ZIP 打包下载 authenticateAlbum 调用 |
| 前端路由 | `ui/src/components/routes/Routes.tsx` | /share/:token 路由定义 |
| 分享页面 | `ui/src/Pages/SharePage/SharePage.tsx` | 两阶段验证（预检+加载） |
| 密码输入页 | `ui/src/Pages/SharePage/PasswordProtectedShare.tsx` | 密码收集与 Cookie 保存 |
| Cookie 管理 | `ui/src/helpers/authentication.ts` | share-token-pw-{token} 读写 |
| URL 注入 | `ui/src/components/photoGallery/ProtectedMedia.tsx` | ?token= 查询参数附加 |
| 侧边栏管理 | `ui/src/components/sidebar/Sharing.tsx` | 创建/删除/加密码/设置过期 UI |
