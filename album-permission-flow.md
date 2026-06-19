# Photoview 相册、权限与用户隔离流程分析

## 1. 核心数据模型

### 1.1 User（用户）

**文件**: `api/graphql/models/user.go`

```
User {
    ID        int       (主键)
    Username  string    (唯一, 最大128字符)
    Password  *string   (bcrypt 哈希, 可为空)
    Albums    []Album   (多对多, 通过 user_albums 关联)
    Admin     bool      (默认 false)
}
```

**关联表 `user_albums`**:
```
UserAlbums {
    UserID  int  (联合主键, ON DELETE CASCADE)
    AlbumID int  (联合主键, ON DELETE CASCADE)
}
```

**关键方法**:
- `FillAlbums(db)` — 延迟加载用户拥有的相册列表
- `OwnsAlbum(db, album)` — 递归判断用户是否拥有某相册（含父级链上的间接所有权）
- `FavoriteMedia(db, mediaID, favorite)` — 收藏/取消收藏媒体（写入 `user_media_data`）

### 1.2 Album（相册）

**文件**: `api/graphql/models/album.go`

```
Album {
    ID            int     (主键)
    Title         string  (非空)
    ParentAlbumID *int    (父相册ID, 可为空, 有索引)
    ParentAlbum   *Album  (自引用, ON DELETE SET NULL)
    Owners        []User  (多对多, 通过 user_albums 关联)
    Path          string  (文件系统路径, 非空)
    PathHash      string  (MD5(Path), 唯一)
    CoverID       *int    (自定义封面 media ID)
}
```

**关键方法**:
- `GetChildren(db, filter)` — 递归 CTE 查询所有子相册，可传入 filter 做权限过滤
- `GetParents(db, filter)` — 递归 CTE 查询所有父相册，可传入 filter 做权限过滤
- `Thumbnail(db)` — 获取相册封面（优先 CoverID，否则取最新子媒体）

### 1.3 ShareToken（共享令牌）

**文件**: `api/graphql/models/share_token.go`

```
ShareToken {
    ID       int        (主键)
    Value    string     (令牌值)
    OwnerID  int        (创建者用户ID)
    Owner    User       (ON DELETE CASCADE)
    Expire   *time.Time (过期时间, 可为空)
    Password *string    (访问密码 bcrypt 哈希, 可为空)
    AlbumID  *int       (关联相册, 可为空)
    Album    *Album     (ON DELETE CASCADE)
    MediaID  *int       (关联媒体, 可为空)
    Media    *Media     (ON DELETE CASCADE)
}
```

ShareToken 是"共享相册"的核心：AlbumID 和 MediaID 二选一，实现对单个媒体或整个相册（含子相册）的外部分享。

### 1.4 UserMediaData（用户-媒体数据）

```
UserMediaData {
    UserID   int  (联合主键)
    MediaID  int  (联合主键)
    Favorite bool (默认 false)
}
```

每用户独立的收藏状态，是实现用户隔离的"个人维度数据"。

---

## 2. 认证流程

### 2.1 登录获取 Token

```
用户名+密码 → AuthorizeUser() (user.go:76)
    → 数据库查询用户 → bcrypt 校验密码
    → 成功后 GenerateAccessToken() (user.go:126)
    → 生成24位随机字符串，过期时间14天
    → 返回 AccessToken{UserID, Value, Expire}
```

**代码入口**: `api/graphql/resolvers/user.go:24` `AuthorizeUser` mutation

### 2.2 请求鉴权中间件

**文件**: `api/graphql/auth/auth.go`

```
HTTP请求 → auth.Middleware(db)
    → 读取 Cookie "auth-token"
    → dataloader 批量查询 AccessToken 是否有效且未过期
    → 有效: 将 User 存入 context
    → 无效/不存在: context 中无用户信息
```

WebSocket 鉴权走 `AuthWebsocketInit()`，从 `Authorization: Bearer <token>` 提取 token，逻辑相同。

### 2.3 GraphQL 指令级鉴权

**文件**: `api/graphql/directive.go`

- `@isAuthorized` — 检查 context 中有 User，否则返回 `ErrUnauthorized`
- `@isAdmin` — 检查 context 中有 User 且 `Admin == true`

### 2.4 Token → User 的 DataLoader

**文件**: `api/dataloader/userLoader.go`

批量查询 `access_tokens` 表（过滤 `expire > NOW()`），再批量查 `users` 表，按 token 顺序返回 `[]*User`。

---

## 3. 用户-相册所有权（核心权限模型）

### 3.1 多对多关系

`user_albums` 表是权限的基石：**一行记录 = 一个用户对一个相册的直接所有权**。

```
用户A ──owns──→ 相册X
用户B ──owns──→ 相册X    ← 同一相册可有多个 Owner
用户A ──owns──→ 相册Y
```

### 3.2 OwnsAlbum：递归所有权判定

**文件**: `api/graphql/models/user.go:167`

```go
func (user *User) OwnsAlbum(db *gorm.DB, album *Album) (bool, error) {
    filter := func(query *gorm.DB) *gorm.DB {
        return query.Where(
            "EXISTS (SELECT 1 FROM user_albums WHERE user_albums.user_id = ? AND user_albums.album_id = id LIMIT 1)",
            user.ID)
    }
    ownedParents, err := album.GetParents(db, filter)
    return len(ownedParents) > 0, nil
}
```

