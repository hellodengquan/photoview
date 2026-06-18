# Photoview GraphQL 分页链路分析

## 概述

Photoview 采用 **Offset-based 分页**方案（**非** Cursor-based/Relay 风格），通过 `Pagination` + `Ordering` 输入类型配合 gqlgen 的字段选择（Field Collection）机制，实现前后端联动的分页查询。整个链路分为三层协作：

1. **Schema 层**：定义分页/排序的输入类型（`Pagination`、`Ordering`）
2. **游标/排序稳定性层**：通过确定性 `Ordering` + Offset 组合实现类游标稳定性（**不使用真正的 Cursor 编码**）
3. **字段裁剪层**：gqlgen 的 `CollectFields` 在序列化阶段按需裁剪输出字段

---

## 一、Schema 层：分页类型定义

### 1.1 核心输入类型

位置：`api/graphql/resolvers/root.graphql:15-28`

```graphql
"Used to specify pagination on a list of items"
input Pagination {
  limit: Int    # 最大获取条数
  offset: Int   # 跳过的条数（由 Ordering 决定顺序）
}

"Used to specify how to sort items"
input Ordering {
  order_by: String          # 数据库列名，直接映射 SQL ORDER BY 列
  order_direction: OrderDirection  # ASC / DESC
}

enum OrderDirection {
  ASC   # Sort accending A-Z
  DESC  # Sort decending Z-A
}
```

**设计特点**：
- **非 Relay 标准**：不使用 `Connection<T>` / `Edge<T>` / `PageInfo` / `Cursor` 那套规范
- 没有 `totalCount`、`hasNextPage`、`hasPreviousPage`、`startCursor`、`endCursor` 等字段
- 返回值直接是 `[Album!]!` / `[Media!]!` 这种简单数组，不是包装类型
- `order_by` 是 `String` 而不是 `Enum`，意味着前端可以传任意列名（由 FormatSQL 内部透传给 GORM `clause.Column`，存在一定 SQL 注入风险但实际被 clause 内部转义）

### 1.2 支持分页的查询字段汇总

| 查询字段 | Schema 文件 | 分页参数 | 排序参数 | 特有过滤参数 |
|---------|-----------|---------|---------|------------|
| `Query.myAlbums` | `album.graphql:36` | `paginate` | `order` | `onlyRoot`, `showEmpty`, `onlyWithFavorites` |
| `Query.myMedia` | `media.graphql:108` | `paginate` | `order` | — |
| `Query.myTimeline` | `timeline.graphql:5` | `paginate` | **无**（内置多维排序） | `onlyFavorites`, `fromDate` |
| `Query.myFaceGroups` | `faces.graphql:32` | `paginate` | **无**（内置排序） | — |
| `Album.media` | `album.graphql:6` | `paginate` | `order` | `onlyFavorites` |
| `Album.subAlbums` | `album.graphql:14` | `paginate` | `order` | — |
| `FaceGroup.imageFaces` | `faces.graphql:14` | `paginate` | **无** | — |

---

## 二、游标（Cursor）稳定性分析

### 2.1 重要结论：项目**不使用真正的游标**

全局搜索 `cursor` / `Cursor` 关键词，在 `api/` 目录下 **0 匹配**。这意味着：

- ❌ 没有 `encodeCursor()` / `decodeCursor()` 编码/解码函数
- ❌ 没有 Base64 包装的游标值（如 Relay 风格的 `arrayconnection:123`）
- ❌ 没有基于主键 ID + 排序列的 Seek Method（Keyset Pagination）
- ✅ 取而代之的是 **Offset + 确定性排序** 组合方案

### 2.2 FormatSQL：分页 SQL 构建核心

位置：`api/graphql/models/utils.go:11-40`

```go
func FormatSQL(tx *gorm.DB, order *Ordering, paginate *Pagination) *gorm.DB {
    // 1. 应用 Limit / Offset
    if paginate != nil {
        if paginate.Limit != nil {
            tx.Limit(*paginate.Limit)   // → SQL: LIMIT N
        }
        if paginate.Offset != nil {
            tx.Offset(*paginate.Offset) // → SQL: OFFSET M
        }
    }

    // 2. 应用排序
    if order != nil && order.OrderBy != nil {
        desc := false
        if order.OrderDirection != nil && order.OrderDirection.IsValid() {
            if *order.OrderDirection == OrderDirectionDesc {
                desc = true
            }
        }
        tx.Order(clause.OrderByColumn{
            Column: clause.Column{
                Name: *order.OrderBy,  // 列名原样透传，由 clause 内部做安全处理
            },
            Desc: desc,
        })
    }

    return tx
}
```

**调用方式（所有 Resolver 都遵循同一模式）**：

以 `Album.Media` 为例（`api/graphql/resolvers/album.go:21-51`）：
```go
func (r *albumResolver) Media(ctx context.Context, obj *models.Album,
    order *models.Ordering, paginate *models.Pagination, onlyFavorites *bool) ([]*models.Media, error) {

    query := db.Where("media.album_id = ?", obj.ID).
        Where("media.id IN (?)", db.Model(&models.MediaURL{}).
            Select("media_urls.media_id").
            Where("media_urls.media_id = media.id"))

    // ...过滤逻辑...

    query = models.FormatSQL(query, order, paginate)  // ← 统一入口
    var media []*models.Media
    query.Find(&media)
    return media, nil
}
```

### 2.3 稳定性保证：排序键的确定性

