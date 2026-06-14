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

## 11. 海量媒体场景：并发模型与性能估算

### 11.1 并发架构总览

Photoview 的扫描并发分为**两层**：

```
全局（用户级）                        单相册内（媒体级）
───────────────────                  ─────────────────
AddUserToQueue                        ScanAlbum
  │                                     │
  ├── FindAlbumsForUser（单线程BFS）     ├── findMediaForAlbum（串行遍历目录）
  │     └── 对每个Album：入队            ├── for media in albumMedia:
  │                                          scanMedia（串行处理每个媒体）
  └── ScannerQueue 分发
        ├── max_concurrent_tasks = N
        └── 每个 Album 起一个 goroutine
              └── ScanAlbum（单线程跑完整个相册）
```

### 11.2 Worker 配置

**位置**：`api/graphql/models/site_info.go:19-30`

```go
func DefaultSiteInfo(db *gorm.DB) SiteInfo {
    defaultConcurrentWorkers := 3
    if db_drivers.SQLITE.MatchDatabase(db) {
        defaultConcurrentWorkers = 1   // SQLite 强制单 worker，避免写锁冲突
    }
    return SiteInfo{
        ConcurrentWorkers: defaultConcurrentWorkers,
        // ...
    }
}
```

| 数据库类型 | 默认并发 worker 数 | 限制原因 |
|---|---|---|
| MySQL / PostgreSQL | 3 | 无强制限制，可在 UI 调整 |
| SQLite | 1 | SQLite 写锁是库级的，多 worker 写会导致 `database is locked` 错误 |

UI 配置入口：`ui/src/Pages/SettingsPage/ScannerConcurrentWorkers.tsx`，后端 Mutation：`setConcurrentWorkers`。

### 11.3 队列调度机制

**位置**：`api/scanner/scanner_queue/queue.go`

```go
type ScannerQueue struct {
    idle_chan   chan bool
    in_progress []ScannerJob   // 正在执行的任务
    up_next     []ScannerJob   // 待执行队列
    settings    ScannerQueueSettings {
        max_concurrent_tasks int  // = ConcurrentWorkers
    }
}
```

**调度逻辑**（`processQueue` 第 136-192 行）：
1. 从 `up_next` 队头取任务，直到 `in_progress` 达到 `max_concurrent_tasks`
2. 每个任务启动一个独立 goroutine 执行 `job.Run()` → `ScanAlbum()`
3. goroutine 完成后从 `in_progress` 移除，通知 `idle_chan` 触发下一轮调度

**关键特征**：
- 并发粒度是 **相册级**，不是文件级
- 单个相册内的所有媒体文件**串行处理**（`scanner_album.go:101` 的 `for` 循环）
- 单相册串行保证了：DB 连接复用、事务隔离简单、缓存命中高
- 极端场景：一个相册 10 万张图，其他相册空 → 并发度实际只有 1

### 11.4 处理管道的内部并发

`scanMedia` 内部的 12 个任务是**串行执行**的：

```
scanMedia
  ├── BeforeProcessMedia（串行过 12 个 task）
  ├── ProcessMedia（串行过 12 个 task）
  │     ├── CounterpartFilesTask
  │     ├── SidecarTask
  │     ├── ProcessPhotoTask / ProcessVideoTask
  │     ├── FaceDetectionTask  ← 全局单例，互斥锁保护
  │     ├── BlurhashTask
  │     ├── ExifTask           ← 全局互斥锁（exiftool 单实例）
  │     └── ...
  └── AfterProcessMedia（串行过 12 个 task）
```

**天然串行点**：
1. **EXIF 解析**（`api/scanner/externaltools/exif/exif.go:45`）：
   ```go
   var globalMu sync.Mutex
   func Parse(filepath string) (*models.MediaEXIF, error) {
       globalMu.Lock()
       defer globalMu.Unlock()
       // 单进程 exiftool 实例，所有 worker 串行排队
   }
   ```
2. **人脸检测**：`GlobalFaceDetector` 也是全局单例，推理时串行

### 11.5 吞吐量估算（10 万文件场景）

假设硬件：8 核 CPU / 16GB RAM / SSD 本地存储 / MySQL / ConcurrentWorkers = 4

| 阶段 | 单文件耗时 | 并发度 | 总耗时（10 万文件） | 瓶颈 |
|---|---|---|---|---|
| **遍历 + 查重**（Phase 1） | ~0.1 ms | 4（相册级） | **~2.5 秒** | `os.ReadDir` + `SELECT path_hash` |
| **仅新增**（第一次扫） | | | | |
| 图片缩略图生成 | ~100 ms | 4 | **~42 分钟** | MagickWand CPU 编解码 |
| EXIF 提取 | ~20 ms | 1（全局锁） | **~33 分钟** | exiftool 单进程 |
| 人脸检测 | ~500 ms | 1 | **~13.9 小时** | CNN 推理 CPU 密集 |
| Blurhash | ~5 ms | 4 | **~2 分钟** | 像素计算 |
| **全量处理**（10 万张图） | - | - | **~14-15 小时** | 人脸检测 |

**重复扫描**（文件无变化）：
- 已存在的文件会跳过 `ProcessMedia` 阶段，仅执行 `MediaFound`/`AfterMediaFound` 钩子
- 耗时：**~10-30 秒**（取决于目录树深度和 DB 速度）

**关键优化参数**：
| 参数 | 建议值（10 万+） | 影响 |
|---|---|---|
| `ConcurrentWorkers` | CPU 核心数 × 0.75 | 太小浪费 CPU，太大 DB 连接耗尽 |
| 禁用视频转码 | `PHOTOVIEW_DISABLE_VIDEO_ENCODING=1` | 视频转码是最慢环节 |
| 禁用人脸检测 | `PHOTOVIEW_DISABLE_FACE_DETECTION=1` | 消除最大瓶颈（14h → 1h） |
| GPU 加速 | 配置 VA-API / NVENC | 视频转码加速 5-10 倍 |

### 11.6 周期性扫描

**位置**：`api/scanner/periodic_scanner/periodic_scanner.go`

- 由 `PeriodicScanInterval`（秒）控制，0 表示禁用
- 触发逻辑：`time.Ticker` + `select`，到期后调用 `AddAllToQueue()`
- 如果上一轮扫描尚未完成，新任务会进入 `up_next` 队列等待，不会重复并发

---

## 12. 跨文件系统场景：NFS/SMB 挂载下的行为分析

Photoview **没有针对网络文件系统（NFS/SMB/CIFS）做特殊适配**，所有文件操作直接通过 Go 标准库 `os` 包完成。以下是实际行为分析。

### 12.1 符号链接处理

**位置**：`api/utils/utils.go:68-92`

```go
func IsDirSymlink(linkPath string) (bool, error) {
    fileInfo, err := os.Lstat(linkPath)          // 不跟随 symlink
    if fileInfo.Mode()&os.ModeSymlink == os.ModeSymlink {
        resolvedPath, err := filepath.EvalSymlinks(linkPath)  // 解析目标
        if err != nil {
            return false, fmt.Errorf("cannot resolve symlink... skipping it")
        }
        resolvedFile, err := os.Stat(resolvedPath)
        return resolvedFile.IsDir(), nil
    }
    return false, nil
}
```

**NFS/SMB 下的风险**：
- NFS 挂载选项 `nosymfollow` 会导致 `EvalSymlinks` 失败 → 降级为 `isDirSymlink = false`，该目录被跳过
- SMB 的 DFS 符号链接在某些内核版本上解析失败
- 解析失败后 **不中断扫描**，仅记录 warn 日志并假装不是目录符号链接

### 12.2 元数据操作的稳定性

| 操作 | 本地 FS | NFS v3 | NFS v4 | SMB 3.x | 备注 |
|---|---|---|---|---|---|
| `os.ReadDir` | 微秒级 | 毫秒级/文件 | 毫秒级/文件 | 毫秒级/文件 | 大目录（>1 万文件）可能秒级超时 |
| `os.Stat` | < 1µs | 1-10ms | 1-10ms | 5-20ms | 受 `actimeo` 属性缓存影响 |
| `ModTime` 精度 | 纳秒 | 秒级 | 纳秒（依赖服务端） | 100ns | NFS v3 可能导致 EXIF 时间偏差 |
| `os.Open` 读文件 | 块设备缓存 | 网络传输 + 客户端缓存 | 同左 | 同左 | 大文件（>50MB）可能超时 |

**NFS 特有问题**：
- **Stale NFS file handle**：服务端重启或 inode 回收后，扫描到一半报错
- **属性缓存不一致**：`actimeo=60` 时，文件已删但属性缓存还在 → 扫描到 "幽灵文件" → `os.Open` 失败
- **ETIMEDOUT / EHOSTDOWN**：网络抖动导致单个文件失败

**SMB 特有问题**：
- **文件锁定语义差异**：Windows 客户端的独占锁在 Linux cifs 客户端上表现为 `EBUSY`
- **大小写不敏感**：`IMG.jpg` 和 `img.jpg` 在 SMB 上是同一文件，但 path_hash 不同 → 可能重复扫描或死循环

### 12.3 现有容错机制

**位置**：`api/scanner/scanner_album.go:163-166`

```go
if err != nil {
    scanner_utils.ScannerError(ctx, "Error scanning media for album (%d): %s\n",
        ctx.GetAlbum().ID, err)
    continue  // 单个文件失败，继续下一个
}
```

- 单个文件的 `os.Stat` / `os.Open` 失败**不会中止整个相册扫描**
- 错误被记录到日志，该文件在本次扫描中被跳过
- 下一次全量扫描时会重试

**未处理的边缘情况**：
1. NFS 挂载点完全失联 → `os.ReadDir` 整个相册失败 → 该相册所有 media 会被 `CleanupMedia` **误判为删除**
2. `actimeo` 缓存过期导致同一文件在一次扫描中 `Stat` 两次结果不一致
3. SMB 大小写折叠导致的 path_hash 冲突

### 12.4 运营建议（针对 NFS/SMB 部署）

1. **NFS 挂载参数**：
   ```bash
   mount -t nfs -o vers=4,sec=sys,actimeo=1,noatime,hard,timeo=600,retrans=3 \
       nfs-server:/photos /media/photos
   ```
   - `actimeo=1`：属性缓存 1 秒，减少不一致窗口
   - `hard`：避免软挂载导致的偶发 I/O 错误
   - `vers=4`：推荐 NFS v4，锁和属性语义更完善

2. **SMB 挂载参数**：
   ```bash
   mount -t cifs -o vers=3.0,username=xxx,password=xxx,uid=1000,gid=1000,file_mode=0644,dir_mode=0755,cache=strict \
       //smb-server/photos /media/photos
   ```

3. **避免误删除**：
   - 网络不稳定时**先暂停周期性扫描**（`PeriodicScanInterval = 0`）
   - 重要数据建议做**本地缓存副本**再扫描

4. **大目录拆分**：
   - NFS/SMB 上单目录不要超过 1 万文件
   - 按年/月分子目录（`2024/06/IMG_xxx.jpg`）

---

## 13. 并发写入场景：扫描与上传的竞态保护

### 13.1 典型竞态场景

```
用户上传文件                    后台扫描线程
──────────                    ───────────
1. 创建 /album/IMG_123.jpg.tmp
2. 写入数据...
3. 完成后 rename → IMG_123.jpg    ← 原子操作
                                4. os.ReadDir 看到 IMG_123.jpg
                                5. ScanMedia(path_hash) → 未命中，判定为新增
                                6. os.Stat 获取 ModTime
                                7. 开始 ProcessMedia
                                8. MagickWand.ReadImage()
```

如果第 8 步发生时用户还在写入（rename 还没发生），会读到不完整文件。

### 13.2 现有保护机制

