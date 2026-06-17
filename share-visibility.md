# Photoview 分享链接与可见性机制

## 一、分享链接生成时记录的字段

### 数据库模型 (`ShareToken`)

> 代码位置：`api/graphql/models/share_token.go:7-18`

```
ShareToken
├── Model（基类）
│   ├── ID        int          主键自增
│   ├── CreatedAt time.Time
│   └── UpdatedAt time.Time
├── Value    string           随机令牌（8 位，大小写字母+数字）
├── OwnerID  int              创建者用户 ID（NOT NULL, 索引）
├── Owner    User             创建者（级联删除）
├── Expire   *time.Time       过期时间（可选，索引）
├── Password *string          密码的 bcrypt 哈希（可选）
├── AlbumID  *int             关联相册 ID（可选，索引）
├── Album    *Album           关联相册（级联删除）
├── MediaID  *int             关联媒体 ID（可选，索引）
└── Media    *Media           关联媒体（级联删除）
```

**互斥规则**：`AlbumID` 与 `MediaID` 二选一，分享链接要么指向一个相册，要么指向一个媒体，不会同时存在。

### 令牌生成流程

| 操作 | 入口 | 关键逻辑 |
|------|------|----------|
| 分享相册 | `Mutation.shareAlbum` → `actions.AddAlbumShare` | 校验 `user_albums` 中用户拥有该相册 → 生成 8 位随机 Value → bcrypt 哈希密码 → 写入 DB |
| 分享媒体 | `Mutation.shareMedia` → `actions.AddMediaShare` | 校验媒体的 Album 存在于 `user_albums` → 同上 |

> 代码位置：`api/graphql/models/actions/share_token_actions.go:15-103`

令牌值通过 `utils.GenerateToken()` 生成（`api/utils/utils.go:13-29`），仅 8 位字符，熵有限。

### 修改令牌的操作

| 操作 | 函数 | 说明 |
|------|------|------|
| 设置/清除密码 | `ProtectShareToken` | 传 `nil` 清除密码，否则 bcrypt 哈希后写入 |
| 设置/清除过期 | `SetExpireShareToken` | 传 `nil` 表示永不过期 |
| 删除令牌 | `DeleteShareToken` | 物理删除行 |

以上修改均需经过 `getUserToken` 校验，规则为：**令牌所有者本人 或 管理员**才能操作（`Owner.id = ? OR Owner.admin = TRUE`）。

---

## 二、外部访问命中链接后可见范围的判定

### 2.1 GraphQL 查询层的判定

访问相册或媒体详情时，Resolver 提供了**双通道**：

#### 查询相册 `Query.album(id, tokenCredentials)`

> 代码位置：`api/graphql/resolvers/album.go:129-162`

```
1. 如果携带 tokenCredentials → 调用 ShareToken() 验证令牌
   ├── 验证通过后，令牌指向的 Album.ID == id → 直接返回该相册
   └── 否则，检查 id 是否为令牌相册的子相册（递归 CTE GetChildren）
       └── 是 → 返回子相册（可见性下探到子相册）
2. 如果不携带 tokenCredentials → 走登录用户路径（见第三节）
```

**关键**：相册分享令牌的可见范围 = 该相册本身 + 所有递归子相册。

#### 查询媒体 `Query.media(id, tokenCredentials)`

> 代码位置：`api/graphql/resolvers/media.go:171-205`

```
1. 如果携带 tokenCredentials → 验证令牌
   └── 令牌的 MediaID == id → 直接返回该媒体
2. 否则 → 走登录用户路径
```

**注意**：媒体分享令牌只能访问那一个媒体，不能扩散。

#### 令牌验证核心 `Query.shareToken(credentials)`

> 代码位置：`api/graphql/resolvers/share_token.go:74-111`

```
1. 按 Value 查找令牌，Preload 所有关联
2. 检查过期（用 UTC 截断到秒比较）
   └── Expire != nil && now.After(*Expire) → "share expired"
3. 检查密码
   └── Password != nil → bcrypt 比对 credentials.Password
       ├── 不匹配 → "unauthorized"
       └── 匹配 → 通过
4. 返回完整 ShareToken（含 Album/Media 关联数据）
```

