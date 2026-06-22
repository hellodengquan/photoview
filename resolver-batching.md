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

## 八、GraphQL Subscription 长连接场景下的 DataLoader 缓存生命周期

### 8.1 WebSocket 连接建立与 Context 传递

PhotoView 的 GraphQL subscription 基于 gqlgen 的 `transport.Websocket` 实现，DataLoader 通过 HTTP 中间件注入到 context 中后，会伴随整个 WebSocket 长连接的生命周期。

**完整连接建立链路**：

```
客户端发起 WebSocket 升级请求 (HTTP GET + Upgrade header)
    │
    ▼
rootRouter.Use(dataloader.Middleware(db))  [api/server.go:75]
    └─ 创建 Loaders 对象，注入 context  ←─────── DataLoader 在此诞生
    │
    ▼
rootRouter.Use(auth.Middleware(db))  [api/server.go:76]
    └─ 尝试从 Cookie 读取 auth-token（WebSocket 通常不带 Cookie）
    │
    ▼
/graphql endpoint → transport.Websocket
    │
    ▼
WebSocket 协议升级成功 → 进入 WebSocket 协议
    │
    ▼
客户端发送 "connection_init" 消息
    │
    ▼
transport.Websocket.InitFunc = auth.AuthWebsocketInit()  [api/graphql/endpoint/graphql_endpoint.go:34]
    └─ 从 initPayload["Authorization"] 提取 Bearer token
    └─ 从 context 中获取 dataloader
    └─ loaders.UserFromAccessToken.Load(token) → 验证用户
    └─ 将 user 写入 context
    └─ 返回新的 context（包含 user + 原始 dataloader）
    │
    ▼
WebSocket 连接建立完成，等待 subscription 操作
```

**关键代码位置**：
- WebSocket 传输配置：`api/graphql/endpoint/graphql_endpoint.go:31-35`
- WebSocket 认证初始化：`api/graphql/auth/auth.go:92-130`

### 8.2 DataLoader 在 WebSocket 中的存活范围

**核心结论**：DataLoader 实例在整个 WebSocket 连接期间是**同一个实例**，缓存会在所有 subscription 操作之间共享和累积。

原因分析：

```go
// api/dataloader/loaders.go:23-40
// DataLoader 中间件在 HTTP 请求级别创建实例
func Middleware(db *gorm.DB) mux.MiddlewareFunc {
    return mux.MiddlewareFunc(func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx := context.WithValue(r.Context(), loadersKey, &Loaders{...})
            r = r.WithContext(ctx)
            next.ServeHTTP(w, r)  // WebSocket 升级在此 handler 内完成
        })
    })
}
```

WebSocket 升级发生在 `next.ServeHTTP` 调用内部，升级完成后：
- 原始 HTTP 请求的 `context` 被 WebSocket 传输层持有
- 后续所有 WebSocket 消息（包括 subscription）都从这个 context 派生
- DataLoader 实例跟随 context 一起存活，**不会随单个 subscription 结束而销毁**

### 8.3 Subscription 执行流程中的 DataLoader 使用

以 `notification` subscription 为例：

```
客户端发送 "start" 消息（subscription 操作）
    │
    ▼
gqlgen 执行 _Subscription(ctx, selectionSet)  [api/graphql/generated.go:11512]
    └─ context 来自连接级 context（包含 dataloader）
    │
    ▼
ec._Subscription_notification(ctx, field)  [api/graphql/generated.go:7567]
    └─ graphql.ResolveFieldStream(...)
        │
        ├─ 调用 resolver → ec.Resolvers.Subscription().Notification(ctx)
        │   └─ subscriptionResolver.Notification(ctx)  [api/graphql/resolvers/notification.go:18-34]
        │       ├─ user := auth.UserFromContext(ctx)  ← 从连接级 context 获取用户
        │       ├─ notification.RegisterListener(user, channel)
        │       ├─ go func() { <-ctx.Done(); DeregisterListener() }()
        │       └─ 返回 notificationChannel (<-chan *models.Notification)
        │
        └─ 监听 channel，每次有新值时：
            ├─ 调用 marshalNNotification2...(ctx, selections, v)
            │   └─ 解析 Notification 类型的各字段
            │       ├─ key, type, header, content... （标量，无 DataLoader）
            │       └─ 如果有嵌套复杂字段，会触发对应 resolver
            │           └─ resolver 中可通过 dataloader.For(ctx) 获取 Loader
            │
            └─ 通过 WebSocket 发送 "next" 消息给客户端
```

**代码位置**：
- Subscription resolver：`api/graphql/resolvers/notification.go:17-34`
- 生成的 subscription 执行：`api/graphql/generated.go:7567-7585`

### 8.4 多次推送事件下的缓存累积

当前 Notification 类型的字段全是标量（String/Boolean/Float/Int），不触发 DataLoader 调用，因此缓存累积问题不明显。

**但如果 Notification 包含嵌套的 Media 字段（如 notification.media），则会出现以下行为**：

```
时间线：
T0: WebSocket 连接建立 → DataLoader 实例创建，cache = nil
    │
T1: 订阅 notification subscription
    │
T2: 第 1 次通知推送
    │   └─ Notification.media 解析
    │       └─ MediaThumbnail.Load(100) → 未命中缓存 → 加入 batch → fetch SQL
    │       └─ 缓存写入 {100: MediaURL{...}}
    │
T3: 第 2 次通知推送
    │   └─ Notification.media 解析
    │       └─ MediaThumbnail.Load(100) → ✅ 命中缓存！无 SQL
    │       └─ MediaThumbnail.Load(200) → 未命中 → 加入 batch → fetch SQL
    │       └─ 缓存累积：{100: ..., 200: ...}
    │
T4: 第 N 次通知推送
    │
T5: WebSocket 连接断开 → DataLoader 被 GC 回收 → 缓存释放
```

**关键特性**：
- 缓存**跨多次推送事件**持续累积
- 同一 media ID 在第一次推送时查询，后续推送直接命中缓存
- 缓存大小随推送次数单调增长（只要涉及新的 key）
- 连接不断开，缓存不释放

