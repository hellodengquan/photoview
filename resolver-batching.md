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

### 10.1 前端分页策略与请求结构

PhotoView 的相册详情页采用**滚动分页加载**策略，通过 `useScrollPagination` hook 实现：

```tsx
// ui/src/Pages/AlbumPage/AlbumPage.tsx:50-62
const { loading, error, data, refetch, fetchMore } = useQuery<...>(ALBUM_QUERY, {
    variables: {
        id: albumId,
        offset: 0,
        limit: 200,    // ← 每页 200 个媒体
        // ...
    },
})

const { containerElem, finished: finishedLoadingMore } =
    useScrollPagination<albumQuery>({
        loading,
        fetchMore,
        data,
        getItems: data => data.album.media,
    })
```

**代码位置**：`ui/src/Pages/AlbumPage/AlbumPage.tsx:50-70`

**分页查询参数**：
- 初始页：`offset=0, limit=200`
- 第 2 页：`offset=200, limit=200`
- 第 N 页：`offset=200*(N-1), limit=200`

每页媒体返回后，前端请求的 Media Gallery fragment 包含以下需要 DataLoader 的字段：

```graphql
# ui/src/components/photoGallery/MediaGallery.tsx:37-55
fragment MediaGalleryFields on Media {
    id
    thumbnail { url, width, height }   # ← MediaThumbnail.Load(mediaID)
    highRes { url }                     # ← MediaHighres.Load(mediaID)
    videoWeb { url }                    # ← MediaVideoWeb.Load(mediaID)
    favorite                            # ← UserMediaFavorite.Load(&UserMediaData{...})
}
```

**每页 200 个媒体，每个媒体触发 4 次 DataLoader 调用**：
- MediaThumbnail：200 次调用
- MediaHighres：200 次调用
- MediaVideoWeb：200 次调用
- UserMediaFavorite：200 次调用
- **总计：每页 800 次 Load 调用**

### 10.2 maxBatch 参数的触发机制

所有 DataLoader 的 `maxBatch` 统一设为 **100**：

```go
// api/dataloader/mediaURLLoader.go:50
NewThumbnailMediaURLLoader: &MediaURLLoader{
    maxBatch: 100,  // ← 最大批量
    wait:     5 * time.Millisecond,
    fetch:    makeMediaURLLoader(...),
}

// api/dataloader/userLoader.go:13
NewUserLoaderByToken: &UserLoader{
    maxBatch: 100,
    wait:     5 * time.Millisecond,
    // ...
}

// api/dataloader/userFavoriteLoader.go:13
NewUserFavoriteLoader: &UserFavoritesLoader{
    maxBatch: 100,
    wait:     5 * time.Millisecond,
    // ...
}
```

**maxBatch 触发代码**：

```go
// api/dataloader/gen_mediaurlloader.go:181-203
func (b *mediaURLLoaderBatch) keyIndex(l *MediaURLLoader, key int) int {
    // ...去重检查...

    pos := len(b.keys)
    b.keys = append(b.keys, key)
    if pos == 0 {
        go b.startTimer(l)  // 第一个 key 启动定时器
    }

    // ⚠️ 达到批量上限时立即触发
    if l.maxBatch != 0 && pos >= l.maxBatch-1 {
        if !b.closing {
            b.closing = true
            l.batch = nil    // 断开当前 batch，后续 key 创建新 batch
            go b.end(l)      // 异步执行 fetch，不等待 5ms
        }
    }

    return pos
}
```

**代码位置**：`api/dataloader/gen_mediaurlloader.go:194-199`

**触发条件详解**：
- 当 `pos == 99`（即第 100 个 key 加入）时，`pos >= l.maxBatch-1` → `99 >= 99` → true
- 立即**关闭当前 batch**，不等 5ms 定时器
- 将 `l.batch = nil`，后续新的 Load 调用会创建**新的 batch**
- 启动 goroutine 执行当前 batch 的 fetch

### 10.3 200 个媒体分页的完整分批过程

以 `MediaThumbnail` Loader 为例，200 个媒体的分批过程：

**场景**：gqlgen 并行解析 200 个 Media.thumbnail resolver，在极短时间（<1ms）内调用 200 次 `Load(key)`

