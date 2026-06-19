# Photoview OIDC 与本地账号鉴权协作流程分析

## 概述

Photoview 本身**没有内置 OIDC/OAuth2 客户端实现**，但代码架构为 OIDC 集成提供了基础支持。OIDC 认证通常在**反向代理层**完成，然后通过 HTTP Header 将认证信息传递给应用。

---

## 一、核心数据模型

### 1.1 User 模型设计

**文件**: `api/graphql/models/user.go:14-21`

```go
type User struct {
    Model
    Username string  `gorm:"unique;size:128"`
    Password *string `gorm:"size:256"`  // 关键：指针类型，允许为 nil
    Albums   []Album `gorm:"many2many:user_albums;constraint:OnDelete:CASCADE;"`
    Admin    bool    `gorm:"default:false"`
}
```

**关键设计**:
- `Password` 字段是 `*string`（可空指针），支持**无密码用户**
- 这是 OIDC 集成的核心基础：OIDC 用户不需要本地密码

### 1.2 AccessToken 模型

**文件**: `api/graphql/models/user.go:35-41`

```go
type AccessToken struct {
    Model
    UserID int       `gorm:"not null;index"`
    User   User      `gorm:"constraint:OnDelete:CASCADE;"`
    Value  string    `gorm:"not null;size:24;index"`
    Expire time.Time `gorm:"not null;index"`
}
```

**关键特性**:
- 每个 `AccessToken` 是独立记录，支持**多设备多会话**：同一用户可同时拥有多个有效 token
- `OnDelete:CASCADE`：用户删除时关联 token 自动清除

### 1.3 ShareToken 模型

**文件**: `api/graphql/models/share_token.go:7-18`

```go
type ShareToken struct {
    Model
    Value    string     `gorm:"not null"`
    OwnerID  int        `gorm:"not null;index"`
    Owner    User       `gorm:"constraint:OnDelete:CASCADE;"`
    Expire   *time.Time `gorm:"index"`     // 可空：nil 表示永不过期
    Password *string                          // 可空：nil 表示无密码保护
    AlbumID  *int   `gorm:"index"`
    Album    *Album `gorm:"constraint:OnDelete:CASCADE;"`
    MediaID  *int   `gorm:"index"`
    Media    *Media `gorm:"constraint:OnDelete:CASCADE;"`
}
```

---

## 二、本地账号鉴权流程

### 2.1 登录入口全景

Photoview 存在**三条独立的登录/鉴权入口**，互不干扰：

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        鉴权入口总览                                        │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  入口 A: 用户名密码登录（已认证用户）                                      │
│     LoginPage → authorizeUser mutation → GenerateAccessToken → Cookie    │
│                                                                          │
│  入口 B: Share 链接匿名访问（无需登录）                                    │
│     /share/:token → shareToken query → 密码验证 → 媒体资源访问            │
│                                                                          │
│  入口 C: OIDC 反向代理登录（需新增中间件）                                 │
│     X-Remote-User Header → 查询/创建无密码用户 → GenerateAccessToken     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.2 入口 A：用户名密码登录

#### 前端登录页面
**文件**: `ui/src/Pages/LoginPage/LoginPage.tsx`

```typescript
const authorizeMutation = gql`
  mutation Authorize($username: String!, $password: String!) {
    authorizeUser(username: $username, password: $password) {
      success
      status
      token
    }
  }
`
```

#### 后端 Resolver
**文件**: `api/graphql/resolvers/user.go:24-54`

```go
func (r *mutationResolver) AuthorizeUser(ctx context.Context, username string, password string) (*models.AuthorizeResult, error) {
    db := r.DB(ctx)
    user, err := models.AuthorizeUser(db, username, password)
    // ... 生成 AccessToken
    token, err = user.GenerateAccessToken(tx)
    // ...
}
```

### 2.3 密码验证逻辑

**文件**: `api/graphql/models/user.go:76-100`

```go
func AuthorizeUser(db *gorm.DB, username string, password string) (*User, error) {
    var user User
    result := db.Where("username = ?", username).First(&user)
    
    if user.Password == nil {
        return nil, errors.New("user does not have a password")  // OIDC 用户会在此处失败
    }
    
    if err := bcrypt.CompareHashAndPassword([]byte(*user.Password), []byte(password)); err != nil {
        return nil, ErrorInvalidUserCredentials
    }
    
    return &user, nil
}
```

### 2.4 用户注册

**文件**: `api/graphql/models/user.go:102-124`

```go
func RegisterUser(db *gorm.DB, username string, password *string, admin bool) (*User, error) {
    user := User{
        Username: username,
        Admin:    admin,
    }
    
    if password != nil {
        hashedPassBytes, _ := bcrypt.GenerateFromPassword([]byte(*password), 12)
        hashedPass := string(hashedPassBytes)
        user.Password = &hashedPass
    }
    // password == nil 时，创建无密码用户（用于 OIDC）
    
    result := db.Create(&user)
    return &user, nil
}
```

**测试用例验证**: `api/routes/photos_test.go:26`
```go
user, err := models.RegisterUser(db, "testuser", nil, false)  // 创建无密码用户
```

---

## 三、会话保持机制

### 3.1 Token 生成

**文件**: `api/graphql/models/user.go:126-151`

```go
func (user *User) GenerateAccessToken(db *gorm.DB) (*AccessToken, error) {
    // 生成 24 位随机字符串
    bytes := make([]byte, 24)
    rand.Read(bytes)
    // ... 编码为字母数字
    tokenValue := string(bytes)
    expire := time.Now().Add(14 * 24 * time.Hour)  // 有效期 14 天
    
    token := AccessToken{
        UserID: user.ID,
        Value:  tokenValue,
        Expire: expire,
    }
    db.Create(&token)
    return &token, nil
}
```

**重要设计**:
- 每次登录调用都会创建**新的 AccessToken 记录**，旧记录不会被删除
- 这是多设备支持的基础：每次新设备登录都生成独立 token
- 没有续期（refresh）机制，也没有滑动窗口

### 3.2 前端 Token 存储

**文件**: `ui/src/helpers/authentication.ts`

```typescript
const AUTH_TOKEN_COOKIE_NAME = 'auth-token'
const AUTH_TOKEN_MAX_AGE_IN_DAYS = 14

export function saveTokenCookie(token: string) {
  const options = {
    path: '/',
    sameSite: 'Lax',
    expires: AUTH_TOKEN_MAX_AGE_IN_DAYS,
  }
  Cookies.set(AUTH_TOKEN_COOKIE_NAME, token, options)
}

export function clearTokenCookie() {
  Cookies.remove(AUTH_TOKEN_COOKIE_NAME)
}

export function authToken() {
  return Cookies.get(AUTH_TOKEN_COOKIE_NAME)
}
```

### 3.3 认证中间件（Token 失效判定）

**文件**: `api/graphql/auth/auth.go:31-70`

```go
func Middleware(db *gorm.DB) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if tokenCookie, err := r.Cookie("auth-token"); err == nil {
                loaders := dataloader.For(r.Context())
                if loaders == nil {
                    http.Error(w, INTERNAL_SERVER_ERROR, http.StatusInternalServerError)
                    return
                }

                user, err := loaders.UserFromAccessToken.Load(tokenCookie.Value)
                if err != nil {
                    log.Error(r.Context(), "Error loading user from token", "error", err)
                    http.Error(w, INVALID_AUTH_TOKEN, http.StatusUnauthorized)
                    return
                }

                // 如果 user 为 nil，表示 token 不存在或已过期
                if user == nil {
                    log.Error(r.Context(), "Token not found in database")
                    http.Error(w, INVALID_AUTH_TOKEN, http.StatusUnauthorized)
                    return
                }

                ctx := AddUserToContext(r.Context(), user)
                r = r.WithContext(ctx)
            }
            // 注意：如果没有 auth-token cookie，直接放行（不返回 401）
            // 是否需要认证由后续 GraphQL directive 决定
            next.ServeHTTP(w, r)
        })
    }
}
```

### 3.4 Token 验证与过期检查（Dataloader）

**文件**: `api/dataloader/userLoader.go:10-71`

```go
func NewUserLoaderByToken(db *gorm.DB) *UserLoader {
    return &UserLoader{
        maxBatch: 100,
        wait:     5 * time.Millisecond,
        fetch: func(tokens []string) ([]*models.User, []error) {
            // 关键：只查询 expire > now 的 token
            // 过期 token 直接被过滤，返回 nil user
            var accessTokens []*models.AccessToken
            err := db.Where("expire > ?", time.Now()).Where("value IN (?)", tokens).Find(&accessTokens).Error

            rows, err := db.Table("access_tokens").Select("distinct user_id").
                Where("expire > ?", time.Now()).
                Where("value IN (?)", tokens).Rows()

            // ... 构建 userMap 和 tokenMap
            // 对于过期或不存在的 token，result[i] = nil
            result := make([]*models.User, len(tokens))
            for i, token := range tokens {
                accessToken, tokenFound := tokenMap[token]
                if tokenFound {
                    user, userFound := userMap[accessToken.UserID]
                    if userFound {
                        result[i] = user
                    }
                }
            }
            return result, nil
        },
    }
}
```

### 3.5 Token 过期 14 天后的处理（无自动续期机制）

经过完整代码追踪，Photoview **没有任何 Token 自动续期机制**，完整的过期路径如下：