### 8.5 与普通 HTTP 请求的对比

| 维度 | HTTP 请求 | WebSocket Subscription |
|------|----------|---------------------|
| DataLoader 生命周期 | 单次 HTTP 请求 | 整个 WebSocket 连接 |
| 缓存持续时间 | 几百毫秒 ~ 几秒 | 几分钟 ~ 几小时 |
| 缓存累积速度 | 单查询内累积 | 随推送事件持续增长 |
| 内存风险 | 低（请求结束即释放） | 中高（长连接可能累积大量缓存） |
| 缓存失效机制 | 请求结束自动失效 | 需手动 Clear 或连接断开 |

### 8.6 长连接下的潜在问题与风险

#### 8.6.1 内存累积风险

对于高活跃、长时间保持连接的客户端：
- `MediaThumbnail` 缓存：每个 key 约 ~100 字节，10000 条约 1MB（可接受）
- 但如果新增更多 DataLoader（如 MediaFacesLoader），累积效应会放大
- 极端场景下：用户浏览大量媒体后，连接保持数小时，缓存可能膨胀

#### 8.6.2 数据过期问题

HTTP 请求模式下，缓存随请求结束销毁，天然避免了数据过期问题。

长连接模式下：
- 数据更新（如重新生成缩略图、用户取消收藏）后，缓存中仍是旧值
- 由于没有 TTL 失效机制，客户端会一直看到旧数据
- 直到连接断开重连，才能获取新数据

#### 8.6.3 UserFavoritesLoader 的指针问题被放大

由于 `UserFavoritesLoader` 使用指针作为 cache key：
- 每次推送创建新的 `*UserMediaData` 指针对象
- 即使 (UserID, MediaID) 相同，也无法命中缓存
- 长连接下，缓存条目持续增长但无法复用
- 既浪费内存又无法获得缓存收益

### 8.7 改进建议（长连接场景）

1. **定期清理缓存**：为长连接场景的 DataLoader 添加定时清理机制
   ```go
   // 伪代码：每隔 5 分钟清空缓存
   go func() {
       ticker := time.NewTicker(5 * time.Minute)
       defer ticker.Stop()
       for range ticker.C {
           l.mu.Lock()
           l.cache = nil
           l.mu.Unlock()
       }
   }()
   ```

2. **使用 LRU 缓存替代 map**：限制最大缓存条目数，避免无限增长

3. **修复 UserFavoritesLoader 的 key 类型**：使用值类型或字符串复合 key

4. **为不同场景配置不同的 DataLoader 策略**：
   - HTTP 请求：保持现状（请求级缓存）
   - WebSocket 长连接：使用带 TTL 的缓存或更小的 maxBatch

---

## 九、AlbumThumbnailLoader 批量获取相册封面的代码实现路径

### 9.1 现有实现的 N+1 问题根源

相册列表的 N+1 问题源于 `Album.Thumbnail()` 方法的逐次调用模式：

```go
// api/graphql/resolvers/album.go:72-75
func (r *albumResolver) Thumbnail(ctx context.Context, obj *models.Album) (*models.Media, error) {
    return obj.Thumbnail(r.DB(ctx))  // 每个相册调用一次
}
```

每个相册独立调用 `obj.Thumbnail(db)`，而该方法内部有两条查询路径，每条都是一次独立的 SQL：

```go
// api/graphql/models/album.go:83-115
func (a *Album) Thumbnail(db *gorm.DB) (*Media, error) {
    var media Media

    // 路径 A：有 CoverID → 简单查询（1 SQL）
    if a.CoverID != nil {
        if err := db.First(&media, *a.CoverID).Error; err != nil {
            return nil, err
        }
        return &media, nil
    }

    // 路径 B：无 CoverID → 递归 CTE 查询（1 SQL，但更重）
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
    // ...
}
```

**代码位置**：`api/graphql/models/album.go:83-115`

### 9.2 AlbumThumbnailLoader 的设计思路

批量获取相册封面的核心挑战：同一批相册中，有的有 CoverID，有的没有，需要分别处理后合并结果。

**整体策略：三阶段处理**
1. **批量查询 CoverID**：一次性获取所有相册的 cover_id 字段
2. **批量查询 Cover Media**：对有 CoverID 的相册，用 `IN` 查询批量获取对应 Media
3. **批量递归查询**：对无 CoverID 的相册，使用批量递归 CTE 查询子相册媒体

### 9.3 Fetch 函数的完整实现路径

以下是完整的 `AlbumThumbnailLoader.fetch` 实现路径详解：

