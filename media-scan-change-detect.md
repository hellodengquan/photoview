# 媒体目录扫描与变更识别逻辑

本文档从代码实现角度，梳理 Photoview 中媒体目录扫描的完整流程，以及**新增、改名、删除**三类变更的识别算法。

---

## 1. 核心设计思想

Photoview 的变更识别**不基于文件内容哈希**，而是基于**文件路径哈希（path_hash）**进行匹配。也就是说：

- 「文件是否已存在」通过**绝对路径的 MD5** 判定
- 「文件是否被删除」通过**「数据库记录集」与「本次扫描到的文件集」做差集**判定
- 「文件/目录改名」**不被当作原子操作**，而是等价于「旧路径被删除 + 新路径被新增」

唯一例外：Sidecar 文件（`.xmp`）的变更检测使用**内容 MD5 哈希**。

---

## 2. 关键数据结构

### 2.1 Media 模型（`api/graphql/models/media.go`）

```go
type Media struct {
    Title     string
    Path      string    // 绝对路径
    PathHash  string    // unique, MD5(Path)，在 BeforeSave 钩子中自动计算
    AlbumID   int
    DateShot  time.Time
    Type      MediaType
    // ... 其他字段
}

func (m *Media) BeforeSave(tx *gorm.DB) error {
    m.PathHash = MD5Hash(m.Path)  // 写入前自动生成路径哈希
    return nil
}
```

### 2.2 Album 模型（`api/graphql/models/album.go`）

```go
type Album struct {
    Title         string
    ParentAlbumID *int
    Path          string
    PathHash      string  // unique, MD5(Path)
    // ...
}
```

### 2.3 MD5Hash 工具（`api/graphql/models/utils.go`）

```go
func MD5Hash(value string) string {
    hash := md5.Sum([]byte(value))
    return hex.EncodeToString(hash[:])
}
```

---

## 3. 扫描总流程

### 3.1 调用入口

| 入口函数 | 文件 | 作用 |
|---|---|---|
| `AddAllToQueue()` | `scanner_queue/queue.go:204` | 所有用户入队扫描 |
| `AddUserToQueue(user)` | `scanner_queue/queue.go:223` | 单个用户入队扫描 |

### 3.2 扫描链路（自顶向下）

```
AddUserToQueue
  └── FindAlbumsForUser()            // 第1阶段：递归构建/匹配相册树
        ├── 目录BFS遍历
        ├── 对每个目录：通过 path_hash 查 Album
        │     ├── 命中 → 复用已有 Album 记录
        │     └── 未命中 → 创建新 Album 记录
        └── DeleteOldUserAlbums()    // 删除用户层面已消失的相册

  └── 对每个 Album 提交 ScannerJob
        └── ScanAlbum()              // 第2阶段：扫描单相册内媒体
              ├── BeforeScanAlbum
              ├── findMediaForAlbum()
              │     ├── os.ReadDir(相册路径)
              │     └── 对每个文件：
              │           ├── MediaFound 过滤钩子
              │           ├── ScanMedia()  ← 【新增识别点】
              │           └── AfterMediaFound
              ├── 对每个 media：scanMedia() 处理（缩略图/EXIF/人脸…）
              └── AfterScanAlbum
                    └── MediaCleanupTask
                          └── CleanupMedia()  ← 【删除识别点】
```

---

## 4. 新增识别：`ScanMedia()`

**位置**：`api/scanner/scanner_media.go:21-73`

### 4.1 算法伪代码

```go
func ScanMedia(tx *gorm.DB, mediaPath string, albumId int, cache) (*Media, bool /* isNew */, error) {

    // 1. 通过路径哈希查重
    hash := MD5Hash(mediaPath)
    var existing []*Media
    tx.Where("path_hash = ?", hash).Find(&existing)

    if len(existing) > 0 {
        return existing[0], false /* 非新增 */, nil  // ← 命中，直接返回
    }

    // 2. 未命中，说明是新文件 → 创建新记录
    mediaType := cache.GetMediaType(mediaPath)     // photo / video
    stat, _ := os.Stat(mediaPath)

    media := Media{
        Title:    path.Base(mediaPath),
        Path:     mediaPath,
        AlbumID:  albumId,
        Type:     mediaTypeText,
        DateShot: stat.ModTime(),
    }
    tx.Create(&media)   // BeforeSave 会自动写入 PathHash

    return &media, true /* 新增 */, nil
}
```

### 4.2 判定依据

| 条件 | 结论 |
|---|---|
| `SELECT * FROM media WHERE path_hash = MD5(mediaPath)` 有结果 | 已存在，**非新增** |
| 无结果 | **新增**，插入 DB |