### 2.2 HTTP 资源层的判定（照片/视频/下载）

媒体文件的实际访问走 HTTP 路由，认证链如下：

```
请求到达 → auth.Middleware（从 cookie 读 auth-token 注入 User 到 context）
         → authenticateMedia / authenticateAlbum
```

> 代码位置：`api/routes/authenticate_routes.go:19-69`

#### `authenticateMedia(media, db, r)`

```
1. user = auth.UserFromContext(r)
2. if user != nil（已登录）:
   ├── 查出 media 所属 Album
   ├── user.OwnsAlbum(db, &album) 递归查 user_albums
   └── 不拥有 → 403
3. else（未登录）:
   └── shareTokenFromRequest(db, r, &media.ID, &media.AlbumID)
```

#### `authenticateAlbum(album, db, r)`

```
1. user = auth.UserFromContext(r)
2. if user != nil → user.OwnsAlbum(db, album)
3. else → shareTokenFromRequest(db, r, nil, &album.ID)
```

#### `shareTokenFromRequest` — 核心校验

> 代码位置：`api/routes/authenticate_routes.go:71-154`

```
1. 从 URL query 参数取 token（?token=xxx）
   └── 缺失 → 403
2. 按 Value 查找 ShareToken
   └── 不存在 → 403
3. 检查过期 time.Now().UTC().After(shareToken.Expire.UTC())
   └── 已过期 → 403
4. 检查密码
   ├── 从 cookie "share-token-pw-{token}" 取明文密码
   └── bcrypt 比对 → 不匹配 → 403
5. 校验资源匹配
   ├── AlbumID 令牌：请求的 albumID 必须等于令牌 AlbumID，或者是其子相册（递归 CTE）
   └── MediaID 令牌：请求的 mediaID 必须严格等于令牌 MediaID
6. 全部通过 → 200
```

**可见范围总结表**：

| 令牌类型 | 可访问范围 | 是否可扩散 |
|----------|-----------|-----------|
| 相册令牌 | 该相册 + 所有递归子相册内的媒体 | ✅ 子相册可访问 |
| 媒体令牌 | 仅该单个媒体文件 | ❌ 不可扩散 |

---

## 三、登录用户权限的叠加路径

### 3.1 身份注入

> 代码位置：`api/graphql/auth/auth.go:31-69`

```
HTTP 请求 → Middleware 从 cookie "auth-token" 读取
         → dataloader UserFromAccessToken.Load(cookieValue)
         → 查 access_tokens 表，关联 user
         → 注入 User 到 context
```

GraphQL WebSocket 连接从 `Authorization: Bearer <24位token>` 头读取，流程类似。

### 3.2 权限体系

#### 指令级拦截

| 指令 | 逻辑 | 位置 |
|------|------|------|
| `@isAuthorized` | `user != nil` | `api/graphql/directive.go:20-27` |
| `@isAdmin` | `user != nil && user.Admin == true` | `api/graphql/directive.go:11-18` |

#### 相册所有权判定 — `User.OwnsAlbum`

> 代码位置：`api/graphql/models/user.go:167-179`

```
User.OwnsAlbum(db, album) =
  album.GetParents(db, filter)
  其中 filter = EXISTS (SELECT 1 FROM user_albums WHERE user_id = ? AND album_id = id)
```

**逻辑**：从目标相册向上递归（CTE `super_albums`），如果任何一级祖先存在于 `user_albums` 关联表中，则该用户"拥有"此相册。

这意味着：用户只要拥有一个**父级相册**，就自动拥有对**子相册**的访问权。

#### 用户-相册关联模型

```
User ←── many2many: user_albums ──→ Album
       (user_id, album_id) 复合主键
```

Album 没有单一 OwnerID，而是多对多关系——一个相册可被多个用户"拥有"。

### 3.3 登录用户 vs 分享令牌的判定优先级

在所有关键入口中，**登录用户优先，分享令牌兜底**：

```
authenticateMedia / authenticateAlbum:
  if user != nil:
      → 仅用用户权限判定（OwnsAlbum）
      → 不检查分享令牌
  else:
      → 仅用分享令牌判定
```

