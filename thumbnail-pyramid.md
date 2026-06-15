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

## 六、关键代码索引

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
