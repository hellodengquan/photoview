# 缩略图多分辨率生成与缓存流程

## 概述

Photoview 的缩略图系统采用**两级分辨率**策略（不是严格意义上的金字塔结构），为每张照片生成两种优化版本：

| 分辨率等级 | Purpose 常量 | 最大尺寸 | 格式 | 用途 |
|-----------|-------------|---------|------|------|
| 缩略图 | `PhotoThumbnail` | 1024×1024 | JPEG (70%) | 相册网格、列表预览 |
| 高分辨率 | `PhotoHighRes` | 原始尺寸（仅格式转换） | JPEG (70%) | 大图查看、RAW 格式转码 |
| 原图 | `MediaOriginal` | 原始尺寸 | 原始格式 | 下载、原始格式查看 |

> **注意**：对于本身就是 Web 兼容格式（如 JPEG、PNG 等）的照片，不会生成 `PhotoHighRes` 版本，直接使用原图。

---

## 一、调度入口

缩略图生成有**两个调度入口**：批量扫描触发 和 按需访问触发。

### 1.1 批量扫描触发（主路径）

当用户启动相册扫描时，扫描器会遍历所有媒体文件并生成缩略图。

```
扫描启动
  ↓
scanner_media.go: ScanMedia()          # 发现新媒体，写入数据库
  ↓
media_scan.go: scanMedia()             # 单个媒体处理总入口
  ↓
scanner_tasks.go: Tasks.ProcessMedia() # 任务管道依次执行
  ↓
process_photo_task.go: ProcessMedia()  # 照片处理任务（核心）
  ↓
生成 thumbnail + high-res + original 三种 MediaURL
```

**调用链关键文件**：
- `api/scanner/scanner_user.go` - 用户级扫描入口
- `api/scanner/scanner_album.go` - 相册级扫描
- `api/scanner/scanner_media.go` - 媒体发现与入库
- `api/scanner/media_scan.go` - 媒体处理总入口
- `api/scanner/scanner_tasks/scanner_tasks.go` - 任务管道编排
- `api/scanner/scanner_tasks/processing_tasks/process_photo_task.go` - 照片处理核心逻辑

### 1.2 按需访问触发（缓存未命中回退）

当用户访问照片 URL 时，如果磁盘缓存不存在，会触发即时重新生成。

```
用户访问 /photo/{media_name}
  ↓
photos.go: RegisterPhotoRoutes()       # HTTP 路由处理
  ↓
数据库查询 MediaURL 记录
  ↓
检查磁盘缓存文件是否存在
  ├─ 存在 → 直接返回文件（缓存命中）
  └─ 不存在 → 调用 ProcessSingleMediaFunc 重新生成
       ↓
       scanner_media.go: ProcessSingleMedia()
       ↓
       走完整的扫描处理流程
```

**关键代码**（`api/routes/photos.go:52-71`）：
```go
if _, err := os.Stat(cachedPath); os.IsNotExist((err)) {
    if err = scanner.ProcessSingleMediaFunc(r.Context(), db, media); err != nil {
        // 处理失败
    }
    // 再次检查文件
}
```

---

## 二、照片处理核心流程

`ProcessPhotoTask.ProcessMedia()` 是缩略图生成的核心调度函数，位于 `api/scanner/scanner_tasks/processing_tasks/process_photo_task.go`。

### 2.1 处理顺序

处理流程严格按照以下顺序执行：

```
1. 检查/生成 high-res 版本
   ├─ 若数据库中无记录 → 生成 high-res JPEG → 写入 MediaURL
   └─ 若数据库有记录但磁盘无文件 → 重新生成 high-res 到磁盘

2. 检查/生成 original 版本
   └─ 若数据库中无记录 → 写入 MediaURL（不复制文件，指向原路径）

3. 检查/生成 thumbnail 版本
   ├─ 若数据库中无记录 → 生成缩略图 → 写入 MediaURL
   └─ 若数据库有记录但磁盘无文件 → 重新生成缩略图到磁盘
```

### 2.2 baseImagePath 的选择

生成缩略图时，`baseImagePath`（基准图片路径）的选择逻辑：

```
if high-res 存在（生成了或已存在）:
    baseImagePath = high-res 文件路径
else:
    baseImagePath = 原始照片路径
```

这样做的好处是：对于 RAW 格式照片，先转成 JPEG（high-res），再基于 JPEG 生成缩略图，避免重复解码 RAW 文件。

---

## 三、缩略图生成算法

### 3.1 尺寸计算

尺寸计算位于 `api/scanner/media_encoding/encode_photo.go:25-51` 的 `ThumbnailScale()` 方法。

**算法逻辑**：
- 目标：最长边不超过 1024px
- 保持原始宽高比
- 如果原图小于 1024px，则使用原图尺寸

```go
aspect = width / height

if aspect > 1:        // 横图
    width = 1024
    height = 1024 / aspect
else:                 // 竖图
    width = 1024 * aspect
    height = 1024

if width > 原宽度:    // 原图更小，不放大
    使用原图尺寸
```

### 3.2 底层实现

使用 ImageMagick（MagickWand）进行图像处理，位于 `api/scanner/media_encoding/executable_worker/magickwand.go`。

**缩略图生成流程**：
```go
GenerateThumbnail(inputPath, outputPath, width, height):
    1. 读取图片
    2. AutoOrientImage()        // 根据 EXIF 自动旋转
    3. SetImageOrientation()   // 重置方向标记为 top-left
    4. ThumbnailImage(w, h)    // 调整大小（使用 Imagick 的优化算法）
    5. SetFormat("JPEG")       // 转 JPEG 格式
    6. SetImageCompressionQuality(70)  // 质量 70%
    7. WriteImage(outputPath)  // 写入磁盘
```

---

## 四、缓存结构

### 4.1 磁盘缓存目录结构

缓存根目录由环境变量 `PHOTOVIEW_MEDIA_CACHE` 控制，默认 `./media_cache`。

```
media_cache/
└── {album_id}/                    # 按相册分目录
    └── {media_id}/                # 按媒体分目录
        ├── thumbnail_{name}_{token}.jpg   # 缩略图
        └── highres_{name}_{token}.jpg     # 高分辨率（仅非 web 兼容格式）
```

**缓存路径构建**：`api/utils/media_cache.go:13-39` 的 `CachePathForMedia()`

### 4.2 数据库模型

`MediaURL` 表（`api/graphql/models/media.go:106-116`）存储所有分辨率版本的元数据：

| 字段 | 说明 |
|-----|------|
| `media_id` | 关联的媒体 ID |
| `media_name` | 唯一生成的文件名（用于 URL） |
| `width` / `height` | 实际尺寸 |
| `purpose` | 用途：`thumbnail` / `high-res` / `original` |
| `content_type` | MIME 类型 |
| `file_size` | 文件大小（字节） |

**命名规则**：`{prefix}_{原文件名}_{随机token}.jpg`，例如：
- `thumbnail_IMG_1234_jpg_a1b2c3.jpg`
- `highres_IMG_1234_raw_d4e5f6.jpg`

命名函数：`processing_helpers.go:34-39` 的 `generateUniqueMediaNamePrefixed()`

---

## 五、缓存命中路径

