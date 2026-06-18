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

---

## 十二、并发增删时 offset 一致性影响分析

### 12.1 项目的事务与隔离级别现状

**查询 Resolver（所有 myAlbums / myMedia / myTimeline / Album.media / ...）均无显式事务**：

```go
// api/graphql/resolvers/resolver.go:22-24
func (r *Resolver) DB(ctx context.Context) *gorm.DB {
    return r.database.WithContext(ctx)  // ← 仅加入 context，无 Transaction()
}

// actions/album_actions.go:42-45
query.Find(&albums)  // ← 裸查询，无 BEGIN / COMMIT 包裹
```

**代码中仅 10 处显式事务**，且全部集中在写操作：

| 位置 | 事务用途 | 与分页查询的关系 |
|-----|---------|---------------|
| `scanner_user.go:127` | Album 创建 + Owner 关联原子化 | 相册目录扫描写入时并发 |
| `cleanup_media.go:111` | UserAlbums 删除 + Album 删除原子化 | 清理旧相册时并发 |
| `user.go:36,72,151,202` | 用户注册/授权/删除操作 | 用户管理写操作 |
| `faces.go:182,242,313,342` | 人脸组合并/移动/重命名 | 人脸管理写操作 |

**隔离级别**：代码中**未自定义隔离级别**，完全依赖数据库默认：

| 数据库 | 默认隔离级别 | 能否防止不可重复读 | 对分页一致性的影响 |
|-------|-----------|---------------|------------------|
| **PostgreSQL** | READ COMMITTED | ❌ 不能 | **同一翻页会话可能遇到不可重复读**（翻到第二页时第一页数据已变） |
| **MySQL InnoDB** | REPEATABLE READ | ✅ 可以 | **同一事务内可重复读**，但 Resolver 未开事务 → 仍是 READ COMMITTED 级别 |
| **SQLite (WAL 模式)** | SERIALIZABLE（读无锁） | ✅ 可以 | WAL 下写操作做快照，读操作一致但**只对单个语句** |

### 12.2 写操作触发源：可能引起数据漂移的 4 条路径

在分页浏览的同时，以下路径可能**修改结果集**：

#### 路径 1：Scanner 扫描新文件（插入）

位置：`api/scanner/scanner_media.go:21-73`

```go
func ScanMedia(tx *gorm.DB, mediaPath string, albumId int, ...) (*models.Media, bool, error) {
    // ...
    media := models.Media{
        Title:    mediaName,
        Path:     mediaPath,
        AlbumID:  albumId,
        DateShot: stat.ModTime(),  // ← 日期值取决于文件修改时间
    }

    tx.Create(&media)  // ← 插入新行，可能改变排序结果
    return &media, true, nil
}
```

**对 offset 的影响**：
- Scanner 是**后台 goroutine 异步**执行，插入时机完全不可控
- 若按 `date_shot DESC` 排序，**新照片的 `DateShot` 可能是任意值**，既可能排在前面（最近修改），也可能在中间，也可能在最后
- **典型漂移场景**：用户在 `AlbumPage` 翻到第 3 页（offset=400），此时 Scanner 扫描到 5 张新照片，`DateShot` 刚好位于第 2 页和第 3 页之间 → 用户翻第 4 页时会**重复看到第 3 页已经看过的条目**

#### 路径 2：CleanupMedia 清理旧文件（删除）

位置：`api/scanner/scanner_tasks/cleanup_tasks/cleanup_media.go:17-65`

```go
func CleanupMedia(db *gorm.DB, albumId int, albumMedia []*models.Media) []error {
    // ...
    var mediaList []models.Media
    query := db.Where("album_id = ?", albumId)
    if len(albumMedia) > 0 {
        query = query.Where("NOT id IN (?)", albumMediaIds)  // ← 选中磁盘上已不存在的
    }
    query.Find(&mediaList)

    // ...
    db.Where("id IN (?)", mediaIDs).Delete(models.Media{})  // ← 批量删除
    // ...
}
```

**对 offset 的影响**：
- 删除发生在**第一页到第 N 页之间**的条目时，会导致**后续条目整体前移**
- **典型漂移场景**：用户刚翻完第 1 页（offset 0-199），此时 Scanner 清理掉了第 1 页上的 3 条记录 → 翻到第 2 页（offset=200）时，实际看到的是原来第 203 条开始的记录，**遗漏了原本应该是 200-202 的三条记录**
- 更极端：清理量很大时，第 2 页可能直接跳到与第 1 页差出几十条的位置

#### 路径 3：收藏/取消收藏（改变 onlyFavorites 过滤结果集）

位置：`api/graphql/models/user.go:183-200`

```go
func (user *User) FavoriteMedia(db *gorm.DB, mediaID int, favorite bool) (*Media, error) {
    userMediaData := UserMediaData{
        UserID:   user.ID,
        MediaID:  mediaID,
        Favorite: favorite,
    }
    // ← UPSERT（OnConflict UpdateAll），无事务包裹整个操作
    db.Clauses(clause.OnConflict{UpdateAll: true}).Create(&userMediaData)

    db.First(&media, mediaID)
    return &media, nil
}
```

**对 offset 的影响**（仅当查询带 `onlyFavorites=true` 时）：
- **收藏新照片**：原来不在结果集里的照片变成了结果集成员 → 若 `OrderBy` 排序键恰好把它排在正在浏览的窗口**之前**，会导致后续页面重复或错乱
- **取消收藏**：照片从结果集中消失 → 后续页面前移，可能漏掉记录
- **多用户并发收藏**：A 用户在翻页，B 用户同时对同一相册的照片大量操作，结果集变化与 A 无关（`user_media_data.user_id` 过滤隔离），但 B 自己翻页会受自己操作影响

#### 路径 4：删除人脸组 / 合并人脸组（改变 FaceGroup 查询结果）

位置：`api/graphql/resolvers/faces.go:182-242,313-342`

```go
// CombineFaceGroups 事务内：先查重复 media → 批量更新 ImageFace → 更新 FaceGroup 计数
updateError := db.Transaction(func(tx *gorm.DB) error {
    // ... 批量 UPDATE image_faces SET face_group_id = dest ...
    // ...
})
```

**对 offset 的影响**：仅对 `myFaceGroups` 和 `FaceGroup.imageFaces` 分页生效，影响较小。

### 12.3 6 种典型并发漂移场景

假设相册有 1000 张照片，按 `date_shot DESC` 排序，前端每页 200 张：

| 场景 | 并发操作时机 | 第一页结果 (0-199) | 第二页结果 (200-399) | 用户体验 |
|-----|-----------|------------------|-------------------|---------|
| **S1. 前置插入** | 第一页渲染完成后，Scanner 插入 5 张 date_shot 比当前第 200 条**新**的照片 | 不变（已加载） | 原 195-194 + 原 200-398（5 条重复出现在第 1 页未刷新的内容） | **看到重复照片** |
| **S2. 中置插入** | 第一页与第二页之间插入 5 张照片，date_shot 恰好落在 200-201 之间 | 不变 | 原 195-199 + 新 5 + 原 200-394（**5 条与第 1 页末尾重复**） | 看到重复照片 |
| **S3. 前置删除** | 第一页渲染完成后，Cleanup 删除了第 1 页前 10 条 | 不变 | 原 210-409（丢失原 200-209 共 10 条） | **照片莫名其妙缺失** |
| **S4. 后置删除** | 翻到第 2 页后，删除原 205-214 共 10 条 | 不变 | 原 200-204 + 原 215-404（**原 205-214 从第 2 页消失**） | 第 2 页只剩 190 条不重复，但与第 1 页逻辑上衔接正常 |
| **S5. 排序键更新** | 翻到第 2 页时，Scanner 重新处理了第 1 页某些照片的 EXIF → 更新了 date_shot | 不变 | 顺序完全错乱，部分记录换页 | **严重错乱** |
| **S6. onlyFavorites 并发收藏** | 第 1 页浏览中用户快速收藏了 3 张本应在第 3 页的照片 | 不变（仅 200 条内） | 第 2 页可能混入本应第 3 页的条目 | 顺序错乱 |

### 12.4 代码中**未**使用的一致性保障手段

| 保障手段 | 是否使用 | 说明 |
|---------|---------|------|
| **查询时开启 REPEATABLE READ 事务** | ❌ 未用 | 所有分页查询均无事务包裹 |
| **Snapshot Isolation / FOR SHARE 锁** | ❌ 未用 | 无 `Clauses(clause.Locking{Strength: "SHARE"})` |
| **Cursor-based Keyset Pagination** | ❌ 未用 | 用 `(id, sort_key) > (last_id, last_val)` 天然免疫漂移 |
| **Limit+1 预读一页（hasNextPage 验证）** | ❌ 未用 | 无 `limit + 1` 模式 |
| **时间戳快照过滤** | ❌ 未用 | 无 `WHERE updated_at <= snapshot_time` |
| **单用户串行化 Scanner 与查询** | ❌ 未用 | Scanner 与 HTTP 请求无协调 |

### 12.5 实际风险评估

| 场景 | 发生频率 | 用户可见影响 | 严重度 |
|-----|---------|------------|-------|
| 日常小量增删（S1-S2） | 中（每次扫描都有） | 偶见重复或漏掉 1-2 张 | 低 |
| 大规模清理后翻页（S3） | 低（仅定期扫描） | 可能漏掉几十张 | 中 |
| Scanner 批量导入新照片（S1-S2 极端） | 中 | 重复率高 | 中 |
| onlyFavorites 翻页中操作（S6） | 低 | 顺序错乱 | 低 |
| EXIF 更新重排序（S5） | 极低（一般不重处理） | 严重错乱 | 中 |
| 大 offset 扫描期间数据变更 | 高（每次大翻页都有） | 偏移计算与实际行位置不对应 | 中 |

---

## 十三、大 offset 在 ORM/数据库侧的性能成本

### 13.1 GORM 中的 offset/limit 传递链

```go
// 第 1 层: FormatSQL (api/graphql/models/utils.go:11-40)
func FormatSQL(tx *gorm.DB, order *Ordering, paginate *Pagination) *gorm.DB {
    if paginate != nil {
        if paginate.Limit != nil {
            tx.Limit(*paginate.Limit)   // ← 第 1 步: 写入 tx.Statement.Limit
        }
        if paginate.Offset != nil {
            tx.Offset(*paginate.Offset) // ← 第 2 步: 写入 tx.Statement.Offset
        }
    }
    if order != nil && order.OrderBy != nil {
        tx.Order(clause.OrderByColumn{...}) // ← 写入 ORDER BY
    }
    return tx
}
```

**GORM 内部处理 Limit/Offset**（`gorm.io/gorm` 库，版本由 `go.mod` 决定）：
- `tx.Limit(N)` → 设置 `Statement.Clauses["LIMIT"] = clause.Limit{Limit: N}`
- `tx.Offset(M)` → 设置 `Statement.Clauses["LIMIT"] = clause.Limit{Offset: M}`（同一条 clause，内部区分）
- 到 Build SQL 阶段（PostgreSQL 驱动）：`SELECT ... ORDER BY ... LIMIT $1 OFFSET $2`，参数化绑定

**GORM 侧无任何优化**：不会改写 SQL 为游标/seek 模式，不会自动分页，不会预判 offset 合法性。完全原样透传至数据库。

### 13.2 数据库侧 LIMIT/OFFSET 执行路径

以 PostgreSQL 为例（MySQL/SQLite 原理相同）：

```sql
-- 用户翻到第 100 页（每页 200）：offset=19800, limit=200
SELECT * FROM media
WHERE media.album_id IN (
    SELECT user_albums.album_id FROM user_albums WHERE user_albums.user_id = 42
)
ORDER BY media.date_shot DESC
LIMIT 200 OFFSET 19800;
```

**PostgreSQL 查询执行器内部路径**：

