# Resolver 批量加载与 N+1 查询分析

## 概述

本文档分析 PhotoView 项目中 GraphQL Resolver 的字段加载机制、DataLoader 批量合并策略、缓存命中逻辑，以及相册列表打开时的 N+1 查询问题。

项目使用 `github.com/vektah/dataloaden` 代码生成器创建 DataLoader，并通过 `github.com/99designs/gqlgen` 框架实现 GraphQL 服务。gqlgen 默认会**并行执行**同层级的多个 resolver，这使得 DataLoader 的批量合并机制能够正常工作。

---

## 一、DataLoader 架构总览

### 1.1 HTTP 请求管道中的中间件注入

DataLoader 通过 Gorilla Mux 中间件在请求处理早期注入到 context 中，位于认证中间件之前：

```
请求进入
    │
    ▼
rootRouter.Use(dataloader.Middleware(db))  ← DataLoader 注入（api/server.go:75）
    │
    ▼
rootRouter.Use(auth.Middleware(db))        ← 用户认证
    │
    ▼
rootRouter.Use(server.LoggingMiddleware)
    │
    ▼
rootRouter.Use(server.CORSMiddleware)
    │
    ▼
GraphQL Handler（gqlgen 并行执行 resolver）
```

**代码位置**：`api/server.go:74-78`

### 1.2 DataLoader 生成与类型

项目使用 `github.com/vektah/dataloaden` 工具生成 DataLoader 代码，共定义了 **5 个 DataLoader 实例**，封装在 `Loaders` 结构体中：

| Loader 名称              | 类型                  | Key 类型          | Value 类型          | 用途                     |
| ------------------------- | --------------------- | ----------------- | ------------------- | ------------------------ |
| `MediaThumbnail`          | `MediaURLLoader`      | `int` (media ID)  | `*models.MediaURL`  | 加载媒体缩略图 URL       |
| `MediaHighres`            | `MediaURLLoader`      | `int` (media ID)  | `*models.MediaURL`  | 加载高清图 URL           |
| `MediaVideoWeb`           | `MediaURLLoader`      | `int` (media ID)  | `*models.MediaURL`  | 加载 Web 优化视频 URL    |
| `UserFromAccessToken`     | `UserLoader`          | `string` (token)  | `*models.User`      | 通过 access token 查用户 |
| `UserMediaFavorite`       | `UserFavoritesLoader` | `*UserMediaData`  | `bool`              | 查询用户是否收藏媒体     |

**代码位置**：`api/dataloader/loaders.go:15-21`

### 1.3 中间件注入

DataLoader 通过 HTTP 中间件注入到请求上下文中，每个请求创建独立的 Loader 实例：

```go
func Middleware(db *gorm.DB) mux.MiddlewareFunc {
    return mux.MiddlewareFunc(func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx := context.WithValue(r.Context(), loadersKey, &Loaders{
                MediaThumbnail:      NewThumbnailMediaURLLoader(db),
                MediaHighres:        NewHighresMediaURLLoader(db),
                MediaVideoWeb:       NewVideoWebMediaURLLoader(db),
                UserFromAccessToken: NewUserLoaderByToken(db),
                UserMediaFavorite:   NewUserFavoriteLoader(db),
            })
            r = r.WithContext(ctx)
            next.ServeHTTP(w, r)
        })
    })
}
```

**代码位置**：`api/dataloader/loaders.go:23-40`

**关键特性**：
- **请求级隔离**：每个 HTTP 请求拥有独立的 DataLoader 实例和缓存
- **生命周期**：随请求开始而创建，随请求结束而销毁
- **上下文获取**：通过 `dataloader.For(ctx)` 从 context 中获取
- **5 个 Loader 并行独立**：每个 Loader 维护自己的 batch 和 cache，互不干扰

---

## 二、DataLoader 核心机制详解

### 2.1 通用结构（以 MediaURLLoader 为例）

```go
type MediaURLLoader struct {
    fetch    func(keys []int) ([]*models.MediaURL, []error)  // 批量获取函数
    wait     time.Duration                                     // 等待窗口（5ms）
    maxBatch int                                               // 最大批量（100）
    cache    map[int]*models.MediaURL                          // 缓存（懒加载）
    batch    *mediaURLLoaderBatch                              // 当前批次
    mu       sync.Mutex                                        // 互斥锁
}
```

**代码位置**：`api/dataloader/gen_mediaurlloader.go:34-55`

### 2.2 Load 调用流程（完整时序）