### 5.1 完整访问链路

```
前端 GraphQL 请求
  ↓
media.go resolver: Thumbnail() / HighRes()
  ↓
DataLoader (mediaURLLoader.go) 批量加载
  ↓
数据库查询 MediaURL 表
  ↓
返回 MediaURL 对象（含 URL 字段）
  ↓
前端请求 /photo/{media_name}
  ↓
photos.go 路由处理
  ├─ 鉴权
  ├─ 查数据库获取 MediaURL
  ├─ 计算磁盘缓存路径
  ├─ 检查文件是否存在
  │   ├─ 存在 → 直接 ServeFile
  │   └─ 不存在 → 调用 ProcessSingleMedia 重新生成
  └─ 设置 Cache-Control: private, max-age=31536000, immutable
```

### 5.2 DataLoader 批量加载

为了优化 N+1 查询，使用 DataLoader 批量获取缩略图 URL：

- `NewThumbnailMediaURLLoader` - 缩略图加载器
- `NewHighresMediaURLLoader` - 高分辨率加载器

位于 `api/dataloader/mediaURLLoader.go`，批大小 100，等待 5ms。

### 5.3 HTTP 缓存策略

命中成功的图片设置强缓存头：
```
Cache-Control: private, max-age=31536000, immutable
```
即浏览器缓存一年，且不可变（因为文件名包含随机 token，内容不会变）。

---

## 七、并发扫描：锁与队列模型

### 7.1 全局扫描队列（ScannerQueue）

Photoview 使用一个全局队列管理器来控制并发扫描，避免资源耗尽。

**数据结构**（`api/scanner/scanner_queue/queue.go:47-56`）：
```go
type ScannerQueue struct {
    mutex       sync.Mutex      // 保护队列状态的互斥锁
    idle_chan   chan bool       // 空闲通知通道（缓冲1）
    in_progress []ScannerJob    // 正在执行的任务
    up_next     []ScannerJob    // 等待执行的任务
    db          *gorm.DB        // 数据库连接
    settings    ScannerQueueSettings  // max_concurrent_tasks
    close_chan  *chan bool      // 关闭信号通道
    running     bool            // 运行状态标志
}
```

### 7.2 并发度控制

并发度从数据库 `site_info.concurrent_workers` 读取，可在 UI 中动态调整。

**初始化**（`api/scanner/scanner_queue/queue.go:60-86`）：
```go
func InitializeScannerQueue(db *gorm.DB) error {
    site_info, _ := models.GetSiteInfo(db)
    concurrentWorkers := site_info.ConcurrentWorkers
    
    global_scanner_queue = ScannerQueue{
        idle_chan:   make(chan bool, 1),
        settings:    ScannerQueueSettings{max_concurrent_tasks: concurrentWorkers},
        // ...
    }
    
    go global_scanner_queue.startBackgroundWorker()
}
```

**动态调整**（`api/scanner/scanner_queue/queue.go:92-98`）：
```go
func ChangeScannerConcurrentWorkers(newMaxWorkers int) {
    global_scanner_queue.mutex.Lock()
    defer global_scanner_queue.mutex.Unlock()
    global_scanner_queue.settings.max_concurrent_tasks = newMaxWorkers
}
```

### 7.3 后台工作协程

**工作循环**（`api/scanner/scanner_queue/queue.go:100-121`）：
```
startBackgroundWorker():
    无限循环:
        1. 阻塞等待 idle_chan 信号
        2. 检查是否需要关闭
        3. 调用 processQueue() 启动新任务
```

### 7.4 任务调度逻辑

**processQueue() 核心**（`api/scanner/scanner_queue/queue.go:136-192`）：
```
加锁（mutex.Lock）
  ↓
while 运行中任务数 < maxJobs 且 有等待任务:
    取出 up_next[0]，移入 in_progress
    启动 goroutine 执行任务:
        1. 调用 job.Run() → ScanAlbum()
        2. 加锁，从 in_progress 中移除已完成任务
        3. 解锁
        4. 调用 notify() 唤醒调度器
  ↓
解锁（mutex.Unlock）
  ↓
发送进度通知（节流 500ms）
```

**关键点**：
- `mutex` 保护 `in_progress` 和 `up_next` 两个切片的并发访问
- 每个相册任务在独立 goroutine 中运行
- 任务完成后通过 `notify()` 触发下一轮调度
- `idle_chan` 采用非阻塞发送（`select default`），避免重复通知

### 7.5 任务入队

**AddUserToQueue**（`api/scanner/scanner_queue/queue.go:223-239`）：
```
加锁
  ↓
遍历用户所有根相册
  为每个相册创建 ScannerJob
  检查是否已在队列中（in_progress + up_next）
  若不在队列，追加到 up_next
  ↓
解锁
  ↓
notify() 唤醒调度器
```

**去重逻辑**：同一相册不会重复入队，通过比较 album.ID 判断。

### 7.6 专辑内部串行处理

**重要**：队列的并发度是**相册级别的**，但**单个相册内的照片是串行处理的**。

`api/scanner/scanner_album.go:101-107`:
```go
for i, media := range albumMedia {
    mediaData := media_encoding.NewEncodeMediaData(media)
    if err := scanMedia(ctx, media, &mediaData, i, len(albumMedia)); err != nil {
        // 错误处理
    }
}
```

**并发模型总结**：
```
全局队列（ScannerQueue）
  ├─ 并发度：N（由 concurrent_workers 控制）
  ├─ 任务单元：单个相册
  │   └─ 相册内照片：串行处理
  └─ 锁：sync.Mutex 保护队列状态
```

---

## 八、RAW 图片解码栈

### 8.1 RAW 处理流程概览

```
RAW 文件 (.CR2, .NEF, .ARW 等)
  ↓
1. 媒体类型检测（exiftool → MIME 类型）
  ↓
2. 配套文件查找（CounterpartFilesTask）
  ├─ 若存在同名 JPEG → 使用 JPEG 作为解码源（跳过 RAW 解码）
  └─ 不存在 → 继续
  ↓
3. ImageMagick + LibRaw 解码（MagickWand）
  ↓
4. 生成 high-res JPEG（质量 70%）
  ↓
5. 基于 high-res 生成 thumbnail
```

### 8.2 媒体类型检测

**MIME 类型检测**（`api/scanner/externaltools/exif/exif.go:96-114`）：
```go
func MIMEType(filepath string) (string, error) {
    globalMu.Lock()    // exiftool 是全局单例，加互斥锁
    defer globalMu.Unlock()
    
    var mime exiftool.MIMEType
    globalExifParser.QueryJSONTagsByNumber(filepath, &mime)
    return *mime.MIMEType, nil
}
```

> **注意**：exiftool 实例是全局单例，所有线程共享，通过 `sync.Mutex` 保护。

**类型判断**（`api/scanner/media_type/media_type.go:87-101`）：
```go
func (t MediaType) IsWebCompatible() bool {
    // 检查是否在 webImageMimetypes 或 webVideoMimetypes 集合中
    // 包含: JPEG, PNG, WebP, BMP, GIF, MP4, MPEG, WebM, OGG
}
```