```
Token 过期时间线
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  登录成功                                                           │
│    │                                                                │
│    ▼                                                                │
│  GenerateAccessToken() → 数据库写入 expire = now + 14天            │
│    │                                                                │
│    ▼                                                                │
│  前端保存 Cookie，expires = 14天                                    │
│    │                                                                │
│    ▼                                                                │
│  每次请求 → auth.Middleware → Dataloader 检查 expire > now         │
│    │                                                                │
│    ├── 未过期（14天内）: user != nil → 正常访问                     │
│    │                                                                │
│    └── 已过期（超过14天）: user == nil → 返回 401 Unauthorized      │
│             │                                                       │
│             ▼                                                       │
│        前端 apolloClient 错误处理                                    │
│        文件: ui/src/apolloClient.ts:97-106                          │
│                                                                     │
│        const linkError = onError(({ graphQLErrors, networkError }) │
│          if (graphQLErrors.find(x => x.message == 'unauthorized'))  │
│            console.log('Unauthorized, clearing token cookie')       │
│            clearTokenCookie()            // ← 清除本地 Cookie       │
│            // 注意：没有 location.reload()，也不会重定向到登录页     │
│          }                                                           │
│          if (networkError) {                                         │
│            clearTokenCookie()            // ← 清除本地 Cookie       │
│          }                                                           │
│        })                                                            │
│             │                                                       │
│             ▼                                                       │
│        后续页面访问                                                  │
│        文件: ui/src/components/routes/AuthorizedRoute.tsx:35-43     │
│                                                                     │
│        const AuthorizedRoute = ({ children }) => {                  │
│          const token = authToken()                                   │
│          if (!token) {                                               │
│            return <Navigate to="/" />      // ← 重定向到根路径      │
│          }                                                           │
│          return <>{children}</>                                      │
│        }                                                             │
│             │                                                       │
│             ▼                                                       │
│        IndexPage 路由判断                                            │
│        文件: ui/src/components/routes/Routes.tsx:139-145            │
│                                                                     │
│        const IndexPage = () => {                                     │
│          const token = authToken()                                   │
│          const dest = token ? '/timeline' : '/login'  // ← 跳转登录 │
│          return <Navigate to={dest} />                               │
│        }                                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**关键结论**:
- ❌ **没有 Token 续期（refresh token）机制**：14 天后必须重新登录
- ❌ **没有滑动窗口续期**：每次请求不会刷新 token 的过期时间
- ❌ **没有服务端主动清除过期 token**：过期的 AccessToken 记录永远留在数据库中
- ✅ **失效路径是强制重新登录**：过期 → 401 → 清除 Cookie → 重定向登录页

### 3.6 WebSocket 认证

**文件**: `api/graphql/auth/auth.go:92-131`

```go
func AuthWebsocketInit() func(context.Context, transport.InitPayload) (context.Context, *transport.InitPayload, error) {
    return func(ctx context.Context, initPayload transport.InitPayload) (context.Context, *transport.InitPayload, error) {
        bearer, exists := initPayload["Authorization"].(string)
        if !exists {
            return ctx, nil, nil  // 没有 token，返回不带 user 的 context
        }

        token, err := TokenFromBearer(&bearer)
        // ... 通过 dataloader 验证 token（同样检查 expire）
        userCtx := context.WithValue(ctx, userCtxKey, user)
        return userCtx, nil, nil
    }
}
```

### 3.7 WebSocket 订阅 Token 中途失效路径

#### 3.7.1 WebSocket 连接配置

**文件**: `api/graphql/endpoint/graphql_endpoint.go:31-35`

```go
graphqlServer.AddTransport(transport.Websocket{
    KeepAlivePingInterval: 10 * time.Second,
    Upgrader:              server.WebsocketUpgrader(utils.DevelopmentMode()),
    InitFunc:              auth.AuthWebsocketInit(),
})
```

#### 3.7.2 前端 WebSocket 建立与 Bearer Token 传递

**文件**: `ui/src/apolloClient.ts:38-53`

```typescript
const wsLink = new WebSocketLink({
  uri: websocketUri.toString(),
  options: {
    reconnect: true,
    lazy: true,
    connectionParams: () => {
      const token = authToken()
      if (token) {
        return {
          Authorization: `Bearer ${token}`  // 只在连接建立时传递一次
        }
      }
      return {}
    }
  }
})
```

#### 3.7.3 唯一的订阅：通知推送

**文件**: `api/graphql/resolvers/notification.go:17-34`

```go
func (r *subscriptionResolver) Notification(ctx context.Context) (<-chan *models.Notification, error) {
    user := auth.UserFromContext(ctx)
    if user == nil {
        return nil, auth.ErrUnauthorized
    }

    notificationChannel := make(chan *models.Notification, 1)
    listenerID := notification.RegisterListener(user, notificationChannel)

    go func() {
        <-ctx.Done()  // 监听 context 取消（连接断开时触发）
        notification.DeregisterListener(listenerID)
    }()

    return notificationChannel, nil
}
```

**关键设计**:
- `InitFunc` **只在连接建立时调用一次**，之后不再校验 token
- `KeepAlivePingInterval: 10s` 只是心跳保活，**不做 token 校验**
- 连接的生命周期绑定到 context，只有 `ctx.Done()` 时才会清理

#### 3.7.4 Token 中途失效的完整路径

```
Token 中途失效路径（共 3 层，层层穿透）

┌─────────────────────────────────────────────────────────────────────┐
│  第一层：数据库层面（token 自然过期或被删除）                          │
│  access_tokens.expire < now 或记录被 DELETE                          │
└───────────────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│  第二层：服务端行为（完全无感知）                                      │
│  ✅ 没有定时任务轮询 token 有效性                                      │
│  ✅ KeepAlive ping 不校验 token                                        │
│  ✅ context 没有过期时间，与 token 有效期无绑定                         │
│  ✅ WebSocket 连接建立后，resolver 用的 user 存在 context 中，永不刷新  │
│  ✅ 已注册的 notification listener 继续接收推送直到连接断开            │
└───────────────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│  第三层：客户端感知（只能靠重连触发）                                  │
│  路径 A：连接自然断开（网络波动、页面刷新）→ 触发重连                  │
│            → 重连时调用 connectionParams() 取最新 token              │
│            → InitFunc 重新校验，发现 token 无效 → 返回错误            │
│            → 前端 onError 捕获 → clearTokenCookie() → 重定向登录      │
│                                                                     │
│  路径 B：客户端主动刷新（没有机制）                                    │
│            ❌ 没有定时重连逻辑                                        │
│            ❌ 没有在 subscription error 时强制重连                     │
│            ❌ 没有监听 token cookie 变化而重建连接                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 3.7.5 前端订阅错误处理

**文件**: `ui/src/components/messages/SubscriptionsHook.ts:51-69`

```typescript
const { data, error } = useSubscription<notificationSubscription>(
  NOTIFICATION_SUBSCRIPTION
)

useEffect(() => {
  if (error) {
    // 仅显示错误消息，不会触发重连或清 Cookie
    setMessages(state => [...state, {
      key: Math.random().toString(26),
      type: NotificationType.Message,
      props: {
        header: 'Network error',
        content: error.message,
        negative: true,
      },
    }])
  }
}, [data, error])
```

**关键发现**：
- ❌ **服务端不会主动断开已过期 token 的连接**
- ❌ **客户端没有任何机制主动检测 token 失效**
- ❌ **最长可达 14 天 + WebSocket 连接保持时间**（理论上无限期，只要不断开）
- ❌ **修改密码或吊销 token 对已建立的 WebSocket 连接完全无影响**
- ✅ **唯一失效时机：连接断开后的重连阶段**

#### 3.7.6 服务端主动断开的唯一触发点

唯一能让服务端主动断开连接的机制是通过 `notification.DeregisterListener`，但这只有在：
1. `ctx.Done()` 触发（客户端断开连接）
2. 服务进程重启
3. 内存中 listener 列表被清空（如用户被删除）

**文件**: `api/graphql/notification/notification.go:28-31`

```go
go func() {
    <-ctx.Done()
    notification.DeregisterListener(listenerID)
}()
```

---

## 四、Share 链接与匿名访问独立鉴权流程

### 4.1 Share Token 生成

**文件**: `api/graphql/models/actions/share_token_actions.go:15-103`

```go
func AddMediaShare(db *gorm.DB, user *models.User, mediaID int, expire *time.Time, password *string) (*models.ShareToken, error) {
    // 1. 验证用户拥有该媒体
    err := db.Joins("Album").
        Where("EXISTS (SELECT * FROM user_albums WHERE user_albums.album_id = Album.id AND user_albums.user_id = ?)", user.ID).
        First(&media, mediaID).Error

    // 2. 哈希密码（如果提供）
    hashedPassword, err := hashSharePassword(password)

    // 3. 生成 8 位随机 token（注意：比 AccessToken 的 24 位短）
    shareToken := models.ShareToken{
        Value:    utils.GenerateToken(),  // 文件: api/utils/utils.go:13-25
        OwnerID:  user.ID,
        Expire:   expire,    // 可空：nil = 永不过期
        Password: hashedPassword,
        MediaID:  &mediaID,
    }
    db.Create(&shareToken)
    return &shareToken, nil
}
```

### 4.2 Share 链接前端鉴权流程

