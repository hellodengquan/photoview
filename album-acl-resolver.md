# 相册与媒体可见性：GraphQL 三线 ACL 过滤拆解

## 总览

Photoview 的 GraphQL 层对相册（Album）和媒体（Media）的可见性由三条独立的权限线控制：

| 线 | 核心机制 | 入口位置 |
|---|---------|---------|
| **用户登录态** | HTTP Cookie / WebSocket Bearer → `auth.UserFromContext(ctx)` | `api/graphql/auth/auth.go` |
| **相册 ACL（user_albums）** | `user_albums` 多对多关联表 + `OwnsAlbum` 递归父级校验 | `api/graphql/models/user.go` → `OwnsAlbum` |
| **分享 Token** | `ShareTokenCredentials` 参数 → `shareToken` resolver 校验过期/密码 | `api/graphql/resolvers/share_token.go` |

三条线在 GraphQL 字段 resolver 里以 **"分享 token 优先 → 登录态兜底"** 的 if-else 模式合并，而非多层中间件堆叠。

---

## 第一层：用户登录态（Authentication）

### 1.1 登录态注入

**文件**: `api/graphql/auth/auth.go:31-70`

`Middleware` 函数作为 HTTP 中间件，从请求 Cookie `auth-token` 或 WebSocket `Authorization: Bearer <24位token>` 中提取 access token，通过 dataloader 查数据库拿到 `*models.User`，写入 `context`：

```go
// auth.go:31-70
func Middleware(db *gorm.DB) func(http.Handler) http.Handler {
    // 从 Cookie "auth-token" 取 token
    user, err := loaders.UserFromAccessToken.Load(tokenCookie.Value)
    // 写入 context
    ctx := AddUserToContext(r.Context(), user)
    r = r.WithContext(ctx)
}
```

WebSocket 走 `AuthWebsocketInit`（`auth.go:92-131`），逻辑一致，token 来自 `InitPayload["Authorization"]`。

### 1.2 从 Context 取用户

```go
// auth.go:87-90
func UserFromContext(ctx context.Context) *models.User {
    raw, _ := ctx.Value(userCtxKey).(*models.User)
    return raw
}
```

**关键点**：如果未登录，`UserFromContext` 返回 `nil`，不报错。后续 resolver 根据业务决定是否拒绝。

### 1.3 Directive 守卫

**文件**: `api/graphql/directive.go`

两个 GraphQL directive 在字段解析前拦截：

| Directive | 逻辑 | Schema 声明 |
|-----------|------|-------------|
| `@isAuthorized` | `user == nil` → `ErrUnauthorized` | `root.graphql:1` |
| `@isAdmin` | `user == nil || !user.Admin` → error | `root.graphql:2` |

```go
// directive.go:20-27
func IsAuthorized(ctx context.Context, obj interface{}, next graphql.Resolver) (res interface{}, err error) {
    user := auth.UserFromContext(ctx)
    if user == nil {
        return nil, auth.ErrUnauthorized
    }
    return next(ctx)
}
```

**受 @isAuthorized 保护的 Query 字段**：
- `myAlbums` — `album.graphql:28`
- `myMedia` — `media.graphql:108`
- `myTimeline` — `timeline.graphql:10`
- `myUser` / `myUserPreferences` — `user.graphql:55-58`
- `myMediaGeoJson` — `media_geo_json.graphql`

**受 @isAdmin 保护的字段**：
- `User.albums` / `User.rootAlbums` — `user.graphql:5-7`
- `user` / `updateUser` / `createUser` / `deleteUser` / `userAddRootPath` / `userRemoveRootAlbum` — `user.graphql`

---

## 第二层：相册 ACL（Authorization via user_albums）

### 2.1 数据模型

**文件**: `api/graphql/models/user.go:14-21`, `album.go:10-21`

```
User ──many2many:user_albums── Album
```

`user_albums` 表是 User 和 Album 的多对多关联表（`UserAlbums` struct，`user.go:30-33`）。一个相册可以有多个 Owner。

### 2.2 FillAlbums — 加载用户拥有的相册 ID 列表

**文件**: `api/graphql/models/user.go:153-165`

```go
func (user *User) FillAlbums(db *gorm.DB) error {
    if len(user.Albums) > 0 { return nil }
    return db.Model(&user).Association("Albums").Find(&user.Albums)
}
```

用 GORM Association 查 `user_albums` 表，把该用户直接关联的 Album 加载到 `user.Albums`。**注意**：这只包含直接关联的相册，不含子相册。

### 2.3 OwnsAlbum — 递归父级校验（核心 ACL）

**文件**: `api/graphql/models/user.go:167-180`

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

逻辑：从目标 album 向上递归查找所有父级 album（`GetParents`，`album.go:59-81`），用 `filter` 过滤出在 `user_albums` 中存在的父级。只要**任一祖先**（包括自身）在 `user_albums` 中与该用户关联，就返回 `true`。

递归 SQL（`album.go:63-81`）：
```sql
WITH recursive super_albums AS (
    SELECT * FROM albums AS leaf WHERE id = ?
    UNION ALL
    SELECT parent.* FROM albums AS parent
    JOIN super_albums ON parent.id = super_albums.parent_album_id
)
SELECT * FROM super_albums WHERE EXISTS (
    SELECT 1 FROM user_albums
    WHERE user_albums.user_id = ? AND user_albums.album_id = id LIMIT 1
)
```

**这意味着**：用户拥有父相册即自动拥有子相册的访问权。

### 2.4 各 resolver 中 ACL 过滤方式

| Resolver | ACL 实现方式 | 代码位置 |
|----------|-------------|---------|
| `MyAlbums` | `FillAlbums` → `WHERE id IN (userAlbumIDs)` | `album_actions.go:9-48` |
| `Album(id)` | `actions.Album(db, user, id)` → `OwnsAlbum` | `album.go:161`, `album_actions.go:83-102` |
| `Media(id)` | `EXISTS (SELECT * FROM user_albums WHERE album_id = media.album_id AND user_id = ?)` | `media.go:191-198` |
| `MediaList(ids)` | `LEFT JOIN user_albums ON album_id = media.album_id WHERE user_id = ?` | `media.go:220-224` |
| `MyMedia` | `FillAlbums` → `WHERE album_id IN (SELECT album_id FROM user_albums WHERE user_id = ?)` | `media_actions.go:8-22` |
| `Search` | `EXISTS (SELECT * FROM user_albums WHERE user_id = ? AND album_id = Album.id)` | `search_actions.go:13-76` |
| `MyTimeline` | `albums.id IN (SELECT album_id FROM user_albums WHERE user_id = ?)` | `timeline_actions.go:16-18` |
| `MyMediaGeoJSON` | `INNER JOIN user_albums ON media.album_id = user_albums.album_id WHERE user_id = ?` | `media_geo_json.go:33` |
| `AlbumPath` | 递归查路径，逐级 `OwnsAlbum` 过滤 | `album_actions.go:104-137` |
| `FaceGroup` / `MyFaceGroups` | `FillAlbums` → `WHERE album_id IN (userAlbumIDs)` | `faces.go:377-414` |

---

## 第三层：分享 Token（Share Token）

### 3.1 数据模型

**文件**: `api/graphql/models/share_token.go`