RAW 格式（如 `image/x-canon-cr2`）不在上述集合中，`IsWebCompatible()` 返回 `false`。

### 8.3 配套文件机制（Counterpart Files）

为了避免重复解码，Photoview 支持 RAW + JPEG 配套文件模式。

**查找逻辑**（`api/scanner/media_type/counterpart.go:10-45`）：
```go
func FindWebCounterpart(imagePath string) (string, bool) {
    // 同目录下查找同名但不同扩展名的文件
    // 例如: IMG_1234.CR2 → 查找 IMG_1234.*
    // 返回第一个匹配的 Web 兼容格式文件
}

func FindRawCounterpart(imagePath string) (string, bool) {
    // 反向：JPEG 文件查找对应的 RAW 文件
}
```

**任务管道中的处理**（`api/scanner/scanner_tasks/processing_tasks/counterpart_files_task.go`）：

1. **MediaFound 阶段**：决定是否跳过文件
   - 如果是 JPEG 且存在同名 RAW → 跳过 JPEG（以 RAW 为准）
   - 如果是 RAW 且禁用 RAW 处理 → 跳过

2. **BeforeProcessMedia 阶段**：设置解码源
   ```go
   if !mediaType.IsWebCompatible() {
       if counterpartFile, ok := media_type.FindWebCounterpart(mediaPath); ok {
           mediaData.CounterpartPath = &counterpartFile
       }
   }
   ```

### 8.4 底层解码栈

**ImageMagick + LibRaw 编译链**：

1. **LibRaw**（`dependencies/build_libraw.sh`）：
   - 从源码编译，支持 OpenMP 多线程
   - 启用 JPEG、LCMS（色彩管理）、ZLIB
   - 输出静态库 `libraw.a` 和动态库 `libraw.so`

2. **ImageMagick**（`dependencies/build_imagemagick.sh`）：
   - 编译时启用 `--with-raw` 选项
   - 链接 LibRaw 库
   - 支持的 RAW 格式由 LibRaw 决定
   - 同时支持 HEIC、JPEG XL、WebP 等现代格式

**Go 绑定**：使用 `gographics/imagick.v3` 绑定 ImageMagick 的 MagickWand API。

### 8.5 解码代码走向

**high-res 生成**（`api/scanner/media_encoding/encode_photo.go:129-153`）：
```go
func (img *EncodeMediaData) EncodeHighRes(outputPath string) error {
    contentType, _ := img.ContentType()
    
    if contentType.IsImage() && !contentType.IsWebCompatible() {
        imgPath := img.Media.Path
        if img.CounterpartPath != nil {
            imgPath = *img.CounterpartPath  // 优先使用配套 JPEG
        }
        
        // 调用 MagickWand 转码
        executable_worker.Magick.EncodeJpeg(imgPath, outputPath, 70)
    }
}
```

**MagickWand 编码**（`api/scanner/media_encoding/executable_worker/magickwand.go:35-55`）：
```go
func (cli *MagickWand) EncodeJpeg(inputPath, outputPath string, quality uint) error {
    wand := imagick.NewMagickWand()
    wand.ReadImage(inputPath)      // ← LibRaw 在这一步解码 RAW
    wand.AutoOrientImage()         // 自动旋转
    wand.SetImageOrientation(imagick.ORIENTATION_TOP_LEFT)
    wand.SetFormat("JPEG")
    wand.SetImageCompressionQuality(quality)
    wand.WriteImage(outputPath)    // 写入 JPEG
}
```

### 8.6 RAW 处理环境变量

- `PHOTOVIEW_DISABLE_RAW_PROCESSING=1`：完全禁用 RAW 处理，只处理 JPEG
- 未设置时，RAW 文件会被解码生成 high-res 和 thumbnail

---

## 九、视频解码与转码栈

### 9.1 视频处理流程

```
视频文件 (.MP4, .AVI, .MOV, .MKV 等)
  ↓
1. 媒体类型检测（exiftool → MIME）
  ↓
2. ffprobe 读取元数据（时长、分辨率、编码格式）
  ↓
3. 判断是否 Web 兼容
  ├─ 兼容（MP4、WebM 等）→ 直接记录 original
  └─ 不兼容 → ffmpeg 转码为 H.264 MP4
  ↓
4. 生成视频缩略图（截取 25% 处的帧）
  ↓
5. 保存 MediaURL 记录
```

### 9.2 视频转码核心

**转码条件**（`api/scanner/scanner_tasks/processing_tasks/process_video_task.go:87-88`）：
```go
if videoWebURL == nil && !videoType.IsWebCompatible() {
    // 执行转码
}
```

**ffmpeg 转码命令**（`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:74-96`）：
```bash
ffmpeg -i input.mkv \
    -vcodec h264 \           # 视频编码器（支持硬件加速）
    -acodec aac \            # 音频编码器
    -vf "scale='min(1080,iw)':'min(1080,ih)':force_original_aspect_ratio=decrease:force_divisible_by=2" \
    -movflags +faststart+use_metadata_tags \
    output.mp4
```

**参数说明**：
- 分辨率：最长边 ≤ 1080px，保持宽高比
- `force_divisible_by=2`：H.264 要求宽高为偶数
- `+faststart`：将 moov atom 移到文件开头，支持流式播放
- `+use_metadata_tags`：保留原视频元数据

### 9.3 硬件加速支持

**编码器映射**（`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:13-19`）：
```go
var hwAccToCodec = map[string]string{
    "qsv":   "h264_qsv",     // Intel 核显
    "vaapi": "h264_vaapi",   // VA-API (AMD/Intel)
    "nvenc": "h264_nvenc",   // NVIDIA NVENC
}
```

**配置方式**：环境变量 `PHOTOVIEW_VIDEO_HARDWARE_ACCELERATION`
- 设置为 `qsv` / `vaapi` / `nvenc` 启用对应硬件加速
- 设置为 `_libx265` 可直接指定编码器（下划线前缀）
- 默认使用软件编码 `h264`（libx264）

**秘密功能**（`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:54-56`）：
```go
if strings.HasPrefix(hwAcc, "_") {
    codec = hwAcc[1:]  // 直接使用下划线后的编码器名称
}
```

### 9.4 视频缩略图生成

**截取策略**（`api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:98-122`）：
- 截取视频 25% 处的帧（避免全黑片头）
- 尺寸：最长边 ≤ 1024px
- 格式：JPEG

**ffmpeg 命令**：
```bash
ffmpeg -ss 25%_time -i input.mp4 \
    -vframes 1 \
    -an \
    -vf "scale='min(1024,iw)':'min(1024,ih)':..." \
    thumbnail.jpg
```

### 9.5 ffprobe 元数据读取

**元数据读取**（`api/scanner/media_encoding/encode_photo.go:155-170`）：
```go
func (enc *EncodeMediaData) VideoMetadata() (*ffprobe.ProbeData, error) {
    ctx, cancelFn := context.WithTimeout(context.Background(), utils.MediaProbeTimeout())
    defer cancelFn()
    
    data, err := ffprobe.ProbeURL(ctx, enc.Media.Path)
    return data, err
}
```

**超时控制**：默认 5 秒，可通过 `PHOTOVIEW_MEDIA_PROBE_TIMEOUT` 调整。