**逻辑**：
1. 对目标相册执行递归 CTE `GetParents`，向上遍历整条父级链
2. 在 CTE 结果上应用 filter：只保留在 `user_albums` 中与当前用户关联的记录
3. 如果父级链上有任何一个相册属于该用户 → 返回 true

**这意味着**：如果用户拥有父相册，则自动拥有所有子相册的访问权。不需要显式给子相册分配所有权。

### 3.3 相册树的父子关系

相册通过 `ParentAlbumID` 形成树结构。扫描时自动建立：

```
/Photos              ← 根相册 (ParentAlbumID = NULL)
  ├── /Photos/2023   ← 子相册 (ParentAlbumID = 根相册ID)
  └── /Photos/2024   ← 子相册
```

---

## 4. 相册扫描与所有权分配

### 4.1 新建根相册

**文件**: `api/scanner/scanner_album.go:19` `NewRootAlbum()`

```
1. 校验路径有效 (ValidRootPath)
2. 按 PathHash 查找现有相册
   - 已存在: 将用户追加为 Owner (Association("Albums").Append)
   - 不存在: 创建新相册，Owner 为当前用户
```

**关键场景**: 两个用户指向同一文件系统路径时，相册记录只有一条，但 `user_albums` 中有两行，两个用户共享该相册。

### 4.2 递归扫描子相册

**文件**: `api/scanner/scanner_user.go:45` `FindAlbumsForUser()`

```
1. FillAlbums() 加载用户根相册列表
2. 遍历根相册，BFS 扫描子目录
3. 对每个子目录:
   - 按 PathHash 查找是否已有相册记录
   - 新建: 继承父相册的所有 Owners
   - 已存在: 将当前用户追加为 Owner（如果还不是）
4. 最后执行 DeleteOldUserAlbums 清理不再存在的关联
```

**继承 Owner 的代码** (`scanner_user.go:143-165`):
```go
if albumParent != nil {
    albumParentID = &albumParent.ID
    tx.Model(&albumParent).Association("Owners").Find(&parentOwners)
}
album = &models.Album{Title, ParentAlbumID, Path}
tx.Create(&album)
tx.Model(&album).Association("Owners").Append(parentOwners)
```

**关键推论**:
- 子相册的 Owners = 父相册的 Owners（创建时继承）
- 如果两个用户的路径树有交集，交集部分的子相册会有两个 Owner
- 已存在的子相册在扫描时只会追加当前用户为 Owner，不会清除原有 Owner

### 4.3 删除用户时清理相册

**文件**: `api/graphql/models/actions/user_actions.go:14` `DeleteUser()`

```
1. 不允许删除唯一的管理员
2. 事务中:
   a. 清除用户与相册的关联 (Association("Albums").Clear)
   b. 遍历用户相册，检查是否还有其他 Owner
   c. 无其他 Owner → 删除相册记录 + 清理缓存
3. 保留仍被其他用户拥有的相册
```

### 4.4 移除用户的根相册

**文件**: `api/graphql/resolvers/user.go:193` `UserRemoveRootAlbum()`

```
1. 删除 user_albums 中 user_id + album_id 的记录
2. 递归获取子相册，同样删除 user_albums 关联
3. 清理不再有 Owner 的相册
```

---

## 5. 共享相册机制（ShareToken）

### 5.1 创建共享

**文件**: `api/graphql/models/actions/share_token_actions.go`

#### 共享相册 `AddAlbumShare()`
```
1. 验证用户拥有该相册 (user_albums 关联)
2. 生成随机 token
3. 可选: 设置过期时间、访问密码
4. 写入 ShareToken{OwnerID, AlbumID, Expire, Password}
```

#### 共享媒体 `AddMediaShare()`
```
1. 验证用户拥有该媒体所属相册 (Joins("Album").Where(user_albums))
2. 生成随机 token
3. 写入 ShareToken{OwnerID, MediaID, Expire, Password}
```

### 5.2 通过共享访问

**GraphQL 解析器层** (`api/graphql/resolvers/album.go:129`):

```
query Album(id, tokenCredentials) {
    if tokenCredentials != nil {
        1. 验证 ShareToken (过期、密码)
        2. 如果是相册共享:
           - 直接匹配 albumID
           - 或递归查找子相册
        3. 如果是媒体共享:
           - 直接匹配 mediaID
    }
    // 无 token 则走用户认证
    user := auth.UserFromContext(ctx)
    actions.Album(db, user, id) → OwnsAlbum 校验
}
```

**HTTP 路由层** (`api/routes/authenticate_routes.go`):

```
authenticateMedia(media, db, r):
    if user != nil:
        → OwnsAlbum 校验
    else:
        → shareTokenFromRequest 校验

authenticateAlbum(album, db, r):
    同上，区分已登录用户 vs 共享 token
```

### 5.3 共享 Token 验证流程

`shareTokenFromRequest()` (`authenticate_routes.go:71`):

```
1. 从 URL 参数 ?token=xxx 获取 token
2. 数据库查询 ShareToken
3. 检查过期时间
4. 如有密码 → 从 Cookie "share-token-pw-<value>" 取密码 → bcrypt 验证
5. 类型匹配:
   - AlbumID: 直接匹配 或 递归 CTE 匹配子相册
   - MediaID: 必须精确匹配
```

### 5.4 管理 ShareToken

- `DeleteShareToken` — 只有 token Owner 或管理员可以删除
- `ProtectShareToken` — 设置/修改密码（Owner 或管理员）
- `SetExpireShareToken` — 设置过期时间（Owner 或管理员）