Offset-based 分页的经典问题是**数据插入/删除导致重复或跳过**，本项目通过以下方式维持相对稳定：

| 场景 | 排序策略 | 稳定性来源 |
|-----|---------|----------|
| `myAlbums` | 前端指定 `order_by`（如 `title`、`created_at`） | 依赖前端选择合适排序列 |
| `myMedia` | 前端指定 `order_by` | 同上 |
| **`myTimeline`** | **内置多维排序**：年→月→日→album.title→date_shot（`timeline_actions.go:20-40`） | 5 级排序键 + `media.id` 作为隐式 tie-breaker，极高确定性 |
| `myFaceGroups` | `label NULL 优先` → `COUNT(image_faces) DESC`（`faces.go:403-404`） | 排序键稳定 |
| `Album.subAlbums` / `Album.media` | 前端指定 | 同上 |

**Timeline 的排序实现最典型**（`api/graphql/models/actions/timeline_actions.go:20-40`）：
```go
// PostgreSQL 版
query.Order("DATE_TRUNC('year', date_shot) DESC").
      Order("DATE_TRUNC('month', date_shot) DESC").
      Order("DATE_TRUNC('day', date_shot) DESC").
      Order("albums.title ASC").
      Order("media.date_shot DESC")
```

**缺陷与改进空间**：
- 没有显式在 `order_by` 列后追加 `id` 作为最终 tie-breaker（GORM 主键隐式排序行为因数据库而异）
- 当排序键值相同且行很多时，不同页可能出现边界不稳定

### 2.4 前端偏移量的计算方式

位置：`ui/src/hooks/useScrollPagination.ts:18-97`

使用 **IntersectionObserver** 实现无限滚动：

```typescript
// 当滚动到底部触发加载更多时
observer.current = new IntersectionObserver(entities => {
    if (entities.find(x => x.isIntersecting == false)) {
        const itemCount = data !== undefined ? getItems(data).length : 0
        fetchMore({
            variables: {
                offset: itemCount,  // ← 关键：用"当前已累计加载数量"作为 Offset
            }
        }).then(result => {
            const newItemCount = getItems(result.data).length
            if (newItemCount == 0) {
                setFinished(true)  // 加载到 0 条即视为到底
            }
        })
    }
}, options)
```

前端首次请求的典型参数（`ui/src/Pages/AlbumPage/AlbumPage.tsx:54-61`）：
```typescript
variables: {
    id: albumId,
    onlyFavorites,
    mediaOrderBy: orderParams.orderBy,
    orderDirection: orderParams.orderDirection,
    offset: 0,
    limit: 200,   // 每页固定 200 条
}
```

**前端"游标"等价物**：`offset = 已加载条目总数`。稳定性完全依赖 **limit 保持不变** + **排序在多页请求间一致**。

---

## 三、字段裁剪（Field Collection / Projection）

### 3.1 gqlgen 的字段收集机制

位置：`api/graphql/generated.go`（由 gqlgen 从 `.graphql` 文件自动生成）

每个对象类型有一个 `_Type(ctx, sel, obj)` 主方法，**第一步永远是字段收集**：

```go
// api/graphql/generated.go:9333 (Album 类型入口)
func (ec *executionContext) _Album(ctx context.Context, sel ast.SelectionSet, obj *models.Album) graphql.Marshaler {
    // 步骤 1: 只收集用户在 Query 中实际请求的字段 + Fragment
    fields := graphql.CollectFields(ec.OperationContext, sel, albumImplementors)

    // 步骤 2: 初始化输出 FieldSet（长度 = 实际请求字段数）
    out := graphql.NewFieldSet(fields)

    // 步骤 3: 对每个收集到的字段做 switch，只执行实际被请求的 resolver
    for i, field := range fields {
        switch field.Name {
        case "__typename":
            out.Values[i] = graphql.MarshalString("Album")
        case "id":
            out.Values[i] = ec._Album_id(ctx, field, obj)
        case "title":
            out.Values[i] = ec._Album_title(ctx, field, obj)
        case "media":
            // 嵌套/列表字段并发解析（Concurrently）
            out.Concurrently(i, func(ctx context.Context) graphql.Marshaler {
                return ec._Album_media(ctx, field, obj)
            })
        case "subAlbums":
            out.Concurrently(i, func(ctx context.Context) graphql.Marshaler {
                return ec._Album_subAlbums(ctx, field, obj)
            })
        // ...其他字段
        }
    }
    return out
}
```

**顶层 Query 入口同样的模式**（`api/graphql/generated.go:10818`）：
```go
func (ec *executionContext) _Query(ctx context.Context, sel ast.SelectionSet) graphql.Marshaler {
    fields := graphql.CollectFields(ec.OperationContext, sel, queryImplementors)
    // ... switch 分发 myAlbums / myMedia / myTimeline 等顶层查询
    for i, field := range fields {
        switch field.Name {
        case "myAlbums":
            out.Concurrently(i, ...)  // 并发解析顶层字段
        case "album":
            out.Concurrently(i, ...)
        // ...
        }
    }
}
```

### 3.2 字段裁剪的两个层次

| 层次 | 位置 | 机制 | 效果 |
|-----|------|------|------|
| **L1: 解析器调度层** | `generated.go` 中的 `_*` 方法 | `CollectFields` + `switch field.Name` | **只执行实际请求字段的 Resolver**，未请求字段的 Resolver 永不调用 |
| **L2: 序列化输出层** | `graphql.NewFieldSet` → `MarshalGQL` | `FieldSet` 数组按请求顺序存放值 | JSON 输出只包含请求字段，无多余列 |