调用 `loader.Load(key)` 的完整流程：

```
调用 Load(key)
    │
    ├─ 调用 LoadThunk(key)
    │   │
    │   ├─ 加锁 (mu.Lock)
    │   │
    │   ├─ 检查缓存
    │   │   ├─ 命中 → 返回缓存值的 thunk 函数 ──────────────────┐
    │   │   └─ 未命中 → 继续                                    │
    │   │                                                         │
    │   ├─ 创建/获取当前 batch                                     │
    │   │   └─ 若 batch 为 nil，创建新 batch（带 done channel）    │
    │   │                                                         │
    │   ├─ 将 key 加入 batch（keyIndex）                          │
    │   │   ├─ key 已存在 → 返回已有位置                           │
    │   │   └─ key 不存在 → 追加到末尾                             │
    │   │       ├─ 首个 key → 启动定时器 goroutine (startTimer)    │
    │   │       └─ 达到 maxBatch → 立即触发批量获取 (end)          │
    │   │                                                         │
    │   └─ 解锁 (mu.Unlock)                                      │
    │       │                                                     │
    │       └─ 返回 thunk 函数 ◄──────────────────────────────────┘
    │
    └─ 执行 thunk 函数
        │
        ├─ 等待 batch.done channel（阻塞）
        │
        ├─ 根据位置从 batch.data 获取结果
        │
        ├─ 处理错误（单错误 / 按位置错误）
        │
        ├─ 若无错误，将结果写入缓存
        │
        └─ 返回结果
```

**代码位置**：`api/dataloader/gen_mediaurlloader.go:66-112`

**gqlgen 并行执行关键点**：
- gqlgen 对同层级的多个字段 resolver 会并行启动 goroutine
- 每个 resolver 调用 `Load(key)` 后返回一个 thunk
- gqlgen 的 field resolver 会立即调用 thunk 执行（阻塞等待）
- 多个并行的 resolver 会在 5ms 窗口内累积 key，从而触发批量合并

### 2.3 批量合并机制

#### 2.3.1 收集阶段（keyIndex）

```go
func (b *mediaURLLoaderBatch) keyIndex(l *MediaURLLoader, key int) int {
    // 去重：检查 key 是否已在 batch 中
    for i, existingKey := range b.keys {
        if key == existingKey {
            return i  // 返回已有位置
        }
    }

    pos := len(b.keys)
    b.keys = append(b.keys, key)
    
    // 第一个 key：启动等待定时器
    if pos == 0 {
        go b.startTimer(l)
    }

    // 达到最大批量：立即执行
    if l.maxBatch != 0 && pos >= l.maxBatch-1 {
        if !b.closing {
            b.closing = true
            l.batch = nil    // 断开当前 batch，新请求创建新 batch
            go b.end(l)      // 异步执行批量获取
        }
    }

    return pos
}
```

**代码位置**：`api/dataloader/gen_mediaurlloader.go:181-203`

**合并逻辑细节**：
1. **去重机制**：线性扫描 `b.keys`，值比较（MediaURLLoader 为 `int` 比较，UserFavoritesLoader 为**指针比较**）
2. **启动条件**：
   - 第一个 key 加入时，启动 5ms 定时器 goroutine
   - 当 key 数量达到 `maxBatch-1`（即第 100 个 key）时，立即触发
3. **断开逻辑**：触发批量获取前，将 `l.batch = nil`，新的 `Load` 调用会创建新 batch

#### 2.3.2 等待定时器（startTimer）

```go
func (b *mediaURLLoaderBatch) startTimer(l *MediaURLLoader) {
    time.Sleep(l.wait)  // 等待 5ms
    l.mu.Lock()
    
    // 如果已因达到 maxBatch 而关闭，则不重复执行
    if b.closing {
        l.mu.Unlock()
        return
    }

    l.batch = nil  // 断开当前 batch
    l.mu.Unlock()
    
    b.end(l)  // 执行批量获取
}
```

**代码位置**：`api/dataloader/gen_mediaurlloader.go:205-219`

#### 2.3.3 批量获取（end）

```go
func (b *mediaURLLoaderBatch) end(l *MediaURLLoader) {
    b.data, b.error = l.fetch(b.keys)  // 调用用户提供的 fetch 函数
    close(b.done)                      // 通知所有等待的 thunk
}
```

**代码位置**：`api/dataloader/gen_mediaurlloader.go:221-224`

### 2.4 缓存机制详解