### 4.3 关键特征

- **不读取文件内容**，仅依赖路径字符串，扫描速度快
- PathHash 是 `unique` 索引，查重高效
- 重命名会导致 PathHash 变化，因此**不被视为同一条记录**

---

## 5. 删除识别：`CleanupMedia()`

**位置**：`api/scanner/scanner_tasks/cleanup_tasks/cleanup_media.go:17-65`

### 5.1 触发时机

在单个相册所有文件扫描完成后，由 `MediaCleanupTask.AfterScanAlbum()` 回调触发：

```go
// api/scanner/scanner_tasks/cleanup_tasks/media_cleanup_task.go
func (t MediaCleanupTask) AfterScanAlbum(ctx, changedMedia, albumMedia) error {
    CleanupMedia(ctx.GetDB(), ctx.GetAlbum().ID, albumMedia)
    return nil
}
```

其中 `albumMedia` 是**本次扫描到、且仍在文件系统上的媒体集合**。

### 5.2 算法伪代码

```go
func CleanupMedia(db *gorm.DB, albumId int, albumMedia []*Media) []error {

    // 1. 收集本次扫描到的 media ID
    scanIds := make([]int, len(albumMedia))
    for i, m := range albumMedia { scanIds[i] = m.ID }

    // 2. 查询「在数据库中，但不在本次扫描结果中」的记录
    var toDelete []Media
    query := db.Where("album_id = ?", albumId)
    if len(scanIds) > 0 {
        query = query.Where("NOT id IN (?)", scanIds)
    }
    query.Find(&toDelete)   // ← 差集即为「已删除文件」

    // 3. 执行删除
    //   a) 删除磁盘缓存目录
    //   b) DELETE FROM media WHERE id IN (...)
    //   c) 重新加载人脸检测器索引
}
```

### 5.3 判定依据

集合差集：`DB(album_id = X)` \ `本次扫描到的 media`

- 在 DB 里，本次没扫到 → **删除**
- 在 DB 里，本次也扫到 → 保留

### 5.4 相册级删除：`DeleteOldUserAlbums()`

**位置**：`cleanup_media.go:68-136`

同样是差集法：

```
DB 中该用户拥有的 Album 集合  \  本次扫描到的 Album 集合
= 应该删除的 Album（级联删除其下所有 media）
```

---

## 6. 改名识别（隐式）

### 6.1 本质：删除 + 新增

Photoview **没有专门的「改名检测」模块**。改名通过以下链路被间接感知：

| 场景 | ScanMedia 判定 | CleanupMedia 判定 | 宏观表现 |
|---|---|---|---|
| `/a/IMG_001.jpg` → `/a/IMG_999.jpg`（同目录） | 新路径 `MD5` 未命中 → 新增 1 条 | 旧路径未扫到 → 删除 1 条 | 总数不变，ID 变化 |
| `/a/faces/` → `/a/faces_moved/`（目录改名） | 新目录下所有文件 path_hash 不命中 → N 条新增 | 旧目录 Album 不在扫描集 → 级联删除 N 条 | 总数不变，Album ID 与 Media ID 均变化 |
| `/a/IMG.jpg` → `/b/IMG.jpg`（跨目录移动） | 新目录 album_id 变 → 新增 | 旧目录下扫不到 → 删除 | 总数不变 |

### 6.2 测试证据（`cleanup_media_test.go`）

```go
// 场景1：目录改名
os.Rename(testDir + "/faces", testDir + "/faces_moved")
RunScannerAll(db)
assert.Equal(t, 9, countAllMedia())    // 仍为 9：删了旧 faces 的 + 新增 faces_moved 的

// 场景2：文件改名
os.Rename(testDir + "/buttercup_close_summer_yellow.jpg",
          testDir + "/yellow-flower.jpg")
RunScannerAll(db)
assert.Equal(t, 3, countAllMedia())    // 仍为 3：1删1增

// 场景3：文件删除（对比参照）
os.Remove(testDir + "/lilac_lilac_bush_lilac.jpg")
RunScannerAll(db)
assert.Equal(t, 2, countAllMedia())    // 3→2：净减1
```

### 6.3 副作用

- 改名**不会保留原记录的 ID**，也不会保留与该 ID 关联的人工标签、收藏等元数据（除非这些关联模型的外键是级联更新，但 GORM 里通常只配置了 `OnDelete:CASCADE`）
- 改名会触发**全量重新处理**（缩略图、EXIF、人脸检测、Blurhash），因为对系统而言它是「全新的媒体」
- 相册/媒体的封面引用可能因原 ID 被删而失效（依赖上层 `CoverID` 的容错处理）