```go
type ShareToken struct {
    Model
    Value    string     // 分享码
    OwnerID  int        // 创建者 user id
    Owner    User
    Expire   *time.Time // 过期时间
    Password *string    // 可选密码 (bcrypt hash)
    AlbumID  *int       // 关联的相册（相册分享）
    Album    *Album
    MediaID  *int       // 关联的媒体（媒体分享）
    Media    *Media
}
```

### 3.2 shareToken Resolver — Token 校验入口

**文件**: `api/graphql/resolvers/share_token.go:74-111`

```go
func (r *queryResolver) ShareToken(ctx context.Context, credentials models.ShareTokenCredentials) (*models.ShareToken, error) {
    // 1. 按 token value 查数据库
    db.Preload(clause.Associations).Where("value = ?", credentials.Token).First(&token)

    // 2. 检查过期
    if token.Expire != nil && fakeTime.After(*token.Expire) {
        return nil, errors.New("share expired")
    }

    // 3. 校验密码
    if token.Password != nil {
        bcrypt.CompareHashAndPassword([]byte(*token.Password), []byte(*credentials.Password))
    }

    return &token, nil
}
```

**校验链**：token 存在性 → 过期检查 → 密码校验。任一环节失败直接报错。

`ShareTokenCredentials` 结构（`models/generated.go:90-93`）：
```go
type ShareTokenCredentials struct {
    Token    string  `json:"token"`
    Password *string `json:"password,omitempty"`
}
```

### 3.3 分享 Token 在 HTTP 路由层的校验

**文件**: `api/routes/authenticate_routes.go`

这是媒体文件（图片/视频）HTTP 访问时的鉴权，不走 GraphQL：

```go
// authenticate_routes.go:19-46
func authenticateMedia(media, db, r) {
    user := auth.UserFromContext(r.Context())
    if user != nil {
        // 登录态 → OwnsAlbum 校验
        ownsAlbum, _ := user.OwnsAlbum(db, &album)
        if !ownsAlbum { return false }
    } else {
        // 未登录 → 走 shareTokenFromRequest
        shareTokenFromRequest(db, r, &media.ID, &media.AlbumID)
    }
}
```

`shareTokenFromRequest`（`authenticate_routes.go:71-155`）校验流程：
1. 从 URL query param `?token=xxx` 取 token
2. 查数据库验证存在性
3. 检查过期时间
4. 校验密码（从 Cookie `share-token-pw-<value>` 取）
5. 验证 album/media ID 匹配：
   - 相册分享：如果请求的 albumID ≠ token.AlbumID，递归查子相册是否匹配
   - 媒体分享：mediaID 必须完全匹配

---

## 三线合并：GraphQL 字段层如何组织

### 4.1 合并模式：分享 Token 优先，登录态兜底

核心模式出现在 `album(id, tokenCredentials)` 和 `media(id, tokenCredentials)` 两个 Query resolver 中。

#### Album Query 合并逻辑

**文件**: `api/graphql/resolvers/album.go:128-162`

```go
func (r *queryResolver) Album(ctx context.Context, id int, tokenCredentials *models.ShareTokenCredentials) (*models.Album, error) {
    db := r.DB(ctx)

    // === 第一优先级：分享 Token ===
    if tokenCredentials != nil {
        shareToken, err := r.ShareToken(ctx, *tokenCredentials)  // 内部校验过期+密码
        if err != nil {
            return nil, err  // token 无效直接拒绝，不回退到登录态
        }

        if shareToken.Album != nil {
            // token 关联的是相册分享
            if *shareToken.AlbumID == id {
                return shareToken.Album, nil  // 精确匹配，直接返回
            }
            // 不精确匹配 → 查子相册
            subAlbum, err := shareToken.Album.GetChildren(db, func(query *gorm.DB) *gorm.DB {
                return query.Where("sub_albums.id = ?", id)
            })
            if len(subAlbum) > 0 {
                return subAlbum[0], nil  // 子相册匹配，返回
            }
        }
        // token 校验通过但 album 不匹配 → 继续走登录态
    }

    // === 第二优先级：登录态 + 相册 ACL ===
    user := auth.UserFromContext(ctx)
    if user == nil {
        return nil, auth.ErrUnauthorized  // 未登录又没 token → 拒绝
    }

    return actions.Album(db, user, id)  // 内部调用 OwnsAlbum 递归校验
}
```

#### Media Query 合并逻辑

**文件**: `api/graphql/resolvers/media.go:170-205`

```go
func (r *queryResolver) Media(ctx context.Context, id int, tokenCredentials *models.ShareTokenCredentials) (*models.Media, error) {
    db := r.DB(ctx)

    // === 第一优先级：分享 Token ===
    if tokenCredentials != nil {
        shareToken, err := r.ShareToken(ctx, *tokenCredentials)
        if err != nil {
            return nil, err
        }
        if *shareToken.MediaID == id {
            return shareToken.Media, nil  // 媒体分享只做精确匹配，不查子级
        }
        // token 不匹配此 media → 继续走登录态
    }

    // === 第二优先级：登录态 + 相册 ACL ===
    user := auth.UserFromContext(ctx)
    if user == nil {
        return nil, auth.ErrUnauthorized
    }

    // SQL 层 JOIN Album + user_albums 校验
    db.Joins("Album").
        Where("media.id = ?", id).
        Where("EXISTS (SELECT * FROM user_albums WHERE album_id = media.album_id AND user_id = ?)", user.ID).
        Where("media.id IN (SELECT media_id FROM media_urls WHERE media_urls.media_id = media.id)").
        First(&media)
}
```

### 4.2 合并流程图

```
GraphQL 请求进入
│
├─ 字段带 @isAuthorized / @isAdmin directive？
│   ├─ 是 → directive 先执行，未登录/非 admin 直接拒绝
│   └─ 否 → 进入 resolver
│
├─ resolver 内部分支：
│   │
│   ├─ 有 tokenCredentials 参数？
│   │   ├─ 是 → 调用 ShareToken() 校验（过期 + 密码）
│   │   │   ├─ 校验失败 → 报错，不回退
│   │   │   └─ 校验成功 → 检查 album/media 是否匹配
│   │   │       ├─ 匹配 → 直接返回（跳过登录态校验）
│   │   │       └─ 不匹配 → 继续下一优先级
│   │   └─ 否 → 直接跳到登录态
│   │
│   ├─ 登录态校验：UserFromContext(ctx)
│   │   ├─ user == nil → ErrUnauthorized
│   │   └─ user != nil → 相册 ACL 校验
│   │
│   └─ 相册 ACL 校验：OwnsAlbum / user_albums SQL
│       ├─ 通过 → 返回数据
│       └─ 不通过 → "forbidden"
```

### 4.3 不同 Query 字段的三线覆盖情况