```
时间 T0+0μs：
  Load(1) → pos=0 → 启动 5ms 定时器，batch.keys=[1]
  Load(2) → pos=1 → batch.keys=[1, 2]
  ...
  Load(99) → pos=98 → batch.keys=[1..99]
  Load(100) → pos=99 → ⚠️ 触发 maxBatch！
      ├─ b.closing = true
      ├─ l.batch = nil  ← 断开连接
      └─ go b.end(l)    ← 立即执行 batch#1
          └─ fetch(keys=[1..100]) → 1 次 SQL: WHERE media_id IN (1..100)

时间 T0+~100μs：
  Load(101) → l.batch == nil → 创建 batch#2
      └─ pos=0 → 启动新的 5ms 定时器，batch.keys=[101]
  Load(102) → pos=1 → batch.keys=[101, 102]
  ...
  Load(199) → pos=98 → batch.keys=[101..199]
  Load(200) → pos=99 → ⚠️ 触发 maxBatch！
      ├─ b.closing = true
      ├─ l.batch = nil
      └─ go b.end(l) ← 立即执行 batch#2
          └─ fetch(keys=[101..200]) → 1 次 SQL: WHERE media_id IN (101..200)
```

**4 个 Loader 的总 SQL 查询统计**：

| Loader | 批量 1 (1-100) | 批量 2 (101-200) | 总查询数 |
|--------|---------------|-----------------|---------|
| MediaThumbnail | 1 SQL | 1 SQL | 2 |
| MediaHighres | 1 SQL | 1 SQL | 2 |
| MediaVideoWeb | 1 SQL | 1 SQL | 2 |
| UserMediaFavorite | 1 SQL | 1 SQL | 2 |
| **总计** | **4 SQL** | **4 SQL** | **8 SQL** |

**对比**：如果没有 DataLoader（N+1 场景），每页需要 800 次 SQL。使用 DataLoader 后降为 8 次，减少 **99%** 的查询次数。

### 10.4 降级机制：key 分批到达的场景

在真实网络环境中，gqlgen 的并行 resolver 启动并非完全同时，可能存在微秒级时间差。以下是 key 分批到达场景下的行为：

#### 场景 A：慢速分页（key 间隔 > 5ms）

```
T0: Load(1..50) → batch#1 创建，启动 5ms 定时器
T0+5ms: 定时器触发 → fetch([1..50]) → SQL #1
T0+6ms: Load(51..100) → batch#2 创建，启动定时器
T0+11ms: 定时器触发 → fetch([51..100]) → SQL #2
T0+12ms: Load(101..150) → batch#3 ...
T0+17ms: 触发 → SQL #3
T0+18ms: Load(151..200) → batch#4 ...
T0+23ms: 触发 → SQL #4
```

**结果**：4 个 Loader 各 4 批次，共 **16 次 SQL**（多于同时到达的 8 次）。

#### 场景 B：边界条件——第 100 个 key 到达时定时器已触发

存在一个**竞态窗口**：如果 5ms 定时器即将触发（第 199 个 key 还未到达），`startTimer` 正在执行 `l.mu.Lock()` 之前的间隙：

```go
// api/dataloader/gen_mediaurlloader.go:205-219
func (b *mediaURLLoaderBatch) startTimer(l *MediaURLLoader) {
    time.Sleep(l.wait)      // T0+5ms: 睡眠结束
    l.mu.Lock()             // ← 等待锁（可能被 keyIndex 持有）

    // 检查是否已因 maxBatch 关闭
    if b.closing {
        l.mu.Unlock()
        return  // ← 如果 maxBatch 先触发，定时器这里不重复执行
    }

    l.batch = nil
    l.mu.Unlock()
    b.end(l)
}
```

**保护机制**：`b.closing` 标志位防止 `end()` 被重复调用：
- 如果 `keyIndex` 先触发 maxBatch → `b.closing = true`，定时器获取锁后发现并退出
- 如果定时器先获取锁 → 断开 batch 并执行 `end()`，`keyIndex` 中达到 maxBatch 的后续 key 会检查 `b.closing` 并跳过重复执行

#### 场景 C：去重命中导致不足 100 个唯一 key

如果分页中包含重复 media（如同一媒体被多次引用）：