#### 机制 1：空文件跳过

**位置**：`api/scanner/scanner_cache/cache.go:108-127`

```go
func (c *AlbumScannerCache) IsPathMedia(mediaPath string) bool {
    // ...
    // Make sure file isn't empty
    fileStats, err := os.Stat(mediaPath)
    if err != nil || fileStats.Size() == 0 {
        return false   // ← 空文件被当作非媒体跳过
    }
    return true
}
```

**保护范围**：
- ✓ 写入刚开始、文件大小为 0 的场景
- ✗ 写入到一半、大小 > 0 但内容不完整的场景
- ✗ 上传工具先分配空间再写入（预分配空洞文件）的场景

#### 机制 2：作业去重

**位置**：`api/scanner/scanner_queue/queue.go:253-264`

```go
func (queue *ScannerQueue) jobOnQueue(job *ScannerJob) (bool, error) {
    scannerJobs := append(queue.in_progress, queue.up_next...)
    for _, scannerJob := range scannerJobs {
        if scannerJob.ctx.GetAlbum().ID == job.ctx.GetAlbum().ID {
            return true, nil   // 同一相册不会同时入队两次
        }
    }
    return false, nil
}
```

**保护范围**：
- ✓ 防止同一相册被重复扫描
- ✗ 不涉及文件级并发

#### 机制 3：PathHash 唯一索引

**位置**：`api/graphql/models/media.go:19`

```go
PathHash string `gorm:"not null;unique"`
```

**保护范围**：
- ✓ 防止同一文件路径重复插入（即使两个 goroutine 同时扫描到）
- ✗ 存在 TOCTOU（Time Of Check, Time Of Use）竞态窗口

```go
// ScanMedia 中的查-插窗口
// T1: SELECT WHERE path_hash = X → 空
// T2: SELECT WHERE path_hash = X → 空（T1 还没插入）
// T1: INSERT → 成功
// T2: INSERT → 唯一键冲突错误 → 该文件被跳过但未回滚
```

#### 机制 4：数据库事务

**位置**：`api/scanner/scanner_task/scanner_task.go:71-75`

```go
func (c TaskContext) DatabaseTransaction(transFunc func(ctx TaskContext) error, opts ...*sql.TxOptions) error {
    return c.GetDB().Transaction(func(tx *gorm.DB) error {
        return transFunc(c.WithDB(tx))
    }, opts...)
}
```

- `ScanMedia`（`scanner_album.go:148`）和 `scanMedia`（`media_scan.go:22`）都在事务内
- GORM 默认隔离级别：由数据库驱动决定（MySQL 是 `REPEATABLE READ`，PostgreSQL 是 `READ COMMITTED`）
- 事务只保护 DB 操作的原子性，**不保护文件系统操作**

#### 机制 5：缓存锁

**位置**：`api/scanner/scanner_cache/cache.go:16`

```go
type AlbumScannerCache struct {
    mutex sync.Mutex
    path_contains_photos map[string]bool
    photo_types          map[string]media_type.MediaType
    ignore_data          map[string][]string
}
```

所有缓存读写都受 `sync.Mutex` 保护，避免并发 map 读写 panic。

### 13.3 未覆盖的竞态窗口

| 场景 | 风险 | 后果 |
|---|---|---|
| 写入过程中文件大小 > 0 但内容不完整 | `ReadImage` 读到坏数据 | 缩略图生成失败，该媒体标记为错误 |
| 扫描期间文件被移动（跨相册 rename） | 旧相册没扫到，新相册也没扫到 | 文件暂时从库中消失，下次扫描恢复 |
| 扫描期间文件被删除 | `os.Stat` 成功但 `os.Open` 失败 | 单文件处理失败，记录错误 |
| 两个用户共享同一相册，同时触发扫描 | path_hash 唯一键冲突 | 其中一方失败，文件漏处理 |
| Sidecar `.xmp` 正在写入时被读取 | 解析到不完整的 XML | EXIF 字段缺失或解析错误 |

### 13.4 运营建议

1. **上传端原子写入**：
   ```bash
   # ✗ 不推荐：直接写入目标路径
   cp local.jpg /media/album/IMG_123.jpg

   # ✓ 推荐：先写临时文件再 rename
   cp local.jpg /media/album/IMG_123.jpg.upload
   mv /media/album/IMG_123.jpg.upload /media/album/IMG_123.jpg
   ```
   POSIX 中同目录 rename 是原子操作，可避免读到半写文件。

2. **临时文件过滤**：
   在 `.photoviewignore` 中添加：
   ```
   *.upload
   *.tmp
   *.part
   .DS_Store
   ```

3. **上传与扫描窗口错开**：
   - 上传集中在 0-6 点，扫描配置在 6-8 点执行
   - 或上传完成后通过 API 手动触发单用户扫描

4. **Sidecar 写入保护**：
   Lightroom/ Capture One 导出 XMP 时也遵循 tmp → rename 模式

---

## 14. Sidecar XMP/JSON 元数据变更识别与批量标签修改

### 14.1 Sidecar 文件支持范围

Photoview 仅支持 **XMP 格式** 的 sidecar 文件，不支持 JSON 格式。

**位置**：`api/scanner/scanner_tasks/processing_tasks/sidecar_task.go:133-141`

```go
func scanForSideCarFile(path string) *string {
    testPath := path + ".xmp"   // 仅匹配 <photo>.xmp
    if scanner_utils.FileExists(testPath) {
        return &testPath
    }
    return nil
}
```

**匹配规则**：
- 仅查找 `<photo_filename>.xmp`，即照片同名加 `.xmp` 后缀
- 不支持 JSON sidecar、不支持 XML sidecar、不支持 Lightroom 目录级别的 `.lrcat`
- 大小写敏感（Linux 下 `IMG.jpg.XMP` 不会被匹配）

### 14.2 变更识别算法：内容 MD5 哈希

Sidecar 是系统中**唯一使用内容哈希**做变更检测的模块（见第 7 章）。此处补充细节：

**哈希计算位置**：`sidecar_task.go:143-160`

```go
func hashSideCarFile(path *string) *string {
    f, _ := os.Open(*path)
    defer f.Close()
    h := md5.New()
    io.Copy(h, f)           // 流式读取整个 XMP 文件内容
    hash := hex.EncodeToString(h.Sum(nil))
    return &hash
}
```

**存储位置**：`models.Media.SideCarHash`（`media.go:30`）

```go
type Media struct {
    // ...
    SideCarPath     *string
    SideCarHash     *string   `gorm:"unique"`   // XMP 文件的 MD5
}
```

### 14.3 变更触发后实际执行的操作

Sidecar 变更**不会重新解析 EXIF**，也**不会触发人脸识别**，仅做两件事：

**位置**：`sidecar_task.go:85-130`

| 操作 | 是否执行 | 说明 |
|---|---|---|
| 重新生成高分辨率 JPEG | ✅ | `generateSaveHighResJPEG`，会应用新的 XMP 调色参数 |
| 重新生成缩略图 JPEG | ✅ | `generateSaveThumbnailJPEG` |
| 更新 MediaURL.Width/Height/FileSize | ✅ | XMP 可能裁剪了图片，尺寸会变 |
| 更新 `SideCarHash` 到 DB | ✅ | 保存新哈希避免下次重复处理 |
| 重新解析 EXIF | ❌ | EXIF 仅在 `newMedia=true` 时解析一次 |
| 重新计算 DateShot | ❌ | 时间戳只在首次入库时由 EXIF 或文件 ModTime 决定 |
| 重新人脸识别 | ❌ | 人脸检测在 FaceDetectionTask 中独立执行 |
| 重新计算 Blurhash | ❌ | 不涉及 |

**关键结论**：XMP 变更仅影响**视觉输出（缩略图/高分辨率图）**，不影响数据库中的 EXIF 元数据字段。

### 14.4 EXIF 解析的执行时机

对比 `ExifTask.AfterMediaFound`（`exif_task.go:19-29`）：

```go
func (t ExifTask) AfterMediaFound(ctx, media, newMedia bool) error {
    if !newMedia {          // 仅在媒体首次入库时执行
        return nil
    }
    SaveEXIF(ctx.GetDB(), media)
    return nil
}
```

**EXIF 解析条件矩阵**：

| 场景 | newMedia | SideCarHash 变化 | EXIF 重新解析 |
|---|---|---|---|
| 文件第一次被扫描 | true | 不相关 | ✅ |
| 文件已存在，仅 XMP 被修改 | false | 是 | ❌ |
| 文件已存在，XMP 未变 | false | 否 | ❌ |
| 文件改名（等效新文件） | true | 不相关 | ✅ |

**运营影响**：
- 在 Lightroom 中修改「描述」「关键词」「标题」等仅存于 XMP 的字段 → **数据库不更新这些字段**（Photoview 不解析 XMP 标签字段，仅用 XMP 调色）
- 修改 EXIF（如相机信息、GPS、拍摄时间）后，需**先删除 DB 中该媒体记录或触发重新扫描**才能生效

### 14.5 EXIF 解析实际提取的字段

**位置**：`api/scanner/externaltools/exiftool/values.go` + `api/scanner/externaltools/exif/exif.go:53-94`

Photoview 从 EXIF/XMP 中提取以下字段存入 `MediaEXIF` 表：

| EXIF 字段 | DB 列 | 来源 |
|---|---|---|
| Model / Make | Camera, Maker | EXIF |
| LensModel | Lens | EXIF |
| ISO / Flash / Orientation | Iso, Flash, Orientation | EXIF |
| ExposureProgram / ExposureTime | ExposureProgram, Exposure | EXIF |
| Aperture / FocalLength | Aperture, FocalLength | EXIF |
| ImageDescription | Description | EXIF |
| DateTimeOriginal / SubSecDateTimeOriginal | DateShot, OffsetSecShot | EXIF（优先级最高） |
| GPSLatitude / GPSLongitude | GPSLatitude, GPSLongitude | EXIF |

**注意**：Photoview **不解析 XMP 中的 `dc:subject`（关键词/标签）、`lr:hierarchicalSubject`（层级关键字）、`xmp:Rating`（评分）** 等元数据。这些字段在 Lightroom 批量修改后，Photoview 数据库中**完全不可见**。

### 14.6 批量修改标签的实际影响

运营常见场景：用 Lightroom 对 1 万张图批量修改关键词/评分/调色，随后触发全量重扫。

| 修改项 | 全量重扫是否生效 | 性能影响 |
|---|---|---|
| 调色（曝光/色温/裁剪） | ✅ 缩略图和高分辨率图会重新生成 | 大！1 万张图全部重新编码 JPEG |
| XMP 描述/标题 | ❌ DB 中 Description 不更新（仅首次扫时从 EXIF 读） | 同上（仍会重编码，因为 XMP 哈希变了） |
| XMP 关键词/标签 | ❌ Photoview 不存这些字段 | 同上（仍会重编码） |
| XMP 评分/颜色标签 | ❌ Photoview 不存这些字段 | 同上（仍会重编码） |
| EXIF GPS / 拍摄时间 | ❌ EXIF 仅 newMedia 时解析一次 | 同上（XMP 哈希变则重编码，但 EXIF 字段不更新） |

**关键结论**：批量改标签后执行全量重扫，**只会重新生成所有缩略图/高分辨率图（耗时几小时），但数据库中的 EXIF 元数据完全不变**。如果运营目的是更新 DB 中的描述/关键词，当前架构**无法实现**。

### 14.7 RAW + JPEG 配对文件的 sidecar 行为

`CounterpartFilesTask`（`counterpart_files_task.go`）处理 RAW + JPEG 配对：