| Query 字段 | Directive | 分享 Token 入口 | 登录态校验 | ACL 校验方式 |
|-----------|-----------|-----------------|-----------|-------------|
| `album(id, tokenCredentials)` | 无 | `tokenCredentials` 参数 | `UserFromContext` | `actions.Album` → `OwnsAlbum` |
| `media(id, tokenCredentials)` | 无 | `tokenCredentials` 参数 | `UserFromContext` | `EXISTS user_albums` SQL |
| `myAlbums` | `@isAuthorized` | 无 | directive 拦截 | `FillAlbums` + `WHERE IN` |
| `myMedia` | `@isAuthorized` | 无 | directive 拦截 | `FillAlbums` + `WHERE IN` |
| `myTimeline` | `@isAuthorized` | 无 | directive 拦截 | `user_albums` 子查询 |
| `search` | 无 | 无 | resolver 内 `UserFromContext` | `EXISTS user_albums` |
| `shareToken(credentials)` | 无 | 直接校验 credentials | 不涉及 | 不涉及 |
| `mediaList(ids)` | 无 | 无 | resolver 内 `UserFromContext` | `LEFT JOIN user_albums` |

### 4.4 字段子 Resolver 的权限继承

当一个 Album 通过分享 Token 被获取后，其子字段的 resolver **不会重新校验 token**：

| 字段 Resolver | 权限检查 | 说明 |
|--------------|---------|------|
| `Album.media` | 无额外检查 | 直接按 album_id 查 media |
| `Album.subAlbums` | 无额外检查 | 直接按 parent_album_id 查 |
| `Album.path` | `UserFromContext` → 若未登录返回空列表 | 分享场景下返回空，但不报错 |
| `Album.shares` | 无额外检查 | 返回该 album 所有 share token |
| `Album.thumbnail` | 无额外检查 | 直接查数据库 |
| `Media.album` | 无额外检查 | 返回关联的 album 对象 |
| `Media.favorite` | 必须登录 | 未登录返回错误 |
| `Media.shares` | 无额外检查 | 返回该 media 所有 share token |
| `Media.downloads` | 无额外检查 | 返回所有下载链接 |

**注意**：`Album.media` 和 `Album.subAlbums` 在分享场景下不做范围限制——如果分享的是父相册，子相册和其中的 media 都可以通过子字段 resolver 获取，这与 `Album(id)` resolver 中 `GetChildren` 的逻辑一致。

---

## HTTP 路由层（非 GraphQL）的对称逻辑

**文件**: `api/routes/authenticate_routes.go`

媒体文件的 HTTP 访问（缩略图、原图、视频）走独立的鉴权：

```
HTTP 请求
├─ 登录态 (Cookie) → UserFromContext
│   ├─ user != nil → OwnsAlbum 校验
│   └─ user == nil → 走分享 token 校验
│       ├─ URL ?token=xxx
│       ├─ 数据库查 token → 过期检查 → 密码校验 (Cookie)
│       ├─ AlbumID 匹配（含递归子相册）
│       └─ MediaID 精确匹配
```

与 GraphQL 层的关键差异：
1. 分享 token 从 URL query param 获取，非 GraphQL 参数
2. 密码从 Cookie `share-token-pw-<value>` 获取
3. **登录态与 token 是互斥的**：有登录态只走 `OwnsAlbum`，没登录态才走 token

---

## 关键发现与潜在问题

1. **分享 Token 校验失败不回退**：如果 `tokenCredentials` 提供了但校验失败（过期/密码错误），直接报错，不会再尝试登录态。这是有意设计——避免用 token 枚举来探测未授权访问。

2. **Album.shares 无权限过滤**：`albumResolver.Shares`（`album.go:89-96`）直接查 `WHERE album_id = ?` 返回所有 share token，不校验当前用户是否是 album owner。任何能访问该 album 的人（包括通过分享 token）都能看到所有分享链接。

3. **子字段 resolver 不感知分享边界**：`Album.media` 和 `Album.subAlbums` 不检查当前请求是否来自分享 token，因此如果分享的是父相册，所有子相册和 media 都可以被遍历。

4. **OwnsAlbum 的递归方向是向上**：`OwnsAlbum` 从目标 album 向上找父级，只要某个祖先在 `user_albums` 中就通过。这意味着 `user_albums` 只需存储根相册关联，子相册自动继承。

5. **Media 直接走 SQL 而非 OwnsAlbum**：`Media(id)` resolver 用 `EXISTS (SELECT * FROM user_albums WHERE album_id = media.album_id AND user_id = ?)` 只检查 media 直接所属的 album，不做递归。这依赖一个前提：`user_albums` 中每个用户关联了完整的相册树，而非只关联根相册。从 `scanner_user.go` 的扫描逻辑看，新增相册时会递归地把所有子相册都写入 `user_albums`。

---

## 补充一：多层嵌套 subAlbums 递归查询时 ACL 的传递路径

### 场景描述

当相册树存在多层嵌套时（如 `Root > A > B > C`），不同入口获取子相册的 ACL 传递路径不同。以下按「登录态」和「分享 token」两条线分别拆解。

### 1. 登录态路径：user_albums 全量写入 + resolver 无二次裁剪

#### 1.1 写入时机 — 扫描器把每层子相册都写入 user_albums

**文件**: `api/scanner/scanner_user.go:45-226`

扫描器 `FindAlbumsForUser` 使用 BFS 遍历用户根目录下的所有子目录。对每个目录：

```go
// scanner_user.go:138-165
if len(albumResult) == 0 {
    // 新相册：继承父相册的所有 Owners
    album = &models.Album{Title: albumTitle, ParentAlbumID: albumParentID, Path: albumPath}
    tx.Create(&album)
    tx.Model(&album).Association("Owners").Append(parentOwners)  // 继承父相册的 Owners
} else {
    // 已有相册：检查当前用户是否已是 Owner，不是则添加
    var userAlbumOwner []models.User
    tx.Model(&album).Association("Owners").Find(&userAlbumOwner, "user_albums.user_id = ?", user.ID)
    if len(userAlbumOwner) == 0 {
        tx.Model(&album).Association("Owners").Append(&newUser)
    }
}
```

关键逻辑：

- **新相册**：从 `albumParent`（父相册对象）获取 `parentOwners`，把这些 Owners 写入新相册的 `user_albums` 关联。这意味着子相册的 Owners 列表与父相册完全一致。
- **已有相册**：检查当前扫描用户是否在该相册的 Owners 列表中，不在则追加。
- **子目录入队**：`scanQueue.PushBack(scanInfo{path: subalbumPath, parent: album, ...})`，parent 就是当前相册，使得子相册创建时能继承父相册的 Owners。

**结果**：扫描完成后，`user_albums` 表中包含该用户对相册树中**每一层**的直接关联记录。例如用户 U 拥有 Root，则 `user_albums` 中会有 `(U, Root), (U, A), (U, B), (U, C)` 四条记录。

#### 1.2 读取路径 — Album.subAlbums resolver 不做 ACL 过滤

**文件**: `api/graphql/resolvers/album.go:54-65`

```go
func (r *albumResolver) SubAlbums(ctx context.Context, obj *models.Album, ...) ([]*models.Album, error) {
    var albums []*models.Album
    query := r.DB(ctx).Where("parent_album_id = ?", obj.ID)
    query = models.FormatSQL(query, order, paginate)
    query.Find(&albums)
    return albums, nil
}
```

`SubAlbums` 只按 `parent_album_id` 查询，**不检查 user_albums、不检查 context 中的用户身份、不检查分享 token**。这意味着：