**文件**: `ui/src/Pages/SharePage/SharePage.tsx`

Share 页面是完全独立的路由，**不经过 AuthorizedRoute，不需要登录**：

```
/share/:token 路由处理流程（Routes.tsx:78-80）
│
└── ▶ TokenRoute 组件（SharePage.tsx:154-201）
    │
    ├── Step 1: 检查是否有缓存的密码 Cookie
    │         password = getSharePassword(token)
    │         Cookie 名: share-token-pw-{token}
    │
    ├── Step 2: 调用 shareTokenValidatePassword query
    │         （GraphQL: share_token.graphql:28-29）
    │         验证 Share Token 是否有效、是否过期、密码是否正确
    │
    ├── Step 3: 结果分支
    │   │
    │   ├── share not found / share expired → 显示错误页面
    │   │
    │   ├── 需要密码但未提供 / 密码错误 → 显示 PasswordProtectedShare
    │   │   （PasswordProtectedShare.tsx）
    │   │     │
    │   │     └── 用户输入密码 → saveSharePassword() 存 Cookie → refetch
    │   │
    │   └── 验证成功 → 进入 AuthorizedTokenRoute
    │
    └── Step 4: AuthorizedTokenRoute 组件（SharePage.tsx:95-147）
        │
        ├── 调用 shareToken query 获取完整信息（含媒体数据）
        │
        ├── 如果关联 Album → AlbumSharePage 组件
        │   └── 相册内媒体通过 /api/photo/{name}?token={shareToken} 访问
        │
        └── 如果关联 Media → MediaSharePage 组件
            └── 单张媒体通过 /api/photo/{name}?token={shareToken} 访问
```

### 4.3 Share Token GraphQL 验证

**文件**: `api/graphql/resolvers/share_token.go:74-156`

```go
// 查询完整 Share 信息（包含媒体数据）
func (r *queryResolver) ShareToken(ctx context.Context, credentials models.ShareTokenCredentials) (*models.ShareToken, error) {
    var token models.ShareToken
    r.DB(ctx).Preload(clause.Associations).Where("value = ?", credentials.Token).First(&token)

    // 过期检查（客户端时间按 UTC 归一化）
    now := time.Now()
    fakeTime := time.Date(now.Year(), now.Month(), now.Day(), now.Hour(), now.Minute(), now.Second(), 0, time.UTC)
    if token.Expire != nil && fakeTime.After(*token.Expire) {
        return nil, errors.New("share expired")
    }

    // 密码检查（如果设置了密码）
    if token.Password != nil {
        if err := bcrypt.CompareHashAndPassword([]byte(*token.Password), []byte(*credentials.Password)); err != nil {
            return nil, errors.New("unauthorized")
        }
    }
    return &token, nil
}

// 仅验证密码（用于前端判断是否显示密码输入框）
func (r *queryResolver) ShareTokenValidatePassword(ctx context.Context, credentials models.ShareTokenCredentials) (bool, error) {
    // ... 同样的过期检查和密码检查逻辑
    // 返回 true/false 而不是 ShareToken 对象
}
```

### 4.4 媒体资源访问的双重鉴权路径

**文件**: `api/routes/authenticate_routes.go:19-155`

媒体和相册下载的 HTTP 接口有两条并行的鉴权路径：

```go
func authenticateMedia(media *models.Media, db *gorm.DB, r *http.Request) (success bool, ...) {
    user := auth.UserFromContext(r.Context())

    if user != nil {
        // 路径 1：已登录用户 —— 检查用户是否拥有该相册
        ownsAlbum, err := user.OwnsAlbum(db, &album)
        if !ownsAlbum {
            return false, "invalid credentials", http.StatusForbidden, nil
        }
    } else {
        // 路径 2：匿名用户 —— 检查 Share Token
        if success, respMsg, respStatus, err := shareTokenFromRequest(db, r, &media.ID, &media.AlbumID); !success {
            return success, respMsg, respStatus, err
        }
    }
    return true, "success", http.StatusAccepted, nil
}

func shareTokenFromRequest(db *gorm.DB, r *http.Request, mediaID *int, albumID *int) (success bool, ...) {
    // Step 1: 从 URL Query 读取 ?token=xxx
    token := r.URL.Query().Get("token")
    if token == "" {
        return false, "unauthorized", http.StatusForbidden, errors.New("share token not provided")
    }

    // Step 2: 查询 share_tokens 表
    db.Where("value = ?", token).First(&shareToken)

    // Step 3: 检查过期
    if shareToken.Expire != nil && time.Now().UTC().After(shareToken.Expire.UTC()) {
        return false, "unauthorized", http.StatusForbidden, errors.New("invalid share token")
    }

    // Step 4: 检查密码（从 Cookie 读取 share-token-pw-{token}）
    if shareToken.Password != nil {
        tokenPasswordCookie, err := r.Cookie(fmt.Sprintf("share-token-pw-%s", shareToken.Value))
        if err != nil {
            return false, "unauthorized", http.StatusForbidden, ...
        }
        if err := bcrypt.CompareHashAndPassword([]byte(*shareToken.Password), []byte(tokenPasswordCookie.Value)); err != nil {
            return false, "unauthorized", http.StatusForbidden, ...
        }
    }

    // Step 5: 检查 token 是否关联该媒体/相册（含子相册递归检查）
    // ...
    return true, "", 0, nil
}
```

**媒体资源路由的实际使用**:
- 照片：`api/routes/photos.go:35` → `authenticateMedia()`
- 视频：`api/routes/videos.go` → 同样的 `authenticateMedia()` 模式
- 下载：`api/routes/downloads.go:31` → `authenticateAlbum()`

### 4.5 Share 链接与主登录体系的关系

| 维度 | 用户登录体系（auth-token） | Share 匿名体系（share token） |
|-----|--------------------------|-------------------------------|
| Cookie 名 | `auth-token` | `share-token-pw-{token}` |
| Token 长度 | 24 位 | 8 位 |
| 过期时间 | 固定 14 天 | 可选，nil 为永不过期 |
| 密码保护 | 用户名密码（bcrypt） | 可选密码（bcrypt） |
| 权限范围 | 用户拥有的所有相册 | 单个指定相册或单张媒体 |
| 是否需要用户账号 | 是 | 否，完全匿名 |
| GraphQL directive | `@isAuthorized` / `@isAdmin` | 无，resolver 内手动校验 |
| 路由保护 | `AuthorizedRoute` 组件 | 无，SharePage 独立处理 |

---

## 五、Logout 销毁路径与多设备会话

### 5.1 前端 Logout 流程

**文件**: `ui/src/Pages/SettingsPage/UserPreferences.tsx:79-92`

```typescript
const LogoutButton = () => {
  const { t } = useTranslation()
  return (
    <Button
      onClick={() => {
        location.href = '/logout'   // 直接跳转，不调用 API
      }}
    >
      {t('settings.logout', 'Log out')}
    </Button>
  )
}
```

**文件**: `ui/src/components/routes/Routes.tsx:151-155`

```typescript
const LogoutPage = ({ navigate }: { navigate: NavigateFunction }) => {
  clearTokenCookie()    // 仅清除本地 Cookie
  navigate('/')
  return null
}
```

### 5.2 Logout 的实际效果

经过完整代码追踪，Photoview 的 Logout **只做了一件事：清除浏览器端的 Cookie**。完整的影响分析如下：

```
Logout 影响范围
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  ✅ 已清除：                                                         │
│     ├── 浏览器 Cookie: auth-token                                  │
│     └── 当前浏览器会话立即失去身份                                  │
│                                                                    │
│  ❌ 未清除（仍有效）：                                               │
│     ├── 数据库中的 AccessToken 记录（14 天后自然过期）              │
│     ├── 其他设备/浏览器上的同用户登录会话                           │
│     ├── WebSocket 已建立连接（直到断开重连才失效）                  │
│     └── Share Token（不受 Logout 影响，独立过期）                  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 5.3 多设备会话管理机制的缺失

**数据库层面**（`AccessToken` 表）：
```
access_tokens 表结构（支持多设备）
┌────┬─────────┬──────────────────────────┬─────────────────────┐
│ id │ user_id │ value                    │ expire              │
├────┼─────────┼──────────────────────────┼─────────────────────┤
│ 1  │ 1       │ abcdefghijklmnopqrstuvwx │ 2026-06-20 10:00:00 │ ← 设备 A
│ 2  │ 1       │ yyyy...24chars           │ 2026-07-01 15:30:00 │ ← 设备 B
│ 3  │ 1       │ zzzz...24chars           │ 2026-07-03 08:00:00 │ ← 设备 C
│ 4  │ 2       │ xxxx...24chars           │ 2026-07-02 12:00:00 │ ← 其他用户
└────┴─────────┴──────────────────────────┴─────────────────────┘
```

**现状分析**:
- ✅ 数据库设计**支持**多设备：每次 `GenerateAccessToken()` 都 INSERT 新记录
- ❌ **没有 API** 来查看当前用户的所有活跃会话
- ❌ **没有 API** 来强制注销某一设备（DELETE 特定 access_token）
- ❌ **没有 API** 来强制注销所有其他设备
- ❌ **没有后台清理任务**来清除已过期的 AccessToken 记录
- ❌ **没有修改密码后使旧 token 失效**的逻辑

### 5.4 会话失效的完整路径汇总

| 失效触发方式 | 代码路径 | 影响范围 | 是否立即生效 |
|------------|---------|---------|------------|
| 用户主动 Logout | 前端 `clearTokenCookie()` | 仅当前浏览器 Cookie | 是 |
| Token 自然过期（14 天） | Dataloader `expire > now` 过滤 | 该特定 token | 是，下次请求时 |
| GraphQL 认证失败 | `apolloClient.ts` onError → `clearTokenCookie()` | 当前浏览器 Cookie | 是 |
| 网络错误 | `apolloClient.ts` onError → `clearTokenCookie()` | 当前浏览器 Cookie | 是 |
| 用户被删除 | 数据库 `OnDelete:CASCADE` | 该用户所有 token | 是，下次请求时 |
| 管理员强制某设备下线 | ❌ 无此功能 | - | - |
| 修改密码后失效旧 token | ❌ 无此功能 | - | - |
| 定期清理过期 token | ❌ 无后台清理任务 | - | - |

---

## 六、OIDC 集成架构

### 6.1 典型部署架构

```
┌─────────────────┐     OIDC Redirect     ┌──────────────────┐
│   User Browser  │ ─────────────────────>│  OIDC Provider   │
│                 │ <─────────────────────│  (e.g. Authelia) │
└─────────────────┘     Auth Code         └──────────────────┘
          │                                            │
          │                                            │ 验证成功
          │                                            ▼
          │                              ┌──────────────────────────┐
          │                              │   Reverse Proxy          │
          │                              │  (Nginx/Traefik/Caddy)   │
          │                              │  - 设置 X-Remote-User    │
          │                              │  Header                  │
          │                              └──────────────────────────┘
          │                                            │
          │                       HTTP 请求带 Header    │
          ▼                                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Photoview Application                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  需要新增: OIDC 中间件                                    │  │