```go
func NewAlbumThumbnailLoader(db *gorm.DB) *AlbumThumbnailLoader {
    return &AlbumThumbnailLoader{
        maxBatch: 100,
        wait:     5 * time.Millisecond,
        fetch: func(albumIDs []int) ([]*models.Media, []error) {
            // ============================================================
            // 阶段 1: 批量查询所有相册的 cover_id
            // ============================================================
            type albumCoverInfo struct {
                ID      int
                CoverID *int
            }
            var coverInfos []albumCoverInfo
            err := db.Model(&models.Album{}).
                Select("id, cover_id").
                Where("id IN (?)", albumIDs).
                Find(&coverInfos).Error
            if err != nil {
                return nil, []error{err}
            }
            // 用时：~1 次 SQL 查询
            // 结果：coverInfos = [{ID: 1, CoverID: 100}, {ID: 2, CoverID: nil}, ...]

            // ============================================================
            // 阶段 2: 分组 - 有 CoverID vs 无 CoverID
            // ============================================================
            albumToCoverID := make(map[int]*int, len(coverInfos))
            var withCoverAlbumIDs []int    // 有 CoverID 的相册
            var withoutCoverAlbumIDs []int // 无 CoverID 的相册

            for _, info := range coverInfos {
                albumToCoverID[info.ID] = info.CoverID
                if info.CoverID != nil {
                    withCoverAlbumIDs = append(withCoverAlbumIDs, info.ID)
                } else {
                    withoutCoverAlbumIDs = append(withoutCoverAlbumIDs, info.ID)
                }
            }

            // ============================================================
            // 阶段 3: 批量查询有 CoverID 的相册对应的 Media
            // ============================================================
            coverMediaMap := make(map[int]*models.Media)
            if len(withCoverAlbumIDs) > 0 {
                // 提取去重的 CoverID
                coverIDSet := make(map[int]struct{})
                for _, albumID := range withCoverAlbumIDs {
                    coverIDSet[*albumToCoverID[albumID]] = struct{}{}
                }
                coverIDList := make([]int, 0, len(coverIDSet))
                for id := range coverIDSet {
                    coverIDList = append(coverIDList, id)
                }

                // 批量查询 Media
                var coverMedia []*models.Media
                if err := db.Where("id IN (?)", coverIDList).Find(&coverMedia).Error; err != nil {
                    return nil, []error{err}
                }
                // 构建 mediaID → Media 映射
                for _, m := range coverMedia {
                    coverMediaMap[m.ID] = m
                }
            }
            // 用时：~1 次 SQL 查询（即使多个相册引用同一个 Cover Media，也只查一次）

            // ============================================================
            // 阶段 4: 批量查询无 CoverID 的相册（递归 CTE）
            // ============================================================
            noCoverThumbnailMap := make(map[int]*models.Media)
            if len(withoutCoverAlbumIDs) > 0 {
                // 对每个无 CoverID 的相册，递归查找其子相册中的最新媒体
                // 策略：使用单个 SQL 同时处理多个根相册
                query := `
                    WITH RECURSIVE sub_albums AS (
                        -- 锚点：每个根相册作为起点，标记 root_id
                        SELECT id AS album_id, id AS root_id FROM albums WHERE id IN (?)
                        UNION ALL
                        -- 递归：子相册继承 root_id
                        SELECT child.id AS album_id, sa.root_id
                        FROM albums AS child
                        INNER JOIN sub_albums sa ON child.parent_album_id = sa.album_id
                    )
                    -- 对每个 root_id，找到最新的媒体
                    SELECT DISTINCT ON (sa.root_id) m.*
                    FROM media m
                    INNER JOIN sub_albums sa ON m.album_id = sa.album_id
                    ORDER BY sa.root_id, m.id DESC
                `
                // 注意：DISTINCT ON 是 PostgreSQL 特性
                // 如果是 MySQL 需要用其他方式（如窗口函数 ROW_NUMBER）

                var thumbnailMedia []*models.Media
                rows, err := db.Raw(query, withoutCoverAlbumIDs).Rows()
                if err != nil {
                    return nil, []error{err}
                }
                defer rows.Close()

                // 扫描结果，构建 root_id → Media 映射
                for rows.Next() {
                    var rootID int
                    var media models.Media
                    // 需要手动扫描所有列...
                    // 更简单的方式是使用 GORM 的 Scan 配合一个结构体
                }
            }
            // 用时：~1 次 SQL 查询（无论多少个无 CoverID 的相册）
            // 注意：这是最重的查询，但比 N 次递归查询快得多

            // ============================================================
            // 阶段 5: 按输入顺序组装结果
            // ============================================================
            result := make([]*models.Media, len(albumIDs))
            for i, albumID := range albumIDs {
                coverID := albumToCoverID[albumID]
                if coverID != nil {
                    // 有 CoverID：从 coverMediaMap 获取
                    result[i] = coverMediaMap[*coverID]
                } else {
                    // 无 CoverID：从递归查询结果获取
                    result[i] = noCoverThumbnailMap[albumID]
                }
                // 找不到则为 nil（空相册）
            }

            return result, nil
        },
    }
}
```

**SQL 查询次数对比**：

| 方案 | 查询次数 | 说明 |
|------|---------|------|
| 原逐次查询 | N 次 | 每个相册 1 次，递归 CTE 代价高 |
| AlbumThumbnailLoader | ~3 次 | 1次查 CoverID + 1次查 Cover Media + 1次批量递归查询 |
| **优化后比例** | **~3/N** | N=100 时减少 97% 的查询次数 |

### 9.4 集成到 Resolver 的方式

修改 `albumResolver.Thumbnail` 方法，从直接调用改为使用 DataLoader：

```go
// 修改前：api/graphql/resolvers/album.go:72-75
func (r *albumResolver) Thumbnail(ctx context.Context, obj *models.Album) (*models.Media, error) {
    return obj.Thumbnail(r.DB(ctx))  // 每次单独查询
}

// 修改后
func (r *albumResolver) Thumbnail(ctx context.Context, obj *models.Album) (*models.Media, error) {
    return dataloader.For(ctx).AlbumThumbnail.Load(obj.ID)  // 批量合并
}
```

**集成步骤**：

1. **生成 Loader 代码**：使用 dataloaden 生成 `gen_albumthumbarloader.go`
   ```bash
   dataloaden -pkg dataloader -name AlbumThumbnail -key-type int -value-type "*github.com/photoview/photoview/api/graphql/models.Media"
   ```

2. **注册到 Loaders 结构体**：`api/dataloader/loaders.go`
   ```go
   type Loaders struct {
       MediaThumbnail      *MediaURLLoader
       MediaHighres        *MediaURLLoader
       MediaVideoWeb       *MediaURLLoader
       UserFromAccessToken *UserLoader
       UserMediaFavorite   *UserFavoritesLoader
       AlbumThumbnail      *AlbumThumbnailLoader  // 新增
   }
   ```

3. **在 Middleware 中初始化**：`api/dataloader/loaders.go`
   ```go
   ctx := context.WithValue(r.Context(), loadersKey, &Loaders{
       // ...
       AlbumThumbnail: NewAlbumThumbnailLoader(db),  // 新增
   })
   ```

4. **修改 albumResolver**：如上所示

### 9.5 与现有代码的兼容性考量

#### 9.5.1 保持模型方法不变

`Album.Thumbnail(db)` 方法应保留，因为：
- 其他地方可能直接调用（非 GraphQL 场景）
- 单元测试直接使用该方法（如 `album_test.go`）
- 符合单一职责原则（模型方法不依赖 DataLoader）

