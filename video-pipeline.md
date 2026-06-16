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