### 9.6 视频处理环境变量

- `PHOTOVIEW_DISABLE_VIDEO_ENCODING=1`：禁用视频转码
- `PHOTOVIEW_VIDEO_HARDWARE_ACCELERATION`：硬件加速类型
- `PHOTOVIEW_MEDIA_PROBE_TIMEOUT`：ffprobe 超时（秒）

---

## 十、Worker 初始化与生命周期

### 10.1 启动流程

`api/server.go:41-72`:
```go
func main() {
    terminateWorkers := executable_worker.Initialize()  // 1. MagickWand + ffmpeg
    defer terminateWorkers()
    
    exifCleanup, _ := exif.Initialize()                 // 2. exiftool 单例
    defer exifCleanup()
    
    scanner_queue.InitializeScannerQueue(db)            // 3. 扫描队列
    periodic_scanner.InitializePeriodicScanner(db)      // 4. 定时扫描
    face_detection.InitializeFaceDetector(db)           // 5. 人脸检测
}
```

### 10.2 Worker 初始化详情

**executable_worker.Initialize()**（`api/scanner/media_encoding/executable_worker/executable_worker.go:17-29`）：
```go
func Initialize() func() {
    Magick = newMagickWand()    // 初始化 ImageMagick，全局单例
    Ffmpeg = newFfmpegCli()     // 检查 ffmpeg 可用性
    SetFfprobePath()            // 查找 ffprobe 路径
    
    return func() {
        Magick.Terminate()      // 清理 ImageMagick 资源
        Magick = nil
    }
}
```

> **注意**：MagickWand 和 exiftool 都是全局单例，且都受互斥锁保护，不支持并发调用。

### 10.3 优雅关闭

`api/server.go:140-159`:
```go
func setupGracefulShutdown(svr *http.Server) {
    // ...
    periodic_scanner.ShutdownPeriodicScanner()   // 停止定时任务
    scanner_queue.CloseScannerQueue()            // 等待队列中任务完成
    svr.Shutdown(ctx)
}
```

`CloseScannerQueue()` 会等待 `in_progress` 为空后才返回，确保正在处理的任务正常完成。

---

## 十一、关键代码索引

| 功能 | 文件 | 行号 |
|-----|------|------|
| 缩略图尺寸计算 | `api/scanner/media_encoding/encode_photo.go` | 25-51 |
| 缩略图编码函数 | `api/scanner/media_encoding/encode_photo.go` | 66-94 |
| 照片处理任务 | `api/scanner/scanner_tasks/processing_tasks/process_photo_task.go` | 18-126 |
| 缩略图生成与保存 | `api/scanner/scanner_tasks/processing_tasks/processing_functions.go` | 58-97 |
| 缓存路径构建 | `api/utils/media_cache.go` | 12-39 |
| MediaURL 模型 | `api/graphql/models/media.go` | 106-147 |
| 照片访问路由 | `api/routes/photos.go` | 15-81 |
| 单媒体处理函数 | `api/scanner/scanner_media.go` | 77-93 |
| MagickWand 缩略图 | `api/scanner/media_encoding/executable_worker/magickwand.go` | 57-81 |
| DataLoader 缩略图 | `api/dataloader/mediaURLLoader.go` | 44-52 |
| **并发队列** | | |
| 扫描队列结构体 | `api/scanner/scanner_queue/queue.go` | 47-56 |
| 队列初始化 | `api/scanner/scanner_queue/queue.go` | 60-86 |
| 后台工作协程 | `api/scanner/scanner_queue/queue.go` | 100-121 |
| 任务调度逻辑 | `api/scanner/scanner_queue/queue.go` | 136-192 |
| 任务入队 | `api/scanner/scanner_queue/queue.go` | 223-239 |
| 相册扫描入口 | `api/scanner/scanner_album.go` | 87-114 |
| **RAW 解码** | | |
| MIME 类型检测 | `api/scanner/externaltools/exif/exif.go` | 96-114 |
| 媒体类型判断 | `api/scanner/media_type/media_type.go` | 87-101 |
| 配套文件查找 | `api/scanner/media_type/counterpart.go` | 10-45 |
| 配套文件任务 | `api/scanner/scanner_tasks/processing_tasks/counterpart_files_task.go` | 17-62 |
| High-res 编码 | `api/scanner/media_encoding/encode_photo.go` | 129-153 |
| MagickWand JPEG 编码 | `api/scanner/media_encoding/executable_worker/magickwand.go` | 35-55 |
| LibRaw 编译 | `dependencies/build_libraw.sh` | 全文 |
| ImageMagick 编译 | `dependencies/build_imagemagick.sh` | 全文 |
| **视频处理** | | |
| 视频处理任务 | `api/scanner/scanner_tasks/processing_tasks/process_video_task.go` | 24-204 |
| ffmpeg 转码 | `api/scanner/media_encoding/executable_worker/ffmpeg_cli.go` | 74-96 |
| 视频缩略图 | `api/scanner/media_encoding/executable_worker/ffmpeg_cli.go` | 98-122 |
| 硬件加速映射 | `api/scanner/media_encoding/executable_worker/ffmpeg_cli.go` | 13-19 |
| 视频元数据 | `api/scanner/media_encoding/encode_photo.go` | 155-170 |
| ffmpeg 初始化 | `api/scanner/media_encoding/executable_worker/ffmpeg_cli.go` | 27-68 |
| **Worker 生命周期** | | |
| 服务器启动 | `api/server.go` | 34-138 |
| Worker 初始化 | `api/scanner/media_encoding/executable_worker/executable_worker.go` | 17-29 |
| exif 初始化 | `api/scanner/externaltools/exif/exif.go` | 15-41 |
| 优雅关闭 | `api/server.go` | 140-159 |
| **缓存清理** | | |
| 媒体清理函数 | `api/scanner/scanner_tasks/cleanup_tasks/cleanup_media.go` | 17-65 |
| 相册清理函数 | `api/scanner/scanner_tasks/cleanup_tasks/cleanup_media.go` | 68-136 |
| 清理任务入口 | `api/scanner/scanner_tasks/cleanup_tasks/media_cleanup_task.go` | 9-21 |
| 定期扫描调度器 | `api/scanner/periodic_scanner/periodic_scanner.go` | 144-174 |
| **权限与共享** | | |
| 媒体鉴权函数 | `api/routes/authenticate_routes.go` | 19-46 |
| 相册鉴权函数 | `api/routes/authenticate_routes.go` | 48-69 |
| 共享令牌校验 | `api/routes/authenticate_routes.go` | 71-155 |
| 用户认证中间件 | `api/graphql/auth/auth.go` | 31-69 |
| 用户-相册归属 | `api/graphql/models/user.go` | 167-180 |
| 相册父子关系 | `api/graphql/models/album.go` | 59-81 |
| 共享令牌模型 | `api/graphql/models/share_token.go` | 7-18 |
| 共享令牌创建 | `api/graphql/models/actions/share_token_actions.go` | 15-103 |

---

## 十二、媒体缓存清理与磁盘空间回收

### 12.1 清理机制概览

