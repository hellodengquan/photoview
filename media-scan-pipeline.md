# Photoview 媒体扫描管线：从目录变更检测到缩略图落盘

## 全局概览

Photoview 的媒体扫描是一条从「发现文件」到「缩略图持久化」的完整管线，核心走向如下：

```
触发源                   入队调度                     逐文件处理                        缩略图落盘
┌──────────┐      ┌───────────────┐      ┌──────────────────────────┐      ┌──────────────┐
│ 定时扫描  │──┐   │               │      │ BeforeProcessMedia       │      │              │
│ GraphQL   │──┼──▶│ ScannerQueue  │──┐   │   ├─ CounterpartFiles   │      │ MediaCache   │
│ 手动重处理 │──┘   │ (并发 worker) │  │   │ ProcessMedia (串行任务链) │─────▶│ 文件系统写入  │
│           │      │               │  │   │   ├─ SidecarTask        │      │ 数据库记录    │
└──────────┘      └───────────────┘  │   │   ├─ ProcessPhotoTask   │      └──────────────┘
                                      │   │   ├─ ProcessVideoTask   │
                                      │   │   ├─ FaceDetectionTask  │
                                      │   │   ├─ BlurhashTask       │
                                      │   │   ├─ ExifTask           │
                                      │   │   ├─ VideoMetadataTask  │
                                      │   │   └─ MediaCleanupTask   │
                                      │   │ AfterProcessMedia       │
                                      │   └──────────────────────────┘
                                      │
                                      └── ScanAlbum
                                           ├─ BeforeScanAlbum
                                           ├─ findMediaForAlbum
                                           │    ├─ MediaFound (过滤)
                                           │    ├─ ScanMedia (入库)
                                           │    └─ AfterMediaFound
                                           ├─ scanMedia (逐文件)
                                           └─ AfterScanAlbum
```

---

## 一、扫描任务的触发

扫描任务有三种触发方式，最终都汇入同一个 `ScannerQueue`。

### 1.1 GraphQL 手动触发

文件：`api/graphql/resolvers/scanner.go`

用户通过 GraphQL mutation 触发扫描：

- **`scanAll`**：调用 `scanner_queue.AddAllToQueue()`，把所有用户加入队列
- **`scanUser(userID)`**：调用 `scanner_queue.AddUserToQueue(&user)`，只扫描指定用户

这两个 mutation 不会阻塞，立即返回 `ScannerResult{Finished: false}`，扫描在后台异步执行。

### 1.2 定时扫描（Periodic Scanner）

文件：`api/scanner/periodic_scanner/periodic_scanner.go`

`periodicScanner` 在服务启动时由 `InitializePeriodicScanner(db)` 初始化（见 `server.go:66`）。其核心逻辑：

1. 从 `SiteInfo.PeriodicScanInterval` 读取定时间隔（秒）
2. 创建 `time.Ticker`，在 `scanIntervalRunner` goroutine 中 select 等待三个信号：
   - `done` channel：关闭信号
   - `ticker_changed` channel：间隔配置变更信号
   - `ticker.C`：定时触发 → 调用 `scannerQueue.AddAllToQueue()`
3. 当间隔设为 0 时，ticker 为 nil，定时扫描禁用

间隔可由 GraphQL `setPeriodicScanInterval` mutation 动态修改，会调用 `ChangePeriodicScanInterval()` 替换 ticker。

### 1.3 单文件重处理

文件：`api/scanner/scanner_media.go:77`

`ProcessSingleMedia(ctx, db, media)` 直接调用 `scanMedia()` 跳过队列，同步处理单个媒体（用于修复损坏缓存等场景）。

---

## 二、目录变更检测与入队

### 2.1 从用户到 Album 的发现

文件：`api/scanner/scanner_user.go`

`AddUserToQueue(user)` 的流程：