#### 2.4.1 缓存写入时机

缓存的写入发生在 **thunk 执行完成后**，而非批量获取返回时：

```go
// 在 thunk 函数中
if err == nil {
    l.mu.Lock()
    l.unsafeSet(key, data)  // 写入缓存
    l.mu.Unlock()
}
```

**代码位置**：`api/dataloader/gen_mediaurlloader.go:104-108`

这意味着：
- 即使批量 fetch 返回了多个 key 的结果，只有当对应位置的 thunk 被实际执行时，该 key 的结果才会被写入缓存
- 同一 key 第二次 Load 时，若第一次的 thunk 已执行完毕，则缓存命中；若第一次 thunk 仍在等待 `batch.done`，则会走批量合并逻辑（加入同一个 batch）

#### 2.4.2 缓存结构对比

| Loader 类型              | Cache Key 类型       | 缓存 Key 对比方式       | 去重比较方式           |
| ------------------------- | -------------------- | ----------------------- | ---------------------- |
| `MediaURLLoader`          | `int`                | 值比较 (`==`)           | 值比较                 |
| `UserLoader`              | `string`             | 值比较 (`==`)           | 值比较                 |
| `UserFavoritesLoader`     | `*UserMediaData`     | **指针比较** (`==`)     | **指针比较**           |

**⚠️ UserFavoritesLoader 的缓存问题详解**：

`UserFavoritesLoader` 的 key 类型是 `*models.UserMediaData`（指针），缓存 map 定义为：

```go
cache map[*models.UserMediaData]bool  // api/dataloader/gen_userfavoritesloader.go:47
```

在 `mediaResolver.Favorite` 中使用方式：
```go
return dataloader.For(ctx).UserMediaFavorite.Load(&models.UserMediaData{
    UserID:  user.ID,
    MediaID: obj.ID,
})
```
**代码位置**：`api/graphql/resolvers/media.go:76-79`

**问题场景示例**：
```go
// 第一次调用：创建指针对象 A
key1 := &models.UserMediaData{UserID: 1, MediaID: 100}
loader.Load(key1)  // 缓存 key = key1（指针地址）

// 第二次调用：相同内容，但创建了新指针对象 B
key2 := &models.UserMediaData{UserID: 1, MediaID: 100}
loader.Load(key2)  // 缓存 key = key2（不同指针地址）
                   // ❌ 缓存未命中！因为 key1 != key2（指针不同）
```

**影响**：
- batch 内去重同样基于指针比较，相同内容的不同指针会被当成不同 key，导致重复查询
- 缓存命中率低于预期，相同 (UserID, MediaID) 组合可能被多次查询

**代码位置**：`api/dataloader/gen_userfavoritesloader.go:47, 178-183`

#### 2.4.3 Prime 预填充

DataLoader 支持通过 `Prime` 方法预填充缓存：

```go
func (l *MediaURLLoader) Prime(key int, value *models.MediaURL) bool
```

`Prime` 的特殊行为（MediaURLLoader/UserLoader）：
```go
// make a copy when writing to the cache, its easy to pass a pointer in from a loop var
// and end up with the whole cache pointing to the same value.
cpy := *value
l.unsafeSet(key, &cpy)
```

会对 value 进行深拷贝，避免循环变量引用问题。但 `UserFavoritesLoader.Prime` **不进行拷贝**，因为 value 类型为 `bool`（值类型）。

**代码位置**：`api/dataloader/gen_mediaurlloader.go:152-163`

#### 2.4.4 懒加载初始化

缓存 map 采用懒加载方式创建：
```go
func (l *MediaURLLoader) unsafeSet(key int, value *models.MediaURL) {
    if l.cache == nil {
        l.cache = map[int]*models.MediaURL{}
    }
    l.cache[key] = value
}
```
即如果从未有过成功的查询，cache 将始终为 `nil`，不占内存。

---

## 三、各 DataLoader 的 Fetch 实现

### 3.1 MediaURLLoader（通用模板）

`makeMediaURLLoader` 是工厂函数，接受一个 filter 回调来定制查询条件。