权限检查 (`share_token_actions.go:167`):
```go
// 只允许 Owner 本人或管理员操作
query: "Owner.id = ? OR Owner.admin = TRUE"
```

---

## 6. 媒体可见性与用户隔离

### 6.1 核心原则

**媒体的可见性完全由其所属相册（album_id）的用户所有权决定。**

```
Media → Album → user_albums → User
```

媒体本身没有独立的权限表，所有隔离都通过 `user_albums` 间接实现。

### 6.2 各查询的隔离实现

| 查询 | 文件 | 隔离方式 |
|------|------|----------|
| `MyAlbums` | `album_actions.go:9` | `WHERE id IN (用户相册IDs)` |
| `Album(id)` | `album_actions.go:83` | `OwnsAlbum()` 递归校验 |
| `AlbumPath` | `album_actions.go:104` | 遍历路径上每个相册，`OwnsAlbum()` 逐个校验，截断无权限部分 |
| `MyMedia` | `media_actions.go:8` | `WHERE album_id IN (SELECT album_id FROM user_albums WHERE user_id = ?)` |
| `Media(id)` | `resolvers/media.go:171` | 先检查 ShareToken，否则 `WHERE EXISTS (SELECT FROM user_albums WHERE album_id = media.album_id AND user_id = ?)` |
| `MediaList` | `resolvers/media.go:208` | `LEFT JOIN user_albums ... WHERE user_id = ?` |
| `MyTimeline` | `timeline_actions.go:11` | `WHERE albums.id IN (SELECT album_id FROM user_albums WHERE user_id = ?)` |
| `Search` | `search_actions.go:13` | 媒体和相册都通过 `EXISTS (SELECT FROM user_albums WHERE user_id = ?)` 过滤 |

### 6.3 子相册的可见性

子相册不需要在 `user_albums` 中有记录即可被访问——只要父级链上有所有权，`OwnsAlbum()` 的递归 CTE 就会放行。

但 `MyAlbums(onlyRoot=true)` 只返回 `user_albums` 中直接关联的根相册，子相册通过 GraphQL `SubAlbums` 字段展开。

### 6.4 收藏的隔离

`UserMediaData` 以 `(UserID, MediaID)` 为联合主键，收藏状态完全用户隔离：

```
用户A 收藏 Media1 → UserMediaData{UserID: A, MediaID: 1, Favorite: true}
用户B 未收藏 Media1 → 无记录或 Favorite: false
```

`onlyFavorites` 过滤也绑定当前用户 ID：
```go
db.Model(&UserMediaData{UserID: user.ID}).
    Where("user_media_data.media_id = media.id").
    Where("user_media_data.favorite = true")
```

### 6.5 人脸数据的隔离（详见第 12 章）

人脸检测（`ImageFace`）附着在 `Media` 上，`FaceGroup` 是全局分组（无 user_id 字段）。读取和写入操作的隔离通过"用户相册 ID 过滤"在查询层实现。详见第 12 章的深度分析。

---

## 7. 管理员权限

### 7.1 Admin 标记

`User.Admin` 布尔字段，在 GraphQL 层通过 `@isAdmin` 指令控制：
- 用户管理 (CRUD)
- 站点信息修改
- 扫描触发

### 7.2 管理员与相册所有权

**管理员并不自动拥有所有相册。** 管理员仍需通过 `user_albums` 显式关联才能访问相册。

但在 ShareToken 管理中，管理员可以操作任何人的 token：
```go
// getUserToken 的查询条件
"Owner.id = ? OR Owner.admin = TRUE"
```

### 7.3 用户查询的隔离缺陷

`query User()` (`resolvers/user.go:276`) 返回所有用户，没有按 Admin 过滤。这是 GraphQL schema 层 `@isAdmin` 指令保护的，非管理员无法调用。

---

## 8. 完整权限判定流程图

```
┌─────────────────────────────────────────────────────────┐
│                    请求进入                                │
│                  (HTTP / GraphQL)                        │
└─────────────┬───────────────────────────────────────────┘
              │
              ▼
┌──────────────────────┐
│ auth.Middleware       │  读取 Cookie "auth-token"
│ 解析 AccessToken      │  → User 存入 Context
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐     ┌──────────────────────┐
│ 有 User Context?      │──否──▶│ 检查 ShareToken       │
│                      │     │ (URL ?token=xxx)      │
└──────┬───────────────┘     └──────────┬───────────┘
       │ 是                              │
       ▼                                 ▼
┌──────────────────────┐     ┌──────────────────────┐
│ OwnsAlbum()          │     │ 验证 Token 有效性      │
│ 递归 CTE 查父级链     │     │ 检查过期时间           │
│ user_albums 匹配     │     │ 验证密码 (如有)        │
└──────┬───────────────┘     │ 匹配 AlbumID/MediaID  │
       │                     └──────────┬───────────┘
       ▼                                ▼
┌──────────────────────────────────────────────────────┐
│              权限校验通过 → 返回数据                      │
│              权限校验失败 → 403 / ErrUnauthorized       │
└──────────────────────────────────────────────────────┘
```

---

## 9. 多用户共享相册的场景

### 场景：两个用户有部分重叠的目录

```
用户A: /photos        (根路径)
用户B: /photos/shared (根路径)
```