**关键洞察 —— 没有做 SQL 级别的列裁剪**：

所有 Resolver 的查询都是 GORM 默认的 `SELECT *`：
```go
// albumResolver.Media 中：
query.Find(&media)  // → SELECT * FROM media WHERE ... LIMIT N OFFSET M
```

- GORM 会把 Album/Media 模型的**所有列**都从数据库读出来
- 但是**只有用户请求的字段**会在 gqlgen 序列化时被写入 JSON
- 对于关联数据（如 `Media.exif`、`Media.faces`、`FaceGroup.imageFaces`），通过 `gqlgen.yml` 中 `fields.*.resolver: true` 标记，实现**按需延迟加载**

### 3.3 gqlgen.yml 中标记为 Resolver 的字段（Lazy Load）

位置：`api/gqlgen.yml:24-66`

```yaml
models:
  User:
    model: .../models.User
    fields:
      albums:
        resolver: true   # 不直接从 User 结构体读，必须走 Resolver
  Media:
    model: .../models.Media
    fields:
      exif:
        resolver: true   # 懒加载 EXIF
      faces:
        resolver: true   # 懒加载人脸
      type:
        resolver: true   # 枚举值格式化
      album:
        resolver: true   # 关联查询
  MediaEXIF:
    model: .../models.MediaEXIF
    fields:
      dateShot:
        fieldName: DateShotWithOffset  # 自定义映射字段名
  FaceGroup:
    model: .../models.FaceGroup
    fields:
      imageFaces:
        resolver: true   # 懒加载 + 分页
```

**效果链路**：
1. 如果前端**没有请求** `Media.exif`，`_Media_exif` 永远不会被执行 → 不查 `media_exifs` 表
2. 如果前端**请求了** `Media.exif`，才会进入 `mediaResolver.Exif()`（`api/graphql/resolvers/media.go:56-67`）：
   ```go
   func (r *mediaResolver) Exif(ctx context.Context, obj *models.Media) (*models.MediaEXIF, error) {
       if obj.Exif != nil { return obj.Exif, nil }
       var exif models.MediaEXIF
       r.DB(ctx).Model(obj).Association("Exif").Find(&exif)  // ← 按需额外查一次
       return &exif, nil
   }
   ```
3. **这才是真正"字段裁剪影响后端行为"的地方** —— 通过 Resolver 调度，减少不必要的关联表查询

### 3.4 前端查询示例：字段裁剪的实际效果

**Timeline 查询**（`ui/src/components/timelineGallery/TimelineGallery.tsx:22-58`）：
```graphql
query myTimeline($limit: Int, $offset: Int, $fromDate: Time, $onlyFavorites: Boolean) {
  myTimeline(paginate: { limit: $limit, offset: $offset }, fromDate: $fromDate, onlyFavorites: $onlyFavorites) {
    id          # ✓ 请求：Media.ID 列
    title       # ✓ 请求：Media.Title 列
    type        # ✓ 请求：触发 Media.type Resolver
    blurhash    # ✓ 请求：Media.Blurhash 列
    thumbnail { url width height }  # ✓ 请求：触发 Media.thumbnail Resolver → Dataloader 批量查 media_urls
    highRes   { url width height }  # ✓ 请求：触发 Media.highRes Resolver
    videoWeb  { url }               # ✓ 请求：触发 Media.videoWeb Resolver
    favorite    # ✓ 请求：触发 Media.favorite Resolver → Dataloader 查 user_media_data
    album { id title }             # ✓ 请求：触发 Media.album Resolver → 查 albums
    date        # ✓ 请求：Media.DateShot
    # ✗ 未请求 exif → Media.exif Resolver 永远不执行，不查 media_exifs 表
    # ✗ 未请求 faces → Media.faces Resolver 永远不执行，不查 image_faces 表
    # ✗ 未请求 downloads / shares → 对应 Resolver 不执行
  }
}
```

---

## 四、三者配合：完整链路图

以 `AlbumPage` 请求 `album(id: 42) { media(paginate, order) { ... } }` 为例：

