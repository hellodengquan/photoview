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

### 6.5 人脸数据的隔离

人脸检测 (`ImageFace`) 附着在 `Media` 上，无用户维度隔离。同一媒体的所有 Owner 看到相同的人脸数据。

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

共享密码通过 Cookie `share-token-pw-<value>` 传递，而非 HTTP Header。这意味着密码验证依赖浏览器的 Cookie 机制。