```
优化器生成计划（EXPLAIN）：
  ├─ 如果 ORDER BY 列有索引（media.date_shot）→ Index Scan Backward（索引反向扫描）
  │    但 OFFSET 19800 仍需跳过前 19800 条索引页，逐一丢弃 → **成本仍高**
  │
  ├─ 如果 ORDER BY 列无索引（media.date_shot 默认无索引）→ Sort + Limit
  │    ├─ 步骤 1: Seq Scan on media（全表扫描找到所有匹配行）
  │    │   → 对 100 万行 media 做过滤 album_id IN (...)
  │    ├─ 步骤 2: Sort (Sort Method: external merge Disk Sort 或 quicksort memory)
  │    │   → 对过滤后的 50 万行按 date_shot DESC 排序
  │    │   → 需要 O(N log N) 时间，可能溢出到磁盘
  │    └─ 步骤 3: Limit
  │         → 跳过前 19800 条已排序的行，取出 200 条
  │         → **前 19800 行虽然丢弃但必须实际排完序**
  │
  └─ 有额外 WHERE 子查询（onlyFavorites / onlyWithFavorites）
       → 子查询每一行都要执行 EXISTS 判断，复杂度进一步放大
```

### 13.3 Media 模型索引现状（关键点：`date_shot` 无索引）

模型定义（`api/graphql/models/media.go:15-33`）：

```go
type Media struct {
    Model           // ID: gorm:"primarykey"（隐含索引 PRIMARY KEY）
    Title    string
    Path     string
    PathHash string `gorm:"not null;unique"`       // UNIQUE INDEX
    AlbumID  int    `gorm:"not null;index"`        // ✅ 有索引 btree(album_id)
    DateShot time.Time `gorm:"not null"`           // ❌ 无索引（最常用的排序键！）
    Type     MediaType  `gorm:"not null;index"`    // ✅ btree(type)
    // ...
}
```

**实际表的索引矩阵**：

| 索引名 | 列 | 对分页的作用 |
|-------|-----|-----------|
| PRIMARY KEY | `id` | 按 ID 排序可走索引，但前端默认按 date_shot 或 title，用不上 |
| `media_path_hash_key` | `path_hash` | UNIQUE，分页无用 |
| `idx_media_album_id` | `album_id` | ✅ 过滤 Album.media 能走索引 |
| `idx_media_exif_id` | `exif_id` | 关联查询用 |
| `idx_media_type` | `type` | 筛选 photo/video 用 |
| `idx_media_video_metadata_id` | `video_metadata_id` | 关联查询用 |
| **(缺失)** | `date_shot` | ❌ **Timeline 和默认排序都需要，全表排序瓶颈** |
| **(缺失)** | `(album_id, date_shot)` | ❌ **Album.media 按日期排序最佳复合索引** |
| **(缺失)** | `title` | ❌ 按标题排序不走索引 |
| **(缺失)** | `created_at` | ❌ 按导入时间排序不走索引 |

### 13.4 各分页查询的性能瓶颈

#### A. `Album.media(order: {order_by: "date_shot", DESC})`

```
瓶颈来源:
  1. AlbumID 过滤可用 idx_media_album_id → 快速缩小到该相册的 50,000 行
  2. 但 date_shot 无索引 → 对 50,000 行做 filesort（50,000 * log(50,000) ≈ 50k * 16 ≈ 80万次比较）
  3. OFFSET 10000 → 必须完整排序后丢弃前 10000 行 → 与 offset 成正比

假设相册 50,000 张照片：
  OFFSET 0      → 取 200: 需排序 50,000 行
  OFFSET 10,000 → 取 200: 仍需完整排序 50,000 行 + 额外丢弃 10,000 条
  OFFSET 40,000 → 取 200: 仍需完整排序 50,000 行 + 额外丢弃 40,000 条
```

**时间复杂度**：O(N log N) + O(OFFSET)，其中 N 是过滤后的总行数。OFFSET 不参与复杂度核心项但加线性开销。

#### B. `myTimeline(paginate)`

Timeline 有**内置多维排序**且有日期函数参与：

PostgreSQL 版（`timeline_actions.go:20-40`）：
```go
query.Order("DATE_TRUNC('year', date_shot) DESC").
      Order("DATE_TRUNC('month', date_shot) DESC").
      Order("DATE_TRUNC('day', date_shot) DESC").
      Order("albums.title ASC").
      Order("media.date_shot DESC")
```

**性能特征**：
- `DATE_TRUNC(..., date_shot)` → **函数计算，无法使用任何列索引**
- 必须为每一行计算 3 个日期截断值 → O(N) 额外计算开销
- 排序 5 个键 → 排序成本更高
- 即使在 date_shot 上加了普通索引也用不上（因为是函数调用，不是原始列排序）
- 必须使用**表达式索引**才可能走索引：
  ```sql
  CREATE INDEX idx_media_date_year ON media(DATE_TRUNC('year', date_shot));
  CREATE INDEX idx_media_date_month ON media(DATE_TRUNC('month', date_shot));
  CREATE INDEX idx_media_date_day ON media(DATE_TRUNC('day', date_shot));
  -- 或更理想的复合表达式索引
  CREATE INDEX idx_timeline_sort
  ON media(
    DATE_TRUNC('year', date_shot) DESC,
    DATE_TRUNC('month', date_shot) DESC,
    DATE_TRUNC('day', date_shot) DESC,
    date_shot DESC
  );
  ```
- **当前代码未创建任何表达式索引** → 每次 Timeline 查询都是全表排序

#### C. `myAlbums(paginate, order)`

- 排序键是字符串列 title → 文本排序开销大于数字/日期
- `onlyRoot` + `onlyWithFavorites` 引入子查询 EXISTS → 每行额外查数据库
- albums 表通常只有几十到几百行 → 大 offset 场景不常见，**实际影响小**

#### D. `myFaceGroups(paginate)` + `FaceGroup.imageFaces(paginate)`

- 人脸组通常数量有限（即使上万张照片也通常 < 1000 组）
- 排序内置：`label NULL 优先` + `COUNT(image_faces) DESC`
- COUNT() 聚合排序 → 同样无法走索引
- 但规模小 → **实际性能问题不大**

### 13.5 不同数据库的 LIMIT/OFFSET 实现差异

| 数据库 | OFFSET 算法 | 大 offset 性能 | WAL/并发读影响 |
|-------|-----------|-------------|-------------|
| **PostgreSQL** | 排序完成后用 scan direction + skip counter | 差，线性增长 | MVCC，读无锁但构建快照有开销 |
| **MySQL InnoDB** | 类似 PG 但 Filesort 实现略不同 | 差，线性增长 | MVCC，RR 级别需构建 read view |
| **SQLite (WAL)** | 需先排序到临时 B-Tree/VDBE cursor，再 seek | 更差（内存小） | WAL 下写做 checkpoing 时可能阻塞读 |

### 13.6 量化估算：100 万行 media 的表现

假设数据库硬件：4 核 CPU、16GB RAM、SSD、单用户相册 50 万张照片：

| offset | limit | PostgreSQL（有 album_id 索引，date_shot 无） | SQLite（WAL，相同数据） | 用户体感 |
|-------|------|------------------------------------------|----------------------|---------|
| 0 | 200 | ~50ms（排序 50 万行 → 但有 work_mem 优化） | ~200ms | 即时加载 |
| 2,000 | 200 | ~70ms（仍需完整排序，丢弃仅线性开销） | ~350ms | 轻微卡顿 |
| 10,000 | 200 | ~100ms | ~700ms | 明显等待 |
| 50,000 | 200 | ~250ms | ~2s | 超过 UI 轮询阈值，可能超时 |
| 100,000 | 200 | ~400ms | ~4s | 用户体验极差，可能触发 UI loading indicator |
| 500,000 | 200 | ~1.5s | ~15s | ❌ 基本上不可用 |

**关键洞察**：即使排序时间相同（因为 OFFSET 前的排序必须对全量完成），OFFSET 本身也会带来线性增长的"seek"开销，因为数据库必须真的遍历并丢弃前 N 条已排序记录。

### 13.7 代码中已有的性能优化（与分页相关）

#### (1) SQLite WAL 模式

位置：`api/database/database.go:63-70`

```go
queryValues.Add("_journal_mode", "WAL")    // Write-Ahead Logging
queryValues.Add("_locking_mode", "NORMAL") // 允许并发读写
queryValues.Add("_foreign_keys", "ON")
```

**对分页的作用**：
- 写操作写入 WAL 文件而不是主 DB → 读操作（分页查询）无需等待写锁
- 显著改善"Scanner 后台扫描 + 前端浏览"并发场景下的性能
- 但**对大 offset 本身的算法复杂度无帮助**

#### (2) 最大连接数

位置：`api/database/database.go:140`
```go
sqlDB.SetMaxOpenConns(80)
```

80 个连接池 → 支持 80 个并发分页查询同时执行，但每个查询本身的性能瓶颈不会变。

#### (3) Dataloader 批量加载（对关联查询）

位置：`api/dataloader/`（`gen_mediaurlloader.go`、`gen_userloader.go`、`gen_userfavoritesloader.go`）

**对分页的作用**：
- 分页返回 200 条 Media → 请求 Media.thumbnail 时，Dataloader 会合并 200 次单独查询为 1 次批量查询（`WHERE media_id IN (?,?,?,…)`）
- 解决了"n+1 查询"问题，减少了 Resolver 层额外查询
- 但**对主查询（media 列表分页）的 offset/limit 性能无帮助**

### 13.8 可行的优化方向（代码中未实现）

| 优化 | 实现思路 | 对大 offset 的改进 | 开发成本 |
|-----|---------|------------------|---------|
| **增加复合索引** | `CREATE INDEX idx_media_album_date ON media(album_id, date_shot DESC)` | 消除 filesort，但 OFFSET 仍线性 | 低（一次 migration） |
| **增加表达式索引** | Timeline 专用函数索引（见 13.4 B） | Timeline 从全表排序变 Index Scan | 中（需按 DB 适配） |
| **Keyset Pagination** | 用 `WHERE (date_shot, id) < (last_date, last_id) ORDER BY date_shot DESC LIMIT 200` 替代 OFFSET | **O(log N) + O(LIMIT)**，与 offset 无关 | 高（需改 Schema、Resolver、前端） |
| **Limit+1 预估 hasNext** | `Limit(limit+1)` 查询，判断返回 `len > limit` → 告诉前端有下一页 | 前端可提前停止，减少无效请求 | 低（无需 Schema 改动，内部优化） |
| **offset 上限校验** | `if *paginate.Offset > 50000 { return Err }` | 防止 DoS 级大 offset | 极低 |
| **列裁剪 SELECT** | 从 gqlgen 的 `CollectFields` 结果构建 `db.Select(...)` | 减少内存消耗和磁盘 IO | 中（需字段到列的映射维护） |
| **只请求 count** | 首次请求额外可选 COUNT(*) | 前端可显示进度条，减少用户盲目翻页 | 低 |

---

## 十四、底层存储全表扫描路径追踪（以 Album.media + 大 offset 为例）

以以下真实查询为例：

```graphql
query {
  album(id: 42) {
    media(order: { order_by: "date_shot", order_direction: DESC },
          paginate: { limit: 200, offset: 40000 }) {
      id title thumbnail { url }
    }
  }
}
```

### 14.1 GORM 到 PostgreSQL 的完整追踪