```
场景：
  IMG_0001.ARW (SONY RAW)
  IMG_0001.JPG (对应 JPEG)
  IMG_0001.JPG.xmp (sidecar)

行为：
  - JPEG 被识别为 RAW 的 counterpart 而跳过（不入库）
  - 仅 RAW 入库，但其编码处理使用 JPEG 作为 baseImagePath
  - sidecar 关联到 RAW 文件的 media 记录上
```

**修改 RAW 的 XMP 调色 → 实际上重编码的是 counterpart JPEG**。

---

## 15. 媒体删除：软删（Trash）与硬删（Hard Delete）恢复路径

### 15.1 系统不存在软删机制

经过完整代码分析，Photoview **没有实现 Trash/回收站/软删 功能**。

**证据 1：Model 基类无 DeletedAt**
`api/graphql/models/base.go:7-15`

```go
type Model struct {
    ID int `gorm:"primarykey"`
    ModelTimestamps
}

type ModelTimestamps struct {
    CreatedAt time.Time
    UpdatedAt time.Time
    // 没有 DeletedAt time.Time → GORM 不会启用软删
}
```

GORM 软删需要嵌入 `gorm.Model` 或手动添加 `DeletedAt gorm.DeletedAt` 字段，当前基类均无。

**证据 2：所有删除操作均为 DELETE SQL**

`CleanupMedia`（`cleanup_media.go:52`）：
```go
db.Where("id IN (?)", mediaIDs).Delete(models.Media{})
// 执行：DELETE FROM media WHERE id IN (...)
```

`DeleteOldUserAlbums`（`cleanup_media.go:111-121`）：
```go
tx.Where("album_id IN (?)", deleteAlbumIDs).Delete(&models.UserAlbums{})
tx.Where("id IN (?)", deleteAlbumIDs).Delete(models.Album{})
```

均为**物理删除**，无 `UPDATE deleted_at = NOW()` 软删语句。

### 15.2 删除操作的完整链路

```
CleanupMedia(albumId, albumMedia)
  │
  ├── 1. 计算差集：DB media - albumMedia = 待删除集合
  │
  ├── 2. 删除磁盘缓存
  │     cachePath = <MediaCachePath>/<albumId>/<mediaId>/
  │     os.RemoveAll(cachePath)       ← 递归删除缩略图/高分辨率图
  │
  ├── 3. 删除 DB 记录
  │     DELETE FROM media WHERE id IN (...)
  │     └── GORM 级联 OnDelete:CASCADE
  │           ├── DELETE FROM media_urls        (缩略图/高分辨率元数据)
  │           ├── DELETE FROM media_exif         (EXIF 信息)
  │           ├── DELETE FROM image_faces        (人脸识别结果)
  │           └── DELETE FROM video_metadata     (视频元数据)
  │
  └── 4. 重新加载人脸检测器索引
        face_detection.GlobalFaceDetector.ReloadFacesFromDatabase(db)
```

**级联删除配置**（`media.go:21-31`）：
```go
type Media struct {
    Album           Album          `gorm:"constraint:OnDelete:CASCADE;"`
    Exif            *MediaEXIF     `gorm:"constraint:OnDelete:CASCADE;"`
    MediaURL        []MediaURL     `gorm:"constraint:OnDelete:CASCADE;"`
    VideoMetadata   *VideoMetadata `gorm:"constraint:OnDelete:CASCADE;"`
    Faces           []*ImageFace   `gorm:"constraint:OnDelete:CASCADE;"`
}
```

### 15.3 Trash 软删 vs Hard Delete 对比

| 维度 | 理想 Trash 软删 | 当前实现（Hard Delete） |
|---|---|---|
| DB 操作 | `UPDATE media SET deleted_at = NOW()` | `DELETE FROM media` |
| 缓存目录 | 移动到 trash/ 或保留 | `os.RemoveAll` 立即删除 |
| 恢复路径 | `UPDATE media SET deleted_at = NULL` + 重建缓存 | ❌ 无法恢复 |
| UI 可见性 | 被 WHERE deleted_at IS NULL 过滤 | 记录已不存在 |
| 用户收藏/标签 | 保留在 user_media_data 表 | 可能成为孤儿（无外键级联） |
| 误删恢复时间 | 秒级 | 不可恢复，只能重新扫描（丢失人工标签） |

### 15.4 当前唯一"恢复路径"

由于是硬删，**Photoview 自身无恢复能力**。唯一可行路径：

```
方案 A：从数据库备份恢复
  ├── 前提：定期备份 PostgreSQL/MySQL
  ├── 操作：还原备份 → 重新扫描未备份期间的增量
  └── 丢失：备份时间点之后的人工标签/收藏/分享链接

方案 B：重新扫描源文件
  ├── 前提：源文件仍在磁盘上
  ├── 操作：触发全量扫描，文件重新入库
  └── 丢失：所有人工元数据（收藏、分享、人脸标注、相册封面设置）全部清零

方案 C：缓存目录手动恢复
  ├── 前提：有文件系统备份（如 ZFS snapshot / Time Machine）
  └── 操作：还原 <MediaCachePath>/<albumId>/<mediaId>/ 目录
      （但 DB 记录已删，仅还原缓存无意义，需配合方案 A+B）
```

### 15.5 删除的触发入口

| 触发方式 | 调用方 | 行为 |
|---|---|---|
| 自动扫描 | `MediaCleanupTask.AfterScanAlbum` | 差集计算后硬删 |
| 用户触发扫描 | `scanner_queue.AddUserToQueue` | 同上 |
| 周期性扫描 | `periodic_scanner.AddAllToQueue` | 同上 |
| UI 删除按钮 | ❌ 无此功能 | - |
| GraphQL 删除 Mutation | ❌ 无 `deleteMedia` mutation | - |

**注意**：当前系统**没有提供用户主动删除单张图片的 API**，所有删除都是扫描时「源文件已不存在」的被动触发。

---

## 16. 专辑封面与缩略图：源文件删除后的孤儿清理

### 16.1 专辑封面的引用方式

**数据结构**（`album.go:10-21`）：

```go
type Album struct {
    // ...
    CoverID  *int       // 指向 media.id，可为 NULL
}
```

**封面设置 API**（`album_actions.go:139-166`）：

```go
func SetAlbumCover(db, user, mediaID) (*Album, error) {
    db.Find(&media, mediaID)              // 校验 media 存在
    db.Find(&album, media.AlbumID)        // 校验用户有权限
    db.Model(&album).Update("cover_id", mediaID)
}

func ResetAlbumCover(db, user, albumID) (*Album, error) {
    db.Model(&album).Update("cover_id", nil)  // 置 NULL
}
```

**封面读取逻辑**（`album.go:83-115`，使用递归 CTE）：

```go
func (a *Album) Thumbnail(db) (*Media, error) {
    // 1. 如果 CoverID 有值，优先使用指定封面
    if a.CoverID != nil {
        db.First(&media, *a.CoverID)
        if err == nil { return &media, nil }
        // CoverID 指向的 media 不存在 → 继续执行兜底逻辑
    }

    // 2. 兜底：取该相册（含子相册）下最新的一张 media
    query := `
        WITH RECURSIVE sub_albums AS (...)
        SELECT * FROM media
        WHERE media.album_id IN (SELECT id FROM sub_albums)
        ORDER BY media.id DESC
        LIMIT 1
    `
    db.Raw(query, a.ID).Scan(&media)
    return &media, nil
}
```

### 16.2 封面成为孤儿的场景

```
场景 1：CoverID 指向的 media 被 CleanupMedia 删除
  ├── 触发：源文件被删，扫描后 DB 执行 DELETE FROM media
  ├── 结果：album.cover_id 仍指向已删除的 media.id（悬挂指针）
  └── 读取：Album.Thumbnail() 中 db.First 失败 → 自动回退到兜底逻辑

场景 2：CoverID 指向的 media 因改名被删+重建
  ├── 触发：源文件改名，旧 media.id 被删，新 media.id 生成
  ├── 结果：album.cover_id 指向旧的（已不存在）media.id
  └── 读取：同上，兜底为最新图片

场景 3：整个相册被删除（DeleteOldUserAlbums）
  ├── 触发：相册目录被删，扫描后 Album 记录被 DELETE
  ├── 结果：子相册 CoverID 关联父相册的 media 可能也失效
  └── 但 Album 记录本身已删，不影响
```

### 16.3 孤儿 CoverID 的清理流程

**结论：Photoview 没有显式的孤儿 CoverID 清理流程。**

现有代码中**无任何地方**执行：
```go
// 不存在这样的代码
db.Where("cover_id NOT IN (SELECT id FROM media)").Update("cover_id", nil)
```

**CoverID 的容错依赖读取时兜底**：

```
用户访问 Album → resolver 调用 Album.Thumbnail()
  → db.First(media, CoverID) → 失败（gorm.ErrRecordNotFound）
  → 执行兜底 SQL：SELECT ... ORDER BY media.id DESC LIMIT 1
  → 返回子树中最新一张图片作为封面
  → 但 album.cover_id 仍为旧值，下次访问仍重复此过程
```

**运营影响**：
- 封面图片被删后，相册**不会显示空白**（兜底逻辑生效）
- 但会出现「封面不一致」：UI 实际显示的图片 ≠ `album.cover_id` 指向的图片
- 大量孤儿 CoverID 会导致每次封面读取都多一次无效的 DB 查询

### 16.4 缩略图缓存（MediaCache）的孤儿清理

缩略图缓存目录结构：`api/utils/media_cache.go:59-73`

```
<MediaCachePath>/                  默认: ./media_cache
├── <albumId>/
│   ├── <mediaId>/
│   │   ├── thumbnail_xxx.jpg
│   │   ├── highres_xxx.jpg
│   │   └── video-web_xxx.mp4
```

**清理逻辑**（与 Media 删除同步执行）：

`CleanupMedia` 中（`cleanup_media.go:40-48`）：
```go
for _, media := range mediaList {
    cachePath := path.Join(utils.MediaCachePath(),
        strconv.Itoa(albumId), strconv.Itoa(media.ID))
    os.RemoveAll(cachePath)    // ← 精确匹配 albumId + mediaId 删除
}
```

`DeleteOldUserAlbums` 中（`cleanup_media.go:100-108`）：
```go
for _, album := range deleteAlbums {
    cachePath := path.Join(utils.MediaCachePath(), strconv.Itoa(album.ID))
    os.RemoveAll(cachePath)    // ← 删除整个相册缓存目录
}
```

### 16.5 媒体缩略图（MediaURL）的孤儿清理

`media_urls` 表通过外键约束自动清理：

`media.go:24`：
```go
MediaURL []MediaURL `gorm:"constraint:OnDelete:CASCADE;"`
```

→ 删除 Media 记录时，数据库自动级联删除关联的 `media_urls` 行。

### 16.6 未被清理的孤儿类型总结

| 孤儿类型 | 是否自动清理 | 清理机制 | 影响 |
|---|---|---|---|
| `album.cover_id` 指向已删除 media | ❌ 否 | 仅读取时兜底，DB 值仍脏 | 无效 DB 查询 |
| `media_cache/<albumId>/` 目录（Album 被删） | ✅ 是 | `DeleteOldUserAlbums → os.RemoveAll` | - |
| `media_cache/<albumId>/<mediaId>/` 目录（Media 被删） | ✅ 是 | `CleanupMedia → os.RemoveAll` | - |
| `media_urls` 表记录（Media 被删） | ✅ 是 | DB ON DELETE CASCADE | - |
| `media_exif` / `image_faces` 表记录 | ✅ 是 | DB ON DELETE CASCADE | - |
| `user_media_data` 用户收藏/标签 | ❓ 依赖外键 | 需检查 user_media_data 是否有级联 | 可能成为孤儿 |
| `share_tokens` 分享链接 | ❓ 依赖外键 | 需检查 share_tokens 是否有级联 | 可能成为孤儿 |
| 空的 `media_cache/<albumId>/` 目录（Album 下 media 全删光） | ❌ 否 | 无定期清理空目录逻辑 | 占用少量 inode |