#### 9.5.2 递归 CTE 的数据库兼容性

当前代码使用 PostgreSQL 特有的 `DISTINCT ON` 语法。如果项目需要支持多种数据库：

- **PostgreSQL**：使用 `DISTINCT ON (root_id)`（当前方案）
- **MySQL 8.0+**：使用 `ROW_NUMBER() OVER (PARTITION BY root_id ORDER BY m.id DESC)` + 子查询
- **MySQL 5.x**：使用 `GROUP BY` + `MAX(id)` 关联查询

#### 9.5.3 错误处理

当前实现返回 `[]error` 长度为 1 的全局错误。如果需要逐 key 错误：
```go
// 逐 key 错误模式（更灵活，但实现更复杂）
errors := make([]error, len(albumIDs))
// ...
result[i] = media
errors[i] = nil  // 或具体错误
return result, errors
```

### 9.6 性能影响预估

假设相册列表返回 50 个相册，其中 40 个有 CoverID，10 个没有：

| 指标 | 现有方案 | DataLoader 方案 | 改善比例 |
|------|---------|----------------|---------|
| SQL 查询次数 | 50 次 | 3 次 | 94% ↓ |
| 递归 CTE 执行次数 | 10 次 | 1 次 | 90% ↓ |
| 总延迟（估算） | 50 × 5ms = 250ms | 3 × 5ms = 15ms | 94% ↓ |
| 数据库连接占用 | 高（50 次往返） | 低（3 次往返） | 显著降低 |

### 9.7 边界情况处理

| 场景 | 处理方式 |
|------|---------|
| 相册不存在 | 返回 nil（与现有行为一致） |
| 相册有 CoverID 但对应 Media 已删除 | 返回 nil（db.First 失败的等价行为） |
| 空相册（无任何媒体） | 返回 nil（与现有 `media.ID == 0` 判断一致） |
| CoverID 指向的媒体不属于该相册 | 仍然返回该媒体（与现有 db.First 行为一致，不校验归属） |
| 子相册有多个媒体 | 返回 ID 最大的（与现有 ORDER BY id DESC LIMIT 1 一致） |

---

## 十、分页大相册场景下的批量上限与降级机制

### 10.1 maxBatch 参数的含义与触发条件

所有 DataLoader 均配置了两个关键参数：
- `wait = 5 * time.Millisecond`：收集窗口
- `maxBatch = 100`：单批次最大 key 数量

**触发批量获取有两种独立的机制，先到先触发**：

```go
// api/dataloader/gen_mediaurlloader.go:181-203
func (b *mediaURLLoaderBatch) keyIndex(l *MediaURLLoader, key int) int {
    // ...去重检查...
    
    pos := len(b.keys)
    b.keys = append(b.keys, key)
    
    // 条件 A: 第一个 key 加入 → 启动定时器
    if pos == 0 {
        go b.startTimer(l)  // 5ms 后触发
    }

    // 条件 B: 达到 maxBatch → 立即触发
    if l.maxBatch != 0 && pos >= l.maxBatch-1 {
        // pos 从 0 开始，所以 pos==99 时就是第 100 个 key
        if !b.closing {
            b.closing = true
            l.batch = nil    // ★ 关键：断开当前 batch
            go b.end(l)      // ★ 异步执行：不阻塞当前 Load 调用
        }
    }

    return pos
}
```

**代码位置**：`api/dataloader/gen_mediaurlloader.go:181-203`

**两个触发条件的竞争关系**：

```
时间轴 (ms):  0        1        2        3        4        5
              │                                                │
条件 B:    [第100个 key 到达 → 立即触发]     [第200个 key → 再次触发]
                │                                     │
                └→ l.batch = nil, 新请求创建新 batch  └→ 同上

条件 A:    [启动 5ms 定时器] ──────────────────────────── [5ms 到点 → 触发]
                                                           │
                                                           └→ 如果 batch 已被条件 B 关闭
                                                              则 startTimer 中的 closing 检查
                                                              会跳过 end() 调用
```

**关键细节**：
- `pos >= l.maxBatch-1` 而非 `pos >= l.maxBatch`：pos 从 0 开始，第 100 个 key 的 pos = 99
- `go b.end(l)` 异步执行：当前 goroutine 立即返回，不会阻塞等待 SQL 返回
- `l.batch = nil` 断开引用：后续的 `Load` 调用会检测 `l.batch == nil`，创建新 batch

### 10.2 大相册分页场景的完整链路分析

**场景设定**：
- 某个相册有 5000 张图片（大相册）
- 前端查询：`album.media(paginate: { limit: 500, offset: 0 })`
- Media 字段选择：`{ id, thumbnail { url }, highRes { url }, favorite }`
- 即每个 Media 触发 **3 次 DataLoader 调用**（MediaThumbnail、MediaHighres、UserMediaFavorite）

**分页查询入口**：
```go
// api/graphql/resolvers/album.go:21-51
func (r *albumResolver) Media(ctx context.Context, obj *models.Album, order *models.Ordering, paginate *models.Pagination, onlyFavorites *bool) ([]*models.Media, error) {
    db := r.DB(ctx)
    query := db.Where("media.album_id = ?", obj.ID)
          .Where("media.id IN (subquery for media_urls)")
    // ... onlyFavorites 过滤 ...
    
    query = models.FormatSQL(query, order, paginate)  // 应用分页
    // paginate.Limit = 500 → tx.Limit(500)
    // paginate.Offset = 0   → tx.Offset(0)
    
    var media []*models.Media
    query.Find(&media)  // 返回 500 个 Media 对象
    return media, nil
}
```

**代码位置**：`api/graphql/resolvers/album.go:21-51`

**FormatSQL 实现**：
```go
// api/graphql/models/utils.go:11-40
func FormatSQL(tx *gorm.DB, order *Ordering, paginate *Pagination) *gorm.DB {
    if paginate != nil {
        if paginate.Limit != nil {
            tx.Limit(*paginate.Limit)    // 单页最大 500
        }
        if paginate.Offset != nil {
            tx.Offset(*paginate.Offset)
        }
    }
    // ... order 处理 ...
    return tx
}
```