```
前端 (AlbumPage.tsx:50-70)
  │
  │  Apollo useQuery(ALBUM_QUERY, variables: {
  │    id: 42, offset: 0, limit: 200,
  │    mediaOrderBy: "date_shot", orderDirection: DESC
  │  })
  ▼
GraphQL Request POST /api/graphql
  {
    "query": "... album(id: $id) { media(paginate: $paginate, order: $order) { ... } }",
    "variables": { "id": 42, "limit": 200, "offset": 0, "mediaOrderBy": "date_shot", "orderDirection": "DESC" }
  }
  │
  ▼
gqlgen ExecutableSchema.Exec()  [generated.go:1513]
  │
  │  1. graphql.BuildUnmarshalerMap → 解析 Pagination/Ordering 输入
  │  2. ec._Query(ctx, Operation.SelectionSet)
  ▼
_Query()  [generated.go:10818]
  │
  │  graphql.CollectFields → 收集到: ["__typename", "album"]
  │  switch "album" → out.Concurrently(i, ec._Query_album)
  ▼
_Query_album()  [generated.go:6276]
  │
  │  1. directive @isAuthorized 鉴权
  │  2. ec.fieldContext_Query_album → 解包 args: { id: 42 }
  │  3. 调用 Resolver: ec.Resolvers.Query().Album(ctx, id=42, nil)
  ▼
resolver/album.go:129 (queryResolver.Album)
  │
  │  验证用户所有权 → actions.Album(db, user, id=42)
  │  返回 *models.Album 实例（所有列已从 DB 加载）
  ▼
回到 gqlgen 序列化: marshalOAlbum(ctx, selections, album)
  │
  ▼
_Album(ctx, selections { media }, obj=album)  [generated.go:9333]
  │
  │  graphql.CollectFields → 收集到: ["__typename", "media"]（还有 id, title, subAlbums 等取决于请求）
  │  switch "media" → out.Concurrently(i, ec._Album_media)
  ▼
_Album_media()  [generated.go:3014]
  │
  │  1. fieldContext_Album_media(ctx, field)
  │     → 解包 args: {
  │          order:    &Ordering{OrderBy:"date_shot", OrderDirection: DESC},
  │          paginate: &Pagination{Limit:200, Offset:0},
  │          onlyFavorites: nil
  │        }
  │  2. 调用 Resolver: Album().Media(ctx, obj=album, order, paginate, onlyFavorites)
  ▼
────────────── 三组件协作核心区 ──────────────
resolver/album.go:21 (albumResolver.Media)
  │
  │  ① 认证 + 过滤（onlyFavorites 子查询）
  │  ② 构造基础 query: db.Where("album_id = ?", obj.ID)...
  │
  │  ┌── Schema 层 ──┐
  │  │ Ordering{     │   OrderBy="date_shot", DESC
  │  │  order_by     │   → clause.OrderByColumn{Name:"date_shot", Desc:true}
  │  │  order_direction │
  │  └───────────────┘
  │           │
  │  ┌── Pagination ─┐
  │  │ limit=200     │   → tx.Limit(200)  → SQL LIMIT 200
  │  │ offset=0      │   → tx.Offset(0)   → SQL OFFSET 0
  │  └───────────────┘
  │           │
  │           ▼
  │  ③ models.FormatSQL(query, order, paginate)  ← 统一 SQL 构建器 [utils.go:11]
  │     → 产出 GORM query 带: ORDER BY date_shot DESC LIMIT 200 OFFSET 0
  │
  │     (注: Timeline 则先自行内置多维排序，再用 FormatSQL 加 limit/offset)
  │
  │  ④ query.Find(&media) → 执行 SQL，加载 []*Media（SELECT *，所有列）
  │
  │  ┌── 字段裁剪层 ──┐
  │  │ 注意: SQL 层并未根据请求字段做列选择，        │
  │  │ 但 Media 关联的子字段是否加载取决于 Resolver 调度 │
  │  └───────────────┘
  │
  │  返回 []*models.Media（长度 ≤ 200）
  ▼
回到 gqlgen: marshalNMedia(ctx, selections, medias)
  │
  │  对每个 media 循环调用 _Media(ctx, selections, media)
  │  _Media 中:
  │    CollectFields → 只处理前端请求的字段 (thumbnail, favorite, date, ...)
  │    switch field.Name {
  │      case "thumbnail": → Concurrently 调用 Media().Thumbnail(ctx, media) → Dataloader
  │      case "favorite":  → Concurrently 调用 Media().Favorite(ctx, media)  → Dataloader
  │      case "date":      → 直接读 media.DateShot (无额外 SQL)
  │      case "exif":      → 未请求，此分支永不进入！
  │    }
  │
  │  → 最终 JSON 输出只包含请求字段
  ▼
响应 JSON → Apollo Client → 更新 UI
  │
  │  滚动到底部 → IntersectionObserver 触发 (useScrollPagination)
  │  fetchMore({ variables: { offset: 200, limit: 200 } })
  │  → 重新走同样链路，唯一不同: paginate.Offset = 200
```

---

## 五、设计优缺点总结

### 优点

| 方面 | 说明 |
|-----|------|
| **简单性** | 避开 Relay Connection 规范的复杂性，Schema 和 Resolver 代码量少 |
| **前端易用** | `useScrollPagination` Hook 统一封装无限滚动，Apollo `fetchMore` 自然 |
| **字段懒加载** | 通过 `gqlgen.yml` 的 `resolver: true` 配置，关联数据（EXIF、人脸、URL）按需加载 |
| **排序灵活** | `order_by` 是自由字符串，前端可选任意数据库列做排序 |
| **Timeline 排序** | 多数据库（PostgreSQL/SQLite/MySQL）分别适配日期截断函数，实现一致的按日分组 |

### 缺点 / 潜在改进

| 方面 | 说明 | 可能的改进 |
|-----|------|----------|
| **Offset 性能** | 大 offset（如 `OFFSET 10000 LIMIT 200`）时数据库需扫描并丢弃前 10000 行，性能线性恶化 | 引入 Keyset Pagination（基于 `(id, sort_key) > (last_id, last_value)` 游标） |
| **缺少总计数** | 无 `totalCount`，前端无法知道"共多少页"，只能靠"加载到 0 条为止"判断到底 | 增加可选 `*Count` 字段或包装 Connection 类型 |
| **无显式 tie-breaker** | `order_by` 单字段排序在值重复时可能不稳定（取决于 DB 实现） | 在 FormatSQL 中自动追加 `, id ASC` 作为最终排序键 |
| **SQL 列未裁剪** | GORM 默认 `SELECT *`，大表（含 Blurhash 等重列）时浪费 IO | 利用 `graphql.CollectFields` 结果构建 `db.Select(...)`，只请求需要的列 |
| **order_by 注入面** | String 类型直接进 clause.Column（虽经 clause 转义但非 Enum） | 改为 Schema enum（`AlbumOrderField`、`MediaOrderField`），后端白名单校验 |
| **Offset 漂移** | 翻页期间新增/删除数据时，会出现重复项或遗漏（经典 Offset 问题） | 引入基于最后一项主键+排序列的 Cursor |