### 16.7 运营建议：CoverID 孤儿修复

```sql
-- 定期执行（如每天 cron），清理悬挂的 album.cover_id
UPDATE albums
SET cover_id = NULL
WHERE cover_id IS NOT NULL
  AND cover_id NOT IN (SELECT id FROM media);
```

```bash
# 定期清理空的缓存目录
find ./media_cache -type d -empty -delete
```

---

## 17. 扫描过程中的 GC 行为与磁盘占用监控

### 17.1 GC（垃圾回收）监控现状

Photoview 代码中**没有任何显式的 GC 触发或内存监控逻辑**。在整个 `api/` 目录下 grep 以下关键字均无业务层调用：

| 关键字 | 命中情况 |
|---|---|
| `runtime.GC()` | ❌ 无 |
| `runtime.ReadMemStats` | ❌ 无 |
| `debug.FreeOSMemory` | ❌ 无 |
| `debug.SetGCPercent` | ❌ 无 |
| `GOGC` 环境变量使用 | ❌ 无 |

**GC 完全依赖 Go 运行时默认行为**（触发阈值：堆增长 100% 或 2 分钟定时器）。扫描过程中 Go 的 GC 不会被主动干扰。

### 17.2 扫描过程中的主要内存消耗点

按量级从大到小排序：

#### 1. 人脸检测内存（最大）
**位置**：`face_detection/face_detector_impl.go:17-23`

```go
type faceDetector struct {
    mutex           sync.Mutex
    rec             *face.Recognizer
    faceDescriptors []face.Descriptor   // 每张人脸 128 个 float32 = 512 字节
    faceGroupIDs    []int32
    imageFaceIDs    []int
}
```

- 每张照片平均 2-3 张人脸，10 万张照片约 25 万张人脸 → `faceDescriptors` 占用约 **128 MB**
- `face.Recognizer`（dlib 模型）自身常驻约 **200-300 MB**
- 每次 `ReloadFacesFromDatabase` 时旧切片被 GC 回收，但峰值会短暂翻倍

#### 2. MagickWand 像素缓冲区
**位置**：`media_encoding/executable_worker/magickwand.go`

- 处理 24MP 照片（6000×4000）解码为 RGB：`6000*4000*3 = 72 MB / 张`
- 每个相册 worker 同时只处理 1 张，但 `ConcurrentWorkers = 4` 时峰值可达 **288 MB**
- MagickWand C 库内存不由 Go GC 管理，依赖 `DestroyMagickWand()` 手动释放

#### 3. ffmpeg 视频转码临时文件
- 视频转码时 `ffmpeg` 子进程独立占用内存，不受 Go GC 控制
- 1080p MP4 转码约占 **200-500 MB / 进程**

#### 4. 缩略图/高分辨率图生成的中间 JPEG
- 单张高分辨率 JPEG 约 **5-15 MB**
- 存入 `[]byte` 后写磁盘，函数返回时被 GC

#### 5. 目录遍历缓冲区
- `os.ReadDir` 返回的 `[]DirEntry` 对大目录（1 万文件）约 **几百 KB**
- 遍历完即释放

### 17.3 磁盘占用监控现状

**同样无任何磁盘空间/容量检测代码**。grep 关键字结果：

| 关键字 | 命中情况 |
|---|---|
| `statfs` / `unix.Statfs` | ❌ 无 |
| `disk.Available` | ❌ 无 |
| `AvailableSpace` | ❌ 无 |
| `df` 命令调用 | ❌ 无 |

以下高危操作**均不做磁盘空间预检查**：

```go
// 生成高分辨率 JPEG（process_photo_task.go）
// 直接写磁盘，无剩余空间判断
jpeg.Encode(file, img, &jpeg.Options{Quality: 95})

// ffmpeg 转码输出（ffmpeg_cli.go）
// 直接运行 ffmpeg -i input -o output.mp4，无空间预检查
cmd := exec.Command("ffmpeg", ..., outputPath)
```

### 17.4 磁盘空间不足的实际后果

| 操作 | 磁盘满时的表现 |
|---|---|
| 缩略图生成 | `jpeg.Encode` 返回 `ENOSPC` 错误 → 该 media 处理失败，记录错误日志，**但 media 已入库**（仅 `MediaURL` 缺失） |
| 高分辨率图生成 | 同上，缩略图可能已写成功但高分辨率图失败 → 状态不一致 |
| ffmpeg 视频转码 | ffmpeg 进程退出码非 0 → `ProcessVideoTask` 报错，media 已入库但无播放 URL |
| `CleanupMedia` 删除 | 成功释放空间（仅删文件不写） |
| EXIF 解析 | 只读，不受影响 |
| Blurhash 计算 | 只读，不受影响 |

**问题**：Media 记录已创建但 `MediaURL` 缺失，前端访问时返回 404 或空白。下次全量扫描不会修复，因为 `ScanMedia` 判定为「已存在」（path_hash 命中），不会重新走 `ProcessMedia`。

### 17.5 磁盘占用总量估算

以 10 万张 12MP JPEG（平均 5MB 源文件）为例：

| 项目 | 单文件大小 | 10 万文件总量 |
|---|---|---|
| 源文件（只读） | 5 MB | **500 GB** |
| 缩略图 thumbnail_1024.jpg | ~80 KB | **8 GB** |
| 缩略图 thumbnail_2048_high.jpg | ~300 KB | **30 GB** |
| 高分辨率图 highres.jpg | ~2-3 MB | **250 GB** |
| 缓存合计 | - | **~290 GB**（约源文件的 58%） |
| 数据库（含人脸 descriptor BLOB） | - | **~2-5 GB** |

**关键结论**：`PHOTOVIEW_MEDIA_CACHE` 目录需预留**源文件大小的 60%** 以上空间。

### 17.6 运营建议

**内存侧**：
```bash
# 建议在 systemd/docker 中显式设置
GOGC=50                    # 更积极的 GC（默认 100），降低峰值内存
GOMEMLIMIT=6GiB            # Go 1.19+ 软内存上限（优先推荐）
```

**磁盘侧**：
```bash
# 用外部监控（prometheus/node_exporter）告警
- node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"} < 10%
- 监控 PHOTOVIEW_MEDIA_CACHE 目录增长速率
```

**代码层建议**：
- 缩略图生成前执行 `unix.Statfs` 预估剩余空间，< 1GB 时暂停扫描并告警
- `ProcessMedia` 失败时标记 `media.status = 'partial'`，下次扫描重试失败项

---

## 18. 多用户多相册场景：扫描权限边界分析

### 18.1 权限模型总览

**核心关系表**：`user_albums`（多对多）

```
users ──< user_albums >── albums ──< media
          (user_id, album_id)         (album_id)
```

`models/user.go:19`：
```go
type User struct {
    Albums []Album `gorm:"many2many:user_albums;constraint:OnDelete:CASCADE;"`
    Admin  bool    `gorm:"default:false"`
}
```

**所有权判定**：`User.OwnsAlbum()`（`user.go:167-180`）

```go
func (user *User) OwnsAlbum(db *gorm.DB, album *Album) (bool, error) {
    filter := func(query *gorm.DB) *gorm.DB {
        return query.Where(
            "EXISTS (SELECT 1 FROM user_albums WHERE user_id = ? AND album_id = id LIMIT 1)",
            user.ID)
    }
    ownedParents, err := album.GetParents(db, filter)
    // 如果 album 本身或任意父级在 user_albums 中，则视为拥有
    return len(ownedParents) > 0, nil
}
```

→ **递归向上判断父目录是否在用户的相册列表中**，子相册自动继承父相册的所有权。

### 18.2 Root Album 的创建与重叠检测

**位置**：`scanner_album.go:19-73`

```go
func NewRootAlbum(db, rootPath, owner) (*Album, error) {

    // 1. 路径有效性检查
    if !ValidRootPath(rootPath) { return ErrorInvalidRootPath }

    // 2. 通过 path_hash 查是否已有该 Album
    db.Where("path_hash = ?", MD5Hash(rootPath)).Find(&matchedAlbums)

    for _, matchedAlbum := range matchedAlbums {

        // 3. 检测路径重叠：新路径不能是已有路径的子目录，反之亦然
        var count int64
        db.Raw(`
            SELECT COUNT(*) FROM user_albums
            JOIN albums ON user_albums.album_id = albums.id
            WHERE user_albums.user_id = ?
              AND (? LIKE albums.path || '%' OR albums.path LIKE ? || '%')
        `, owner.ID, rootPath, rootPath).Scan(&count)

        if count > 0 {
            return nil, errors.New("user already owns a path containing this path")
        }

        // 4. 已有 Album 但用户不拥有 → 直接关联（共享相册）
        db.Create(&UserAlbums{UserID: owner.ID, AlbumID: matchedAlbum.ID})
        return &matchedAlbum, nil
    }

    // 5. 新路径 → 创建 Album + UserAlbums 关联
    db.Create(&album)
    db.Create(&UserAlbums{UserID: owner.ID, AlbumID: album.ID})
    return &album, nil
}
```

**重叠检测矩阵**：

| 用户已有路径 | 新添加路径 | 结果 |
|---|---|---|
| `/photos/2024` | `/photos/2024/06` | ❌ 拒绝（包含关系） |
| `/photos/2024/06` | `/photos/2024` | ❌ 拒绝（被包含） |
| `/photos/2024` | `/photos/vacation` | ✅ 允许（平级） |
| `/photos/2024`（用户A） | `/photos/2024`（用户B） | ✅ 允许（共享 Album，不同 UserAlbums 行） |
| `/photos/2024`（用户A已有） | `/photos/2024`（用户A重复） | ✅ 复用已有 Album（幂等） |

### 18.3 扫描时的权限边界问题

#### 问题 1：FindAlbumsForUser 不校验每级目录的用户身份

**位置**：`scanner_user.go:45`

```go
func FindAlbumsForUser(db, user, cache) ([]*ScannerJob, error) {
    for _, rootAlbum := range user.Albums {
        // 从 rootAlbum.Path 开始 BFS 遍历
        // 遍历到的所有子目录全部创建/关联 Album
        // 过程中没有再次校验 user 是否有权限访问每个子目录
    }
}
```

潜在风险：如果用户 A 拥有 `/photos`，而 `/photos/private` 在文件系统层面只对用户 B 可读（Unix 权限 `0700`），扫描时 `os.ReadDir("/photos/private")` 会失败 → 整个扫描任务报错中止。

#### 问题 2：多用户共享 Album 时的 CleanupMedia 互影响

用户 A 和 B 共享同一个 Album（path_hash 相同，不同 UserAlbums 行）：

```
T1: 用户A触发扫描 → ScanAlbum(albumId=5)
T2: 用户B触发扫描 → ScannerQueue.jobOnQueue 检测 albumId=5 正在执行 → 跳过入队
T3: 用户A的扫描在 CleanupMedia 中删除 albumId=5 的过期 media
    → 对用户B也生效（因为 media 不按用户隔离）
```

→ **media 是全局共享的，CleanupMedia 以 album_id 为粒度，删除对所有共享该相册的用户同时生效**。这是合理设计，但需运营知晓。

#### 问题 3：人脸检测不按用户隔离

`ReloadFacesFromDatabase` 加载全库 `image_faces`：
```go
func getSamplesFromDatabase(db) (...) {
    db.Find(&imageFaces)   // 没有 WHERE user_id = X
}
```

→ 用户 A 的扫描触发人脸检测后，会把用户 B 的人脸 descriptor 也加载到内存。`RecognizeUnlabeledFaces` 中通过 JOIN `user_albums` 限制用户可见范围，但全局内存不隔离。