│  │  1. 读取 X-Remote-User Header                            │  │
│  │  2. 查询或创建对应用户（无密码）                          │  │
│  │  3. 生成 AccessToken 并设置 cookie                        │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### 6.2 OIDC 用户账号绑定

OIDC 用户的账号绑定需要以下步骤：

1. **前置准备**: 管理员通过 API 创建无密码用户
   ```graphql
   mutation {
     createUser(username: "oidc_user@example.com", admin: false) {
       id
       username
     }
   }
   ```
   注意: `password` 参数留空或传 `null`

2. **OIDC 认证**: 用户通过反向代理完成 OIDC 登录

3. **Header 传递**: 反向代理将用户名写入 HTTP Header（如 `X-Remote-User`）

4. **自动登录（需要新增中间件）**:
   - 读取 Header 中的用户名
   - 查询数据库中是否存在该用户
   - 如果存在且 `Password == nil`，则生成 AccessToken 并设置 cookie
   - 如果不存在，可配置自动创建用户或拒绝访问

### 6.3 需要新增的 OIDC 中间件示例

```go
// 伪代码：需要新增的 OIDC 认证中间件
func OIDCMiddleware(db *gorm.DB) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // 1. 跳过已通过 cookie 认证的请求
            if _, err := r.Cookie("auth-token"); err == nil {
                next.ServeHTTP(w, r)
                return
            }
            
            // 2. 读取反向代理设置的 Header
            remoteUser := r.Header.Get("X-Remote-User")
            if remoteUser == "" {
                next.ServeHTTP(w, r)
                return
            }
            
            // 3. 查询或创建用户
            var user models.User
            result := db.Where("username = ?", remoteUser).First(&user)
            if result.Error != nil {
                if errors.Is(result.Error, gorm.ErrRecordNotFound) {
                    // 可配置：自动创建无密码用户
                    user, _ = models.RegisterUser(db, remoteUser, nil, false)
                } else {
                    next.ServeHTTP(w, r)
                    return
                }
            }
            
            // 4. 确保用户是无密码用户（安全检查）
            if user.Password != nil {
                next.ServeHTTP(w, r)
                return
            }
            
            // 5. 生成 AccessToken 并设置 cookie
            token, _ := user.GenerateAccessToken(db)
            http.SetCookie(w, &http.Cookie{
                Name:     "auth-token",
                Value:    token.Value,
                Path:     "/",
                SameSite: http.SameSiteLaxMode,
                Expires:  token.Expire,
            })
            
            // 6. 将用户加入 context
            ctx := auth.AddUserToContext(r.Context(), &user)
            r = r.WithContext(ctx)
            
            next.ServeHTTP(w, r)
        })
    }
}
```

### 6.4 OIDC Logout 的特殊考虑

集成 OIDC 后，Logout 需要考虑双重登出：

```
OIDC 场景下的完整 Logout
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  1. 前端：点击 Logout → /logout → clearTokenCookie()         │
│                                                              │
│  2. 可选（需新增）：跳转到 OIDC Provider 的 logout endpoint   │
│     例如 Authelia: https://auth.example.com/logout           │
│     否则 OIDC Provider 的会话仍然有效，立即重新访问会免密登录 │
│                                                              │
│  3. 可选（需新增）：反向代理清除自身认证 Cookie/Session        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 6.5 OIDC 回调与本地账号合并策略

#### 6.5.1 当前数据模型的合并约束

Photoview 的 User 模型（`api/graphql/models/user.go:14-21`）有一个关键约束：

```go
type User struct {
    Model
    Username string  `gorm:"unique;size:128"`  // ← 唯一约束
    Password *string `gorm:"size:256"`
    Albums   []Album `gorm:"many2many:user_albums;constraint:OnDelete:CASCADE;"`
    Admin    bool    `gorm:"default:false"`
}
```

`Username` 字段有 `unique` 约束，这意味着：
- 本地用户 `alice` 和 OIDC 用户 `alice` 不能共存
- 合并时必须解决 Username 冲突

#### 6.5.2 当前代码中没有任何合并逻辑

经过完整代码搜索，Photoview **完全没有**以下任何实现：

| 缺失能力 | 代码证据 |
|---------|---------|
| OIDC 用户与本地用户的自动合并 | 无相关代码 |
| 字段冲突解决策略（Username、Admin） | 无相关代码 |
| OIDC identity 到本地 User 的映射表 | 无 `oauth_identities` 或类似表 |
| OIDC sub claim 到 User 的关联字段 | User 模型无 `OIDCSub` 或类似字段 |
| 合并后的数据迁移（相册、收藏等） | 无相关代码 |

#### 6.5.3 反向代理模式下的合并场景分析

在反向代理 OIDC 模式下，账号合并的核心问题是：**OIDC Provider 传来的用户名如何映射到本地用户**。

```
OIDC 回调后账号合并的三种场景

场景 A：纯 OIDC 用户（无冲突）
┌─────────────────────────────────────────────────────────────┐
│  X-Remote-User: alice@oidc                                  │
│  数据库查询: SELECT * FROM users WHERE username = 'alice@oidc' │
│  结果: 未找到                                               │
│  操作: RegisterUser(db, "alice@oidc", nil, false)            │
│  结果: 新建无密码用户，相册为空                              │
└─────────────────────────────────────────────────────────────┘

场景 B：OIDC 用户名与本地用户冲突（当前会静默失败）
┌─────────────────────────────────────────────────────────────┐
│  X-Remote-User: alice                                       │
│  数据库查询: SELECT * FROM users WHERE username = 'alice'    │
│  结果: 找到，但 Password != nil（本地用户）                  │
│  当前行为: OIDC 中间件跳过（Password != nil 安全检查）       │
│  结果: 用户无法通过 OIDC 登录，也无法理解原因                │
│                                                             │
│  ❌ 没有: 提示用户名冲突、建议合并、自动关联等逻辑           │
└─────────────────────────────────────────────────────────────┘