```
[Go 代码层]
  albumResolver.Media(ctx, album(42), order, paginate(limit=200, offset=40000), nil)
  │
  │  1. db.Where("media.album_id = ?", 42)
  │  2. db.Where("media.id IN (SELECT media_id FROM media_urls WHERE media_id = media.id)")
  │        ↑ 保证至少有一个 MediaURL（可显示的有效图片）
  │  3. models.FormatSQL(query, order, paginate)
  │     → query.Limit(200).Offset(40000)
  │     → query.Order(clause.OrderByColumn{Column: clause.Column{Name:"date_shot"}, Desc:true})
  │  4. query.Find(&media)
  ▼
[GORM Statement 层]
  Statement.SQL 字符串构建
  │
  │  SELECT * FROM "media"
  │  WHERE media.album_id = $1
  │    AND media.id IN (SELECT media_id FROM media_urls WHERE media_id = media.id)
  │  ORDER BY "date_shot" DESC
  │  LIMIT $2 OFFSET $3
  │
  │  参数绑定: [$1=42, $2=200, $3=40000]
  ▼
[lib/pq 驱动层]
  发送 PostgreSQL 二进制协议 (Extended Query)
  │  Parse  → "SELECT * FROM media WHERE ... ORDER BY date_shot DESC LIMIT $1 OFFSET $2"
  │  Bind   → [200, 40000, 42] （参数重新排序）
  │  Execute
  ▼
[PostgreSQL 服务器层 - Backend Process]
  │
  ├─ Parser: 解析 SQL，生成 Parse Tree
  ├─ Analyzer: 语义分析，检查权限、类型
  ├─ Rewriter: 应用规则（无影响）
  ├─ Planner:
  │   ├─ 读取 pg_stats 统计信息
  │   ├─ 评估 3 种计划：
  │   │   Plan A: Seq Scan + Sort
  │   │      → rows=50000 (album_id=42 的估算)
  │   │      → cost=50000*2.5 (seq_page_cost) + 50000*log(50000)*cpu_operator_cost
  │   │      → total_cost ≈ 125000 + 50000*16*0.0025 ≈ 125000 + 2000 = 127000
  │   │   Plan B: Index Scan using idx_media_album_id + Sort
  │   │      → Bitmap Heap Scan 读取 50000 行（随机 IO 代价高）
  │   │      → Sort 50000 行
  │   │      → total_cost ≈ 50000*4.0 + 2000 = 202000（比 A 更差）
  │   │   Plan C: 直接走 date_shot 上的索引（但 date_shot 无索引，跳过）
  │   └─ 选定 Plan A: Seq Scan on media → Sort → Limit
  │
  ├─ Executor:
  │   ├─ 节点 1: Seq Scan on media
  │   │   ├─ 遍历 1,000,000 个页面（假设 media 表 100 万行）
  │   │   ├─ 对每一行检查 album_id = 42 子句
  │   │   └─ 通过后再检查 media_id IN (media_urls 子查询)
  │   │   └─ 输出 → 约 50,000 行匹配
  │   │
  │   ├─ 节点 2: Sort (Sort Method: quicksort, memory: 12000kB)
  │   │   ├─ 读取 50,000 行到内存 tuplesort
  │   │   ├─ 执行快速排序按 date_shot DESC
  │   │   ├─ 排序完成，写回 sorted tuples 数组
  │   │   └─ 内存不够则溢出到磁盘临时文件 merge sort
  │   │
  │   └─ 节点 3: Limit
  │        ├─ 创建 cursor 指向 sorted[0]
  │        ├─ FORWARD 40,000 个位置 → 逐个跳过（内部 goto，仍需寻址）
  │        └─ 读取下一个 200 条 → 返回给客户端
  │
  └─ 统计信息输出:
       - total_exec_time: ≈ 200-400ms
       - buffers_read: ≈ 12,000 (主要是 media 表的页面)
       - sort_mem_used: ≈ 10MB
       - rows_returned: 200

[网络传输层]
  200 行 * (每行列数据平均 ~2KB，含 Blurhash 等字段) ≈ 400KB payload
  → PostgreSQL → Go → GraphQL JSON 序列化 → HTTP Response → 前端
```

### 14.2 如果有了 (album_id, date_shot DESC) 复合索引后

```
Planner 选择:
  Index Scan Backward using idx_media_album_date on media
    Index Cond: (album_id = 42)
    → 直接从索引顺序读取，无需 Sort 节点

执行路径对比：
  节点 1: Index Scan Backward (idx_media_album_date)
    ├─ 定位索引叶节点 (42, max_date)
    ├─ 沿索引链表反向（DESC）读取
    ├─ 每条记录直接查回主表（索引包含 album_id 和 date_shot，但返回 SELECT * 需回表）
    ├─ 每行额外做 media_urls 子查询检查
    ├─ FORWARD 40,000 条（沿索引链表逐个跳过 → 仍有开销但比排序快很多）
    └─ 取 200 条

收益：
  - 消除 50,000 行 Sort（O(N log N) 变为 O(1) 定位）
  - 大 offset 从 "排序 + 跳过" 变为 "索引跳过 + 回表"
  - 实际时间从 200-400ms 降至 50-80ms
  - 内存：不再需要 Sort Buffer
```

### 14.3 SQLite 的全表扫描实现差异

```
SQLite 执行路径（VDBE 虚拟机字节码）：
  OpenRead 0 → 打开 media 表 B-Tree 根页
  OpenRead 1 → 打开 idx_media_album_id 索引（若用 album_id）

  方案 A（无 date_shot 索引）：
    Rewind 0 → 定位表开头
    Loop:  // 全表扫描
      Column 0, album_id → 比较 = 42
      若匹配 → 复制到 ephemeral B-Tree (临时表，以 date_shot DESC 为键)
      Next 0 → 下一行，直到 EOF

    // 临时表构建完毕（含 50,000 条）
    OpenEphemeral 2 → 打开排序后的临时表
    Limit 40000 → 跳过 40,000 个 B-Tree 条目（向下 seek）
    Loop 200:
      Column → 构建输出行
      Next

  方案 B（有复合索引）：
    OpenRead 1 (idx_media_album_date)
    SeekGE 1 (42, MAXINT64) → 定位到 (42, 最大日期)
    Limit 40000 → Next 1 × 40000 次（沿 B-Tree Leaf 链表跳）
    Loop 200:
      IdxRowid → 回表取完整行
      Next 1

关键区别：
  - SQLite 用临时 B-Tree 做排序，PostgreSQL 用 quicksort/mergesort
  - SQLite 在 OFFSET 时也是真的 Next 40000 次（但 B-Tree seek 比数组遍历略高效）
  - SQLite 默认 page cache = 2000 页 → 大表扫描时频繁换入换出
```

### 14.4 "SELECT *" 加剧大 offset 的成本

所有 Resolver 都使用默认的 `query.Find(&media)`，即 **SELECT 全部列**：

```go
// media.go 模型
type Media struct {
    Model           // ID (int) + CreatedAt + UpdatedAt → ~32 bytes
    Title    string // ~100 bytes
    Path     string // ~500 bytes
    PathHash string // 32 bytes (MD5 hex)
    AlbumID  int
    DateShot time.Time // 8 bytes
    Type     MediaType // ~10 bytes
    Blurhash *string // ~256 bytes 字符串（base64）
    // ... 还有若干指针列、关联列
}
// 每行总计 ≈ 1000-2000 bytes（取决于 Path 长度和 Blurhash 存在性）
```

**含义**：
- `OFFSET 40000 LIMIT 200` 时，即使最终只返回 200 行给客户端，数据库在 Sort 阶段也**必须处理 50,000 行的完整列**（200 bytes × 50,000 = 10MB 数据要排序）
- 加上 Blurhash 列（base64 字符串 ~256 字节），sort buffer 需要额外 12.5MB
- 如果字段裁剪实现为 `db.Select("id", "title", "date_shot")` → **排序时处理的数据量可缩小到 1/10**，显著加速
- 但 gqlgen `CollectFields` 仅调度 Resolver，不会把列名回传到 FormatSQL → 需要额外开发

### 14.5 全表扫描触发条件总结

| 查询场景 | 是否触发全表排序 | 原因 | 可被优化的方向 |
|---------|---------------|------|-------------|
| `Album.media` 按 `date_shot` 排序 | ✅ 100% 触发 | album_id 单索引不足以提供排序 | 加复合索引 |
| `Album.media` 按 `title` 排序 | ✅ 100% 触发 | 无 title 索引 | 加 (album_id, title) 索引 |
| `Album.media` 按 `id` 排序 | ⚠️ 可走主键索引，但前端默认不按 id | 主键天然有序 | 无需优化 |
| `myMedia` 任意排序 | ✅ 100% 触发 | user_albums 子查询 + JOIN，不连贯 | 加 (id, date_shot) 索引 + 改查询 |
| `myTimeline` | ✅ 100% 触发 | DATE_TRUNC 函数调用，列不原生排序 | 表达式索引 |
| `myAlbums` | ⚠️ albums 行数少（通常 <1000），影响可忽略 | 虽无索引但数据量小 | 无需优化 |
| `myFaceGroups` | ⚠️ 同上，通常 <1000 组 | 用 COUNT 排序但量小 | 无需优化 |

---

## 十五、GORM Session 与事务隔离：按代码顺序拆解

### 15.1 GORM DB 实例从启动到 Resolver 的完整生命周期