```go
func makeMediaURLLoader(db *gorm.DB, filter func(query *gorm.DB) *gorm.DB) func(keys []int) ([]*models.MediaURL, []error) {
    return func(mediaIDs []int) ([]*models.MediaURL, []error) {
        var urls []*models.MediaURL
        query := db.Where("media_id IN (?)", mediaIDs)
        query = filter(query)
        
        if err := query.Find(&urls).Error; err != nil {
            return nil, []error{err}
        }

        // 构建 mediaID → MediaURL 映射
        resultMap := make(map[int]*models.MediaURL, len(mediaIDs))
        for _, url := range urls {
            resultMap[url.MediaID] = url
        }

        // 按输入顺序返回结果
        result := make([]*models.MediaURL, len(mediaIDs))
        for i, mediaID := range mediaIDs {
            result[i] = resultMap[mediaID]  // 找不到为 nil
        }

        return result, nil
    }
}
```

**代码位置**：`api/dataloader/mediaURLLoader.go:13-42`

**关键保证**：输出长度必须等于输入长度，且第 `i` 个输出对应第 `i` 个输入 key。找不到对应数据时返回零值（`nil`）。

#### 三个具体实例的差异：

| 实例               | Filter 条件                                                                 |
| ------------------ | --------------------------------------------------------------------------- |
| `MediaThumbnail`   | `purpose IN (thumbnail, video-thumbnail)`                                   |
| `MediaHighres`     | `purpose = high-res OR (purpose = original AND content_type IN web-types)`  |
| `MediaVideoWeb`    | `purpose IN (video-web, original)`                                          |

**Highres 和 VideoWeb 的 ORDER BY 细节**：
```go
// Highres: PhotoHighRes 优先，找不到时回退到 MediaOriginal
Order("media_id ASC, CASE purpose WHEN 'original' THEN 0 WHEN 'high-res' THEN 1 END ASC")

// VideoWeb: VideoWeb 优先，找不到时回退到 MediaOriginal
Order("media_id ASC, CASE purpose WHEN 'original' THEN 0 WHEN 'video-web' THEN 1 END ASC")
```
由于 `resultMap[url.MediaID] = url` 会**后写入覆盖先写入**，ORDER BY 使得优先项（high-res/video-web）排在后面，从而覆盖回退项（original）。

**代码位置**：`api/dataloader/mediaURLLoader.go:54-81`

### 3.2 UserLoader（通过 Token）

```go
func NewUserLoaderByToken(db *gorm.DB) *UserLoader {
    return &UserLoader{
        maxBatch: 100,
        wait:     5 * time.Millisecond,
        fetch: func(tokens []string) ([]*models.User, []error) {
            // 步骤1: 查询有效的 access tokens
            var accessTokens []*models.AccessToken
            db.Where("expire > ?", time.Now()).Where("value IN (?)", tokens).Find(&accessTokens)

            // 步骤2: 获取去重后的关联 user_id 列表
            // 使用原始 SQL 行扫描去重
            rows, _ := db.Table("access_tokens").
                Select("distinct user_id").
                Where("expire > ?", time.Now()).
                Where("value IN (?)", tokens).
                Rows()
            
            userIDs := make([]int, 0)
            for rows.Next() {
                var id int
                db.ScanRows(rows, &id)
                userIDs = append(userIDs, id)
            }
            rows.Close()

            // 步骤3: 批量查询用户
            var users []*models.User
            if len(userIDs) > 0 {
                db.Where("id IN (?)", userIDs).Find(&users)
            }
            
            // 步骤4: 构建映射并按输入顺序返回结果
            userMap := make(map[int]*models.User)
            tokenMap := make(map[string]*models.AccessToken)
            // ...填充 map...
            
            result := make([]*models.User, len(tokens))
            for i, token := range tokens {
                // 通过 token → accessToken → user 的链查找
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

**代码位置**：`api/dataloader/userLoader.go:10-71`

**查询次数**：最多 3 次 SQL 查询（access_tokens → distinct user_id → users），无论输入多少个 token。

### 3.3 UserFavoritesLoader

```go
func NewUserFavoriteLoader(db *gorm.DB) *UserFavoritesLoader {
    return &UserFavoritesLoader{
        maxBatch: 100,
        wait:     5 * time.Millisecond,
        fetch: func(keys []*models.UserMediaData) ([]bool, []error) {
            // 步骤1: 提取去重后的 userID 和 mediaID 集合
            userIDMap := make(map[int]struct{}, len(keys))
            mediaIDMap := make(map[int]struct{}, len(keys))
            for _, key := range keys {
                userIDMap[key.UserID] = struct{}{}
                mediaIDMap[key.MediaID] = struct{}{}
            }

            // 转换为切片
            uniqueUserIDs := make([]int, len(userIDMap))
            uniqueMediaIDs := make([]int, len(mediaIDMap))
            // ...填充...

            // 步骤2: 批量查询收藏记录（使用笛卡尔积条件）
            var userMediaFavorites []*models.UserMediaData
            db.Where("user_id IN (?)", uniqueUserIDs).
                Where("media_id IN (?)", uniqueMediaIDs).
                Where("favorite = TRUE").
                Find(&userMediaFavorites)

            // 步骤3: 按输入顺序匹配结果（O(N*M) 嵌套循环）
            result := make([]bool, len(keys))
            for i, key := range keys {
                favorite := false
                for _, fav := range userMediaFavorites {
                    if fav.UserID == key.UserID && fav.MediaID == key.MediaID {
                        favorite = true
                        break
                    }
                }
                result[i] = favorite
            }

            return result, nil
        },
    }
}
```

**代码位置**：`api/dataloader/userFavoriteLoader.go:10-60`

**性能注意**：步骤 3 使用嵌套循环匹配，时间复杂度 O(keys_count * favorites_count)。当 batch 较大且收藏较多时可能成为瓶颈。

---

## 四、相册列表的 N+1 查询分析

### 4.1 GraphQL Schema 定义

相册列表查询的 Schema 定义（`api/graphql/resolvers/album.graphql`）：

```graphql
type Album {
  id: ID!
  title: String!
  parent: Album
  subAlbums(order: Ordering, paginate: Pagination): [Album!]!
  media(order: Ordering, paginate: Pagination, onlyFavorites: Boolean): [Media!]!
  thumbnail: Media           # ← 此字段存在 N+1 问题
  owner: User!
  path: [Album!]!
  shares: [ShareToken!]!
}