#### 问题 4：UserRemoveRootAlbum 的级联清理

`resolvers/user.go:193-244`：

```go
func UserRemoveRootAlbum(...) {
    // 1. 删除 user_albums 中 user 的关联（含子相册）
    // 2. cleanup(tx, albumID, childAlbumIDs):
    //    如果 album 没有任何 user 关联了
    //    → DELETE FROM albums → 级联删除所有 media
    //    → os.RemoveAll(media_cache/<albumId>/)
}
```

**边界场景**：
- 用户 A 和 B 共享相册 → A 移除根相册 → B 仍可正常访问
- 用户 A 是唯一拥有者 → A 移除根相册 → 相册及其下所有 media 被**硬删**（对所有用户生效）

### 18.4 权限边界总结表

| 场景 | 是否按用户隔离 | 风险 |
|---|---|---|
| Album 可见性 | ✅ 是（user_albums） | - |
| Media 可见性 | ✅ 是（通过 album 归属） | - |
| Media DB 记录共享 | ❌ 否（同 path 的 media 全局唯一） | 多用户共享同一路径时共用记录 |
| CleanupMedia 删除范围 | ❌ 否（按 album_id 全局删） | 用户A的扫描删除对B也生效 |
| 人脸检测内存 | ❌ 否（全库加载） | 内存不隔离，推理时 JOIN 过滤 |
| 用户收藏 `user_media_data` | ✅ 是（user_id 主键） | - |
| 缩略图缓存目录 | ❌ 否（按 albumId/mediaId） | 多用户共享 |
| Root Path 重叠检测 | ✅ 是（SQL LIKE） | 符号链接可能绕过检测 |

### 18.5 运营建议

1. **多用户共享同一物理目录**：推荐用 `UserAddRootPath` 让多个用户关联同一个 Album，避免重复扫描和重复缓存
2. **Unix 文件权限**：确保 photoview 进程用户（通常 `UID=1000`）对所有 Root Path 的所有子目录都有 `r-x` 权限，避免扫描中途 `EACCES`
3. **符号链接攻击面**：`NewRootAlbum` 的重叠检测用字符串 `LIKE` 比较，用户通过 symlink 将 `/public/link → /private/data` 指向已有路径可能绕过。建议在 `ValidRootPath` 中解析所有符号链接后用 `filepath.EvalSymlinks` 比较真实路径
4. **用户移除相册前提醒**：如果用户是该 Album 的唯一拥有者，UI 应二次确认「该相册将被永久删除，所有用户都无法访问」

---

## 19. 备份恢复场景：DB 与文件系统不一致时的扫描修复路径

### 19.1 Photoview 无原生备份/恢复机制

系统中**不存在**以下功能：

| 功能 | 代码中是否存在 |
|---|---|
| `backupDatabase` GraphQL Mutation | ❌ 无 |
| `restoreDatabase` GraphQL Mutation | ❌ 无 |
| 自动 DB 快照（每日/每周） | ❌ 无 |
| 缓存目录快照/版本管理 | ❌ 无 |
| schema 迁移回滚 | ❌ 仅有向前迁移（`database/migration_exif.go`、`database/migrations/`） |

运营备份完全依赖外部工具（`pg_dump`、`mysqldump`、SQLite 文件拷贝、ZFS snapshot、BorgBackup 等）。

### 19.2 常见不一致场景与修复路径

#### 场景 A：DB 有记录，文件系统无文件（媒体被误删）

```
恢复前状态：
  DB:    media 表有 id=123, path="/photos/2024/06/IMG_001.jpg"
  磁盘:  /photos/2024/06/IMG_001.jpg 不存在
```

**扫描修复路径**（全自动）：

```
触发全量扫描
  → FindAlbumsForUser（相册树正常，因为目录还在）
  → ScanAlbum(albumId=X)
    → findMediaForAlbum: os.ReadDir 不包含 IMG_001.jpg
    → albumMedia = [其它文件]
    → ...
    → AfterScanAlbum → CleanupMedia(albumId=X, albumMedia)
      → 差集 DB - albumMedia = {id=123}
      → os.RemoveAll(media_cache/X/123/)
      → DELETE FROM media WHERE id IN (123)
          级联删除：media_urls / media_exif / image_faces
      → ReloadFacesFromDatabase
```

**修复结果**：DB 与文件系统重新一致，但该媒体的人工元数据（收藏、分享、人脸标注）**全部丢失且无法通过扫描恢复**。

#### 场景 B：文件系统有文件，DB 无记录（DB 回滚到旧版本）

```
恢复前状态：
  DB:    旧备份，无 id=456 记录
  磁盘:  /photos/2024/07/IMG_999.jpg 存在
```

**扫描修复路径**（全自动）：

```
触发全量扫描
  → FindAlbumsForUser（目录结构可能需要重建 Album）
  → ScanAlbum(albumId=Y)
    → findMediaForAlbum: os.ReadDir 包含 IMG_999.jpg
    → ScanMedia("/photos/2024/07/IMG_999.jpg")
      → SELECT path_hash=MD5(...) → 空
      → INSERT INTO media (path=..., path_hash=..., album_id=Y)
      → return media, isNew=true
    → scanMedia（全套处理：缩略图/EXIF/人脸/Blurhash）
```

**修复结果**：文件被重新入库，但**新的 media.id ≠ 原来的 id=456**，导致：
- `album.cover_id` 如果原来指向 456 → 现在成为孤儿 CoverID
- `user_media_data.user_id + media_id=456` 的收藏 → 成为孤儿行（无外键级联）
- `share_tokens` 中的 media_id=456 → 指向不存在的媒体
- 人脸检测会重新识别，新的 face_group_id 与旧的不一致

#### 场景 C：DB 与文件系统都有，但路径变了（磁盘目录被重命名）

```
恢复前状态：
  DB:    media.path = "/photos/2024_trip/IMG.jpg", path_hash=MD5("/photos/2024_trip/IMG.jpg")
  磁盘:  目录被改名 → "/photos/2024_trip_renamed/IMG.jpg"
```

**扫描修复路径**（全自动，但等同删除+重建）：

```
触发全量扫描
  → FindAlbumsForUser
    → "/photos/2024_trip" 不存在 → DeleteOldUserAlbums 级联删除旧 Album + 所有 media
    → "/photos/2024_trip_renamed" 作为新目录 → 新建 Album，path_hash 不同
  → ScanAlbum(newAlbumId)
    → 所有文件 path_hash 不命中 → 全部作为新增，重新处理全套
```

**修复结果**：所有 Media ID、Album ID 全部重新生成，人工元数据 100% 丢失。

#### 场景 D：DB 回滚了，但缓存目录未同步（缓存孤儿）

```
恢复前状态：
  DB:        回滚到 7 天前，media 最大 id = 80000
  缓存目录:  media_cache/5/80123/  （有 id=80123 的缩略图）
  磁盘源文件: /photos/... 完整
```

**扫描修复路径**：

```
触发全量扫描
  → ScanMedia 遇到路径 MD5 不命中（因为 DB 是旧的）→ 重新插入，新 id ≈ 80001-80123
  → CleanupMedia 不会删除 80123（因为 DB 里根本没有 80123，查不到）
  → 结果：media_cache/5/80123/ 成为永久孤儿目录
```

**修复**：需要手动执行
```bash
# 方案 1：清空缓存，让扫描重新生成
rm -rf ./media_cache/*
# 方案 2：只清理孤儿（更精细）
find ./media_cache -mindepth 2 -maxdepth 2 -type d -exec bash -c '
    id=$(basename {})
    # 检查 DB 中是否存在该 media_id
    psql -c "SELECT 1 FROM media WHERE id = $id" | grep -q 1 || rm -rf {}
' \;
```

### 19.3 不一致类型与扫描修复能力总览

| 不一致类型 | 扫描能否自动修复 | 修复后人工元数据保留 | 缓存一致性 |
|---|---|---|---|
| DB 有，磁盘无（场景 A） | ✅ 能，CleanupMedia 清理 | ❌ 收藏/分享/标签丢失 | ✅ 同步删除 |
| 磁盘有，DB 无（场景 B） | ✅ 能，ScanMedia 重新入库 | ❌ 媒体 ID 变化，关联全部失效 | ❌ 旧缓存目录成孤儿 |
| 路径/文件名改变（场景 C） | ✅ 能，但等于全删全建 | ❌ 完全丢失 | ✅ 旧删新生成 |
| DB 有，缓存无（缓存被删） | ⚠️ 部分能 | N/A | ⚠️ 仅扫描到的变更媒体会重建缓存；**未变更媒体的缓存不重建** |
| DB 回滚，缓存未删（场景 D） | ❌ 不能 | ❌ ID 不匹配 | ❌ 孤儿缓存需手动清理 |
| Album 表乱了（目录结构变了） | ✅ 能，FindAlbumsForUser 重建 | ❌ Album ID 变，CoverID 失效 | ✅ 旧 albumId 目录被删 |
| 人脸表乱了（image_faces 被删） | ❌ 不能，仅新入库媒体做人脸检测 | N/A | N/A |
| SiteInfo / User 配置乱了 | ❌ 不能，扫描不涉及 | N/A | N/A |

### 19.4 标准备份恢复流程（推荐运营手册）