```
[进程启动阶段] main.go / server setup
  │
  │  SetupDatabase()   [database.go:113-152]
  │    │
  │    ├─ gorm.Config{}  ← 空配置，无自定义 Session
  │    │   └─ Logger: 根据开发模式决定 Info/Warn
  │    │
  │    ├─ ConfigureDatabase(&config)  [database.go:77-110]
  │    │   ├─ drivers.DatabaseDriverFromEnv()  → 读 PHOTOVIEW_DATABASE_DRIVER
  │    │   │   → mysql / sqlite / postgres 三选一
  │    │   │
  │    │   ├─ [mysql] GetMysqlAddress() → 解析 DSN
  │    │   │     └─ config.MultiStatements = true
  │    │   │        config.ParseTime = true
  │    │   │        ↑ 注意：未设置 parsetime 默认事务隔离级别
  │    │   │        ↑ 默认用 go-sql-driver/mysql 默认 = REPEATABLE-READ
  │    │   │
  │    │   ├─ [sqlite] GetSqliteAddress()  [database.go:53-75]  ← 关键配置
  │    │   │     ├─ _journal_mode = WAL        (Write-Ahead Logging)
  │    │   │     ├─ _locking_mode = NORMAL     (读写并发)
  │    │   │     ├─ _foreign_keys = ON
  │    │   │     ├─ cache = shared
  │    │   │     └─ mode = rwc                 (read-write-create)
  │    │   │
  │    │   ├─ [postgres] GetPostgresAddress() → url.Parse，无额外参数
  │    │   │     → 完全由 DSN 中的 sslmode、options 决定
  │    │   │     → 未指定 default_transaction_isolation
  │    │   │
  │    │   └─ gorm.Open(dialector, config)  → 返回 *gorm.DB 单例
  │    │         ↑ 此实例内部持有：连接池 + Config + 空 Statement
  │    │
  │    ├─ sqlDB.SetMaxOpenConns(80)   [database.go:140]
  │    │     → 全局连接池最大 80 个连接
  │    │     → 每个查询独立获取连接，用完归还
  │    │
  │    └─ 返回 *gorm.DB 实例
  │
  │  NewRootResolver(db)  [resolver.go:15-19]
  │    └─ Resolver { database: db }  ← 保存单例引用
  ▼
[HTTP 请求阶段] GraphQL 请求到达
  │
  │  Auth Middleware  [auth.go:31-70]
  │    └─ r.WithContext(ctx)  ← 注入 User 到 context，无 DB 变更
  │
  │  Dataloader Middleware  [loaders.go:35]
  │    └─ r.WithContext(ctx)  ← 注入 Dataloaders，无 DB 变更
  │
  ▼
[Resolver 执行阶段]
  │
  │  albumResolver.Media(ctx, obj, order, paginate, onlyFavorites)
  │    │
  │    ├─ r.DB(ctx)   [resolver.go:22-24]
  │    │    │
  │    │    │    // 代码：
  │    │    │    // func (r *Resolver) DB(ctx context.Context) *gorm.DB {
  │    │    │    //     return r.database.WithContext(ctx)
  │    │    │    // }
  │    │    │
  │    │    └─ r.database.WithContext(ctx)  ← ★ 关键点 1
  │    │         │
  │    │         │  GORM 内部实现 (gorm.io/gorm v1.31.1)：
  │    │         │  func (db *DB) WithContext(ctx context.Context) *DB {
  │    │         │      if ctx == nil {
  │    │         │          return db
  │    │         │      }
  │    │         │      // 克隆一个新 Session（浅拷贝 Statement，不共享内存）
  │    │         │      tx := db.getInstance()
  │    │         │      tx.Statement.Context = ctx
  │    │         │      return tx
  │    │         │  }
  │    │         │
  │    │         └─ 返回**新克隆**的 *gorm.DB，ctx 已绑定
  │    │              → 每个 Resolver 拿到独立的 DB 实例
  │    │              → 但底层共享同一个 sql.DB 连接池
  │    │
  │    ├─ query := r.DB(ctx).Where(...).Where(...)
  │    │    │
  │    │    │  每次链式调用都会内部再次克隆：
  │    │    │  func (db *DB) Where(query interface{}, args ...interface{}) (tx *DB) {
  │    │    │      tx = db.getInstance()   // 每次克隆
  │    │    │      tx.Statement.AddClause(clause.Where{...})
  │    │    │      return
  │    │    │  }
  │    │    │
  │    │    └─ 产生一个独立 query（Statement 独立）
  │    │
  │    ├─ models.FormatSQL(query, order, paginate)   [utils.go:11-40]
  │    │    ├─ tx.Limit(*paginate.Limit)    → 克隆 + 设置 Clause.LIMIT
  │    │    ├─ tx.Offset(*paginate.Offset)  → 克隆 + 修改 Clause.LIMIT.Offset
  │    │    └─ tx.Order(clause.OrderByColumn{...})  → 克隆 + 设置 Clause.ORDER BY
  │    │
  │    └─ query.Find(&media)   [GORM 内部]
  │         │
  │         ├─ 1. 从连接池获取 sql.Conn（可能新建，可能复用）
  │         │      → sqlDB.Conn(ctx) 阻塞直到拿到连接
  │         │      → ★ 关键点 2：没有 BEGIN，是 auto-commit 模式！
  │         │
  │         ├─ 2. db.Statement.Build("SELECT")
  │         │      → 组装 SQL 字符串（参数化占位符）
  │         │
  │         ├─ 3. stmt, err := tx.Statement.ConnPool.PrepareContext(ctx, sql)
  │         │      → DB 驱动层预编译
  │         │
  │         ├─ 4. rows, err := stmt.QueryContext(ctx, vars...)
  │         │      → 发送到数据库执行
  │         │      → ★ 关键点 3：单条 SELECT，无事务包裹
  │         │
  │         └─ 5. rows.Close() → 归还连接到连接池
  ▼
[后续字段解析] gqlgen 调度子 Resolver
  │
  │  mediaResolver.Thumbnail(ctx, media)
  │    ├─ r.DB(ctx)   ← 再次 WithContext，又一个独立 *gorm.DB
  │    └─ Dataloader 批量查 media_urls
  │         └─ 单独获取连接，单独执行 SQL
  │
  └─ 同一张表的两次查询可能拿到**不同的连接池连接**
      → 完全独立的执行上下文
      → 无法看到彼此的未提交写入（但所有写入都已提交，因为 auto-commit）
```

### 15.2 Session 隔离分析：每个 Resolver 的独立性

| 概念 | 实际情况 | 对分页一致性的影响 |
|-----|---------|------------------|
| **GORM *gorm.DB 实例共享性** | 每个 `WithContext` + 链式调用都克隆 | ✅ 无数据竞争（独立 Statement） |
| **Statement 内存共享** | 每个查询独立 clone | ✅ 不会互相污染 WHERE/LIMIT/OFFSET |
| **SQL 连接共享** | 从 80 连接池随机获取 | ❌ 两次查询可能不同连接，MVCC 视图不同 |
| **显式事务** | 分页查询均无，完全 auto-commit | ❌ 每条 SELECT 是独立快照 |
| **事务隔离级别** | 使用各数据库驱动默认 | ❌ 差异大（详见 15.3） |
| **跨 Resolver 一致性** | 完全无保障 | ❌ 翻第一页和第二页的可见性可能不一致 |

### 15.3 三种数据库驱动的默认事务隔离级别对比

#### A. PostgreSQL（默认 READ COMMITTED）

驱动：`gorm.io/driver/postgres v1.6.0` → 底层 `github.com/jackc/pgx/v5`

```go
// PostgreSQL 默认行为（未在 DSN 中指定 transaction_isolation）：
//   数据库默认 SHOW default_transaction_isolation = 'read committed'
//
// 对单条 SELECT（auto-commit）的含义：
//   1. 语句开始时获取快照
//   2. 语句执行期间只能看到语句开始前已提交的数据
//   3. 同一会话的下一条 SELECT 获取新快照
//
// 对分页的影响（最常见）：
//   T0: 用户点击下一页，前端 fetchMore(offset=200)
//   T1: 第 1 页 SELECT (offset 0-199) 开始 → 快照 S1
//   T2: Scanner 提交了新照片（已插入且 COMMIT）
//   T3: 第 2 页 SELECT (offset 200-399) 开始 → 快照 S2
//        ↑ S2 能看到 T2 的新插入，S1 看不到
//        → 如果新照片排序在 199 和 200 之间
//        → 第 2 页第 1 条是原第 199 条（重复）
//
// READ COMMITTED 下，同一翻页会话的两次请求**必然**不同快照
```

#### B. MySQL InnoDB（默认 REPEATABLE READ）

驱动：`gorm.io/driver/mysql v1.6.0` → 底层 `github.com/go-sql-driver/mysql v1.10.0`

```go
// MySQL 连接配置 [database.go:24-38]：
func GetMysqlAddress(addressString string) (string, error) {
    config, _ := mysql.ParseDSN(addressString)
    config.MultiStatements = true   // 允许多语句
    config.ParseTime = true         // 时间解析
    // ★ 未设置 config.Params["transaction_isolation"]
    // → 使用 MySQL Server 默认
    // → MySQL 8.0 默认 transaction_isolation = REPEATABLE-READ
}

// MySQL InnoDB 的 REPEATABLE READ 特性：
//   ★ 但这仅在**显式 START TRANSACTION**内生效！
//   ★ auto-commit 模式下的单条 SELECT：
//     - 行为与 READ COMMITTED 几乎相同
//     - 每条语句都获取新的 ReadView（读视图）
//     - 因为没有 BEGIN，InnoDB 不启动一致性快照
//
// 对分页的影响：
//   与 PostgreSQL READ COMMITTED 完全相同
//   → 翻到下一页时可能看到第一页之后提交的写入
//   → 重复/遗漏同样可能发生
//
// 额外风险：config.MultiStatements = true
//   → 虽不直接影响 OFFSET 一致性，但放宽了驱动对 SQL 注入的限制
//   → 若 ORDER BY 被注入，攻击者可构造多语句
```

#### C. SQLite（WAL 模式 + NORMAL 锁）

驱动：`gorm.io/driver/sqlite v1.6.0` → 底层 `github.com/mattn/go-sqlite3`

```go
// SQLite 关键配置 [database.go:63-70]：
queryValues.Add("cache", "shared")         // 多连接共享页缓存
queryValues.Add("mode", "rwc")             // read-write-create
queryValues.Add("_journal_mode", "WAL")    // Write-Ahead Logging ← 关键
queryValues.Add("_locking_mode", "NORMAL") // 读写并发 ← 关键
queryValues.Add("_foreign_keys", "ON")

// SQLite WAL 模式下的并发行为：
//   1. 写操作写入 .wal 文件，不阻塞读
//   2. 读操作读主库 + .wal 中已提交的页
//   3. 每个读连接看到的是连接开始时的快照
//   4. 每次新获取连接（从 Go sql.DB 连接池）→ 新快照
//
// SQLite 的隔离级别是 SERIALIZABLE，但只对"连接内"有保证
//   由于连接池的存在，第 1 页和第 2 页可能使用不同连接
//   → 看到不同的 .wal 快照
//
// 对分页的影响：
//   与 PG/MySQL 相同：翻下一页可能看到扫描期间新增的写入
//   但 SQLite 的写性能瓶颈明显（CHECKPOINT 时可能短暂阻塞读）
//   → 大规模并发翻页场景下反而比 PG/MySQL 更差
//
// 额外注意：[scanner.go:87-89] 明确限制
//   if workers > 1 && drivers.DatabaseDriverFromEnv() == drivers.SQLITE {
//       return 0, errors.New("multiple workers not supported for SQLite databases")
//   }
//   → SQLite 仅允许 1 个 Scanner worker 串行写
//   → 减少了并发写竞争，但不消除读不一致
```

### 15.4 各驱动对并发增删 OFFSET 一致性的对比总结

| 特性 | PostgreSQL | MySQL InnoDB | SQLite WAL |
|-----|-----------|-------------|-----------|
| 驱动默认隔离 | READ COMMITTED | REPEATABLE READ（仅显式事务内） | SERIALIZABLE（仅连接内） |
| auto-commit 单条 SELECT 实际隔离 | READ COMMITTED | READ COMMITTED 级别读视图 | 连接级快照 |
| 翻两页是否同一事务 | ❌ 不是 | ❌ 不是 | ❌ 可能是也可能不是（连接池） |
| 翻两页是否同一连接 | ❌ 几乎必然不同 | ❌ 几乎必然不同 | ⚠️ 可能相同（连接池空闲少） |
| 是否可看到翻页期间新提交的写入 | ✅ 第 2 页必然可见 | ✅ 第 2 页必然可见 | ✅ 新连接可见 |
| 重复/遗漏风险 | 中 | 中 | 低（串行 Scanner 写入少） |
| 大 offset 与并发写叠加 | 排序+丢弃期间写提交可能改变顺序 | 同 PG | WAL checkpoing 期间锁竞争 |
| 是否可用 REPEATABLE READ 统一保护 | ✅ `BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ` | ✅ `SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ` | ✅ 但 SQLite 连接复用困难 |
| 项目代码是否实现上述保护 | ❌ 未实现 | ❌ 未实现 | ❌ 未实现 |

### 15.5 现有 Session 配置的使用位置

全代码库仅有 **2 处显式 Session 配置**，且都在**写操作**中，与分页查询无关：

```go
// scanner.go:61-65（SetPeriodicScanInterval）
db.
    Session(&gorm.Session{AllowGlobalUpdate: true}).  // 允许不带 WHERE 的全局 UPDATE
    Model(&models.SiteInfo{}).
    Update("periodic_scan_interval", interval)

// scanner.go:91-95（SetScannerConcurrentWorkers）
db.
    Session(&gorm.Session{AllowGlobalUpdate: true}).
    Model(&models.SiteInfo{}).
    Update("concurrent_workers", workers)
```

→ 分页查询从未使用 `Session()` 配置，完全使用 GORM 默认。

---

## 十六、sort-by 排序键注入到查询的完整代码生成路径

### 16.1 端到端链路追踪：从浏览器下拉框到 SQL ORDER BY