Photoview **没有基于磁盘配额或缓存大小的主动 GC**。清理逻辑完全由**扫描事件驱动**：当扫描器发现文件系统上的文件/目录已不存在时，才删除对应的数据库记录和缓存文件。

```
扫描事件触发
  ↓
AfterScanAlbum 钩子
  ↓
CleanupMedia()    → 删除已消失媒体 + 对应缓存目录
DeleteOldUserAlbums() → 删除已消失相册 + 对应缓存目录
```

### 12.2 媒体级清理：CleanupMedia

**触发时机**：每次扫描完一个相册后，由 `MediaCleanupTask.AfterScanAlbum()` 调用。

`api/scanner/scanner_tasks/cleanup_tasks/media_cleanup_task.go:13-21`:
```go
func (t MediaCleanupTask) AfterScanAlbum(ctx TaskContext, changedMedia []*models.Media,
    albumMedia []*models.Media) error {
    cleanupErrors := CleanupMedia(ctx.GetDB(), ctx.GetAlbum().ID, albumMedia)
    // ...
}
```

**清理逻辑**（`api/scanner/scanner_tasks/cleanup_tasks/cleanup_media.go:17-65`）：
```
1. 查询数据库: SELECT * FROM media WHERE album_id = ? AND id NOT IN (本次扫描到的ID列表)
   ↓
2. 对每条「不在磁盘上」的媒体记录:
   a. 删除缓存目录: os.RemoveAll(media_cache/{album_id}/{media_id})
   b. 收集 mediaID
   ↓
3. 批量删除数据库: DELETE FROM media WHERE id IN (待删除ID列表)
   ↓
4. 如果有人脸检测器，重新加载人脸数据
```

**关键点**：
- 缓存清理粒度是**单个媒体目录**（`{album_id}/{media_id}/`），会删除该目录下所有分辨率的缓存文件
- 数据库删除使用批量操作，不是逐条删除
- 缓存文件删除失败不会中断流程，只记录错误

### 12.3 相册级清理：DeleteOldUserAlbums

**触发时机**：在 `FindAlbumsForUser()` 末尾调用（`api/scanner/scanner_user.go:222`）。

`api/scanner/scanner_tasks/cleanup_tasks/cleanup_media.go:68-136`:
```
1. 查询数据库: 找到用户关联但本次未扫描到的相册
   (通过 user_albums JOIN albums, WHERE album_id NOT IN (本次扫描到的相册ID))
   ↓
2. 对每个待删除相册:
   删除缓存目录: os.RemoveAll(media_cache/{album_id}/)
   ↓
3. 事务内:
   a. DELETE FROM user_albums WHERE album_id IN (待删除相册ID)
   b. DELETE FROM albums WHERE id IN (待删除相册ID)
   ↓
4. 重新加载人脸数据
```

**关键点**：
- 相册级清理会删除**整个相册缓存目录**（包含该相册下所有媒体的缓存）
- 先删缓存，再删数据库关联（user_albums），最后删相册记录
- 操作在事务内执行，保证数据库一致性
- 但缓存删除不在事务内——如果数据库回滚，已删除的缓存文件不会恢复

### 12.4 定期扫描调度器

**PeriodicScanner**（`api/scanner/periodic_scanner/periodic_scanner.go`）是唯一可自动触发清理的后台机制。

**工作方式**：
```
定时器触发（间隔由 site_info.periodic_scan_interval 控制）
  ↓
AddAllToQueue() → 将所有用户的根相册加入扫描队列
  ↓
ScannerQueue 执行扫描
  ↓
每个相册扫描完成后 → CleanupMedia() → 清理已消失的媒体
每个用户扫描完成后 → DeleteOldUserAlbums() → 清理已消失的相册
```

**配置**：
- `site_info.periodic_scan_interval`：扫描间隔（秒），0 表示禁用
- 可在 UI 中动态调整，调用 `ChangePeriodicScanInterval()`
- 间隔改变后立即生效（通过 `ticker_changed` 通道通知）

**数据结构**：
```go
type periodicScanner struct {
    ticker         *time.Ticker     // 定时器
    tickerLocker   sync.Mutex       // 保护 ticker
    ticker_changed chan bool        // 间隔变更通知
    done           chan struct{}     // 关闭信号
    db             *gorm.DB
    scannerQueue   ScannerQueue     // 依赖扫描队列
}
```

### 12.5 关于缓存大小限制

**Photoview 当前没有实现基于磁盘配额的缓存大小限制。**

代码中不存在以下功能：
- ❌ 缓存总大小上限
- ❌ LRU 驱逐策略
- ❌ 按文件大小清理
- ❌ 缓存过期时间（TTL）

缓存的唯一回收途径是**源文件从磁盘消失后，下次扫描时被清理**。这意味着：

1. **缓存只会增长**：只要源文件存在，缓存文件不会被主动删除
2. **唯一的清理窗口**：修改 `PHOTOVIEW_MEDIA_CACHE` 环境变量指向的路径对应的磁盘空间时需要外部监控
3. `MediaURL.FileSize` 字段记录了每个缓存文件的大小，但仅用于前端展示，不参与任何驱逐决策

---

## 十三、缓存命中路径上的权限校验

### 13.1 完整请求处理链路

以照片访问为例（`api/routes/photos.go:15-81`），权限校验在缓存查找**之前**执行：

```
GET /photo/{media_name}
  ↓
① 数据库查询 MediaURL + JOIN Media
  ↓
② authenticateMedia() 权限校验 ← 在此拦截
  ↓  通过
③ 计算 CachedPath
  ↓
④ os.Stat(cachedPath) 检查磁盘
  ├─ 存在 → ServeFile
  └─ 不存在 → ProcessSingleMedia → ServeFile
```

**顺序关键**：权限校验（②）在缓存路径计算（③）和磁盘 IO（④）之前。未授权的请求不会触发任何磁盘操作或重新编码。

### 13.2 authenticateMedia 详细流程

`api/routes/authenticate_routes.go:19-46`:

```
authenticateMedia(media, db, request)
  ↓
从 Context 获取 user（由 auth 中间件注入）
  ├─ user != nil（已登录用户）
  │   ↓
  │   查询 Album: db.First(&album, media.AlbumID)
  │   ↓
  │   user.OwnsAlbum(db, &album)
  │   ├─ true  → ✅ 放行
  │   └─ false → ❌ 403 Forbidden
  │
  └─ user == nil（未登录，匿名访问）
      ↓
      shareTokenFromRequest(db, r, &media.ID, &albumID)
      ├─ 校验通过 → ✅ 放行
      └─ 校验失败 → ❌ 403 Forbidden
```

### 13.3 已登录用户权限校验：OwnsAlbum

`api/graphql/models/user.go:167-180`:

```go
func (user *User) OwnsAlbum(db *gorm.DB, album *Album) (bool, error) {
    filter := func(query *gorm.DB) *gorm.DB {
        return query.Where(
            "EXISTS (SELECT 1 FROM user_albums WHERE user_albums.user_id = ? AND user_albums.album_id = id LIMIT 1)",
            user.ID)
    }
    ownedParents, _ := album.GetParents(db, filter)
    return len(ownedParents) > 0, nil
}
```

**逻辑**：用户只要拥有媒体所属相册的**任意祖先相册**，就视为有权限。

