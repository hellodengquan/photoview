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

## 十四、已知问题与改进方向

基于以上分析，视频转码管线目前存在以下可改进点：

1. **FFmpeg 子进程无法被中断**：改用 `exec.CommandContext` 传入 context，让取消信号能真正杀掉子进程
2. **缺少部分写入保护**：视频转码可以参考 sidecar 的做法，先写 `.tmp` 文件，成功后 rename，避免断电留下残次文件
3. **孤儿缓存文件**：转码失败或中断时，带随机 token 的不完整文件不会被清理，应在启动时或定期扫描清理
4. **文件内容变更检测**：仅靠 `path_hash` 无法检测文件替换，可以加入文件大小 + mtime 或内容 hash 校验
5. **并发转码重复劳动**：可以加一个媒体级的 in-flight map，同一个媒体同时只跑一次转码
6. **磁盘空间检查**：转码前预估所需空间，不足时告警或降级
7. **硬件加速可用性探测**：设置了硬件加速但不可用时，自动 fallback 到软编码
8. **孤儿缓存清理任务**：增加一个清理步骤，删除 DB 中没有对应 MediaURL 记录的缓存文件