type Query {
  myAlbums(
    order: Ordering,
    paginate: Pagination,
    onlyRoot: Boolean,
    showEmpty: Boolean,
    onlyWithFavorites: Boolean
  ): [Album!]!
}
```

### 4.2 前端查询

前端查询（`ui/src/Pages/AllAlbumsPage/AlbumsPage.tsx:11-28`）：

```graphql
query getMyAlbums($orderBy: String, $orderDirection: OrderDirection) {
  myAlbums(
    order: { order_by: $orderBy, order_direction: $orderDirection }
    onlyRoot: true
    showEmpty: true
  ) {
    id
    title
    thumbnail {           # ← N+1 问题来源
      id
      thumbnail {         # ← 使用 DataLoader，无 N+1
        url
      }
    }
  }
}
```

### 4.3 完整调用链路与查询统计

假设相册列表返回 N 个相册：

```
1. myAlbums 查询
   └─ queryResolver.MyAlbums()  [api/graphql/resolvers/album.go:119-126]
      └─ actions.MyAlbums()     [api/graphql/models/actions/album_actions.go:9-48]
         ├─ user.FillAlbums()   [api/graphql/models/user.go:154-165]
         │   └─ db.Model(&user).Association("Albums").Find()
         │      → 1 次 SQL：查询 user_albums + albums 关联
         │
         └─ db.Where("id IN (?)", userAlbumIDs).Find(&albums)
            → 1 次 SQL：查询完整 album 数据
   ✅ 总计：2 次查询，返回 N 个 Album 对象

2. 每个 Album 的 thumbnail 字段解析（gqlgen 并行执行 N 个 resolver）
   └─ albumResolver.Thumbnail()  [api/graphql/resolvers/album.go:72-75]
      └─ obj.Thumbnail(r.DB(ctx))  [api/graphql/models/album.go:83-115]
         ├─ 路径 A（有 CoverID）：
         │  └─ db.First(&media, *a.CoverID)
         │     → 1 次 SQL / 每个相册
         │
         └─ 路径 B（无 CoverID）：
            └─ db.Raw(recursive CTE query).Scan(&media)
               → WITH RECURSIVE sub_albums ... SELECT * FROM media ... LIMIT 1
               → 1 次 SQL / 每个相册（含递归子查询）
   ❌ 总计：N 次查询（每个相册一次）

3. 每个 thumbnail Media 的 thumbnail 字段解析（gqlgen 并行执行）
   └─ mediaResolver.Thumbnail()  [api/graphql/resolvers/media.go:22-25]
      └─ dataloader.For(ctx).MediaThumbnail.Load(obj.ID)
         └─ 5ms 窗口内合并 → 1 次批量 SQL：WHERE media_id IN (?, ?, ...)
   ✅ 总计：1 次查询（批量）