```
Load(1), Load(2), Load(1), Load(3), Load(2), ..., Load(100)
  ↓ 去重后实际 batch.keys = [1, 2, 3, ..., 50]  (只有 50 个唯一 key)
  ↓ pos 永远达不到 99
  ↓ 等待 5ms 定时器触发
  ↓ fetch([1..50]) → 1 次 SQL
```

**但 UserFavoritesLoader 的指针问题导致去重失效**：即使 (UserID, MediaID) 相同，不同指针无法去重，batch 中会重复填充 key。

### 10.5 超大相册场景的性能分析

假设一个相册有 **10,000 个媒体**，用户滚动加载所有分页：

```
第 1 页 (0-199): 4 Loader × 2 batch = 8 SQL
第 2 页 (200-399): 4 Loader × 2 batch = 8 SQL
...
第 50 页 (9800-9999): 4 Loader × 2 batch = 8 SQL
```

**总 SQL 查询数**：50 页 × 8 = **400 次 SQL**

**没有 DataLoader 的情况下**：10,000 媒体 × 4 字段 = **40,000 次 SQL**

**但在 WebSocket 长连接场景下有额外影响**：
- 每一页的查询结果都会写入 DataLoader 缓存
- 50 页 × 200 媒体 × 4 Loader = 约 **40,000 个缓存条目**
- 缓存内存占用约：40,000 × (~100 字节) ≈ **4MB**（可接受，但会持续累积到连接断开）

### 10.6 降级与溢出处理建议

当前代码在以下边界条件下缺乏明确处理：

| 边界场景 | 当前行为 | 潜在问题 | 建议方案 |
|---------|---------|---------|---------|
| `IN` 子句参数过多（如 500 个 key） | 直接发送 SQL | PostgreSQL 参数限制，或查询性能下降 | 在 fetch 内部再分片（如每 300 个 key 执行 1 次 UNION ALL） |
| 单请求内 Loader 调用过万次 | 不断创建新 batch | 瞬时 SQL 并发过高 | 增加信号量限制 fetch 并发 |
| fetch SQL 执行时间远超 5ms | 后续 batch 继续累积 | 数据库连接池耗尽 | 自适应 wait 时间（根据上次 fetch 耗时动态调整） |
| 同一 key 快速反复 Prime/Load | 缓存替换 | 高频 key 抖动 | LRU 缓存替代 map |

**fetch 内部分片参考实现**：

```go
func makeMediaURLLoader(db *gorm.DB, filter func(query *gorm.DB) *gorm.DB) func(keys []int) ([]*models.MediaURL, []error) {
    return func(mediaIDs []int) ([]*models.MediaURL, []error) {
        // ⬇️ 新增：IN 子句分片（每 300 个 key 一片）
        const chunkSize = 300
        var allUrls []*models.MediaURL

        for start := 0; start < len(mediaIDs); start += chunkSize {
            end := start + chunkSize
            if end > len(mediaIDs) {
                end = len(mediaIDs)
            }
            chunk := mediaIDs[start:end]

            var urls []*models.MediaURL
            query := db.Where("media_id IN (?)", chunk)
            query = filter(query)
            if err := query.Find(&urls).Error; err != nil {
                return nil, []error{err}
            }
            allUrls = append(allUrls, urls...)
        }

        // 后续 resultMap 组装不变...
    }
}
```

---

## 十一、并发 Subscription 同时订阅同一字段时的批次合并行为

### 11.1 并发场景分类

需要区分三个层级的「并发订阅」，每个层级的 DataLoader 共享行为不同：

| 层级 | 并发类型 | DataLoader 实例 | 批次合并可能性 |
|------|---------|----------------|--------------|
| L1 | **同一连接内**的多个 subscription 操作 | ✅ **同一个实例** | ✅ 可以合并 |
| L2 | **同一连接内**的多次推送事件并行解析 | ✅ **同一个实例** | ✅ 条件性合并（5ms 窗口内） |
| L3 | **不同连接**之间的 subscription | ❌ **不同实例** | ❌ 无法合并 |

### 11.2 L1：同一连接内多 Subscription 操作的共享

gqlgen 的 WebSocket 处理模型中，**单个 WebSocket 连接可以承载多个并发的 subscription 操作**（符合 GraphQL over WebSocket Protocol）。