```
[前端 UI 层] AlbumFilter.tsx:68-88
  │
  │  // 排序选项硬编码白名单（TS 类型约束）
  │  export type SortingOptionValue = 'date_shot' | 'updated_at' | 'title' | 'type'
  │
  │  const defaultOptions = [
  │    { value: 'date_shot',   label: 'Date shot'   },
  │    { value: 'updated_at',  label: 'Date imported' },
  │    { value: 'title',       label: 'Title'        },
  │    { value: 'type',        label: 'Kind'         },
  │  ]
  │
  │  changeOrderBy = (value: SortingOptionValue) => {
  │    setOrdering({ orderBy: value })  // ← TS 类型检查：只能是 4 个值之一
  │  }
  ▼
[前端 URL 状态层] useOrderingParams.ts:15-47
  │
  │  // orderBy 从 URL query 参数读取
  │  const rawOrderBy = getParam('orderBy', defaultOrderBy)
  │  // ↑ ★ 关键安全隐患 1：URL 参数是完全开放的字符串，不经过 TS 类型检查
  │  //   用户可以手动编辑 URL：?orderBy=(SELECT+password+from+users)
  │
  │  const orderBy = rawOrderBy === null || rawOrderBy === ''
  │    ? defaultOrderBy   // 空值时回退到 'date_shot'
  │    : rawOrderBy       // ← 其他字符串原样保留！
  │
  │  // orderDirection 有枚举校验，但 orderBy 完全没有白名单校验
  │  const orderDirection =
  │    Object.values(OrderDirection).includes(rawOrderDir)
  │      ? rawOrderDir
  │      : OrderDirection.ASC
  ▼
[前端 GraphQL 变量] AlbumGallery.tsx:22-34
  │
  │  query AlbumPage($mediaOrderBy: String) {
  │    album(id: $id) {
  │      media(order: { order_by: $mediaOrderBy, order_direction: $orderDirection }) {
  │        ...
  │      }
  │    }
  │  }
  │
  │  // $mediaOrderBy 的类型是 String！（不是 Enum）
  │  // ↑ ★ 关键安全隐患 2：GraphQL Schema 中 order_by 是 String 类型
  │  //   无 Schema 层白名单
  ▼
[GraphQL 协议层] → HTTP POST JSON payload
  │
  │  {
  │    "variables": {
  │      "mediaOrderBy": "(SELECT password FROM users WHERE admin=true LIMIT 1)"
  │    }
  │  }
  ▼
[gqlgen 参数解析层] generated.go:9212-9247
  │
  │  func unmarshalInputOrdering(ctx, obj any) (models.Ordering, error) {
  │    // ...
  │    case "order_by":
  │      data, err := ec.unmarshalOString2ᚖstring(ctx, v)
  │      // ↑ 无任何校验：v 是任意 string，直接转为 *string
  │      it.OrderBy = data
  │    // ...
  │  }
  │
  │  // unmarshalOString2ᚖstring [generated.go:12865-12876]
  │  func (ec *executionContext) unmarshalOString2...(ctx, v any) (*string, error) {
  │    if v == nil { return nil, nil }
  │    res, err := graphql.UnmarshalString(v)
  │    // ↑ gqlgen 内置：只确保是合法的 JSON string 字面量
  │    // ↑ 不检查长度、不检查字符集、不做 SQL 关键字校验
  │    return &res, ...
  │  }
  ▼
[Resolver 层] album.go:21-51（albumResolver.Media）
  │
  │  func (r *albumResolver) Media(ctx, obj, order *models.Ordering, paginate, onlyFavorites) {
  │    // order 参数是 gqlgen 解析后的 models.Ordering
  │    // order.OrderBy = *string（可能是任意字符串）
  │    // ↑ 无任何白名单校验：直接透传
  │    query := db.Where(...)
  │    // ...
  │    query = models.FormatSQL(query, order, paginate)
  │    // ...
  │  }
  ▼
[FormatSQL 层] utils.go:11-40
  │
  │  func FormatSQL(tx *gorm.DB, order *Ordering, paginate *Pagination) *gorm.DB {
  │    if order != nil && order.OrderBy != nil {
  │      // ... 解析 orderDirection（此处有枚举校验 .IsValid()）
  │      desc := ...
  │
  │      tx.Order(clause.OrderByColumn{
  │        Column: clause.Column{
  │          Name: *order.OrderBy,   // ← ★ 关键点：注入点
  │          // *order.OrderBy 是任意字符串，直接写入 clause.Column.Name
  │        },
  │        Desc: desc,
  │      })
  │    }
  │    return tx
  │  }
  ▼
[GORM Clause 构建层] gorm.io/gorm/clause（v1.31.1 内置）
  │
  │  // clause.Column 的 Build 方法（GORM 内部）：
  │  func (col Column) Build(builder Builder) {
  │    if col.Table != "" {
  │      builder.Write(quoteChar)
  │      builder.WriteString(col.Table)
  │      builder.Write(quoteChar)
  │      builder.WriteByte('.')
  │    }
  │    if col.Name == "*" {
  │      builder.WriteByte('*')
  │    } else {
  │      builder.Write(quoteChar)        // ← 添加 " 或 ` 取决于数据库
  │      builder.WriteString(col.Name)   // ← ★ 安全边界：原样写入 col.Name
  │      builder.Write(quoteChar)        // ← 添加闭合引号
  │      // ↑ 这是 GORM 的主要防注入手段：把列名包在引号中
  │    }
  │    // ... 处理 Alias 等
  │  }
  │
  │  // 最终 SQL 形如（PostgreSQL）：
  │  //   ORDER BY "date_shot" DESC
  │  //
  │  // 如果用户传入：date_shot; DROP TABLE media; --
  │  // 结果：ORDER BY "date_shot; DROP TABLE media; --" DESC
  │  //        ↑ 引号将其整体括起，被当作列名
  │  //        ↑ 不会真正执行 DROP TABLE
  │
  │  // quoteChar 选择：
  │  //   PostgreSQL: "  （双引号标识符）
  │  //   MySQL:      `  （反引号标识符）
  │  //   SQLite:     "  （双引号，兼容 SQL 标准）
  │
  ▼
[数据库最终执行 SQL]
  │
  │  -- 正常请求 date_shot：
  │  SELECT * FROM "media"
  │    WHERE "media"."album_id" = 42
  │    ORDER BY "date_shot" DESC
  │    LIMIT 200 OFFSET 0
  │
  │  -- 恶意输入：(CASE WHEN (SELECT count(*)>0 FROM users WHERE password LIKE 'a%') THEN date_shot ELSE id END)
  │  SELECT * FROM "media"
  │    WHERE "media"."album_id" = 42
  │    ORDER BY "(CASE WHEN (SELECT count(*)>0 FROM users WHERE password LIKE 'a%') THEN date_shot ELSE id END)" DESC
  │    ↑ 被整体引号包裹，数据库会尝试以此为列名查找
  │    ↑ 通常报: column "(CASE ...)" does not exist → **不会真正执行子查询**
  │
  │  -- 但如果攻击者用 " 闭合引号呢？
  │  -- 输入：date_shot" DESC; DROP TABLE media; --
  │  SELECT * FROM "media"
  │    WHERE ...
  │    ORDER BY "date_shot" DESC; DROP TABLE media; --" DESC
  │    ↑ PostgreSQL: 单条 query 只执行第一个分号前的语句
  │    ↑ MySQL: 如果 MultiStatements=true（本项目开启了！）则会执行多语句！
  │    ↑ SQLite: 通常不支持多语句（但 Mattn 驱动有编译选项）
  ▼
[安全防线总结]
  │
  ├─ 防线 1（前端 TS 类型）: SortingOptionValue 联合类型
  │   → 绕过：编辑 URL 参数、直接发 GraphQL 请求
  │   → 强度：弱（仅 UI 提示，非强制）
  │
  ├─ 防线 2（前端 GraphQL 类型）: String 而非 Enum
  │   → 绕过：GraphQL 类型系统允许任意字符串
  │   → 强度：无
  │
  ├─ 防线 3（gqlgen unmarshal）: 仅验证 JSON string 字面量
  │   → 绕过：任意字符串字面量都通过
  │   → 强度：无
  │
  ├─ 防线 4（后端 Resolver）: 无白名单校验
  │   → 绕过：直接传递
  │   → 强度：无
  │
  ├─ 防线 5（GORM clause.Column 引号转义）: 将 Name 包在 " 或 ` 中
  │   → 绕过：闭合引号 + 特殊情况（见 16.2）
  │   → 强度：中（PG/SQLite 较好，MySQL 有风险）
  │
  └─ 最终防线（数据库驱动）: 单语句限制
      → 绕过：MySQL MultiStatements=true（本项目开启）
      → 强度：PG: 强；MySQL: 弱；SQLite: 中
```

### 16.2 注入风险分析：三种数据库分别评估

#### A. PostgreSQL

```
clause.Column 引号策略: " (双引号，SQL 标准标识符)

能否绕过引号执行子查询？
  例：输入 →  id" DESC, (SELECT CASE WHEN current_user='postgres' THEN 1 ELSE 2 END)
  结果 →  ORDER BY "id" DESC, (SELECT CASE WHEN current_user='postgres' THEN 1 ELSE 2 END)"
          ↑ 引号在最后，(SELECT...) 仍在引号内，被当作标识符
          ↑ 报：column "id DESC, (SELECT ...)" does not exist
          → **不能执行任意子查询**

能否用分号执行多语句？
  pg_query 协议层面：一次 Parse 只允许 1 条语句
  → 即使传入分号，驱动会报：cannot insert multiple commands into a prepared statement
  → **PG 下安全**

实际可行的注入（仍是 ORDER BY 列名层面）：
  输入 →  CASE WHEN EXISTS(SELECT 1 FROM users WHERE username='admin' AND substr(password,1,1)='a') THEN id ELSE date_shot END
  → 被括成 "CASE WHEN ... END"，当作列名
  → 报错，无法盲注

结论：PostgreSQL 下由于引号包裹 + 单语句限制，注入风险**极低**
```

#### B. MySQL（⚠️ 本项目开启了 MultiStatements=true）

```
clause.Column 引号策略: ` (反引号)

关键：database.go:34 → config.MultiStatements = true

注入示例 1：闭合引号 + 分号 + DROP
  输入 →  date_shot` DESC; DROP TABLE media; --
  GORM 生成 SQL：
    ORDER BY `date_shot` DESC; DROP TABLE media; --` DESC
  ↑ MultiStatements=true → MySQL 驱动将分号拆分执行 3 条：
    1. SELECT ... ORDER BY `date_shot` DESC  → 正常（无错误）
    2. DROP TABLE media                      → ★ 实际删除！
    3. --` DESC                               → 注释，忽略
  → **高危：可真执行任意 DDL/DML！**

注入示例 2：基于布尔盲注
  输入 →  id` DESC, IF((SELECT COUNT(*) FROM users)=1, SLEEP(5), 0) --
  → 第 2 条 IF(...) 在 ORDER BY 表达式中计算
  → 如果用户表有 1 条记录，查询延迟 5 秒（盲注成功）

结论：MySQL MultiStatements=true 情况下，注入风险**极高**
```

#### C. SQLite

```
clause.Column 引号策略: " (双引号，SQL 标准)

SQLite 默认驱动（mattn/go-sqlite3）：
  - 是否支持多语句取决于编译选项（通常开启）
  - 但 SQLite 的 DROP TABLE 需要写锁
  - 结合 WAL 模式，写锁只在 COMMIT 时需要

注入分析：
  与 PG 类似，引号整体包裹 → 子查询不会真正执行
  多语句：若驱动允许且 WAL 锁竞争少，可能注入成功

实际风险：SQLite 通常单用户本地部署
  → 攻击者与合法用户是同一人
  → 即使注入成功，也是删自己的数据
  → 风险**中**（非零但场景有限）
```

### 16.3 Timeline 的特殊情况：内置排序无注入风险

`myTimeline` 的排序是**硬编码的字符串**，不接受 order 参数：

```go
// timeline_actions.go:20-40
switch drivers.GetDatabaseDriverType(db) {
case drivers.POSTGRES:
    query = query.
        Order("DATE_TRUNC('year', date_shot) DESC").    // 硬编码字符串
        Order("DATE_TRUNC('month', date_shot) DESC").   // 不经过 clause.Column
        Order("DATE_TRUNC('day', date_shot) DESC").
        Order("albums.title ASC").
        Order("media.date_shot DESC")
case drivers.SQLITE:
    query = query.
        Order("strftime('%Y-%m-%d', media.date_shot) DESC").
        Order(albumsTitleASC).
        Order("TIME(media.date_shot) DESC")
// ...
}
query = models.FormatSQL(query, nil, paginate)  // order 参数传入 nil
```