```

**总计**：2（相册列表）+ N（相册封面）+ 1（缩略图 URL）= **N + 3 次 SQL 查询**

### 4.4 Album.Thumbnail() 方法的两种路径详解

```go
func (a *Album) Thumbnail(db *gorm.DB) (*Media, error) {
    var media Media

    // 路径 A：相册显式设置了 CoverID
    if a.CoverID != nil {
        if err := db.First(&media, *a.CoverID).Error; err != nil {
            return nil, err
        }
        return &media, nil
    }

    // 路径 B：无 CoverID，递归查找第一个子相册中的最新媒体
    query := `
        WITH RECURSIVE sub_albums AS (
            SELECT id FROM albums WHERE id = ?
            UNION ALL
            SELECT children.id FROM albums AS children
            INNER JOIN sub_albums ON children.parent_album_id = sub_albums.id
        )
        SELECT * FROM media
        WHERE media.album_id IN (SELECT id FROM sub_albums)
        ORDER BY media.id DESC
        LIMIT 1
    `

    if err := db.Raw(query, a.ID).Scan(&media).Error; err != nil {
        return nil, err
    }

    if media.ID == 0 {
        return nil, nil // Return nil for empty albums
    }

    return &media, nil
}
```

**代码位置**：`api/graphql/models/album.go:83-115`

**路径 B 的代价更高**：每次查询都涉及递归 CTE，扫描整个相册树的媒体表。

### 4.5 N+1 问题总结

| 字段                 | 解析方式               | 是否批量 | 查询次数 |
| -------------------- | ---------------------- | -------- | -------- |
| `myAlbums`           | 直接 SQL 查询          | ✅ 是     | 1 + 1    |
| `Album.thumbnail`    | 逐个调用 `Thumbnail()` | ❌ 否     | N        |
| `Media.thumbnail`    | DataLoader 批量        | ✅ 是     | 1        |
| `Media.highRes`      | DataLoader 批量        | ✅ 是     | 1        |
| `Media.videoWeb`     | DataLoader 批量        | ✅ 是     | 1        |
| `Media.favorite`     | DataLoader 批量        | ✅ 是     | 1        |
| `Media.album`        | 逐个查询               | ❌ 否     | N        |
| `Media.exif`         | 逐个查询               | ❌ 否     | N        |
| `Media.faces`        | 逐个查询               | ❌ 否     | N        |

---

## 五、相册详情页的 N+1 查询分析

### 5.1 调用链路

假设相册中有 M 个媒体：

```
1. album 查询 → 1 次查询

2. album.media 查询 → 1 次查询（获取 M 个 Media）
   └─ albumResolver.Media()  [api/graphql/resolvers/album.go:21-51]
      └─ db.Where("media.album_id = ?", obj.ID)
         .Where("media.id IN (subquery for media_urls)")
         .Find(&media)
   ✅ 总计：1 次查询

3. 各字段解析（gqlgen 并行执行 M 个 resolver）：
   ├─ Media.thumbnail → MediaThumbnail.Load → 1 次 ✅
   ├─ Media.highRes   → MediaHighres.Load   → 1 次 ✅
   ├─ Media.videoWeb  → MediaVideoWeb.Load  → 1 次 ✅
   └─ Media.favorite  → UserMediaFavorite.Load → 1 次 ✅
   ✅ 总计：4 次批量查询

4. 若查询 exif / faces / album 字段：
   ├─ Media.exif  → r.DB(ctx).Model(obj).Association("Exif").Find()
   │  → 1 次 SQL / 每个 media ❌
   ├─ Media.faces → r.DB(ctx).Model(obj).Association("Faces").Find()
   │  → 1 次 SQL / 每个 media ❌
   └─ Media.album → r.DB(ctx).Find(&album, obj.AlbumID)
      → 1 次 SQL / 每个 media ❌
   ❌ 总计：3M 次查询（N+1）