### 10.3 500 个 Media 的 DataLoader 分批处理

以 `MediaThumbnail` DataLoader 为例，500 个 Media 会触发 500 次 `Load(mediaID)` 调用。由于 gqlgen **并行执行** resolver，这些调用会在短时间内密集到来。

**分批过程详解**：

```
T0 (ms):
  gqlgen 启动 500 个并行 field resolver，每个调用 MediaThumbnail.Load(mediaID)

  调用 1-100 (mediaID 1 到 100):
    ├─ Load(1):  pos=0 → 启动 5ms 定时器
    ├─ Load(2):  pos=1 → 无特殊操作
    ├─ ...
    └─ Load(100): pos=99 → 达到 maxBatch-1
                     ├─ b.closing = true
                     ├─ l.batch = nil  ← 断开！
                     └─ go b.end(l)    ← 异步执行 Batch #1
                            └→ SQL: WHERE media_id IN (1,2,...,100)

T0.5 (ms, 调用 101):
  Load(101): 检测到 l.batch == nil → 创建新 batch B2
    ├─ pos=0 → 启动**新的** 5ms 定时器（定时器 #2）
    └─ ...

  调用 101-200 (mediaID 101 到 200):
    └─ Load(200): pos=99 → 达到 maxBatch-1
                    ├─ b.closing = true
                    ├─ l.batch = nil
                    └─ go b.end(l)   ← 异步执行 Batch #2
                           └→ SQL: WHERE media_id IN (101,102,...,200)

T1.0 (ms, 调用 201):
  Load(201): l.batch == nil → 创建新 batch B3
    ...

T1.5 (ms): Batch #3 触发 (ID 201-300)
T2.0 (ms): Batch #4 触发 (ID 301-400)
T2.5 (ms): Batch #5 触发 (ID 401-500)

T5.0 (ms):
  定时器 #1 (从 T0 启动) 到时 → startTimer() 中检测 b.closing == true → 跳过，不重复执行
  定时器 #2 (从 T0.5 启动) 到时 → 同样跳过
  ...以此类推
```

**最终结果**：
| DataLoader | 调用次数 | 实际 SQL 批次数 | 每批 key 数 |
|-----------|---------|---------------|-----------|
| MediaThumbnail | 500 | 5 | 100, 100, 100, 100, 100 |
| MediaHighres | 500 | 5 | 100, 100, 100, 100, 100 |
| UserMediaFavorite | 500 | **可能 ~500** | 受指针 key 去重失败影响 |

**总计**：无分页限制时若 5000 张图片，会产生 50 批次 × 3 Loader = 150 次 SQL，远优于 5000×3 = 15000 次（降低 99%）。

### 10.4 降级机制：maxBatch 的保护作用

`maxBatch = 100` 不是性能优化参数，而是**数据库保护机制**：

#### 10.4.1 防止超长 IN 列表

PostgreSQL 对 SQL 语句长度和参数数量有限制（虽然很高，但不是无限）。超长 IN 列表会导致：
- SQL 解析器负担增加
- 查询计划优化时间变长
- 网络传输数据量大

**示例对比**：
```sql
-- 无 maxBatch 限制：5000 个 key 的 IN 列表
SELECT * FROM media_urls WHERE media_id IN (1,2,3,...,5000);
-- SQL 文本大小：约 30KB，参数绑定 5000 个

-- 有 maxBatch=100：拆分为 50 次查询
SELECT * FROM media_urls WHERE media_id IN (1,2,...,100);
-- SQL 文本大小：约 500B，参数绑定 100 个 × 50 次
```

#### 10.4.2 防止查询超时

单批次 10000 个 key 的查询可能因为：
- 回表扫描行数过多
- 临时表/排序内存不足
- 单事务持有时间过长

导致查询超时甚至数据库连接池耗尽。拆分为小批次后，每批查询时间可控。

#### 10.4.3 锁持有时间限制

InnoDB 等存储引擎在查询时会持有各种锁。超长查询会导致：
- 锁等待队列增长
- 写入操作被阻塞（ALTER TABLE、VACUUM 等维护操作无法执行）

### 10.5 分页场景下的极端情况分析

#### 情况 1：单个请求查询多个相册的媒体

```graphql
query {
  myAlbums {
    id
    media(paginate: { limit: 50 }) { id thumbnail { url } }
  }
}
```

假设返回 20 个相册，每个相册 50 个 media，共 1000 个 media：
- **注意**：不同相册的 `albumResolver.Media()` 是**并行执行**的（gqlgen 特性）
- 它们返回的 media 总数 = 20 × 50 = 1000 个 Media 对象
- gqlgen 解析这 1000 个 Media 的 `thumbnail` 字段时，所有 `Load` 调用会在 5ms 窗口内混合累积
- **DataLoader 不区分 media 来自哪个相册**，只按 media_id 去重和分批
- 最终 MediaThumbnail 会产生 **10 批次**（每批 100）

#### 情况 2：同一批次中重复的 media_id

场景：多个相册共享同一个封面（例如子相册继承父相册封面），此时 keyIndex 中的去重逻辑发挥作用：

```go
// api/dataloader/gen_mediaurlloader.go:182-187
for i, existingKey := range b.keys {
    if key == existingKey {   // 值比较
        return i              // 返回已有位置
    }
}
```

500 个 Load 调用中若有 100 个是重复的 mediaID：
- 实际 batch.keys 长度 = 400（而非 500）
- 产生 4 批次而非 5 批次
- 节省 20% 的 SQL 次数

#### 情况 3：定时器先触发，key 不足 100

场景：分页 limit = 50，且只有这一个字段需要 DataLoader。

```
T0: Load(1) → 启动 5ms 定时器
T0.1: Load(2) ~ T0.5: Load(50)
    └─ pos=49，未达到 maxBatch-1=99
T5: 定时器到时
    └─ 触发 batch.end() → SQL: media_id IN (1..50)
```

**降级**：产生 1 批次，每批 50 个 key。此时 SQL IN 列表较短，但仍然是批量查询，开销可接受。