**代码路径**：

```
客户端连接到 /graphql (WebSocket 升级)
    │
    ├─ transport.Websocket 持有原始 context（含 Loaders）
    │
    ├─ 收到 "start" #1 (id=1, operation=notification subscription)
    │   └─ 使用连接级 context → ec._Subscription_notification(ctx, ...)
    │       └─ ctx 包含 loadersKey → 同一个 Loaders 对象
    │
    ├─ 收到 "start" #2 (id=2, operation=假设的 mediaCreated subscription)
    │   └─ 使用同一个连接级 context → ec._Subscription_mediaCreated(ctx, ...)
    │       └─ ctx 包含 loadersKey → 同一个 Loaders 对象 ⬅️ 共享！
    │
    └─ ...可以同时活跃 N 个 subscription
```

**如果两个 subscription 的推送事件几乎同时到达（5ms 窗口内），会触发 DataLoader 批量合并**。

**具体示例**（假设有 `mediaCreated` 和 `notification` 两个 subscription，且都解析嵌套的 Media.thumbnail 字段）：

```
T0: 连接建立 → Loaders{MediaThumbnail: {cache: nil, batch: nil}}

T1: subscription#1 推送事件 A → 解析 Media.thumbnail → Load(MediaID=100)
    └─ batch 创建，batch.keys=[100]，启动 5ms 定时器

T1+1ms: subscription#2 推送事件 B → 解析 Media.thumbnail → Load(MediaID=100)
    └─ 检查 batch.keys → 已存在 100 → 返回 pos=0（去重命中，复用！）

T1+1ms: subscription#2 推送事件 B → 解析另一个 Media.thumbnail → Load(MediaID=200)
    └─ batch.keys=[100, 200], pos=1

T1+5ms: 定时器触发 → fetch([100, 200]) → 1 次 SQL
    └─ 事件 A 获得 MediaID=100 的结果
    └─ 事件 B 获得 MediaID=100 和 200 的结果（100 是同一个 batch.data 的同一位置）
```

**关键收益**：两个并发 subscription 共享同一个 batch，减少了 SQL 查询次数。

### 11.3 L2：同一 Subscription 多次推送事件的并发合并

以 `notification` subscription 为例，当通知事件密集发生时：

```
T0: notification#1 推送 → 解析字段（如果嵌套复杂字段）
T0+1ms: notification#2 推送 → 解析字段
T0+2ms: notification#3 推送 → 解析字段
```

**gqlgen 的推送解析是串行还是并发？** 需要看 gqlgen 的 `ResolveFieldStream` 实现：

```go
// gqlgen 的 ResolveFieldStream 伪代码
func ResolveFieldStream(ctx, ..., resolver func() (<-chan T, ...)) {
    ch, _ := resolver(ctx, field)  // 调用用户的 subscription resolver
    // ...
    go func() {
        for v := range ch {
            // 对 channel 中的每个值：
            // 调用 marshalNNotification...(ctx, selections, v)
            // 这个调用包含字段解析，如果解析过程中有 goroutine 并行执行...
            sendOverWebsocket()
        }
    }()
}
```

**如果 Marshal 中有并行字段解析**（嵌套类型有多个 resolver），则会出现以下行为：

```
notification#1 推送解析开始:
    ├─ 并行 resolver A → MediaThumbnail.Load(100) → 创建 batch，keys=[100]
    └─ 并行 resolver B → MediaThumbnail.Load(200) → keys=[100, 200]

（+1ms 后）notification#2 推送解析开始:
    ├─ 并行 resolver C → MediaThumbnail.Load(100) → batch 内去重命中 pos=0
    └─ 并行 resolver D → MediaThumbnail.Load(300) → keys=[100, 200, 300]

（5ms 后）定时器触发:
    └─ fetch([100, 200, 300]) → 1 次 SQL
        └─ notification#1: 拿到 100、200 的结果
        └─ notification#2: 拿到 100、300 的结果
```

**但当前 Notification 类型没有复杂嵌套字段**，所以这种合并在当前代码中不生效。如果未来 Notification 增加嵌套的 Media/Album 字段，就会触发此机制。

### 11.4 L3：不同连接之间的隔离性