- 如果用户通过 `myAlbums` 或 `Album(id)` 获取到了某个 Album 对象，后续通过 `album.subAlbums` 展开时，GraphQL 引擎会为每个返回的 Album 再次调用 `subAlbums` resolver，逐层展开所有子相册。
- **ACL 只在入口处检查**（`myAlbums` 过滤了用户拥有的相册，`Album(id)` 用 `OwnsAlbum` 校验），子字段 resolver 信任父级已通过校验。

#### 1.3 递归查询时的完整调用链

以 GraphQL 查询为例：

```graphql
query {
  album(id: 1) {        # 入口：Album(id) → OwnsAlbum 校验
    subAlbums {          # 第二层：albumResolver.SubAlbums → 无 ACL
      subAlbums {        # 第三层：albumResolver.SubAlbums → 无 ACL
        media { ... }    # 第四层：albumResolver.Media → 无 ACL
      }
    }
  }
}
```

```
Album(id=1)
  │
  ├─ user = UserFromContext(ctx)       # auth.go:87
  ├─ actions.Album(db, user, 1)        # album_actions.go:83
  │   └─ user.OwnsAlbum(db, album)     # user.go:167
  │       └─ album.GetParents(db, filter)  # 递归 CTE 向上查父级
  │           └─ EXISTS (user_albums WHERE user_id=? AND album_id=id)
  │
  └─ 返回 Album#1 对象
      │
      └─ subAlbums resolver → WHERE parent_album_id = 1
          │
          ├─ Album#2 (A)          ← 无 ACL 检查，直接返回
          │   └─ subAlbums resolver → WHERE parent_album_id = 2
          │       ├─ Album#3 (B)  ← 无 ACL 检查
          │       │   └─ subAlbums → WHERE parent_album_id = 3
          │       │       └─ Album#4 (C)  ← 无 ACL 检查
          │       │           └─ media → WHERE album_id = 4  ← 无 ACL 检查
```

**结论**：在登录态路径下，多层 subAlbums 的 ACL 传递依赖 `user_albums` 的全量写入（扫描时每层都有关联记录）+ 入口处的单次校验。子字段 resolver 假设「如果父相册可访问，则所有子相册也可访问」，不做逐层裁剪。

### 2. 分享 Token 路径：入口递归匹配 + 子字段无边界

#### 2.1 入口处 — GetChildren 递归匹配子相册

**文件**: `api/graphql/resolvers/album.go:128-154`

```go
if tokenCredentials != nil {
    shareToken, err := r.ShareToken(ctx, *tokenCredentials)
    // ...
    if shareToken.Album != nil {
        if *shareToken.AlbumID == id {
            return shareToken.Album, nil   // 精确匹配
        }
        // 查子相册
        subAlbum, err := shareToken.Album.GetChildren(db, func(query *gorm.DB) *gorm.DB {
            return query.Where("sub_albums.id = ?", id)
        })
        if len(subAlbum) > 0 {
            return subAlbum[0], nil   // 子相册匹配
        }
    }
}
```

`GetChildren` 的递归 CTE（`album.go:39-57`）：

```sql
WITH recursive sub_albums AS (
    SELECT * FROM albums AS root WHERE id IN (?)   -- 起点是 token 关联的相册
    UNION ALL
    SELECT child.* FROM albums AS child
    JOIN sub_albums ON child.parent_album_id = sub_albums.id
)
SELECT * FROM sub_albums WHERE sub_albums.id = ?   -- 过滤出目标 id
```

这意味着分享入口只做**单次递归匹配**：如果请求的 album id 在 token 关联相册的子孙树中，就直接返回。不存在逐层校验的概念。

#### 2.2 子字段展开 — 无分享边界约束

一旦入口返回了 Album 对象，后续的 `subAlbums` 和 `media` 字段 resolver 行为与登录态完全相同——不做任何分享范围检查：

```go
// album.go:54-65
func (r *albumResolver) SubAlbums(...) ([]*models.Album, error) {
    query := r.DB(ctx).Where("parent_album_id = ?", obj.ID)
    // 无分享 token 检查，无 user_albums 检查
}
```

**影响**：假设分享 token 关联的是 Album#1（Root），查询 `album(id:1, tokenCredentials:{token:"xxx"})` 后展开 `subAlbums`，会返回 Album#1 的所有直接子相册。继续展开子相册的 `subAlbums`，可以遍历整个子孙树。这与入口处 `GetChildren` 的语义一致——分享的是父相册，子相册理应可访问。

**但存在边界问题**：如果子相册拥有不属于 token 关联用户的 Owners（多用户共享子相册），通过分享 token 同样可以遍历到这些子相册及其 media，因为 `subAlbums` resolver 不做 owner 过滤。

#### 2.3 HTTP 路由层的递归匹配

**文件**: `api/routes/authenticate_routes.go:125-147`

```go
if shareToken.AlbumID != nil && *albumID != *shareToken.AlbumID {
    // 检查子相册
    var count int
    db.Raw(`
        WITH recursive child_albums AS (
            SELECT * FROM albums WHERE parent_album_id = ?
            UNION ALL
            SELECT child.* FROM albums child
            JOIN child_albums parent ON parent.id = child.parent_album_id
        )
        SELECT COUNT(id) FROM child_albums WHERE id = ?
    `, *shareToken.AlbumID, albumID).Find(&count)
}
```

HTTP 路由层对媒体文件的分享验证同样使用递归 CTE，起点是 `shareToken.AlbumID` 的直接子相册（`parent_album_id = ?`），向下递归所有层级。与 GraphQL 层的 `GetChildren` 语义一致。

### 3. 两条路径的对比总结

| 维度 | 登录态路径 | 分享 Token 路径 |
|------|-----------|----------------|
| 入口 ACL | `OwnsAlbum` 向上递归查 `user_albums` | `GetChildren` 从 token 相册向下递归匹配 |
| ACL 检查范围 | 单次入口校验 | 单次入口校验 |
| subAlbums 逐层展开 | 无 ACL 过滤，信任入口 | 无 ACL 过滤，信任入口 |
| media 字段 | 无 ACL 过滤 | 无 ACL 过滤 |
| ACL 数据源 | `user_albums` 表（每层都有记录） | ShareToken.AlbumID + 递归 CTE |
| 跨用户子相册 | 不可见（入口 OwnsAlbum 拦截） | 可见（subAlbums 不检查 owner） |

---

## 补充二：分享 Token 过期与用户登录态过期同时发生时的优先级判断

### 场景描述

一个请求可能同时携带：
- 用户登录凭证：Cookie `auth-token`（对应 `AccessToken`，有过期时间）
- 分享 Token 凭证：GraphQL 参数 `tokenCredentials`（对应 `ShareToken`，可选过期时间）

两条线各有独立的过期检查机制，当两者同时过期时，resolver 的优先级判断取决于代码执行顺序。

### 1. 用户登录态的过期检查

#### 1.1 过期时机 — dataloader SQL 查询中隐式过滤

**文件**: `api/dataloader/userLoader.go:10-70`

```go
func NewUserLoaderByToken(db *gorm.DB) *UserLoader {
    return &UserLoader{
        fetch: func(tokens []string) ([]*models.User, []error) {
            // 关键：WHERE expire > ?
            var accessTokens []*models.AccessToken
            db.Where("expire > ?", time.Now()).Where("value IN (?)", tokens).Find(&accessTokens)

            rows, _ := db.Table("access_tokens").
                Select("distinct user_id").
                Where("expire > ?", time.Now()).       // 同样的过期过滤
                Where("value IN (?)", tokens).Rows()
            // ...
        },
    }
}
```