### 10.6 降级时机判断与配置建议

当前 `maxBatch = 100`、`wait = 5ms` 是通用配置，但不同场景可以考虑不同策略：

| 场景 | 建议 maxBatch | 建议 wait | 理由 |
|-----|-------------|----------|-----|
| 相册媒体缩略图 | 200~500 | 5ms | media_urls 表小、查询简单 |
| 用户收藏（UserMediaFavorite） | 50~100 | 5ms | 笛卡尔积查询，查询更重 |
| 递归 CTE 查询（AlbumThumbnail） | 10~20 | 10ms | 查询极重，宁可多分几批 |
| 长连接 subscription | 20~50 | 10ms | 响应延迟比吞吐量更重要 |

**注意**：当前所有 Loader 硬编码为 `maxBatch=100, wait=5ms`，无法按场景配置。如需调整需修改各 Loader 工厂函数的初始化参数。

---

## 十一、并发 Subscription 同时订阅同一字段时的批次合并行为

### 11.1 进程级 Notification 广播机制

PhotoView 的通知系统基于**全局进程内数组 + 互斥锁**实现广播：

```go
// api/graphql/notification/Notification.go:28-30
var notificationListeners []*NotificationListener = make([]*NotificationListener, 0)
var nextNotificationId = 0
var notificationLock = &sync.Mutex{}
```

**广播过程**：

```go
// api/graphql/notification/Notification.go:70-83
func BroadcastNotification(notification *models.Notification) {
    notificationLock.Lock()
    defer notificationLock.Unlock()

    for _, listener := range notificationListeners {
        listener.channel <- notification
    }
}
```

**代码位置**：`api/graphql/notification/Notification.go:70-83`

**关键特性**：
- `notificationListeners` 是**全局单例数组**：所有 WebSocket 连接的订阅者都注册到同一个数组
- `BroadcastNotification` 是**同步遍历写入**：依次向每个 listener 的 channel 写入 notification 对象
- `listener.channel` 是 `chan<- *models.Notification`（写端 channel）
- 同一 notification 指针对象被**所有 listener 共享**（不是拷贝）

### 11.2 多订阅者场景的 DataLoader 隔离性

假设有 3 个客户端同时建立 WebSocket 连接并订阅 notification：

```
客户端 A (WebSocket Conn #1)
    └─ DataLoader 实例 LA (cache_A: map[int]*MediaURL)
    
客户端 B (WebSocket Conn #2)
    └─ DataLoader 实例 LB (cache_B: map[int]*MediaURL)
    
客户端 C (WebSocket Conn #3)
    └─ DataLoader 实例 LC (cache_C: map[int]*MediaURL)

全局 listeners 数组: [A-listener, B-listener, C-listener]
                        │            │            │
                        └─ channel_A  │            │
                                     └─ channel_B  │
                                                  └─ channel_C
```

**核心结论：跨连接 DataLoader 完全隔离，不共享 batch 和 cache。**

根本原因：
- `dataloader.Middleware` 在每个 WebSocket 升级的 HTTP 请求中创建独立的 Loaders 对象
- 每个 WebSocket 连接持有独立的 context，包含独立的 DataLoader
- `notificationListeners` 数组中只保存 channel，不共享任何 Loader 状态

### 11.3 单次广播事件触发多连接解析的时序

#### 11.3.1 假设 Notification 包含 Media 字段（扩展场景）

假设 Schema 定义为：
```graphql
type Notification {
    id: ID!
    key: String!
    relatedMedia: Media   # 新增：关联的媒体
}
```

客户端查询：
```graphql
subscription {
  notification {
    relatedMedia {
      id
      thumbnail { url }   # 使用 MediaThumbnail DataLoader
      favorite            # 使用 UserMediaFavorite DataLoader
    }
  }
}
```

#### 11.3.2 广播到解析的完整时序

```
T0: 扫描完成，调用 BroadcastNotification(&Notification{relatedMediaID: 100})
    │
    ├─ notificationLock.Lock()
    │
    ├─ 遍历 listeners:
    │   ├─ channel_A <- notif  (非阻塞写？取决于 channel buffer)
    │   ├─ channel_B <- notif
    │   └─ channel_C <- notif
    │
    └─ notificationLock.Unlock()

T0.1: A、B、C 的 subscription resolver 各自从 channel 收到 notif
    │
    ├─ 连接 A: gqlgen 启动字段解析
    │   ├─ Notification.relatedMedia → 返回 &Media{ID: 100}
    │   └─ Media.thumbnail 解析 → LA.Load(100)
    │       ├─ LA.cache[100] 未命中（首次加载）
    │       ├─ 创建 batch_A_1，keys=[100]，pos=0 → 启动 5ms 定时器
    │       └─ 返回 thunk_A
    │
    ├─ 连接 B: gqlgen 启动字段解析
    │   └─ Media.thumbnail 解析 → LB.Load(100)
    │       ├─ LB.cache[100] 未命中
    │       ├─ 创建 batch_B_1，keys=[100]，pos=0 → 启动 5ms 定时器
    │       └─ 返回 thunk_B
    │
    └─ 连接 C: gqlgen 启动字段解析
        └─ Media.thumbnail 解析 → LC.Load(100)
            ├─ LC.cache[100] 未命中
            ├─ 创建 batch_C_1，keys=[100]，pos=0 → 启动 5ms 定时器
            └─ 返回 thunk_C

T5.0: 三个定时器**各自**到时
    ├─ batch_A_1.end() → SQL (连接 A): WHERE media_id IN (100)  ← 独立查询
    ├─ batch_B_1.end() → SQL (连接 B): WHERE media_id IN (100)  ← 独立查询  
    └─ batch_C_1.end() → SQL (连接 C): WHERE media_id IN (100)  ← 独立查询

结果：3 个客户端查询相同的 media_id=100，但产生了 3 次独立 SQL 查询！
```

### 11.4 同一连接内多个 Subscription 的合并

**gqlgen 支持单个 WebSocket 连接上同时运行多个 subscription 操作**（通过 graphql-ws 协议的多 operation ID）。