**不同 WebSocket 连接（不同客户端）拥有完全独立的 DataLoader 实例**，批次之间无法合并。

原因：每个 HTTP 请求（WebSocket 升级也是一次 HTTP 请求）经过中间件时创建独立的 Loaders：

```go
// api/dataloader/loaders.go:23-40
func Middleware(db *gorm.DB) mux.MiddlewareFunc {
    return func(next http.Handler) http.Handler {
        return func(w http.ResponseWriter, r *http.Request) {
            // ✅ 每次 HTTP 请求创建全新的 Loaders 实例
            ctx := context.WithValue(r.Context(), loadersKey, &Loaders{
                MediaThumbnail:      NewThumbnailMediaURLLoader(db),
                MediaHighres:        NewHighresMediaURLLoader(db),
                MediaVideoWeb:       NewVideoWebMediaURLLoader(db),
                UserFromAccessToken: NewUserLoaderByToken(db),
                UserMediaFavorite:   NewUserFavoriteLoader(db),
            })
            r = r.WithContext(ctx)
            next.ServeHTTP(w, r)
        }
    }
}
```

**连接 1** 和 **连接 2** 的 batch 无法互相看见：
```
客户端 A (WebSocket#1)      客户端 B (WebSocket#2)
    │                            │
    Load(100) ─────┐             Load(100) ─────┐
                   ▼                            ▼
           batch#1.keys=[100]           batch#2.keys=[100]
           (独立的 Loader 实例)          (独立的 Loader 实例)
                   │                            │
                   ▼                            ▼
           fetch([100]) SQL #1           fetch([100]) SQL #2
           (5ms 后触发)                  (5ms 后触发)
```

**1000 个客户端同时查询同一个 media**：依然执行 **1000 次相同的 SQL**，没有跨连接合并。

### 11.5 锁机制与并发安全分析

DataLoader 的并发安全依赖 `sync.Mutex`，在两个关键位置加锁：

**位置 1：LoadThunk 中访问 batch 和缓存**

```go
// api/dataloader/gen_mediaurlloader.go:73-94
func (l *MediaURLLoader) LoadThunk(key int, initialValue func() (*models.MediaURL, error)) func() (*models.MediaURL, error) {
    l.mu.Lock()  // ⬅️ 加锁 1
    
    if l.cache != nil {
        // 检查缓存
    }
    
    if l.batch == nil {
        l.batch = &mediaURLLoaderBatch{done: make(chan struct{})}
    }
    
    index := l.batch.keyIndex(l, key)  // keyIndex 在锁内执行
    
    l.mu.Unlock() // ⬅️ 解锁
    
    // 返回 thunk（锁已释放，不阻塞等待 batch 完成）
    return func() (*models.MediaURL, error) {
        <-l.batch.done  // 阻塞等待 batch（无锁）
        // ...处理结果...
    }
}
```

**位置 2：startTimer 中修改 batch**

```go
// api/dataloader/gen_mediaurlloader.go:205-219
func (b *mediaURLLoaderBatch) startTimer(l *MediaURLLoader) {
    time.Sleep(l.wait)
    l.mu.Lock()  // ⬅️ 加锁 2（与 LoadThunk 互斥）
    
    if b.closing {
        l.mu.Unlock()
        return
    }
    
    l.batch = nil
    l.mu.Unlock()
    
    b.end(l)  // end() 不需要锁，因为 batch 已断开连接
}
```

**并发场景下的锁获取时序**：

```
Goroutine A (Load 100)     Goroutine B (Load 200)     Timer Goroutine
       │                        │                        │
       ├─ mu.Lock() ✅           ├─ mu.Lock() ❌           │
       ├─ 创建 batch             │ (等待锁)                 │
       ├─ keyIndex → pos=0       │                        │
       │   └─ go startTimer ──────────────────────────────┤ sleep(5ms)
       ├─ mu.Unlock()            ├─ mu.Lock() ✅           │
       │                        ├─ keyIndex → pos=1       │
       │                        ├─ mu.Unlock()            │
       └─ thunk 执行             └─ thunk 执行             │
       :                          :                         ├─ sleep 结束
       <-batch.done-              <-batch.done-             ├─ mu.Lock() ✅
                                                             ├─ l.batch = nil
                                                             ├─ mu.Unlock()
                                                             └─ b.end(l) → fetch SQL
```