---

## 7. Sidecar 文件变更识别（内容哈希）

这是整个系统中**唯一使用内容哈希做变更检测**的场景。

**位置**：`api/scanner/scanner_tasks/processing_tasks/sidecar_task.go`

### 7.1 检测点：`ProcessMedia()`（第 56-131 行）

```go
func (t SidecarTask) ProcessMedia(ctx, mediaData, mediaCachePath) {

    currentSideCarPath := scanForSideCarFile(photo.Path) // 查 .xmp

    // 1. 计算当前 sidecar 的内容 MD5
    var currentFileHash *string
    if currentSideCarPath != nil {
        currentFileHash = hashSideCarFile(currentSideCarPath)
    }

    // 2. 与数据库中保存的 SideCarHash 对比
    sideCarFileHasChanged := false

    switch {
    case currentFileHash == nil && photo.SideCarHash != nil:
        // sidecar 被删了
        sideCarFileHasChanged = true
    case currentFileHash != nil && photo.SideCarHash == nil:
        // sidecar 新增了
        sideCarFileHasChanged = true
    case currentFileHash != nil && *currentFileHash != *photo.SideCarHash:
        // sidecar 内容变了
        sideCarFileHasChanged = true
    }

    // 3. 若变更 → 重新生成缩略图与高分辨率图
    if sideCarFileHasChanged {
        generateSaveHighResJPEG(...)
        generateSaveThumbnailJPEG(...)
        photo.SideCarHash = currentFileHash  // 更新哈希
    }
}
```

### 7.2 哈希算法

```go
func hashSideCarFile(path *string) *string {
    f, _ := os.Open(*path)
    defer f.Close()
    h := md5.New()
    io.Copy(h, f)                          // 流式读取整个文件
    hash := hex.EncodeToString(h.Sum(nil))
    return &hash
}
```

### 7.3 判定矩阵

| DB 中 SideCarHash | 当前磁盘 sidecar | 结论 |
|---|---|---|
| 有 | 无（被删） | 变更 → 重生成 |
| 无 | 有（新增） | 变更 → 重生成 |
| 有，值 = H1 | 有，值 = H1 | 不变 |
| 有，值 = H1 | 有，值 = H2 ≠ H1 | 变更 → 重生成 |

---

## 8. 三种变更类型识别总览表

| 变更类型 | 识别位置 | 识别依据 | 数据结构匹配键 | 是否保留原 ID | 是否重新处理 |
|---|---|---|---|---|---|
| **新增（文件）** | `ScanMedia()` | `path_hash` 未命中 DB | MD5(绝对路径) | N/A | 是 |
| **删除（文件）** | `CleanupMedia()` | DB ∖ 本次扫描集 差集 | media.id | N/A | N/A |
| **改名/移动（文件）** | `ScanMedia()` + `CleanupMedia()` 组合 | 新路径新增 + 旧路径删除 | MD5(绝对路径) | **否**（新记录） | **是**（全套重做） |
| **新增（相册）** | `FindAlbumsForUser()` | Album `path_hash` 未命中 | MD5(目录绝对路径) | N/A | N/A |
| **删除（相册）** | `DeleteOldUserAlbums()` | 用户 Album 集差集 | album.id | N/A | 级联删所有媒体 |
| **改名（相册）** | 上述两者组合 | 旧 Album 删除 + 新 Album 创建 | MD5(目录绝对路径) | **否** | 其下媒体全部重做 |
| **Sidecar 变更** | `SidecarTask.ProcessMedia()` | sidecar 文件 MD5 对比 | 内容 MD5 | 是（复用 media.id） | 仅重生成缩略图 |

---

## 9. 典型场景时序图

### 9.1 新增一张图

```
filesystem: /album/new.jpg  (新增)
    │
    ▼
ScanAlbum
  findMediaForAlbum
    ScanMedia("/album/new.jpg")
      ├── SELECT * FROM media WHERE path_hash=MD5("/album/new.jpg")  → 空
      ├── INSERT INTO media (path="/album/new.jpg", path_hash=..., ...)
      └── return media, isNew=true
  scanMedia → 生成缩略图 / EXIF / 人脸 / Blurhash
  AfterScanAlbum → CleanupMedia
    └── albumMedia 包含 new.jpg → 无删除
```

### 9.2 删除一张图

