# Photoview 视频转码管线深度分析

> **重要说明**：经过对整个代码库的全面搜索（关键词：`hls`、`m3u8`、`segment`、`fragment`），Photoview **并未实现 HLS 视频片段化/分片传输**。它的视频处理管线是将非 Web 兼容的视频文件转码为 MP4(H.264+AAC) 格式，存储在本地缓存后供浏览器直接播放。本文档基于实际代码对完整管线进行梳理。

---

## 一、整体架构概览

```
                          ┌─────────────────────┐
                          │   触发入口          │
                          │  (GraphQL / 定时 /  │
                          │   HTTP 请求补转码)  │
                          └─────────┬───────────┘
                                    │
                                    ▼
                          ┌─────────────────────┐
                          │  scanner_queue      │
                          │  (任务队列 + 并发   │
                          │   worker 调度)      │
                          └─────────┬───────────┘
                                    │
                                    ▼
                          ┌─────────────────────┐
                          │   ScanAlbum         │
                          │  (遍历目录 + 识别   │
                          │   媒体类型)         │
                          └─────────┬───────────┘
                                    │
                                    ▼
                          ┌─────────────────────┐
                          │  scanner_tasks      │
                          │  (任务链串行执行)   │
                          └─────────┬───────────┘
                                    │
                     ┌──────────────┼──────────────┐
                     ▼              ▼              ▼
            ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
            │VideoMetadata │ │ProcessVideo  │ │  其他任务... │
            │    Task      │ │    Task      │ │              │
            └──────┬───────┘ └──────┬───────┘ └──────────────┘
                   │                │
                   ▼                ▼
            ┌──────────────┐ ┌──────────────────────────────┐
            │  ffprobe     │ │  FfmpegCli (H.264 + AAC)     │
            │  元数据采集  │ │  - EncodeMp4()               │
            └──────────────┘ │  - EncodeVideoThumbnail()    │
                             └──────────────┬───────────────┘
                                            │
                                            ▼
                             ┌──────────────────────────────┐
                             │  缓存写盘 + DB 记录          │
                             │  media_cache/                │
                             │    └── albumID/              │
                             │        └── mediaID/          │
                             │            ├── web_video_*.mp4│
                             │            └── video_thumb_*.jpg
                             └──────────────┬───────────────┘
                                            │
                                            ▼
                             ┌──────────────────────────────┐
                             │  HTTP 路由 /video/{name}     │
                             │  - 缓存命中检查              │
                             │  - 缺失则触发补转码          │
                             │  - 带 Cache-Control 返回文件 │
                             └──────────────────────────────┘
```

---

## 二、扫描入口：视频处理的触发方式

视频处理有 **三种触发路径**：

### 2.1 GraphQL 手动触发

**文件**：`api/graphql/resolvers/scanner.go`

```go
// 全量扫描
func (r *mutationResolver) ScanAll(ctx context.Context) (*models.ScannerResult, error) {
    err := scanner_queue.AddAllToQueue()  // line 23
    ...
}

// 单用户扫描
func (r *mutationResolver) ScanUser(ctx context.Context, userID int) (*models.ScannerResult, error) {
    ...
    scanner_queue.AddUserToQueue(&user)   // line 44
    ...
}
```

### 2.2 定时扫描触发

**文件**：`api/scanner/periodic_scanner/periodic_scanner.go`

- `InitializePeriodicScanner()` 启动后台 goroutine (`scanIntervalRunner`)
- 根据 `SiteInfo.PeriodicScanInterval` 配置的时间间隔周期性触发
- 到期后调用 `scannerQueue.AddAllToQueue()` 将所有用户加入扫描队列

### 2.3 HTTP 请求时按需补转码（缓存未命中时）

**文件**：`api/routes/videos.go` + `api/routes/photos.go`

```go
func handleVideoRequest(...) {
    // line 81-118: 检查缓存文件是否存在
    if _, err := os.Stat(cachedPath); err != nil {
        if os.IsNotExist(err) {
            // 缓存缺失，即时触发单媒体处理
            if err := processSingleMediaFn(r.Context(), db, media); err != nil {
                ...
            }
        }
    }
    // line 122-124: 命中后设置强缓存头并返回文件
    w.Header().Set("Cache-Control", "private, max-age=31536000, immutable")
    http.ServeFile(w, r, cachedPath)
}
```

---

## 三、任务调度：scanner_queue 队列机制

**文件**：`api/scanner/scanner_queue/queue.go`

### 3.1 队列结构

```go
type ScannerQueue struct {
    mutex       sync.Mutex
    idle_chan   chan bool
    in_progress []ScannerJob   // 正在执行的任务
    up_next     []ScannerJob   // 等待执行的任务
    db          *gorm.DB
    settings    ScannerQueueSettings  // max_concurrent_tasks
    ...
}
```

### 3.2 执行流程

1. `AddAllToQueue()` / `AddUserToQueue()` → 查找用户所有相册，封装成 `ScannerJob` 加入 `up_next`
2. 后台 goroutine `startBackgroundWorker()` 监听 `idle_chan`，收到信号后执行 `processQueue()`
3. `processQueue()` 根据 `max_concurrent_tasks` 配置，从 `up_next` 取任务启动 goroutine 执行
4. 每个 job 最终调用 `scanner.ScanAlbum(job.ctx)` (`queue.go:37`)

### 3.3 并发控制

并发 worker 数量来自数据库 `SiteInfo.ConcurrentWorkers`，可通过 GraphQL `SetScannerConcurrentWorkers` 动态调整。

---

## 四、媒体发现与类型识别

**文件**：`api/scanner/scanner_album.go`

### 4.1 相册扫描入口 `ScanAlbum()`

```go
func ScanAlbum(ctx scanner_task.TaskContext) error {
    // line 88: BeforeScanAlbum - 所有任务的前置钩子
    newCtx, err := scanner_tasks.Tasks.BeforeScanAlbum(ctx)

    // line 95: 遍历目录发现媒体文件
    albumMedia, err := findMediaForAlbum(ctx)

    // line 101-107: 逐个媒体处理
    for i, media := range albumMedia {
        mediaData := media_encoding.NewEncodeMediaData(media)
        if err := scanMedia(ctx, media, &mediaData, i, len(albumMedia)); err != nil {
            ...
        }
    }
    ...
}
```

### 4.2 媒体文件发现 `findMediaForAlbum()`

```go
func findMediaForAlbum(ctx scanner_task.TaskContext) ([]*models.Media, error) {
    dirContent, _ := os.ReadDir(ctx.GetAlbum().Path)
    for _, item := range dirContent {
        // line 135: 检查是否为支持的媒体类型
        if ctx.GetCache().IsPathMedia(mediaPath) {
            // line 149: 写入数据库（去重：path_hash）
            media, isNewMedia, err := ScanMedia(ctx.GetDB(), mediaPath, ...)
            // line 154: AfterMediaFound 钩子（含 VideoMetadataTask）
            scanner_tasks.Tasks.AfterMediaFound(ctx, media, isNewMedia)
        }
    }
}
```

### 4.3 媒体类型判断

**文件**：`api/scanner/media_type/media_type.go`

- 通过 `exif.MIMEType()` 读取文件真实 MIME
- Web 兼容视频类型（`IsWebCompatible()`）：`video/mp4`、`video/mpeg`、`video/webm`、`video/ogg`
- 非 Web 兼容类型（如 `.avi`、`.mkv`、`.mov`、`.wmv`）需要转码

---

## 五、任务链执行

**文件**：`api/scanner/scanner_tasks/scanner_tasks.go`

### 5.1 任务执行顺序

所有任务按 `allTasks` 数组顺序串行执行：

```go
var allTasks []scanner_task.ScannerTask = []scanner_task.ScannerTask{
    NotificationTask{},
    IgnorefileTask{},
    processing_tasks.CounterpartFilesTask{},
    processing_tasks.SidecarTask{},
    processing_tasks.ProcessPhotoTask{},
    processing_tasks.ProcessVideoTask{},    // ← 视频转码核心任务
    FaceDetectionTask{},
    BlurhashTask{},
    ExifTask{},
    VideoMetadataTask{},                    // ← 视频元数据采集
    cleanup_tasks.MediaCleanupTask{},
}
```

### 5.2 单媒体处理流程 `scanMedia()`

**文件**：`api/scanner/media_scan.go`

```go
func scanMedia(ctx scanner_task.TaskContext, media *models.Media, mediaData *media_encoding.EncodeMediaData, ...) error {
    // line 12: 所有任务 BeforeProcessMedia 钩子
    newCtx, _ := scanner_tasks.Tasks.BeforeProcessMedia(ctx, mediaData)

    // line 17: 获取/创建缓存目录
    mediaCachePath, _ := media.CachePath()

    // line 22-33: 数据库事务内执行 ProcessMedia 和 AfterProcessMedia
    newCtx.DatabaseTransaction(func(ctx scanner_task.TaskContext) error {
        updatedURLs, _ := scanner_tasks.Tasks.ProcessMedia(newCtx, mediaData, mediaCachePath)
        scanner_tasks.Tasks.AfterProcessMedia(newCtx, mediaData, updatedURLs, ...)
        return nil
    })
}
```

---

## 六、FFmpeg 调用：转码命令构建与执行

### 6.1 FfmpegCli 初始化

**文件**：`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:27-68`

```go
func newFfmpegCli() *FfmpegCli {
    // 检查环境变量 PHOTOVIEW_DISABLE_VIDEO_ENCODING
    if utils.EnvDisableVideoEncoding.GetBool() {
        return &FfmpegCli{err: ErrDisabledFunction}
    }

    // 通过 exec.LookPath 查找 ffmpeg 可执行文件
    path, _ := exec.LookPath("ffmpeg")

    // 根据 PHOTOVIEW_VIDEO_HARDWARE_ACCELERATION 选择编码器
    hwAcc := utils.EnvVideoHardwareAcceleration.GetValue()
    // 支持 qsv → h264_qsv, vaapi → h264_vaapi, nvenc → h264_nvenc
    // 默认使用 h264 软编码
    // 特殊：值以 "_" 开头时，直接使用后缀作为编码器名（调试用）
}
```

**文件**：`api/scanner/media_encoding/executable_worker/executable_worker.go:17-29`

`Initialize()` 在服务启动时调用，初始化全局单例 `Ffmpeg` 和 `Magick`，并设置 ffprobe 路径。

### 6.2 视频转码 MP4 `EncodeMp4()`

**文件**：`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:74-96`

```go
func (cli *FfmpegCli) EncodeMp4(inputPath string, outputPath string) error {
    args := []string{
        "-i", inputPath,
        "-vcodec", cli.videoCodec,               // h264 / h264_qsv / h264_vaapi / h264_nvenc
        "-acodec", "aac",                         // 音频统一转 AAC
        "-vf", "scale='min(1080,iw)':'min(1080,ih)':force_original_aspect_ratio=decrease:force_divisible_by=2",
        // ↑ 视频缩放：最大边不超过 1080px，保持宽高比，尺寸为偶数（H.264 要求）
        "-movflags", "+faststart+use_metadata_tags",
        // ↑ faststart: 将 moov atom 移到文件头，便于边下边播
        outputPath,
    }
    cmd := exec.Command(cli.path, args...)
    return cmd.Run()
}
```

**关键参数解析**：

| 参数 | 作用 |
|------|------|
| `-vcodec h264` | 视频编码 H.264（兼容所有现代浏览器） |
| `-acodec aac` | 音频编码 AAC（Web 标准音频编码） |
| `scale=...` | 分辨率限制在 1080p 以内，保持比例 |
| `+faststart` | 优化 Web 播放：moov atom 前置，支持渐进式下载 |

### 6.3 视频缩略图提取 `EncodeVideoThumbnail()`

**文件**：`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:98-122`

```go
func (cli *FfmpegCli) EncodeVideoThumbnail(inputPath string, outputPath string, probeData *ffprobe.ProbeData) error {
    thumbnailOffsetSeconds := fmt.Sprintf("%.f", probeData.Format.DurationSeconds*0.25)
    // ↑ 在视频 25% 时间点截取缩略图（避免全黑的开头帧）

    args := []string{
        "-ss", thumbnailOffsetSeconds,  // 先 seek 到指定时间点（性能更好）
        "-i", inputPath,
        "-vframes", "1",                // 只输出 1 帧
        "-an",                          // 忽略音频流
        "-vf", "scale='min(1024,iw)':'min(1024,ih)':force_original_aspect_ratio=decrease:force_divisible_by=2",
        outputPath,
    }
    ...
}
```

---

## 七、视频处理核心任务 `ProcessVideoTask`

**文件**：`api/scanner/scanner_tasks/processing_tasks/process_video_task.go`

这是整个视频管线的中枢，按顺序完成 **原始视频登记 → Web 视频转码 → 缩略图生成** 三个阶段。

### 7.1 阶段一：原始视频登记（Web 兼容时无需转码）

```go
// line 56-85
if videoOriginalURL == nil && videoType.IsWebCompatible() {
    // 读取原始视频元数据
    webMetadata, _ := ReadVideoStreamMetadata(origVideoPath)
    fileStats, _ := os.Stat(origVideoPath)

    mediaURL := models.MediaURL{
        MediaID:     video.ID,
        MediaName:   generateUniqueMediaName(video.Path), // 文件名_随机token.ext
        Width:       webMetadata.Width,
        Height:      webMetadata.Height,
        Purpose:     models.MediaOriginal,
        ContentType: videoType.String(),
        FileSize:    fileStats.Size(),
    }
    ctx.GetDB().Create(&mediaURL)
}
```

### 7.2 阶段二：非 Web 兼容视频转码

```go
// line 87-125
if videoWebURL == nil && !videoType.IsWebCompatible() {
    // 生成唯一缓存文件名：web_video_原文件名_随机token.mp4
    webVideoName := fmt.Sprintf("web_video_%s_%s", path.Base(video.Path), utils.GenerateToken())
    webVideoName = strings.ReplaceAll(webVideoName, ".", "_")
    webVideoName = strings.ReplaceAll(webVideoName, " ", "_")
    webVideoName = webVideoName + ".mp4"

    webVideoPath := path.Join(mediaCachePath, webVideoName)

    // 调用 FFmpeg 转码
    executable_worker.Ffmpeg.EncodeMp4(video.Path, webVideoPath)

    // 读取转码后视频的元数据，写入 MediaURL 表
    webMetadata, _ := ReadVideoStreamMetadata(webVideoPath)
    fileStats, _ := os.Stat(webVideoPath)

    mediaURL := models.MediaURL{
        MediaID:     video.ID,
        MediaName:   webVideoName,
        Width:       webMetadata.Width,
        Height:      webMetadata.Height,
        Purpose:     models.VideoWeb,
        ContentType: "video/mp4",
        FileSize:    fileStats.Size(),
    }
    ctx.GetDB().Create(&mediaURL)
}
```