### 11.6 高并发场景的潜在死锁与性能瓶颈

#### 11.6.1 潜在性能瓶颈：thunk 执行时的缓存写入竞争

多个 thunk 函数同时完成等待后，会竞争写缓存：

```go
// thunk 函数（N 个 goroutine 同时执行这段）
return func() (*models.MediaURL, error) {
    <-b.done  // 同时解除等待
    
    if err == nil {
        l.mu.Lock()      // N 个 goroutine 竞争同一把锁
        l.unsafeSet(key, data)
        l.mu.Unlock()
    }
}
```

100 个 key 的 batch 会触发 100 次锁竞争。单次写操作很快（纳秒级），影响不大，但在超大规模 batch（如 1000）下可能造成微秒级延迟。

#### 11.6.2 fetch 阻塞期间的新 key 处理

当 batch 的 `fetch` 正在执行（比如慢查询 100ms）时，新的 Load 调用：

```
batch#1 end() 开始 → fetch(SQL, 耗时 100ms)
    │
    ├─ +10ms: Load(新 key) → l.batch == nil → 创建 batch#2 → 正常处理 ✅
    ├─ +20ms: Load(新 key) → batch#2.keys 追加
    └─ +105ms: batch#1.fetch 返回 → close(done) → thunk 解锁
```

**行为正确**：旧 batch 的 fetch 不阻塞新 key，新 key 直接进入新 batch。

#### 11.6.3 startTimer 与 keyIndex 的竞态保护

```
时间线：
T0: 第 99 个 key 加入（pos=98）→ 尚未触发 maxBatch
T0+5ms: startTimer sleep 结束，准备获取锁
T0+5ms+1μs: 第 100 个 key 加入 → keyIndex 获取锁 → pos=99 → 触发 maxBatch
             ├─ b.closing = true
             ├─ l.batch = nil
             ├─ go end()
             └─ 释放锁
T0+5ms+2μs: startTimer 获取锁 → 检查 b.closing → true → 解锁并退出 ✅
```

**结果**：`end()` 仅被 keyIndex 触发一次，startTimer 不重复执行，避免了重复 SQL。

### 11.7 并发订阅场景的优化建议

1. **跨连接查询结果缓存（可选）**：
   ```go
   // 使用进程级 LRU 缓存（如 hashicorp/golang-lru）
   var globalMediaURLCache *lru.Cache  // 20MB 容量，带 TTL
   
   // 在 fetch 中先查全局缓存，未命中再查数据库
   func makeMediaURLLoader(db *gorm.DB, filter ...) func([]int)([]*MediaURL, []error) {
       return func(mediaIDs []int) ([]*models.MediaURL, []error) {
           // 步骤1: 先从全局缓存取
           // 步骤2: 未命中的 key 才查数据库
           // 步骤3: 结果写回全局缓存
       }
   }
   ```
   收益：跨连接复用相同查询结果，L3 场景也能减少 SQL。

2. **推送解析并发控制**：密集推送场景下，限制同一连接内并行解析的事件数。

3. **连接级 Loader 定期重置**：结合第八章的建议，长连接下定期清空缓存，避免无限增长。

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
| WebSocket 传输配置  | `api/graphql/endpoint/graphql_endpoint.go`                  | 31-35  |
| WebSocket 认证      | `api/graphql/auth/auth.go`                                  | 92-130 |
| Notification 订阅   | `api/graphql/resolvers/notification.go`                     | 17-34  |
| 订阅生成代码        | `api/graphql/generated.go`                                  | 7567-7585 |
| maxBatch 触发逻辑   | `api/dataloader/gen_mediaurlloader.go`                      | 194-199 |
| startTimer 竞态保护 | `api/dataloader/gen_mediaurlloader.go`                      | 205-219 |
| FormatSQL 分页处理  | `api/graphql/models/utils.go`                               | 11-40  |
| Pagination 结构体   | `api/graphql/models/generated.go`                           | 63-68  |
| 前端分页查询配置    | `ui/src/Pages/AlbumPage/AlbumPage.tsx`                      | 50-70  |
| Media Gallery 字段  | `ui/src/components/photoGallery/MediaGallery.tsx`           | 37-55  |