**递归向上查找**（`api/graphql/models/album.go:59-81`）：
```sql
WITH recursive super_albums AS (
    SELECT * FROM albums AS leaf WHERE id = ?
    UNION ALL
    SELECT parent.* FROM albums AS parent
    JOIN super_albums ON parent.id = super_albums.parent_album_id
)
SELECT * FROM super_albums
WHERE EXISTS (SELECT 1 FROM user_albums
              WHERE user_albums.user_id = ? AND user_albums.album_id = id)
```

**举例**：
```
用户拥有相册 /Vacation（ID=5）
  └── 子相册 /Vacation/Beach（ID=10）
       └── 媒体 photo.jpg（album_id=10）

访问 photo.jpg 时：
  OwnsAlbum 查找 album_id=10 的所有父级
  发现 ID=5 在 user_albums 中 → ✅ 有权限
```

### 13.4 匿名用户权限校验：ShareToken

`api/routes/authenticate_routes.go:71-155`:

```
shareTokenFromRequest(db, request, mediaID, albumID)
  ↓
① 从 URL 参数获取 token: r.URL.Query().Get("token")
  └─ 空值 → ❌ 403 "share token not provided"
  ↓
② 数据库查找 ShareToken: WHERE value = ?
  └─ 未找到 → ❌ 403 "invalid share token"
  ↓
③ 检查过期: shareToken.Expire != nil && now.After(expire)
  └─ 已过期 → ❌ 403 "invalid share token"
  ↓
④ 检查密码（如果有）:
   从 Cookie 读取: share-token-pw-{token_value}
   bcrypt.CompareHashAndPassword 校验
  └─ 不匹配 → ❌ 403 "share token password invalid"
  ↓
⑤ 校验范围:
   ├─ Album ShareToken: albumID 必须匹配，或 albumID 是 shareToken.AlbumID 的子相册
   │   （递归 SQL 查找子相册）
   └─ Media ShareToken: mediaID 必须完全匹配
  ↓
✅ 校验通过
```

**ShareToken 模型**（`api/graphql/models/share_token.go`）：

| 字段 | 类型 | 说明 |
|-----|------|------|
| `Value` | string | 令牌值（24位随机字符串） |
| `OwnerID` | int | 创建者用户 ID |
| `Expire` | *time.Time | 过期时间（nil 永不过期） |
| `Password` | *string | 访问密码的 bcrypt 哈希（nil 无密码） |
| `AlbumID` | *int | 关联相册 ID（Album 类型的共享） |
| `MediaID` | *int | 关联媒体 ID（Media 类型的共享） |

**Album 共享的子相册访问**：
当共享令牌关联的是父相册时，子相册下的媒体也可以访问。通过递归 CTE 查询验证：

```sql
WITH recursive child_albums AS (
    SELECT * FROM albums WHERE parent_album_id = ?
    UNION ALL
    SELECT child.* FROM albums child
    JOIN child_albums parent ON parent.id = child.parent_album_id
)
SELECT COUNT(id) FROM child_albums WHERE id = ?
```

### 13.5 认证中间件注入链

```
HTTP 请求进入
  ↓
auth.Middleware(db)（全局中间件）
  ├─ 读取 Cookie: auth-token
  ├─ DataLoader 查找用户: UserFromAccessToken.Load(cookieValue)
  ├─ 用户存在 → AddUserToContext(ctx, user)
  └─ 用户不存在 → ctx 中无 user（后续按匿名处理）
  ↓
路由处理函数
  ↓
auth.UserFromContext(ctx) → 获取 user（可能为 nil）
  ↓
authenticateMedia() / authenticateAlbum()
```

**关键**：认证中间件不会拒绝未认证的请求，它只是尽力将 user 注入 Context。权限拒绝发生在路由层的 `authenticateMedia()` / `authenticateAlbum()` 中。

### 13.6 三种资源的鉴权差异

| 资源 | 路由 | 鉴权函数 | 共享令牌支持 |
|------|------|---------|------------|
| 照片 | `/photo/{name}` | `authenticateMedia()` | ✅ Media + Album 令牌 |
| 视频 | `/video/{name}` | `authenticateMedia()` | ✅ Media + Album 令牌 |
| 相册下载 | `/download/album/{id}/{purpose}` | `authenticateAlbum()` | ✅ 仅 Album 令牌 |

**注意**：
- 照片和视频路由使用 `authenticateMedia()`，同时支持 Media 和 Album 类型的共享令牌
- 相册下载路由使用 `authenticateAlbum()`，只支持 Album 类型的共享令牌
- 所有鉴权在缓存文件访问之前完成，未授权请求不会触发磁盘 IO

---

## 十四、ScannerTasks 子任务注册、接口与触发顺序

### 14.1 任务注册表

所有子任务在 `api/scanner/scanner_tasks/scanner_tasks.go:15-27` 注册为一个有序列表：

```go
var allTasks []scanner_task.ScannerTask = []scanner_task.ScannerTask{
    NotificationTask{},             // 1. 通知
    IgnorefileTask{},               // 2. 忽略文件
    processing_tasks.CounterpartFilesTask{}, // 3. 配套文件
    processing_tasks.SidecarTask{},         // 4. Sidecar XMP
    processing_tasks.ProcessPhotoTask{},    // 5. 照片处理
    processing_tasks.ProcessVideoTask{},    // 6. 视频处理
    FaceDetectionTask{},            // 7. 人脸检测
    BlurhashTask{},                 // 8. BlurHash
    ExifTask{},                     // 9. EXIF 解析
    VideoMetadataTask{},            // 10. 视频元数据
    cleanup_tasks.MediaCleanupTask{},      // 11. 缓存清理
}
```

**注册顺序即执行顺序**，每个生命周期钩子按此列表依次调用。

### 14.2 ScannerTask 接口定义

`api/scanner/scanner_task/scanner_task.go:16-36` 定义了 7 个生命周期钩子：

```go
type ScannerTask interface {
    BeforeScanAlbum(ctx TaskContext) (TaskContext, error)
    AfterScanAlbum(ctx TaskContext, changedMedia []*models.Media, albumMedia []*models.Media) error
    MediaFound(ctx TaskContext, fileInfo fs.FileInfo, mediaPath string) (skip bool, err error)
    AfterMediaFound(ctx TaskContext, media *models.Media, newMedia bool) error
    BeforeProcessMedia(ctx TaskContext, mediaData *EncodeMediaData) (TaskContext, error)
    ProcessMedia(ctx TaskContext, mediaData *EncodeMediaData, mediaCachePath string) ([]*MediaURL, error)
    AfterProcessMedia(ctx TaskContext, mediaData *EncodeMediaData, updatedURLs []*MediaURL, mediaIndex int, mediaTotal int) error
}
```

`ScannerTaskBase`（`api/scanner/scanner_task/scanner_task_base.go`）提供所有钩子的空实现，子任务只需覆盖自己关心的钩子。

### 14.3 生命周期钩子触发时机