**扫描后**:
1. 扫描用户A → 创建相册 `/photos` (Owner: A) 和 `/photos/shared` (Owner: A)
2. 扫描用户B → `/photos/shared` 已存在，追加 B 为 Owner
3. 最终 `user_albums`:
   - A → `/photos`
   - A → `/photos/shared`
   - B → `/photos/shared`

**结果**:
- 用户A 能看到 `/photos` 和 `/photos/shared`
- 用户B 只能看到 `/photos/shared`
- 两者都看到 `/photos/shared` 下相同的媒体

### 场景：子相册的间接所有权

```
用户A 拥有相册 /vacation (user_albums 有记录)
    └── /vacation/2023 (user_albums 无记录)
    └── /vacation/2024 (user_albums 无记录)
```

用户A 访问 `/vacation/2023` 时，`OwnsAlbum()` 递归查父级链，发现 `/vacation` 属于自己 → 允许访问。

---

## 10. 关键发现与潜在风险

### 10.1 SubAlbums 未做权限过滤

`album.go:54` `SubAlbums` 解析器直接查 `parent_album_id = ?`，未检查子相册是否属于当前用户。不过由于 `Album(id)` 查询需要 `OwnsAlbum` 校验，攻击者无法获取子相册内容，但可能泄露子相册的存在和标题。

### 10.2 Shares 字段无权限保护

`album.go:89` `Shares` 解析器直接按 `album_id` 查询 ShareToken，未校验当前用户是否拥有该相册。任何已认证用户若知道 album_id，理论上可以枚举共享链接。

### 10.3 Media.Shares 同理

`media.go:89` `Shares` 直接按 `media_id` 查询，无所有权校验。

### 10.4 AlbumPath 的截断逻辑

`album_actions.go:117` 从根向叶遍历路径，遇到无权限的相册则截断。这意味着如果路径中间有一个不属于用户的相册，该相册及其上层路径对用户不可见。

### 10.5 管理员与相册访问

管理员不会自动获得所有相册的访问权。这是设计决策，但可能不符合某些部署场景的预期。

### 10.6 ShareToken 的密码通过 Cookie 传递

共享密码通过 Cookie `share-token-pw-<value>` 传递，而非 HTTP Header。这意味着密码验证依赖浏览器的 Cookie 机制。CORS 配置允许 `TokenPassword` header（`cors_middleware.go:46`），但实际验证逻辑走的是 Cookie 路径。

### 10.7 不存在定期清理过期 ShareToken 的后台任务

过期 ShareToken 仅在使用时被拦截（GraphQL `ShareToken` query 和 HTTP `shareTokenFromRequest` 均检查 `Expire`），但没有任何 periodic job 或 cron 会从数据库中物理删除过期行。过期 token 会永久堆积。

---

## 11. ShareToken 外链：生成 → 分发 → 校验 → 撤销的完整挂载点

### 11.1 生成：Token 随机字符串的算法

**文件**: `api/utils/utils.go:13` `GenerateToken()`

```go
const charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
const length = 8
// 使用 crypto/rand 从 charset 中随机挑选 8 个字符
```

- **熵值**: 62^8 ≈ 2.18 × 10^14 ≈ 48 bits
- **来源**: `crypto/rand` 的 `rand.Int()`，加密安全

**注意**: ShareToken 仅 8 字符（对比 AccessToken 是 24 字符），强度偏低。配合过期时间和可选密码使用。

**所有调用点**:
| 位置 | 用途 |
|------|------|
| `share_token_actions.go:46` | `AddMediaShare()` 创建媒体外链 |
| `share_token_actions.go:90` | `AddAlbumShare()` 创建相册外链 |
| `notification_task.go:25` | 通知推送的唯一标识 key（非 token） |
| `scanner_error.go:18` | 扫描错误唯一键（非 token） |
| `processing_helpers.go:35/47` | 媒体 URL 随机后缀 |
| `process_video_task.go:88/133` | 视频文件名随机后缀 |

### 11.2 创建 ShareToken 的 GraphQL 入口

**GraphQL 挂载点（schema 层）**:
- `mutation ShareAlbum(albumId: Int!, expire: Time, password: String): ShareToken`
- `mutation ShareMedia(mediaId: Int!, expire: Time, password: String): ShareToken`

**Resolver 层**: `api/graphql/resolvers/share_token.go`

```
ShareAlbum(albumID, expire, password)        share_token.go:24
    └─ auth.UserFromContext → 必须登录
    └─ actions.AddAlbumShare(db, user, ...)  share_token_actions.go:61
         └─ 权限校验: user_albums 中存在关联
         └─ hashSharePassword(password) → bcrypt cost=12
         └─ 构造 ShareToken{
              Value:   utils.GenerateToken(),
              OwnerID: user.ID,
              Expire:  expire,
              Password: hashedPassword,
              AlbumID:  &albumID,
              MediaID:  nil
            }
         └─ db.Create(&shareToken)

ShareMedia(mediaID, expire, password)        share_token.go:34
    └─ 同上流程 → actions.AddMediaShare      share_token_actions.go:15
         └─ 权限校验: Joins("Album") + user_albums 关联
         └─ ShareToken{ AlbumID: nil, MediaID: &mediaID }
```

### 11.3 校验：两条并行路径

**路径 A：GraphQL 查询层（元数据读取）**

**挂载点**:
- `query Album(id: Int!, tokenCredentials: ShareTokenCredentials)`
- `query Media(id: Int!, tokenCredentials: ShareTokenCredentials)`
- `query ShareToken(credentials: ShareTokenCredentials)`
- `query ShareTokenValidatePassword(credentials: ShareTokenCredentials): Boolean`