```
filesystem: /album/old.jpg  (被删)
    │
    ▼
ScanAlbum
  findMediaForAlbum
    ├── os.ReadDir → 列表里没有 old.jpg
    └── albumMedia = [其它文件]
  ...
  AfterScanAlbum → CleanupMedia
    ├── DB.album_id = X 的 media 集合 = [..., old.jpg.id, ...]
    ├── 差集 = {old.jpg.id}
    ├── rm -rf cache/<albumId>/<old.jpg.id>/
    └── DELETE FROM media WHERE id = old.jpg.id
```

### 9.3 改名一张图

```
filesystem: /album/A.jpg  ─rename─▶  /album/B.jpg
    │
    ▼
ScanAlbum
  findMediaForAlbum
    ├── 遍历到 B.jpg
    │     ScanMedia("/album/B.jpg")
    │       ├── SELECT path_hash=MD5("/album/B.jpg") → 空
    │       ├── INSERT → 新记录 (id=2)
    │       └── albumMedia append 2
    └── 遍历时不会看到 A.jpg（已不在磁盘）
  ...
  AfterScanAlbum → CleanupMedia
    ├── DB 里有 [id=1 (A.jpg), id=2 (B.jpg)]
    ├── albumMedia = [id=2 (B.jpg)]
    ├── 差集 = {id=1}
    └── DELETE FROM media WHERE id = 1

最终：media 表数量不变，但 id 从 1 变成了 2，所有处理重新做过
```

---

## 10. 代码文件索引

| 路径 | 作用 | 关键行 |
|---|---|---|
| `api/scanner/scanner_queue/queue.go` | 扫描队列，入口调度 | `AddUserToQueue:223`, `AddAllToQueue:204` |
| `api/scanner/scanner_user.go` | 用户→相册树递归扫描 | `FindAlbumsForUser:45` |
| `api/scanner/scanner_album.go` | 单相册扫描主流程 | `ScanAlbum:87`, `findMediaForAlbum:116` |
| `api/scanner/scanner_media.go` | 单媒体查重/入库（**新增识别**） | `ScanMedia:21-73` |
| `api/scanner/media_scan.go` | 单媒体处理管道（缩略图/EXIF…） | `scanMedia:11` |
| `api/scanner/scanner_tasks/scanner_tasks.go` | 任务管道编排，串联所有扫描任务钩子 | `Tasks:33`, `AfterScanAlbum:94` |
| `api/scanner/scanner_tasks/cleanup_tasks/cleanup_media.go` | **删除识别** + 相册清理 | `CleanupMedia:17`, `DeleteOldUserAlbums:68` |
| `api/scanner/scanner_tasks/cleanup_tasks/media_cleanup_task.go` | 删除任务钩子 | `AfterScanAlbum:13` |
| `api/scanner/scanner_tasks/processing_tasks/sidecar_task.go` | Sidecar **内容哈希**变更检测 | `ProcessMedia:56`, `hashSideCarFile:143` |
| `api/graphql/models/media.go` | Media 结构与 `path_hash` 钩子 | `BeforeSave:39-44` |
| `api/graphql/models/album.go` | Album 结构与 `path_hash` 钩子 | `BeforeSave:27-31` |
| `api/graphql/models/utils.go` | `MD5Hash()` 工具函数 | `MD5Hash:43-46` |
| `api/scanner/scanner_tasks/cleanup_tasks/cleanup_media_test.go` | 改名/删除行为测试用例 | `TestCleanupMedia:20` |

---

## 11. 设计思考

### 优点

1. **实现简洁**：所有变更识别基于字符串哈希 + 集合差集，无复杂状态机
2. **扫描高效**：文件级匹配不读取内容，对大型图库友好
3. **强一致性**：以「每次扫描的文件系统状态」为唯一真相源，DB 永远收敛到与磁盘一致

### 局限

1. **改名感知粗粒度**：不能识别「同文件改名」，导致重复处理与元数据丢失风险
2. **不支持增量**：每次都要全量遍历目录（虽有缓存，但不涉及变更检测层面）
3. **并发 rename**：如果扫描期间文件跨目录移动，可能出现「两个都扫到」或「两个都没扫到」的短暂窗口（由 DB 事务保证最终一致，但中间态可能出现重复或短暂丢失）

### 优化方向（潜在）

- 引入「文件内容哈希 + 大小 + 修改时间」多重签名，可实现真正的改名/移动检测，保留原 ID 与人工标签
- 为相册/媒体封面引用添加 `ON DELETE SET NULL` 或清理逻辑，避免悬挂指针
- 引入 inotify/fsevents 级别的增量扫描，减少全量遍历频率