场景 C：同一用户先本地后 OIDC（需要手动关联）
┌─────────────────────────────────────────────────────────────┐
│  步骤 1: 管理员创建本地用户 alice，设置密码，分配相册        │
│  步骤 2: 部署 OIDC，alice 也想用 OIDC 登录                  │
│  步骤 3: 需要"关联"操作，将 OIDC identity 绑到本地 alice    │
│                                                             │
│  ❌ 没有: 关联操作、identity 绑定表、合并确认流程            │
│  当前唯一方案: 管理员手动将 alice 的密码清空（设为 NULL）    │
│  → UPDATE users SET password = NULL WHERE username = 'alice' │
│  → 但这样 alice 就不能再用密码登录了                        │
└─────────────────────────────────────────────────────────────┘
```

#### 6.5.4 合并所需的字段冲突解决

| 字段 | 冲突场景 | 当前处理 | 需要的处理 |
|------|---------|---------|-----------|
| `Username` | OIDC 用户名与本地用户同名 | unique 约束报错 | 命名空间隔离（如 `oidc:alice`）或关联绑定 |
| `Password` | 本地用户有密码，OIDC 用户无密码 | `Password != nil` 跳过 OIDC 登录 | 支持双认证方式：同时有密码和 OIDC 绑定 |
| `Admin` | OIDC 用户声称 admin=true，本地记录 admin=false | 以本地记录为准 | 可配置：以 OIDC claim 为准 / 以本地为准 |
| `Albums` | 本地用户有相册，OIDC 新建用户无相册 | 两个独立用户，互不影响 | 合并后应继承原用户的相册和收藏 |

#### 6.5.5 建议的合并架构改造

```sql
-- 新增 oauth_identities 表，实现 OIDC 与本地用户的多对多绑定
CREATE TABLE oauth_identities (
    id          INTEGER PRIMARY KEY,
    user_id     INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider    VARCHAR(64) NOT NULL,    -- 'authelia', 'keycloak', 'google' 等
    subject     VARCHAR(256) NOT NULL,   -- OIDC sub claim
    created_at  DATETIME,
    updated_at  DATETIME,
    UNIQUE(provider, subject)            -- 同一 provider 的 sub 唯一
);
```

```go
// OIDC 中间件改造：先查 oauth_identities，再查 users
func OIDCMiddleware(db *gorm.DB) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            remoteUser := r.Header.Get("X-Remote-User")
            if remoteUser == "" {
                next.ServeHTTP(w, r)
                return
            }

            // 路径 1：通过 oauth_identities 查找已绑定用户
            var identity OAuthIdentity
            if err := db.Where("provider = ? AND subject = ?", "proxy", remoteUser).
                First(&identity).Error; err == nil {
                // 已绑定，直接生成 token
                var user models.User
                db.First(&user, identity.UserID)
                // ... 生成 AccessToken
                return
            }

            // 路径 2：未绑定，查询是否已有同名用户
            var user models.User
            if err := db.Where("username = ?", remoteUser).First(&user).Error; err == nil {
                // 同名用户存在 → 拒绝自动创建，提示需要管理员关联
                http.Error(w, "Username conflicts with existing local user", http.StatusForbidden)
                return
            }

            // 路径 3：完全新用户，自动创建
            user, _ = models.RegisterUser(db, remoteUser, nil, false)
            db.Create(&OAuthIdentity{UserID: user.ID, Provider: "proxy", Subject: remoteUser})
            // ... 生成 AccessToken
        })
    }
}
```

---

## 六-A、登录失败风控与防爆破机制

### 6A.1 当前代码的防爆破现状

经过对 `AuthorizeUser`、`authorizeUser` resolver、`auth.Middleware` 以及整个中间件链的完整审查：

**文件**: `api/graphql/resolvers/user.go:24-54`

```go
func (r *mutationResolver) AuthorizeUser(ctx context.Context, username string, password string) (*models.AuthorizeResult, error) {
    db := r.DB(ctx)
    user, err := models.AuthorizeUser(db, username, password)
    if err != nil {
        // 登录失败：直接返回错误，没有任何计数或限制
        return &models.AuthorizeResult{
            Success: false,
            Status:  err.Error(),
        }, nil
    }
    // ... 生成 token
}
```

**文件**: `api/graphql/models/user.go:76-100`

```go
func AuthorizeUser(db *gorm.DB, username string, password string) (*User, error) {
    var user User
    result := db.Where("username = ?", username).First(&user)
    // 用户不存在 → 返回固定错误
    // 密码错误   → 返回固定错误
    // 没有任何延迟、计数、锁定逻辑
}
```

**完整中间件链**（`api/server.go:74-78`）：

```go
rootRouter := mux.NewRouter()
rootRouter.Use(dataloader.Middleware(db))
rootRouter.Use(auth.Middleware(db))
rootRouter.Use(server.LoggingMiddleware)
rootRouter.Use(server.CORSMiddleware(devMode))
// ❌ 没有 rate limiting 中间件
// ❌ 没有 login attempt 计数中间件
```

### 6A.2 项目中唯一的 Throttle 机制

**文件**: `api/utils/throttle.go:1-25`

```go
type Throttle struct {
    interval   time.Duration
    lastAction time.Time
}

func (t *Throttle) Trigger(action func()) {
    if time.Now().After(t.lastAction.Add(t.interval)) {
        t.lastAction = time.Now()
        action()
    }
}
```

这个 `Throttle` **只用于扫描器通知频率控制**（`api/scanner/scanner_tasks/notification_task.go:21`），与登录风控完全无关。

### 6A.3 缺失的防爆破能力清单

| 缺失能力 | 攻击面 | 影响 |
|---------|--------|------|
| 登录失败次数限制 | `authorizeUser` mutation | 无限次尝试密码 |
| 账号锁定机制 | 无 `login_attempts` / `locked_until` 字段 | 无法自动锁定被攻击账号 |
| IP 频率限制 | 无 per-IP rate limiter | 单 IP 可无限暴力破解 |
| 全局频率限制 | 无 per-endpoint rate limiter | 大规模分布式爆破无法遏制 |
| 登录失败延迟 | 无指数退避 | 快速尝试无惩罚 |
| 验证码 | 无 CAPTCHA 集成 | 无法区分人与机器 |
| 登录审计日志 | 无 `auth_audit_log` 表 | 无法事后追溯攻击 |

### 6A.4 暴力破解 PoC 路径

```
攻击路径（当前代码零防护）

攻击者 → POST /api/graphql
         {
           "query": "mutation { authorizeUser(username: \"admin\", password: \"password123\") { success } }"
         }
         ↓
         AuthorizeUser(db, "admin", "password123")
         ↓ bcrypt 比对（耗时 ~100ms，cost=12）
         失败 → 返回 { "success": false, "status": "invalid credentials" }
         ↓
         攻击者立即重试（无延迟、无限制）
         ↓
         每秒可尝试 ~10 次 → 6 分钟约 3600 次 → 覆盖常见弱密码字典
```

**唯一的自然延迟**：bcrypt cost=12 的哈希比对约 100ms/次，但这只是延迟攻击速度，并不阻止攻击。

### 6A.5 建议的防爆破改造

```
防爆破改造方案（三层递进）

┌─────────────────────────────────────────────────────────────────┐
│  第一层：应用层速率限制（优先实施）                               │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ 中间件：golang.org/x/time/rate 或 github.com/ulule/limiter │ │
│  │ - Per-IP: 10 次/分钟                                       │ │
│  │ - Per-username: 5 次/分钟                                   │ │
│  │ - 全局: 100 次/分钟                                         │ │
│  │ 超限 → HTTP 429 Too Many Requests                          │ │
│  └────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  第二层：登录失败计数与锁定                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ User 模型新增字段:                                          │ │
│  │   FailedLoginAttempts int     `gorm:"default:0"`           │ │
│  │   LockedUntil         *time.Time                           │ │
│  │                                                             │ │
│  │ AuthorizeUser 逻辑改造:                                     │ │
│  │   if LockedUntil != nil && now.Before(*LockedUntil) {      │ │
│  │     return nil, ErrAccountLocked                            │ │
│  │   }                                                         │ │
│  │   // 密码错误后:                                             │ │
│  │   FailedLoginAttempts++                                     │ │
│  │   if FailedLoginAttempts >= 5 {                             │ │
│  │     LockedUntil = now + 15 * time.Minute                    │ │
│  │   }                                                         │ │
│  └────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  第三层：审计日志（可选）                                        │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ 新增 auth_audit_logs 表:                                    │ │
│  │   user_id, ip, action, success, timestamp                   │ │
│  │ 用于事后追溯和安全分析                                       │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六-B、第三方撤销授权后本地 Token 清理链路

### 6B.1 OIDC Back-Channel Logout 的概念

OIDC 规范定义了 **Back-Channel Logout** 机制：当用户在 OIDC Provider 端注销或管理员撤销授权时，Provider 会主动向应用发送 logout 请求，应用据此清理本地会话。

### 6B.2 当前代码中的撤销链路现状

经过完整搜索（`oauth`、`revoke`、`backchannel`、`end_session`、`introspect` 等关键词），Photoview **没有任何第三方撤销授权的处理逻辑**：

| 缺失能力 | OIDC 规范对应 | 当前代码 |
|---------|-------------|---------|
| Back-Channel Logout 端点 | `backchannel_logout_uri` | ❌ 无 |
| Front-Channel Logout 支持 | `post_logout_redirect_uri` | ❌ 无 |
| Token Introspection | `GET /introspect` | ❌ 无 |
| Session 管理 | OIDC Session / `sid` claim | ❌ 无 |
| OIDC Provider → Photoview 的回调 | HTTP 端点 | ❌ 无 |

### 6B.3 反向代理模式下的撤销链路分析

在反向代理 OIDC 模式下，撤销授权的清理链路完全依赖反向代理：