**分析**：
- `query.Order("DATE_TRUNC('year', date_shot) DESC")` 使用的是 GORM 的 `Order(string)` 重载
- 这是一个**原始 SQL 字符串**，不经过 `clause.Column`，直接拼接到 ORDER BY
- 但由于是硬编码（无变量插值），完全安全
- 然而这也意味着：**如果 order 参数被错误传入，会有额外风险**（当前 FormatSQL 当 order=nil 时跳过，所以安全）

### 16.4 Search 的特殊情况：使用 clause.Expr 直接拼 SQL

```go
// search_actions.go:39-44
Clauses(clause.OrderBy{
    Expression: clause.Expr{
        SQL: "(CASE WHEN LOWER(media.title) LIKE ? THEN 2 WHEN LOWER(media.path) LIKE ? THEN 1 END) DESC",
        Vars:               []interface{}{wildQuery, wildQuery},
        WithoutParentheses: true,
    },
})
```

**分析**：
- 使用 `clause.Expr`（表达式）而非 `clause.OrderByColumn`
- `SQL` 是硬编码字符串，`Vars` 通过参数化绑定（`?` 占位符）
- **安全**：wildQuery 中的特殊字符会被驱动作为参数处理，不会解析为 SQL 关键字
- 但如果有人把 user input 直接拼进 `SQL` 字段（而不是用 Vars），就会注入

### 16.5 注入风险矩阵汇总

| 查询 | order_by 来源 | 注入风险 | 原因 |
|-----|-------------|---------|------|
| `Query.myAlbums(order)` | 前端变量 → `clause.OrderByColumn` | ⚠️ PG: 低；MySQL: **高**；SQLite: 中 | 见 16.2 |
| `Query.myMedia(order)` | 同上 | ⚠️ 同上 | 同上 |
| `Album.media(order)` | 同上 | ⚠️ 同上 | 同上 |
| `Album.subAlbums(order)` | 同上 | ⚠️ 同上 | 同上 |
| `ShareAlbum.media(order_by)` | 同上 | ⚠️ 同上 | 同上 |
| `Query.myTimeline` | **硬编码**，order 参数传入 nil | ✅ 安全 | 无用户可控输入 |
| `Query.myFaceGroups` | **硬编码**（label NULL 优先 + COUNT） | ✅ 安全 | FormatSQL 传入 order=nil |
| `Search (media)` | **硬编码** + `clause.Expr` 参数化 | ✅ 安全 | `?` 占位符绑定 wildQuery |
| `Search (albums)` | 同上 | ✅ 安全 | 同上 |
| `FaceGroup.imageFaces(paginate)` | **无排序** | ✅ 安全 | FormatSQL 传入 order=nil |

### 16.6 建议的安全加固（代码中未实现）

```go
// 方案 A：在 FormatSQL 中加白名单校验
var validOrderByColumns = map[string]struct{}{
    "id":         {},
    "title":      {},
    "date_shot":  {},
    "updated_at": {},
    "created_at": {},
    "type":       {},
    "album_id":   {},
    // ... 其他合法列名
}

func FormatSQL(tx *gorm.DB, order *Ordering, paginate *Pagination) *gorm.DB {
    // ...
    if order != nil && order.OrderBy != nil {
        if _, ok := validOrderByColumns[*order.OrderBy]; !ok {
            // 非法列名：回退到默认（date_shot）或报错
            *order.OrderBy = "date_shot"
            // 或：return tx.Clauses(clause.Where{Exprs: []clause.Expression{gorm.ErrInvalidTransaction}})
        }
        tx.Order(clause.OrderByColumn{
            Column: clause.Column{Name: *order.OrderBy},
            Desc:   desc,
        })
    }
    // ...
}

// 方案 B：Schema 层改为 Enum（强烈推荐）
// graphql schema:
// enum MediaOrderField {
//   ID
//   TITLE
//   DATE_SHOT
//   UPDATED_AT
//   CREATED_AT
//   TYPE
// }
//
// input Ordering {
//   order_by: MediaOrderField    ← 从 String 改为 Enum
//   order_direction: OrderDirection
// }
// → gqlgen 自动校验非法值，无需后端额外代码
```

---

## 十七、代码路径索引补充

| 主题 | 文件 | 行号 |
|-----|------|-----|
| 数据库启动 SetupDatabase | `api/database/database.go` | 113-152 |
| 数据库 ConfigureDatabase | `api/database/database.go` | 77-110 |
| MySQL DSN 解析（MultiStatements=true） | `api/database/database.go` | 24-38 |
| SQLite WAL / _locking_mode 配置 | `api/database/database.go` | 63-70 |
| Resolver.DB(ctx) → WithContext | `api/graphql/resolvers/resolver.go` | 22-24 |
| FormatSQL → clause.OrderByColumn | `api/graphql/models/utils.go` | 11-40 |
| gqlgen unmarshalInputOrdering | `api/graphql/generated.go` | 9212-9247 |
| gqlgen unmarshalInputPagination | `api/graphql/generated.go` | 9249-9284 |
| Timeline 硬编码多数据库排序 | `api/graphql/models/actions/timeline_actions.go` | 20-40 |
| Search clause.Expr 参数化排序 | `api/graphql/models/actions/search_actions.go` | 39-44, 56-61 |
| 前端 SortingOptionValue 白名单类型 | `ui/src/components/album/AlbumFilter.tsx` | 14, 68-88 |
| 前端 useOrderingParams（orderBy 无白名单） | `ui/src/hooks/useOrderingParams.ts` | 15-47 |
| SQLite Scanner 单 worker 限制 | `api/graphql/resolvers/scanner.go` | 87-89 |
| Session AllowGlobalUpdate（非分页用） | `api/graphql/resolvers/scanner.go` | 62, 92 |
| 驱动类型枚举与判断 | `api/database/drivers/database_drivers.go` | 10-55 |
| 连接池 SetMaxOpenConns(80) | `api/database/database.go` | 140 |

---

## 十八、事务序列化失败时的重试/回退策略：代码中**不存在**

### 18.1 关键事实：全局搜索结果

在 `api/` 目录下搜索以下关键词，与事务重试相关的代码**全部为零**：

| 搜索关键词 | 匹配数 | 含义 |
|-----------|-------|------|
| `retry` | 2（均非事务相关） | `periodic_scanner_test.go` 中的测试重试、`database.go` 中的连接重试 |
| `backoff` | 0 | 无退避策略 |
| `deadlock` / `Deadlock` | 0 | 无死锁检测 |
| `serialization` / `SERIALIZATION` | 0 | 无序列化失败处理 |
| `cannot serialize` | 0 | 无 PG 错误码 40001 匹配 |
| `40001` / `40P01` | 0 | 无 SQLSTATE 错误码处理 |

### 18.2 项目中仅有的"重试"逻辑：数据库连接重试

唯一与重试相关的代码是启动阶段的**数据库连接重试**（非查询重试）：

```go
// database.go:126-149
for retryCount := 1; retryCount <= 5; retryCount++ {
    var err error
    db, err = ConfigureDatabase(&config)
    if err == nil {
        sqlDB, dbErr := db.DB()
        // ...
        err = sqlDB.PingContext(ctx)
        // ...
        if err == nil {
            return db, nil   // ← 连接成功则立即返回
        }
    }
    log.Printf("WARN: Could not ping database: %s. Will retry after 5 seconds\n", err)
    time.Sleep(5 * time.Second)  // ← 固定 5 秒间隔，无指数退避
}
return db, nil  // ← 重试 5 次后即使失败也返回（可能返回 nil db）
```

**这不是查询级别的重试**——仅用于进程启动时确保数据库可达。一旦 GraphQL 服务开始接收请求，**任何查询错误（包括序列化失败）都直接返回给客户端**。

### 18.3 GORM Transaction() 内置重试机制（项目未使用）

GORM v1.31.1 的 `db.Transaction()` 方法内部实现了**基于 SavePoint 的重试**：

```go
// GORM 内部源码（gorm.io/gorm/finisher_api.go）
func (db *DB) Transaction(fc func(tx *DB) error, opts ...*sql.TxOptions) error {
    // ...
    for i := 0; i < len(db.Statement.Settings); i++ {
        if db.Statement.Settings[i].Key == "retry_on_deadlock" {
            // 但这需要用户主动设置，默认关闭
        }
    }
    tx = tx.Begin(opts...)
    // ... 执行 fc(tx)
    // 如果遇到死锁错误且配置了重试，会自动重试
    // 但 Photoview 的代码从未设置此配置
}
```

**项目中的 10 处显式事务**（`db.Transaction()`）全部是**简单模式**，无重试配置：

```go
// 典型模式（faces.go:182）
err := db.Transaction(func(tx *gorm.DB) error {
    // ... 操作
    return nil  // 出错时 return err → GORM 自动 Rollback
})
// ← 出错后直接 return err 给 Resolver → 直接返回给前端
// ← 无任何重试逻辑
```

### 18.4 事务序列化失败可能发生的场景

虽然项目分页查询使用 auto-commit（无显式事务），但以下写操作使用事务，**可能与分页查询的读操作产生冲突**：

| 写事务 | 冲突点 | 可能的序列化错误 |
|-------|-------|----------------|
| `FavoriteMedia` (user.go:183) | 分页查询同时读 `user_media_data` | PG: 无（auto-commit 不会被阻塞）<br>MySQL: 行锁等待超时<br>SQLite: WAL 写锁竞争 |
| `CombineFaceGroups` (faces.go:182) | `myFaceGroups` 分页查询 | 同上 |
| `Scanner` 插入 Media (scanner_media.go:68) | `Album.media` 分页查询 | PG: 无（MVCC 互不干扰）<br>MySQL: Gap Lock 可能阻塞<br>SQLite: WAL 写锁竞争 |
| `CleanupMedia` 删除 (cleanup_media.go:52) | 任何 media 分页查询 | 同上 |

**关键结论**：由于分页查询是 **auto-commit 单语句**，PostgreSQL 的 MVCC 机制确保读操作永远不会被写事务阻塞（读不阻塞写，写不阻塞读）。序列化失败只会发生在**两个写事务之间**——而写事务的失败会直接返回 GraphQL Error，不会重试。

### 18.5 当前错误传播链路（无重试）

```
数据库执行错误
  │
  ├─ GORM Find() 返回 err
  │    └─ query.Find(&media).Error → err != nil
  │
  ├─ Resolver 直接返回
  │    └─ return nil, err
  │
  ├─ gqlgen 捕获错误
  │    └─ graphql.ErrorOnPath(ctx, err)
  │       → 构造 graphql.Error{Message: err.Error()}
  │
  └─ JSON 响应
       └─ {"errors": [{"message": "...", "path": ["myAlbums"]}]}
       ↑ 客户端收到 200 HTTP + GraphQL errors 数组
       ↑ 前端 Apollo Client 收到 error → useQuery 的 error 状态
       ↑ 前端不重试（无 Apollo retry link 配置）
```

### 18.6 三个数据库驱动下的具体错误行为

#### A. PostgreSQL

```
读操作（分页查询）：
  → 永远不会遇到序列化失败（READ COMMITTED + auto-commit）
  → 可能遇到：lock_timeout（如果被 FOR UPDATE 阻塞，但项目未使用 FOR UPDATE）
  → 可能遇到：statement_timeout（如果查询超时）
  → 实际最常见的错误：context canceled（客户端断开连接）

写操作（事务）：
  → 可能遇到：could not serialize access due to concurrent update (SQLSTATE 40001)
  → 但项目只在 SERIALIZABLE 隔离级别才会出现，当前默认 READ COMMITTED 不会
  → 实际最常见的写冲突：deadlock detected (SQLSTATE 40P01)
  → 项目中完全未处理 40P01
```

#### B. MySQL InnoDB