```
AddUserToQueue(user)
  │
  ├─ scanner.FindAlbumsForUser(db, user, albumCache)
  │    │
  │    ├─ user.FillAlbums(db)  // 加载用户已有的 album
  │    │
  │    ├─ 查询用户根 Album（parent_album_id IS NULL 或不在用户 album 集合中）
  │    │
  │    └─ BFS 遍历目录树：
  │         对每个目录：
  │           ├─ os.ReadDir 读取目录内容
  │           ├─ 检查 .photoviewignore（合并 ignore 列表）
  │           ├─ 数据库事务：按 path_hash 查找或创建 Album 记录
  │           └─ 对子目录：directoryContainsPhotos() 判断是否含图片
  │              若包含 → 加入 BFS 队列继续遍历
  │
  ├─ cleanup_tasks.DeleteOldUserAlbums()  // 删除文件系统中已不存在的旧 Album
  │
  └─ 对每个发现的 Album，创建 ScannerJob 加入队列
```

**关键点**：`directoryContainsPhotos()` 使用缓存加速——`AlbumScannerCache` 用 map 记录路径是否包含图片，避免重复遍历。

### 2.2 目录变更检测策略

Photoview **没有使用 inotify/fswatch 等文件系统监听机制**，而是采用「全量扫描 + 数据库比对」的策略：

- **新文件检测**：`ScanMedia()` (`scanner_media.go:21`) 用 `path_hash` 查数据库，若不存在则创建新 `Media` 记录
- **删除文件检测**：`CleanupMedia()` (`cleanup_media.go:17`) 扫描结束后，查找数据库中存在但文件系统中已不存在的 media 并删除
- **缓存失效检测**：处理任务中会 `os.Stat` 检查缓存文件是否存在，若丢失则重新生成

### 2.3 ScannerQueue 入队

文件：`api/scanner/scanner_queue/queue.go`

```go
type ScannerQueue struct {
    mutex       sync.Mutex
    idle_chan   chan bool          // 唤醒后台 worker 的信号
    in_progress []ScannerJob       // 正在执行的 job
    up_next     []ScannerJob       // 等待执行的 job
    db          *gorm.DB
    settings    ScannerQueueSettings  // max_concurrent_tasks
    close_chan  *chan bool
    running     bool
}
```

`addJob()` 先检查 `jobOnQueue()` 防重（按 album ID 去重），然后把 job 追加到 `up_next`，并 `notify()` 向 `idle_chan` 发信号唤醒 worker。

---

## 三、任务调度与并发执行

### 3.1 后台 Worker 主循环

文件：`api/scanner/scanner_queue/queue.go:100-122`

```
startBackgroundWorker() goroutine:
  loop:
    ← idle_chan 等待唤醒
    检查 close_chan 是否需要停止
    调用 processQueue()
```

`processQueue()` 的核心逻辑（`queue.go:136-192`）：

```go
for len(queue.in_progress) < maxJobs && len(queue.up_next) > 0 {
    nextJob := queue.up_next[0]
    queue.up_next = queue.up_next[1:]
    queue.in_progress = append(queue.in_progress, nextJob)

    go func() {
        nextJob.Run(queue.db)   // 执行 scanner.ScanAlbum()

        // 从 in_progress 移除
        queue.notify()          // 唤醒主循环继续处理
    }()
}
```

- 并发度由 `SiteInfo.ConcurrentWorkers` 控制，可通过 GraphQL `setScannerConcurrentWorkers` 动态调整
- SQLite 不支持多 worker（数据库锁限制），仅允许 `workers=1`
- 每个 `ScannerJob.Run()` 就是在一个 goroutine 中调用 `scanner.ScanAlbum()`

### 3.2 通知机制

`processQueue()` 还负责向前端推送 WebSocket 通知（通过 `notification.BroadcastNotification`）：

- 扫描中：`"Scanning media"` + 进度信息（用 500ms 节流避免通知风暴）
- 扫描完成：`"Scanner complete"`

---

## 四、ScannerTask 管线架构

### 4.1 ScannerTask 接口

文件：`api/scanner/scanner_task/scanner_task.go`

