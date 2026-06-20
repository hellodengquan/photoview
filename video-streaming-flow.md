# Photoview 视频流式播放与缓存流程分析

## 概述

**重要说明**：Photoview **并未使用 HLS (HTTP Live Streaming) 或 DASH 等基于分片 (m3u8/chunk) 的流媒体协议。

它采用的是更简单直接的方案：**整文件转码 + 浏览器原生 `<video>` 标签 + HTTP Range 请求实现的渐进式下载/伪流式播放。

---

## 一、核心架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                         前端 (React)                           │
│  ┌──────────────────────────────────────────────────┐          │
│  │  ProtectedVideo 组件                          │          │
│  │  ┌──────────────────────────────────────┐    │          │
│  │  │  <video controls>                 │    │          │
│  │  │    <source src="/api/video/{name}" │    │          │
│  │  │            type="video/mp4"     │    │          │
│  │  │  </video>                    │    │          │
│  │  └──────────────────────────────────────┘    │          │
│  └──────────────────────────────────────────────────┘          │
│                         │                                  │
│                         │ HTTP Range 请求 (分段拉取)             │
└─────────────────────────┼──────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                       后端 (Go + GORM)                      │
│                                                           │
│  GraphQL 查询获取元数据 → Media.videoWeb.url                          │
│         │                                               │
│         ▼                                               │
│  /api/video/{name} 路由                                   │
│         │                                               │
│         ├─ 鉴权 (authenticateMedia)                        │
│         ├─ 查询 MediaURL (purpose=video-web)               │
│         ├─ 计算缓存路径                                   │
│         ├─ 检查缓存文件是否存在                               │
│         │   ├─ 存在 → 直接 ServeFile (支持 Range)         │
│         │   └─ 不存在 → 触发转码 (ProcessSingleMedia)      │
│         │       └─ FFmpeg 编码 MP4 → 存入缓存            │
│         └─ http.ServeFile 返回 (天然支持 HTTP/206 部分内容)     │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    文件系统缓存层                              │
│  {PHOTOVIEW_MEDIA_CACHE}/                                 │
│      {albumID}/                                           │
│          {mediaID}/                                         │
│              web_video_{原文件名}_{token}.mp4          │
│              video_thumb_{原文件名}_{token}.jpg            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、后端处理流程详解

### 2.1 路由注册 (api/server.go:94-98

```go
photoRouter := endpointRouter.PathPrefix("/photo").Subrouter()
routes.RegisterPhotoRoutes(db, photoRouter)

videoRouter := endpointRouter.PathPrefix("/video").Subrouter()
routes.RegisterVideoRoutes(db, videoRouter)
```

视频请求走 `/api/video/{name}` 路径，图片走 `/api/video/{name}`，两者是两个独立的子路由。

### 2.2 视频请求处理 (api/routes/videos.go:23-125

`handleVideoRequest` 函数的完整流程：

**阶段 1：数据库查询与鉴权
```go
// 通过 mediaName 查找 MediaURL 记录
WHERE media_urls.media_name = ? AND media_urls.purpose = 'video-web'
```

鉴权两种方式：
- 已登录用户：校验用户是否拥有该相册权限 `user.OwnsAlbum()
- 未登录用户：通过 URL query 参数 `?token=` 分享令牌

**阶段 2：计算缓存路径 (api/routes/videos.go:67-79

```go
cachedPath = getCachePathFn(albumID, mediaID, mediaURL.MediaName)
```

缓存路径生成函数 `generateCacheFilename` (api/routes/videos.go:127-129)：
```go
func generateCacheFilename(albumID, mediaID int, filename string) string {
    return path.Join(utils.MediaCachePath(), strconv.Itoa(albumID), strconv.Itoa(mediaID), filename)
}
```

**阶段 3：缓存缺失时触发转码 (api/routes/videos.go:81-119

```
如果缓存文件不存在：
└─→ 调用 processSingleMediaFn() → scanner.ProcessSingleMedia()
    └─→ 在数据库事务内执行完整的媒体处理管线