用户登录态的过期检查**不在 HTTP 中间件里**，而在 dataloader 的 SQL 查询里。如果 `AccessToken.Expire < time.Now()`，查询结果为空，dataloader 返回 `nil` user。

#### 1.2 中间件处理 nil user 的方式

**文件**: `api/graphql/auth/auth.go:31-70`

```go
func Middleware(db *gorm.DB) func(http.Handler) http.Handler {
    // 从 Cookie 取 token
    user, err := loaders.UserFromAccessToken.Load(tokenCookie.Value)
    if err != nil {
        http.Error(w, INVALID_AUTH_TOKEN, http.StatusUnauthorized)
        return   // ← 数据库错误：直接中断请求，返回 401
    }
    if user == nil {
        http.Error(w, INVALID_AUTH_TOKEN, http.StatusUnauthorized)
        return   // ← token 过期/无效：直接中断请求，返回 401
    }
    // user 非空：写入 context
    ctx := AddUserToContext(r.Context(), user)
    r = r.WithContext(ctx)
}
```

**关键行为差异**：
- 如果请求**携带了** Cookie `auth-token` 但 token 已过期 → 中间件直接返回 HTTP 401，请求不会到达 GraphQL resolver。
- 如果请求**没有** Cookie `auth-token` → 中间件跳过，`UserFromContext` 返回 `nil`，请求正常进入 resolver。

**这意味着**：一旦请求携带了过期的登录 Cookie，整个请求在 HTTP 中间件层就被拦截，GraphQL resolver 根本没有机会执行分享 token 逻辑。

#### 1.3 WebSocket 场景的过期处理

**文件**: `api/graphql/auth/auth.go:92-131`

```go
func AuthWebsocketInit() ... {
    user, err := loaders.UserFromAccessToken.Load(*token)
    if user == nil {
        return nil, nil, errors.New(INVALID_AUTH_TOKEN)  // 过期 → WebSocket 握手失败
    }
    return userCtx, nil, nil
}
```

WebSocket 的 `InitFunc` 在连接建立时执行。如果 Bearer token 过期，握手直接失败，不会建立连接。

### 2. 分享 Token 的过期检查

**文件**: `api/graphql/resolvers/share_token.go:74-111`

```go
func (r *queryResolver) ShareToken(ctx context.Context, credentials models.ShareTokenCredentials) (*models.ShareToken, error) {
    // 查数据库
    db.Preload(clause.Associations).Where("value = ?", credentials.Token).First(&token)

    // 过期检查
    if token.Expire != nil && fakeTime.After(*token.Expire) {
        return nil, errors.New("share expired")  // ← 过期直接报错
    }

    // 密码检查
    if token.Password != nil {
        bcrypt.CompareHashAndPassword(...)
    }
    return &token, nil
}
```

分享 token 的过期检查在 GraphQL resolver 层执行，是请求处理链中**靠后**的环节。

### 3. 双过期场景的优先级判断

#### 场景 A：请求同时携带过期登录 Cookie + 有效分享 Token

```
请求: Cookie: auth-token=expired  +  GraphQL: { album(id:1, tokenCredentials:{token:"valid"}) }
```

**执行流程**：

1. `auth.Middleware` 读取 Cookie `auth-token`
2. `loaders.UserFromAccessToken.Load("expired")` → SQL `WHERE expire > NOW()` → 返回 `nil`
3. `user == nil` → **HTTP 401，请求终止**

**结果**：分享 Token 完全没有机会被校验。即使分享 Token 有效，请求也在 HTTP 中间件层被拒绝。

#### 场景 B：请求同时携带过期登录 Cookie + 过期分享 Token

同场景 A，HTTP 中间件层直接返回 401。

#### 场景 C：请求无登录 Cookie + 过期分享 Token

```
请求: 无 Cookie  +  GraphQL: { album(id:1, tokenCredentials:{token:"expired"}) }
```

**执行流程**：

1. `auth.Middleware` 没找到 Cookie → 跳过，`UserFromContext` 将返回 `nil`
2. 请求进入 GraphQL resolver
3. `Album` resolver 检查 `tokenCredentials != nil` → 调用 `ShareToken()`
4. `ShareToken()` 检查过期 → 返回 `"share expired"` 错误
5. `Album` resolver 返回错误，**不会回退到登录态**（因为 `UserFromContext` 为 `nil`）

**结果**：返回分享 Token 过期错误。

#### 场景 D：请求无登录 Cookie + 有效分享 Token

正常通过分享 Token 校验，返回数据。

#### 场景 E：请求携带有效登录 Cookie + 过期分享 Token

```
请求: Cookie: auth-token=valid  +  GraphQL: { album(id:1, tokenCredentials:{token:"expired"}) }
```

**执行流程**：

1. `auth.Middleware` 读取 Cookie → `UserFromAccessToken.Load("valid")` → 返回有效 user → 写入 context
2. 请求进入 GraphQL resolver
3. `Album` resolver 检查 `tokenCredentials != nil` → 调用 `ShareToken()`
4. `ShareToken()` 检查过期 → 返回 `"share expired"` 错误
5. **Album resolver 直接返回错误，不会回退到登录态**

```go
// album.go:131-136
if tokenCredentials != nil {
    shareToken, err := r.ShareToken(ctx, *tokenCredentials)
    if err != nil {
        return nil, err  // ← 分享 token 过期直接报错，不走登录态
    }
}
```

**结果**：即使登录态有效，只要提供了过期的分享 Token，请求也会被拒绝。

#### 场景 F：请求携带有效登录 Cookie + 有效分享 Token（但 album 不匹配）

```
请求: Cookie: auth-token=valid  +  GraphQL: { album(id:999, tokenCredentials:{token:"valid_for_album_1"}) }
```

**执行流程**：

1. 登录态有效，写入 context
2. `ShareToken()` 校验通过，但 album id 不匹配
3. **回退到登录态**：`user = UserFromContext(ctx)` → `actions.Album(db, user, 999)` → `OwnsAlbum` 校验

**结果**：分享 Token 不匹配时，登录态兜底。

### 4. 优先级矩阵总结

| 登录态 | 分享 Token | 提供方式 | 结果 | 原因 |
|--------|-----------|---------|------|------|
| 有效 | 有效且匹配 | Cookie + GraphQL 参数 | 分享 Token 路径通过 | Token 优先 |
| 有效 | 有效但不匹配 | Cookie + GraphQL 参数 | 登录态兜底 | Token 不匹配后回退 |
| 有效 | 过期 | Cookie + GraphQL 参数 | **失败** | Token 过期直接报错，不回退 |
| 过期 | 有效 | Cookie + GraphQL 参数 | **失败（HTTP 401）** | 中间件层拦截，Token 无机会 |
| 过期 | 过期 | Cookie + GraphQL 参数 | **失败（HTTP 401）** | 中间件层拦截 |
| 无 | 有效且匹配 | 仅 GraphQL 参数 | 分享 Token 路径通过 | 正常分享流程 |
| 无 | 过期 | 仅 GraphQL 参数 | **失败** | Token 过期报错 |
| 无 | 不提供 | — | **失败** | 无凭证 |