```
第三方撤销授权的清理链路

场景 A：OIDC Provider 撤销用户会话（Back-Channel Logout）
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. OIDC Provider 发送 Back-Channel Logout 请求                 │
│     POST https://photoview.example.com/backchannel-logout       │
│     Body: { "sub": "alice", "sid": "xxx" }                     │
│                                                                 │
│  2. Photoview 当前行为:                                         │
│     ❌ 没有此端点 → 404                                        │
│     ❌ 无法接收撤销通知                                         │
│     ❌ 该用户的所有 AccessToken 继续有效直到 14 天过期          │
│                                                                 │
│  3. 反向代理行为:                                               │
│     如果 OIDC Provider → 反向代理有集成:                        │
│     ✅ 反向代理清除自己的 Session Cookie                        │
│     ✅ 用户下次访问时反向代理重新要求 OIDC 认证                 │
│     ❌ 但 Photoview 旧 AccessToken 仍然有效（如果被盗用）       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

场景 B：用户在 OIDC Provider 端主动注销（Front-Channel Logout）
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. OIDC Provider 在浏览器中加载 iframe:                        │
│     https://photoview.example.com/frontchannel-logout            │
│                                                                 │
│  2. Photoview 当前行为:                                         │
│     ❌ 没有此端点 → 404                                        │
│     ❌ 不会清除 auth-token Cookie                               │
│     ❌ 用户仍然可以访问 Photoview API（Cookie 仍有效）          │
│                                                                 │
│  3. 反向代理行为:                                               │
│     ✅ 反向代理可能清除自己的 Cookie                            │
│     ✅ 用户下次 API 请求时反向代理阻断                          │
│     ❌ 但如果请求绕过反向代理，旧 token 仍可用                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

场景 C：管理员在 OIDC Provider 中禁用/删除用户
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. OIDC Provider 触发 Back-Channel Logout                      │
│     → 同场景 A，Photoview 无法处理                              │
│                                                                 │
│  2. 反向代理不再为该用户签发认证 Header                         │
│     → 用户无法发起新请求                                       │
│     → 但已持有的 AccessToken 继续有效最长 14 天                 │
│                                                                 │
│  3. Photoview 端:                                              │
│     ❌ 不会主动删除该用户的 access_tokens 记录                  │
│     ❌ 不会断开该用户的 WebSocket 连接                          │
│     ❌ 唯一触发点: 用户被删除时 OnDelete:CASCADE                │
│        但 OIDC 禁用 ≠ Photoview 删除用户                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6B.4 撤销授权清理的完整缺口

```
撤销授权清理的 4 步缺口

┌─────────────────────────────────────────────────────────────────┐
│ Step 1: 接收撤销通知                                            │
│ ❌ 无 Back-Channel Logout 端点                                  │
│ ❌ 无 Front-Channel Logout 端点                                 │
│ ❌ 无 Webhook 回调                                              │
│ → Photoview 无法得知用户已被第三方撤销                          │
├─────────────────────────────────────────────────────────────────┤
│ Step 2: 解析撤销内容                                            │
│ ❌ 无 OIDC sub/sid 到 User 的映射表                             │
│ ❌ 无法将 OIDC 撤销通知关联到本地用户                          │
│ → 即使收到通知，也不知道该清理哪个用户                          │
├─────────────────────────────────────────────────────────────────┤
│ Step 3: 清理本地 Token                                          │
│ ❌ 无 DELETE FROM access_tokens WHERE user_id = ?               │
│ ❌ 无断开 WebSocket 连接的机制                                  │
│ ❌ 无清除 auth-token Cookie 的 Set-Cookie 响应                  │
│ → 旧会话持续有效                                                │
├─────────────────────────────────────────────────────────────────┤
│ Step 4: 确认清理完成                                            │
│ ❌ 无向 OIDC Provider 回复确认的机制                            │
│ → OIDC Provider 无法知道清理是否成功                            │
└─────────────────────────────────────────────────────────────────┘
```

### 6B.5 建议的撤销授权清理改造

```
改造方案：增加 Back-Channel Logout 端点 + OIDC Identity 映射

前提：需要 6.5.5 节的 oauth_identities 表（provider + subject → user_id 映射）

1. 新增 HTTP 端点：POST /api/backchannel-logout
   - 接收 OIDC Provider 的 logout_token (JWT)
   - 验证 JWT 签名和 claims
   - 提取 sub 和 sid
   - 查询 oauth_identities 找到本地 user_id
   - DELETE FROM access_tokens WHERE user_id = ?
   - 断开该用户的所有 WebSocket notification listeners
   - 返回 200 OK

2. 新增 HTTP 端点：GET /api/frontchannel-logout
   - 清除 auth-token Cookie
   - 重定向到首页

3. 可选：定期 Token Introspection
   - 后台 goroutine 每小时运行一次
   - 对所有 expire > now 的 AccessToken
   - 向反向代理/OIDC Provider 验证对应用户是否仍有效
   - 无效则 DELETE access_tokens 并断开 WebSocket
```

---

## 七、完整认证流程对比

| 流程步骤 | 本地账号鉴权 | OIDC 鉴权（反向代理模式） | Share 链接匿名访问 |
|---------|-------------|-------------------------|-----------------|
| 1. 用户访问 | 打开 /login | 访问任意页面 | 打开 /share/:token |
| 2. 认证方式 | 输入用户名密码 | 反向代理 OIDC 重定向 | 可选输入密码 |
| 3. 凭证验证 | `AuthorizeUser()` 验证 bcrypt | 反向代理验证 OIDC Token | `shareTokenValidatePassword` query |
| 4. 用户识别 | 通过 username 查询 User | 通过 X-Remote-User 查询 User | 无需用户，仅验证 ShareToken |
| 5. 账号检查 | `Password != nil` | `Password == nil` | ShareToken.Expire / Password |
| 6. Token 生成 | `GenerateAccessToken()` | `GenerateAccessToken()` | 生成时已创建 ShareToken |
| 7. 会话保持 | `auth-token` Cookie | `auth-token` Cookie | `share-token-pw-{token}` Cookie + URL token |
| 8. 过期时间 | 固定 14 天 | 固定 14 天 | 自定义或永不过期 |
| 9. 后续请求 | 中间件验证 Cookie → User Context | 中间件验证 Cookie → User Context | authenticateMedia → shareTokenFromRequest |
| 10. 过期失效 | 401 → 清除 Cookie → 重定向登录 | 401 → 清除 Cookie → 反向代理重新 OIDC | 显示 share expired 页面 |

---

## 八、GraphQL 认证指令

**文件**: `api/graphql/directive.go`

```go
// 需要管理员权限
func IsAdmin(ctx context.Context, obj interface{}, next graphql.Resolver) (res interface{}, err error) {
    user := auth.UserFromContext(ctx)
    if user == nil || user.Admin == false {
        return nil, errors.New("user must be admin")
    }
    return next(ctx)
}

// 需要登录权限
func IsAuthorized(ctx context.Context, obj interface{}, next graphql.Resolver) (res interface{}, err error) {
    user := auth.UserFromContext(ctx)
    if user == nil {
        return nil, auth.ErrUnauthorized
    }
    return next(ctx)
}
```

**Share 相关的 Mutation 都需要 `@isAuthorized`**，即只有登录用户才能创建/修改分享链接：
```graphql
extend type Mutation {
  shareAlbum(albumId: ID!, expire: Time, password: String): ShareToken! @isAuthorized
  shareMedia(mediaId: ID!, expire: Time, password: String): ShareToken! @isAuthorized
  deleteShareToken(token: String!): ShareToken! @isAuthorized
  protectShareToken(token: String!, password: String): ShareToken! @isAuthorized
  setExpireShareToken(token: String!, expire: Time): ShareToken! @isAuthorized
}
```

但 `shareToken` 和 `shareTokenValidatePassword` 这两个 Query **没有 directive**，允许匿名访问。

---

## 九、安全注意事项

### 9.1 OIDC 集成安全要点

1. **Header 信任范围**: 必须确保只有反向代理能设置 `X-Remote-User` Header
   - 配置防火墙只允许反向代理访问 Photoview
   - 或者在中间件中验证请求来源 IP

2. **用户区分**: 必须通过 `Password == nil` 来区分 OIDC 用户和本地用户
   - 本地用户必须有密码，不能通过 OIDC Header 登录
   - OIDC 用户必须无密码，不能通过本地登录页面登录

3. **自动创建用户风险**: 如果启用自动创建无密码用户，需要考虑：
   - OIDC Provider 是否已经对用户进行了授权
   - 是否需要管理员预先审批

4. **OIDC Logout 同步**: Photoview 清除 Cookie 后必须联动 OIDC Provider 登出，否则会立即自动重新登录

### 9.2 Token 管理的安全隐患

#### 9.2.1 修改密码不失效旧 Token（安全漏洞）

**当前实现**（`api/graphql/resolvers/user.go:110-145`）：

```go
func (r *mutationResolver) UpdateUser(ctx context.Context, id int, username *string, password *string, admin *bool) (*models.User, error) {
    db := r.DB(ctx)

    var user models.User
    if err := db.First(&user, id).Error; err != nil {
        return nil, err
    }

    // ... 更新 username、admin 字段

    if password != nil {
        // 只更新密码哈希，不触及 access_tokens 表
        hashedPassBytes, err := bcrypt.GenerateFromPassword([]byte(*password), 12)
        hashedPass := string(hashedPassBytes)
        user.Password = &hashedPass
    }

    if err := db.Save(&user).Error; err != nil {  // 只 UPDATE users 表
        return nil, fmt.Errorf("failed to update user: %w", err)
    }

    return &user, nil
}
```

**问题分析**：
- ❌ `db.Save(&user)` 只更新 `users` 表的记录
- ❌ **完全没有** `DELETE FROM access_tokens WHERE user_id = ?`
- ❌ 该用户的所有 AccessToken（可能分布在多个设备）继续有效直到 14 天自然过期
- ❌ 已建立的 WebSocket 连接继续有效直到断开

**攻击场景**：
1. 用户在公共设备登录，忘记 Logout，只在个人设备修改密码
2. 攻击者窃取了用户的 token（通过 XSS、本地文件泄露等）
3. 用户修改密码试图保护账号
4. 攻击者持有的旧 token 继续有效最长 14 天

#### 9.2.2 修复方案（具体改造点）

```go
// 改造后的 UpdateUser
func (r *mutationResolver) UpdateUser(ctx context.Context, id int, username *string, password *string, admin *bool) (*models.User, error) {
    db := r.DB(ctx)

    var user models.User
    if err := db.First(&user, id).Error; err != nil {
        return nil, err
    }

    // ... 其他字段更新 ...

    if password != nil {
        hashedPassBytes, err := bcrypt.GenerateFromPassword([]byte(*password), 12)
        if err != nil {
            return nil, err
        }
        hashedPass := string(hashedPassBytes)
        user.Password = &hashedPass
    }

    err := db.Transaction(func(tx *gorm.DB) error {
        if err := tx.Save(&user).Error; err != nil {
            return err
        }

        // ===== 新增：修改密码时清除该用户的所有 AccessToken =====
        if password != nil {
            if err := tx.Where("user_id = ?", id).Delete(&models.AccessToken{}).Error; err != nil {
                return fmt.Errorf("failed to revoke access tokens: %w", err)
            }

            // 可选：广播通知所有已连接的 WebSocket 客户端需要重新认证
            // notification.BroadcastNotification(&models.Notification{...})
        }

        return nil
    })

    return &user, nil
}
```

#### 9.2.3 补充：缺少的 Token 管理 API

除了修改密码时的自动清除，还缺少以下 Token 管理能力：

| 缺失功能 | 建议的 GraphQL Schema | 实现要点 |
|---------|----------------------|---------|
| 查看当前用户的所有活跃会话 | `myAccessTokens: [AccessToken!]! @isAuthorized` | 查询 `access_tokens` 表，过滤 `expire > now` |
| 吊销特定会话 | `revokeAccessToken(token: String!): Boolean! @isAuthorized` | `DELETE FROM access_tokens WHERE value = ? AND user_id = ?` |
| 吊销除当前会话外的所有会话 | `revokeOtherAccessTokens(currentToken: String!): Int! @isAuthorized` | `DELETE FROM access_tokens WHERE user_id = ? AND value != ?` |
| 管理员吊销特定用户所有会话 | `revokeAllUserAccessTokens(userId: ID!): Int! @isAdmin` | `DELETE FROM access_tokens WHERE user_id = ?` |
| 定期清理过期 Token | 后台 cron job | `DELETE FROM access_tokens WHERE expire < NOW() - INTERVAL '1 day'` |

#### 9.2.4 其他 Token 管理隐患

2. **无强制失效机制**:
   - 管理员无法吊销特定设备的登录会话
   - 数据库中过期 token 永远堆积，无清理任务

3. **WebSocket 会话窗口**:
   - WebSocket 连接建立时校验一次 token，之后即使 token 过期连接仍然有效
   - 最长可达 14 天 + 连接保持时间
   - 建议改造：在 KeepAlive ping 时附加 token 校验，或者给 context 绑定过期时间

4. **Share Token 安全**:
   - Share Token 仅 8 位字符（AccessToken 是 24 位），熵值较低
   - 建议：总是给 Share Token 设置密码和过期时间
   - 同样缺少定期清理过期 Share Token 的后台任务

### 9.3 Cookie SameSite 与 CSRF 防护协同

#### 9.3.1 Cookie SameSite 配置

**文件**: `ui/src/helpers/authentication.ts`

```typescript
const AUTH_TOKEN_COOKIE_NAME = 'auth-token'