```

---

## 六、DataLoader 字段合并明细

### 6.1 已合并的字段（使用 DataLoader）

| Resolver 字段          | DataLoader           | 合并 Key | fetch 中的 SQL                          |
| ---------------------- | -------------------- | -------- | --------------------------------------- |
| `Media.thumbnail`      | `MediaThumbnail`     | media ID | `WHERE media_id IN (?) AND purpose IN (thumbnail, video-thumbnail)` |
| `Media.highRes`        | `MediaHighres`       | media ID | `WHERE media_id IN (?) AND (purpose = high-res OR ...)` |
| `Media.videoWeb`       | `MediaVideoWeb`      | media ID | `WHERE media_id IN (?) AND purpose IN (video-web, original)` |
| `Media.favorite`       | `UserMediaFavorite`  | (userID, mediaID) 指针 | `WHERE user_id IN (?) AND media_id IN (?) AND favorite = TRUE` |
| 用户认证（内部使用）   | `UserFromAccessToken`| token    | `WHERE value IN (?) AND expire > ?` → 关联 users 表 |

### 6.2 未合并的字段（存在 N+1 风险）

| Resolver 字段     | 实现方式                     | N+1 场景          | 单次查询类型    | 潜在影响       |
| ----------------- | ---------------------------- | ----------------- | --------------- | -------------- |
| `Album.thumbnail` | `obj.Thumbnail(db)`          | 相册列表          | First / 递归CTE | 高（相册数量多）|
| `Album.subAlbums` | 直接 `Find` 查询              | 嵌套相册列表      | SELECT WHERE IN | 中             |
| `Album.owner`     | 未实现（panic）              | -                 | -               | -              |
| `Album.shares`    | 直接 `Find` 查询              | 相册列表          | SELECT WHERE    | 中             |
| `Media.album`     | 直接 `Find` 查询              | 媒体列表          | SELECT BY ID    | 中             |
| `Media.exif`      | Association `Find`            | 媒体列表含 exif   | Association     | 高             |
| `Media.faces`     | Association `Find`            | 媒体列表含 faces  | Association     | 高             |
| `Media.shares`    | 直接 `Find` 查询              | 媒体列表          | SELECT WHERE    | 低             |
| `Media.downloads` | 直接 `Find` 查询              | 下载列表          | SELECT WHERE    | 低             |

---

## 七、缓存命中处理过程详解

### 7.1 缓存命中判定流程

```
Load(key) 被调用
    │
    ├─ mu.Lock()
    │
    ├─ l.cache == nil ?
    │   └─ 是 → 缓存未初始化，跳过检查
    │
    ├─ it, ok := l.cache[key]
    │   │
    │   ├─ ok == true（命中）
    │   │   ├─ mu.Unlock()
    │   │   └─ 返回立即完成的 thunk: func() { return it, nil }
    │   │      → 不会加入 batch，不会触发 fetch
    │   │
    │   └─ ok == false（未命中）
    │       ├─ 创建/获取 batch
    │       ├─ batch.keyIndex(l, key) → 加入批量
    │       ├─ mu.Unlock()
    │       └─ 返回等待 batch.done 的 thunk
    │
    └─ thunk 执行后（批量 fetch 返回）
        └─ err == nil ?
            └─ 是 → l.unsafeSet(key, data) → 写入缓存
```

**代码位置**：`api/dataloader/gen_mediaurlloader.go:73-112`

### 7.2 缓存命中场景

1. **同请求内重复查询相同 key**：
   ```go
   // 第一次调用：未命中，加入 batch
   url1, _ := loader.Load(100)
   // thunk 执行完成后，100 被写入缓存
   
   // 第二次调用：命中缓存，直接返回，无 SQL
   url2, _ := loader.Load(100)
   ```

2. **Prime 预填充**：通过 `Prime` 方法提前将已知数据写入缓存，后续 `Load` 直接命中。

3. **同 batch 内重复 key**：通过 `keyIndex` 中的去重逻辑，相同 key 在 batch 中只出现一次，减少 fetch 参数数量。

### 7.3 缓存未命中场景

1. **首次调用**：key 不在缓存中
2. **不同请求**：每个 HTTP 请求有独立的 DataLoader 实例，缓存不跨请求共享
3. **指针 key 问题**：`UserFavoritesLoader` 使用指针作为 key，相同内容的不同指针对象不会命中
4. **查询失败**：当 `err != nil` 时，thunk 不会将结果写入缓存，下次 Load 仍会重新查询

### 7.4 缓存生命周期

```
HTTP 请求开始
    │
    ├─ dataloader.Middleware 创建 Loaders 对象
    │   └─ 5 个 Loader 实例，cache 均为 nil（懒加载）
    │
    ├─ GraphQL 查询处理
    │   ├─ Resolver 调用 Loader.Load(key)
    │   │   ├─ 首次成功后 cache map 被创建
    │   │   └─ 后续相同 key 命中缓存
    │   └─ ...
    │
    └─ HTTP 请求结束
        └─ Loaders 对象失去引用，被 GC 回收
           └─ 所有 cache 随之释放