```
┌──────────────────────────────────────────────────────────────────┐
│                     标准恢复操作流程                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 停止服务                                                     │
│     systemctl stop photoview 或 docker-compose down              │
│                                                                  │
│  2. 恢复数据库                                                   │
│     pg_restore -d photoview photoview_2024-06-01.dump            │
│     或 mysql -u photoview -p photoview < backup.sql              │
│                                                                  │
│  3. （可选）恢复缓存目录                                          │
│     rsync -a --delete backup/media_cache/ ./media_cache/         │
│     跳过此步 → 让扫描重新生成（慢但一致性好）                      │
│                                                                  │
│  4. 启动服务                                                     │
│     systemctl start photoview                                    │
│                                                                  │
│  5. 触发全量扫描（所有用户）                                       │
│     调用 GraphQL: mutation { scanAllUsers }                      │
│     或等待周期性扫描触发                                          │
│                                                                  │
│  6. 扫描完成后，执行数据一致性校验                                  │
│     SELECT COUNT(*) FROM media;                                   │
│     -- 对比预期文件数量                                            │
│     find /photos -type f \( -name "*.jpg" ... \) | wc -l         │
│                                                                  │
│  7. 清理孤儿 CoverID                                              │
│     UPDATE albums SET cover_id = NULL                             │
│     WHERE cover_id NOT IN (SELECT id FROM media);                │
│                                                                  │
│  8. 清理孤儿缓存目录                                              │
│     find ./media_cache -mindepth 2 -maxdepth 2 -type d           │
│       -exec bash -c '[...]' \;                                   │
│                                                                  │
│  9. （可选）人工元数据从旧 DB 单独迁移                              │
│     INSERT INTO user_media_data SELECT * FROM old_db.user_media  │
│       ON CONFLICT DO NOTHING;                                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 19.5 代码层改进方向

- 引入「修复扫描」模式：不检测删除（跳过 CleanupMedia），仅补全缺失的 media 和缓存
- `ScanMedia` 中增加「如果 DB 有 path_hash 但 MediaURL 缺失则重新 ProcessMedia」的逻辑（修复场景 D 的部分情况）
- 提供 `orphanCleanup` GraphQL mutation，一次清理 CoverID 孤儿、空缓存目录、`user_media_data` 孤儿行
- 提供 `mediaRehash` mutation：当源文件路径批量改名后，基于文件内容哈希匹配旧记录并更新 `path/path_hash`（保留 ID 和元数据）

---

## 20. 原始文件元数据被外部工具改写时的检测方法

### 20.1 当前检测能力总览

Photoview 对「原始文件本身发生变化」的检测**几乎为零**。唯一例外是 sidecar `.xmp` 文件。以下是各层检测能力的完整梳理：

| 变更类型 | 当前是否检测 | 检测机制 | 代码位置 |
|---|---|---|---|
| 文件新增（新路径） | ✅ | `path_hash` 查重 | `scanner_media.go:28` |
| 文件删除（路径消失） | ✅ | 差集比对 | `cleanup_media.go:26-31` |
| 文件改名/移动 | ⚠️ 间接 | 等效删除+新增 | 同上 |
| Sidecar `.xmp` 内容变更 | ✅ | 内容 MD5 哈希 | `sidecar_task.go:72-79` |
| **原始文件内容被改写** | ❌ | 无 | - |
| **原始文件 EXIF/GPS 被改写** | ❌ | 无 | - |
| **原始文件被原地覆盖（同名同路径）** | ❌ | 无 | - |
| **原始文件大小/修改时间变化** | ❌ | 无 | - |
| **原始文件 inode 变化（编辑器保存）** | ❌ | 无 | - |

### 20.2 为什么 mtime 检测不准确

`ScanMedia`（`scanner_media.go:55-65`）中唯一一次读取 `os.Stat`：

```go
stat, err := os.Stat(mediaPath)
// ...
media := models.Media{
    Title:    mediaName,
    Path:     mediaPath,
    AlbumID:  albumId,
    Type:     mediaTypeText,
    DateShot: stat.ModTime(),    // 仅用于 DateShot 初始值
}
```

**这段代码只在 `newMedia=true` 时执行**。对已存在文件，`ScanMedia` 在第 28-37 行直接返回：

```go
result := tx.Where("path_hash = ?", models.MD5Hash(mediaPath)).Find(&media)
if result.RowsAffected > 0 {
    return media[0], false, nil   // 直接返回，不再 os.Stat
}
```

→ **已存在文件的 mtime 变化完全被忽略**。即使文件内容已彻底改变，只要路径不变，`ScanMedia` 不会触发任何重处理。

### 20.3 外部工具改写文件的典型场景

| 场景 | 触发工具 | 文件系统变化 | Photoview 感知 |
|---|---|---|---|
| JPEG 无损旋转 | `exiftran -ai`、`jhead -autorot` | 内容变、mtime 变、inode 不变 | ❌ 无感知 |
| EXIF 日期/ GPS 修正 | `exiftool -GPS*=...`、`exiv2` | 内容变、mtime 变、inode 不变 | ❌ 无感知 |
| 照片编辑后保存 | Lightroom、GIMP、Photoshop | 内容变、mtime 变、inode 可能变（tmp→rename） | ❌ 无感知 |
| 视频转码覆盖 | `ffmpeg -i in.mp4 -c:v ... out.mp4` | 内容变、大小变、mtime 变 | ❌ 无感知 |
| `touch` 修改 mtime | `touch -t 202401010000 IMG.jpg` | 仅 mtime 变、内容不变 | ❌ 无感知 |
| 文件系统 fsck 修复 | `fsck -y` | inode 可能变、内容可能变 | ❌ 无感知 |
| 日志轮转覆盖 | `logrotate` 式 rename+create | inode 变、内容变 | 被视为「已存在」 |

### 20.4 理论上的检测方法对比

| 检测方法 | 可靠性 | 性能开销 | 实现复杂度 | 适用场景 |
|---|---|---|---|---|
| **mtime + size 对比** | ⚠️ 中 | O(1) per file | 低 | 大多数编辑场景 |
| **inode + dev 对比** | ⚠️ 中 | O(1) per file | 低 | 检测「原地覆盖」（inode 变） |
| **文件内容 MD5/SHA256** | ✅ 高 | O(file_size) | 中 | 绝对可靠，但需全文件读取 |
| **头部采样哈希**（前 64KB） | ⚠️ 中-高 | O(64KB) | 中 | 平衡方案，大文件友好 |
| **EXIF 标签采样** | ⚠️ 中 | O(1) exiftool 调用 | 高 | 仅检测 EXIF 变更 |
| **inotify/fsevents** | ✅ 高 | O(event) | 高 | 实时检测，但仅限本地 FS |

### 20.5 mtime 检测的局限分析

即使在 `ScanMedia` 中加入 mtime 比对，仍存在以下问题：

1. **mtime 可被伪造**：`touch -r reference.jpg target.jpg` 可以让 mtime 回到任意值
2. **NFS 属性缓存**：`actimeo` 导致 mtime 有秒级偏差
3. **复制保留 mtime**：`cp -p` 保留原始 mtime，内容实际已变
4. **精度问题**：某些文件系统 mtime 精度仅到秒，1 秒内多次修改无法区分
5. **编辑器行为差异**：
   - 原地写入：mtime 变、inode 不变（`echo > file`）
   - 安全保存：mtime 变、inode 变（`tmpfile → rename`，大多数编辑器的默认行为）

**结论**：mtime 可作为**快速初筛**，但不应作为唯一判定依据。建议采用 **mtime + file_size 双重检查**作为低成本方案，**内容哈希**作为高可靠方案。

### 20.6 推荐的分层检测策略

```
Layer 1: 快速检测（每次扫描必做）
  ├── 比较 DB 中保存的 file_size 与 os.Stat().Size()
  ├── 比较 DB 中保存的 mtime 与 os.Stat().ModTime()
  └── 任一不匹配 → 标记为 "可能变更"

Layer 2: 精确确认（仅对 Layer 1 标记的文件）
  ├── 计算文件内容 SHA256（或前 64KB 采样）
  ├── 与 DB 中保存的 content_hash 对比
  └── 不匹配 → 触发重处理（重新 EXIF + 缩略图 + 人脸）

Layer 3: 全量确认（按需触发，如 `forceRescan` mutation）
  └── 对所有文件重新计算 content_hash 并比对
```

### 20.7 需新增的 DB 字段

```sql
ALTER TABLE media ADD COLUMN file_size BIGINT;         -- os.Stat().Size()
ALTER TABLE media ADD COLUMN file_mod_time TIMESTAMP;   -- os.Stat().ModTime()
ALTER TABLE media ADD COLUMN content_hash VARCHAR(64);  -- SHA256 头部采样或全量
ALTER TABLE media ADD COLUMN inode BIGINT;              -- os.Stat().Sys().(*syscall.Stat_t).Ino
ALTER TABLE media ADD COLUMN rescan_needed BOOLEAN DEFAULT FALSE;  -- Layer 1 标记
```

---

## 21. 视频/照片缩略图与人脸检测共享缓存目录的清理冲突

### 21.1 缓存目录结构回顾

```
media_cache/
└── <albumId>/
    └── <mediaId>/
        ├── thumbnail_<token>.jpg     ← 照片缩略图 (Purpose=PhotoThumbnail)
        ├── highres_<token>.jpg       ← 照片高分辨率 (Purpose=PhotoHighRes)
        ├── video_thumb_<token>.jpg   ← 视频缩略图 (Purpose=VideoThumbnail)
        ├── web_video_<token>.mp4     ← 视频转码 (Purpose=VideoWeb)
        └── (原始文件不在此目录，直接读源路径)
```

**关键事实**：同一个 `<mediaId>/` 目录下，照片缩略图和视频缩略图**不可能同时存在**，因为一个 Media 记录的 `Type` 要么是 `photo` 要么是 `video`。

### 21.2 缩略图的生成者与消费者

| 缩略图类型 | 生成者 | 文件名模式 | 消费者 |
|---|---|---|---|
| `PhotoThumbnail` | `ProcessPhotoTask` | `thumbnail_<base>_<token>.jpg` | 前端列表展示、`BlurhashTask`、`FaceDetectionTask` |
| `VideoThumbnail` | `ProcessVideoTask` | `video_thumb_<base>_<token>.jpg` | 前端列表展示、`BlurhashTask` |
| `PhotoHighRes` | `ProcessPhotoTask` | `highres_<base>_<token>.jpg` | 前端全屏查看 |
| `VideoWeb` | `ProcessVideoTask` | `web_video_<base>_<token>.mp4` | 前端视频播放 |

### 21.3 人脸检测对缩略图的依赖链

**位置**：`face_detection/face_detector_impl.go:92-128`

```go
func (fd *faceDetector) DetectFaces(db, media) error {
    db.Model(media).Preload("MediaURL").First(&media)

    var thumbnailURL *models.MediaURL
    for _, url := range media.MediaURL {
        if url.Purpose == models.PhotoThumbnail {   // ← 只找 PhotoThumbnail
            thumbnailURL = &url
            break
        }
    }

    thumbnailPath, _ := thumbnailURL.CachedPath()   // ← 读缓存文件

    fd.mutex.Lock()
    faces, _ := fd.rec.RecognizeFile(thumbnailPath)  // ← 推理输入
    fd.mutex.Unlock()
}
```

**依赖链**：

```
FaceDetectionTask.AfterProcessMedia
  → GlobalFaceDetector.DetectFaces
    → 查找 PhotoThumbnail MediaURL
    → CachedPath() → media_cache/<albumId>/<mediaId>/thumbnail_xxx.jpg
    → face.RecognizeFile(thumbnailPath)
```

**关键**：人脸检测**仅使用 `PhotoThumbnail`**，不使用 `VideoThumbnail`。对视频媒体，`FaceDetectionTask` 在第 20 行检查 `mediaData.Media.Type == models.MediaTypePhoto` 后直接跳过：

```go
if didProcess && mediaData.Media.Type == models.MediaTypePhoto {
    // 仅照片做人脸检测
}
```

### 21.4 缓存清理操作分析

`CleanupMedia` 的清理粒度是**整个 `<mediaId>/` 目录**：

```go
cachePath := path.Join(utils.MediaCachePath(),
    strconv.Itoa(int(albumId)), strconv.Itoa(int(media.ID)))
os.RemoveAll(cachePath)    // ← 递归删除该 media 的所有缓存文件
```

**不存在"只删缩略图不删视频"或"只删人脸数据不删缩略图"的细粒度清理**。

### 21.5 潜在冲突场景

#### 场景 1：Sidecar 变更触发缩略图重生成，人脸数据未更新

```
时序：
  1. 用户在 Lightroom 中修改照片调色 → XMP 文件变更
  2. SidecarTask 检测到 SideCarHash 变化
  3. 重新生成 PhotoThumbnail 和 PhotoHighRes
     ├── 新的 thumbnail_<new_token>.jpg 写入缓存
     ├── 旧的 thumbnail_<old_token>.jpg 仍在磁盘上
     └── MediaURL 表更新为新的 MediaName
  4. FaceDetectionTask.AfterProcessMedia:
     ├── updatedURLs 包含新的 thumbnail URL → didProcess = true
     ├── media.Type == photo → 执行 DetectFaces
     ├── DetectFaces 读取新的 thumbnail → 在新图片上重新检测人脸
     └── 但旧的人脸 rectangle 坐标可能不再准确（裁剪/旋转后）
```

**问题**：Sidecar 变更可能导致图片被裁剪/旋转，人脸位置坐标（`ImageFace.Rectangle` 存储的是相对坐标 0.0-1.0）会偏移，但 `DetectFaces` 会重新检测，**新的 ImageFace 记录会追加而非替换旧的**。

具体来看 `classifyFace`（`face_detector_impl.go:134-189`）：

```go
// 如果匹配到已有 face_group → Association Append 新 imageFace
// 如果没有匹配 → 创建新 faceGroup + imageFace
// 旧 imageFace 记录不会被删除
```

→ **Sidecar 变更后，同一张照片可能同时存在新旧两套人脸记录**，旧的 rectangle 坐标指向原图位置，新的指向修改后图片位置。前端显示时可能出现重复人脸框或偏移。

#### 场景 2：ProcessPhotoTask 中缩略图缓存缺失自动补全

`process_photo_task.go:73-81` 和 `110-123`：

```go
// 如果 highResURL 存在但文件不存在 → 重新编码
if _, err := os.Stat(baseImagePath); os.IsNotExist(err) {
    log.Info(ctx, "High-res photo found in database but not in cache, re-encoding...")
    err = mediaData.EncodeHighRes(baseImagePath)
}