### 7.3 阶段三：视频缩略图生成（含缓存校验）

```go
// line 132-201
if videoThumbnailURL == nil {
    // 缩略图不存在 → 生成
    videoThumbName := fmt.Sprintf("video_thumb_%s_%s", ...) + ".jpg"
    thumbImagePath := path.Join(mediaCachePath, videoThumbName)

    executable_worker.Ffmpeg.EncodeVideoThumbnail(video.Path, thumbImagePath, probeData)

    thumbDimensions, _ := media_encoding.GetPhotoDimensions(thumbImagePath)
    fileStats, _ := os.Stat(thumbImagePath)

    thumbMediaURL := models.MediaURL{
        MediaID:     video.ID,
        MediaName:   videoThumbName,
        Width:       thumbDimensions.Width,
        Height:      thumbDimensions.Height,
        Purpose:     models.VideoThumbnail,
        ContentType: "image/jpeg",
        FileSize:    fileStats.Size(),
    }
    ctx.GetDB().Create(&thumbMediaURL)
} else {
    // 缩略图 DB 记录存在 → 校验磁盘文件
    thumbImagePath := path.Join(mediaCachePath, videoThumbnailURL.MediaName)

    if _, err := os.Stat(thumbImagePath); os.IsNotExist(err) {
        // 数据库有记录但磁盘文件丢失 → 重新生成
        executable_worker.Ffmpeg.EncodeVideoThumbnail(video.Path, thumbImagePath, probeData)
        // 更新 DB 中的宽高和文件大小
        ctx.GetDB().Save(videoThumbnailURL)
    }
}
```

---

## 八、视频元数据采集 `VideoMetadataTask`

**文件**：`api/scanner/scanner_tasks/video_metadata_task.go`

在 `AfterMediaFound` 钩子中执行（新视频入库后立即触发），使用 ffprobe 采集：

```go
func ScanVideoMetadata(tx *gorm.DB, video *models.Media) error {
    data, _ := processing_tasks.ReadVideoMetadata(video.Path)
    stream := data.FirstVideoStream()

    videoMetadata := models.VideoMetadata{
        Width:        stream.Width,
        Height:       stream.Height,
        Duration:     data.Format.DurationSeconds,  // 秒
        Codec:        &stream.CodecLongName,        // 如 "H.264 / AVC / MPEG-4 AVC"
        Framerate:    framerate,                     // 解析 AvgFrameRate "30000/1001" → 29.97
        Bitrate:      &stream.BitRate,
        ColorProfile: &stream.Profile,               // 如 "High"
        Audio:        &audioText,                    // "Stereo audio" / "No audio"
    }
    video.VideoMetadata = &videoMetadata
    tx.Save(video)
}
```

---

## 九、缓存命中与重用机制

### 9.1 缓存目录结构

**文件**：`api/utils/media_cache.go`

```
MediaCachePath() (默认 ./media_cache，可通过 PHOTOVIEW_MEDIA_CACHE 环境变量配置)
└── {albumID}/
    └── {mediaID}/
        ├── web_video_{原文件名}_{token}.mp4      ← Purpose: video-web
        ├── video_thumb_{原文件名}_{token}.jpg    ← Purpose: video-thumbnail
        └── (对于图片还有 thumbnail.jpg, high-res.jpg 等)
```

目录按需创建（`CachePathForMedia()` 中有三级 `os.Mkdir` 检查）。

### 9.2 两级缓存检查

缓存检查发生在 **处理前（避免重复工作）** 和 **请求时（处理缓存丢失）** 两个时机：

#### 第一级：数据库检查（MediaURL 记录是否存在）

**文件**：`api/scanner/scanner_tasks/processing_tasks/processing_helpers.go:16-32`

```go
func makePhotoURLChecker(tx *gorm.DB, mediaID int) func(purpose models.MediaPurpose) (*models.MediaURL, error) {
    return func(purpose models.MediaPurpose) (*models.MediaURL, error) {
        var mediaURL []*models.MediaURL
        result := tx.Where("purpose = ?", purpose).Where("media_id = ?", mediaID).Find(&mediaURL)
        if result.RowsAffected > 0 {
            return mediaURL[0], nil
        }
        return nil, nil
    }
}
```

在 `ProcessVideoTask` 开头对三种 purpose 各查一次：

```go
mediaURLFromDB := makePhotoURLChecker(ctx.GetDB(), video.ID)
videoOriginalURL, _  := mediaURLFromDB(models.MediaOriginal)
videoWebURL, _       := mediaURLFromDB(models.VideoWeb)
videoThumbnailURL, _ := mediaURLFromDB(models.VideoThumbnail)
```

**存在就跳过对应阶段**（DB 有记录 = 已经处理过）。

#### 第二级：磁盘文件检查

仅对缩略图做了 DB+磁盘双重校验（`process_video_task.go:170-200`）：

```go
} else {
    // DB 有记录，再看磁盘文件是否还在
    thumbImagePath := path.Join(mediaCachePath, videoThumbnailURL.MediaName)
    if _, err := os.Stat(thumbImagePath); os.IsNotExist(err) {
        // 文件丢了 → 重新编码
        executable_worker.Ffmpeg.EncodeVideoThumbnail(...)
        ctx.GetDB().Save(videoThumbnailURL)
    }
}
```

> **注意**：Web 视频（`VideoWeb`）在 `ProcessVideoTask` 中只查了 DB，没做磁盘文件存在性校验。磁盘文件丢失的兜底发生在 HTTP 请求阶段（见下文）。

### 9.3 HTTP 请求时的缓存兜底补转码

**文件**：`api/routes/videos.go:81-118`

```go
// 1. 先尝试从磁盘读
if _, err := os.Stat(cachedPath); err != nil {
    if os.IsNotExist(err) {
        // 2. 磁盘不存在 → 调用 ProcessSingleMedia 完整走一遍处理流程
        if err := processSingleMediaFn(r.Context(), db, media); err != nil {
            // 错误处理...
        }
        // 3. 处理完再检查一次
        if _, err := os.Stat(cachedPath); err != nil {
            // 还是没有 → 返回 500
        }
    }
}
// 4. 存在 → 设置一年强缓存头返回
w.Header().Set("Cache-Control", "private, max-age=31536000, immutable")
http.ServeFile(w, r, cachedPath)
```

`ProcessSingleMedia` 本质是走完整的 `scanMedia` 流程：

**文件**：`api/scanner/scanner_media.go:77-93`

```go
func ProcessSingleMedia(ctx context.Context, db *gorm.DB, media *models.Media) error {
    albumCache := scanner_cache.MakeAlbumCache()
    var album models.Album
    db.Model(media).Association("Album").Find(&album)
    mediaData := media_encoding.NewEncodeMediaData(media)
    taskContext := scanner_task.NewTaskContext(ctx, db, &album, albumCache)
    return scanMedia(taskContext, media, &mediaData, 0, 1)
}
```

### 9.4 缓存命名与唯一性

**文件**：`api/scanner/scanner_tasks/processing_tasks/processing_helpers.go:41-51`

```go
func generateUniqueMediaName(mediaPath string) string {
    filename := path.Base(mediaPath)
    baseName := filename[0 : len(filename)-len(path.Ext(filename))]
    baseExt := path.Ext(filename)
    mediaName := fmt.Sprintf("%s_%s", baseName, utils.GenerateToken())
    mediaName = models.SanitizeMediaName(mediaName) + baseExt
    return mediaName
}
```

缓存文件名 = `原文件名_随机Token`，经过 `SanitizeMediaName` 清理（去除 `/`、`\`、空格替换为 `_`、点替换为 `_`）。这使得每个处理产物都有唯一文件名，避免覆盖。

---

## 十、数据模型关联

**文件**：`api/graphql/models/media.go`

```
Media (视频)
├── ID, Title, Path, PathHash, AlbumID, DateShot
├── Type = "video"
├── VideoMetadataID → VideoMetadata (1:1)
│   ├── Width, Height, Duration, Codec, Framerate, Bitrate, ColorProfile, Audio
└── MediaURL (1:N)
    ├── MediaID, MediaName (缓存文件名)
    ├── Width, Height, FileSize
    ├── Purpose: "original" | "video-web" | "video-thumbnail"
    └── ContentType: "video/mp4" | "image/jpeg" | ...