`ScannerTask` 定义了 7 个钩子，按扫描阶段分为三组：

| 阶段 | 钩子 | 说明 |
|------|------|------|
| **Album 级** | `BeforeScanAlbum` | Album 扫描开始前，可注入上下文数据 |
| | `AfterScanAlbum` | Album 扫描结束后，清理/通知 |
| **文件发现** | `MediaFound` | 文件系统发现文件时，返回 skip=true 可跳过 |
| | `AfterMediaFound` | 文件入库后（无论新旧），可做元数据提取 |
| **文件处理** | `BeforeProcessMedia` | 处理前准备（如查找配对文件） |
| | `ProcessMedia` | 核心：生成缩略图/转码，返回更新后的 MediaURL 列表 |
| | `AfterProcessMedia` | 处理后（如人脸检测、blurhash） |

`ScannerTaskBase` 提供所有钩子的空实现，各具体 Task 可只覆写关心的钩子。

### 4.2 Task 注册与编排

文件：`api/scanner/scanner_tasks/scanner_tasks.go`

```go
var allTasks []scanner_task.ScannerTask = []scanner_task.ScannerTask{
    NotificationTask{},
    IgnorefileTask{},
    processing_tasks.CounterpartFilesTask{},
    processing_tasks.SidecarTask{},
    processing_tasks.ProcessPhotoTask{},
    processing_tasks.ProcessVideoTask{},
    FaceDetectionTask{},
    BlurhashTask{},
    ExifTask{},
    VideoMetadataTask{},
    cleanup_tasks.MediaCleanupTask{},
}
```

`scannerTasks` 是所有 task 的组合器，按顺序依次调用每个 task 的钩子。`ProcessMedia` 比较特殊——它收集所有 task 返回的 `[]*MediaURL` 合并为最终结果。

### 4.3 调用链路

`ScanAlbum()` (`scanner_album.go:87`) 的完整流程：

```
ScanAlbum(ctx)
  │
  ├─ Tasks.BeforeScanAlbum(ctx)
  │    └─ IgnorefileTask: 编译 .photoviewignore 规则注入 context
  │
  ├─ findMediaForAlbum(ctx)
  │    │
  │    ├─ os.ReadDir(album.Path)
  │    │
  │    └─ 对每个文件：
  │         ├─ cache.IsPathMedia(mediaPath)  // 判断是否支持的媒体类型
  │         ├─ Tasks.MediaFound(ctx, fileInfo, mediaPath)
  │         │    ├─ IgnorefileTask: 命中 ignore → skip
  │         │    └─ CounterpartFilesTask: 若是 RAW 对应的 JPEG → skip
  │         │
  │         ├─ ScanMedia(db, mediaPath, albumID, cache)
  │         │    └─ 按 path_hash 查数据库，不存在则创建 Media 记录
  │         │
  │         └─ Tasks.AfterMediaFound(ctx, media, isNewMedia)
  │              ├─ NotificationTask: 新文件 → 发通知
  │              ├─ SidecarTask: 非 Web 兼容图片 → 扫描 .xmp sidecar
  │              ├─ ExifTask: 新文件 → 解析 EXIF 入库
  │              └─ VideoMetadataTask: 新视频 → 读 ffprobe 元数据入库
  │
  ├─ 对每个发现的 media，调用 scanMedia(ctx, media, mediaData, i, total):
  │    │
  │    ├─ Tasks.BeforeProcessMedia(ctx, mediaData)
  │    │    └─ CounterpartFilesTask: 为 RAW 文件查找 JPEG 配对
  │    │
  │    ├─ media.CachePath()  // 确保缓存目录存在
  │    │
  │    ├─ [数据库事务] Tasks.ProcessMedia(ctx, mediaData, mediaCachePath)
  │    │    ├─ SidecarTask: 检查 sidecar 变更 → 重新生成 highres+thumbnail
  │    │    ├─ ProcessPhotoTask: 照片 → highres JPEG + thumbnail
  │    │    ├─ ProcessVideoTask: 视频 → web MP4 + video thumbnail
  │    │    └─ 其余 task 的 ProcessMedia 返回空
  │    │
  │    └─ [数据库事务] Tasks.AfterProcessMedia(ctx, mediaData, updatedURLs, i, total)
  │         ├─ NotificationTask: 进度通知
  │         ├─ FaceDetectionTask: 照片 → 人脸检测
  │         └─ BlurhashTask: 缩略图 → blurhash 编码入库
  │
  └─ Tasks.AfterScanAlbum(ctx, changedMedia, albumMedia)
       ├─ NotificationTask: 完成通知
       └─ MediaCleanupTask: 删除文件系统中已不存在的 media
```