```

---

## 八、改进建议

### 8.1 解决 Album.thumbnail 的 N+1 问题（最高优先级）

**问题**：相册列表中，每个相册的 `thumbnail` 字段触发一次独立查询，含递归 CTE 的查询尤其昂贵。

**方案**：实现 AlbumThumbnail DataLoader，批量获取相册封面。

```go
// 伪代码：批量获取相册封面
func NewAlbumThumbnailLoader(db *gorm.DB) *AlbumThumbnailLoader {
    return &AlbumThumbnailLoader{
        maxBatch: 100,
        wait:     5 * time.Millisecond,
        fetch: func(albumIDs []int) ([]*models.Media, []error) {
            // 步骤1: 查询有 CoverID 的相册
            var albumsWithCover []struct {
                AlbumID int
                CoverID *int
            }
            db.Model(&models.Album{}).
                Select("id, cover_id").
                Where("id IN (?)", albumIDs).
                Find(&albumsWithCover)
            
            // 步骤2: 提取 CoverID 列表，批量查询对应 Media
            coverIDs := collectCoverIDs(albumsWithCover)
            var coverMedia []*models.Media
            if len(coverIDs) > 0 {
                db.Where("id IN (?)", coverIDs).Find(&coverMedia)
            }
            
            // 步骤3: 对无 CoverID 的相册，使用批量递归查询
            // （可以用单个 UNION ALL 查询处理多个根相册）
            
            // 步骤4: 按输入顺序组装结果
        },
    }
}
```

**预期收益**：相册列表查询次数从 N+3 降低到约 4-5 次。

### 8.2 解决 UserFavoritesLoader 的缓存问题

**问题**：使用指针作为 cache key，导致相同内容无法命中缓存，batch 去重也失效。

**方案 A**：使用复合字符串 key：
```go
// 改造为 string key: fmt.Sprintf("%d:%d", userID, mediaID)
// cache map[string]bool
```

**方案 B**：自定义 hashCode 或实现 `comparable` 接口（Go 1.21+）。

### 8.3 扩展更多 DataLoader

为高频查询字段添加 DataLoader：

| 新增 Loader          | 对应字段         | Key 类型     |
| -------------------- | ---------------- | ------------ |
| `MediaExifLoader`    | `Media.exif`     | media ID     |
| `MediaFacesLoader`   | `Media.faces`    | media ID     |
| `AlbumByIDLoader`    | `Media.album`    | album ID     |
| `SubAlbumsLoader`    | `Album.subAlbums`| parent album ID |

---

## 九、关键代码位置索引

| 功能                | 文件路径                                                    | 行号   |
| ------------------- | ----------------------------------------------------------- | ------ |
| 中间件注册          | `api/server.go`                                             | 75     |
| Loaders 结构体定义  | `api/dataloader/loaders.go`                                 | 15-21  |
| 中间件注入实现      | `api/dataloader/loaders.go`                                 | 23-40  |
| MediaURLLoader 生成 | `api/dataloader/gen_mediaurlloader.go`                      | -      |
| LoadThunk 核心实现  | `api/dataloader/gen_mediaurlloader.go`                      | 73-112 |
| keyIndex 批量收集   | `api/dataloader/gen_mediaurlloader.go`                      | 181-203|
| MediaURL fetch 实现 | `api/dataloader/mediaURLLoader.go`                          | 13-82  |
| UserFavorite fetch  | `api/dataloader/userFavoriteLoader.go`                      | 10-60  |
| UserLoader fetch    | `api/dataloader/userLoader.go`                              | 10-71  |
| UserFavorites 缓存  | `api/dataloader/gen_userfavoritesloader.go`                 | 47     |
| 相册列表 resolver   | `api/graphql/resolvers/album.go`                            | 119-126|
| 相册 thumbnail 解析 | `api/graphql/resolvers/album.go`                            | 72-75  |
| 媒体 thumbnail 解析 | `api/graphql/resolvers/media.go`                            | 22-25  |
| 媒体 favorite 解析  | `api/graphql/resolvers/media.go`                            | 69-80  |
| Album.Thumbnail()   | `api/graphql/models/album.go`                               | 83-115 |
| MyAlbums action     | `api/graphql/models/actions/album_actions.go`               | 9-48   |
| User.FillAlbums()   | `api/graphql/models/user.go`                                | 154-165|
| 前端相册列表查询    | `ui/src/Pages/AllAlbumsPage/AlbumsPage.tsx`                 | 11-28  |