**流程** (`resolvers/share_token.go:74` `ShareToken` query):

```
1. Preload(clause.Associations) 加载 ShareToken + Album + Media + Owner
2. 过期检查: 构造 fakeTime (当天 UTC 00:00:00) 对比 token.Expire
   *注意: GraphQL 与 HTTP 的过期判定时间粒度不同！见 11.5*
3. 密码检查:
   - token.Password != nil: bcrypt.CompareHashAndPassword(credentials.Password)
   - 不匹配 → 返回 errors.New("unauthorized")
4. 返回 token 对象（含 preload 的 Album/Media）
```

**`Album` query 的 token 解析** (`resolvers/album.go:129`):
```
1. 先通过 r.ShareToken() 完整校验
2. 若 shareToken.AlbumID == id → 直接返回 Album
3. 否则: shareToken.Album.GetChildren(db, filter id=xxx)
   → 递归 CTE 检查 id 是否在相册树中
```

**`Media` query 的 token 解析** (`resolvers/media.go:171`):
```
1. 完整校验 ShareToken
2. 精确匹配 shareToken.MediaID == id
   → 不支持相册共享 token 访问任意子媒体！需走普通用户路径
```

**路径 B：HTTP 文件路由层（二进制读取）**

**挂载点**:
- `GET /api/photo/{name}` → `routes/photos.go:15` `RegisterPhotoRoutes`
- `GET /api/video/{name}` → `routes/videos.go:131` `RegisterVideoRoutes`
- `GET /api/download/album/{album_id}/{media_purpose}` → `routes/downloads.go:18` `RegisterDownloadRoutes`

**流程** (`routes/authenticate_routes.go:19` `authenticateMedia`):

```
1. auth.UserFromContext(r.Context())
   ├─ 有 User → user.OwnsAlbum 校验（同已登录路径）
   └─ 无 User → shareTokenFromRequest(db, r, &mediaID, &albumID)
        1. 从 r.URL.Query().Get("token") 取 token
        2. db 查询 ShareToken
        3. 过期检查: time.Now().UTC() 精确到秒对比 Expire.UTC()
        4. 密码检查: 从 r.Cookie("share-token-pw-<value>") 读取 + bcrypt 校验
        5. 类型匹配:
           - AlbumID: 相等 或 递归 CTE 检查子相册
           - MediaID: 必须精确相等
```

### 11.4 撤销（删除/失效）的完整路径

**显式撤销**:

| 操作 | GraphQL Mutation | Action 函数 | 权限 |
|------|-----------------|-------------|------|
| 删除 Token | `DeleteShareToken(token: String!)` | `actions.DeleteShareToken()` | Owner 本人或 Admin |
| 设置密码 | `ProtectShareToken(token: String!, password: String)` | `actions.ProtectShareToken()` | Owner 本人或 Admin |
| 清除密码 | `ProtectShareToken(token, password: null)` | 同上（password nil → 保存 nil） | Owner 本人或 Admin |
| 修改过期时间 | `SetExpireShareToken(token: String!, expire: Time)` | `actions.SetExpireShareToken()` | Owner 本人或 Admin |
| 立即过期 | `SetExpireShareToken(token, expire: 过去时间)` | 同上 | Owner 本人或 Admin |

**权限检查核心** (`share_token_actions.go:167` `getUserToken`):
```sql
WHERE share_tokens.value = ?
JOIN Owner ON ...
WHERE (Owner.id = ? OR Owner.admin = TRUE)
```

**隐式失效机制**:
1. **时间过期** — 访问时被拦截（GraphQL 和 HTTP 两条路径都检查）
2. **级联删除** — 删除 Album 时 `ON DELETE CASCADE` 自动删除关联 ShareToken
   （见 `models/share_token.go` `Album gorm:"constraint:OnDelete:CASCADE;"`）
3. **删除 Media** — 同上，级联删除关联的媒体级 ShareToken
4. **删除 Owner 用户** — `Owner gorm:"constraint:OnDelete:CASCADE;"` 删除用户同时删除其所有 ShareToken

### 11.5 过期判定的细微差异（GraphQL vs HTTP）

这是代码中一个值得注意的实现细节：

| 路径 | 过期判定实现 | 时间粒度 |
|------|-------------|---------|
| GraphQL `ShareToken` query | `time.Date(now.Year(), now.Month(), now.Day(), ..., 0, time.UTC)` 与 `Expire` 对比 | 按**天**粒度。当天 00:00:00 之后就算过期 |
| GraphQL `ShareTokenValidatePassword` | 同上（fakeTime 构造） | 按**天**粒度 |
| HTTP `shareTokenFromRequest` | `time.Now().UTC().After(shareToken.Expire.UTC())` | **精确到秒** |

**后果**: 一个 Expire 为 "2026-06-19 23:00:00 UTC" 的 token：
- 在 2026-06-19 08:00 用 GraphQL 查询 → fakeTime=2026-06-19 00:00:00，23:00 > 00:00 → **有效** ✓
- 在 2026-06-20 01:00 用 GraphQL 查询 → fakeTime=2026-06-20 00:00:00，23:00 < 00:00 → **已过期** ✗
- 在 2026-06-19 23:30 用 HTTP 路由访问 → `Now() > Expire` → **已过期** ✗

---

## 12. 面部识别数据在多用户共享相册场景下的隔离边界

### 12.1 数据模型结构（无用户维度）