```
ScanAlbum()
│
├─ 1. BeforeScanAlbum()           ← 扫描相册前
│     └─ IgnorefileTask: 编译 .photoviewignore 规则
│
├─ 2. findMediaForAlbum()         ← 遍历目录发现媒体
│     ├─ 对每个文件: MediaFound()
│     │   ├─ IgnorefileTask:     匹配 .photoviewignore → skip?
│     │   └─ CounterpartFilesTask: JPEG 有同名 RAW → skip?
│     │
│     └─ ScanMedia() 入库后: AfterMediaFound()
│         ├─ NotificationTask:   广播「发现新媒体」通知（节流 500ms）
│         ├─ ExifTask:           解析 EXIF 并保存（仅 newMedia）
│         ├─ VideoMetadataTask:  解析视频元数据（仅 newMedia + 视频类型）
│         └─ SidecarTask:        查找 .xmp sidecar（仅 newMedia + 照片 + 非Web兼容）
│
├─ 3. scanMedia() 对每个媒体:     ← 处理媒体
│     ├─ BeforeProcessMedia()
│     │   └─ CounterpartFilesTask: 设置 CounterpartPath
│     │
│     ├─ ProcessMedia()          ← 生成缓存文件
│     │   ├─ SidecarTask:        sidecar 变更 → 重新生成 high-res + thumbnail
│     │   ├─ ProcessPhotoTask:   生成 thumbnail + high-res + original
│     │   └─ ProcessVideoTask:   生成 video-web + video-thumbnail + original
│     │
│     └─ AfterProcessMedia()
│         ├─ NotificationTask:   广播处理进度
│         ├─ FaceDetectionTask:  检测人脸（仅照片 + 有更新）
│         └─ BlurhashTask:       生成 BlurHash（仅 thumbnail 有更新）
│
└─ 4. AfterScanAlbum()            ← 扫描相册后
      ├─ NotificationTask:        广播「处理完成」通知
      └─ MediaCleanupTask:        清理已消失的媒体缓存
```

### 14.4 各子任务详细说明

#### ① NotificationTask — 通知广播

| 钩子 | 行为 |
|------|------|
| `AfterMediaFound` | 发现新媒体时广播通知（节流 500ms） |
| `AfterProcessMedia` | 处理进度通知（百分比） |
| `AfterScanAlbum` | 扫描完成通知 |

**初始化**：`NewNotificationTask()` 创建带节流器和唯一 albumKey 的实例。
**注意**：因为注册在列表第一位，`AfterMediaFound` 会在 EXIF/视频元数据解析之前执行。

#### ② IgnorefileTask — 忽略文件过滤

| 钩子 | 行为 |
|------|------|
| `BeforeScanAlbum` | 编译 `.photoviewignore` 规则到 TaskContext |
| `MediaFound` | 匹配文件名，命中则 skip=true |

`.photoviewignore` 语法与 `.gitignore` 一致（使用 `go-gitignore` 库）。

#### ③ CounterpartFilesTask — 配套文件处理

| 钩子 | 行为 |
|------|------|
| `MediaFound` | JPEG 有同名 RAW → skip；RAW + 禁用 RAW 处理 → skip |
| `BeforeProcessMedia` | 为 RAW 文件查找同名 JPEG，设置 `CounterpartPath` |

#### ④ SidecarTask — XMP Sidecar 处理

| 钩子 | 行为 |
|------|------|
| `AfterMediaFound` | 新 RAW 文件查找 `.xmp` sidecar，记录路径和 MD5 哈希 |
| `ProcessMedia` | 检测 sidecar 变更（哈希不同或被删除），触发重新生成 high-res + thumbnail |

**Sidecar 变更检测**：通过 MD5 哈希对比，如果 sidecar 文件新增/修改/删除，重新编码对应的 JPEG。
**安全措施**：重新编码前将原文件重命名为 `.hold`，失败后恢复。

#### ⑤ ProcessPhotoTask — 照片处理

| 钩子 | 行为 |
|------|------|
| `ProcessMedia` | 生成 high-res（非 Web 兼容时）+ original + thumbnail |

（详见第二章「照片处理核心流程」）

#### ⑥ ProcessVideoTask — 视频处理

| 钩子 | 行为 |
|------|------|
| `ProcessMedia` | 生成 video-web（非 Web 兼容时）+ video-thumbnail + original |

（详见第九章「视频解码与转码栈」）

#### ⑦ FaceDetectionTask — 人脸检测

| 钩子 | 行为 |
|------|------|
| `AfterProcessMedia` | 照片类型 + 有更新 → 调用 `GlobalFaceDetector.DetectFaces()` |

**执行条件**：`len(updatedURLs) > 0` 且 `media.Type == MediaTypePhoto` 且 `GlobalFaceDetector != nil`

**DetectFaces 流程**（`api/scanner/face_detection/face_detector_impl.go:92-128`）：
```
1. 加载媒体的 MediaURL，找到 PhotoThumbnail
2. 获取缩略图磁盘路径
3. 加锁（mutex.Lock）
4. face.Recognizer.RecognizeFile(thumbnailPath) → 识别所有人脸
5. 解锁
6. 对每个人脸: classifyFace()
   ├─ classifyDescriptor() → 与已有样本对比，阈值 0.2
   ├─ 无匹配 → 创建新 FaceGroup
   └─ 有匹配 → 追加到已有 FaceGroup
7. 更新内存中的 faceDescriptors / faceGroupIDs / imageFaceIDs
```

**线程安全**：`faceDetector.mutex` 保护 `Recognizer` 的调用和样本数据更新。Recognizer 本身不是并发安全的。

**禁用方式**：
- 环境变量：`PHOTOVIEW_DISABLE_FACE_RECOGNITION=1`
- 编译标签：`-tags no_face_detection`（使用 `face_detector_shim.go`）

#### ⑧ BlurhashTask — BlurHash 生成

| 钩子 | 行为 |
|------|------|
| `AfterProcessMedia` | thumbnail 有更新 → 生成 BlurHash 字符串 |

**执行条件**：`updatedURLs` 中包含 `PhotoThumbnail` 或 `VideoThumbnail`，或 `media.Blurhash == nil`

**生成流程**（`api/scanner/scanner_tasks/blurhash_task.go:60-87`）：
```
1. 从 MediaURL 获取缩略图路径
2. Go 标准库 image.Decode() 解码图片
3. blurhash.Encode(4, 3, imageData) → 生成 4×3 分量的 BlurHash
4. 保存到 media.Blurhash 字段
```

**参数**：`componentX=4, componentY=3`，产生约 30 字符的哈希字符串。

#### ⑨ ExifTask — EXIF 元数据解析

| 钩子 | 行为 |
|------|------|
| `AfterMediaFound` | 仅 newMedia → 解析 EXIF 并保存到数据库 |

**SaveEXIF 流程**（`api/scanner/scanner_tasks/exif_task.go:32-73`）：
```
1. 检查 media.ExifID 是否已存在 → 已有则跳过
2. exif.Parse(media.Path) → 调用 exiftool 解析
3. Replace EXIF 关联到 media
4. 如果 EXIF.DateShot != media.DateShot → 更新 media.DateShot
```

**exif.Parse()** 使用全局单例 `globalExifParser`，受 `sync.Mutex` 保护（`api/scanner/externaltools/exif/exif.go:43-44`）。

#### ⑩ VideoMetadataTask — 视频元数据解析