```

MediaURL 的 URL 路由由其 `Purpose` 决定：
- `VideoWeb` → `/video/{MediaName}` （`models/media.go:121-125`）
- 其他 → `/photo/{MediaName}`

---

## 十一、关键文件索引

| 文件 | 职责 |
|------|------|
| `api/graphql/resolvers/scanner.go` | GraphQL 扫描触发入口 |
| `api/scanner/periodic_scanner/periodic_scanner.go` | 定时扫描调度 |
| `api/scanner/scanner_queue/queue.go` | 任务队列与并发 worker 调度 |
| `api/scanner/scanner_album.go` | 相册扫描、媒体文件发现 |
| `api/scanner/scanner_media.go` | 单媒体扫描入口 `ProcessSingleMedia` |
| `api/scanner/media_scan.go` | `scanMedia()` 任务链调度 |
| `api/scanner/scanner_tasks/scanner_tasks.go` | 所有扫描任务的注册与串行调度 |
| `api/scanner/scanner_tasks/processing_tasks/process_video_task.go` | **视频处理核心**：原始登记、MP4 转码、缩略图生成 |
| `api/scanner/scanner_tasks/processing_tasks/processing_helpers.go` | `makePhotoURLChecker` 缓存存在性检查、文件名生成 |
| `api/scanner/scanner_tasks/video_metadata_task.go` | ffprobe 视频元数据采集 |
| `api/scanner/media_encoding/executable_worker/executable_worker.go` | FFmpeg/ffprobe/Magick 初始化 |
| `api/scanner/media_encoding/executable_worker/ffmpeg_cli.go` | **FFmpeg 命令构建**：`EncodeMp4`、`EncodeVideoThumbnail` |
| `api/scanner/media_encoding/encode_photo.go` | `EncodeMediaData`（带缓存的媒体数据包装器） |
| `api/scanner/media_type/media_type.go` | MIME 类型识别、Web 兼容性判断 |
| `api/utils/media_cache.go` | 缓存路径计算与目录创建 |
| `api/routes/videos.go` | `/video/*` HTTP 路由、缓存兜底补转码 |
| `api/graphql/models/media.go` | `Media`、`MediaURL`、`VideoMetadata` 模型定义 |

---

## 十二、关于 "片段化" 的说明

经过对代码库的完整搜索（含 `hls`、`m3u8`、`segment`、`fragment` 等关键词），Photoview **当前版本不支持 HLS/DASH 等视频分片流式传输**。它的处理策略是：

1. **全量转码**：将整个视频一次性转码为 MP4(H.264+AAC)
2. **渐进式下载**：通过 `-movflags +faststart` 将 moov atom 前置，使浏览器可以在未完全下载完成前开始播放
3. **HTTP 静态文件服务**：通过 `http.ServeFile` 直接输出 MP4 文件，依赖 HTTP Range 请求实现拖动进度条

如果未来需要支持真正的 HLS 片段化（`.m3u8` 播放列表 + `.ts` 分片），需要在 `ffmpeg_cli.go` 中新增类似 `EncodeHLS` 的方法，使用 ffmpeg 的 `-f hls -hls_time 10 -hls_list_size 0` 等参数，并在 `ProcessVideoTask` 中增加对应 Purpose 和路由处理。

---

## 十三、深度细节分析

### 13.1 全量与增量扫描的区分

**结论：Photoview 采用"全量目录扫描 + 增量式处理"的混合模式。**

#### 全量扫描（目录遍历是全量的）

每次扫描都会递归遍历用户所有根相册目录下的所有文件：

**文件**：`api/scanner/scanner_user.go:45` → `FindAlbumsForUser()`

```go
// BFS 遍历所有子目录
scanQueue := list.New()
for _, album := range userRootAlbums {
    scanQueue.PushBack(scanInfo{path: album.Path, ...})
}
for scanQueue.Front() != nil {
    albumInfo := scanQueue.Front().Value.(scanInfo)
    scanQueue.Remove(scanQueue.Front())
    dirContent, _ := os.ReadDir(albumPath)
    // 每个子目录都入队
    for _, item := range dirContent {
        if item.IsDir() {
            scanQueue.PushBack(scanInfo{path: subalbumPath, ...})
        }
    }
}
```

每次扫描都是从根目录开始完整遍历一遍，没有基于 mtime 或文件大小的跳过优化。

#### 增量处理（已存在的媒体跳过处理）

虽然目录是全量扫，但媒体的发现和处理是增量的：

**第一级去重：媒体级（path_hash）**

**文件**：`api/scanner/scanner_media.go:21-38` → `ScanMedia()`

```go
// 用 path_hash 检查媒体是否已存在于数据库
var media []*models.Media
result := tx.Where("path_hash = ?", models.MD5Hash(mediaPath)).Find(&media)
if result.RowsAffected > 0 {
    return media[0], false, nil  // isNewMedia = false，已存在就跳过
}
```

`path_hash` 是文件路径的 MD5（见 13.4 详解），只与路径有关，与内容无关。

**第二级去重：处理产物级（MediaURL 记录）**

**文件**：`api/scanner/scanner_tasks/processing_tasks/process_video_task.go:34-49`

```go
// 对三种 Purpose 分别查 DB
mediaURLFromDB := makePhotoURLChecker(ctx.GetDB(), video.ID)
videoOriginalURL, _  := mediaURLFromDB(models.MediaOriginal)
videoWebURL, _       := mediaURLFromDB(models.VideoWeb)
videoThumbnailURL, _  := mediaURLFromDB(models.VideoThumbnail)

// 有记录就跳过对应阶段
if videoWebURL == nil && !videoType.IsWebCompatible() { ... 转码 ... }
if videoThumbnailURL == nil { ... 生成缩略图 ... }
```

**新旧媒体标识**

`AfterMediaFound` 钩子会收到 `newMedia bool` 参数：
- `newMedia = true`：第一次发现的新媒体，触发 `VideoMetadataTask` 采集元数据
- `newMedia = false`：已存在的媒体，跳过元数据采集

但不管新旧，都会进入 `scanMedia()` 走完整的处理任务链，只是任务内部会通过 DB 检查自动跳过已完成的步骤。

> **实际效果**：首次扫描 = 全量处理；后续扫描 = 全量目录遍历 + 仅处理新文件 + 校验旧文件缓存完整性。每次扫描的目录遍历开销是 O(N) 的，但处理开销是 O(新增文件数) 的。

---

### 13.2 FFmpeg 子进程超时与中断机制

**结论：FFmpeg 子进程本身没有超时控制，但上层有两种中断路径。**

#### FFmpeg 命令无超时

**文件**：`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:89-93`

```go
func (cli *FfmpegCli) EncodeMp4(inputPath string, outputPath string) error {
    args := [...]
    cmd := exec.Command(cli.path, args...)  // ← 用的是 exec.Command，不是 CommandContext
    return cmd.Run()                         // ← 阻塞等待，没有超时
}
```

**关键发现**：
- 直接使用 `exec.Command` 而非 `exec.CommandContext`
- `EncodeMp4()` 和 `EncodeVideoThumbnail()` 都没有传入 context，也没有设置超时
- 理论上一个大视频可以无限期地转码下去

#### 两种中断路径

**路径 1：HTTP 请求取消（客户端断开连接）**

**文件**：`api/routes/videos.go:92-99`

```go
if err := processSingleMediaFn(r.Context(), db, media); err != nil {
    if r.Context().Err() != nil && errors.Is(r.Context().Err(), context.Canceled) {
        log.Warn(r.Context(), "video processing cancelled due to client disconnect", ...)
        return  // 客户端断了，直接返回，不发响应
    }
}
```

但注意：**context 取消只能中断 Go 层面的逻辑，不能直接杀掉正在运行的 ffmpeg 子进程**。因为 ffmpeg 调用没有传入 context，子进程会继续在后台运行直到完成，只是 Go 代码不再等它了。这是一个潜在的资源泄漏点。

**路径 2：扫描任务的 context 取消**

`scanner_task.TaskContext` 内嵌了 `context.Context`，任务链的每一步都会检查 `ctx.Done()`：

**文件**：`api/scanner/scanner_tasks/scanner_tasks.go:38-49`

```go
func simpleCombinedTasks(...) error {
    for _, task := range allTasks {
        select {
        case <-ctx.Done():    // ← 每个任务前检查 context 是否取消
            return ctx.Err()
        default:
        }
        err := doTask(ctx, task)
        ...
    }
}
```

但同样的问题：只能在 ffmpeg 命令结束后、进入下一个任务时才能检测到取消。正在运行的 ffmpeg 子进程不会被中断。

#### ffprobe 有超时

相比之下，ffprobe 是有超时的：

**文件**：`api/utils/environment_variables.go:87-95`

```go
func MediaProbeTimeout() time.Duration {
    if val := EnvMediaProbeTimeout.GetValue(); val != "" {
        if seconds, err := strconv.Atoi(val); err == nil && seconds > 0 {
            return time.Duration(seconds) * time.Second
        }
    }
    return 5 * time.Second  // 默认 5 秒
}
```

**文件**：`api/scanner/scanner_tasks/processing_tasks/process_video_task.go:206-216`

```go
func ReadVideoMetadata(videoPath string) (*ffprobe.ProbeData, error) {
    ctx, cancelFn := context.WithTimeout(context.Background(), utils.MediaProbeTimeout())
    defer cancelFn()
    data, err := ffprobe.ProbeURL(ctx, videoPath)  // ← 用了带超时的 context
    ...
}
```

> **总结**：ffprobe 有 5 秒默认超时；ffmpeg 转码没有超时，也不能被 context 取消中断，只能被动等待完成。HTTP 请求端虽然会检测取消，但只是不等了而已，子进程还在跑。

---

### 13.3 断电时部分写入清理

**结论：没有显式的部分写入清理机制，但依靠"先写文件、后写 DB"的顺序 + 下次扫描重处理，实现了最终一致性。**

#### 视频转码的写入顺序

**文件**：`api/scanner/scanner_tasks/processing_tasks/process_video_task.go:87-125`

```go
// 1. 直接写最终文件名
webVideoPath := path.Join(mediaCachePath, webVideoName)
executable_worker.Ffmpeg.EncodeMp4(video.Path, webVideoPath)  // ← 直接写目标路径

// 2. 转码成功后才写 DB
mediaURL := models.MediaURL{...}
ctx.GetDB().Create(&mediaURL)  // ← 成功后才入库
```

**断电场景分析**：

| 断电时机 | 后果 | 恢复方式 |
|----------|------|----------|
| 转码过程中断电 | 磁盘上有不完整的 `.mp4` 文件，DB 无记录 | 下次扫描/请求时 DB 查不到 → 重新转码，覆盖不完整文件 |
| 刚写完文件、还没写 DB 时断电 | 同上 | 同上 |
| DB 写入中断电 | 事务回滚（`DatabaseTransaction` 包在事务里），相当于没写 | 同上 |

> 关键：因为 **文件写入不在数据库事务内**，且 **直接写最终文件名**，所以断电后会留下残次文件。但下次处理时因为 DB 查不到 MediaURL 记录，会重新转码覆盖掉不完整的文件，最终是一致的。

#### 对比：图片处理有临时文件模式（视频没有）

有趣的是，在 sidecar 任务中图片处理用了 `.hold` 后缀的临时文件模式：

**文件**：`api/scanner/scanner_tasks/processing_tasks/sidecar_task.go:99-117`

```go
tempHighResPath := baseImagePath + ".hold"  // 先备份原文件
os.Rename(baseImagePath, tempHighResPath)
// ... 重新生成 ...
if err != nil {
    os.Rename(tempHighResPath, baseImagePath)  // 失败就回滚
    return err
}
os.Remove(tempHighResPath)  // 成功才删掉备份
```

但视频转码没有用这种模式，直接写目标路径。

#### 清理任务的职责

`MediaCleanupTask`（`cleanup_media.go`）只清理**DB 中存在但磁盘上源文件已删除**的媒体（即用户删了源文件，清理缓存和 DB 记录），不是用来清理部分写入的。

> **风险点**：如果转码到一半失败且留下了不完整文件，而 DB 还没写，这个不完整文件会一直躺在缓存目录里占空间。因为文件名带随机 token（`web_video_xxx_<token>.mp4`），下次重新转码会用**新的随机 token** 写新文件，旧的残次文件就成了孤儿文件，永远不会被清理。

---

### 13.4 Hash 是否考虑元数据修改

**结论：完全不考虑。`path_hash` 只是文件路径的 MD5，与内容、元数据、文件大小、修改时间统统无关。**

#### MD5Hash 实现

**文件**：`api/graphql/models/utils.go:42-46`

```go
// MD5Hash hashes value to a 32 length digest, the result is the same as the MYSQL function md5()
func MD5Hash(value string) string {
    hash := md5.Sum([]byte(value))
    return hex.EncodeToString(hash[:])
}
```

**文件**：`api/graphql/models/media.go:39-44`

```go
func (m *Media) BeforeSave(tx *gorm.DB) error {
    m.PathHash = MD5Hash(m.Path)  // ← 只哈希路径字符串
    return nil
}
```

#### 这意味着什么

| 场景 | 是否重新扫描/处理 |
|------|------------------|
| 文件路径不变，内容被替换（如覆盖一个同名视频） | ❌ 不会。`path_hash` 不变，DB 认为是同一个媒体，不会重新转码 |
| 文件改名（路径变了） | ✅ 会。`path_hash` 变了，当成新媒体重新处理 |
| 文件移动到别的相册 | ✅ 会。路径变了 |
| EXIF/元数据修改（文件内容变了但路径不变） | ❌ 不会 |
| 视频重新剪辑后替换原文件 | ❌ 不会。缓存的转码结果还是旧的 |

> **设计意图推测**：Photoview 定位是"只读地浏览你的照片/视频库"，假设源文件是静态的、只会新增不会修改。这与面向用户上传的系统（需要检测文件变化）有本质区别。

#### 补充：缓存文件名的随机性

还有一个有趣的点：缓存文件名带随机 token（`web_video_filename_abc123.mp4`），即使源文件没改，如果 DB 记录丢了重新生成，缓存文件名也会不一样。这说明缓存没有"内容寻址"的设计，完全靠 DB 记录来关联。

---

### 13.5 VAAPI 与 NVENC 硬件加速分支

**结论：支持三种硬件加速（QSV、VAAPI、NVENC），通过环境变量切换，默认软编码 H.264。**

#### 编解码器映射表

**文件**：`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:13-19`

```go
const defaultCodec = "h264"

var hwAccToCodec = map[string]string{
    "qsv":   defaultCodec + "_qsv",     // h264_qsv   - Intel Quick Sync Video
    "vaapi": defaultCodec + "_vaapi",   // h264_vaapi - Video Acceleration API (Linux)
    "nvenc": defaultCodec + "_nvenc",   // h264_nvenc - NVIDIA NVENC
}
```

#### 初始化逻辑

**文件**：`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:51-67`

```go
func newFfmpegCli() *FfmpegCli {
    ...
    hwAcc := utils.EnvVideoHardwareAcceleration.GetValue()
    codec, ok := hwAccToCodec[hwAcc]
    if !ok {
        if strings.HasPrefix(hwAcc, "_") {
            // A secret way to set the codec directly.
            codec = hwAcc[1:]  // ← 彩蛋：下划线开头直接当编码器名用
        } else {
            codec = defaultCodec  // 默认 h264 软编
        }
    }
    ...
}
```

#### 环境变量配置

**文件**：`api/utils/environment_variables.go:46`

```go
EnvVideoHardwareAcceleration EnvironmentVariable = "PHOTOVIEW_VIDEO_HARDWARE_ACCELERATION"
```

#### 支持的配置值

| 配置值 | 编码器 | 适用硬件 |
|--------|--------|----------|
| （空或其他） | `h264` (libx264) | 通用 CPU 软编码 |
| `qsv` | `h264_qsv` | Intel 核显 (Quick Sync Video) |
| `vaapi` | `h264_vaapi` | Linux VA-API (AMD/Intel 通用) |
| `nvenc` | `h264_nvenc` | NVIDIA GPU |
| `_<任意编码器名>` | 直接使用 | 调试/高级用户（如 `_libx265` 试试 HEVC） |

#### 注意事项

1. **只有视频编码走硬件加速，音频固定是 AAC 软编**（`-acodec aac`）
2. **没有可用性探测**：设置了 `nvenc` 但系统没有 NVIDIA 显卡的话，ffmpeg 会直接报错退出，转码失败
3. **没有降级策略**：硬件加速失败不会自动回退到软编码
4. **彩蛋功能**：`_` 开头可以指定任意 ffmpeg 编码器名，给高级用户调试用。比如设为 `_libx265` 就会用 H.265 编码（但前端播放器兼容性没保证）

---

### 13.6 并发同片段去重机制

**结论：相册级有去重，媒体级没有去重。同一个视频可能被并发转码多次。**

#### 相册级去重（队列层面）

**文件**：`api/scanner/scanner_queue/queue.go:253-263`

```go
func (queue *ScannerQueue) jobOnQueue(job *ScannerJob) (bool, error) {
    scannerJobs := append(queue.in_progress, queue.up_next...)
    for _, scannerJob := range scannerJobs {
        if scannerJob.ctx.GetAlbum().ID == job.ctx.GetAlbum().ID {  // ← 按相册 ID 去重
            return true, nil
        }
    }
    return false, nil
}
```

同一个相册不会同时有两个扫描任务在跑。

#### 媒体级：没有去重

媒体层面**没有任何互斥**。同一个媒体可能被并发处理多次：

**场景举例**：
1. 扫描任务正在后台转码视频 A
2. 恰好用户在浏览器打开视频 A，缓存未命中
3. HTTP 请求触发 `ProcessSingleMedia()` 也开始转码视频 A
4. 两个 ffmpeg 进程同时在跑，各写各的文件（因为文件名带随机 token，不会冲突）

#### 为什么不会冲突但会重复劳动

缓存文件名格式：`web_video_<原文件名>_<随机token>.mp4`

**文件**：`api/scanner/scanner_tasks/processing_tasks/process_video_task.go:88-91`

```go
webVideoName := fmt.Sprintf("web_video_%s_%s", path.Base(video.Path), utils.GenerateToken())
// web_video_mymovie_abc123def.mp4
```

每次转码都用新的随机 token，所以两个并发转码会写两个不同的文件。最后 DB 里可能会有两条 MediaURL 记录（都指向同一个 media_id，purpose 都是 video-web），查询时取最新的：

**文件**：`api/routes/videos.go:32-37`

```go
result := db.Model(&models.MediaURL{}).
    Where("media_urls.media_name = ? AND media_urls.purpose = ?", mediaName, models.VideoWeb).
    Order("created_at DESC").  // ← 多个的话按时间倒序，取第一个
    Find(&mediaURLs)
```

> **影响**：并发转码不会出错，但会浪费 CPU 和磁盘 IO。对于大视频文件，可能同时跑两个 ffmpeg 把系统资源吃满。

---

### 13.7 磁盘空间不足降级策略

**结论：完全没有。磁盘满了就直接报错，没有任何降级策略。**

#### 现状

搜索 `disk`、`space`、`ENOSPC` 等关键词，整个代码库中没有任何磁盘空间检查逻辑。

转码过程中如果磁盘空间不足：
1. FFmpeg 写文件会失败，返回非零退出码
2. `cmd.Run()` 返回 error
3. 错误层层向上传递，任务失败
4. 记录一条错误日志，继续处理下一个文件

**文件**：`api/scanner/scanner_album.go:103-106`

```go
if err := scanMedia(ctx, media, &mediaData, i, len(albumMedia)); err != nil {
    scanner_utils.ScannerError(ctx, "Error scanning media for album (%d) file (%s): %s\n", ...)
    // ← 只是打日志，继续下一个
}
```

#### 没有的功能

- ❌ 转码前检查剩余磁盘空间
- ❌ 磁盘不足时自动降低分辨率/码率
- ❌ 磁盘不足时跳过视频转码（只保留原始文件）
- ❌ 磁盘不足时暂停扫描等待
- ❌ 磁盘空间阈值告警

> **实际行为**：扫到哪个文件磁盘满了，那个文件就失败，后面的继续试，一个个失败，直到全部失败或者用户发现。

---

### 13.8 HLS 完成通知前端 Channel

**结论：没有 HLS 自然也没有 HLS 完成通知。但有通用的扫描进度通知系统，基于 WebSocket + Channel 广播。**

#### 通知系统架构

```
  ┌─────────────────────┐
  │  scanner 任务产生    │
  │  Notification 对象  │
  └─────────┬───────────┘
            │
            ▼
  ┌─────────────────────┐
  │ BroadcastNotification│
  │ (遍历所有 listener)  │
  └─────────┬───────────┘
            │
            ▼
  ┌──────────────────────────┐
  │ 每个 WebSocket 连接一个  │
  │  Go channel → 前端接收   │
  └──────────────────────────┘
```

#### 通知注册与广播

**文件**：`api/graphql/notification/Notification.go:28-40`

```go
var notificationListeners []*NotificationListener = make([]*NotificationListener, 0)
var notificationLock = &sync.Mutex{}

func RegisterListener(user *models.User, channel NotificationChannel) int {
    notificationLock.Lock()
    defer notificationLock.Unlock()
    notificationListeners = append(notificationListeners, NewListener(*user, channel))
    return nextNotificationId
}
```

**文件**：`api/graphql/notification/Notification.go:70-82`

```go
func BroadcastNotification(notification *models.Notification) {
    notificationLock.Lock()
    defer notificationLock.Unlock()
    for _, listener := range notificationListeners {
        listener.channel <- notification  // ← 逐个发往每个 listener 的 channel
    }
}
```

> 注意：这里是**阻塞式发送**（`listener.channel <-`），如果某个前端 WebSocket 消费慢了，会阻塞整个广播。而且是全局一把大锁。

#### 通知类型

视频处理相关的通知由 `NotificationTask` 发出：

**文件**：`api/scanner/scanner_tasks/notification_task.go:45-60`

```go
func (t NotificationTask) AfterProcessMedia(..., updatedURLs []*models.MediaURL, mediaIndex int, mediaTotal int) error {
    if len(updatedURLs) > 0 {
        progress := float64(mediaIndex) / float64(mediaTotal) * 100.0
        notification.BroadcastNotification(&models.Notification{
            Key:      t.albumKey,
            Type:     models.NotificationTypeProgress,  // 进度通知
            Header:   fmt.Sprintf("Processing media for album '%s'", ...),
            Progress: &progress,
        })
    }
    return nil
}
```

**文件**：`api/scanner/scanner_tasks/notification_task.go:62-78`

```go
func (t NotificationTask) AfterScanAlbum(...) error {
    if len(changedMedia) > 0 {
        timeoutDelay := 2000
        notification.BroadcastNotification(&models.Notification{
            Type:     models.NotificationTypeMessage,
            Positive: true,  // 成功通知
            Header:   fmt.Sprintf("Done processing media for album '%s'", ...),
            Timeout:  &timeoutDelay,
        })
    }
    return nil
}
```

#### 与视频转码的关系

- 视频转码完成后，如果产生了新的 `MediaURL`，`AfterProcessMedia` 会发一条进度通知
- 整个相册扫描完会发完成通知
- **但没有专门的"单个视频转码完成"事件**，前端只能看到整体进度百分比，不知道哪个视频刚转好

#### WebSocket 连接

**文件**：`api/server/websocket.go` → `WebsocketUpgrader()`

通知通过 GraphQL Subscription 推送到前端 WebSocket 连接。每个连接对应一个 channel，注册到全局 listener 列表。

---

## 十五、深度细节分析（续）

### 15.1 时钟回拨下 last_scan_time 兜底

**结论：没有 `last_scan_time` 字段，也没有时钟回拨保护。周期性扫描基于 `time.Ticker`，时钟回拨会导致 ticker 不准，但不会崩溃。**

#### 没有 last_scan_time

搜索 `last_scan`、`lastScan`、`scan_time` 等关键词，整个代码库**完全没有**记录上次扫描时间的字段或逻辑。扫描不是"增量时间窗"模式（只处理上次扫描后变更的文件），而是每次都完整遍历。

#### 周期性扫描实现

**文件**：`api/scanner/periodic_scanner/periodic_scanner.go`

```go
type periodicScanner struct {
    ticker         *time.Ticker
    tickerLocker   sync.Mutex
    ticker_changed chan bool
    done           chan struct{}
    ...
}

func (ps *periodicScanner) scanIntervalRunner() {
    var scanTicker <-chan time.Time
    // line 99-120
    ps.tickerLocker.Lock()
    if ps.ticker != nil {
        scanTicker = ps.ticker.C
    }
    ps.tickerLocker.Unlock()

    for {
        select {
        case <-scanTicker:
            log.Println("Starting periodic scan")
            ps.scannerQueue.AddAllToQueue()  // 触发全量扫描
        case <-ps.ticker_changed:
            // 配置变更，重建 ticker
        case <-ps.done:
            return
        }
    }
}
```

#### 时钟回拨的影响

`time.Ticker` 基于单调时钟（monotonic clock），在 Go 1.9+ 中不受系统时间回拨影响。但 `time.Now()` 用于其他时间计算时如果用到了墙上时钟（wall clock），可能出现问题。

**实际风险**：
- 如果系统时间被向后拨了 1 小时，已设置的 `time.Ticker` 会继续按原间隔 tick（因为用的是单调时钟）
- 但如果扫描间隔配置是 "每天凌晨 3 点" 这类基于绝对时间的调度（当前不是），就会有问题
- 当前 Photoview 是简单的 "每隔 N 秒扫一次"，时钟回拨不会导致扫描停止

#### 没有的保护

- ❌ 没有记录上次扫描时间的 DB 字段
- ❌ 没有 `last_scan > now` 的时钟回拨检测和修正
- ❌ 没有基于时间窗的增量扫描（只扫上次扫描后变更的文件）

---

### 15.2 FFmpeg 超时后子进程清理

**结论：完全没有。FFmpeg 子进程没有超时，没有 context，也没有清理孤儿进程的机制。**

#### 现状确认

**文件**：`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:89-93`

```go
func (cli *FfmpegCli) EncodeMp4(inputPath string, outputPath string) error {
    args := [...]
    cmd := exec.Command(cli.path, args...)  // ← exec.Command，不带 context
    return cmd.Run()                         // ← 无限期阻塞
}
```

#### 会发生什么

| 场景 | 后果 |
|------|------|
| 视频损坏导致 ffmpeg 挂起 | Go 代码永久阻塞，goroutine 泄漏，占用一个 worker 槽位 |
| HTTP 请求取消，Go 代码不等了但进程还在跑 | 子进程变成孤儿继续跑，直到完成或失败，占用 CPU 和磁盘 IO |
| 程序退出（Ctrl+C / 容器重启）而 ffmpeg 还在跑 | 子进程被 init 进程接管，继续后台运行，直到完成 |
| ffmpeg 死循环（罕见） | 进程永远存在，需要手动 `kill` |

#### 对比：ffprobe 有超时

**文件**：`api/scanner/scanner_tasks/processing_tasks/process_video_task.go:206-216`

```go
func ReadVideoMetadata(videoPath string) (*ffprobe.ProbeData, error) {
    ctx, cancelFn := context.WithTimeout(context.Background(), utils.MediaProbeTimeout())
    defer cancelFn()
    data, err := ffprobe.ProbeURL(ctx, videoPath)  // ← 带 5 秒超时
    ...
}
```

#### 改进需要做什么

```go
// 应该改成这样：
func (cli *FfmpegCli) EncodeMp4(ctx context.Context, inputPath string, outputPath string) error {
    // 加超时（比如 4 小时，防止无限期挂起）
    ctx, cancel := context.WithTimeout(ctx, 4*time.Hour)
    defer cancel()

    cmd := exec.CommandContext(ctx, cli.path, args...)  // ← 用 CommandContext
    return cmd.Run()
}
```

`CommandContext` 会在 context 取消时调用 `cmd.Process.Kill()` 真正杀掉子进程。

---

### 15.3 `.cache` 子目录残留处理

**结论：有隐藏文件过滤，但没有专门的 `.cache` 目录清理逻辑。**

#### 隐藏文件过滤

**文件**：`api/scanner/scanner_cache/cache.go:108-114` → `IsPathMedia()`

```go
func (c *AlbumScannerCache) IsPathMedia(mediaPath string) bool {
    // Ignore hidden files
    if path.Base(mediaPath)[0:1] == "." {  // ← 文件名以 . 开头就跳过
        return false
    }
    ...
}
```

这意味着：
- `.cache/` 目录本身不会被当成媒体（因为是目录）
- `.cache/` 目录下的文件也不会被当成媒体（因为目录被跳过，不会递归进去）
- 但如果 `.cache/` 目录就在相册根目录下，目录遍历时会被 `os.ReadDir` 读到，然后被 `IsDir()` 跳过

#### `.photoview_ignore` 支持

**文件**：`api/scanner/scanner_tasks/ignorefile_task.go`

支持 `.photoview_ignore` 文件列出要忽略的模式，但 `.cache` 不是默认忽略项，需要用户手动配置。

#### 残留问题

- ❌ 缓存目录 `media_cache/` 本身不会被扫描（因为它是独立配置的，不在相册根目录内）
- ❌ 但如果用户在相册目录内手动创建了 `.cache/` 目录放东西，不会被扫描
- ❌ 没有定期清理空目录或孤儿缓存目录的逻辑
- ❌ 转码失败留下的带随机 token 的残次文件（见 13.3）不会被自动清理

#### 对比：`.thumbnail/` 等目录

其他软件常见的缩略图目录如 `.thumbnails/`、`@eaDir/`（Synology）、`.DS_Store` 等**都没有特殊处理**，统一走"点开头就跳过"的规则。

---

### 15.4 SHA256 大文件性能

**结论：根本没有 SHA256。只有 MD5，且只用于字符串哈希（路径）和小文件内容哈希（sidecar XMP）。**

#### MD5 的两处使用

**使用 1：路径哈希（完全不读文件内容）**

**文件**：`api/graphql/models/utils.go:42-46`

```go
func MD5Hash(value string) string {
    hash := md5.Sum([]byte(value))  // ← 只哈希字符串，跟文件内容无关
    return hex.EncodeToString(hash[:])
}
```

用于：
- `Media.PathHash` = `MD5Hash(media.Path)` （`models/media.go:41`）
- `Album.PathHash` = `MD5Hash(album.Path)` （`scanner_user.go:132`）

**使用 2：sidecar XMP 文件内容哈希（小文件）**

**文件**：`api/scanner/scanner_tasks/processing_tasks/sidecar_task.go:151-158`

```go
func processSidecarFile(...) error {
    f, _ := os.Open(sidecarPath)
    defer f.Close()
    h := md5.New()
    if _, err := io.Copy(h, f); err != nil {  // ← 读整个文件算 MD5
        log.Printf("ERROR: %s", err)
    }
    hash := hex.EncodeToString(h.Sum(nil))
    if media.SidecarHash == nil || *media.SidecarHash != hash {
        // sidecar 变了，重新处理图片
        ...
    }
}
```

#### 性能分析

| 场景 | 算法 | 数据量 | 性能 |
|------|------|--------|------|
| 媒体去重 | MD5 | 路径字符串（~100字节） | 极快，O(1)，不碰磁盘 |
| Sidecar 变更检测 | MD5 | XMP 文件（通常 < 1MB） | 快，小文件全读没问题 |
| 视频内容变更检测 | - | - | 完全没做 |

#### 为什么不用 SHA256

- 对于路径哈希，防碰撞不是主要目标，MD5 足够快
- 对于 sidecar 哈希，只是检测变更，不是防篡改，MD5 足够
- 视频大文件如果做内容哈希，无论 MD5 还是 SHA256 都很慢（读几十 GB），所以干脆没做

> **注意**：这里存在一个设计权衡——不做内容哈希意味着无法检测文件替换（同路径不同内容），但换来的是每次扫描不需要读几十 GB 的视频文件，性能提升巨大。对于"只读媒体库"场景，这个权衡是合理的。

---

### 15.5 AMD AMF 硬件加速支持

**结论：目前不支持 AMD AMF。只支持 Intel QSV、VA-API、NVIDIA NVENC。**

#### 当前支持的硬件加速

**文件**：`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:13-19`

```go
const defaultCodec = "h264"

var hwAccToCodec = map[string]string{
    "qsv":   defaultCodec + "_qsv",     // h264_qsv   - Intel Quick Sync Video
    "vaapi": defaultCodec + "_vaapi",   // h264_vaapi - Video Acceleration API
    "nvenc": defaultCodec + "_nvenc",   // h264_nvenc - NVIDIA NVENC
}
```

#### AMD 用户的选择

| 方案 | 说明 |
|------|------|
| **VA-API** | AMD 显卡在 Linux 上可以用 mesa 的 VA-API 驱动（`mesa-va-drivers` / `amdgpu`），设为 `vaapi` 即可。这是 AMD 用户的首选。 |
| **AMF 编码器** | FFmpeg 中有 `h264_amf` 编码器（AMD Advanced Media Framework），但当前没有映射。可以通过彩蛋功能使用：`PHOTOVIEW_VIDEO_HARDWARE_ACCELERATION=_h264_amf`（见 13.5 彩蛋说明） |
| **软编码** | 留空或其他值，默认 `h264` (libx264) 软编码 |

#### 彩蛋功能的验证

**文件**：`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:57-62`

```go
codec, ok := hwAccToCodec[hwAcc]
if !ok {
    if strings.HasPrefix(hwAcc, "_") {
        codec = hwAcc[1:]  // ← 下划线开头，直接当编码器名
    } else {
        codec = defaultCodec
    }
}
```

所以 AMD 用户想用 AMF 的话：
```bash
PHOTOVIEW_VIDEO_HARDWARE_ACCELERATION=_h264_amf
```

#### AMF 支持的改进

如果要正式支持，只需加一行：
```go
var hwAccToCodec = map[string]string{
    "qsv":   defaultCodec + "_qsv",
    "vaapi": defaultCodec + "_vaapi",
    "nvenc": defaultCodec + "_nvenc",
    "amf":   defaultCodec + "_amf",     // ← 加这行
}
```

但需要注意：
- AMF 只在 Windows 和 Linux 上可用，macOS 不支持
- 需要 ffmpeg 编译时启用 `--enable-amf`
- 没有可用性检测，设置了不可用会直接报错

---

### 15.6 Mutex 细粒度死锁分析

**结论：目前是 5 把全局大锁，设计简单但有死锁风险和性能瓶颈。**

#### 5 把全局 Mutex 位置

| 锁 | 位置 | 保护的数据 | 粒度 |
|----|------|------------|------|
| `notificationLock` | `api/graphql/notification/Notification.go:30` | `notificationListeners` 全局 slice | 全局大锁，所有用户共享 |
| `global_scanner_queue.mutex` | `api/scanner/scanner_queue/queue.go:48` | `in_progress`、`up_next` 队列 | 全局大锁，所有用户共享 |
| `mainPeriodicScannerLocker` | `api/scanner/periodic_scanner/periodic_scanner.go:34` | `mainPeriodicScanner` 单例 | 全局大锁，初始化用一次 |
| `testCachePathLocker` | `api/utils/media_cache.go:43` | `testCachePath` 测试变量 | 测试用，RWMutex |
| `ps.tickerLocker` | `api/scanner/periodic_scanner/periodic_scanner.go:26` | `ps.ticker` | 单实例内部锁 |

另外还有每相册扫描实例的：
- `AlbumScannerCache.mutex`（`cache.go:16`） - 每个扫描任务一个，保护本地 map

#### 死锁风险分析

**风险 1：`notificationLock` 阻塞式发送 + 大锁 = 死锁**

**文件**：`api/graphql/notification/Notification.go:76-82`

```go
func BroadcastNotification(notification *models.Notification) {
    notificationLock.Lock()
    defer notificationLock.Unlock()

    for _, listener := range notificationListeners {
        listener.channel <- notification  // ← 阻塞式发送！
    }
}
```

channel 是无缓冲的（`make(chan *models.Notification, 1)` 有缓冲 1，但还是可能阻塞）。如果某个前端 WebSocket 消费慢了：
1. Goroutine A 拿着 `notificationLock`，阻塞在 `listener.channel <-`
2. Goroutine B 想调 `RegisterListener` 或 `DeregisterListener`，需要 `notificationLock.Lock()`，被阻塞
3. Goroutine C 想调 `BroadcastNotification`，也被阻塞
4. 整个通知系统完全挂死

**风险 2：锁顺序不一致**

目前的调用链：
- `scanner_queue.processQueue()` 拿 `mutex` → 调 `ScanAlbum` → 调 `notification.BroadcastNotification` 拿 `notificationLock`
- 没有反方向的调用，暂时不会死锁

但未来如果有通知回调里操作扫描队列，就会形成 `notificationLock` → `mutex` 的反序，产生死锁。

**风险 3：粗粒度锁的性能瓶颈**

- `notificationLock`：1000 个用户在线，每次通知要遍历 1000 个 listener，期间没人能注册/注销
- `global_scanner_queue.mutex`：所有用户的扫描任务都在一个队列里，加任务、取任务、完成任务都要抢这一把锁

#### 改进方向

1. `BroadcastNotification` 改为非阻塞发送或给 channel 足够的缓冲
2. 通知按用户分组，每个用户自己的锁
3. 扫描队列按用户/相册分片，减少锁竞争

---

### 15.7 NFS 文件系统差异处理

**结论：只有符号链接处理，没有针对 NFS 的特殊优化或兼容性处理。**

#### 已有的文件系统相关逻辑

**符号链接处理**

**文件**：`api/utils/utils.go:68-92` → `IsDirSymlink()`

```go
func IsDirSymlink(linkPath string) (bool, error) {
    fileInfo, err := os.Lstat(linkPath)  // ← Lstat 不跟随符号链接
    if err != nil {
        return false, err
    }
    if fileInfo.Mode()&os.ModeSymlink == os.ModeSymlink {
        resolvedPath, _ := filepath.EvalSymlinks(linkPath)  // ← 解析符号链接
        resolvedFile, _ := os.Stat(resolvedPath)            // ← Stat 跟随
        return resolvedFile.IsDir(), nil
    }
    return false, nil
}
```

使用位置：
- `scanner_user.go:206` - 扫描用户相册时
- `scanner_user.go:264` - 递归扫描子目录时
- `scanner_album.go:129` - 扫描单个相册时

#### NFS 特有的问题，Photoview 都没处理

| NFS 问题 | 影响 | Photoview 是否处理 |
|----------|------|-------------------|
| **不一致的 `os.Stat` 结果** | NFS 客户端缓存可能导致刚写的文件 `os.Stat` 查不到 | ❌ 没有，直接假设 `os.Stat` 是一致的 |
| **`os.Rename` 跨设备失败** | NFS 挂载点和本地磁盘之间 `rename` 会返回 EXDEV 错误 | ❌ 没有，`sidecar_task.go` 里的 `.hold` rename 逻辑假设在同一文件系统 |
| **文件句柄泄漏** | NFS 服务端重启会导致已打开的文件句柄失效（ESTALE） | ❌ 没有重试逻辑，打开失败就报错退出 |
| **弱一致性的 `os.ReadDir`** | NFS 目录列表可能不一致，刚创建的文件可能列不出来 | ❌ 没有，假设 `os.ReadDir` 是准确的 |
| **`mtime` 精度** | 某些 NFS 实现只有秒级 mtime 精度 | ❌ 不依赖 mtime，所以无所谓 |
| **锁的语义** | NFS 的文件锁（`fcntl(F_SETLK)`）语义特殊 | ❌ 没有用文件锁，用的是 Go 内存锁 |

#### 一个具体的 bug 场景

如果 `media_cache` 目录在 NFS 上，而源视频在本地磁盘（或另一个 NFS 挂载点）：

1. 转码过程中 NFS 服务端重启
2. `os.Stat(webVideoPath)` 返回 `ESTALE` 错误
3. 转码失败，错误日志记录
4. 下次扫描时，DB 中没有 MediaURL 记录，会重新转码
5. 但之前转了一半的文件可能因为 `ESTALE` 删不掉，变成孤儿

这不是严重问题（最终会重转），但会产生孤儿文件浪费空间。

#### 对 NFS 用户的建议

- 尽量把 `media_cache` 放在本地 SSD 上（转码需要高 IO）
- 如果必须用 NFS，确保 NFS 客户端挂载参数用 `hard`（默认）而不是 `soft`
- 定期清理孤儿缓存文件

---

### 15.8 WebSocket 断开重连机制

**结论：服务端没有断线重连状态保持，完全靠前端重连后重新订阅。**

#### 服务端实现

**文件**：`api/graphql/resolvers/notification.go:18-34`

```go
func (r *subscriptionResolver) Notification(ctx context.Context) (<-chan *models.Notification, error) {
    user := auth.UserFromContext(ctx)
    ...
    notificationChannel := make(chan *models.Notification, 1)  // 缓冲 1
    listenerID := notification.RegisterListener(user, notificationChannel)

    go func() {
        <-ctx.Done()                     // ← 连接断开时触发
        notification.DeregisterListener(listenerID)  // ← 直接注销，不保存状态
    }()

    return notificationChannel, nil
}
```

#### WebSocket 保活

**文件**：`api/graphql/endpoint/graphql_endpoint.go:31-35`

```go
graphqlServer.AddTransport(transport.Websocket{
    KeepAlivePingInterval: 10 * time.Second,  // ← 每 10 秒发 ping
    Upgrader:              server.WebsocketUpgrader(...),
    InitFunc:              auth.AuthWebsocketInit(),
})
```

gqlgen 内置的 websocket transport 会：
- 每 10 秒向客户端发 ping
- 客户端应该回 pong
- 收不到 pong 会认为连接已断开，取消 context

#### 断开重连的实际行为

```
  前端                          后端
    │                             │
    │───── 订阅 Notification ────▶│
    │                             │  注册 listener #123
    │◀──── 进度通知 1 ────────────│
    │◀──── 进度通知 2 ────────────│
    │   (网络闪断)                 │
    │                             │  <-ctx.Done()->  DeregisterListener(#123)
    │                             │  listener 被删了，没保存任何状态
    │                             │
    │───── 重连，重新订阅 ───────▶│
    │                             │  注册新 listener #124
    │◀──── 新的进度通知 ──────────│
    │                             │
```

**关键问题**：重连后是新的 listener，**之前断线期间的通知全部丢失**。前端只会收到重连后的新通知。

#### 对视频转码的影响

- 视频转码是后台任务，转码完成的进度通知如果恰好发生在断线窗口内，前端就收不到
- 但前端可以刷新页面手动检查（因为 DB 里已经有记录了）
- 没有"转码完成"的持久化事件队列，断线就丢了

#### 改进方向

1. 给通知加递增序列号，前端重连时可以请求"从序号 N 开始补发"
2. 或者在前端维护已处理媒体的本地缓存，重连后拉取最新状态 diff
3. 增加缓冲 channel 大小（当前只有 1），减少阻塞概率

---

## 十七、深度细节分析（再续）

### 17.1 重启后 monotonic clock 兜底失效

**结论：monotonic clock 在进程重启后自然失效，但 Photoview 的周期性扫描不依赖 monotonic clock 的持久性，所以重启后失效不是问题。**

#### Go 的 monotonic clock 特性

Go 的 `time.Ticker` 和 `time.Now()` 使用单调时钟（monotonic clock）只在**同一进程内有效**：
- 进程运行中：系统时间回拨不影响 ticker（用单调时钟不受影响）
- 进程重启后：单调时钟从零重新计数，之前的计时全部丢失

#### Photoview 的周期性扫描实现

**文件**：`api/scanner/periodic_scanner/periodic_scanner.go:24-31`

```go
type periodicScanner struct {
    ticker         *time.Ticker    // ← 基于 monotonic clock
    tickerLocker   sync.Mutex
    ticker_changed chan bool
    done           chan struct{}
    ...
}
```

#### 重启后的实际行为

```
进程启动时：
  ├─ InitializePeriodicScanner()
  │    ├─ 从 DB 读 PeriodicScanInterval
  │    └─ 创建新的 time.NewTicker(interval) ← 从零开始计时
  │
  └─ (比如 interval = 1 小时
     └─ 重启后 1 小时才会触发第一次扫描
```

**问题**：如果设置的是"每 24 小时扫一次"，每次重启后都要等 24 小时才会触发第一次扫描，而不是"距离上次扫描 24 小时后"。

#### 与"兜底失效的真实影响

| 场景 | 后果 |
|------|------|
| 设置 1 小时间隔，频繁重启 | 永远扫不到（每次重启都重新等 1 小时） |
| 设置 24 小时间隔，每天重启一次 | 每天都要等重启后 24 小时才扫，实际间隔远超 24 小时 |
| 容器滚动更新 | 下次扫描时间推迟 |

#### 对比：有 `last_scan_time` 方案的缺失

**完全没有 `last_scan_time` 字段（见 15.1），所以也没有"启动时检查距离上次扫描多久了，够间隔到了就立即扫"的逻辑。每次启动都是从零开始等。

#### 优雅关闭 vs 暴力关闭

**优雅关闭**（SIGTERM / Ctrl+C）：

**文件**：`api/server.go:140-160`

```go
func setupGracefulShutdown(svr *http.Server) {
    signal.Notify(c, os.Interrupt, syscall.SIGTERM)
    go func() {
        <-c
        ctx, cancel := context.WithTimeout(context.Background(), time.Minute) // 1 分钟超时
        defer cancel()
        periodic_scanner.ShutdownPeriodicScanner()  // 停 ticker
        scanner_queue.CloseScannerQueue()        // 关队列
        svr.Shutdown(ctx)                        // 关 HTTP
    }()
}
```

优雅关闭时：
1. 停 periodic scanner 的 ticker 会 `Stop()`
2. scanner queue 会等所有 in-progress 任务完成（但 ffmpeg 子进程不会被杀
3. 1 分钟后强制退出

**暴力关闭**（SIGKILL / kill -9 / 断电）：
- 什么清理都没有，直接退出
- 正在运行的 ffmpeg 子进程变成孤儿，被 init 进程接管继续跑

---

### 17.2 僵尸 ffmpeg 进程检测

**结论：完全没有检测和清理机制。进程崩溃或被强制退出时，正在运行的 ffmpeg 子进程会变成孤儿继续跑。**

#### 子进程生命周期

```
正常情况：
  Go 进程
    ├─ exec.Command("ffmpeg", ...)
    │   └─ ffmpeg 子进程
    └─ cmd.Run() 等待子进程退出

异常情况（Go 进程被 kill -9）：
  init 进程 (PID 1)
    └─ ffmpeg 子进程 (变成孤儿，继续跑直到完成）
```

#### 哪些场景会产生僵尸/孤儿进程

| 场景 | 是否产生孤儿进程 |
|------|------------|
| 正常转码完成 | ❌ 不会，cmd.Wait() 正常回收 |
| 优雅关闭 (SIGTERM) | ❌ 不会，等 1 分钟让任务完成 |
| 优雅关闭但 1 分钟内没跑完 | ✅ 会，1 分钟超时后主进程退出，ffmpeg 变成孤儿 |
| 强制 kill -9 | ✅ 会，直接退出，子进程没人管 |
| 容器 OOM kill | ✅ 会 |
| 断电 | ✅ 会 |

#### 为什么不会变僵尸 vs 孤儿

注意区分两个概念：
- **僵尸进程**（zombie）：子进程已退出，但父进程没 `wait`，PID 还占着
- **孤儿进程**（orphan）：父进程死了，子进程还在跑，被 init 接管

Photoview 的情况是**孤儿进程**（ffmpeg 还在继续转码），不是僵尸进程。

#### 检测方法（当前完全没做

检测孤儿 ffmpeg 进程的方法：

1. **PID 文件记录法：启动 ffmpeg 后把 PID 写到文件里，启动时检查 PID 是否还活着
2. **进程名匹配法：启动时遍历 `/proc` 找 `ffmpeg -i ... photoview_cache/...` 模式的进程
3. **cgroup 法：（Docker 环境）把所有子进程放到同一个 cgroup，退出时整个 cgroup 杀

Photoview 一种都没做。

#### 对比：exiftool 有长驻进程管理

**文件**：`api/scanner/externaltools/exiftool/exiftool.go:50`

```go
cmd := exec.Command(path, "-stay_open", "True", "-@", "-")
// 长驻进程，通过 stdin/stdout 通信
```

exiftool 是一个长驻进程反复用，有 `Close()` 方法优雅关闭。但 ffmpeg 是每次启动一个新进程，用完就丢。

---

### 17.3 运行时 .cache 增量清理

**结论：没有增量清理。只有全量清理（删整个 mediaID 目录）有，增量清理（删目录内的孤儿文件）没有。**

#### 现有的清理逻辑

**清理 1：源文件删除后清理整个缓存目录**

**文件**：`api/scanner/scanner_tasks/cleanup_tasks/cleanup_media.go:17-65`

```go
func CleanupMedia(db *gorm.DB, albumId int, albumMedia []*models.Media) []error {
    // 找出 DB 中有但磁盘上没有的媒体
    query := db.Where("album_id = ?", albumId)
    if len(albumMedia) > 0 {
        query = query.Where("NOT id IN (?)", albumMediaIds)
    }
    query.Find(&mediaList)

    for _, media := range mediaList {
        // 整个缓存目录删掉
        cachePath := path.Join(utils.MediaCachePath(), strconv.Itoa(albumId), strconv.Itoa(media.ID))
        os.RemoveAll(cachePath)  // ← 整个目录全删
    }

    // DB 里的媒体记录也删掉
    db.Where("id IN (?)", mediaIDs).Delete(models.Media{})
}
```

触发时机：每次相册扫描完后，`MediaCleanupTask.AfterScanAlbum()` 调用。

**清理 2：相册删除后清理整个相册缓存目录**

**文件**：`api/scanner/scanner_tasks/cleanup_tasks/cleanup_media.go:67-136`

```go
func DeleteOldUserAlbums(...) []error {
    // 找出 DB 中有但磁盘上没有的相册
    // 每个相册的缓存目录整个删掉
    cachePath := path.Join(utils.MediaCachePath(), strconv.Itoa(album.ID))
    os.RemoveAll(cachePath)
}
```

#### 没有的增量清理

- ❌ 没有"DB 有 MediaURL 记录但磁盘文件丢失 → 这种情况视频缩略图有磁盘检查并重生成（见 13.3），但不会清理其他孤儿文件

- ❌ 没有"磁盘有文件但 DB 中没有对应 MediaURL 记录 → 孤儿文件永远占空间

- ❌ 没有"旧格式缓存文件清理（比如之前版本生成的旧格式文件名）

- ❌ 没有按访问时间清理（LRU）

- ❌ 没有按大小限制清理（超过多少 G）

#### 缓存目录结构

```
media_cache/
  └─ {albumID}/
      └─ {mediaID}/
          ├─ web_video_movie_abc123.mp4    ← Purpose: video-web
          ├─ video_thumb_movie_def456.jpg    ← Purpose: video-thumbnail
          ├─ web_video_movie_old789.mp4     ← 孤儿！DB 里没记录（转码失败留下的，永远不会被清
          └─ thumbnail.jpg                   ← Purpose: photo-thumbnail（图片的）
```

孤儿文件产生的原因：
1. 转码到一半失败/中断 → 文件名带随机 token，下次重转用新 token，旧的变孤儿
2. 旧版本软件生成的旧格式文件名
3. 手动改了 purpose 配置变了，旧文件还在

---

### 17.4 SHA256 streaming 并发限制

**结论：完全没有 SHA256，也没有 streaming 哈希，也没有并发限制。**

#### 现有的哈希使用情况

| 用途 | 算法 | 数据量 | 是否 streaming | 是否有并发限制 |
|------|------|--------|----------------|--------------|
| 路径去重 | MD5 | 路径字符串 (~100B) | ❌ 直接 `md5.Sum()` 一次性 | - |
| Sidecar 变更检测 | MD5 | XMP 文件 (<1MB) | ❌ `io.Copy(h, f) 流式，但文件小 | ❌ 没有，全局一把锁串行（`globalMu` 互斥） |
| 视频内容哈希 | - | - | - | - |
| 图片内容哈希 | - | - | - | - |

#### Sidecar 的 MD5 是流式的

**文件**：`api/scanner/scanner_tasks/processing_tasks/sidecar_task.go:151-158`

```go
f, _ := os.Open(sidecarPath)
defer f.Close()
h := md5.New()
if _, err := io.Copy(h, f); err != nil {  // ← io.Copy 是流式的，边读边算
    log.Printf("ERROR: %s", err)
}
hash := hex.EncodeToString(h.Sum(nil))
```

但 sidecar 文件很小（XMP 一般几十 KB 到几 MB，流不流式无所谓。

#### 为什么没有视频内容哈希

设计取舍：
1. 视频文件大（几 GB 到几十 GB）
2. 读一遍算哈希很慢（磁盘 IO 瓶颈
3. 每次扫描都算的话，扫描速度会从 O(N) 变成 O(N * 文件大小)
4. Photoview 定位是"只读浏览媒体库"，假设源文件不会变

如果真要做的话，折中方案是"文件大小 + mtime"快速校验，只有大小和 mtime 都没变就认为没改了，才去算内容哈希。

#### 并发限制

也没有并发限制。FFmpeg 转码的并发由 scanner queue 的 `max_concurrent_tasks` 控制，但那是**任务级**的并发控制，不是**哈希计算**的并发控制。

---

### 17.5 Intel QSV 硬件加速

**结论：支持，但只是简单地换了个编码器名，没有任何 QSV 特有的优化（如帧格式转换、设备管理）都没有。

#### QSV 支持现状

**文件**：`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:13-19`

```go
var hwAccToCodec = map[string]string{
    "qsv":   defaultCodec + "_qsv",     // h264_qsv
    ...
}
```

就这么简单。设置为 "qsv" → 编码器名换成 `h264_qsv`。

#### QSV 完整用法的标准做法

通常 ffmpeg QSV 完整的硬件加速流水线：

```bash
# 最简单的用法（就是 Photoview 现在的做法）：
ffmpeg -i input.mp4 -c:v h264_qsv output.mp4
# 这种方式：解码器还是软解 → CPU 解码 → GPU 编码

# 完整硬件流水线（性能更好）：
ffmpeg -hwaccel qsv -hwaccel_output_format qsv -i input.mp4 \
  -c:v h264_qsv output.mp4
# 这种方式：GPU 解码 → GPU 编码 → 零拷贝
```

Photoview 的做法是第一种：
- ✅ 编码用硬件（h264_qsv）
- ❌ 解码还是软件（默认）
- ❌ 没有 `-hwaccel qsv` 硬件解码
- ❌ 没有 `-hwaccel_output_format qsv` 帧格式
- ❌ 没有设备选择（`-qsv_device`）

相当于 GPU 和 CPU 之间还是要做一次内存拷贝，性能不如全硬件流水线快。

#### QSV 的其他限制

- **Linux**：
  - Intel Gen 5+ 的核显
  - 驱动：`intel-media-va-driver / intel-media-driver

- **Windows**：也支持
  用 Intel Media SDK / oneVPL

- **macOS**：不支持 QSV（macOS 用 VideoToolbox）

#### 可用性检测

**没有可用性检测**。设置了 `qsv` 但机器没有 Intel 核显 / 没装驱动，ffmpeg 直接报错，转码失败。不会 fallback 软编。

对比一下 exiftool 和 ffprobe 在初始化时会检测：

```go
// exiftool 初始化检测
path, err := exec.LookPath("exiftool")
if err != nil {
    return nil, err  // 找不到就返回错误
}
```

但 ffmpeg QSV 没有 "编码器不可用 → 转码时才知道。

---

### 17.6 运行时死锁检测

**结论：完全没有。没有使用任何死锁检测库，也没有 watchdog 监控。

#### 依赖中没有死锁检测库

`go.mod` 里搜索不到 `go-deadlock` 之类的：

```go
// go.mod 里没有任何死锁检测相关
```

Go 标准也没有死锁检测

Go 运行时只有：

- `-race` 竞态检测（测试用
- 没有内置死锁检测器

#### 可能的死锁点（已在 15.6 分析过）

1. **`notificationLock` 阻塞式发送**

最可能死锁：
```go
notificationLock.Lock()
defer notificationLock.Unlock()
for _, listener := range notificationListeners {
    listener.channel <- notification  // ← 这里可能永远阻塞
}
```

某个 listener 的 channel 满了，发送阻塞，拿着全局大锁不释放，其他人都拿不到锁。

2. **锁顺序不一致**（目前没问题，但未来有风险）

当前调用链：
```
scanner_queue.mutex → ScanAlbum → notificationLock
```

如果将来如果有通知回调里操作扫描队列，就会死锁

3. **数据库事务 + 锁**

DB 事务和 Go mutex 混合使用的场景也要小心顺序。

#### 测试里的死锁检测

测试里有提到 "Test passes if no deadlock occurs" 这种注释：

**文件**：`api/scanner/periodic_scanner/periodic_scanner_test.go:262

```go
// Test passes if no deadlock occurs
```

但这只是"跑一下看看会不会卡死，没有自动检测。

#### 怎么检测死锁

常用的 Go 死锁检测方案：

1. **`go-deadlock`**：替换 `sync.Mutex` 为 `deadlock.Mutex`，运行时检测死锁
2. **`goleak`**：检测 goroutine 泄漏（测试用
3. **pprof**：`http://localhost:6060/debug/pprof/goroutine?debug=2 看 goroutine 栈

Photoview 一种都没集成。

---

### 17.7 SMB 文件系统兼容

**结论：和 NFS 一样，没有针对 SMB/CIFS 的特殊处理。

#### SMB vs NFS 共性问题

| 问题 | SMB 影响 | Photoview 是否处理 |
|------|--------|-------------------|
| 缓存一致性 | 刚写的文件可能读不到（客户端缓存） | ❌ 没有 |
| `os.Rename 跨设备失败 | rename 不同挂载点之间 rename 失败 | ❌ 没有 |
| 文件锁语义 | SMB 的文件锁和本地不一样 | ❌ 没有用文件锁 |
| mtime 精度 | 可能只有秒级 | ❌ 不依赖 mtime |
| 断开重连 | 网络闪断可能导致 IO 错误 | ❌ 没有重试 |

#### SMB 特有的问题

1. **SMB1/SMB2/SMB3 协议差异

不同版本协议的行为不一样。

2. **文件名大小写**

Windows SMB 服务器大小写不敏感的，Linux 客户端大小写敏感。如果源文件在 SMB 共享上，可能出现 `Photo.jpg 和 `photo.jpg 被当成同一个文件？

Photoview 的 `path_hash` 是区分大小写的（MD5 区分大小写），所以会当成不同的文件。

3. **权限模型**

Windows 的权限模型和 Unix 不一样，`os.Chmod` 可能不支持。

Photoview 不 `os.Chmod（没用到。

#### 实际影响最大的问题

对视频转码的影响

```
源视频在 SMB 共享上：
  ├─ 读源文件 → SMB 读 → ffmpeg 解码
     ├─ 网络闪断 → ffmpeg 读到一半失败
     └─ 转码失败

缓存输出在 SMB 共享上：
  ├─ ffmpeg 写 → SMB 写
     ├─ 网络闪断 → 写失败，写了一半的文件
     └─ 下次扫描重新转码（因为 DB 没记录

和 NFS 类似。

#### 对比：符号链接 SMB 的处理

有通用符号链接处理了（`IsDirSymlink`），但那是通用的，不是 SMB 特有的。

---

### 17.8 WebSocket 断网长时间后状态恢复

**结论：服务端完全无状态保持，断多久都直接删 listener，重连重新注册，期间的通知全丢。

#### 当前实现

**文件**：`api/graphql/resolvers/notification.go:18-34

```go
func (r *subscriptionResolver) Notification(ctx context.Context) (<-chan *models.Notification, error) {
    user := auth.UserFromContext(ctx)
    ...
    notificationChannel := make(chan *models.Notification, 1)  // 缓冲 1
    listenerID := notification.RegisterListener(user, notificationChannel)

    go func() {
        <-ctx.Done()                     // 连接断开
        notification.DeregisterListener(listenerID)  // 直接注销
    }()

    return notificationChannel, nil
}
```

#### 断网场景分析

**短时间断网（几秒）：

```
前端断网 → ping 超时 → 检测到断开 → ctx.Done() → DeregisterListener
  ↓
前端重连 → 新的 listener 重新注册 → 新 channel

断期间的通知：全丢了
```

**长时间断网（几分钟/小时）：

一样的，只是断的时间更长，丢的通知更多。

#### 没有的状态

- ：

1. **消息队列（message queue）：把消息存起来，重连后补发
2. **序列号**：每条通知有序号，重连时说"我上次收到 N 号，从 N+1 开始发
3. **状态同步**：重连后拉取完整状态对比 diff

一种都没做。

#### gqlgen 层的保活

**文件**：`api/graphql/endpoint/graphql_endpoint.go:31-35`

```go
graphqlServer.AddTransport(transport.Websocket{
    KeepAlivePingInterval: 10 * time.Second,  // 每 10 秒发 ping
    ...
})
```

gqlgen 内置的 websocket transport 每 10 秒发 ping，客户端要回 pong。不回就认为断开。

所以断网后最多 10 秒左右就会检测到断开。

#### 视频转码进度的丢失场景

一个视频转码需要 5 分钟：

```
0: 开始转码
1: 转码 20% → 通知 20
2: 断网了
3: 转码 50% → 通知发不出去（阻塞或者丢了
4: 转码 100% → 通知也丢了
5: 用户重连 → 啥通知一个通知都没收到，以为还没开始转
```

但实际上转码完了，DB 里有记录，刷新页面就能看到。

---

## 十八、已知问题与改进方向（完整汇总）

基于以上所有分析，视频转码管线目前存在以下可改进点，按优先级排序：

### 高优先级

1. **FFmpeg 子进程无法被中断**（13.2 / 15.2 / 17.2）：改用 `exec.CommandContext` 传入 context + 超时，让取消信号能真正杀掉子进程，防止 goroutine 和子进程泄漏
2. **通知系统阻塞式发送死锁**（15.6 / 17.6）：`BroadcastNotification` 中 `listener.channel <-` 是阻塞发送，拿着全局大锁遍历所有 listener，单个慢消费者会挂死整个通知系统
3. **孤儿缓存文件**（13.3 / 17.3）：转码失败/中断/断电时，带随机 token 的不完整文件不会被清理，永久占用磁盘空间
4. **并发转码重复劳动**（13.6）：后台扫描 + HTTP 请求补转码可能同时跑两个 ffmpeg 转同一个视频，浪费资源

### 中优先级

5. **缺少部分写入保护**（13.3）：视频转码可以参考 sidecar 的做法，先写 `.tmp` 文件，成功后 rename，避免断电留下残次文件
6. **文件内容变更检测**（13.4 / 15.4）：仅靠 `path_hash` 无法检测文件替换，可以加入 `文件大小 + mtime` 的快速校验
7. **磁盘空间检查**（13.7）：转码前预估所需空间，不足时告警或降级，避免一个个文件失败
8. **硬件加速可用性探测**（13.5 / 15.5 / 17.5）：设置了硬件加速但不可用时，自动 fallback 到软编码，而不是直接报错
9. **重启后立即触发首次扫描**（17.1）：启动时检查配置的间隔，而不是等一整 interval 才第一次扫
10. **僵尸 ffmpeg 进程清理**（17.2）：启动时检测并清理上一次运行留下的孤儿 ffmpeg 进程
11. **增量缓存清理**（17.3）：定期扫描缓存目录，清理 DB 中没有对应 MediaURL 记录的孤儿文件

### 低优先级

12. **FFmpeg 转码超时**（15.2）：给 ffmpeg 转码加一个合理的超时（如 4 小时），防止损坏视频导致永久挂起
13. **断线通知补发**（15.8 / 17.8）：WebSocket 重连后丢失的通知可以通过序列号机制补发
14. **正式支持 AMD AMF**（15.5）：在 `hwAccToCodec` map 中增加 `"amf": "h264_amf"`
15. **时钟回拨保护**（15.1）：如果未来要做基于时间窗的增量扫描，需要考虑时钟回拨问题
16. **NFS / SMB 兼容性**（15.7 / 17.7）：对于 `ESTALE` 等网络文件系统特有错误增加重试逻辑
17. **Mutex 细粒度化**（15.6）：全局大锁改为按用户/按相册分片，减少锁竞争
18. **QSV 全硬件流水线**（17.5）：增加 `-hwaccel qsv -hwaccel_output_format qsv` 参数，实现零拷贝全硬件编解码
19. **运行时死锁检测**（17.6）：集成 go-deadlock 或 pprof，开发环境启用
20. **SHA256 内容哈希**（15.4 / 17.4）：如果需要检测文件内容变更，加入 SHA256 内容哈希（配合大小+mtime

---

## 十九、深度细节分析（四）

### 19.1 时区切换下时钟兜底

**结论：EXIF 时间解析存在时区丢失问题，但服务端时间逻辑不受时区切换影响。**

#### EXIF 时间解析

**文件**：`api/scanner/externaltools/exiftool/values.go:63-95`

```go
const layout = "2006:01:02 15:04:05.999"
const layoutWithTimezone = "2006:01:02 15:04:05.999Z07:00"

func (t TimeAll) TimeInLocal() time.Time {
    for _, dateP := range []*string{
        t.SubSecDateTimeOriginal,
        t.SubSecCreateDate,
        t.DateTimeOriginal,
        t.CreateDate,
        t.TrackCreateDate,
        t.MediaCreateDate,
        t.FileModifyDate,
    } {
        if dateP == nil {
            continue
        }

        date := *dateP

        // Ignore timezone  ← 注意这行：主动丢弃时区信息
        if zoneIndex := strings.IndexAny(date, "+-Z"); zoneIndex >= 0 {
            date = date[:zoneIndex]
        }

        if date, err := time.ParseInLocation(layout, date, time.UTC); err == nil {
            return date  // ← 按 UTC 解析，丢掉了原始时区
        }
    }

    return time.Time{}
}
```

**关键发现**：
1. `TimeInLocal()` 主动丢弃 EXIF 中的时区信息（`+08:00`、`Z` 等）
2. 然后按 UTC 解析剩余的时间字符串
3. 这意味着一个在北京时间 14:00 拍的照片，EXIF 记录 `2025:01:01 14:00:00+08:00`
4. 丢弃时区后变成 `2025:01:01 14:00:00`，按 UTC 解析，实际是 UTC 14:00
5. 但真实 UTC 时间应该是 06:00（14:00 - 8h）
6. **结果：照片拍摄时间被记录为比实际 UTC 时间早了 8 小时**

#### OffsetSecs 的补救

**文件**：`api/scanner/externaltools/exiftool/values.go:98-137`

```go
func (t TimeAll) OffsetSecs(local time.Time) (int, bool) {
    // 1. 优先用 OffsetTimeOriginal / OffsetTime（EXIF 2.3.1+ 的标准时区标签）
    for _, offsetP := range []*string{
        t.OffsetTimeOriginal,
        t.OffsetTime,
    } {
        if offsetP != nil {
            if t, err := time.Parse("-07:00", *offsetP); err == nil {
                _, offsetSecs := t.Zone()
                return offsetSecs, true
            }
        }
    }

    // 2. 其次用 TimeZone 字段（单位：分钟）
    if t.TimeZone != nil {
        return *t.TimeZone * 60, true
    }

    // 3. 最后用 GPS 时间推算偏移
    if t.GPSDateTime != nil {
        gpsDate, _ := time.Parse(layoutWithTimezone, *t.GPSDateTime)
        gpsDate = gpsDate.UTC()
        offset := int(local.Sub(gpsDate).Seconds())
        return offset, true
    }

    return 0, false
}
```

三层时区回退策略：EXIF 时区标签 → TimeZone 字段 → GPS 时间推算。

#### 时区切换的影响

| 场景 | 影响 |
|------|------|
| 服务器时区从 UTC 切换到 UTC+8 | `TimeInLocal()` 不受影响（硬编码 UTC），但 `OffsetSecs()` 的 `local.Sub(gpsDate)` 结果不变（两个 `time.Time` 比较不受时区影响） |
| Docker 容器默认 UTC | 与代码假设一致，没问题 |
| EXIF 没有时区信息且没 GPS | `OffsetSecs` 返回 `(0, false)`，前端无法知道真实时区，拍摄时间可能是"当地时间当 UTC" |
| 系统时间被改了时区 | `time.Ticker` 基于 monotonic clock 不受影响（见 17.1） |

#### 兜底缺失

- ❌ 没有"检测到时区偏移异常时发出警告"的逻辑
- ❌ 没有"服务器时区与 EXIF 时区不一致时提示用户"的机制
- ❌ `OffsetSecs` 返回 `(0, false)` 时，前端只是不显示时区，不会报错

---

### 19.2 快速 fork-exec 风暴的 reaping 策略

**结论：没有 reaping 策略。大量视频同时转码时，短时间内会 fork 大量子进程，全靠操作系统调度。**

#### 子进程创建模式

Photoview 中有三类外部进程：

| 进程 | 创建方式 | 生命周期 | 并发控制 |
|------|----------|----------|----------|
| **ffmpeg** (转码) | `exec.Command().Run()` | 一次性，用完退出 | 受 `max_concurrent_tasks` 间接控制 |
| **ffmpeg** (截帧) | `exec.Command().Run()` | 一次性，用完退出 | 同上 |
| **exiftool** | `exec.Command().Start()` | 长驻进程，stdin/stdout 通信 | 单例，不支持并发（代码注释明确说了） |

**文件**：`api/scanner/externaltools/exiftool/exiftool.go:15`

```go
// Exiftool launches an external `exiftool` process to query photos' exif info.
// It doesn't support concurrency usage.  ← 不支持并发
type Exiftool struct { ... }
```

#### fork-exec 风暴场景

`max_concurrent_tasks` 默认值是可配置的（从 DB `site_info.ConcurrentWorkers` 读），最小为 1，SQLite 下强制为 1：

**文件**：`api/graphql/resolvers/scanner.go:83-89`

```go
func (r *mutationResolver) SetScannerConcurrentWorkers(ctx context.Context, workers int) (int, error) {
    if workers < 1 {
        return 0, errors.New("concurrent workers must at least be 1")
    }
    if workers > 1 && drivers.DatabaseDriverFromEnv() == drivers.SQLITE {
        return 0, errors.New("multiple workers not supported for SQLite databases")
    }
    ...
}
```

假设设为 4 个 worker，每个相册里 100 个视频，每个视频要调 2 次 ffmpeg（1 次转码 + 1 次截帧）：

```
4 workers × 每个视频 2 次 ffmpeg = 最多 8 个 ffmpeg 子进程同时存在
```

**不算风暴**，但问题是：

1. **每个 worker 串行处理**：一个 worker 处理完一个视频（转码 + 截帧）才处理下一个，不是一次性 fork 一堆
2. **没有 batch 优化**：每次 ffmpeg 都是独立进程，没有 ffmpeg 滤镜图合并多个输入
3. **exiftool 是瓶颈**：它是单例长驻进程且不支持并发，如果每个媒体都要 exiftool 查 EXIF，它会是串行瓶颈

#### 没有 reaping 的后果

```
进程 A 正在转码（ffmpeg 子进程 PID 12345）
进程 A 又启动了截帧（ffmpeg 子进程 PID 12346）

如果进程 A 被杀：
  ├─ PID 12345 变成孤儿 → init 接管 → 继续跑 → 最终退出 → init 回收
  └─ PID 12346 变成孤儿 → 同上

没有 SIGCHLD 处理，没有 waitpid 循环。
Go 的 os/exec 包内部会调用 cmd.Wait() 回收子进程，但前提是父进程还活着。
```

Go 的 `os/exec` 包内部已经处理了子进程回收（通过 `cmd.Wait()` 调用 `wait4`/`waitid`），不会产生僵尸进程——前提是父进程调了 `Wait()`。当前代码用的是 `cmd.Run()` = `Start()` + `Wait()`，正常情况下不会有僵尸。

---

### 19.3 cgroup 隔离下 cache 清理

**结论：Dockerfile 没有配置 cgroup 感知的缓存清理，容器被 OOM kill 时缓存目录不会被自动清理。**

#### Dockerfile 分析

**文件**：`Dockerfile:155-177`

```dockerfile
ENV PHOTOVIEW_MEDIA_CACHE=/home/photoview/media-cache
...
USER photoview
ENTRYPOINT ["/app/photoview"]
```

缓存目录在 `/home/photoview/media-cache`，容器内。

#### cgroup 相关的缺失

| 功能 | 是否有 | 说明 |
|------|--------|------|
| 缓存大小限制 | ❌ | 没有配置 `memory.limit_in_bytes` 或 cgroup v2 等效 |
| OOM 时清理缓存 | ❌ | 没有 OOM handler，被 kill 就退出了 |
| 磁盘配额 | ❌ | 没有配置 cgroup v2 的 `io.max` 或磁盘配额 |
| 优雅退出时清理 | ❌ | `SIGTERM` 时只等 1 分钟就退出，不清理缓存 |
| 缓存目录 tmpfs | ❌ | 没有把缓存放在 tmpfs（容器重启自动清空） |
| 缓存目录 volume | ✅ | 可以用 Docker volume 挂载，但用户要自己配 |

#### 容器重启的缓存状态

```
容器重启后：
  ├─ 如果 PHOTOVIEW_MEDIA_CACHE 是 volume mount → 缓存还在
  │   └─ DB 还在 → MediaURL 记录还在 → 缓存命中 ✅
  │
  ├─ 如果 PHOTOVIEW_MEDIA_CACHE 是容器内路径 → 缓存丢了
  │   └─ DB 还在 → MediaURL 记录还在 → 磁盘文件没了 → 404
  │       └─ 但 process_video_task.go:174 有重生成逻辑：
  │           if _, err := os.Stat(thumbImagePath); os.IsNotExist(err) {
  │               // 重新生成缩略图
  │           }
  │       └─ 注意：这只检查了缩略图，没检查转码后的视频文件
  │
  └─ 如果 DB 也丢了（SQLite 在容器内） → 全部重来
```

#### 缓存文件磁盘泄漏

在 cgroup 隔离下，如果容器磁盘配额有限（如 10G），缓存不断增长最终会满：

1. 转码产生的 `.mp4` 文件（每个可能几百 MB）
2. 孤儿缓存文件（转码失败/中断后没人清理）
3. 没有基于大小的 LRU 清理

容器被 OOM kill 或磁盘满了，只是杀掉进程，不清理缓存。下次启动又继续往满了的磁盘写。

---

### 19.4 semaphore 优先级反转

**结论：没有 semaphore，用的是 Mutex + channel。没有优先级概念，不存在优先级反转问题，但存在"低优先级任务阻塞高优先级任务"的等价问题。**

#### 并发控制机制

Photoview 使用的并发控制：

| 机制 | 位置 | 保护对象 |
|------|------|----------|
| `sync.Mutex` | `scanner_queue/queue.go:48` | 任务队列 |
| `sync.Mutex` | `notification/Notification.go:30` | 通知 listener 列表 |
| `sync.Mutex` | `periodic_scanner.go:34` | 全局 scanner 单例 |
| `chan bool (buffer 1)` | `scanner_queue/queue.go:49` | 通知 worker 有活干 |
| `make(chan *models.Notification, 1)` | `resolvers/notification.go:27` | 通知 channel |
| DB `ConcurrentWorkers` | `queue.go:68` | 最大并发任务数 |

#### "优先级反转"的等价问题

虽然没有严格意义的优先级反转（没有优先级概念），但存在**功能等价的问题**：

**问题 1：全局队列阻塞**

```
用户 A 有 1000 个视频（低优先级：大相册慢扫）
用户 B 有 10 个视频（高优先级：小相册快扫）

队列是全局的，先来先服务：
  ├─ 用户 A 的 4 个 worker 全占满（每个跑 1 小时）
  └─ 用户 B 的任务排在 up_next 里等（可能等 4 小时）

没有"小任务优先"或"用户公平调度"的机制。
```

**问题 2：exiftool 串行瓶颈**

```go
// It doesn't support concurrency usage.
type Exiftool struct { ... }
```

exiftool 是单例，所有 worker 都要排队用。一个慢查询会阻塞其他 worker：

```
Worker 1: exiftool 查询中（大图，慢）← 持有 exiftool
Worker 2: 等 exiftool...               ← 被阻塞
Worker 3: 等 exiftool...               ← 被阻塞
Worker 4: 等 exiftool...               ← 被阻塞
```

这就是典型的"优先级反转"——Worker 1 在做低价值的 EXIF 查询，Worker 2/3/4 可能已经拿到了 EXIF 数据只想做转码，但因为 exiftool 不可用而排队。

**问题 3：`notificationLock` 全局阻塞**（已在 15.6 分析）

低优先级的"扫描进度通知"发送时拿着全局锁，会阻塞高优先级的"注册新 listener"操作。

---

### 19.5 QSV 回退 CPU 路径

**结论：没有回退。QSV 不可用时直接报错退出，不会 fallback 到 h264 软编码。**

#### 错误传播路径

```
1. ffmpeg -c:v h264_qsv 启动
2. QSV 不可用（没有核显/没装驱动/在容器里没映射 /dev/dri）
3. ffmpeg 输出错误到 stderr，返回非零退出码
4. cmd.Run() 返回 error
5. EncodeMp4() 返回 fmt.Errorf("encoding video ... error: %w", err)
6. ProcessMedia() 返回错误
7. scanMedia() 记录日志，继续处理下一个文件
8. 下一个文件也用 h264_qsv，也失败，也跳过
9. 所有视频全部失败
```

#### 没有回退的代码证据

**文件**：`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:74-96`

```go
func (cli *FfmpegCli) EncodeMp4(inputPath string, outputPath string) error {
    if cli.err != nil {
        return fmt.Errorf("encoding video %q error: ffmpeg: %w", inputPath, cli.err)
    }

    args := []string{
        "-i", inputPath,
        "-vcodec", cli.videoCodec,  // ← 固定用初始化时选的编码器
        "-acodec", "aac",
        ...
    }

    cmd := exec.Command(cli.path, args...)

    if err := cmd.Run(); err != nil {
        return fmt.Errorf("encoding video with %q %v error: %w", cli.path, args, err)
        // ← 直接返回错误，没有尝试用 defaultCodec 重试
    }

    return nil
}
```

`cli.videoCodec` 在初始化时就固定了，运行时不改变，失败也不回退。

#### 对比：MagickWand 的 IsInstalled 模式

**文件**：`api/scanner/media_encoding/executable_worker/magickwand.go:31-32`

```go
func (cli *MagickWand) IsInstalled() bool {
    return cli != nil && cli.initialized
}
```

MagickWand 有 `IsInstalled()` 检查，不可用时可以走 ffmpeg 的 JPEG 编码作为 fallback。但 ffmpeg 没有"编码器不可用"的 fallback。

#### 改进方案

```go
func (cli *FfmpegCli) EncodeMp4(inputPath string, outputPath string) error {
    if cli.err != nil {
        return fmt.Errorf("encoding video %q error: ffmpeg: %w", inputPath, cli.err)
    }

    err := cli.encodeWithCodec(cli.videoCodec, inputPath, outputPath)
    if err != nil && cli.videoCodec != defaultCodec {
        log.Warn(nil, "Hardware encoder failed, falling back to software encoding",
            "failed_codec", cli.videoCodec, "fallback_codec", defaultCodec)
        err = cli.encodeWithCodec(defaultCodec, inputPath, outputPath)
    }

    return err
}
```

#### Docker 容器中的 QSV 问题

**文件**：`Dockerfile:176`

```dockerfile
USER photoview
ENTRYPOINT ["/app/photoview"]
```

Docker 容器默认不映射 `/dev/dri`（Intel GPU 设备），所以容器内 QSV 不可用。要启用需要：

```bash
docker run --device /dev/dri:/dev/dri ...
```

但如果设了 `PHOTOVIEW_VIDEO_HARDWARE_ACCELERATION=qsv` 又没映射设备，所有视频转码都会失败，且不回退。

---

### 19.6 TSAN 生产开销

**结论：没有在生产环境启用 TSAN/race detector。`go.mod` 和 Dockerfile 中都没有 `-race` 编译标志。**

#### 搜索结果

在 `go.mod`、`Dockerfile`、构建脚本中搜索 `-race`、`TSAN`、`thread sanitizer`：

- `go.mod`：无
- `Dockerfile`：`go build -v -o photoview .`（没有 `-race`）
- 测试代码：无 `-race` 标志

**文件**：`Dockerfile:101-103`

```dockerfile
RUN set -a && source /env && set +a \
    && go env \
    && go build -v -o photoview .
# ↑ 没有 -race 标志
```

#### TSAN 的开销

| 指标 | 正常编译 | `-race` 编译 |
|------|----------|-------------|
| 内存开销 | 1x | 5-10x |
| CPU 开销 | 1x | 5-20x |
| 二进制大小 | ~30MB | ~100MB+ |
| 执行速度 | 正常 | 慢 5-20 倍 |

**绝对不能在生产开启**。TSAN 只在开发和测试时使用。

#### 已知的竞态风险

虽然没开 TSAN，但从代码结构可以推断出竞态风险点：

1. **`notificationListeners` slice 的 append**：`RegisterListener` 在锁内 append，`DeregisterListener` 在锁内删除，`BroadcastNotification` 在锁内遍历——这三个操作都在 `notificationLock` 保护下，**没有竞态**，但有死锁风险

2. **`global_scanner_queue` 的 goroutine**：worker goroutine 修改 `in_progress` slice 在 `queue.mutex` 保护下，**没有竞态**

3. **`AlbumScannerCache` 的本地 map**：每个相册扫描任务一个实例，单 goroutine 使用，**没有竞态**

4. **DB 操作**：GORM 本身是并发安全的（驱动层有锁），**没有竞态**

5. **潜在的竞态**：`nextNotificationId` 全局变量在 `RegisterListener` 中递增，但在 `notificationLock` 保护下，**没有竞态**

> 结论：Photoview 的并发模型相对简单（全局大锁），竞态风险低，但死锁风险高。开 `-race` 跑测试是个好实践，但当前没做。

---

### 19.7 SMB 锁升级

**结论：Photoview 不使用文件锁，所以不存在 SMB 锁升级问题。但 SMB 的 byte-range lock 语义差异可能在 ffmpeg 层面造成影响。**

#### Photoview 使用的"锁"

| 锁类型 | 位置 | 作用 |
|--------|------|------|
| Go `sync.Mutex` | 内存中 | 保护内存数据结构 |
| Go `sync.RWMutex` | 内存中 | 保护 `testCachePath` |
| GORM 事务 | 数据库 | 保护 DB 操作 |
| ❌ 文件锁 | 无 | 不使用 |

没有任何 `flock()`、`fcntl(F_SETLK)`、`LockFileEx()` 调用。

#### SMB 锁升级问题（背景知识）

SMB 的锁升级（lock upgrade）指从共享锁升级到排他锁。在 POSIX 系统上，`fcntl` 支持锁升级；在 SMB 协议上，锁升级行为取决于 SMB 版本和实现：

| SMB 版本 | 锁升级支持 | 说明 |
|----------|-----------|------|
| SMB1 | 有但行为怪异 | byte-range lock，升级可能死锁 |
| SMB2/3 | 有 | 更规范，但 Windows 和 Linux 客户端行为可能不同 |

**Photoview 不用文件锁，所以完全不受影响。**

#### ffmpeg 内部的文件锁

ffmpeg 在写输出文件时可能会对文件加锁（取决于输出格式和平台），但这是 ffmpeg 内部行为，Photoview 不控制。

如果 ffmpeg 的输出文件在 SMB 共享上，ffmpeg 内部的写操作可能受 SMB 锁语义影响，但这超出了 Photoview 的控制范围。

#### 可能的间接影响

如果多个 Photoview 实例（如 Docker 集群）同时写同一个 SMB 共享的缓存目录：

```
实例 A 转码视频 1 → 写 /smb_cache/1/web_video_xxx.mp4
实例 B 转码视频 2 → 写 /smb_cache/2/web_video_yyy.mp4
```

因为文件名带随机 token，不会写同一个文件。但 `os.MkdirAll` 创建目录时，SMB 的目录操作可能不是原子的。不过当前代码不检查 `os.MkdirAll` 的错误——如果目录已存在就忽略。

---

### 19.8 WebSocket session resume 历史消息回放

**结论：完全不支持。重连后是新 session，没有任何历史消息回放机制。**

#### 当前 session 生命周期

```
客户端连接
  ├─ gqlgen 升级 WebSocket
  ├─ AuthWebsocketInit() 验证 JWT
  ├─ 注册 NotificationListener（新 ID、新 channel）
  ├─ gqlgen 开始发 ping（10 秒间隔）
  │
  ├─ 正常运行：收到通知 → 通过 channel 发给客户端
  │
  ├─ 断开（网络/客户端关闭/服务器关闭）
  │   ├─ ctx.Done() 触发
  │   ├─ DeregisterListener() 删除 listener
  │   └─ channel 被 GC 回收
  │
  └─ 重连
      ├─ 新的 WebSocket 连接
      ├─ 新的 JWT 验证
      ├─ 新的 NotificationListener（新 ID、新 channel）
      └─ 只收到重连后的新通知
```

#### 缺失的组件

| 组件 | 作用 | 是否有 |
|------|------|--------|
| Session ID | 标识唯一会话 | ❌ |
| 消息序列号 | 标识消息顺序 | ❌ |
| 消息缓冲区 | 暂存最近 N 条消息 | ❌ |
| 重连握手 | 客户端说"我上次收到 seq=X" | ❌ |
| 历史回放 | 服务端从 seq+1 开始补发 | ❌ |
| Session 状态持久化 | 服务端记住断连前的状态 | ❌ |

#### 与视频转码的关联

视频转码的状态查询有两个路径：

1. **实时通知**（WebSocket subscription）：转码进度、完成通知——**断连即丢失**
2. **主动查询**（GraphQL query）：查 `MediaURL` 记录——**总是可用**

前端可以在重连后主动发一个 GraphQL query 来获取最新的转码状态，这是唯一的"回放"机制——但它不是自动的，需要前端自己实现。

#### 改进方案

如果要实现历史消息回放，最简单的方案：

```go
type NotificationWithSeq struct {
    Seq           int
    Notification  *models.Notification
}

var notificationHistory = make([]NotificationWithSeq, 0)
var lastSeq = 0

func BroadcastNotification(notification *models.Notification) {
    notificationLock.Lock()
    defer notificationLock.Unlock()

    lastSeq++
    entry := NotificationWithSeq{Seq: lastSeq, Notification: notification}
    notificationHistory = append(notificationHistory, entry)

    // 只保留最近 1000 条
    if len(notificationHistory) > 1000 {
        notificationHistory = notificationHistory[len(notificationHistory)-1000:]
    }

    for _, listener := range notificationListeners {
        select {
        case listener.channel <- notification:
        default:
            // 非阻塞发送，满了就跳过
        }
    }
}
```

然后在 `RegisterListener` 时接受 `lastSeenSeq` 参数，补发之后的消息。

---

## 二十、已知问题与改进方向（最终汇总）

基于以上所有分析，视频转码管线目前存在以下可改进点，按优先级排序：

### 高优先级

1. **FFmpeg 子进程无法被中断**（13.2 / 15.2 / 17.2）：改用 `exec.CommandContext` 传入 context + 超时，让取消信号能真正杀掉子进程，防止 goroutine 和子进程泄漏
2. **通知系统阻塞式发送死锁**（15.6 / 17.6 / 19.4）：`BroadcastNotification` 中 `listener.channel <-` 是阻塞发送，拿着全局大锁遍历所有 listener，单个慢消费者会挂死整个通知系统
3. **孤儿缓存文件**（13.3 / 17.3 / 19.3）：转码失败/中断/断电时，带随机 token 的不完整文件不会被清理，永久占用磁盘空间
4. **并发转码重复劳动**（13.6 / 19.4）：后台扫描 + HTTP 请求补转码可能同时跑两个 ffmpeg 转同一个视频，浪费资源
5. **QSV/NVENC/VAAPI 无回退**（13.5 / 17.5 / 19.5）：硬件编码器不可用时直接报错退出，应该自动 fallback 到 h264 软编码

### 中优先级

6. **缺少部分写入保护**（13.3）：视频转码可以参考 sidecar 的做法，先写 `.tmp` 文件，成功后 rename，避免断电留下残次文件
7. **文件内容变更检测**（13.4 / 15.4）：仅靠 `path_hash` 无法检测文件替换，可以加入文件大小 + mtime 快速校验
8. **磁盘空间检查**（13.7 / 19.3）：转码前预估所需空间，不足时告警或降级，避免一个个文件失败
9. **重启后立即触发首次扫描**（17.1）：启动时检查配置的间隔，而不是等一整 interval 才第一次扫
10. **僵尸 ffmpeg 进程清理**（17.2 / 19.2）：启动时检测并清理上一次运行留下的孤儿 ffmpeg 进程
11. **增量缓存清理**（17.3 / 19.3）：定期扫描缓存目录，清理 DB 中没有对应 MediaURL 记录的孤儿文件
12. **exiftool 并发瓶颈**（19.2 / 19.4）：单例长驻进程不支持并发，多 worker 时成为串行瓶颈，可以启动多个实例或用进程池
13. **EXIF 时区处理**（19.1）：`TimeInLocal()` 主动丢弃时区信息导致拍摄时间偏差，应保留时区或正确转换

### 低优先级

14. **FFmpeg 转码超时**（15.2 / 19.2）：给 ffmpeg 转码加一个合理的超时（如 4 小时），防止损坏视频导致永久挂起
15. **断线通知补发 / session resume**（15.8 / 17.8 / 19.8）：WebSocket 重连后丢失的通知可以通过序列号机制补发
16. **正式支持 AMD AMF**（15.5）：在 `hwAccToCodec` map 中增加 `"amf": "h264_amf"`
17. **时钟回拨保护**（15.1）：如果未来要做基于时间窗的增量扫描，需要考虑时钟回拨问题
18. **NFS / SMB 兼容性**（15.7 / 17.7 / 19.7）：对于 `ESTALE` 等网络文件系统特有错误增加重试逻辑
19. **Mutex 细粒度化**（15.6 / 19.4）：全局大锁改为按用户/按相册分片，减少锁竞争
20. **QSV 全硬件流水线**（17.5 / 19.5）：增加 `-hwaccel qsv -hwaccel_output_format qsv` 参数，实现零拷贝全硬件编解码
21. **运行时死锁检测**（17.6）：集成 go-deadlock 或 pprof，开发环境启用
22. **SHA256 内容哈希**（15.4 / 17.4）：如果需要检测文件内容变更，加入 SHA256 内容哈希（配合大小+mtime）
23. **TSAN 测试**（19.6）：CI 中加 `go test -race`，开发阶段捕获竞态条件
24. **Docker cgroup 缓存管理**（19.3）：配置缓存目录 size limit 或基于 LRU 的自动清理
25. **调度公平性**（19.4）：全局 FIFO 队列改为按用户公平调度，避免大相册饿死小相册