---

## 五、照片处理与缩略图生成

### 5.1 ProcessPhotoTask

文件：`api/scanner/scanner_tasks/processing_tasks/process_photo_task.go`

对每张照片，按需生成三种 `MediaURL`：

| Purpose | 说明 | 生成条件 |
|---------|------|----------|
| `MediaOriginal` | 原始文件引用 | 数据库中不存在时创建（不拷贝文件） |
| `PhotoHighRes` | 高质量 JPEG | 非 Web 兼容格式（RAW 等）时，转为 JPEG |
| `PhotoThumbnail` | 缩略图 JPEG (≤1024px) | 数据库中不存在时生成 |

**处理逻辑**：

1. **HighRes 生成**（仅非 Web 兼容格式）：
   - 调用 `mediaData.EncodeHighRes(baseImagePath)` → `Magick.EncodeJpeg(imgPath, outputPath, 70)`
   - 用 ImageMagick MagickWand 将 RAW 转为 JPEG，质量 70

2. **Original 记录**：
   - 获取原始图片尺寸 `GetPhotoDimensions()` → `Magick.IdentifyDimension()`
   - 创建 `MediaURL{Purpose: MediaOriginal}` 记录，`CachedPath` 直接指向原始文件

3. **Thumbnail 生成**：
   - 调用 `generateSaveThumbnailJPEG()` → `EncodeThumbnail()` → `Magick.GenerateThumbnail()`
   - 缩略图尺寸：`ThumbnailScale()` 保证长边 ≤ 1024px，保持宽高比
   - 使用 `wand.ThumbnailImage()` 缩放 + JPEG 质量 70 输出

**缓存修复**：如果数据库有记录但文件系统上文件丢失，会重新编码并更新数据库。

### 5.2 缩略图尺寸计算

文件：`api/scanner/media_encoding/encode_photo.go:25-51`

```go
func (d *Dimension) ThumbnailScale() Dimension {
    aspect := float64(d.Width) / float64(d.Height)
    if aspect > 1 {       // 横向
        width = 1024; height = int(1024 / aspect)
    } else {              // 纵向
        width = int(1024 * aspect); height = 1024
    }
    // 若原图更小，保持原尺寸
    if width > d.Width { width = d.Width; height = d.Height }
}
```

---

## 六、视频处理与 FFmpeg 转码

### 6.1 ProcessVideoTask

文件：`api/scanner/scanner_tasks/processing_tasks/process_video_task.go`

对每个视频，按需生成三种 `MediaURL`：

| Purpose | 说明 | 生成条件 |
|---------|------|----------|
| `MediaOriginal` | 原始视频引用 | Web 兼容格式（如 mp4/h264）时直接引用 |
| `VideoWeb` | 转码后的 MP4 | 非 Web 兼容格式时，FFmpeg 转码 |
| `VideoThumbnail` | 视频缩略图 JPEG | 数据库中不存在时生成 |

**处理逻辑**：

1. **Original 记录**（仅 Web 兼容格式）：
   - 用 `ffprobe` 读取视频流元数据（宽高）
   - 创建 `MediaURL{Purpose: MediaOriginal}`，直接引用原始文件