// 如果 thumbURL 存在但文件不存在 → 重新编码
if _, err := os.Stat(thumbPath); os.IsNotExist(err) {
    _, err := media_encoding.EncodeThumbnail(ctx.GetDB(), baseImagePath, thumbPath)
}
```

**这个"缓存缺失自动补全"逻辑与人脸检测不协调**：
- 补全缩略图后 `updatedURLs` **不包含**该 URL（因为没有新增 `MediaURL` 记录）
- `FaceDetectionTask` 检查 `len(updatedURLs) > 0` → false → **跳过人脸检测**
- 结果：缩略图被重新生成了，但人脸数据可能仍指向旧图片的坐标

#### 场景 3：CleanupMedia 删除缓存时人脸检测正在读同一文件

```
T1: 相册A的 ScanAlbum → scanMedia(媒体X) → FaceDetectionTask → RecognizeFile(thumbnailPath)
T2: 相册A的 AfterScanAlbum → CleanupMedia → os.RemoveAll(media_cache/A/X/)

注意：T1 和 T2 不会真正并发，因为它们在同一个 ScanAlbum 串行流程中。
但跨相册场景可能发生：

T1: 相册A ScanAlbum → 正在处理媒体X的人脸
T2: （不可能发生，因为同一相册不会并发扫描）

跨相册不会冲突，因为每个相册的缓存目录是独立的（<albumId>/ 不同）。
```

**结论**：同一相册内不会出现缩略图读写冲突（串行保证）。跨相册不会冲突（目录隔离）。

### 21.6 冲突总结表

| 冲突类型 | 是否真实存在 | 影响 |
|---|---|---|
| 缩略图/视频缩略图读写竞争 | ❌ 不存在 | 同一 Media 不可能同时是 photo 和 video |
| 人脸检测与 CleanupMedia 读写竞争 | ❌ 不存在 | 串行保证 + 目录隔离 |
| Sidecar 变更后人脸记录重复 | ✅ 存在 | 新旧 ImageFace 并存，rectangle 偏移 |
| 缓存补全后人脸不重检测 | ✅ 存在 | updatedURLs 为空导致跳过 DetectFaces |
| 缩略图重生成后旧文件残留 | ✅ 存在 | SidecarTask 生成新文件，旧文件仍留在磁盘 |

### 21.7 修复建议

1. **Sidecar 变更时先删旧人脸**：在 `SidecarTask.ProcessMedia` 中，重生成缩略图后调用 `DELETE FROM image_faces WHERE media_id = ?`，然后让 `FaceDetectionTask` 重新检测
2. **缓存补全也触发人脸重检测**：`ProcessPhotoTask` 的缓存补全逻辑应在 `updatedURLs` 中标记（如添加一个空 MediaURL 表示"需要重检测"）
3. **SidecarTask 删除旧缓存文件**：当前用 `.hold` 临时文件保护，但成功后 `os.Remove(tempFile)` 已处理。需确认异常路径下 `.hold` 文件不会残留
4. **定期清理同一 mediaId 目录下的孤儿文件**：遍历 `media_cache/<albumId>/<mediaId>/` 目录，对比 `media_urls` 表中该 `media_id` 的 `MediaName`，删除不在 DB 中的文件

---

## 22. Docker 卷挂载与原生文件系统行为差异对扫描可靠性的影响

### 22.1 Photoview 的 Docker 部署架构

**Dockerfile 关键配置**（`Dockerfile:156-176`）：

```dockerfile
ENV PHOTOVIEW_MEDIA_CACHE=/home/photoview/media-cache
USER photoview                        # UID=999, GID=999
EXPOSE 80
HEALTHCHECK CMD curl --fail http://localhost:80/api/graphql ...
```

**典型 docker-compose 挂载**：

```yaml
volumes:
  - /host/photos:/photos:ro           # 源照片目录（只读）
  - photoview-cache:/home/photoview/media-cache  # 缓存目录
  - photoview-db:/var/lib/sqlite      # SQLite 数据（如用 SQLite）
```

### 22.2 Docker 卷类型与行为差异

| 特性 | Bind Mount (`-v /host/path:/container/path`) | Named Volume (`-v name:/path`) | tmpfs (`--tmpfs /path`) |
|---|---|---|---|
| 存储位置 | 宿主机文件系统直接映射 | Docker 管理的 `/var/lib/docker/volumes/` | 内存/swap |
| 文件系统类型 | 宿主机 FS（ext4/xfs/zfs） | 宿主机 FS | tmpfs |
| inotify/fanotify | ✅ 支持 | ✅ 支持 | ✅ 支持 |
| inode 稳定性 | ✅ 稳定（与宿主机一致） | ✅ 稳定 | ⚠️ 容器重启后变化 |
| UID/GID 映射 | 直接使用宿主机数值 | 直接使用宿主机数值 | N/A |
| 符号链接 | ✅ 遵循 | ✅ 遵循 | ✅ 遵循 |
| 硬链接 | ⚠️ 仅同挂载点内 | ⚠️ 仅同卷内 | ✅ 同 tmpfs 内 |
| `os.ReadDir` 性能 | 接近原生 | 接近原生 | 快于原生 |
| `os.Stat` atime/mtime/ctime | 依赖宿主机挂载参数 | 依赖宿主机挂载参数 | 容器内维护 |
| 文件锁（flock/fcntl） | ✅ 通过内核传递 | ✅ 通过内核传递 | ✅ 容器内 |
| Docker Desktop (macOS/Windows) | ⚠️ 通过 VirtioFS/9P 透传 | ✅ Docker 管理 | ✅ 内存 |

### 22.3 Docker 部署下的特殊问题

#### 问题 1：UID/GID 不匹配

Photoview 容器以 `UID=999` 运行，宿主机文件可能属于 `UID=1000`：

```bash
# 宿主机
ls -la /photos/2024/
drwxr-xr-x 1000 1000 4096 Jun 01 10:00 .
-rw-r--r-- 1000 1000 5.2M Jun 01 10:00 IMG_001.jpg

# 容器内
ls -la /photos/2024/
drwxr-xr-x 999 999 4096 Jun 01 10:00 .        # UID 1000 在容器内可能映射到 nobody
-rw-r--r-- 999 999 5.2M Jun 01 10:00 IMG_001.jpg
```

**后果**：
- bind mount 以 `:ro` 挂载且 UID 不匹配 → `os.ReadDir` 成功但 `os.Open` 读文件失败 → EXIF/缩略图生成失败
- bind mount 以 `:rw` 挂载且 UID 不匹配 → `os.MkdirAll` 创建缓存子目录失败 → 扫描中止

**修复**：
```yaml
# 方案 A：运行时指定 UID
user: "1000:1000"

# 方案 B：修改宿主机目录权限
chmod -R o+rX /photos/

# 方案 C：Dockerfile 中不固定 UID（需自行构建）
```

#### 问题 2：Docker Desktop (macOS/Windows) 的 VirtioFS 性能与语义

| 行为 | Linux 原生 bind mount | Docker Desktop macOS (VirtioFS) |
|---|---|---|
| `os.ReadDir` 10 万文件 | ~2 秒 | ~30-60 秒（VirtioFS 开销） |
| `os.Stat` 单文件 | ~0.01ms | ~0.5-2ms |
| `os.Rename` 原子性 | ✅ 同目录原子 | ⚠️ VirtioFS 可能不保证 |
| inotify | ✅ 实时 | ⚠️ 可能延迟或丢失事件 |
| 硬链接 | ✅ | ❌ VirtioFS 不支持跨挂载硬链接 |
| `os.Open` 大文件 | 流式读 | ⚠️ 可能全量拷贝到 VM 再读 |

**对扫描的实际影响**：
- macOS 开发环境下 10 万文件首次扫描从 ~15 小时（Linux）延长到 **~20-30 小时**
- 缓存目录放在 bind mount 上时，缩略图写入性能下降 5-10 倍
- 建议将 `PHOTOVIEW_MEDIA_CACHE` 放在 **named volume**（Docker 管理的 ext4 卷）而非 bind mount 上

#### 问题 3：OverlayFS 与缓存目录

如果 `PHOTOVIEW_MEDIA_CACHE` 路径在容器的可写层（而非挂载卷），每次容器重建都会丢失全部缓存。Docker 的可写层使用 OverlayFS：

```bash
# ❌ 不推荐：缓存写入容器可写层
# 容器重建 → 缓存全部丢失 → 需要重新全量扫描
ENV PHOTOVIEW_MEDIA_CACHE=/home/photoview/media-cache
# 没有 -v 挂载该路径

# ✅ 推荐：缓存写入 named volume
docker run -v photoview-cache:/home/photoview/media-cache ...
```

#### 问题 4：多容器共享同一照片目录

```yaml
# docker-compose.yml
services:
  photoview-1:
    volumes:
      - /photos:/photos:ro
  photoview-2:        # ❌ 两个实例不能共享同一 SQLite
    volumes:
      - /photos:/photos:ro
```

**问题**：
- SQLite 在多进程写入时会锁定 → 第二个实例启动失败
- MySQL/PostgreSQL 可支持多实例，但 `ScannerQueue` 的作业去重是**进程内**的（`in_progress` 切片），多实例间不共享 → 可能重复扫描
- `CleanupMedia` 在两个实例上同时执行 → DB 唯一键冲突或级联删除竞争

#### 问题 5：Docker 容器内文件系统事件丢失

```go
// 场景：容器内用 inotify 监控文件变更（虽然 Photoview 当前不使用 inotify）
// 容器内只能看到容器 mount namespace 内的事件
// 宿主机上的文件变更通过 bind mount 传递到容器内：
//   - Linux: inotify 事件可穿透 bind mount ✅
//   - macOS Docker Desktop: inotify 不可用 ❌
//   - Windows Docker Desktop: inotify 不可用 ❌
```

Photoview 当前不使用 inotify，依赖周期性全量扫描，因此此问题暂不影响。

### 22.4 Docker 环境下的缓存一致性

| 操作 | 行为 | 风险 |
|---|---|---|
| 容器正常重启 | named volume 数据保留，SQLite 文件锁释放 | ✅ 无风险 |
| 容器被 `docker kill -9` | 可能丢失正在写入的 JPEG/MP4 | ⚠️ 缓存文件可能不完整，但 `ProcessPhotoTask` 的缓存补全逻辑可自动修复 |
| 容器被 `docker rm` 重建 | named volume 保留，容器可写层丢失 | ⚠️ 如 `PHOTOVIEW_MEDIA_CACHE` 未挂载卷则全部丢失 |
| `docker system prune --volumes` | **named volume 也被删除** | ❌ 缓存+SQLite 全部丢失 |
| 宿主机内核升级 | bind mount 重新挂载 | ⚠️ NFS/SMB 挂载可能需要重新 `mount` |
| 容器镜像升级 | 新二进制，数据库 AutoMigrate 可能执行 | ⚠️ 如新版本 schema 不兼容旧 DB，启动失败 |

### 22.5 Docker 部署最佳实践

```yaml
# 推荐的 docker-compose.yml
services:
  photoview:
    image: photoview/photoview:latest
    user: "1000:1000"                       # 与宿主机照片目录 UID 一致
    environment:
      - PHOTOVIEW_DATABASE_DRIVER=postgres  # 不用 SQLite（避免写锁问题）
      - PHOTOVIEW_POSTGRES_URL=postgres://user:pass@db:5432/photoview
      - PHOTOVIEW_MEDIA_CACHE=/cache
      - PHOTOVIEW_DISABLE_FACE_DETECTION=0
    volumes:
      # 源照片：只读 bind mount
      - /host/photos:/photos:ro
      # 缓存：named volume（高性能，持久化）
      - photoview-cache:/cache
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      - POSTGRES_DB=photoview
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - photoview-db:/var/lib/postgresql/data