---

## 六、关键代码位置索引

| 概念 | 文件 | 行号 |
|-----|------|-----|
| Schema 类型定义 | `api/graphql/resolvers/root.graphql` | 8-28 |
| FormatSQL 核心 | `api/graphql/models/utils.go` | 11-40 |
| myAlbums Action | `api/graphql/models/actions/album_actions.go` | 9-48 |
| myMedia Action | `api/graphql/models/actions/media_actions.go` | 8-23 |
| myTimeline Action（多维排序） | `api/graphql/models/actions/timeline_actions.go` | 11-62 |
| albumResolver.Media | `api/graphql/resolvers/album.go` | 21-51 |
| albumResolver.SubAlbums | `api/graphql/resolvers/album.go` | 54-65 |
| queryResolver.MyFaceGroups | `api/graphql/resolvers/faces.go` | 377-414 |
| gqlgen 配置（Lazy Resolver 标记） | `api/gqlgen.yml` | 21-72 |
| _Album 字段调度 | `api/graphql/generated.go` | 9333-9532 |
| _Query 顶层调度 | `api/graphql/generated.go` | 10818-10967 |
| CollectFields 使用点 | `api/graphql/generated.go` | 9334, 9971, 10819 等 |
| 前端 useScrollPagination | `ui/src/hooks/useScrollPagination.ts` | 18-99 |
| AlbumPage 查询 | `ui/src/Pages/AlbumPage/AlbumPage.tsx` | 16-118 |
| Timeline 查询 | `ui/src/components/timelineGallery/TimelineGallery.tsx` | 22-193 |
| unmarshalInputPagination | `api/graphql/generated.go` | 9249-9284 |
| unmarshalInputOrdering | `api/graphql/generated.go` | 9212-9247 |
| field_Query_myAlbums_args | `api/graphql/generated.go` | 2712-2756 |
| field_Album_media_args | `api/graphql/generated.go` | 2078-2106 |
| GenerateToken | `api/utils/utils.go` | 13-29 |
| share_token bcrypt | `api/graphql/models/actions/share_token_actions.go` | 153-165 |
| auth Bearer 校验正则 | `api/graphql/auth/auth.go` | 17 |
| faces 边界测试 | `api/graphql/resolvers/faces_test.go` | 336-373 |

---

## 七、游标二进制编码细节：项目中**不存在**真正的 Cursor

### 7.1 重要事实澄清

在 `api/` 目录下搜索 `cursor` / `Cursor` / `base64` / `base64.StdEncoding` / `encoding.base64` —— **分页相关代码 0 匹配。

项目中 base64 的使用全部集中在 `scanner/`（媒体处理、blurhash、人脸检测等二进制数据序列化），**完全与分页无关。

### 7.2 易被误判为"游标编码"的 Token 机制

项目中有两类 Token，均易与分页游标混淆，但都与分页无关：

#### （1）Auth Token（访问令牌）

位置：`api/utils/utils.go:13-29

```go
func GenerateToken() string {
    const charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    const length = 8

    charLen := big.NewInt(int64(len(charset)))

    b := make([]byte, length)
    for i := range b {
        n, err := rand.Int(rand.Reader, charLen)
        if err != nil {
            log.Panicf("Could not generate random number: %v", err)
        }
        b[i] = charset[n.Int64()]
    }
    return string(b)
}
```

**编码特征**：
- 使用 `crypto/rand` 加密安全随机数（非伪随机）
- 字符集 = `[a-zA-Z0-9] 共 62 个字符
- 长度固定 = 8 字符
- **无 base64 编码**：直接从 charset 中逐个随机选取，结果是纯可读字符串
- 用于：存储在 `access_tokens` 表中，认证时从 Cookie 或 `Bearer` 中提取后直接查库校验

#### （2）Share Token 的 bcrypt 密码签名

位置：`api/graphql/models/actions/share_token_actions.go:153-165

```go
func hashSharePassword(password *string) (*string, error) {
    var hashedPassword *string = nil
    if password != nil {
        hashedPassBytes, err := bcrypt.GenerateFromPassword([]byte(*password), 12)
        if err != nil {
            return nil, errors.Wrap(err, "failed to generate hash for share password")
        }
        hashedStr := string(hashedPassBytes)
        hashedPassword = &hashedStr
    }

    return hashedPassword, nil
}
```

**编码特征**：
- 使用标准 `bcrypt` 哈希，成本因子 = 12
- 输出是 bcrypt 标准格式字符串（`$2a$12$...）
- 仅用于 Share Token 的访问密码保护
- **与分页游标完全无关

### 7.3 分页中"游标等价物"