2. **Web 转码**（仅非 Web 兼容格式）：
   - 调用 `Ffmpeg.EncodeMp4(inputPath, outputPath)`
   - FFmpeg 参数：
     - 视频编码：`h264`（支持 qsv/vaapi/nvenc 硬件加速）
     - 音频编码：`aac`
     - 缩放：`scale='min(1080,iw)':'min(1080,ih)':force_original_aspect_ratio=decrease:force_divisible_by=2`
     - `movflags +faststart+use_metadata_tags`：Web 播放优化
   - 转码完成后用 `ffprobe` 读取新文件元数据，写入 `MediaURL`

3. **视频缩略图**：
   - 调用 `Ffmpeg.EncodeVideoThumbnail(inputPath, outputPath, probeData)`
   - 在视频 25% 时长处截取一帧
   - FFmpeg 参数：`-ss <25%时长> -i <输入> -vframes 1 -an -vf scale='min(1024,iw)':'min(1024,ih)':...`
   - 截帧后读取图片尺寸，写入 `MediaURL`

### 6.2 FFmpeg CLI 封装

文件：`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go`

- 初始化时通过 `exec.LookPath("ffmpeg")` 查找 ffmpeg
- 支持环境变量 `PHOTOVIEW_VIDEO_HARDWARE_ACCELERATION` 配置硬件加速
- 禁用视频编码：`PHOTOVIEW_DISABLE_VIDEO_ENCODING=true`

### 6.3 ImageMagick MagickWand 封装

文件：`api/scanner/media_encoding/executable_worker/magickwand.go`

使用 `gopkg.in/gographics/imagick.v3` Go 绑定，核心方法：

- `EncodeJpeg(inputPath, outputPath, quality)`：转换格式为 JPEG
- `GenerateThumbnail(inputPath, outputPath, width, height)`：缩放并输出 JPEG
- `IdentifyDimension(inputPath)`：读取图片宽高

所有操作都会先 `AutoOrientImage()` 并重置 EXIF 方向，确保输出图片方向正确。

---

## 七、缩略图持久化

### 7.1 缓存目录结构

文件：`api/utils/media_cache.go`

```
<MediaCachePath>/           # 默认 ./media_cache，可通过 PHOTOVIEW_MEDIA_CACHE 环境变量配置
  ├── <albumID>/
  │   ├── <mediaID>/
  │   │   ├── thumbnail_<filename>_<token>.jpg    # 照片缩略图
  │   │   ├── highres_<filename>_<token>.jpg       # 高清 JPEG
  │   │   ├── video_thumb_<filename>_<token>.jpg   # 视频缩略图
  │   │   └── web_video_<filename>_<token>.mp4     # Web 视频
  │   └── <mediaID>/...
  └── <albumID>/...
```

`CachePathForMedia(albumID, mediaID)` 在 `scanMedia()` 中被调用，创建三级目录结构。

### 7.2 数据库与文件系统的双写

缩略图生成的完整持久化流程（以照片缩略图为例）：

```
generateSaveThumbnailJPEG(tx, media, thumbnailName, photoCachePath, baseImagePath, nil)
  │
  ├─ EncodeThumbnail(tx, baseImagePath, thumbOutputPath)
  │    ├─ Magick.IdentifyDimension(inputPath)     // 读取源图尺寸
  │    ├─ ThumbnailScale()                         // 计算缩略图尺寸
  │    └─ Magick.GenerateThumbnail(input, output, w, h)  // 写入文件系统
  │
  ├─ os.Stat(thumbOutputPath)                      // 读取文件大小
  │
  └─ tx.Create(&MediaURL{...})                     // 写入数据库
       MediaName:   thumbnailName                  // 用于 URL 路由
       Width/Height: 缩略图实际尺寸
       Purpose:     PhotoThumbnail
       ContentType: "image/jpeg"
       FileSize:    文件字节数
```

**关键设计**：

- 文件先写磁盘，再写数据库，且在同一个**数据库事务**内完成（`media_scan.go:22`）
- `MediaURL.CachedPath()` 根据 Purpose 拼出磁盘路径：`MediaCachePath/albumID/mediaID/MediaName`
- `MediaURL.URL()` 根据 Purpose 拼出 HTTP 路径：`/photo/MediaName` 或 `/video/MediaName`