### 5. 核心发现

1. **中间件层是硬拦截**：一旦请求携带了 `auth-token` Cookie（无论是否过期），中间件就会执行校验。过期则 HTTP 401 直接返回，GraphQL 分享 Token 逻辑完全没有执行机会。这是两层机制最关键的耦合点——登录态过期检查在 HTTP 层，分享 Token 过期检查在 GraphQL resolver 层，HTTP 层先于 resolver 层执行。

2. **分享 Token 过期不回退是设计决策**：`album.go:134` 中 `if err != nil { return nil, err }` 使得分享 Token 过期时不会回退到登录态，即使用户已登录且有效。这是一种安全策略——避免攻击者通过提供无效 token 来探测合法登录用户的权限。

3. **HTTP 路由层的互斥性不同**：`authenticate_routes.go:19-46` 中，登录态和分享 Token 是 `if/else` 互斥关系（`user != nil` 走登录，`user == nil` 走 token），不存在双过期冲突问题。但 GraphQL 层是顺序执行（先 token 后登录），存在上述优先级问题。

---

## 补充三：完全公开链接（无 token）与登录态共存时的优先级判定

### 场景描述

"完全公开链接"指的是请求**不携带任何分享 token**（GraphQL 参数无 `tokenCredentials`，HTTP URL 无 `?token=`），但请求可能携带了有效的登录 Cookie。此时系统如何判断权限？

### 1. GraphQL 层：无 token 直接走登录态

**文件**: `api/graphql/resolvers/album.go:128-162`

```go
func (r *queryResolver) Album(ctx context.Context, id int, tokenCredentials *models.ShareTokenCredentials) (*models.Album, error) {
    db := r.DB(ctx)

    if tokenCredentials != nil {   // ← tokenCredentials 为 nil，整个分支跳过
        // ... 分享 token 逻辑
    }

    // 直接走到这里
    user := auth.UserFromContext(ctx)
    if user == nil {
        return nil, auth.ErrUnauthorized   // 未登录 → 拒绝
    }

    return actions.Album(db, user, id)     // 登录态 → OwnsAlbum 校验
}
```

当 `tokenCredentials == nil` 时，分享 token 分支完全跳过，直接进入登录态校验。逻辑极其清晰：**没有 token 就只看登录态**。

#### 各 Query 字段在无 token 时的行为

| Query 字段 | 无 token + 已登录 | 无 token + 未登录 |
|-----------|-----------------|-----------------|
| `album(id)` | `OwnsAlbum` 校验 | `ErrUnauthorized` |
| `media(id)` | `user_albums` SQL 校验 | `ErrUnauthorized` |
| `myAlbums` | directive `@isAuthorized` 通过 → `FillAlbums` | directive 拦截 |
| `myMedia` | directive `@isAuthorized` 通过 → `FillAlbums` | directive 拦截 |
| `search` | resolver 内 `UserFromContext` 通过 → `user_albums` | `ErrUnauthorized` |
| `shareToken(credentials)` | 不涉及（需提供 credentials） | 不涉及 |

**结论**：GraphQL 层不存在"无 token 时的优先级歧义"——没有 token 就只有登录态一条路，没登录就拒绝。

### 2. HTTP 路由层：无 token 时匿名请求直接被拒

**文件**: `api/routes/authenticate_routes.go:19-46`

```go
func authenticateMedia(media *models.Media, db *gorm.DB, r *http.Request) (...) {
    user := auth.UserFromContext(r.Context())

    if user != nil {
        // 有登录态 → OwnsAlbum 校验
        ownsAlbum, _ := user.OwnsAlbum(db, &album)
        if !ownsAlbum { return false }
    } else {
        // 无登录态 → 走分享 token
        shareTokenFromRequest(db, r, &media.ID, &media.AlbumID)
    }
    return true
}
```

`shareTokenFromRequest`（`authenticate_routes.go:71-76`）：

```go
func shareTokenFromRequest(...) (...) {
    token := r.URL.Query().Get("token")
    if token == "" {
        return false, "unauthorized", http.StatusForbidden, errors.New("share token not provided")
    }
    // ...
}
```

**关键**：URL 中没有 `?token=` 参数时，`shareTokenFromRequest` 直接返回 403 `"unauthorized"`，不会尝试其他鉴权方式。

#### HTTP 路由层四种组合

| 登录态 | URL ?token= | 结果 | 走哪条路 |
|--------|------------|------|---------|
| 有效 | 无 | `OwnsAlbum` 校验 | `user != nil` 分支 |
| 有效 | 有 | `OwnsAlbum` 校验 | `user != nil` 分支（**token 被忽略**） |
| 无 | 有 | `shareTokenFromRequest` | `user == nil` 分支 |
| 无 | 无 | **403 拒绝** | `shareTokenFromRequest` 返回 "token not provided" |

**核心发现**：HTTP 路由层中，**一旦有登录态，分享 token 完全被忽略**。这是与 GraphQL 层最大的差异——GraphQL 层先看 token 再看登录态，HTTP 层只看登录态（有登录态就不看 token）。

### 3. 与 GraphQL 层优先级差异的根因

| 维度 | GraphQL 层 | HTTP 路由层 |
|------|-----------|------------|
| 代码模式 | `if tokenCredentials != nil { ... } ; user := ...` 顺序 | `if user != nil { ... } else { ... }` 互斥 |
| token 优先级 | token 优先，登录态兜底 | 登录态优先，token 仅匿名可用 |
| 有登录 + 有 token | 两条路径都走（token 优先） | 只走登录态，忽略 token |
| 有登录 + 无 token | 只走登录态 | 只走登录态 |
| 无登录 + 无 token | `ErrUnauthorized` | 403 "share token not provided" |

差异的根因在于两层的代码结构不同：
- GraphQL 层 `album(id, tokenCredentials)` 把 token 作为显式参数，先处理参数再查 context，形成"token 优先"。
- HTTP 路由层 `authenticateMedia` / `authenticateAlbum` 先查 context 中的 user，有 user 就不需要 token，形成"登录态优先"。

**实际影响**：一个已登录用户通过分享链接访问时，两层的鉴权路径不同。GraphQL 层会先校验分享 token（如果提供了的话），HTTP 层则完全靠登录态。这意味着：
- 如果用户有权访问某相册，但分享 token 无效/过期，GraphQL 请求可能失败（token 优先，失败不回退），而 HTTP 图片请求正常（忽略 token，只看登录态）。
- 这会导致分享页面上出现"GraphQL 查询报错但图片能加载"的不一致现象。

---

## 补充四：匿名访问下相册封面缩略图的 ACL 检查路径

### 场景描述

匿名用户（未登录）通过分享链接访问相册时，前端需要加载：
1. 相册封面缩略图（`Album.thumbnail` → `Media.thumbnail` → `MediaURL.url`）
2. 相册内的媒体缩略图（`Album.media` → `Media.thumbnail` → `MediaURL.url`）

这些图片 URL 指向 HTTP `/photo/{mediaName}` 路由，请求到达后端时需经过 ACL 检查。以下追踪完整路径。

### 1. GraphQL 层：封面缩略图的数据获取

#### 1.1 Album.thumbnail resolver

**文件**: `api/graphql/resolvers/album.go:72-75`