**重要细节**：已登录用户访问资源时，即使 URL 带有 `?token=xxx`，系统也**不会**检查分享令牌。两条路径互斥，不存在叠加。

在 GraphQL 查询层（`Query.album` / `Query.media`）则不同：

```
1. 先尝试 tokenCredentials 通道
2. 如果令牌验证失败 → 直接返回错误，不会降级到用户权限
3. 如果没提供 tokenCredentials → 走用户权限
```

同样是互斥的——不会同时叠加令牌权限和用户权限。

### 3.4 管理员特殊权限

| 场景 | 权限 |
|------|------|
| 管理员操作分享令牌 | 可操作任何用户创建的令牌（`Owner.admin = TRUE`） |
| 管理员查看相册/媒体 | **不自动拥有**所有相册的访问权，仍需 OwnsAlbum 判定 |
| 用户管理 | `@isAdmin` 指令保护 |

---

## 四、完整权限判定流程图

```
                          请求到达
                             │
              ┌──────────────┴──────────────┐
              │ auth.Middleware 读取 cookie  │
              │ 尝试注入 User 到 context     │
              └──────────────┬──────────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
           User != nil               User == nil
           (已登录)                  (未登录)
                │                         │
    ┌───────────┴───────────┐    ┌────────┴────────┐
    │ User.OwnsAlbum?       │    │ URL ?token=xxx ? │
    │ (递归查 user_albums   │    │                  │
    │  从目标向上找祖先)     │    ├── 无 token → 403 │
    └───────────┬───────────┘    └────────┬────────┘
                │                         │
         ┌──────┴──────┐          ┌───────┴───────┐
         │             │          │               │
       拥有          不拥有     查 DB 找令牌     令牌不存在 → 403
         │             │          │               │
       200 ✅        403 ❌   ┌───┴────┐     ┌────┴─────┐
                              │        │     │          │
                           未过期   已过期  有密码    无密码
                              │        │     │          │
                              │      403 ❌  验证 cookie   │
                              │             │          │
                           ┌──┴──┐      通过/不通过    │
                           │     │                   │
                      AlbumID  MediaID              │
                           │     │                   │
                      检查目标   检查目标              │
                      是否匹配   是否严格匹配          │
                      含子相册   仅自身               │
                           │     │                   │
                      匹配 → 200 ✅               200 ✅
                      不匹配 → 403 ❌
```

---

## 五、Token 删除/过期后的外部访问响应与记录可查性

### 5.1 删除后的响应路径

**HTTP 资源层**（`api/routes/authenticate_routes.go:80-87`）：
- 按 `Value` 查找 `ShareToken`，若不存在则触发 `gorm.ErrRecordNotFound`
- 统一返回 `403 Forbidden`，响应消息 `"unauthorized"`，错误原因 `"invalid share token"`
- 外部访问者无法区分"令牌不存在"与"令牌过期/密码错误"，均为模糊的 403

**GraphQL 查询层**（`api/graphql/resolvers/share_token.go:76-82`）：
- `Query.shareToken`：找不到返回错误 `"share not found"`
- `Query.shareTokenValidatePassword`：找不到返回 `false` + 错误 `"share not found"`

**前端页面**（`ui/src/Pages/SharePage/SharePage.tsx:169-181`）：
- 检测到 `"share not found"` 错误时，展示友好提示页：
  - 标题：`Share not found`
  - 描述：`Maybe the share has expired or has been deleted.`
- 不区分删除与过期，统一归为"链接失效"

### 5.2 过期后的响应路径

**HTTP 资源层**（`api/routes/authenticate_routes.go:89-92`）：
- 判定逻辑：`time.Now().UTC().After(shareToken.Expire.UTC())`
- 同样返回 403 `"unauthorized"`，错误原因 `"invalid share token"`

**GraphQL 查询层**（`api/graphql/resolvers/share_token.go:84-98`）：
- 使用截断到秒的 UTC 时间比较（`fakeTime`），粒度为秒
- 返回错误 `"share expired"`（明确告知过期）

**注意**：HTTP 层对过期/不存在/密码错误均返回相同的 403 + "unauthorized"，而 GraphQL 层会明确区分 "share expired"、"share not found"、"unauthorized" 三种情况，粒度更细。