### 7.3 缓存一致性

三种一致性保障机制：

1. **新文件**：`MediaURL` 记录不存在 → 生成文件 + 创建记录
2. **文件丢失**：数据库有记录但 `os.Stat` 发现文件不在 → 重新生成 + 更新记录
3. **文件删除**：`CleanupMedia()` 扫描结束后对比数据库与文件系统，删除不再存在的 media 及其缓存目录

---

## 八、Task 执行顺序与协作关系

整个管线中各 Task 在不同阶段的职责分工：

| Task | BeforeScanAlbum | MediaFound | AfterMediaFound | BeforeProcessMedia | ProcessMedia | AfterProcessMedia | AfterScanAlbum |
|------|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| **NotificationTask** | | | ✅ 新文件通知 | | | ✅ 进度通知 | ✅ 完成通知 |
| **IgnorefileTask** | ✅ 编译 ignore | ✅ 过滤 | | | | | |
| **CounterpartFilesTask** | | ✅ 跳过 RAW 的 JPEG 伴生 | | ✅ 查找 JPEG 配对 | | | |
| **SidecarTask** | | | ✅ 扫描 .xmp | | ✅ sidecar 变更重生成 | | |
| **ProcessPhotoTask** | | | | | ✅ 照片缩略图 | | |
| **ProcessVideoTask** | | | | | ✅ 视频转码+缩略图 | | |
| **FaceDetectionTask** | | | | | | ✅ 人脸检测 | |
| **BlurhashTask** | | | | | | ✅ blurhash | |
| **ExifTask** | | | ✅ EXIF 解析 | | | | |
| **VideoMetadataTask** | | | ✅ 视频元数据 | | | | |
| **MediaCleanupTask** | | | | | | | ✅ 清理旧 media |

---

## 九、关键配置与环境变量

| 配置项 | 环境变量 | 说明 |
|--------|---------|------|
| 并发 worker 数 | `SiteInfo.ConcurrentWorkers` | SQLite 下限 1，其他数据库可 >1 |
| 定时扫描间隔 | `SiteInfo.PeriodicScanInterval` | 秒，0 为禁用 |
| 缓存目录 | `PHOTOVIEW_MEDIA_CACHE` | 默认 `./media_cache` |
| 硬件加速 | `PHOTOVIEW_VIDEO_HARDWARE_ACCELERATION` | qsv/vaapi/nvenc |
| 禁用视频转码 | `PHOTOVIEW_DISABLE_VIDEO_ENCODING` | bool |
| 禁用 RAW 处理 | `PHOTOVIEW_DISABLE_RAW_PROCESSING` | bool，跳过非 Web 兼容图片 |

---

## 十、总结

Photoview 的媒体扫描管线是一个**拉取式、全量比对、任务链串行、队列并发**的设计：

1. **触发层**：GraphQL 手动 + 定时器周期 + 单文件重处理，三种入口统一汇入 ScannerQueue
2. **调度层**：ScannerQueue 用 `in_progress` + `up_next` 双数组管理 job，通过 `idle_chan` 信号驱动后台 goroutine，按 `max_concurrent_tasks` 并发执行
3. **发现层**：BFS 遍历用户根目录，`.photoviewignore` 过滤，`directoryContainsPhotos()` 剪枝，新文件 `ScanMedia()` 入库
4. **处理层**：`ScannerTask` 接口的 7 个钩子将处理逻辑解耦为 11 个独立 Task，按注册顺序串行执行
5. **转码层**：照片走 ImageMagick MagickWand（RAW→JPEG + Thumbnail），视频走 FFmpeg CLI（转 MP4 + 截帧缩略图）
6. **持久层**：文件写入 `MediaCachePath/albumID/mediaID/` 目录，元数据写入 `MediaURL` 数据库记录，两者在同一事务内完成，并通过 Stat 检测 + Cleanup 保障一致性