export function saveTokenCookie(token: string) {
  const options = {
    path: '/',
    sameSite: 'Lax',       // 关键：Lax 模式
    expires: AUTH_TOKEN_MAX_AGE_IN_DAYS,
  }
  Cookies.set(AUTH_TOKEN_COOKIE_NAME, token, options)
}
```

**SameSite=Lax 的行为**：
- ✅ 同站请求：携带 Cookie
- ✅ 跨站顶级导航（GET，如点击链接跳转）：携带 Cookie
- ❌ 跨站 POST 请求：**不携带** Cookie
- ❌ 跨站 iframe、图片、AJAX：**不携带** Cookie

#### 9.3.2 CORS 配置

**文件**: `api/server/cors_middleware.go:12-55`

```go
func CORSMiddleware(devMode bool) mux.MiddlewareFunc {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
            if devMode {
                // 开发模式：允许任意来源
                w.Header().Set("Access-Control-Allow-Origin", req.Header.Get("origin"))
            } else {
                // 生产模式：仅允许 PHOTOVIEW_UI_ENDPOINT
                uiEndpoint := utils.UiEndpointUrl()
                if uiEndpoint != nil {
                    w.Header().Set("Access-Control-Allow-Origin",
                        uiEndpoint.Scheme+"://"+uiEndpoint.Host)
                }
            }

            w.Header().Set("Access-Control-Allow-Methods", "GET, POST, OPTIONS")
            w.Header().Set("Access-Control-Allow-Headers",
                "authorization, content-type, content-length, TokenPassword")
            w.Header().Set("Access-Control-Allow-Credentials", "true")
        })
    }
}
```

#### 9.3.3 WebSocket 跨站防护

**文件**: `api/server/websocket.go:12-47`

```go
func WebsocketUpgrader(devMode bool) websocket.Upgrader {
    return websocket.Upgrader{
        CheckOrigin: func(r *http.Request) bool {
            if devMode {
                return true  // 开发模式允许任意来源
            } else {
                uiEndpoint := utils.UiEndpointUrl()
                if uiEndpoint == nil || r.Header.Get("origin") == "" {
                    return true
                }
                originURL, _ := url.Parse(r.Header.Get("origin"))
                return isUIOnSameHost(uiEndpoint, originURL)
            }
        },
    }
}
```

#### 9.3.4 CSRF 防护协同机制

```
CSRF 防护三层协同
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  第一层：SameSite=Lax Cookie（被动防护）                              │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ 攻击者站点 evil.com 发起跨站 POST 到 photoview.com/api/graphql  │ │
│  │ → SameSite=Lax 阻止 auth-token Cookie 被携带                     │ │
│  │ → auth.Middleware 查不到 Cookie → user == nil                     │ │
│  │ → GraphQL directive @isAuthorized 返回 "unauthorized"            │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  第二层：CORS Origin 校验（主动防护）                                │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ 跨站请求 Origin: evil.com 到达服务器                              │ │
│  │ → 生产模式下 Origin 必须精确匹配 PHOTOVIEW_UI_ENDPOINT            │ │
│  │ → 不匹配则不设置 Access-Control-Allow-Origin                     │ │
│  │ → 浏览器拦截响应，前端拿不到数据                                  │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  第三层：WebSocket Origin 校验                                      │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ 跨站 WebSocket 握手请求 Origin: evil.com                         │ │
│  │ → CheckOrigin 校验不通过                                         │ │
│  │ → 服务端直接拒绝连接升级，返回 403                                │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 9.3.5 CSRF 防护的例外

1. **HTTP GET 请求**：SameSite=Lax 允许顶级导航的 GET 请求携带 Cookie
   - 但 `shareToken` 和 `shareTokenValidatePassword` 是 Query，不是 Mutation，无法修改数据
   - 媒体访问 `/photo/{name}` 是 GET，但需要 `?token=` Query 参数，攻击者无法伪造有效 token

2. **`credentials: 'include'`**：
   - **文件**: `ui/src/apolloClient.ts:26-29`
   ```typescript
   const httpLink = new HttpLink({
     uri: GRAPHQL_ENDPOINT,
     credentials: 'include',  // 允许跨站请求携带 Cookie（配合 CORS 使用）
   })
   ```
   - 这是为了支持前端部署在不同域名（如 CDN）的场景
   - 但 CORS 中间件严格限制了 Origin，只有白名单域名才能实际生效

#### 9.3.6 OIDC 集成对 CSRF 防护的影响

集成反向代理 OIDC 后，CSRF 防护栈变化：

| 场景 | 原生 Photoview | 反向代理 OIDC 模式 |
|------|---------------|-------------------|
| SameSite Cookie | Lax | Lax（不变） |
| CORS Origin 校验 | UI_ENDPOINT | 需要允许反向代理域名 |
| WebSocket Origin 校验 | UI_ENDPOINT | 需要允许反向代理域名 |
| 新增风险 | - | 反向代理本身可能引入 CSRF 漏洞 |

**注意**：反向代理（如 Nginx、Traefik）在转发请求时会修改 Origin header，需要确保：
1. 反向代理正确传递 `Origin` header
2. 反向代理自身有 CSRF 防护
3. `PHOTOVIEW_UI_ENDPOINT` 设置为反向代理的域名

### 9.4 现有安全机制

- 密码使用 bcrypt 哈希存储（cost=12）
- AccessToken 为 24 位加密安全随机字符串
- Cookie 使用 `SameSite: Lax` 防止 CSRF
- 所有媒体资源访问都经过鉴权中间件
- Share Token 密码同样使用 bcrypt 哈希
- CORS Origin 严格校验
- WebSocket Origin 严格校验

---