### 5.3 记录可查性

| 维度 | 现状 | 说明 |
|------|------|------|
| 软删除 | ❌ 不支持 | `DeleteShareToken` 为物理删除（`db.Delete(&token)`），删除后数据库无残留 |
| 过期自动清理 | ❌ 无机制 | 过期 token 仍留存于数据库，无后台任务定期清理 |
| 访问审计日志 | ❌ 无 | 无访问计数、访问 IP、访问时间等审计信息 |
| 分享历史 | ❌ 无 | 仅能查到当前有效的 token，无法追溯已删除的分享 |
| 查看权限 | ✅ 所有者/管理员 | `Album.Shares` / `Media.Shares` 仅返回当前登录用户创建的令牌；管理员可通过 `getUserToken` 操作任意令牌 |
| 级联清除 | ✅ 支持 | 删除用户/相册/媒体时，关联 token 级联删除 |

---

## 六、公开标志与 Token 链接的优先级判定

### 6.1 结论：不存在"公开标志"

经过对数据模型和 GraphQL Schema 的全面梳理，**Photoview 没有"公开标志"这一概念**：

- `ShareToken` 模型中无 `is_public`、`visibility` 等字段（`api/graphql/models/share_token.go:7-18`）
- `Album` / `Media` 模型中无公开/私有属性（`api/graphql/models/album.go:10-21`、`api/graphql/models/media.go`）
- `SiteInfo` 中无全站公开设置（`api/graphql/models/site_info.go:9-13`）
- 前端 UI 中的 "Public Link" 文字（`ui/src/components/sidebar/Sharing.tsx:525`）只是对分享链接的称呼，并非独立的权限维度

### 6.2 唯一的公开机制：分享令牌

所有外部访问都**必须**通过分享令牌实现，不存在"直接设为公开即可匿名访问"的模式：

```
访问方式          是否需要 token   鉴权方式
─────────────────────────────────────────────
 GraphQL 查询          是         tokenCredentials 参数
 HTTP 媒体文件         是         URL ?token=xxx 参数
 相册下载 ZIP          是         URL ?token=xxx 参数
 前端分享页面          是         URL path /share/{token}
```

### 6.3 多 Token 并存的优先级

一个相册/媒体可以同时存在**多个**分享令牌（通过 `shares` 查询可见），它们之间是**并列关系**，无优先级差异：

- 每个 token 独立拥有过期时间和密码
- 任意一个有效 token 都能解锁对应资源
- 删除其中一个不影响其他
- 前端"Add shares"按钮可无限追加新令牌

---

## 七、登录态携带非自身 Token 访问时的降级策略

### 7.1 HTTP 资源层：完全忽略 Token

在照片/视频/下载路由中，**只要用户已登录，URL 中的 `?token=xxx` 会被完全无视**。

> 代码位置：`api/routes/authenticate_routes.go:22-43`

```
authenticateMedia:
  if user != nil:
      → 只走 OwnsAlbum 判定
      → 不读取 URL 中的 token 参数
      → 不检查令牌有效性
```

**行为矩阵**：

| 登录态 | 相册权限 | URL 带 token | 结果 |
|--------|---------|-------------|------|
| 已登录 | ✅ 拥有 | 任意 | 200 正常访问 |
| 已登录 | ❌ 不拥有 | ✅ 有效 token | 403（token 被忽略） |
| 已登录 | ❌ 不拥有 | ❌ 无效 token | 403（token 被忽略） |
| 未登录 | — | ✅ 有效 token | 200 正常访问 |
| 未登录 | — | ❌ 无 token | 403 |

**没有降级策略**：两条路径完全互斥，不会因为"用户无权限"就 fallback 到令牌校验。

### 7.2 GraphQL 查询层：Token 优先，不降级

在 `Query.album` 和 `Query.media` 中，规则相反——**只要传了 `tokenCredentials`，就先走令牌路径，且失败不降级**。

> 代码位置：`api/graphql/resolvers/album.go:129-162`、`api/graphql/resolvers/media.go:171-205`