**文件**: `api/graphql/models/face_detection.go`

```
FaceGroup {
    ID         int
    Label      *string        ← 用户可命名，例如 "张三"
    ImageFaces []ImageFace
}

ImageFace {
    ID           int
    FaceGroupID  int            ← 外键 → FaceGroup
    MediaID      int            ← 外键 → Media
    Descriptor   [128]float32  ← 128维人脸向量
    Rectangle    FaceRectangle  ← 人脸在图中的位置
}
```

**关键观察**: `FaceGroup` 和 `ImageFace` 都**没有 user_id 字段**。这意味着：
- 所有用户共享同一套 FaceGroup / ImageFace 表
- 同一张照片（被共享相册的所有 Owner 看到）的人脸数据只会被检测/存储一次
- 用户隔离完全是在**查询过滤层**实现的，而非数据存储层

### 12.2 写入时的检测流程（无用户隔离）

**挂载点**: `api/scanner/scanner_tasks/face_detection_task.go:15` `FaceDetectionTask.AfterProcessMedia`

```
扫描媒体 → AfterProcessMedia 钩子
    └─ 媒体是 Photo 且 GlobalFaceDetector != nil
        └─ face_detector_impl.go:92 DetectFaces(db, media)
             ├─ 找到 PhotoThumbnail 类型的 MediaURL
             ├─ fd.rec.RecognizeFile(缩略图路径) → 检测所有脸
             └─ 对每张脸: classifyFace
                  ├─ fd.classifyDescriptor(descriptor)
                  │    → rec.ClassifyThreshold(descriptor, 0.2)
                  │    → 从内存中的全局样本库匹配
                  ├─ 无匹配 → 新建 FaceGroup + ImageFace
                  └─ 有匹配 → 向已有 FaceGroup 追加 ImageFace
```

**内存中的全局分类器** (`face_detector_impl.go:17-23`):

```go
type faceDetector struct {
    mutex           sync.Mutex
    rec             *face.Recognizer
    faceDescriptors []face.Descriptor   ← 全库所有人脸向量
    faceGroupIDs    []int32             ← 一一对应的分组 ID
    imageFaceIDs    []int               ← 一一对应的 ImageFace ID
}
```

**初始化** (`face_detector_impl.go:25` `InitializeFaceDetector`):
```
SELECT * FROM image_faces（全表！）
→ 所有 ImageFace 的 Descriptor + FaceGroupID 全部加载到内存
→ 不分用户
```

**关键结论**:
- 人脸检测运行在**媒体被处理时**，由扫描任务触发，与具体哪个用户触发扫描无关
- 两个用户共享相册 X，相册 X 中的照片只会被检测一次人脸
- `classifyDescriptor` 使用的是**全局**已训练样本池，会跨用户地匹配人脸

### 12.3 读取时的用户过滤（查询层隔离）

所有 FaceGroup / ImageFace 的读取操作在 Resolver 层都加入了 `userAlbumIDs` 过滤。

#### 12.3.1 列出自己的人脸分组

`faces.go:377` `MyFaceGroups`:
```sql
SELECT face_groups.*
FROM face_groups
JOIN image_faces ON image_faces.face_group_id = face_groups.id
WHERE image_faces.media_id IN (
    SELECT media.id FROM media WHERE media.album_id IN (<用户相册IDs>)
)
GROUP BY face_groups.id
ORDER BY (label IS NULL) ASC, COUNT(image_faces.id) DESC
```

→ 只返回"该用户拥有的相册中出现过的脸"所归属的 FaceGroup。

#### 12.3.2 查询 FaceGroup 的详情

`faces.go:417` `FaceGroup(id)`:
```sql
SELECT face_groups.*
FROM face_groups
LEFT JOIN image_faces ON ...
LEFT JOIN media ON image_faces.media_id = media.id
WHERE face_groups.id = <id>
  AND media.album_id IN (<用户相册IDs>)   ← 关键：隔离边界
```

→ 用户查询的 FaceGroup 必须包含至少一张他拥有的照片中的脸。

#### 12.3.3 FaceGroup 下的 ImageFace 列表

`faces.go:20` `ImageFaces`:
```sql
SELECT image_faces.*
FROM image_faces
JOIN Media ON media.id = image_faces.media_id
WHERE face_group_id = ?
  AND media.album_id IN (<用户相册IDs>)
```

#### 12.3.4 FaceGroup 的图片计数

`faces.go:56` `ImageFaceCount` — 同 12.3.3，加 COUNT。

### 12.4 写入（修改）时的权限检查

所有 mutation 通过两个辅助函数验证用户对"这张脸/这个分组"的操作权：

#### `userOwnedFaceGroup(db, user, faceGroupID)`

**文件**: `api/graphql/resolvers/faces.util.go:18`

```go
if user.Admin → 直接通过，不校验
否则:
    userAlbumIDs = user 拥有的相册ID
    imageFaceQuery = 属于这些相册的 ImageFace ID
    查找 FaceGroup:
        JOIN image_faces ON face_groups.id = image_faces.face_group_id
        WHERE face_groups.id = faceGroupID
          AND image_faces.id IN (imageFaceQuery)
```

**要求**: FaceGroup 中至少有一张 ImageFace 关联的媒体属于该用户。

#### `getUserOwnedImageFaces(tx, user, imageFaceIDs)`

**文件**: `faces.util.go:61`