## 十、关键文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| User 模型 | `api/graphql/models/user.go` |
| AccessToken / ShareToken 模型 | `api/graphql/models/user.go` / `api/graphql/models/share_token.go` |
| 认证中间件（Cookie 校验） | `api/graphql/auth/auth.go` |
| Token Dataloader（过期检查） | `api/dataloader/userLoader.go` |
| 用户名密码登录 Resolver | `api/graphql/resolvers/user.go` |
| 更新用户/密码 Resolver | `api/graphql/resolvers/user.go:110-145` |
| Share Token Resolver | `api/graphql/resolvers/share_token.go` |
| Share Token Actions | `api/graphql/models/actions/share_token_actions.go` |
| 用户操作 Actions（含 DeleteUser） | `api/graphql/models/actions/user_actions.go` |
| 通知订阅 Resolver | `api/graphql/resolvers/notification.go` |
| 通知广播中心 | `api/graphql/notification/notification.go` |
| GraphQL 端点配置 | `api/graphql/endpoint/graphql_endpoint.go` |
| WebSocket 升级器与 Origin 校验 | `api/server/websocket.go` |
| CORS 中间件 | `api/server/cors_middleware.go` |
| Throttle 工具（仅用于扫描器） | `api/utils/throttle.go` |
| 服务器入口（中间件注册链） | `api/server.go` |
| 媒体/相册 HTTP 鉴权 | `api/routes/authenticate_routes.go` |
| 照片路由 | `api/routes/photos.go` |
| 下载路由 | `api/routes/downloads.go` |
| Apollo 错误处理（失效清除 Cookie） | `ui/src/apolloClient.ts` |
| 前端登录页 | `ui/src/Pages/LoginPage/LoginPage.tsx` |
| Share 页面 | `ui/src/Pages/SharePage/SharePage.tsx` |
| Share 密码保护页 | `ui/src/Pages/SharePage/PasswordProtectedShare.tsx` |
| 通知订阅 Hook | `ui/src/components/messages/SubscriptionsHook.ts` |
| 认证 Cookie 辅助函数 | `ui/src/helpers/authentication.ts` |
| 路由与 Logout 页面 | `ui/src/components/routes/Routes.tsx` |
| 受保护路由组件 | `ui/src/components/routes/AuthorizedRoute.tsx` |
| 用户设置页 Logout 按钮 | `ui/src/Pages/SettingsPage/UserPreferences.tsx` |
| GraphQL Schema（用户） | `api/graphql/resolvers/user.graphql` |
| GraphQL Schema（Share） | `api/graphql/resolvers/share_token.graphql` |
| GraphQL Schema（通知） | `api/graphql/resolvers/notification.graphql` |
| GraphQL 认证指令 | `api/graphql/directive.go` |
| SiteInfo 模型 | `api/graphql/models/site_info.go` |
| 环境变量 | `api/utils/environment_variables.go` |
| Token 生成工具函数 | `api/utils/utils.go` |

---

## 十一、总结

Photoview 的代码架构为 OIDC 集成提供了良好的基础，但在 Token 生命周期、多设备管理、WebSocket 安全上有明显缺失：

### 已有的基础能力
1. ✅ **用户模型支持无密码用户**：`Password *string` 字段设计是 OIDC 集成的关键
2. ✅ **三条鉴权入口完全解耦**：用户名密码登录、Share 匿名访问、反向代理 Header 认证互不干扰
3. ✅ **统一的会话机制**：基于 `auth-token` Cookie 的认证对所有登录方式透明
4. ✅ **Share 链接独立鉴权体系**：独立的 Token、密码、过期时间、路由保护
5. ✅ **清晰的权限体系**：`IsAuthorized` 和 `IsAdmin` 指令与认证方式解耦
6. ✅ **CSRF 三层防护协同**：SameSite Cookie + CORS Origin 校验 + WebSocket Origin 校验
7. ✅ **通知订阅体系完整**：WebSocket 订阅用于扫描进度等实时通知

### 本次补充的核心发现

#### OIDC 回调与本地账号合并
- ❌ **User.Username 有 unique 约束**，OIDC 用户名与本地用户名冲突时无法自动合并
- ❌ **没有 `oauth_identities` 映射表**，无法将 OIDC sub claim 关联到本地 User
- ❌ **没有合并逻辑**：当前唯一方案是管理员手动 `UPDATE users SET password = NULL`
- ⚠️ **场景 B（用户名冲突）**：OIDC 中间件 `Password != nil` 检查会静默跳过，用户无法理解原因
- ⚠️ **场景 C（先本地后 OIDC）**：清空密码后用户失去本地登录能力，不支持双认证方式
- ✅ **建议改造**：新增 `oauth_identities` 表（provider + subject → user_id），中间件先查映射再查用户

#### 登录失败风控与防爆破
- ❌ **AuthorizeUser 无任何失败计数**：直接返回错误，无延迟、无锁定
- ❌ **server.go 中间件链无 rate limiter**：仅 dataloader + auth + logging + CORS
- ❌ **唯一的 Throttle 只用于扫描器通知**（`api/utils/throttle.go`），与登录无关
- ❌ **暴力破解 PoC**：bcrypt cost=12 约 100ms/次，每秒可尝试 ~10 次，6 分钟覆盖常见弱密码
- ✅ **建议三层改造**：Per-IP 速率限制 → 登录失败计数与锁定 → 审计日志

#### 第三方撤销授权后本地 Token 清理
- ❌ **没有 Back-Channel Logout 端点**：OIDC Provider 撤销通知无法送达
- ❌ **没有 Front-Channel Logout 端点**：浏览器端注销无法清除 Cookie
- ❌ **没有 Token Introspection**：无法主动验证 token 是否仍被 OIDC Provider 认可
- ❌ **没有 OIDC sub/sid → User 映射**：即使收到通知也无法定位本地用户
- ❌ **撤销后 4 步全部缺失**：接收通知 → 解析内容 → 清理 Token → 确认完成
- ✅ **建议改造**：新增 backchannel-logout / frontchannel-logout 端点 + 定期 Introspection

#### WebSocket 订阅 Token 失效
- ❌ **InitFunc 只在连接建立时校验一次 token**，之后永不重新校验
- ❌ **KeepAlive 10 秒心跳不做 token 校验**，仅用于保持连接
- ❌ **服务端不会主动断开已过期 token 的连接**
- ❌ **修改密码、吊销 token 对已建立的 WebSocket 连接完全无影响**
- ✅ **唯一失效时机**：连接断开后的重连阶段才会重新校验 token
- ⚠️ **最长会话窗口**：14 天 + WebSocket 连接保持时间（理论上无限期）

#### Cookie SameSite 与 CSRF 防护协同
- ✅ **SameSite=Lax**：阻止跨站 POST 请求携带 Cookie，是 CSRF 的第一道防线
- ✅ **CORS 严格校验 Origin**：生产模式下只允许 `PHOTOVIEW_UI_ENDPOINT` 域名
- ✅ **WebSocket CheckOrigin**：同样校验 Origin，防止跨站 WebSocket 劫持
- ✅ **三层防护协同**：任意一层不通过都会阻止攻击
- ⚠️ **OIDC 集成注意**：反向代理转发时需正确传递 Origin header，需将 `PHOTOVIEW_UI_ENDPOINT` 设为反向代理域名

#### 修改密码不失效旧 Token（安全漏洞）
- ❌ **`UpdateUser` 只 UPDATE `users` 表**，完全不触及 `access_tokens` 表
- ❌ 该用户的**所有设备**上的 token 继续有效直到 14 天自然过期
- ❌ 已建立的 WebSocket 连接继续有效直到断开
- ✅ **修复方案明确**：在事务中更新密码后追加 `DELETE FROM access_tokens WHERE user_id = ?`
- ✅ **提供了完整的改造代码示例**和缺失的 Token 管理 API 建议

### 需要补齐的能力
1. ⚠️ **缺少 OIDC 中间件**：需要自行实现反向代理 Header 的解析和自动登录逻辑
2. ⚠️ **缺少 OIDC 用户管理 UI**：当前只能通过 GraphQL API 创建无密码用户
3. ❌ **没有 OIDC 账号合并机制**：Username unique 约束导致 OIDC 用户名与本地用户名冲突时无法自动合并，需新增 `oauth_identities` 映射表
4. ❌ **没有登录失败风控**：无速率限制、无失败计数、无账号锁定、无验证码，bcrypt cost=12 是唯一的自然延迟
5. ❌ **没有第三方撤销授权清理**：无 Back-Channel/Front-Channel Logout 端点、无 Token Introspection、无 OIDC sub→User 映射
6. ❌ **没有 Token 续期机制**：14 天后强制重新登录，无 refresh token，无滑动窗口
7. ❌ **没有会话管理功能**：无法查看、吊销特定设备的登录会话
8. ❌ **没有过期 Token 清理任务**：AccessToken 和 ShareToken 过期后永远留在数据库
9. ❌ **Logout 只清前端 Cookie**：不影响服务端和其他设备的会话
10. ❌ **修改密码不失效旧 Token**：安全漏洞，需在 UpdateUser 中追加 DELETE access_tokens
11. ❌ **WebSocket 无 token 周期校验**：需在心跳或 context 中绑定过期时间
12. ❌ **OIDC Logout 未联动**：清除本地 Cookie 后需同步跳转到 OIDC Provider 登出

OIDC 与本地账号鉴权是**并行关系**，通过 `Password` 字段是否为 `nil` 来区分。Share 链接是完全独立的第三条鉴权路径。三条路径最终都依赖相同的媒体访问控制层（`authenticateMedia`/`authenticateAlbum`），但在入口认证和会话保持上各自独立。
