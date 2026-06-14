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

## 14. 设计思考

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

### 优化方向（潜在）

**变更识别层**：
- 引入「文件内容哈希 + 大小 + 修改时间」多重签名，可实现真正的改名/移动检测，保留原 ID 与人工标签
- 为相册/媒体封面引用添加 `ON DELETE SET NULL` 或清理逻辑，避免悬挂指针
- 引入 inotify/fsevents 级别的增量扫描，减少全量遍历频率

**并发与性能**：
- 将单相册串行改为文件级并发（需要 DB 连接池与事务隔离级别的配合）
- 使用 `exiftool` `-stay_open` 模式或多进程池，消除 EXIF 全局串行瓶颈
- 人脸检测支持批量推理与 GPU 并发

**可靠性**：
- NFS/SMB 挂载健康检测：`os.ReadDir` 失败时跳过该相册的 CleanupMedia，避免误删除
- 增加文件完整性检测：`Size > 0` 后增加 `ModTime` 稳定检查（如 30 秒内未变化）
- `ScanMedia` 的查-插竞态：改用 `INSERT ... ON CONFLICT DO NOTHING` 加返回判断
- SMB 大小写折叠处理：`path_hash` 计算前统一转为小写，避免重复扫描

**运营友好**：
- 扫描进度持久化（按文件粒度记录 offset），中断后可恢复
- 新增 API：按路径范围扫描、按修改时间增量扫描
- 慢扫描告警：单文件处理 > N 秒时输出详细日志（文件大小、耗时分布）