volumes:
  photoview-cache:     # Docker 管理的 ext4 卷
  photoview-db:        # 持久化数据库
```

**关键要点**：
1. 缓存目录必须用 **named volume**，不能放 bind mount（macOS 性能差）也不能放容器可写层（重启丢）
2. 数据库用 **PostgreSQL/MySQL**，不用 SQLite（Docker 多容器场景下 SQLite 写锁是致命的）
3. `user: "1000:1000"` 确保 UID 与宿主机照片目录一致
4. 源照片以 **`:ro`** 挂载，防止扫描进程意外修改源文件

---

## 23. 设计思考

### 优点

1. **实现简洁**：所有变更识别基于字符串哈希 + 集合差集，无复杂状态机
2. **扫描高效**：文件级匹配不读取内容，对大型图库友好
3. **强一致性**：以「每次扫描的文件系统状态」为唯一真相源，DB 永远收敛到与磁盘一致

### 局限

1. **改名感知粗粒度**：不能识别「同文件改名」，导致重复处理与元数据丢失风险
2. **不支持增量**：每次都要全量遍历目录（虽有缓存，但不涉及变更检测层面）
3. **并发 rename 窗口**：扫描期间文件跨目录移动，可能出现「两个都扫到」或「两个都没扫到」的短暂窗口
4. **并发粒度受限**：单相册内串行处理，大相册（10 万+ 文件）会拖慢整体并发度
5. **EXIF 全局串行**：`exiftool` 单进程+互斥锁，成为多核场景下的性能瓶颈
6. **NFS/SMB 无适配**：无网络文件系统检测与容错，挂载失联可能导致整相册媒体被误删
7. **半写文件检测弱**：仅通过 `Size == 0` 跳过，无法处理预分配或部分写入的文件
8. **TOCTOU 竞态**：`ScanMedia` 的查-插窗口可能导致唯一键冲突，文件处理失败
9. **Sidecar 元数据不更新**：XMP 变更仅重生成缩略图，数据库中的 EXIF 字段（描述/时间/GPS）不刷新，批量改标签在 DB 中完全不可见
10. **不支持关键词/标签**：不解析 `dc:subject`、`lr:hierarchicalSubject`、`xmp:Rating` 等 XMP 标签字段
11. **无软删/回收站**：所有删除均为物理删除，误删无恢复路径，人工标签/收藏/分享链接全部丢失
12. **CoverID 孤儿无清理**：`album.cover_id` 指向已删除 media 时不自动置 NULL，仅在读取时兜底，DB 脏数据累积
13. **无用户主动删除 API**：只能通过从磁盘删文件 + 触发扫描来删除媒体，无直接删除 Mutation
14. **Sidecar 格式单一**：仅支持 `.xmp`，不支持 JSON sidecar、Lightroom `.lrcat`、Capture One 目录等
15. **无 GC/内存监控**：无 `runtime.ReadMemStats` 采样和告警，内存泄漏只能靠外部监控发现；人脸检测 + MagickWand 内存峰值不可控
16. **无磁盘空间监控**：缩略图/视频转码写磁盘前不预检查剩余空间，磁盘写满后 Media 已入库但 MediaURL 缺失，下次扫描不会自动修复
17. **多用户权限边界不严**：`FindAlbumsForUser` 遍历不校验文件系统层权限（EACCES 会中止整个扫描）；Root Path 重叠检测用字符串 LIKE 可被符号链接绕过
18. **人脸检测内存不隔离**：`ReloadFacesFromDatabase` 加载全库人脸 descriptor，多用户共用，无按用户的内存隔离
19. **无原生备份恢复**：无 `backup/restore` mutation，恢复依赖外部工具；扫描仅能修复路径一致性，人工元数据（user_media_data、share_tokens）ID 变化后全部失效
20. **缓存孤儿无自动清理**：DB 回滚或媒体重生成后，旧 `media_cache/<albumId>/<oldMediaId>/` 目录永久残留，仅能手动脚本清理
21. **部分处理失败状态无重试**：磁盘空间不足导致 MediaURL 缺失，下次扫描因 `path_hash` 命中不会重新走 `ProcessMedia`，需人工干预
22. **原始文件内容变更无检测**：`ScanMedia` 对已存在文件直接返回，不比较 mtime/size/inode，外部工具（exiftool/Lightroom/GIMP）改写文件后扫描无感知
23. **Sidecar 变更后人脸记录重复**：`classifyFace` 追加新 ImageFace 而不删除旧的，裁剪/旋转后新旧 rectangle 并存导致前端人脸框偏移
24. **缓存补全不触发人脸重检测**：`ProcessPhotoTask` 缓存缺失补全后 `updatedURLs` 为空，`FaceDetectionTask` 跳过检测
25. **Docker UID 不匹配**：容器固定 UID=999，与宿主机照片目录 UID 不一致时扫描失败或权限错误
26. **Docker Desktop VirtioFS 性能差**：macOS/Windows 下 bind mount 的 `ReadDir`/`Stat` 比 Linux 原生慢 10-30 倍

### 优化方向（潜在）

**变更识别层**：
- 引入「文件内容哈希 + 大小 + 修改时间」多重签名，可实现真正的改名/移动检测，保留原 ID 与人工标签
- 为相册/媒体封面引用添加 `ON DELETE SET NULL` 或清理逻辑，避免悬挂指针
- 引入 inotify/fsevents 级别的增量扫描，减少全量遍历频率

**并发与性能**：
- 将单相册串行改为文件级并发（需要 DB 连接池与事务隔离级别的配合）
- `exiftool` 已是 `-stay_open True` 模式，可进一步改为多进程池消除全局互斥锁
- 人脸检测支持批量推理与 GPU 并发

**可靠性**：
- NFS/SMB 挂载健康检测：`os.ReadDir` 失败时跳过该相册的 CleanupMedia，避免误删除
- 增加文件完整性检测：`Size > 0` 后增加 `ModTime` 稳定检查（如 30 秒内未变化）
- `ScanMedia` 的查-插竞态：改用 `INSERT ... ON CONFLICT DO NOTHING` 加返回判断
- SMB 大小写折叠处理：`path_hash` 计算前统一转为小写，避免重复扫描
- 新增媒体 Trash/回收站机制：`Media` 增加 `DeletedAt` 字段，支持软删与一键恢复
- CoverID 孤儿定期清理：在 `AfterScanAlbum` 中增加 `UPDATE albums SET cover_id = NULL WHERE cover_id NOT IN (SELECT id FROM media)`
- 磁盘空间预检查：缩略图/转码前用 `unix.Statfs` 检查剩余空间，< 1GB 时暂停并告警
- 部分处理状态标记：`ProcessMedia` 失败时在 `media` 表记录 `status='partial'`，下次扫描重试失败项

**Sidecar 与元数据**：
- XMP 变更时重新解析 EXIF（当前仅重编码图片不刷新 DB 字段）
- 扩展 sidecar 支持：JSON 格式、大小写不敏感匹配（`.XMP`）
- 解析并持久化 XMP 标签字段：`dc:subject`（关键词）、`xmp:Rating`（评分）、`lr:hierarchicalSubject`（层级标签）
- 提供批量刷新 EXIF 的 API：按相册/按时间范围重解析元数据而不重编码图片

**原始文件变更检测**：
- `ScanMedia` 增加对已存在文件的 `mtime + file_size` 快速比对，不匹配时标记为需重处理
- Media 表新增 `file_size`/`file_mod_time`/`content_hash`/`inode` 字段，支持分层检测
- 提供 `forceRescan` mutation：按媒体 ID/相册/路径范围强制重新处理（忽略 path_hash 命中）
- 内容哈希采用「头部 64KB 采样 + 全量回退」策略，平衡可靠性与性能

**缓存与清理冲突**：
- `SidecarTask.ProcessMedia` 重生成缩略图前先删除该 media 的旧 `image_faces` 记录
- `ProcessPhotoTask` 缓存补全后将补全的 MediaURL 加入 `updatedURLs`，确保 `FaceDetectionTask` 被触发
- 缓存目录内孤儿文件定期清理：扫描 `media_cache/<albumId>/<mediaId>/` 目录，删除不在 `media_urls` 表中的文件
- `.hold` 临时文件超时清理：启动时扫描 `media_cache` 中所有 `.hold` 文件，超过 1 小时的自动删除

**资源监控**：
- 扫描 goroutine 中采样 `runtime.ReadMemStats`，`HeapAlloc` 超过阈值时输出 warn 日志
- 支持通过环境变量 `GOMEMLIMIT` / `GOGC` 配置文档化并在启动时打印
- 新增 Prometheus metrics 端点：`photoview_scan_duration_seconds`、`photoview_media_count`、`photoview_cache_bytes`
- 磁盘空间定期检查（如每扫描 100 个文件检查一次），自动暂停等待人工释放

**多用户与权限**：
- `FindAlbumsForUser` 中对每个子目录捕获 `EACCES` 并跳过该分支（记录告警），不中止整个扫描
- `NewRootAlbum` 重叠检测用 `filepath.EvalSymlinks` 解析真实路径后比较，防止符号链接绕过
- 人脸检测内存按用户隔离：`getSamplesFromDatabase` JOIN `user_albums` 按 user_id 分桶（需权衡内存占用）
- `UserRemoveRootAlbum` 前返回「你是该相册唯一拥有者，删除将影响 X 个媒体」的提示信息

**备份恢复**：
- 提供 `orphanCleanup` GraphQL mutation：一次清理 CoverID 孤儿、空缓存目录、`user_media_data` / `share_tokens` 孤儿行
- 提供 `mediaRehash` mutation：批量改名时基于内容哈希匹配旧记录并更新 `path/path_hash`（保留 ID 和人工元数据）
- 引入「修复扫描」模式：跳过 `CleanupMedia`（不检测删除），仅补全缺失 media 并重建缺失的 MediaURL
- 提供 `databaseBackup` mutation：封装 `pg_dump` / `mysqldump` 并下载

**Docker 部署**：
- Dockerfile 中 UID 可通过 build arg 自定义（`ARG PUID=999`），避免与宿主机冲突
- `docker-compose.yml` 示例中缓存目录使用 named volume，数据库使用 PostgreSQL
- 启动时检测 `PHOTOVIEW_MEDIA_CACHE` 是否在容器可写层，如是则输出 warn 日志
- 文档化 Docker Desktop (macOS/Windows) 的 VirtioFS 性能限制和替代方案

**运营友好**：
- 扫描进度持久化（按文件粒度记录 offset），中断后可恢复
- 新增 API：按路径范围扫描、按修改时间增量扫描、单张图片强制重扫（刷新 EXIF）
- 慢扫描告警：单文件处理 > N 秒时输出详细日志（文件大小、耗时分布）
- 新增 `deleteMedia(id)` GraphQL Mutation：支持用户从 UI 主动删除/恢复单张图片
- 缓存孤儿清理 cron：定期执行 `find ./media_cache -type d -empty -delete`，并在 `CleanupMedia` 中同步清理空 album 目录
- 恢复后校验脚本：输出「DB 媒体数 vs 磁盘文件数差异报告」「孤儿 CoverID 数量」「缓存孤儿目录大小」供运营核对