```

**阶段 4：返回文件 (api/routes/videos.go:121-124

```go
w.Header().Set("Cache-Control", "private, max-age=31536000, immutable")
w.Header().Set("Content-Type", mediaURL.ContentType)
http.ServeFile(w, r, cachedPath)
```

`http.ServeFile` 是 Go 标准库函数，它会**自动解析并响应 HTTP Range 请求，返回 206 Partial Content 响应码，实现浏览器端的拖拽进度条可以正常工作。

### 2.3 缓存路径管理 (api/utils/media_cache.go:12-73

```
MediaCachePath() 优先级：
1. 测试环境 testCachePath (测试时设置
2. PHOTOVIEW_MEDIA_CACHE 环境变量
3. 默认值: "./media_cache"
```

目录结构自动逐级创建：
```
{media_cache}/
├── {albumID}/           # 相册 ID
│   └── {mediaID}/    # 媒体 ID
│   │   ├── web_video_xxx.mp4       # 转码后的 web 视频
│   │   └── video_thumb_xxx.jpg   # 视频缩略图
```

---

## 三、视频转码流程

### 3.1 触发时机

有两个：
1. **后台扫描阶段（扫描器扫描时自动触发转码）
2. **按需转码（用户首次访问时缓存不存在时实时触发）

### 3.2 ProcessVideoTask (api/scanner/scanner_tasks/processing_tasks/process_video_task.go:20-204

处理逻辑分三种情况：

**情况 A：原视频已是 Web 兼容格式 (mp4, webm, ogg, mpeg)
```
videoType.IsWebCompatible() == true
└─→ 不转码，直接将原视频路径作为 MediaOriginal，不产生缓存文件
```

Web 兼容格式定义 (api/scanner/media_type/media_type.go:57-62

**情况 B：原视频非 Web 兼容 (avi, mkv, mov, wmv 等
```
!videoType.IsWebCompatible()
└─→ 调用 FFmpeg 转码为 MP4
    └─→ 输出到缓存目录
```

FFmpeg 转码参数 (api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:74-96

```bash
ffmpeg \
  -i inputPath \
  -vcodec h264 (或硬件加速编码 h264_qsv / h264_vaapi / h264_nvenc) \
  -acodec aac \
  -vf scale='min(1080,iw)':'min(1080,ih)':force_original_aspect_ratio=decrease:force_divisible_by=2 \
  -movflags +faststart+use_metadata_tags \
  outputPath
```

关键点：
- **`-movflags +faststart`：将 moov atom 前置，让浏览器可以快速启动播放无需等整个文件下载完成
- 分辨率限制在 1080p
- 支持硬件加速编码（qsv/vaapi/nvenc）

**情况 C：生成视频缩略图
```
在视频 25% 时长位置截取一帧
└─→ 缩放到最大 1024px
    └─→ 保存为 JPG
```

缩略图参数 (api/scanner/media_encoding/executable_worker/ffmpeg_cli.go:98-121
```bash
ffmpeg \
  -ss {duration*0.25} \
  -i inputPath \
  -vframes 1 \
  -an \
  -vf scale='min(1024,iw)':'min(1024,ih):force_original_aspect_ratio=decrease:force_divisible_by=2 \
  outputPath
```

### 3.3 MediaURL 数据模型 (api/graphql/models/media.go:106-147

```go
type MediaURL struct {
    MediaID     int
    MediaName   string       // 缓存文件名
    Width       int
    Height      int
    Purpose     MediaPurpose // "video-web" | "video-thumbnail" | "original"
    ContentType string       // "video/mp4" 等
    FileSize    int64
}
```

URL 生成逻辑 (api/graphql/models/media.go:118-128)：
```go
func (p *MediaURL) URL() string {
    if p.Purpose != VideoWeb {
        return /api/photo/{name}
    } else {
        return /api/video/{name}
    }
}
```

---

## 四、前端播放流程

### 4.1 GraphQL 查询获取视频元数据

Resolver (api/graphql/resolvers/media.go:36-43

```graphql
query {
  media(id: $id) {
    thumbnail { url }
    videoWeb { url }
  }
}
```

DataLoader 加载器 (api/dataloader/mediaURLLoader.go:69-82

```
NewVideoWebMediaURLLoader
  WHERE purpose IN ('video-web', 'original')
  ORDER BY CASE
    WHEN 'original' THEN 0
    WHEN 'video-web' THEN 1
  → VideoWeb 存在时 VideoWeb，否则 fallback 到 original
```

### 4.2 ProtectedVideo 组件 (ui/src/components/photoGallery/ProtectedMedia.tsx:169-203

```tsx
export const ProtectedVideo = ({ media }: ProtectedVideoProps) => {
  return (
    <video
      controls
      crossOrigin="use-credentials"
      poster={getProtectedUrl(media.thumbnail?.url)}
    >
      <source src={getProtectedUrl(media.videoWeb.url)} type="video/mp4" />
    </video>
  )
}
```

关键点：
- 使用浏览器原生 `<video>` 标签，**无第三方播放器库
- `crossOrigin="use-credentials"`：携带 Cookie 进行鉴权
- `poster`：视频缩略图作为视频未播放前显示封面
- `type="video/mp4`：固定 mp4 类型

### 4.3 URL 鉴权参数注入 (ui/src/components/photoGallery/ProtectedMedia.tsx:13-25

```ts
const getProtectedUrl = (url?: string) => {
  // 分享页面时自动携带 token
  if (location.pathname.match(/^\/share\/([\d\w]+)
  → imgUrl.searchParams.set('token', token)
}
```

---

## 五、HTTP 流式播放实现原理

### 5.1 为什么不需要 HLS？

Photoview 选择整文件 MP4 + Range 的原因：

| 方案 | 优点 | 缺点
|------|------|
| **整文件 MP4 + Range 请求 | 1. 实现简单，依赖 Go 标准库 `http.ServeFile` 自动处理
2. 兼容性好，所有浏览器支持
3. 可 seek (拖拽进度条 | 1. 不支持自适应码率
2. 长视频启动稍慢（但有 faststart 优化后大幅缓解
| HLS/DASH | 1. 自适应码率
2. 适合直播/长视频 | 1. 需要额外的 ffmpeg 生成 m3u8 + 大量 ts 分片
2. 浏览器需 hls.js 等播放器库
3. 实现复杂度高
|

### 5.2 HTTP Range 请求工作流程

```
浏览器                            服务器
  │                                 │
  │  GET /api/video/xxx.mp4          │
  │  Range: bytes=0-                   │  (初始请求，试探服务器是否支持 Range)
  │─────────────────────────────────▶│
  │                                 │
  │  206 Partial Content          │
  │  Accept-Ranges: bytes               │
  │  Content-Length: 10485760       │
  │  Content-Range: bytes 0-10485759/10485760
  │◀───────────────────────────────
  │                                 │
  │  (用户拖拽到 50% 位置)              │
  │  GET /api/video/xxx.mp4          │
  │  Range: bytes=5242880-           │
  │────────────────────────────────▶│
  │                                 │
  │  206 Partial Content          │
  │  Content-Range: bytes 5242880-10485759/10485760
  │◀───────────────────────────────
```

Go `http.ServeFile` 内部自动处理：
1. 解析 `Range` 请求头
2. 打开文件，`Seek` 到对应偏移
3. 设置 206 状态码和相应响应头
4. 流式读取并写入数据

---

## 六、缓存失效与重建

### 6.1 视频缩略图缓存校验 (api/scanner/scanner_tasks/processing_tasks/process_video_task.go:170-201

```go
if videoThumbnailURL != nil {
    thumbImagePath := path.Join(mediaCachePath, videoThumbnailURL.MediaName)
    if _, err := os.Stat(thumbImagePath); os.IsNotExist(err) {
        // 数据库有记录但文件丢失 → 重新生成缩略图
        // 更新数据库中的宽高和文件大小
    }
}
```

### 6.2 视频文件缓存校验 (api/routes/videos.go:81-119

```
请求到达时检查缓存文件
├─ 存在 → 直接返回
└─ 不存在 → 调用 ProcessSingleMedia 重新转码
    注意：此处仅检查文件存在性，不校验文件完整性
```

---

## 七、关键代码索引

| 模块 | 文件路径 | 关键函数/结构 |
|------|----------|---------------|
| 路由 | api/routes/videos.go | `handleVideoRequest`, `RegisterVideoRoutes` |
| 缓存路径 | api/utils/media_cache.go | `CachePathForMedia`, `MediaCachePath` |
| 视频转码 | api/scanner/scanner_tasks/processing_tasks/process_video_task.go | `ProcessVideoTask.ProcessMedia` |
| FFmpeg 封装 | api/scanner/media_encoding/executable_worker/ffmpeg_cli.go | `FfmpegCli.EncodeMp4` |
| 数据模型 | api/graphql/models/media.go | `Media`, `MediaURL`, `MediaURL.URL()`, `MediaURL.CachedPath()` |
| GraphQL | api/graphql/resolvers/media.go | `mediaResolver.VideoWeb` |
| DataLoader | api/dataloader/mediaURLLoader.go | `NewVideoWebMediaURLLoader` |
| 前端播放器 | ui/src/components/photoGallery/ProtectedMedia.tsx | `ProtectedVideo`, `getProtectedUrl` |
| 前端展示 | ui/src/components/photoGallery/presentView/PresentMedia.tsx | `PresentMedia` |
| 媒体类型 | api/scanner/media_type/media_type.go | `IsWebCompatible()` |
| 鉴权 | api/routes/authenticate_routes.go | `authenticateMedia` |

---

## 八、FFmpeg 转码失败时的降级播放路径分析

### 8.1 转码失败的直接处理逻辑

**ProcessVideoTask 内部无降级** (api/scanner/scanner_tasks/processing_tasks/process_video_task.go:95-98)：

```go
err = executable_worker.Ffmpeg.EncodeMp4(video.Path, webVideoPath)
if err != nil {
    // 直接返回错误，无任何 fallback 逻辑
    return []*models.MediaURL{}, errors.Wrapf(err, "could not encode mp4 video (%s)", video.Path)
}
```

转码失败时：
- 整个事务回滚
- 不创建 `VideoWeb` 记录到数据库
- 错误向上冒泡到 `ProcessSingleMedia`
- 最终在路由层返回 500 Internal Server Error (api/routes/videos.go:101-107)

### 8.2 DataLoader 层面的隐含 fallback

`NewVideoWebMediaURLLoader` (api/dataloader/mediaURLLoader.go:69-82) 的查询逻辑：

```go
WHERE purpose IN ('video-web', 'original')
ORDER BY media_id ASC, CASE
    WHEN 'original' THEN 0
    WHEN 'video-web' THEN 1
END ASC
```

**注意**：这个排序逻辑写反了！
- `original` 的排序权重是 0，`video-web` 是 1
- `ORDER BY ASC` 意味着权重小的排前面
- 所以当两者都存在时，**`original` 会排在 `video-web` 前面**
- 但注释写的是 "VideoWeb consistently wins ordering when both exist"，与实际代码相反

不过这不影响 fallback 逻辑：
- 当 `video-web` 不存在但 `original` 存在时，DataLoader 会返回 `original`
- 这是唯一的降级路径

### 8.3 降级路径的触发条件

```
降级播放的完整前置条件：
├─ 1. 原视频必须是 Web 兼容格式 (IsWebCompatible() == true)
│   ├─ mp4, webm, ogg, mpeg 四种
│   └─ 这种情况下 ProcessVideoTask 会创建 MediaOriginal 记录
├─ 2. VideoWeb 记录不存在
│   ├─ 可能是转码失败（但 Web 兼容格式不会走转码分支）
│   ├─ 或转码功能被禁用 (PHOTOVIEW_DISABLE_VIDEO_ENCODING=1)
│   └─ 或转码尚未完成
└─ 3. 必须通过 GraphQL 查询走 DataLoader 路径
    └─ 直接访问 /api/video/{name} 路由只查 VideoWeb，不查 Original，无法降级
```

**路由层的限制** (api/routes/videos.go:34)：
```go
// 硬编码只查 video-web，无法降级到 original
WHERE media_urls.media_name = ? AND media_urls.purpose = 'video-web'
```

这意味着：
- 视频 URL 由 `MediaURL.URL()` 生成，Original 走 `/api/photo/`，VideoWeb 走 `/api/video/`
- 如果 DataLoader fallback 到了 Original，前端拿到的 URL 是 `/api/photo/{name}`
- 这个 URL 会走图片路由 (`/api/photo/`)，而图片路由也支持 `ServeFile` 和 Range 请求
- 所以从技术上讲这个降级是能工作的，但路径非常隐晦

### 8.4 实际降级效果评估

| 场景 | 能否降级播放 | 原因 |
|------|-------------|------|
| 原视频是 mp4，从未转码过 | ✅ 可以 | DataLoader 返回 Original，URL 走 /api/photo/ |
| 原视频是 avi，转码失败 | ❌ 不能 | 没有 Original 记录，DataLoader 返回 null |
| 原视频是 mkv，转码功能被禁用 | ❌ 不能 | 没有 Original 记录 |
| 原视频是 webm，VideoWeb 转码中 | ✅ 可以 | 临时 fallback 到 Original |
| 直接访问 /api/video/{name} 路由 | ❌ 不能 | 路由层硬编码只查 VideoWeb |

---

## 九、HLS.js 与自适应码率 ABR 分析

### 9.1 前端技术栈确认 (ui/package.json)

检查依赖项，**没有任何视频播放器库**：
```json
{
  "dependencies": {
    "react": "^18.2.0",
    "styled-components": "^5.3.5",
    // ... 其他依赖
    // ❌ 没有 hls.js
    // ❌ 没有 shaka-player
    // ❌ 没有 dash.js
    // ❌ 没有 video.js
  }
}
```

### 9.2 前端播放实现确认

`ProtectedVideo` 组件 (ui/src/components/photoGallery/ProtectedMedia.tsx:186-202)：

```tsx
<video
  controls
  crossOrigin="use-credentials"
  poster={getProtectedUrl(media.thumbnail?.url)}
>
  <source src={getProtectedUrl(media.videoWeb.url)} type="video/mp4" />
</video>
```

100% 原生 `<video>` 标签，**type 固定为 `video/mp4`**，没有 `application/x-mpegURL` 或 `application/dash+xml`。

### 9.3 后端是否有 HLS 生成逻辑

全局搜索结果：
- 无 `hls`、`m3u8`、`ts`、`segment`、`chunk` 等关键词（除了 SVG logo 中的无关文本）
- 无 HLS 播放列表生成代码
- 无 ts 分片生成代码
- FFmpeg 调用只有 `EncodeMp4` 和 `EncodeVideoThumbnail` 两个函数

### 9.4 自适应码率 ABR 可能性评估

**结论：完全不支持**

ABR 自适应码率需要三个前提条件，Photoview 一个都不满足：

| ABR 前提 | Photoview 现状 |
|---------|---------------|
| 多码率转码输出 | ❌ 只转一个 1080p MP4，无 480p/720p/1080p 多版本 |
| 流媒体协议（HLS/DASH） | ❌ 只有整文件 MP4 |
| 支持 ABR 的播放器 | ❌ 只有原生 `<video>` |

即使未来想加 ABR，改动量也很大：
- FFmpeg 命令要改成输出多码率 + HLS 分片
- 后端要新增 m3u8 播放列表路由
- 前端要引入 hls.js 或 shaka-player
- 缓存目录结构要调整（按码率分子目录存 ts 分片）

---

## 十、LRU 缓存与 ts 分片 prefetch 机制分析

### 10.1 LRU 缓存的真实用途

`graphql_endpoint.go:41,44` 中有两处 LRU：

```go
import "github.com/99designs/gqlgen/graphql/handler/lru"

// GraphQL 查询 AST 缓存，容量 1000
graphqlServer.SetQueryCache(lru.New[*ast.QueryDocument](1000))

// 持久化查询缓存，容量 100
graphqlServer.Use(extension.AutomaticPersistedQuery{
    Cache: lru.New[string](100),
})
```

**这是 gqlgen 框架内置的 GraphQL 查询缓存**，与媒体文件缓存完全无关。

依赖 `hashicorp/golang-lru/v2` 也是 gqlgen 的间接依赖，不是 Photoview 自己用的。

### 10.2 媒体文件缓存机制

`api/utils/media_cache.go` 中的缓存实现：

```go
// 只负责创建目录，不管理缓存生命周期
func CachePathForMedia(albumID int, mediaID int) (string, error) {
    // 创建 root/albumID/mediaID 三级目录
    // 无容量限制
    // 无 LRU 淘汰
    // 无热度统计
    // 无过期时间
}
```

媒体缓存是**纯文件系统存储**：
- 无限增长，直到磁盘满
- 唯一的清理逻辑在 `cleanup_media.go` 中，但也是**存在性检查**，不是 LRU

`cleanup_media.go` 逻辑概要：
```go
// 扫描数据库中的 MediaURL 记录
// 检查对应的缓存文件是否存在
// 如果数据库有记录但文件不存在 → 重新生成
// 不删除任何文件，不做容量控制
```

### 10.3 ts 分片与 prefetch 机制

**ts 分片：不存在**
- 没有 HLS 就没有 ts 分片概念
- 缓存目录里只有 `.mp4` 和 `.jpg` 两种文件

**prefetch 机制：不存在**
- 前端无预加载逻辑（原生 `<video>` 只有 `preload` 属性，默认为 "metadata"）
- 后端无热度统计，无法判断哪些分片"热度低于阈值"
- 连热度统计都没有，自然不可能基于热度触发 prefetch

### 10.4 缓存策略总结

| 特性 | 现状 |
|------|------|
| LRU 淘汰 | ❌ 无，媒体缓存永不自动删除 |
| 容量限制 | ❌ 无，写满磁盘为止 |
| 热度统计 | ❌ 无 |
| 预加载/prefetch | ❌ 无 |
| ts 分片 | ❌ 无 |
| 过期清理 | ❌ 无，只有 cleanup 任务检查存在性 |

如果需要 LRU 缓存，需要自行实现：
- 增加访问日志表记录每次访问时间
- 定时任务扫描缓存目录，计算最近最少使用的文件
- 删除超过阈值的文件

---

## 十一、fallback_original 触发后 partial 文件清理分析

### 11.1 前提澄清：没有 HLS，也就没有 HLS partial 文件

再次强调：Photoview 不使用 HLS 协议，因此不存在 `.ts` 分片、`.m3u8` 播放列表，也就没有 "partial HLS 文件" 这个概念。

本节讨论的是 **FFmpeg 转码过程中产生的不完整输出文件** 在转码失败后的命运。

### 11.2 视频转码的文件写入模式

**ProcessVideoTask 直接写入目标文件，无临时文件阶段** (api/scanner/scanner_tasks/processing_tasks/process_video_task.go:93-98)：

```go
webVideoPath := path.Join(mediaCachePath, webVideoName)
// 直接把 ffmpeg 输出导向最终路径
err = executable_worker.Ffmpeg.EncodeMp4(video.Path, webVideoPath)
if err != nil {
    // 失败直接 return，不清理 webVideoPath
    return []*models.MediaURL{}, errors.Wrapf(err, "could not encode mp4 video (%s)", video.Path)
}
```

对比 sidecar 任务的 `.hold` 安全模式 (api/scanner/scanner_tasks/processing_tasks/sidecar_task.go:99-106)：

```go
tempHighResPath := baseImagePath + ".hold"
os.Rename(baseImagePath, tempHighResPath)  // 先把旧文件备份
// ... 生成新文件 ...
if err != nil {
    os.Rename(tempHighResPath, baseImagePath)  // 失败则回滚
    return ...
}
os.Remove(tempHighResPath)  // 成功才删除备份
```

**结论**：视频转码没有使用 `.hold` 模式，FFmpeg 直接写入最终路径。

### 11.3 转码失败后残留文件的去向

```
转码失败时的文件状态：
├─ 输出文件 web_video_xxx.mp4 已存在但内容不完整
├─ 数据库中没有 VideoWeb 记录（因为事务回滚）
└─ 这个不完整的文件会永久残留在缓存目录中
```

**为什么会残留**：
1. `process_video_task.go` 失败直接 return，不做 `os.Remove` 清理
2. `scanMedia` 中数据库事务回滚，但不回滚文件系统操作
3. 没有专门的 "脏文件" 清理任务

### 11.4 现有清理机制的覆盖范围

`CleanupMedia` 函数 (api/scanner/scanner_tasks/cleanup_tasks/cleanup_media.go:17-65)：

```go
// 逻辑：找到数据库中不存在的媒体，删除其缓存目录
func CleanupMedia(db *gorm.DB, albumId int, albumMedia []*models.Media) []error {
    // 1. 查出数据库中该相册的所有媒体
    // 2. 减去磁盘上实际存在的媒体
    // 3. 对"数据库有但磁盘没了"的媒体，删除缓存目录
}
```

**这个清理不处理残留转码文件的原因**：
- 它只在**媒体本身从磁盘消失**时才清理缓存
- 不完整的转码文件所在的媒体是存在的，所以不会触发清理
- 它检查的是媒体级别的存在性，不检查单个缓存文件的完整性

### 11.5 残留文件的实际影响

| 场景 | 是否产生残留 | 残留影响 |
|------|-------------|---------|
| 后台扫描时转码失败 | ✅ 是 | 占用磁盘空间，但不影响播放（因为数据库没有记录，不会被请求） |
| 按需转码（用户访问时触发）失败 | ✅ 是 | 同上，残留但不被引用 |
| 转码过程中用户取消请求 | ✅ 可能 | context 取消时 ffmpeg 进程可能被中断，文件不完整 |
| 下次重新转码 | ✅ 覆盖 | FFmpeg 会覆盖同名文件，旧的残留被替换 |

**唯一的清理时机**：
- 该媒体被从相册中删除 → `CleanupMedia` 删除整个缓存目录
- 手动删除缓存目录

---

## 十二、ABR 切换机制与移动网络稳定性分析

### 12.1 ABR 自适应码率：完全不存在

再次确认三项核心前提全部缺失：

| ABR 前提 | Photoview 现状 | 代码证据 |
|---------|---------------|---------|
| 多码率输出 | ❌ 只有 1080p 单码率 | `ffmpeg_cli.go:84` 固定 `min(1080,iw)` |
| 流媒体协议 | ❌ 只有整文件 MP4 | 全局无 m3u8/ts 生成代码 |
| ABR 播放器 | ❌ 只有原生 `<video>` | `ProtectedMedia.tsx:193` 原生 video 标签 |

### 12.2 移动端网络适配：零代码

**前端无任何网络相关逻辑**：
- 没有 `navigator.connection.effectiveType` 检测网络类型
- 没有 `Network Information API` 使用
- 没有根据网络状况切换视频质量的逻辑
- 没有缓冲状态监听（`buffered`, `waiting` 事件）
- 没有低码率降级/高码率升级的策略

**后端也没有移动端特殊处理**：
- 不区分客户端类型返回不同码率
- 没有 User-Agent 检测
- 没有移动端专用的更低码率转码版本

### 12.3 4G 网络抖动时的表现

由于完全依赖浏览器原生 `<video>` 的行为，4G 网络抖动时：

```
网络抖动时的用户体验：
├─ 1. 缓冲耗尽 → 视频暂停，显示 loading  spinner（浏览器原生控件）
├─ 2. 继续下载 → 缓冲足够后自动恢复播放
├─ 3. 持续卡顿 → 用户只能手动暂停等待缓冲
└─ 4. 完全断网 → 播放停止，无法继续
```

**没有的优化手段**：
- ❌ 无码率自动降级（从 1080p 降到 480p）
- ❌ 无预缓冲策略调整
- ❌ 无缝切换（bitrate switching）
- ❌ 帧率自适应

### 12.4 稳定性指标：无任何监控

代码中没有以下 QoE（Quality of Experience）指标的采集：

| 指标 | 是否采集 | 备注 |
|------|---------|------|
| 缓冲事件次数 | ❌ 无 | - |
| 平均缓冲时长 | ❌ 无 | - |
| 首帧加载时间 | ❌ 无 | - |
| 码率切换次数 | ❌ 无 | （本来就没切换） |
| 播放中断率 | ❌ 无 | - |
| 视频加载失败率 | ❌ 无 | - |

前端没有任何视频播放事件的埋点或统计。

### 12.5 与真正 ABR 方案的差距

如果 Photoview 要实现 ABR，需要增加的模块：

```
ABR 实现路径：
├─ 后端
│   ├─ FFmpeg 输出多码率 (240p/360p/480p/720p/1080p)
│   ├─ 生成 HLS 或 DASH 播放列表
│   ├─ ts 分片生成与管理
│   └─ m3u8/mpd 文件路由
└─ 前端
    ├─ 引入 hls.js 或 shaka-player
    ├─ 实现 ABR 策略（带宽估计 + 缓冲区状态）
    ├─ 码率切换 UI
    └─ QoE 指标上报
```

工作量相当于重构整个视频系统。

---

## 十三、prefetch_queue 与缓存主动 evict 机制分析

### 13.1 没有 prefetch_queue

全局搜索结果：
- 没有 `prefetch` 相关代码
- 没有视频预加载队列
- 没有分片预取逻辑
- 没有预读策略

**唯一的 "queue" 是 scanner_queue** (api/scanner/scanner_queue/queue.go)：
```go
type ScannerQueue struct {
    in_progress []ScannerJob  // 正在执行的扫描任务
    up_next     []ScannerJob  // 等待执行的扫描任务
    // ...
}
```

这是**媒体扫描队列**，用于后台扫描相册和转码，与播放时的 prefetch 完全无关。

### 13.2 没有缓存容量限制

**关键发现：媒体缓存是"无限增长"模式**

`api/utils/media_cache.go` 中：
- 无 `MaxSize` / `Capacity` 配置
- 无磁盘空间检查
- 无配额管理
- 无按用户/按相册的容量限制

环境变量中也没有相关配置 (`api/utils/environment_variables.go`)：
```go
// 所有缓存相关环境变量
const (
    EnvMediaCachePath EnvironmentVariable = "PHOTOVIEW_MEDIA_CACHE"
    // ❌ 没有 PHOTOVIEW_MEDIA_CACHE_MAX_SIZE
    // ❌ 没有 PHOTOVIEW_CACHE_QUOTA
)
```

### 13.3 没有主动 evict 机制

整个代码库中：
- ❌ 没有 LRU 缓存淘汰
- ❌ 没有 LFU 缓存淘汰
- ❌ 没有 FIFO 淘汰
- ❌ 没有 TTL 过期淘汰
- ❌ 没有磁盘空间阈值触发清理

**唯一的缓存删除发生在两种情况**：

1. **媒体被删除** (cleanup_media.go:43-47)：
```go
// 当媒体文件从磁盘消失时，删除对应缓存目录
cachePath := path.Join(utils.MediaCachePath(), strconv.Itoa(int(albumId)), strconv.Itoa(int(media.ID)))
err := os.RemoveAll(cachePath)
```

2. **相册被删除** (cleanup_media.go:103-107)：
```go
// 当相册消失时，删除整个相册缓存目录
cachePath := path.Join(utils.MediaCachePath(), strconv.Itoa(int(album.ID)))
err := os.RemoveAll(cachePath)
```

这两种都是**被动删除**，不是基于容量或热度的主动 evict。

### 13.4 缓存接近满时的行为

**结论：什么都不做，直到磁盘写满**

当缓存目录所在磁盘空间不足时：
1. FFmpeg 转码会失败（因为写不出文件）
2. 缩略图生成会失败
3. 错误日志会记录失败原因
4. 但系统不会主动清理任何缓存文件

没有以下保护机制：
- ❌ 磁盘水位线告警（如 80% 警告，95% 紧急）
- ❌ 低水位自动清理
- ❌ 写前检查磁盘空间
- ❌ 缓存淘汰策略

### 13.5 热度统计与 prefetch 的缺失链条

要实现 "ts 分片热度低于阈值时触发 prefetch"，需要完整的链路：

```
prefetch 系统需要的组件：
├─ 1. 访问日志/热度统计
│   ├─ 记录每次视频请求
│   ├─ 记录请求的时间戳和范围
│   └─ 计算热度分数
├─ 2. 预取决策
│   ├─ 判断哪些分片/视频"热"
│   ├─ 判断哪些"冷"且即将被访问
│   └─ 触发预取任务
├─ 3. 预取执行队列
│   ├─ 并发控制
│   ├─ 优先级调度
│   └─ 取消机制
└─ 4. 缓存淘汰
    ├─ 容量上限
    ├─ 淘汰策略 (LRU/LFU)
    └─ 与预取的协同
```

Photoview 这四层全部缺失，连最基础的访问日志都没有。

### 13.6 相关模块总结

| 功能 | 现状 | 可能的误解 |
|------|------|-----------|
| prefetch_queue | ❌ 无 | scanner_queue 是扫描队列，不是预取队列 |
| 缓存容量限制 | ❌ 无 | 会无限增长直到磁盘满 |
| 主动 evict | ❌ 无 | 只有媒体删除时的被动清理 |
| 热度统计 | ❌ 无 | 没有访问日志 |
| 分片预取 | ❌ 无 | 连分片都没有 |
| LRU 缓存 | ❌ 无（媒体缓存） | gqlgen 的 LRU 是 GraphQL 查询缓存，与此无关 |

---

## 十四、cleanup_orphan_segments worker 触发周期分析

### 14.1 核心结论：不存在此 worker

全局代码搜索结果：
- ❌ 没有 `cleanup_orphan_segments` 函数/变量
- ❌ 没有 `orphan` 关键词（除了一个翻译文件中的无关文本）
- ❌ 没有 `segment` 清理相关逻辑
- ❌ 没有独立的后台 worker 进程（除了 scanner queue worker）

### 14.2 唯一的周期性任务：Periodic Scanner

系统中只有一个周期性任务 —— **周期性扫描器** (api/scanner/periodic_scanner/periodic_scanner.go)，但它的功能是**扫描相册媒体文件**，不是清理 orphan segments。

**触发逻辑**：

```go
// periodic_scanner.go:36-43
func getPeriodicScanInterval(db *gorm.DB) (time.Duration, error) {
    var siteInfo models.SiteInfo
    if err := db.First(&siteInfo).Error; err != nil {
        return 0, err
    }
    return time.Duration(siteInfo.PeriodicScanInterval) * time.Second, nil
}
```

**触发周期配置**：

| 配置入口 | 位置 |
|---------|------|
| 数据库字段 | `site_info.periodic_scan_interval` (单位：秒) |
| GraphQL Mutation | `setPeriodicScanInterval(interval: Int!)` |
| 前端 UI | `SettingsPage → PeriodicScanner.tsx` |
| 后端 API | `periodic_scanner.go:91-118` `ChangePeriodicScanInterval()` |
| 默认值 | 0（禁用） |

**前端配置界面** (ui/src/Pages/SettingsPage/PeriodicScanner.tsx:181-253) 支持的时间单位：
- 秒 / 分钟 / 小时 / 天 / 月

### 14.3 暴露给运维的配置方式

**方式 1：Web 管理后台**
- 登录管理员账号 → Settings → Periodic scanner
- 可视化配置，实时生效

**方式 2：GraphQL API**
```graphql
mutation {
  setPeriodicScanInterval(interval: 3600)  # 每小时
}
```

**方式 3：直接修改数据库**
```sql
UPDATE site_info SET periodic_scan_interval = 3600;
```

**没有环境变量配置**（api/utils/environment_variables.go 中无相关配置）。

### 14.4 与 "orphan segments 清理" 的差距

周期性扫描器实际做的事情：
```go
// periodic_scanner.go:159-163
case <-ticker.C:
    log.Info(nil, "Scan interval runner: Starting periodic scan")
    if err := ps.scannerQueue.AddAllToQueue(); err != nil {
        log.Error(nil, "Scan interval runner: Failed to add all users to queue", "error", err)
    }
```

它调用 `AddAllToQueue()` 触发完整的媒体扫描流程，其中会：
1. 扫描磁盘上的媒体文件
2. 处理新增/修改的文件
3. 触发 `CleanupMedia` 清理**数据库存在但磁盘不存在**的媒体

但它**不会**：
- ❌ 扫描缓存目录中的不完整文件
- ❌ 清理残留的转码失败文件
- ❌ 检查单个缓存文件的完整性
- ❌ 专门处理 orphan segments（因为根本没有 segments）

### 14.5 运维配置现状总结

| 配置项 | 是否暴露给运维 | 配置方式 |
|--------|--------------|---------|
| 扫描周期 | ✅ 是 | Web UI / GraphQL / 数据库 |
| 并发 worker 数 | ✅ 是 | Web UI / GraphQL / 数据库 |
| 缓存容量限制 | ❌ 否 | 无此功能 |
|  orphan 文件清理周期 | ❌ 否 | 无此功能 |
| 清理阈值配置 | ❌ 否 | 无此功能 |

---

## 十五、monitor_throughput 采样间隔分析

### 15.1 核心结论：不存在此监控

全局代码搜索结果：
- ❌ 没有 `monitor_throughput` 函数/变量
- ❌ 没有 `throughput` 关键词
- ❌ 没有 `sampling` / `sample` 采样相关代码
- ❌ 没有 `500ms` 硬编码间隔（唯一的 500ms 是 scanner notification throttle）

### 15.2 唯一的 500ms 相关代码

`api/scanner/scanner_queue/queue.go:102` 中有一个 500ms 的 throttling：
```go
notifyThrottle := utils.NewThrottle(500 * time.Millisecond)
```

这是**扫描进度通知的限流**，不是吞吐量监控：
```go
// queue.go:183-191
notifyThrottle.Trigger(func() {
    notification.BroadcastNotification(&models.Notification{
        Key:     globalScannerProgress,
        Type:    models.NotificationTypeMessage,
        Header:  "Scanning media",
        Content: fmt.Sprintf("%d jobs in progress\n%d jobs waiting", inProgressLength, upNextLength),
    })
})
```

目的是避免扫描进度通知过于频繁地推送到前端。

### 15.3 网络吞吐量监控的缺失链条

要实现 `monitor_throughput` + 动态采样间隔，需要：

```
吞吐量监控系统需要的组件：
├─ 1. 网络流量采集点
│   ├─ /api/video 路由处记录字节数
│   ├─ 记录请求开始/结束时间
│   └─ 计算即时吞吐量 (bytes/ms)
├─ 2. 采样调度器
│   ├─ 可变间隔的 ticker
│   ├─ 根据当前网络状况动态调整
│   └─ 5G 高频网络下缩短间隔
├─ 3. 带宽估计算法
│   ├─ 滑动窗口平均
│   ├─ 突发流量检测
│   └─ 趋势预测
└─ 4. 决策执行
    ├─ 触发码率切换（需要 ABR）
    ├─ 调整预取策略
    └─ 通知前端
```

Photoview 这四层**全部缺失**。

### 15.4 5G 高频网络下的现状

在 5G 网络下，Photoview 的表现：

| 特性 | 现状 | 理想情况（有动态采样） |
|------|------|----------------------|
| 吞吐量检测 | ❌ 无 | ✅ 10-50ms 采样间隔 |
| 网络波动响应 | ❌ 依赖浏览器 | ✅ 实时检测并调整 |
| 码率适配 | ❌ 单码率 1080p | ✅ 动态切换 480p/720p/1080p |
| 缓冲策略 | ❌ 固定（浏览器默认） | ✅ 根据带宽调整预缓冲 |
| 卡顿恢复 | ❌ 被动等待 | ✅ 主动降码率恢复 |

**为什么 5G 下需要更短的采样间隔**：
- 5G 网络波动更快（毫秒级），500ms 采样会错过快速变化
- 5G 带宽高但不稳定，需要实时检测吞吐量下降
- 动态采样可以在网络稳定时间隔拉长（省电），在波动时间隔缩短（灵敏）

但 Photoview 没有吞吐量监控，讨论采样间隔调整是无意义的。

### 15.5 所有时间间隔配置一览

| 间隔 | 位置 | 用途 | 可配置 |
|------|------|------|-------|
| 500ms | `queue.go:102` | 扫描进度通知限流 | ❌ 硬编码 |
| 1000 (LRU 容量) | `graphql_endpoint.go:44` | GraphQL 查询缓存大小 | ❌ 硬编码 |
| 100 (LRU 容量) | `graphql_endpoint.go:48` | 持久化查询缓存大小 | ❌ 硬编码 |
| periodic_scan_interval | `site_info` 表 | 周期性扫描间隔 | ✅ 可配置 |
| concurrent_workers | `site_info` 表 | 并发扫描任务数 | ✅ 可配置 |

---

## 十六、lru_evict_predict 算法分析

### 16.1 核心结论：不存在此算法

全局代码搜索结果：
- ❌ 没有 `lru_evict_predict` 函数/变量
- ❌ 没有 `predict` 预测相关代码
- ❌ 没有 `algorithm` 算法相关代码
- ❌ 没有 `evict` 淘汰逻辑（除了 gqlgen 内部 LRU）

### 16.2 唯一的 LRU 实现：gqlgen 内置缓存

`api/graphql/endpoint/graphql_endpoint.go:41,44`：
```go
import "github.com/99designs/gqlgen/graphql/handler/lru"

graphqlServer.SetQueryCache(lru.New[*ast.QueryDocument](1000))
graphqlServer.Use(extension.AutomaticPersistedQuery{
    Cache: lru.New[string](100),
})
```

这是 **gqlgen 框架内置的 LRU 缓存**，用于缓存 GraphQL 查询的 AST 解析结果和持久化查询。它的实现是**标准 LRU**，没有预测功能。

**标准 LRU 的工作原理**：
```
┌─────────────────────────────────────────┐
│  新数据插入 → 放在链表头部              │
│  数据被访问 → 移到链表头部              │
│  缓存满时 → 删除链表尾部元素（最久未用）│
└─────────────────────────────────────────┘
```

**没有的特性**：
- ❌ 没有访问频率统计（LRU 只看最近访问时间，不看访问次数）
- ❌ 没有预测下一个要访问的元素
- ❌ 没有突发流量检测
- ❌ 没有误判率统计
- ❌ 没有预热机制

### 16.3 突发流量场景下的表现

由于根本没有预测算法，讨论 "误判率指标" 是无意义的。但可以分析**标准 LRU 在突发流量下的行为**：

**场景 1：突然大量访问冷门视频（缓存击穿）**
```
用户 A 访问冷门视频 V1 → 缓存不命中 → 触发转码
用户 B 同时访问 V1 → 缓存不命中 → 重复触发转码
用户 C 同时访问 V1 → 缓存不命中 → 重复触发转码
...
转码完成 → V1 进入缓存 → 后续请求命中
```

问题：突发访问同一冷门资源时，会触发多次重复转码，直到第一个转码完成。

**场景 2：循环访问大量视频（缓存颠簸）**
```
访问 V1 → V1 入队
访问 V2 → V2 入队
...
访问 V1001 → V1 被淘汰（因为容量是 1000）
访问 V1 → 缓存不命中 → 重新转码
```

问题：当访问量超过缓存容量时，会发生频繁的淘汰和重新转码。

**场景 3：热点视频突然变冷**
```
V1 连续被访问 1000 次 → 一直在缓存头部
突然没人访问 V1 → V1 逐渐被新数据挤到尾部
最终被淘汰 → 下次访问需要重新转码
```

标准 LRU 无法预测 "这个热点以后还会不会被访问"。

### 16.4 预测算法需要的组件

要实现 `lru_evict_predict` 并计算误判率，需要：

```
预测淘汰系统需要的组件：
├─ 1. 访问历史记录
│   ├─ 记录每个视频的所有访问时间
│   ├─ 记录访问间隔分布
│   └─ 记录访问频次
├─ 2. 预测模型
│   ├─ 基于历史访问模式预测下次访问时间
│   ├─ 计算每个缓存项的 "剩余寿命"
│   └─ 选择 "最不可能被访问" 的项淘汰
├─ 3. 误判率统计
│   ├─ 记录被淘汰后 T 时间内又被访问的次数（误判）
│   ├─ 计算误判率 = 误判次数 / 总淘汰次数
│   └─ 阈值触发告警
└─ 4. 动态调整
    ├─ 突发流量时切换算法（如 LFU）
    ├─ 根据误判率调整缓存容量
    └─ 热点数据保护机制
```

Photoview 这四层**全部缺失**，连最基础的访问日志都没有。

### 16.5 误判率指标的计算方法

如果未来要实现，可以参考以下指标定义：

| 指标 | 公式 | 理想值 |
|------|------|--------|
| 误判率 | 被淘汰后 ΔT 内又被访问的次数 / 总淘汰次数 | < 5% |
| 缓存命中率 | 缓存命中次数 / 总请求次数 | > 95% |
| 平均访问间隔 | 同一视频两次访问的平均时间 | - |
| 淘汰后重访时间分布 | 被淘汰后多久被重新访问的分布 | - |
| 突发流量敏感度 | 突发流量期间误判率上升幅度 | < 2x |

其中 `ΔT` 是可配置的时间窗口（如 1 小时、1 天）。

### 16.6 相关模块现状总结

| 功能 | 现状 | 代码位置 |
|------|------|---------|
| LRU 缓存 | ✅ 有（仅 GraphQL 查询） | `graphql_endpoint.go:41,48` |
| LRU 容量 | 1000 + 100（硬编码） | `graphql_endpoint.go:44,48` |
| 媒体缓存 LRU | ❌ 无 | - |
| 访问预测 | ❌ 无 | - |
| 误判率统计 | ❌ 无 | - |
| 突发流量检测 | ❌ 无 | - |
| 动态算法切换 | ❌ 无 | - |

---

## 十七、总结

Photoview 的视频方案设计非常务实，本质是**个人相册级别的简单视频播放方案**，很多企业级特性都没有实现：

### 已实现的特性
1. **没有 HLS/m3u8 分片** —— 使用浏览器原生 `<video>` + HTTP Range 实现伪流式播放
2. **按需转码 + 后台预转码双通道**：扫描时预转码，用户访问也能实时触发
3. **缓存结构清晰**：`{cache}/{albumID}/{mediaID}/` 按相册和媒体分层存储
4. **`+faststart` 优化**：moov atom 前置让 MP4 浏览器快速启动
5. **硬件加速可选**：支持 qsv/vaapi/nvenc 硬件编码
6. **鉴权一体化**：Cookie 登录或分享 token 两种方式
7. **降级路径隐晦**：只有原视频是 Web 兼容格式时，DataLoader 才能 fallback 到 Original
8. **周期性扫描可配置**：扫描周期和并发 worker 数暴露给运维配置

### 未实现的特性
9. **无 ABR 自适应码率**：单码率 MP4，无多版本转码，无移动端网络适配
10. **无 LRU 缓存管理**：媒体缓存无限增长，无淘汰策略，无容量限制
11. **无 prefetch 机制**：无热度统计，无分片预加载，无预取队列
12. **转码失败残留文件**：失败时不清理不完整的输出文件，会占用磁盘空间
13. **无 QoE 监控**：无播放质量指标采集，无网络稳定性统计
14. **无 orphan 文件清理 worker**：只有周期性扫描，不清理残留转码文件
15. **无吞吐量监控**：无采样间隔，无动态调整，无 5G 网络适配
16. **无 LRU 预测算法**：只有标准 LRU（gqlgen 内置），无预测，无误判率统计