```
读操作（分页查询）：
  → 通常不会遇到序列化失败
  → 可能遇到：Lock wait timeout exceeded (ER_LOCK_WAIT_TIMEOUT = 1205)
  → 原因：Gap Lock 与 Insert 冲突
  → 项目中完全未处理 1205

写操作（事务）：
  → 可能遇到：Deadlock found when trying to get lock (ER_LOCK_DEADLOCK = 1213)
  → InnoDB 自动回滚死锁中最小的事务
  → 项目中完全未处理 1213
```

#### C. SQLite WAL

```
读操作：
  → WAL 模式下读永远不会阻塞（除非 checkpoint 正在进行）
  → 可能遇到：database is locked (SQLITE_BUSY = 5)
  → 项目未设置 _busy_timeout（被注释掉了！）
  
  // database.go:66（被注释掉的关键配置）：
  // queryValues.Add("_busy_timeout", "60000") // 1 minute

写操作：
  → 同样可能 SQLITE_BUSY
  → 无 busy_timeout → 立即返回 "database is locked"
  → 项目中完全未处理 SQLITE_BUSY
```

**注意**：`database.go:66` 的 `_busy_timeout` 被注释掉了——这是一个严重的遗漏，导致 SQLite 下写竞争时立即返回错误而非等待。

### 18.7 缺失的重试/回退策略汇总

| 缺失的策略 | 影响的数据库 | 严重度 | 修复方案 |
|-----------|-----------|-------|---------|
| 查询级重试（序列化失败 / 死锁） | PG/MySQL | 中 | 在 `db.Transaction()` 调用中增加 retry-on-deadlock 配置 |
| SQLite busy_timeout | SQLite | **高** | 取消 `database.go:66` 的注释，启用 60s 等待 |
| 指数退避 | 所有 | 低 | 启动连接重试改为指数退避（当前固定 5s） |
| Apollo Client 重试 Link | 前端 | 中 | 添加 `@apollo/client/link/retry` |
| GORM 错误码分类 | 所有 | 中 | 在 Resolver 层检测 SQLSTATE 40001/40P01/1205/1213/5 并决定重试 |
| 事务级 SavePoint 回退 | PG/MySQL | 低 | 长事务中使用 SavePoint 部分回滚 |

---

## 十九、嵌套 Resolver 下 N+1 查询的代码挂载点

### 19.1 N+1 问题的根源：gqlgen 的并发字段解析

gqlgen 为每个对象类型的每个字段生成独立的 Resolver。当分页返回 `[*models.Media]` 时，**每条 Media 的每个子字段都会独立调用 Resolver**：

```go
// generated.go 中 _Media 的结构：
func (ec *executionContext) _Media(ctx context.Context, sel ast.SelectionSet, obj *models.Media) graphql.Marshaler {
    fields := graphql.CollectFields(ec.OperationContext, sel, mediaImplementors)
    out := graphql.NewFieldSet(fields)
    for i, field := range fields {
        switch field.Name {
        case "thumbnail":
            out.Concurrently(i, func(ctx context.Context) graphql.Marshaler {
                return ec._Media_thumbnail(ctx, field, obj)  // ← 每条 Media 独立调用
            })
        case "favorite":
            out.Concurrently(i, func(ctx context.Context) graphql.Marshaler {
                return ec._Media_favorite(ctx, field, obj)   // ← 每条 Media 独立调用
            })
        // ...
        }
    }
    return out
}
```

**如果不做任何优化，200 条 Media × 5 个子字段 = 1000 次独立数据库查询** —— 这就是经典的 N+1 问题。

### 19.2 项目的 N+1 防护架构：两层防御

```
第 1 层：Dataloader（批量合并 + 缓存）
  │  适用于：thumbnail / highRes / videoWeb / favorite
  │  机制：5ms 窗口内收集所有 Load() 调用 → 合并为 1 次 IN 查询
  │
第 2 层：GORM 预加载 / 结构体已填充字段
  │  适用于：exif / faces（已在主查询中通过 GORM Association 加载）
  │  机制：Resolver 检查 obj.Exif != nil → 跳过查询
  │
第 3 层（缺失）：直接查库（N+1 未优化）
  │  适用于：album / shares / downloads
  │  机制：每条 Media 独立查一次 → 真正的 N+1
```

### 19.3 Dataloader 挂载点 1：中间件注册

**位置**：`api/dataloader/loaders.go:23-40`