```
同一个 WebSocket 连接 (Conn #1, DataLoader 实例 L1):
    ├─ subscription Op #1: notification { relatedMedia { thumbnail } }
    └─ subscription Op #2: mediaUpdated { media { thumbnail { url } } }
```

这两个订阅共享**同一个** DataLoader 实例 L1，存在 batch 合并的可能。

#### 11.4.1 合并发生的条件

```
T0: Op #1 收到 notification，包含 mediaID=100
    └─ L1.Load(100) → 创建 batch1，keys=[100]，启动 5ms 定时器

T0.2 (在 5ms 内): Op #2 收到 mediaUpdated 事件，也包含 mediaID=100
    └─ L1.Load(100) → 检测 batch1 存在
        ├─ keyIndex 去重：100 已存在于 keys[0]，返回 pos=0
        └─ 共享同一个 batch1，不重复添加 key

T5: batch1.end() → 1 次 SQL: media_id IN (100)
    ├─ thunk(Op #1, 100) ← 返回结果
    └─ thunk(Op #2, 100) ← 返回结果（同一个 batch，同一个值）

结果：合并成功，只产生 1 次 SQL（而不是 2 次）
```

#### 11.4.2 合并失败的场景

```
T0: Op #1 收到 notification，L1.Load(100) → batch1 创建，启动定时器

T5.1 (超过 5ms): batch1 已经触发 end()，SQL 执行中但结果未返回
    └─ 此时 Op #2 收到事件，L1.Load(100)
        ├─ 检查 L1.cache[100] → nil（thunk 还未执行完，缓存未写入）
        ├─ l.batch == nil（batch1 已被断开）
        ├─ 创建新 batch2，keys=[100]，启动新定时器

T6: batch1 的 SQL 执行完毕，thunk 执行 → L1.cache[100] = result

T10: batch2 定时器触发
    └─ end() → 第 2 次 SQL: media_id IN (100)
       （可以通过「在 fetch 之前检查缓存」的优化来避免）

结果：合并失败，产生 2 次相同 SQL（第 2 次可优化）
```

### 11.5 通知广播的阻塞风险与 channel buffer

在 `BroadcastNotification` 中，向 channel 写入是**阻塞**操作（默认是无缓冲 channel）：

```go
listener.channel <- notification   // 阻塞直到有 goroutine 读取
```

**问题场景**：某个订阅者解析慢（例如字段复杂、DataLoader 批量等待时间长），其 channel 读取不及时。

```
Listeners: [Fast-A, Slow-B, Fast-C]

Broadcast 循环：
  1. channel_A <- notif → 立即成功（A 正在快速消费）
  2. channel_B <- notif → **阻塞**！（B 还在处理上一条消息的字段解析）
     ... B 的 channel 满了，整个循环停滞
  3. channel_C 的写入永远不会发生
```

**全系统影响**：一个慢消费者会阻塞所有后续消费者的通知送达，且阻塞 `notificationLock` 的释放，导致新订阅/注销操作也被阻塞。

**当前代码中未显式设置 channel buffer**。查看 subscription resolver：

```go
// api/graphql/resolvers/notification.go:18-34
func (r *subscriptionResolver) Notification(ctx context.Context) (<-chan *models.Notification, error) {
    user := auth.UserFromContext(ctx)
    
    notificationChannel := make(chan *models.Notification)  // ★ 无缓冲 channel!
    
    listenerID := notification.RegisterListener(user, notificationChannel)
    
    go func() {
        <-ctx.Done()  // 连接断开时
        notification.DeregisterListener(listenerID)
        close(notificationChannel)
    }()
    
    return notificationChannel, nil
}
```

**代码位置**：`api/graphql/resolvers/notification.go:18-34`

### 11.6 Batch 内去重 vs 跨 Batch 去重

**同一连接内，不同 Subscription 事件触发的 Load 调用可以发生两种层面的去重**：

| 去重层面 | 发生时机 | 机制 | 效果 |
|---------|---------|------|-----|
| Batch 内去重 | 5ms 窗口内 | keyIndex 线性扫描 `b.keys` | 相同 key 只在 batch 中存一份 |
| 跨 Batch 去重 | 跨时间窗口 | l.cache 命中 | 相同 key 直接返回缓存，不进新 batch |

**跨 Batch 缓存命中流程**：
```
时间窗口 #1 (T0~T5):
  Load(100) → 未命中缓存 → batch1: [100] → SQL → 写入 cache[100] = url_A

时间窗口 #2 (T20~T25):
  Load(100) → 检查 cache[100] → 命中！
    ├─ 不创建 batch
    ├─ 不加入 keys
    └─ 直接返回 url_A（无需 SQL）
```

**长连接场景下的缓存优势**：
- 频繁更新的同一媒体（例如重新生成缩略图通知），在长连接中**只需查询一次**
- HTTP 请求模式下每次请求都要重新查询
- 这是长连接 DataLoader 缓存的**唯一正面效果**（需权衡内存与过期问题）

### 11.7 并发安全性分析

多个 goroutine 同时操作同一 DataLoader 时的锁互斥分析：

```go
// LoadThunk 中所有共享状态操作都在 mu.Lock 保护下：
func (l *MediaURLLoader) LoadThunk(key int) func() (*models.MediaURL, error) {
    l.mu.Lock()    // ★ 加锁
    defer l.mu.Unlock()
    
    if it, ok := l.cache[key]; ok { ... }      // 读 cache
    if l.batch == nil { l.batch = ... }        // 写 batch 指针
    pos := l.batch.keyIndex(l, key)            // 写 batch.keys
    ...
    return func() { ... }
}
```

**关键保证**：
1. **Batch 创建原子性**：`l.batch == nil` 检查 + 创建赋值在锁内完成，不会创建两个 batch
2. **Cache 读写安全**：所有读写都在锁内，不会出现并发 map 读写 panic
3. **Batch.keys 安全**：keyIndex 只在锁内被调用，切片 append 是安全的
4. **断开逻辑安全**：`l.batch = nil` + `b.closing = true` 在锁内完成，定时器到时的 startTimer 也会加锁检查