```go
Admin: 不过滤
非 Admin:
    JOIN media ON media.id = image_faces.media_id
    WHERE media.album_id IN (userAlbumIDs)
```

→ 返回传入的 `imageFaceIDs` 列表中真正属于用户相册的子集。**用户传入他人的 ImageFace ID 会被静默忽略。**

#### 12.4.1 各 mutation 的权限汇总

| Mutation | 权限检查 | 说明 |
|----------|---------|------|
| `SetFaceGroupLabel` | `userOwnedFaceGroup` | FaceGroup 中至少有一张自己的照片 |
| `CombineFaceGroups` | 源+目标都通过 `userOwnedFaceGroup` | 防止合并完全属于他人的分组 |
| `MoveImageFaces` | 目标 `userOwnedFaceGroup` + 源 `getUserOwnedImageFaces` | 只会移动用户"拥有"的那些脸 |
| `DetachImageFaces` | `getUserOwnedImageFaces` | 只分离用户自己照片中的脸 |
| `RecognizeUnlabeledFaces` | 见 12.5 | |

### 12.5 `RecognizeUnlabeledFaces`：内存中的用户维度过滤

`faces.go:300` `RecognizeUnlabeledFaces` → `face_detector_impl.go:218`

这是最复杂的隔离场景，流程如下：

```
1. 数据库层过滤"属于当前用户且未命名的 FaceGroup":
   SELECT face_groups.*
   JOIN image_faces ON ...
   JOIN media ON image_faces.media_id = media.id
   WHERE face_groups.label IS NULL
     AND media.album_id IN (user 的相册IDs)

2. 遍历内存中全局的 faceDescriptors/faceGroupIDs/imageFaceIDs
   把 "属于未命名 FaceGroup" 的条目单独摘出来放入
   unrecognizedDescriptors / unrecognizedFaceGroupIDs / unrecognizedImageFaceIDs
   其他保留 → new* 列表

3. 用 new* 列表重新 SetSamples → 分类器此时"忘了"用户的未标记脸

4. 对 unrecognized* 中每条重新 classifyDescriptor:
   - 还是无匹配 → 重新放回全局列表
   - 有匹配 → 更新 DB 中 image_faces.face_group_id → 迁移到新 FaceGroup
```

**隔离效果**: 重新分类时参考的样本池排除了"当前用户自己的未命名分组"，但仍包含"其他用户的未命名分组"。这意味着一个用户的未命名脸可能被自动归类到另一个用户已经命名的人脸分组中。

### 12.6 共享相册场景下的边界总结

```
共享相册 X 的照片 P
  └─ 检测出 3 张人脸 → ImageFace A, B, C
        ├─ 自动分到 FaceGroup FG-1 (未知人1)
        ├─ 自动分到 FaceGroup FG-2 (未知人2)
        └─ 自动分到 FaceGroup FG-3 (未知人3)

用户A 是相册 X 的 Owner:
  - MyFaceGroups → 看到 FG-1, FG-2, FG-3
  - 给 FG-1 命名 "妈妈" → 修改 FaceGroup.Label = "妈妈"
  - CombineFaceGroups(FG-3, FG-2) → 合并成一个分组

用户B 也是相册 X 的 Owner（共享）:
  - MyFaceGroups → 同样看到 FG-1, FG-2, FG-3
  - 看到 FG-1 的 Label 也是 "妈妈"  ← 标签共享！
  - 也能对这些 FaceGroup 执行重命名/合并
  - 甚至能 MoveImageFaces 把这些脸移动到自己"私有人脸分组"
    （注：只要 FaceGroup 有一张属于用户的图，用户就能对整个分组操作）

用户C 不拥有相册 X:
  - MyFaceGroups → 看不到任何 FG-*
  - 通过 FaceGroup(id) 直接查询 → 返回空对象或错误
```

**共享边界**:
| 维度 | 是否共享 | 说明 |
|------|---------|------|
| ImageFace 检测结果 | ✅ 共享 | 只检测一次，存储一份 |
| FaceGroup 分组归属 | ✅ 共享 | 同一张脸的分组归属对所有用户一致 |
| FaceGroup 的 Label | ✅ 共享 | 用户A命名的标签，用户B也能看到 |
| 人脸识别的全局样本池 | ✅ 共享 | 所有用户人脸向量在一起训练/匹配 |
| MyFaceGroups 列表 | ❌ 用户维度过滤 | 只看自己相册中出现过的分组 |
| Mutation 操作权限 | ⚠️ 按相册所有权 | 只要分组中有"自己的图"就能操作整个分组 |
| 跨用户自动归类 | ✅ 可能 | 用户A的未标记脸可能被归入用户B已标记的分组 |

**隐含风险**: 用户A在自己私人相册中标记"前女友"，如果共享相册里出现同一个人，RecognizeUnlabeledFaces 运行后，用户B也能看到该分组被命名为"前女友"——泄露了用户A的私人标签。

---

## 13. 匿名访问（ShareToken 访问）的缓存/节流/限速机制

### 13.1 Throttle 工具的真实用途

**文件**: `api/utils/throttle.go`

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

这是一个"**最小触发间隔**"节流器，不是 IP 级/用户级的请求限速器。它保证 `action` 被调用的频率不超过 `1/interval`。

**实际使用场景**（全部是通知推送频率控制，与匿名访问无关）:
| 位置 | Interval | 用途 |
|------|----------|------|
| `scanner_queue.go:102` | 500ms | 扫描进度 WebSocket 通知的推送频率 |
| `notification_task.go:21` | 500ms | "发现新媒体"通知的推送频率 |