| 机制 | 内容 | 是否 Base64 | 是否有签名 |
|------|------|---------|----------|
| **分页游标 | `offset: { limit, offset } | ❌ | ❌ |
| **Auth Token | `crypto/rand` 8 字符 | ❌ | ❌（直接查库校验） |
| **Share Token Value | `crypto/rand` 8 字符 | ❌ | 🔒 Password 用 bcrypt |
| **Share Token Password | bcrypt 哈希 | ❌ | bcrypt 自包含 salt+hash |

**关键结论**：Photoview 分页完全没有使用任何形式的游标编码（Base64、HMAC 签名、或校验和。前端的"翻页标记"就是一个纯整数 `offset`，直接通过 JSON 明文传输。

---

## 八、分页参数解析链路与空值处理

### 8.1 四层参数解析调用栈

以 `Album.media(paginate: { limit: 200, offset: 0 }) 为例：

```
GraphQL Variables JSON
  │
  │  { "paginate": { "limit": 200, "offset": 0 } }
  ▼
第 1 层: field_Album_media_args()   [generated.go:2078]
  │
  │  graphql.ProcessArgField(ctx, rawArgs, "paginate",
  │    func(v any) (*Pagination, error) {
  │      return ec.unmarshalOPagination2ᚖ...(ctx, v)
  │    })
  ▼
第 2 层: unmarshalOPagination2ᚖgithubᚗcomᚋphotoviewᚋphotoviewᚋapiᚋgraphqlᚋmodelsᚐPagination
          [generated.go:12952-12958]
  │
  │  if v == nil → return nil, nil  // 外层为空则返回 (nil, nil)
  │  res, err := ec.unmarshalInputPagination(ctx, v)
  │  return &res, err
  ▼
第 3 层: unmarshalInputPagination()   [generated.go:9249-9284]
  │
  │  var it models.Pagination       // 零值: {Limit: nil, Offset: nil
  │  if obj == nil { return it, nil }
  │
  │  asMap := obj.(map[string]any)
  │  fieldsInOrder := [...]string{"limit", "offset"}
  │  for _, k := range fieldsInOrder {
  │    v, ok := asMap[k]; if !ok { continue }
  │    switch k {
  │      case "limit":
  │        data, err := ec.unmarshalOInt2ᚖint(ctx, v)
  │        it.Limit = data
  │      case "offset":
  │        data, err := ec.unmarshalOInt2ᚖint(ctx, v)
  │        it.Offset = data
  ▼
第 4 层: unmarshalOInt2ᚖint()   [generated.go:12855-12861]
  │
  │  if v == nil { return nil, nil }  // 字段缺失或为 null 时返回 (nil, nil)
  │  res, err := graphql.UnmarshalInt(v)  // gqlgen 库标准 Int 解析
  │  return &res, graphql.ErrorOnPath(ctx, err)
  ▼
最终: Pagination{ Limit: &200, Offset: &0 }
```

### 8.2 空值处理矩阵

| 输入情形 | unmarshalOPagination2... 返回 | 后续行为 |
|---------|-----------------------------|----------|
| **未传 paginate 参数 | `(nil, nil)` | `FormatSQL(tx, nil, nil)` → **无 LIMIT/OFFSET，返回全部 |
| `paginate: null` | `(nil, nil)` | 同上 |
| `paginate: {}` | `&Pagination{Limit: nil, Offset: nil}` | `FormatSQL` 中两个 if 都不触发，同上 |
| `paginate: { limit: null }` | `&Pagination{Limit: nil, Offset: nil}` | 同上 |
| `paginate: { offset: null }` | `&Pagination{Limit: nil, Offset: nil}` | 同上 |
| `paginate: { limit: 200 }` | `&Pagination{Limit: &200, Offset: nil}` | 仅加 LIMIT 200，无 OFFSET |
| `paginate: { offset: 100 }` | `&Pagination{Limit: nil, Offset: &100}` | 仅加 OFFSET 100，无 LIMIT（危险：返回从 101 条到末尾 |
| `paginate: { limit: 200, offset: 0 }` | `&Pagination{Limit: &200, Offset: &0}` | LIMIT 200 OFFSET 0 |

### 8.3 Ordering 的相同的完全相同

Ordering 的四层解析与 Pagination 完全同构：
- `unmarshalOOrdering2... → `unmarshalInputOrdering` → `unmarshalOString2...`（order_by） + `unmarshalOOrderDirection2...`（order_direction）
- 所有字段缺失/为 null 时返回指针为 `nil`
- `FormatSQL` 中 `order != nil && order.OrderBy != nil` 两个条件同时满足才会生成 ORDER BY 子句

---

## 九、边界页处理路径：first=0、空集与小集合

### 9.1 limit = 0 的处理路径

**gqlgen 层：`unmarshalOInt2ᚖint` 对 `0` 会正常解析为 `&0`，无任何特殊处理。

**FormatSQL 层**：

```go
// api/graphql/models/utils.go:16-19
if paginate != nil {
    if paginate.Limit != nil {
        tx.Limit(*paginate.Limit)   // *paginate.Limit = 0 → tx.Limit(0)
    }
}
```

**GORM 层**：`tx.Limit(0)` 直接传递给数据库驱动 → SQL `LIMIT 0`。

**数据库层**：
- PostgreSQL / MySQL / SQLite：`LIMIT 0` 合法，返回 **0 行结果集**，不报错
- 性能：数据库优化器直接识别 LIMIT 0，无需扫描表，几乎零开销

**Resolver 层**：
```go
// albumResolver.Media 中：
query.Find(&media)
// media 为 len(media) == 0，err == nil
```
→ GORM `Find` 对空结果**不报错**，只返回空 slice，err 为 nil

**gqlgen 序列化层**：
```go
// api/graphql/generated.go:12390-12404
func marshalNMedia2ᚕᚖ...Mediaᚄ(ctx, sel, v []*models.Media) {
    ret := graphql.MarshalSliceConcurrently(ctx, len(v), 0, false, ...)
    // len(v) = 0 → 生成空 JSON 数组 []
    for _, e := range ret { ... }  // 循环 0 次
    return ret  // 返回 graphql.MarshalSliceConcurrently 空数组
}
```
→ 最终输出 `"media": []`（空 JSON 数组）

### 9.2 空集合（无匹配数据）的处理路径

**条件**：用户相册为空 / `onlyFavorites: true 但无收藏 / `fromDate` 过滤后无结果。