| 钩子 | 行为 |
|------|------|
| `AfterMediaFound` | 仅 newMedia + 视频类型 → 解析视频元数据 |

**ScanVideoMetadata 流程**（`api/scanner/scanner_tasks/video_metadata_task.go:35-84`）：
```
1. ffprobe.ProbeURL() → 读取视频流信息
2. 提取: 分辨率、时长、编码、帧率、码率、色彩配置、音频信息
3. 创建 VideoMetadata 记录
4. 保存到数据库（media.VideoMetadata 关联）
```

#### ⑪ MediaCleanupTask — 缓存清理

| 钩子 | 行为 |
|------|------|
| `AfterScanAlbum` | 清理已消失媒体的数据库记录和缓存文件 |

（详见第十二章「媒体缓存清理与磁盘空间回收」）

### 14.5 任务间的数据依赖

```
MediaFound 阶段:
  IgnorefileTask.skip ──→ 决定是否继续
  CounterpartFilesTask.skip ──→ 决定是否继续
    ↓ (不 skip 才进入后续)

AfterMediaFound 阶段:
  ExifTask ──→ 更新 media.DateShot（影响排序）
  VideoMetadataTask ──→ 写入 media.VideoMetadata
  SidecarTask ──→ 写入 media.SideCarPath / SideCarHash

BeforeProcessMedia 阶段:
  CounterpartFilesTask ──→ 设置 mediaData.CounterpartPath

ProcessMedia 阶段:
  SidecarTask ──→ sidecar 变更时重建 high-res + thumbnail
  ProcessPhotoTask ──→ 使用 CounterpartPath 决定解码源
  ProcessVideoTask ──→ 使用 VideoMetadata 确定截帧位置

AfterProcessMedia 阶段:
  FaceDetectionTask ──→ 依赖 thumbnail 已生成
  BlurhashTask ──→ 依赖 thumbnail 已生成
```

**关键依赖**：
- FaceDetection 和 BlurHash **必须**在 ProcessPhoto/ProcessVideo 之后执行（因为依赖缩略图文件）
- ExifTask 在 AfterMediaFound 阶段执行，早于 ProcessMedia，但只读不写缓存文件
- SidecarTask 跨越两个阶段：AfterMediaFound 记录信息，ProcessMedia 检测变更触发重建

---

## 十五、DB Schema 升级流程

### 15.1 迁移入口

`api/server.go:51-53`:
```go
if err := database.MigrateDatabase(db); err != nil {
    log.Panicf("Could not migrate database: %s\n", err)
}
```

每次服务启动时执行 `MigrateDatabase()`，在 `SetupDatabase()` 连接成功后、业务逻辑开始前运行。

### 15.2 MigrateDatabase 调用链

`api/database/database.go:173-205`:

```
MigrateDatabase(db)
  │
  ├─ 1. db.SetupJoinTable(&User{}, "Albums", &UserAlbums{})
  │     设置 user_albums 为多对多连接表
  │
  ├─ 2. db.AutoMigrate(database_models...)
  │     自动同步所有模型到数据库 schema
  │
  ├─ 3. 删除废弃列: media.date_imported（v2.1.0）
  │     if HasColumn("date_imported") → DropColumn
  │
  ├─ 4. 迁移 EXIF 字段类型（v2.3.0）
  │     migrateExifFields(db)
  │     ├─ exposure: string "1/100" → float64 0.01
  │     └─ flash: string "Fired" → int 1
  │
  ├─ 5. 修正无效 GPS 数据
  │     migrations.MigrateForExifGPSCorrection(db)
  │     └─ 清除 |经纬度| > 90 的无效记录
  │
  └─ 6. 删除废弃列: site_info.thumbnail_method（v2.5.0）
        if HasColumn("thumbnail_method") → DropColumn
```

### 15.3 AutoMigrate 模型注册表

`api/database/database.go:154-171` 定义了所有需要自动迁移的模型：

```go
var database_models []interface{} = []interface{}{
    &models.User{},
    &models.AccessToken{},
    &models.SiteInfo{},
    &models.Media{},
    &models.MediaURL{},
    &models.Album{},
    &models.MediaEXIF{},
    &models.VideoMetadata{},
    &models.ShareToken{},
    &models.UserMediaData{},
    &models.UserAlbums{},
    &models.UserPreferences{},
    &models.FaceGroup{},
    &models.ImageFace{},
}
```

GORM 的 `AutoMigrate` 会：
- 创建不存在的表
- 添加不存在的列
- 修改列类型（如果 GORM tag 变更）
- **不会**删除列或修改列名

### 15.4 版本迁移详解

#### v2.1.0 — 删除 date_imported

```go
if db.Migrator().HasColumn(&models.Media{}, "date_imported") {
    db.Migrator().DropColumn(&models.Media{}, "date_imported")
}
```

旧的 `date_imported` 被 `Media.CreatedAt`（GORM 内置）替代。

#### v2.3.0 — EXIF 字段类型转换

**exposure 字段**（`api/database/migration_exif.go:105-158`）：
```
1. 清空空字符串: UPDATE media_exif SET exposure = NULL WHERE exposure = ''
2. 批量转换 "1/100" → 0.01:
   SELECT WHERE exposure LIKE '%/%'
   → 按 100 条分批处理
   → 解析分子/分母 → 计算小数
   → Save 回数据库
3. AutoMigrate → 修改列类型为 double
```

**flash 字段**（`api/database/migration_exif.go:160-214`）：
```
1. 检查 information_schema.columns.data_type
   → 如果已是 bigint → 跳过
2. 清空空字符串: UPDATE media_exif SET flash = NULL WHERE flash = ''
3. 批量转换 "Fired" → 1:
   → 使用 flashDescriptions 映射表（0x0~0x5F）
   → 按 100 条分批处理
   → Save 回数据库
```

#### GPS 修正（`api/database/migrations/exif_invalid_gps.go`）：
```go
tx.Model(&models.MediaEXIF{}).
    Where("ABS(gps_longitude) > ?", 90).
    Or("ABS(gps_latitude) > ?", 90).
    Updates(map[string]interface{}{
        "gps_latitude":  nil,
        "gps_longitude": nil,
    })
```

直接将无效 GPS（经度 > 90°或纬度 > 90°）置为 NULL。

#### v2.5.0 — 删除 thumbnail_method

```go
if db.Migrator().HasColumn(&models.SiteInfo{}, "thumbnail_method") {
    db.Migrator().DropColumn(&models.SiteInfo{}, "thumbnail_method")
}
```

缩略图降采样方法选项被移除，统一使用 ImageMagick 默认算法。

### 15.5 迁移特点

1. **无版本号追踪**：不像 Flyway/goose 那样有 migration 版本表，而是通过 `HasColumn()` 和 `ColumnTypes()` 检测当前 schema 状态
2. **幂等设计**：每次启动都执行，已迁移的步骤自动跳过
3. **先迁移再启动**：MigrateDatabase 在所有业务逻辑之前运行
4. **错误处理宽松**：迁移失败只打印日志不中断启动（除 AutoMigrate 外）
5. **分批处理**：大数据量迁移使用 `FindInBatches`（每批 100 条）避免内存溢出