```go
func Middleware(db *gorm.DB) mux.MiddlewareFunc {
    return mux.MiddlewareFunc(func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx := context.WithValue(r.Context(), loadersKey, &Loaders{
                MediaThumbnail:      NewThumbnailMediaURLLoader(db),   // ← 实例化
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

**关键设计**：
- **每个 HTTP 请求创建一组新 Loaders**（请求级生命周期）
- Dataloader 内部缓存（`map[int]*models.MediaURL`）在请求结束后随 GC 回收
- 同一请求内对同一 key 的两次 Load → 第 2 次命中缓存，不重复查询

**服务端挂载位置**：`api/server.go`

```go
r.Use(dataloader.Middleware(db))  // ← 在 GraphQL handler 之前注册
```

### 19.4 Dataloader 挂载点 2：Fetch 函数（实际 SQL 查询）

三个 MediaURL Dataloader 共享同一个 `makeMediaURLLoader` 工厂函数：

**位置**：`api/dataloader/mediaURLLoader.go:13-42`

```go
func makeMediaURLLoader(db *gorm.DB, filter func(query *gorm.DB) *gorm.DB) func(keys []int) ([]*models.MediaURL, []error) {
    return func(mediaIDs []int) ([]*models.MediaURL, []error) {
        var urls []*models.MediaURL
        query := db.Where("media_id IN (?)", mediaIDs)  // ← 批量 IN 查询
        query = filter(query)                            // ← 应用 purpose 过滤

        if err := query.Find(&urls).Error; err != nil {
            return nil, []error{errors.Wrap(err, "media url loader database query")}
        }

        resultMap := make(map[int]*models.MediaURL, len(mediaIDs))
        for _, url := range urls {
            resultMap[url.MediaID] = url
        }

        result := make([]*models.MediaURL, len(mediaIDs))
        for i, mediaID := range mediaIDs {
            mediaURL, found := resultMap[mediaID]
            if found {
                result[i] = mediaURL
            } else {
                result[i] = nil    // ← 找不到则返回 nil（前端展示为空）
            }
        }
        return result, nil
    }
}
```

**三个实例的 filter 区别**：

| Dataloader | filter 条件 | 用途 |
|-----------|------------|------|
| `MediaThumbnail` | `purpose IN ('photo_thumbnail', 'video_thumbnail')` | 缩略图 |
| `MediaHighres` | `purpose = 'photo_highres' OR (purpose = 'media_original' AND content_type IN webMimetypes)` + 优先排序 | 高清图 |
| `MediaVideoWeb` | `purpose IN ('video_web', 'media_original')` + 优先排序 | Web 视频 |

**Favorite Dataloader 的 Fetch 函数**（特殊：复合 key）：

**位置**：`api/dataloader/userFavoriteLoader.go:10-59`

```go
func NewUserFavoriteLoader(db *gorm.DB) *UserFavoritesLoader {
    return &UserFavoritesLoader{
        maxBatch: 100,
        wait:     5 * time.Millisecond,
        fetch: func(keys []*models.UserMediaData) ([]bool, []error) {
            // keys 中的每个元素是 {UserID, MediaID} 复合键
            // 先提取唯一 UserID 和 MediaID 集合
            userIDMap := make(map[int]struct{}, len(keys))
            mediaIDMap := make(map[int]struct{}, len(keys))
            for _, key := range keys {
                userIDMap[key.UserID] = struct{}{}
                mediaIDMap[key.MediaID] = struct{}{}
            }

            // 批量查询：WHERE user_id IN (...) AND media_id IN (...) AND favorite = TRUE
            var userMediaFavorites []*models.UserMediaData
            err := db.Where("user_id IN (?)", uniqueUserIDs).
                Where("media_id IN (?)", uniqueMediaIDs).
                Where("favorite = TRUE").
                Find(&userMediaFavorites).Error

            // 遍历 keys 匹配结果
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

**注意缺陷**：`WHERE user_id IN (...) AND media_id IN (...) AND favorite = TRUE` 的过滤条件是笛卡尔积的子集——如果 user A 有 200 条 media 收藏，user B 有 200 条不同的 media 收藏，查询会返回 400 条，但可能只需要其中 200 条。对于单用户场景这不是问题，但如果多用户共享请求（不会发生，因为每个请求一个用户），可能返回多余数据。

### 19.5 Dataloader 挂载点 3：Resolver 调用

**位置**：`api/graphql/resolvers/media.go`

```go
// line 23-25: Thumbnail → Dataloader
func (r *mediaResolver) Thumbnail(ctx context.Context, obj *models.Media) (*models.MediaURL, error) {
    return dataloader.For(ctx).MediaThumbnail.Load(obj.ID)
}

// line 28-34: HighRes → Dataloader
func (r *mediaResolver) HighRes(ctx context.Context, obj *models.Media) (*models.MediaURL, error) {
    if obj.Type != models.MediaTypePhoto {
        return nil, nil     // ← 提前返回：视频类型直接返回 nil，不查 Dataloader
    }
    return dataloader.For(ctx).MediaHighres.Load(obj.ID)
}

// line 37-43: VideoWeb → Dataloader
func (r *mediaResolver) VideoWeb(ctx context.Context, obj *models.Media) (*models.MediaURL, error) {
    if obj.Type != models.MediaTypeVideo {
        return nil, nil     // ← 提前返回：照片类型直接返回 nil，不查 Dataloader
    }
    return dataloader.For(ctx).MediaVideoWeb.Load(obj.ID)
}

// line 70-80: Favorite → Dataloader
func (r *mediaResolver) Favorite(ctx context.Context, obj *models.Media) (bool, error) {
    user := auth.UserFromContext(ctx)
    if user == nil {
        return false, auth.ErrUnauthorized
    }
    return dataloader.For(ctx).UserMediaFavorite.Load(&models.UserMediaData{
        UserID:  user.ID,
        MediaID: obj.ID,
    })
}
```

### 19.6 非 Dataloader 的子字段：真正的 N+1 查询点

以下 Resolver **没有使用 Dataloader**，每次调用都是独立数据库查询：

```go
// media.go:46-53: Album → 直接查库（N+1！）
func (r *mediaResolver) Album(ctx context.Context, obj *models.Media) (*models.Album, error) {
    var album models.Album
    err := r.DB(ctx).Find(&album, obj.AlbumID).Error  // ← 每条 Media 查一次 albums 表
    if err != nil {
        return nil, err
    }
    return &album, nil
}

// media.go:89-96: Shares → 直接查库（N+1！）
func (r *mediaResolver) Shares(ctx context.Context, obj *models.Media) ([]*models.ShareToken, error) {
    var shareTokens []*models.ShareToken
    if err := r.DB(ctx).Where("media_id = ?", obj.ID).Find(&shareTokens).Error; err != nil {
        return nil, fmt.Errorf("get shares for media (%s): %w", obj.Path, err)
    }
    return shareTokens, nil
}

// media.go:99-130: Downloads → 直接查库（N+1！）
func (r *mediaResolver) Downloads(ctx context.Context, obj *models.Media) ([]*models.MediaDownload, error) {
    var mediaUrls []*models.MediaURL
    if err := r.DB(ctx).Where("media_id = ?", obj.ID).Find(&mediaUrls).Error; err != nil {
        return nil, fmt.Errorf("get downloads for media (%s): %w", obj.Path, err)
    }
    // ... 构建 MediaDownload
}

// media.go:133-148: Faces → GORM Association（N+1！）
func (r *mediaResolver) Faces(ctx context.Context, obj *models.Media) ([]*models.ImageFace, error) {
    if face_detection.GlobalFaceDetector == nil {
        return []*models.ImageFace{}, nil
    }
    if obj.Faces != nil {
        return obj.Faces, nil     // ← 有缓存则跳过
    }
    var faces []*models.ImageFace
    if err := r.DB(ctx).Model(obj).Association("Faces").Find(&faces); err != nil {
        return nil, err           // ← 无缓存则查库（每条 Media 一次）
    }
    return faces, nil
}

// media.go:56-67: Exif → GORM Association（N+1！）
func (r *mediaResolver) Exif(ctx context.Context, obj *models.Media) (*models.MediaEXIF, error) {
    if obj.Exif != nil {
        return obj.Exif, nil      // ← 有缓存则跳过
    }
    var exif models.MediaEXIF
    if err := r.DB(ctx).Model(obj).Association("Exif").Find(&exif); err != nil {
        return nil, err           // ← 无缓存则查库（每条 Media 一次）
    }
    return &exif, nil
}

// album.go:73-75: Album.Thumbnail → 模型方法
func (r *albumResolver) Thumbnail(ctx context.Context, obj *models.Album) (*models.Media, error) {
    return obj.Thumbnail(r.DB(ctx))  // ← 每个相册查一次 cover
}

// album.go:89-96: Album.Shares → 直接查库
func (r *albumResolver) Shares(ctx context.Context, obj *models.Album) ([]*models.ShareToken, error) {
    var shareTokens []*models.ShareToken
    if err := r.DB(ctx).Where("album_id = ?", obj.ID).Find(&shareTokens).Error; err != nil {
        return nil, err
    }
    return shareTokens, nil
}
```

### 19.7 N+1 查询影响矩阵

以最常见的前端查询为例（Timeline 页面：200 条 Media，请求 thumbnail + favorite + date）：

| 子字段 | Resolver 类型 | N+1? | 实际查询次数 | 原因 |
|-------|-------------|------|-----------|------|
| `thumbnail` | Dataloader | ❌ 否 | 1 | 200 个 key 合并为 `WHERE media_id IN (1,2,...,200)` |
| `highRes` | Dataloader | ❌ 否 | 1 | 同上（照片类型才查） |
| `videoWeb` | Dataloader | ❌ 否 | 1 | 同上（视频类型才查） |
| `favorite` | Dataloader | ❌ 否 | 1 | 复合 key 合并为 `WHERE user_id IN (...) AND media_id IN (...)` |
| `date` | 直接读 obj | ❌ 否 | 0 | `obj.DateShot` 已在主查询 SELECT * 中加载 |
| `title` | 直接读 obj | ❌ 否 | 0 | 同上 |
| `blurhash` | 直接读 obj | ❌ 否 | 0 | 同上 |
| `type` | 格式化 | ❌ 否 | 0 | `cases.Title().String()` 纯内存操作 |
| **`album`** | 直接查库 | ✅ **是** | **200** | 每条 Media 查一次 `Find(&album, obj.AlbumID)` |
| **`exif`** | GORM Assoc | ✅ **是** | **200** | 每条 Media 查一次 `Association("Exif").Find()` |
| **`faces`** | GORM Assoc | ✅ **是** | **200** | 每条 Media 查一次 `Association("Faces").Find()` |
| **`shares`** | 直接查库 | ✅ **是** | **200** | 每条 Media 查一次 `WHERE media_id = ?` |
| **`downloads`** | 直接查库 | ✅ **是** | **200** | 每条 Media 查一次 `WHERE media_id = ?` |

### 19.8 Dataloader 的内部调度机制：5ms 窗口 + maxBatch=100

以 `gen_mediaurlloader.go` 为例（三个 MediaURL Dataloader 的核心逻辑相同）：

```
gqlgen _Media 并发调度 200 个 goroutine：
  │
  │  goroutine 1:  mediaResolver.Thumbnail(ctx, media[0])
  │    → dataloader.For(ctx).MediaThumbnail.Load(1)
  │      → cache 未命中 → batch.keyIndex(l, 1) → pos=0
  │      → pos==0 → go b.startTimer(l)   ← 启动 5ms 倒计时
  │
  │  goroutine 2:  mediaResolver.Thumbnail(ctx, media[1])
  │    → dataloader.For(ctx).MediaThumbnail.Load(2)
  │      → cache 未命中 → batch.keyIndex(l, 2) → pos=1
  │
  │  ... (goroutine 3-99 同理，pos=2-98)
  │
  │  goroutine 100: pos=99 → maxBatch=100 → pos >= maxBatch-1
  │    → b.closing = true
  │    → l.batch = nil
  │    → go b.end(l)  ← ★ 第 1 批立即触发！不等 5ms
  │
  │  goroutine 101-200: 新建第 2 批 batch
  │    → batch.keyIndex(l, key) → pos=0
  │    → go b.startTimer(l)   ← 第 2 批 5ms 倒计时
  │
  ▼
5ms 后（或 maxBatch 触发后）：
  │
  │  b.end(l) 被调用：
  │    b.data, b.error = l.fetch(b.keys)
  │    │
  │    │  fetch = makeMediaURLLoader(db, filter)
  │    │  → db.Where("media_id IN (?)", [1,2,...,100]).Find(&urls)
  │    │  → SQL: SELECT * FROM media_urls WHERE media_id IN (1,2,...,100) AND purpose IN (...)
  │    │  → 1 次查询返回 100 条结果
  │    │
  │    → 遍历 resultMap 填充 batch.data[0..99]
  │    → close(batch.done)  ← 通知所有等待的 goroutine
  │
  ▼
goroutine 1-100 从 batch.done channel 收到信号：
  │  ← batch.done 已关闭，不再阻塞
  │  data = batch.data[0]  // media[0] 的 thumbnail
  │  存入 cache: cache[1] = data
  │  返回 data 给 gqlgen
  │
  ▼
5ms 后第 2 批也完成：
  → SQL: SELECT * FROM media_urls WHERE media_id IN (101,...,200) AND purpose IN (...)
  → goroutine 101-200 收到结果
```

**实际效果**：
- 200 条 Media × 3 个 Dataloader 字段（thumbnail + highRes + videoWeb）
- = **6 次 SQL 查询**（每批 100 个 key → 2 批 × 3 个 Dataloader）
- 对比无 Dataloader：200 × 3 = **600 次 SQL 查询**
- **性能提升 ~100 倍**

### 19.9 Dataloader 配置参数对比

| Dataloader | maxBatch | wait | 含义 |
|-----------|----------|------|------|
| `MediaThumbnail` | 100 | 5ms | 最多 100 个 key 一批；首 key 后等 5ms |
| `MediaHighres` | 100 | 5ms | 同上 |
| `MediaVideoWeb` | 100 | 5ms | 同上 |
| `UserFromAccessToken` | 100 | 5ms | 同上 |
| `UserMediaFavorite` | 100 | 5ms | 同上 |

**窗口选择分析**：
- `wait=5ms`：在 gqlgen 的 `Concurrently` 调度下，200 个 goroutine 几乎同时调用 `Load()`，5ms 内全部到达 → 合并为 1-2 批
- `maxBatch=100`：防止超大批次（200 条 → 自动拆为 2 批）
- 对于 Timeline 页面（通常一次 200 条）：恰好触发 2 批 × 3 = 6 次 SQL

### 19.10 N+1 查询的实际影响评估

| 前端页面 | 请求的子字段 | 是否有 N+1 | 实际影响 |
|---------|-----------|-----------|---------|
| Timeline 主页 | thumbnail + favorite + date + album{id,title} | ⚠️ `album` 字段有 | 200 条 Media → 200 次 album 查询（但 album 表通常有索引 + 行数少 → 单次 <1ms，总计 ~200ms） |
| Album 页面 | thumbnail + highRes + videoWeb + date | ❌ 无 | 全部走 Dataloader |
| Album 页面（含 exif） | + exif{...} | ⚠️ `exif` 字段有 | 200 条 Media → 200 次 exif 查询（单次 ~5ms，总计 ~1s） |
| Faces 页面 | imageFaces + faceGroup | ⚠️ `imageFaces` 有 | 人脸组通常 <1000 → 影响可控 |

### 19.11 完整的请求内查询次数统计

以 Timeline 页面请求 `myTimeline(limit: 200) { thumbnail favorite date album{id title} }` 为例：

```
[主查询] 1 次
  SELECT * FROM media JOIN albums ... ORDER BY ... LIMIT 200

[Dataloader: Thumbnail] 2 次（200 key → 2 批 × maxBatch=100）
  SELECT * FROM media_urls WHERE media_id IN (1,...,100) AND purpose IN (...)
  SELECT * FROM media_urls WHERE media_id IN (101,...,200) AND purpose IN (...)

[Dataloader: Favorite] 1 次（同 1 个 user，200 key → 2 批但通常合并为 1 批）
  SELECT * FROM user_media_data WHERE user_id IN (42) AND media_id IN (1,...,200) AND favorite = TRUE

[N+1: Album] 200 次
  SELECT * FROM albums WHERE id = 1
  SELECT * FROM albums WHERE id = 2
  ... (198 more)

───────────────────────
总计：204 次 SQL 查询
理想（全部 Dataloader）：4 次 SQL 查询
差距：51 倍

如果加上 exif 请求：
  +200 次 SELECT * FROM media_exifs WHERE media_id = ?
  总计：404 次 SQL 查询
```

---

## 二十、代码路径索引补充（续）

| 主题 | 文件 | 行号 |
|-----|------|-----|
| Dataloader 中间件注册 | `api/dataloader/loaders.go` | 23-40 |
| Dataloader.For(ctx) 取 Loaders | `api/dataloader/loaders.go` | 42-48 |
| MediaURL Fetch 工厂 | `api/dataloader/mediaURLLoader.go` | 13-42 |
| Thumbnail Dataloader 实例化 | `api/dataloader/mediaURLLoader.go` | 44-52 |
| HighRes Dataloader 实例化 | `api/dataloader/mediaURLLoader.go` | 54-67 |
| VideoWeb Dataloader 实例化 | `api/dataloader/mediaURLLoader.go` | 69-82 |
| UserFavorite Fetch 函数 | `api/dataloader/userFavoriteLoader.go` | 10-59 |
| UserFromToken Fetch 函数 | `api/dataloader/userLoader.go` | 10-70 |
| gen_mediaurlloader 批量调度核心 | `api/dataloader/gen_mediaurlloader.go` | 66-224 |
| gen_userfavoritesloader 批量调度核心 | `api/dataloader/gen_userfavoritesloader.go` | 66-220 |
| gen_userloader 批量调度核心 | `api/dataloader/gen_userloader.go` | 66-223 |
| mediaResolver.Thumbnail → Dataloader | `api/graphql/resolvers/media.go` | 23-25 |
| mediaResolver.HighRes → Dataloader | `api/graphql/resolvers/media.go` | 28-34 |
| mediaResolver.VideoWeb → Dataloader | `api/graphql/resolvers/media.go` | 37-43 |
| mediaResolver.Favorite → Dataloader | `api/graphql/resolvers/media.go` | 70-80 |
| mediaResolver.Album → N+1 直接查库 | `api/graphql/resolvers/media.go` | 46-53 |
| mediaResolver.Exif → GORM Association | `api/graphql/resolvers/media.go` | 56-67 |
| mediaResolver.Faces → GORM Association | `api/graphql/resolvers/media.go` | 133-148 |
| mediaResolver.Shares → N+1 直接查库 | `api/graphql/resolvers/media.go` | 89-96 |
| mediaResolver.Downloads → N+1 直接查库 | `api/graphql/resolvers/media.go` | 99-130 |
| albumResolver.Thumbnail → 模型方法 | `api/graphql/resolvers/album.go` | 73-75 |
| albumResolver.Shares → N+1 直接查库 | `api/graphql/resolvers/album.go` | 89-96 |
| SQLite busy_timeout 被注释掉 | `api/database/database.go` | 66 |
| 数据库连接启动重试（固定 5s） | `api/database/database.go` | 126-149 |
