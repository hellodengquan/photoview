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

## 十一、总结

Photoview 的视频方案设计非常务实：

1. **没有 HLS/m3u8 分片** —— 使用浏览器原生 `<video>` + HTTP Range 实现伪流式播放
2. **按需转码 + 后台预转码双通道**：扫描时预转码，用户访问也能实时触发
3. **缓存结构清晰**：`{cache}/{albumID}/{mediaID}/` 按相册和媒体分层存储
4. **`+faststart` 优化**：moov atom 前置让 MP4 浏览器快速启动
5. **硬件加速可选**：支持 qsv/vaapi/nvenc 硬件编码
6. **鉴权一体化**：Cookie 登录或分享 token 两种方式
7. **降级路径隐晦**：只有原视频是 Web 兼容格式时，DataLoader 才能 fallback 到 Original
8. **无 ABR 自适应码率**：单码率 MP4，无多版本转码
9. **无 LRU 缓存管理**：媒体缓存无限增长，无淘汰策略
10. **无 prefetch 机制**：无热度统计，无分片预加载
