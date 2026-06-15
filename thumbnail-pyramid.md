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