**链路**：
```
resolver → db.Where(...).Find(&media)
  │
  │  GORM: SELECT ... WHERE ... LIMIT 200 OFFSET 0
  │  → 结果 0 行
  ▼
media = []*models.Media{}（空 slice，len=0，cap=0）
  │
  ▼
gqlgen marshalNMedia2[...]
  │
  │  MarshalSliceConcurrently(ctx, 0, 0, false, ...)
  │  → 生成空 JSON 数组 []
  ▼
JSON 输出: "media": []
```

**前端判断逻辑**（`ui/src/hooks/useScrollPagination.ts:53-63`）：
```typescript
fetchMore({ variables: { offset: itemCount } }).then(result => {
    const newItemCount = getItems(result.data).length
    if (newItemCount == 0) {
        setFinished(true)  // ← 加载到 0 条即标记结束，停止触发加载更多
    }
})
```
→ 前端通过"本次新增 0 条"判断已到底部。

### 9.3 小集合（数据量 < limit）的处理路径

**条件**：limit = 200，但实际匹配总数 = 57。

**SQL 层**：`LIMIT 200 OFFSET 0` → 数据库返回 57 行，GORM 自动截断（实际只有 57 行返回）

**Resolver 层**：`len(media) = 57`，**不报错。

**前端第一页**（57 条全部加载）：
```typescript
// useScrollPagination
itemCount = 57
fetchMore({ variables: { offset: 57, limit: 200 }})
  │
  ▼
后端: SELECT ... LIMIT 200 OFFSET 57
  → 返回 0 行
  │
  ▼
newItemCount = 0 → setFinished(true)
```
→ 第二页请求 offset = 57，返回 0 条 → 判定加载完成。

### 9.4 offset 溢出（offset > 总数）的处理路径

**条件**：总数 = 57，但 offset = 1000。

**SQL 层**：`LIMIT 200 OFFSET 1000` → 数据库需跳过 1000 行后，剩余 0 行。

**数据库行为差异**：
| 数据库 | OFFSET 超过总行数 | 性能 |
|-------|----------------|------|
| PostgreSQL | 正常返回空结果 | 仍需扫描并丢弃 offset 行（大 offset 性能差） |
| MySQL | 正常返回空结果 | 同上 |
| SQLite | 正常返回空结果 | 同上 |

**Resolver / gqlgen / 前端**：处理方式与 9.2 空集合完全相同 → 空 JSON 数组 → 前端 `setFinished(true)`。

### 9.5 边界条件测试覆盖情况

| 边界场景 | 单元测试覆盖 | 位置 |
|---------|-----------|------|
| 空输入 ID 列表返回空 | ✅ `TestGetUserOwnedImageFaces` | `faces_test.go:337-347` |
| 用户无关联相册返回空 | ✅ | `faces_test.go:349-373` |
| Timeline 基础分页 | ❌ 无测试 | — |
| `limit=0` | ❌ 无测试 | — |
| offset > 总数 | ❌ 无测试 | — |
| 空相册 media | ❌ 无测试 | — |
| `onlyFavorites` 空 | ✅ `TestMyTimeline` | `timeline_actions_test.go:102-108` |
| `fromDate` 过滤 | ✅ | `timeline_actions_test.go:110-116` |
| 合并相同 media 去重 | ✅ `TestCombineFaceGroups` | `faces_test.go:126-158` |

### 9.6 FormatSQL 的完整代码分析

```go
// api/graphql/models/utils.go:11-40
func FormatSQL(tx *gorm.DB, order *Ordering, paginate *Pagination) *gorm.DB {
    // 1. paginate 为 nil → 完全跳过 limit/offset
    if paginate != nil {
        // 1a. Limit 为 nil → 不设置 LIMIT，返回全部剩余
        if paginate.Limit != nil {
            tx.Limit(*paginate.Limit)
        }
        // 1b. Offset 为 nil → 不设置 OFFSET，从头开始
        if paginate.Offset != nil {
            tx.Offset(*paginate.Offset)
        }
    }

    // 2. order 为 nil → 完全跳过 ORDER BY，使用数据库默认顺序
    if order != nil && order.OrderBy != nil {
        // 2a. OrderDirection 为 nil → 默认 ASC
        desc := false
        if order.OrderDirection != nil && order.OrderDirection.IsValid() {
            if *order.OrderDirection == OrderDirectionDesc {
                desc = true
            }
        }
        tx.Order(clause.OrderByColumn{
            Column: clause.Column{Name: *order.OrderBy},
            Desc: desc,
        })
    }
    return tx
}
```

**所有组合共 9 种情形**：