```go
func (r *albumResolver) Thumbnail(ctx context.Context, obj *models.Album) (*models.Media, error) {
    return obj.Thumbnail(r.DB(ctx))
}
```

**文件**: `api/graphql/models/album.go:83-115`

```go
func (a *Album) Thumbnail(db *gorm.DB) (*Media, error) {
    if a.CoverID != nil {
        return db.First(&media, *a.CoverID)  // 有手动封面 → 直接查
    }
    // 无手动封面 → 递归 CTE 找子相册中最新的一条 media
    db.Raw(`
        WITH RECURSIVE sub_albums AS (
            SELECT id FROM albums WHERE id = ?
            UNION ALL
            SELECT children.id FROM albums AS children
            INNER JOIN sub_albums ON children.parent_album_id = sub_albums.id
        )
        SELECT * FROM media WHERE media.album_id IN (SELECT id FROM sub_albums)
        ORDER BY media.id DESC LIMIT 1
    `, a.ID).Scan(&media)
}
```

**ACL 检查**：无。`Album.thumbnail` resolver 不检查 `UserFromContext`，不检查 `user_albums`，不检查分享 token。只要父 Album 对象已通过入口校验，thumbnail 就能获取。

#### 1.2 Media.thumbnail resolver

**文件**: `api/graphql/resolvers/media.go:23-25`

```go
func (r *mediaResolver) Thumbnail(ctx context.Context, obj *models.Media) (*models.MediaURL, error) {
    return dataloader.For(ctx).MediaThumbnail.Load(obj.ID)
}
```

**文件**: `api/dataloader/mediaURLLoader.go:44-52`

```go
func NewThumbnailMediaURLLoader(db *gorm.DB) *MediaURLLoader {
    return &MediaURLLoader{
        fetch: makeMediaURLLoader(db, func(query *gorm.DB) *gorm.DB {
            return query.Where("purpose IN ?", []string{"thumbnail", "video-thumbnail"})
        }),
    }
}
```

dataloader 只按 `media_id` 和 `purpose` 查 `media_urls` 表，**无 ACL 检查**。

#### 1.3 MediaURL.URL() — 生成图片 HTTP 地址

**文件**: `api/graphql/models/media.go:118-128`

```go
func (p *MediaURL) URL() string {
    imageURL := utils.ApiEndpointUrl()
    if p.Purpose != VideoWeb {
        imageURL.Path = path.Join(imageURL.Path, "photo", p.MediaName)
    } else {
        imageURL.Path = path.Join(imageURL.Path, "video", p.MediaName)
    }
    return imageURL.String()
}
```

`URL()` 生成的是类似 `http://host/api/photo/abc123_thumb.jpg` 的地址，**不带任何 token 参数**。token 的附加由前端完成。

### 2. 前端：分享链接中的 token 注入

**文件**: `ui/src/components/photoGallery/ProtectedMedia.tsx:13-25`

```typescript
const getProtectedUrl = (url?: string) => {
    if (url == undefined) return undefined

    const imgUrl = new URL(url, location.origin)

    // 从当前 URL 路径中提取分享 token
    const tokenRegex = location.pathname.match(/^\/share\/([\d\w]+)(\/?.*)$/)
    if (tokenRegex) {
        const token = tokenRegex[1]
        imgUrl.searchParams.set('token', token)   // ← 注入 ?token=xxx
    }

    return imgUrl.href
}
```

**关键逻辑**：前端 `getProtectedUrl` 从浏览器当前路径 `/share/<token>/...` 中提取 token，然后拼接到图片 URL 的 query 参数中。最终生成的图片请求形如：

```
GET /api/photo/abc123_thumb.jpg?token=SHARE_TOKEN_VALUE
```

**使用位置**：
- `ProtectedImage` 组件：`<img src={getProtectedUrl(src)} crossOrigin="use-credentials" />`
- `ProtectedVideo` 组件：`poster={getProtectedUrl(media.thumbnail?.url)}` 和 `<source src={getProtectedUrl(media.videoWeb.url)} />`
- `AlbumBox` 组件：`<AlbumBoxImage src={album.thumbnail?.thumbnail?.url} />` → 内部用 `ProtectedImage`

所有图片/视频的 `<img>` 和 `<video>` 标签都通过 `ProtectedImage` / `ProtectedVideo` 渲染，都会经过 `getProtectedUrl` 注入 token。

### 3. 后端 HTTP 层：图片请求的 ACL 校验完整路径

#### 3.1 请求到达路由处理器

**文件**: `api/server.go:74-98`

```
rootRouter.Use(dataloader.Middleware(db))    // 1. dataloader 注入
rootRouter.Use(auth.Middleware(db))          // 2. 登录态注入（从 Cookie）
rootRouter.Use(server.LoggingMiddleware)
rootRouter.Use(server.CORSMiddleware(devMode))

photoRouter := endpointRouter.PathPrefix("/photo").Subrouter()
routes.RegisterPhotoRoutes(db, photoRouter)  // 3. /photo/{name} 路由
```

中间件执行顺序：dataloader → auth → 业务路由。`auth.Middleware` 从 Cookie 读取登录态，匿名用户没有 Cookie，所以 `UserFromContext` 返回 `nil`。

#### 3.2 Photo 路由处理器

**文件**: `api/routes/photos.go:15-80`

```go
func RegisterPhotoRoutes(db *gorm.DB, router *mux.Router) {
    router.HandleFunc("/{name}", func(w http.ResponseWriter, r *http.Request) {
        mediaName := mux.Vars(r)["name"]

        // 1. 用 mediaName 查 MediaURL → 关联的 Media
        var mediaURL models.MediaURL
        db.Model(&models.MediaURL{}).Joins("Media").
            Select("media_urls.*").
            Where("media_urls.media_name = ?", mediaName).Scan(&mediaURL)

        // 2. ACL 校验
        if success, response, status, err := authenticateMedia(media, db, r); !success {
            w.WriteHeader(status)
            w.Write([]byte(response))
            return
        }

        // 3. 返回文件
        http.ServeFile(w, r, cachedPath)
    })
}
```

#### 3.3 authenticateMedia — 匿名 + 有 token 的路径

**文件**: `api/routes/authenticate_routes.go:19-46`

```go
func authenticateMedia(media, db, r) {
    user := auth.UserFromContext(r.Context())

    if user != nil {
        // 已登录路径（不会走到这里，因为是匿名）
    } else {
        // 匿名路径 → 走分享 token 校验
        shareTokenFromRequest(db, r, &media.ID, &media.AlbumID)
    }
}
```

#### 3.4 shareTokenFromRequest — 完整校验链

**文件**: `api/routes/authenticate_routes.go:71-155`

匿名请求到达 `shareTokenFromRequest` 时的完整校验链：

