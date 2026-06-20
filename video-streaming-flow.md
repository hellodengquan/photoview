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

## 八、总结

Photoview 的视频方案设计非常务实：

1. **没有 HLS/m3u8 分片** —— 使用浏览器原生 `<video>` + HTTP Range 实现伪流式播放
2. **按需转码 + 后台预转码双通道：扫描时预转码，用户访问也能实时触发
3. **缓存结构清晰**：`{cache/{albumID}/{mediaID}/` 按相册和媒体分层存储
4. **`+faststart` 优化**：moov atom 前置让 MP4 让浏览器快速启动
5. **硬件加速可选**：支持 qsv/vaapi/nvenc 硬件编码
6. **鉴权一体化**：Cookie 登录或分享 token 两种方式