```
Query.album(id, tokenCredentials):
  if tokenCredentials != nil:
      → ShareToken() 验证
      → 成功 → 返回结果（完全不查用户权限）
      → 失败 → 直接报错返回，**不 fallback 到用户权限**
  else:
      → 走用户权限路径
```

**行为矩阵**：

| 登录态 | 传了 tokenCredentials | token 有效 | 用户是否拥有 | 结果 |
|--------|---------------------|-----------|------------|------|
| 已登录 | ✅ 是 | ✅ 有效 | 任意 | 返回 token 指向的资源 |
| 已登录 | ✅ 是 | ❌ 无效/过期 | ✅ 拥有 | 报错（不降级！） |
| 已登录 | ❌ 否 | — | ✅ 拥有 | 返回资源（用户权限） |
| 已登录 | ❌ 否 | — | ❌ 不拥有 | 报错 |
| 未登录 | ✅ 是 | ✅ 有效 | — | 返回 token 指向的资源 |
| 未登录 | ✅ 是 | ❌ 无效 | — | 报错 |

### 7.3 "非自身 Token" 的身份校验

**令牌验证完全不校验 Owner 是否为当前登录用户**：

- `shareTokenFromRequest` 只查 `value = ?`，不关联 `OwnerID`（`api/routes/authenticate_routes.go:80`）
- `ShareToken()` resolver 同样只按 `value` 查找，Preload 了 `Owner` 但仅用于返回，不做权限判定（`api/graphql/resolvers/share_token.go:76`）
- 只要知道 token 值、密码（如有）、未过期，任何人（无论登录与否）都能通过 token 访问资源

因此，"登录态携带非自身 token" 的场景：
- **HTTP 层**：token 被忽略，按登录用户自身权限走
- **GraphQL 层**：token 生效，按令牌权限走，与登录用户身份无关

唯一对 Owner 做校验的是**写操作**（`DeleteShareToken`、`ProtectShareToken`、`SetExpireShareToken`），需满足 `Owner.id = ? OR Owner.admin = TRUE`（`api/graphql/models/actions/share_token_actions.go:167-183`）。

---

## 八、安全注意事项（补充）

1. **令牌熵不足**：`GenerateToken()` 仅 8 位 62 进制字符（约 47.6 bit），暴力破解空间偏小，建议加长到至少 16 位或使用 UUID。
2. **已登录用户不检查令牌**：`authenticateMedia`/`authenticateAlbum` 中，登录用户路径与令牌路径互斥。如果登录用户恰好没有某个相册的权限，即使 URL 携带了有效分享令牌，也会被拒绝——这是一个潜在的体验问题。
3. **密码明文经过 cookie**：`shareTokenFromRequest` 从 cookie `share-token-pw-{token}` 取出明文密码做 bcrypt 比对，cookie 在每次请求中都会传输，需确保 HTTPS。
4. **时间比较忽略时区**：`ShareToken()` resolver 用截断到秒的 UTC 时间与 `Expire` 比较，而 `shareTokenFromRequest` 用 `time.Now().UTC()`，两处逻辑一致，但应保证 `Expire` 存储时也是 UTC。
5. **级联删除**：`ShareToken` 的 `Owner`、`Album`、`Media` 均设置 `OnDelete:CASCADE`，删除用户/相册/媒体时关联令牌自动清除。
6. **无访问审计**：分享链接的访问没有任何日志或计数机制，无法追踪谁通过链接查看了内容。
7. **物理删除无迹**：删除分享令牌为硬删除，无回收站、无操作日志，管理员无法追溯已删除的分享记录。
8. **过期 token 残留**：过期令牌不会自动清理，长期运行后数据库可能累积大量无效记录，且仍可被枚举探测。
9. **GraphQL 与 HTTP 层行为不一致**：HTTP 层对"不存在/过期/密码错"统一返回模糊的 403，而 GraphQL 层会明确区分错误类型，存在信息泄露粒度差异。
10. **登录态 + token 的互斥陷阱**：GraphQL 层传了 token 就只走 token 路径，失败不降级到用户权限；HTTP 层有登录态就忽略 token。两处行为相反，容易在前后端联调时产生预期偏差。