| paginate | order | SQL 结果 |
|---------|-------|---------|
| nil | nil | 无 LIMIT，无 ORDER BY → 全表返回，DB 默认顺序 |
| nil | {OrderBy: "title"} | 无 LIMIT，ORDER BY title ASC → 全表按标题排序 |
| {Limit: 200} | nil | LIMIT 200，无 ORDER BY → 前 200 条，DB 默认顺序 |
| {Limit: 200, Offset: 400} | nil | LIMIT 200 OFFSET 400 → 第 401-600 条 |
| {Limit: 0} | {OrderBy: "date"} | LIMIT 0 ORDER BY date → 0 行 |
| {Offset: 100} | {OrderBy: "title", DESC} | OFFSET 100 ORDER BY title DESC → 第 101 条到末尾，按标题降序 |
| {Limit: 200, Offset: 0} | {OrderBy: "date", DESC} | 正常分页：LIMIT 200 OFFSET 0 ORDER BY date DESC |
| {}（所有字段 nil | {}（所有字段 nil | 与 nil, nil 相同 → 无 LIMIT，无 ORDER BY |
| {Limit: nil, Offset: nil} | {OrderBy: nil, OrderDirection: nil} | 同上 |

---

## 十、签名/校验位置总览（与分页无关但易混淆）

项目中有三类签名/校验机制，**均与分页游标无关，但常被误读：

### 10.1 Auth Token 校验

**位置**：`api/graphql/auth/auth.go:31-70

```go
var bearerRegex = regexp.MustCompile("^(?i)Bearer ([a-zA-Z0-9]{24})$")

func Middleware(db *gorm.DB) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // 从 Cookie 中读取 "auth-token"
            if tokenCookie, err := r.Cookie("auth-token"); err == nil {
                loaders := dataloader.For(r.Context())
                // Dataloader 批量查库校验 token → 返回 User
                user, err := loaders.UserFromAccessToken.Load(tokenCookie.Value)
                // 错误处理（查库失败返回 401
                if err != nil { ... }
                if user == nil { ... }
                // 通过 → 放入 context
                ctx := AddUserToContext(r.Context(), user)
                r = r.WithContext(ctx)
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

**校验方式**：正则 `[a-zA-Z0-9]{24}` 正则校验格式 + Dataloader 查库校验存在性。Token 本身**不签名，存在性校验，token value 直接明文存储。

### 10.2 Share Token 密码签名

**位置**：`api/graphql/models/actions/share_token_actions.go:40, 79-87, 153-165

```go
// 创建时哈希
hashedPassBytes, err := bcrypt.GenerateFromPassword([]byte(*password), 12)
hashedStr := string(hashedPassBytes)
shareToken := models.ShareToken{
    Value:    utils.GenerateToken(),  // 8 字符随机
    Password: &hashedStr,           // bcrypt 哈希
    ...
}

// 校验时（登录/访问时）
err := bcrypt.CompareHashAndPassword([]byte(hashedPassword), []byte(inputPassword))
```

**签名方式**：bcrypt（含 salt + 2^12 = 4096 次迭代）

### 10.3 Media URL 签名

Media 图片 URL 通过 `api/routes/photos.go` 和 `videos.go` 中通过 `api/utils` 文件路由

```go
// 例如照片路由鉴权
// 路由签名逻辑：token 从 URL query 参数中提取，查数据库校验 token 是否属于该媒体
```

### 10.4 与分页的区别

| 方面 | 分页 offset | Auth Token | Share Token Password | Media URL Token |
|-----|-------------|------------|--------------------|----------------|
| **目的 | 翻页位置标记 | 用户身份识别 | 分享链接密码保护 | 媒体访问授权 |
| **编码** | JSON 整数明文 | `[a-zA-Z0-9]{8} | bcrypt `$2a$12$...` | `[a-zA-Z0-9]{8} |
| **Base64** | ❌ | ❌ | ❌（bcrypt 自身编码非 base64） | ❌ |
| **HMAC 签名 | ❌ | ❌ | ✅ bcrypt（单向哈希） | ❌ |
| **校验方式** | 无（直接传数据库） | 查 `access_tokens` 表 | bcrypt.Compare | 查 `media_urls` 表 |
| **过期机制** | 无（始终有效） | Token 行删除即失效 | `Expire` 字段 | 无永久有效 |

---

## 十一、修正与补充：之前"游标"概念在本项目中的实际对应

| Relay/GraphQL 标准术语 | Photoview 中的等价实现 |
|---------------------|---------------------|
| `first: N` | `paginate: { limit: N }` |
| `after: Cursor` | `paginate: { offset: position }` |
| `last: N` | ❌ 不存在（无反向分页） |
| `before: Cursor` | ❌ 不存在 |
| `edges { node, cursor }` | ❌ 不存在（直接返回 `[T]!` 数组） |
| `pageInfo { hasNextPage, hasPreviousPage, startCursor, endCursor }` | ❌ 不存在（前端靠 `len(result)==0` 判断 |
| `totalCount` | ❌ 不存在（总数字段 |
| `encodeCursor()` / `decodeCursor()` | ❌ 不存在（offset 明文传输，无需编解码 |
| `base64("arrayconnection:123")` | ❌ 无编码 |
| Cursor HMAC 签名防篡改 | ❌ 无签名，offset 可被前端任意修改 |

**安全提示**：由于 offset 是明文整数，前端可以构造 offset 参数，**理论上可以构造任意 offset（如 `offset: 999999`），但由于后端没有做范围校验，数据库会执行，但由于 GORM 的 `tx.Offset(999999)` 直接传递给数据库，大 offset 会导致数据库扫描大量行然后丢弃，**存在潜在的 DoS 风险**。