**唯一非原子点**：`go b.end(l)` 是异步执行的，`end()` 调用 `l.fetch(b.keys)` 时不在锁内。但这是安全的，因为：
- `b.keys` 切片在 `closing=true` 后不会再被修改（新 Load 创建新 batch）
- 没有其他 goroutine 会读写 `b.data` 和 `b.error`
- fetch 函数本身使用独立的 DB 连接（由 GORM 连接池管理）

---

## 十二、改进建议

### 12.1 解决 Album.thumbnail 的 N+1 问题（最高优先级）

详见第九章「AlbumThumbnailLoader 批量获取相册封面的代码实现路径」的完整实现方案。

**预期收益**：相册列表查询次数从 N+3 降低到约 4-5 次，延迟降低 90% 以上。

### 12.2 解决 UserFavoritesLoader 的缓存问题

**问题**：使用指针作为 cache key，导致相同内容无法命中缓存，batch 去重也失效。

**方案 A**：使用复合字符串 key：
```go
// 改造为 string key: fmt.Sprintf("%d:%d", userID, mediaID)
// cache map[string]bool
```

**方案 B**：自定义 hashCode 或实现 `comparable` 接口（Go 1.21+）。

### 12.3 扩展更多 DataLoader

为高频查询字段添加 DataLoader：

| 新增 Loader          | 对应字段         | Key 类型     |
| -------------------- | ---------------- | ------------ |
| `MediaExifLoader`    | `Media.exif`     | media ID     |
| `MediaFacesLoader`   | `Media.faces`    | media ID     |
| `AlbumByIDLoader`    | `Media.album`    | album ID     |
| `SubAlbumsLoader`    | `Album.subAlbums`| parent album ID |

### 12.4 配置化 DataLoader 参数（按场景优化）

当前所有 Loader 硬编码 `maxBatch=100, wait=5ms`，建议改为可配置，见第十章 10.6 节的分场景建议表。

### 12.5 为 Notification channel 增加 buffer

解决慢消费者阻塞全局广播问题：

```go
// 修改前：无缓冲
notificationChannel := make(chan *models.Notification)

// 修改后：带缓冲，避免阻塞 Broadcast
notificationChannel := make(chan *models.Notification, 100)
```

同时建议 Broadcast 使用 select + default 做非阻塞写入，避免慢消费者影响其他人。

### 12.6 跨连接查询结果缓存（进阶）

多连接查询同一 mediaID 时产生重复 SQL（第十一章 11.3.2 节），可引入**进程级共享查询结果缓存**：
- 使用带 TTL 的全局 LRU Cache（如 `hashicorp/golang-lru`）
- 在 fetch 函数执行前先查全局缓存，命中则直接返回
- 在 DataLoader 之外独立维护，不影响请求级 DataLoader 隔离语义

---

## 十三、关键代码位置索引

| 功能                | 文件路径                                                    | 行号   |
| ------------------- | ----------------------------------------------------------- | ------ |
| 中间件注册          | `api/server.go`                                             | 75     |
| Loaders 结构体定义  | `api/dataloader/loaders.go`                                 | 15-21  |
| 中间件注入实现      | `api/dataloader/loaders.go`                                 | 23-40  |
| MediaURLLoader 生成 | `api/dataloader/gen_mediaurlloader.go`                      | -      |
| LoadThunk 核心实现  | `api/dataloader/gen_mediaurlloader.go`                      | 73-112 |
| keyIndex 批量收集   | `api/dataloader/gen_mediaurlloader.go`                      | 181-203|
| startTimer 定时器   | `api/dataloader/gen_mediaurlloader.go`                      | 205-219|
| end 批量获取        | `api/dataloader/gen_mediaurlloader.go`                      | 221-224|
| MediaURL fetch 实现 | `api/dataloader/mediaURLLoader.go`                          | 13-82  |
| UserFavorite fetch  | `api/dataloader/userFavoriteLoader.go`                      | 10-60  |
| UserLoader fetch    | `api/dataloader/userLoader.go`                              | 10-71  |
| UserFavorites 缓存  | `api/dataloader/gen_userfavoritesloader.go`                 | 47     |
| 相册列表 resolver   | `api/graphql/resolvers/album.go`                            | 119-126|
| 相册 thumbnail 解析 | `api/graphql/resolvers/album.go`                            | 72-75  |
| 相册媒体分页查询    | `api/graphql/resolvers/album.go`                            | 21-51  |
| 媒体 thumbnail 解析 | `api/graphql/resolvers/media.go`                            | 22-25  |
| 媒体 favorite 解析  | `api/graphql/resolvers/media.go`                            | 69-80  |
| Album.Thumbnail()   | `api/graphql/models/album.go`                               | 83-115 |
| MyAlbums action     | `api/graphql/models/actions/album_actions.go`               | 9-48   |
| User.FillAlbums()   | `api/graphql/models/user.go`                                | 154-165|
| 分页 FormatSQL      | `api/graphql/models/utils.go`                               | 11-40  |
| Pagination 模型     | `api/graphql/models/generated.go`                           | 63-68  |
| 前端相册列表查询    | `ui/src/Pages/AllAlbumsPage/AlbumsPage.tsx`                 | 11-28  |
| WebSocket 传输配置  | `api/graphql/endpoint/graphql_endpoint.go`                  | 31-35  |
| WebSocket 认证      | `api/graphql/auth/auth.go`                                  | 92-130 |
| Notification 订阅   | `api/graphql/resolvers/notification.go`                     | 17-34  |
| Notification 广播   | `api/graphql/notification/Notification.go`                  | 70-83  |
| 监听器全局数组      | `api/graphql/notification/Notification.go`                  | 28-30  |
| myAlbums 生成代码   | `api/graphql/generated.go`                                  | 6219-6251 |
| subscription 生成代码| `api/graphql/generated.go`                                  | 7567-7585 |
| childFields_Album   | `api/graphql/generated.go`                                  | 1642   |
| childFields_Media   | `api/graphql/generated.go`                                  | 1732   |