```
1. 提取 ?token= 参数
   └─ URL: /api/photo/abc123_thumb.jpg?token=SHARE_TOKEN_VALUE
   └─ token = "SHARE_TOKEN_VALUE"
   └─ 若 token 为空 → 403 "share token not provided"

2. 数据库查 ShareToken
   └─ db.Where("value = ?", token).First(&shareToken)
   └─ 不存在 → 403 "invalid share token"

3. 过期检查
   └─ if shareToken.Expire != nil && time.Now().After(*shareToken.Expire)
   └─ 过期 → 403 "invalid share token"

4. 密码检查（如有）
   └─ Cookie "share-token-pw-<token>" 中取密码
   └─ bcrypt 比对
   └─ 不匹配 → 403 "share token password invalid"

5. 类型匹配检查
   ├─ ShareToken 关联的是 AlbumID：
   │   ├─ token.AlbumID == media.AlbumID → 通过（精确匹配）
   │   └─ token.AlbumID != media.AlbumID → 递归 CTE 查子相册
   │       └─ 子相册包含该 album → 通过
   │       └─ 不包含 → 403 "invalid share token"
   └─ ShareToken 关联的是 MediaID：
       └─ token.MediaID == media.ID → 通过（精确匹配）
       └─ 不匹配 → 403 "invalid share token"
```

### 4. 封面缩略图的特殊路径

#### 4.1 封面 media 可能来自子相册

`Album.Thumbnail`（`album.go:83-115`）使用递归 CTE 从子相册中查找封面 media。这意味着封面 media 的 `album_id` 可能**不等于**分享 token 关联的 `AlbumID`。

**文件**: `api/routes/authenticate_routes.go:125-147`

```go
if shareToken.AlbumID != nil && *albumID != *shareToken.AlbumID {
    // 递归查子相册
    var count int
    db.Raw(`
        WITH recursive child_albums AS (
            SELECT * FROM albums WHERE parent_album_id = ?
            UNION ALL
            SELECT child.* FROM albums child
            JOIN child_albums parent ON parent.id = child.parent_album_id
        )
        SELECT COUNT(id) FROM child_albums WHERE id = ?
    `, *shareToken.AlbumID, albumID).Find(&count)
    if count == 0 { return false }
}
```

这个递归 CTE 起点是 `shareToken.AlbumID` 的直接子相册（`parent_album_id = ?`），能匹配到任意深度的子相册中的 media。**封面缩略图即使来自深层子相册，也能通过分享 token 校验。**

#### 4.2 但递归方向与 OwnsAlbum 不同

| 场景 | OwnsAlbum（登录态） | shareTokenFromRequest（匿名） |
|------|-------------------|--------------------------|
| 封面 media 在 token 相册自身 | ✅ 直接匹配 `album_id == token.AlbumID` | ✅ 直接匹配 |
| 封面 media 在子相册 | ✅ `user_albums` 有子相册记录 | ✅ 递归 CTE 匹配子相册 |
| 封面 media 在父相册 | ✅ `OwnsAlbum` 向上递归 | ❌ CTE 向下递归，**不匹配父相册** |

**注意**：虽然封面 media 在父相册的情况在实际中不太可能发生（`Album.Thumbnail` 的 SQL 只从自身和子相册找 media），但如果封面被手动设置为父相册的 media（通过 `setAlbumCover`），匿名分享时该封面图片可能无法通过 HTTP 路由层的 ACL 校验。

### 5. 完整调用链时序图

```
匿名用户访问分享页面 /share/ABC123/1
│
├─ 前端 GraphQL 查询
│   shareToken(credentials: {token: "ABC123"})
│   │
│   ├─ shareToken resolver 校验（过期+密码）→ 通过
│   │   └─ 返回 ShareToken { album: { id: 1, ... } }
│   │
│   └─ album(id: 1, tokenCredentials: {token: "ABC123"})
│       │
│       ├─ ShareToken 校验 → 通过
│       ├─ AlbumID 精确匹配 → 返回 Album#1
│       │
│       ├─ Album.thumbnail resolver（无 ACL）→ 返回 Media#42
│       │   └─ Media.thumbnail resolver（无 ACL）→ 返回 MediaURL { url: "/api/photo/abc_thumb.jpg" }
│       │
│       ├─ Album.subAlbums resolver（无 ACL）→ [Album#2, Album#3]
│       │   └─ Album#2.thumbnail → MediaURL { url: "/api/photo/def_thumb.jpg" }
│       │
│       └─ Album.media resolver（无 ACL）→ [Media#50, Media#51, ...]
│           └─ Media#50.thumbnail → MediaURL { url: "/api/photo/ghi_thumb.jpg" }
│
├─ 前端渲染图片
│   ProtectedImage src="/api/photo/abc_thumb.jpg"
│   │
│   └─ getProtectedUrl() 注入 token
│       → 实际请求: GET /api/photo/abc_thumb.jpg?token=ABC123
│
└─ HTTP 图片请求
    │
    ├─ auth.Middleware：无 Cookie → UserFromContext 为 nil
    │
    ├─ photo route handler：
    │   1. 用 "abc_thumb" 查 MediaURL → Media{ID:42, AlbumID:1}
    │   2. authenticateMedia(media, db, r)
    │      ├─ user == nil → 走 shareTokenFromRequest
    │      │   ├─ URL ?token=ABC123 → 非空
    │      │   ├─ db 查 ShareToken → 找到，未过期，无密码
    │      │   ├─ shareToken.AlbumID=1, media.AlbumID=1 → 精确匹配 ✅
    │      │   └─ 返回 true
    │      └─ ACL 通过 → 返回图片文件
    │
    └─ 子相册封面: GET /api/photo/def_thumb.jpg?token=ABC123
        │
        └─ shareTokenFromRequest
            ├─ shareToken.AlbumID=1, media.AlbumID=2 → 不匹配
            ├─ 递归 CTE 查 AlbumID=1 的子相册
            │   → 子相册包含 Album#2 → count > 0 ✅
            └─ ACL 通过
```

### 6. 关键发现

1. **GraphQL 层与 HTTP 层的 ACL 检查是独立的两套**：GraphQL resolver 获取 MediaURL 对象时不做 ACL 检查，HTTP 图片请求到达时才做 ACL 检查。两层各有自己的逻辑，且互不感知。

2. **前端 token 注入是唯一桥梁**：匿名用户能看到图片，完全依赖 `getProtectedUrl`（`ProtectedMedia.tsx:13-25`）从 URL 路径中提取分享 token 并拼到图片 URL 上。如果前端组件没有用 `ProtectedImage`/`ProtectedVideo`（比如直接用 `<img src={url}>`），token 就不会注入，图片请求会被 403 拒绝。

3. **封面缩略图的 ACL 有隐藏风险**：当封面 media 来自子相册时，HTTP 层的 `shareTokenFromRequest` 能通过递归 CTE 匹配。但如果封面被 `setAlbumCover` 设置为不属于 token 相册树的 media（理论上不可能，但代码未显式防护），HTTP 层会返回 403，导致前端 GraphQL 拿到了 URL 但图片加载失败。

4. **已登录用户通过分享链接访问时的冗余**：如果用户同时有登录 Cookie 和分享 token，HTTP 层会走 `user != nil` 分支（忽略 token），而 GraphQL 层会先走 token 校验。两层对同一请求的鉴权路径不同，可能导致不一致。

5. **`crossOrigin="use-credentials"` 的必要性**：前端所有 `ProtectedImage` 和 `ProtectedVideo` 都设置了 `crossOrigin="use-credentials"`，这确保浏览器在请求图片/视频时会携带 Cookie（包括 `auth-token` 和 `share-token-pw-*`）。没有这个属性，浏览器在跨域请求中不会发 Cookie，导致即使用户已登录，HTTP 图片请求也会被当作匿名处理。