### 13.2 针对匿名访问不存在的限速机制

对使用 `?token=` 参数的匿名文件访问路径，代码中**没有任何**以下机制：

| 机制 | 是否存在 | 说明 |
|------|---------|------|
| 按 IP 请求限速 | ❌ 无 | `photos.go`、`videos.go` 路由 handler 中未见任何 limiter |
| 按 Token 请求限速 | ❌ 无 | token 仅用于权限校验，不计入计数器 |
| 带宽限制 / 并发连接数 | ❌ 无 | 直接调用 `http.ServeFile` |
| 并发请求节流 | ❌ 无 | ProcessSingleMedia 内部虽有串行处理，但多个共享访问各走各的 |
| 防暴力破解共享密码 | ❌ 无 | `ShareToken` query 和 `shareTokenFromRequest` 都没有 bcrypt 以外的错次限制 |

### 13.3 HTTP 层的缓存头策略（唯一的"保护"）

**`GET /api/photo/{name}?token=...`** (`photos.go:74`):
```
Cache-Control: private, max-age=31536000, immutable
```
- `private`: 不允许 CDN/代理缓存（但浏览器可以缓存）
- `max-age=31536000`: 浏览器可缓存 1 年
- `immutable`: 不做条件请求（revalidation）

**`GET /api/video/{name}?token=...`** (`videos.go:122`):
```
Cache-Control: private, max-age=31536000, immutable
```
同上。

**`GET /api/download/album/...`** (`downloads.go:61`):
```
Cache-Control: no-store, no-cache, must-revalidate, max-age=0
Pragma: no-cache
```
ZIP 下载完全禁止缓存。

**效果**:
- 第二次获取同一张图/视频时由浏览器本地缓存提供服务 → 不会打到服务器 → 客观上降低了重复请求量
- 但这是"让客户端自己缓存"而非"服务器端限速"
- 一旦用户/攻击者禁用缓存或更换 URL（`?token=` 加随机参数），就完全失效

### 13.4 GraphQL 层的查询缓存

**文件**: `api/graphql/endpoint/graphql_endpoint.go:41-45`

```go
graphqlServer.SetQueryCache(lru.New[*ast.QueryDocument](1000))

graphqlServer.Use(extension.AutomaticPersistedQuery{
    Cache: lru.New[string](100),
})
```

- Query Document LRU: 最近 1000 个 GraphQL 查询的 AST 缓存（解析/验证阶段 CPU 开销）
- APQ: 最近 100 个 persisted query hash 映射

这两个缓存**对匿名用户和登录用户一视同仁**，能降低 GraphQL endpoint 的 CPU 消耗，但对请求量本身不加限制。

### 13.5 共享 token 图片处理的隐性节流

当图片缓存尚未生成时，`photos.go:54` 会触发 `ProcessSingleMediaFunc`。视频路由 (`videos.go:92`) 同理。

```go
scanner.ProcessSingleMedia(ctx, db, media)
    └─ 内部走 face_detection.DetectFaces 等昂贵操作
```

这些操作的队列化：
- **Media 处理没有队列**，每次请求直接同步执行
- **Album 扫描有队列** (`ScannerQueue`)，但共享 token 触发的是单媒体处理不进该队列

**风险场景**: 如果一个相册共享 token 被公开，图片缓存被清除后，大量并发请求每张图都会触发 `ProcessSingleMedia`（含人脸检测 + 重新编码），相当于对服务器发起了 CPU 密集型 DoS。

### 13.6 代码中所有路径一览

```
匿名访问入口:
├─ /api/graphql → 带 ShareTokenCredentials
│     └─ ShareToken / Album / Media 查询
│     └─ 无限速，只有 Query AST 缓存 (LRU 1000)
│
├─ /api/photo/{name}?token=xxx
│     └─ RegisterPhotoRoutes → authenticateMedia → shareTokenFromRequest
│     └─ 缓存命中: http.ServeFile
│     └─ 缓存未命中: ProcessSingleMedia (CPU 密集，无限流)
│     └─ Cache-Control: private, 1yr, immutable
│
├─ /api/video/{name}?token=xxx
│     └─ 同上，但视频编码更昂贵
│
└─ /api/download/album/{album_id}/{purpose}?token=xxx
      └─ authenticateAlbum → shareTokenFromRequest
      └─ 实时 ZIP 打包流式输出
      └─ 无缓存，无限速
```

### 13.7 匿名访问安全性总结

| 维度 | 现状 | 潜在风险 |
|------|------|---------|
| 密码暴力破解 | bcrypt cost=12 + 无次数限制 | 离线/在线均可尝试；短密码易被 GPU 破解 |
| 大批量下载/爬取 | 无速率限制，仅依赖浏览器缓存 | 公开 token 的相册可被一键爬完 |
| 资源耗尽 DoS | ProcessSingleMedia 无队列/限流 | 缓存失效时大量请求导致 CPU 爆满 |
| Token 枚举 | 8 字符 token (48 bits) + 无频率限制 | 暴力枚举可能，但配合过期时间概率偏低 |
| CORS 限制 | 生产环境限定 UI Endpoint 域名；TokenPassword 在 Cookie | 第三方页面无法跨域读取图片元数据，但 `<img src>` 级联加载不受限 |
| Referer 检查 | ❌ 无 | 外链盗图（token 暴露在 URL 中）无法阻止 |
