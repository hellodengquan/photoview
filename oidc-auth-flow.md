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

---

## 二、本地账号鉴权流程

### 2.1 登录入口

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

### 2.2 密码验证逻辑

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

### 2.3 用户注册

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
```

### 3.3 认证中间件

**文件**: `api/graphql/auth/auth.go:31-70`

```go
func Middleware(db *gorm.DB) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if tokenCookie, err := r.Cookie("auth-token"); err == nil {
                loaders := dataloader.For(r.Context())
                user, err := loaders.UserFromAccessToken.Load(tokenCookie.Value)
                
                if user != nil {
                    ctx := AddUserToContext(r.Context(), user)
                    r = r.WithContext(ctx)
                }
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

### 3.4 Token 验证（Dataloader）

**文件**: `api/dataloader/userLoader.go:10-71`

```go
func NewUserLoaderByToken(db *gorm.DB) *UserLoader {
    return &UserLoader{
        fetch: func(tokens []string) ([]*models.User, []error) {
            // 1. 查询有效的 access_tokens（未过期）
            db.Where("expire > ?", time.Now()).Where("value IN (?)", tokens).Find(&accessTokens)
            // 2. 查询关联的用户
            // 3. 返回用户列表
        },
    }
}
```

### 3.5 WebSocket 认证

**文件**: `api/graphql/auth/auth.go:92-131`

```go
func AuthWebsocketInit() func(context.Context, transport.InitPayload) (context.Context, *transport.InitPayload, error) {
    return func(ctx context.Context, initPayload transport.InitPayload) (context.Context, *transport.InitPayload, error) {
        bearer, exists := initPayload["Authorization"].(string)
        token, err := TokenFromBearer(&bearer)  // 解析 Bearer token
        // ... 通过 dataloader 验证 token
        userCtx := context.WithValue(ctx, userCtxKey, user)
        return userCtx, nil, nil
    }
}
```

---

## 四、OIDC 集成架构

### 4.1 典型部署架构

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

### 4.2 OIDC 用户账号绑定

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

### 4.3 需要新增的 OIDC 中间件示例

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

---

## 五、完整认证流程对比

| 流程步骤 | 本地账号鉴权 | OIDC 鉴权（反向代理模式） |
|---------|-------------|-------------------------|
| 1. 用户访问 | 打开登录页面 | 访问任意页面 |
| 2. 认证方式 | 输入用户名密码 | 重定向到 OIDC Provider 登录 |
| 3. 凭证验证 | `AuthorizeUser()` 验证 bcrypt 密码 | 反向代理验证 OIDC Token |
| 4. 用户识别 | 通过用户名查询用户 | 通过 `X-Remote-User` Header 查询用户 |
| 5. 账号检查 | 检查 `Password != nil` | 检查 `Password == nil` |
| 6. Token 生成 | `GenerateAccessToken()` | `GenerateAccessToken()` |
| 7. 会话保持 | `auth-token` Cookie | `auth-token` Cookie |
| 8. 后续请求 | 中间件验证 Cookie | 中间件验证 Cookie |

---

## 六、GraphQL 认证指令

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

---

## 七、安全注意事项

### 7.1 OIDC 集成安全要点

1. **Header 信任范围**: 必须确保只有反向代理能设置 `X-Remote-User` Header
   - 配置防火墙只允许反向代理访问 Photoview
   - 或者在中间件中验证请求来源 IP

2. **用户区分**: 必须通过 `Password == nil` 来区分 OIDC 用户和本地用户
   - 本地用户必须有密码，不能通过 OIDC Header 登录
   - OIDC 用户必须无密码，不能通过本地登录页面登录

3. **自动创建用户风险**: 如果启用自动创建无密码用户，需要考虑：
   - OIDC Provider 是否已经对用户进行了授权
   - 是否需要管理员预先审批

### 7.2 现有安全机制

- 密码使用 bcrypt 哈希存储（cost=12）
- AccessToken 为 24 位加密安全随机字符串
- Cookie 使用 `SameSite: Lax` 防止 CSRF
- 所有媒体资源访问都经过鉴权中间件

---

## 八、关键文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| User 模型 | `api/graphql/models/user.go` |
| 认证中间件 | `api/graphql/auth/auth.go` |
| GraphQL Resolvers | `api/graphql/resolvers/user.go` |
| Token Dataloader | `api/dataloader/userLoader.go` |
| 媒体鉴权 | `api/routes/authenticate_routes.go` |
| 前端登录 | `ui/src/Pages/LoginPage/LoginPage.tsx` |
| 认证辅助 | `ui/src/helpers/authentication.ts` |
| GraphQL Schema | `api/graphql/resolvers/user.graphql` |
| 服务器入口 | `api/server.go` |
| 环境变量 | `api/utils/environment_variables.go` |

---

## 九、总结

Photoview 的代码架构为 OIDC 集成提供了良好的基础：

1. ✅ **用户模型支持无密码用户**：`Password *string` 字段设计
2. ✅ **统一的会话机制**：基于 `auth-token` Cookie 的认证对所有鉴权方式透明
3. ✅ **清晰的权限体系**：`IsAuthorized` 和 `IsAdmin` 指令与认证方式解耦
4. ⚠️ **缺少 OIDC 中间件**：需要自行实现反向代理 Header 的解析和自动登录逻辑
5. ⚠️ **缺少 OIDC 用户管理 UI**：当前只能通过 GraphQL API 创建无密码用户

OIDC 与本地账号鉴权是**并行关系**，通过 `Password` 字段是否为 `nil` 来区分。两种方式最终都会生成 `AccessToken` 并通过相同的 Cookie 机制维持会话。
