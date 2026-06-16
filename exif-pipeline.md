# EXIF 元数据处理流水线

本文档梳理 Photoview 中 EXIF 元数据从磁盘图片文件到数据库记录的完整链路，覆盖：

1. **外部 exiftool 进程的生命周期管理**
2. **进程间 stdin/stdout 的通信协议**
3. **JSON 字段 → Go 结构体 → DB 表 的三层映射**
4. **扫描任务触发 & GORM 持久化的衔接**

---

## 一、全局架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Photoview 进程                               │
│                                                                     │
│  server.go:56                                                        │
│     └─ exif.Initialize() ──► 启动常驻 exiftool 子进程                │
│                                                                     │
│  扫描任务 (ExifTask)                                                 │
│     └─ SaveEXIF() ──► exif.Parse() ──► QueryJSONTagsByNumber()      │
│                                 │                                    │
│                    ┌────────────┴────────────┐                       │
│                    │    stdin (写入命令)      │                       │
│                    ▼                         ▼                       │
│            ┌──────────────────────────────────────────┐             │
│            │            exiftool (子进程)              │             │
│            │  -stay_open True -@ -  (常驻模式)         │             │
│            └──────────────────────────────────────────┘             │
│                    │                         │                       │
│                    ▼                         ▼                       │
│               MarkReader                MarkReader                   │
│              (stdout 解析)             (stderr 读取)                  │
│                    │                         │                       │
│                    ▼                         ▼                       │
│            JSON 反序列化            错误检查                          │
│                    │                                                  │
│                    ▼                                                  │
│         PhotoMeta + TimeAll + GPS                                    │
│                    │                                                  │
│                    ▼                                                  │
│          models.MediaEXIF (GORM Model)                               │
│                    │                                                  │
│                    ▼                                                  │
│        media_exif 表 + media.ExifID 外键                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、exiftool 进程生命周期

### 2.1 初始化入口

| 文件 | 位置 | 说明 |
|------|------|------|
| `api/server.go` | L56-L60 | 服务启动时调用 `exif.Initialize()` |
| `api/test_utils/integration_setup.go` | L50-L54 | 集成测试同样初始化 |

```go
// server.go:56
exifCleanup, err := exif.Initialize()
if err != nil {
    log.Panicf("Could not initialize exif parser: %s", err)
}
defer exifCleanup()
```

### 2.2 exiftool 常驻进程启动 (`exiftool/exiftool.go:34-96`)

`New()` 函数执行以下步骤：

| 步骤 | 代码位置 | 说明 |
|------|---------|------|
| 1. 查找二进制 | L35 | `exec.LookPath("exiftool")` |
| 2. 校验版本 | L40-L48 | 运行 `exiftool -ver` 获取版本号 |
| 3. 构造命令 | L50 | `exiftool -stay_open True -@ -` |
| 4. 建立管道 | L52-L79 | 获取 stdin / stdout / stderr，并用 `MarkReader` 包装 stdout 和 stderr |
| 5. 启动进程 | L81 | `cmd.Start()` |

关键参数 `-stay_open True -@ -`：
- **`-stay_open True`**：使 exiftool 保持运行，不处理完就退出，通过 stdin 接收后续命令
- **`-@ -`**：从 stdin 读取参数列表，每行一个参数

### 2.3 进程关闭 (`exiftool/exiftool.go:109-120`)

`Close()` 使用 `sync.Once` 确保只关一次：

```
1. 发命令：-stay_open False
2. 关闭 stdin
3. cmd.Wait() 回收进程
4. 如果发命令失败，直接 Kill()
```

---

## 三、进程间通信协议

### 3.1 请求侧：stdin 命令格式 (`rawSendCommand`, exiftool.go:L122-142)

每条命令由以下几行通过 `\n` 分隔依次写入 stdin：

```
[用户参数1]
[用户参数2]
...
-echo4
{ready}
-execute
```

- **`-echo4 {ready}`**：让 exiftool 在输出末尾回显 `{ready}` 作为响应边界标记
- **`-execute`**：执行上述累积的参数并输出结果

发送前后会分别调用 `stdout.Reset()` 和 `stderr.Reset()`，清空上一次的标记状态。

### 3.2 响应侧：MarkReader 边界切分 (`mark_reader.go`)

exiftool 是连续输出的，多次查询结果会混在同一个 stdout 流中。`MarkReader` 是自定义的 `io.Reader`，作用是**在遇到 `{ready}\n` 标记时假装 EOF**，从而把连续流切成一个个独立响应。

核心状态机：

| 状态 | 含义 |
|------|------|
| `buf[start:pending]` | 本次可以安全返回给调用者的有效数据（不含标记） |
| `buf[pending:end]` | 待检测是否包含标记的窗口 |
| `hasMark = true` | 已在 pending 区域找到标记，下次读完有效数据后进入 paused |
| `paused = true` | Read() 返回 `io.EOF`，直到外部调用 `Reset()` |

每次查询的读协议：
1. `rawSendCommand` 写入 stdin 并 flush
2. `json.NewDecoder(e.stdout).Decode(v)` 从 MarkReader 读，直到它因遇到 `{ready}\n` 返回 EOF
3. 丢弃残留字节，读 stderr 检查错误

---

## 四、三类查询方法

`exiftool.go` 对外暴露三种操作，内部统一走 `rawSendCommand` → `MarkReader` 协议：

| 方法 | 用途 | 追加的 exiftool 参数 |
|------|------|---------------------|
| `QueryJSONTagsByNumber(file, value)` | 读取标签到结构体 | `-n -json` |
| `SaveJPEGPreview(src, dst)` | 提取 RAW 内嵌预览图 | `-JpgFromRaw -b -W <dst>` 然后 `-TagsFromFile` 拷标签 |
| `rawUpdateFile(...)` | 原地修改文件元数据 | 直接传入用户参数 |

其中 **`-n` (numerical)** 很关键：让 exiftool 输出数值形式（如 `Orientation=6`）而非人类可读文本（如 `Orientation=Rotate 90 CW`），便于程序消费。

---

## 五、字段解析：三层映射

### 5.1 第一层：exiftool JSON → Go 临时结构体 (`values.go`)

`QueryJSONTagsByNumber` 依赖 Go 的 `encoding/json` 按字段名匹配。exiftool 输出的 JSON key 就是结构体字段名（大小写敏感匹配，由 `json` tag 默认行为处理）。

`exif.go:53-57` 把三类结构体组合成一个匿名整体，一次查询拿齐：

```go
var values struct {
    exiftool.PhotoMeta    // 相机/镜头/曝光参数
    exiftool.TimeAll      // 各种日期时间 + 时区
    exiftool.GPS          // 经纬度
}
```

#### PhotoMeta — 硬件与曝光 (`values.go:L139-167`)

| Go 字段 | EXIF / JSON key | 类型 |
|---------|-----------------|------|
| `ImageDescription` | ImageDescription | `*string` |
| `Model` | Model | `*string` |
| `Make` | Make | `*string` |
| `LensModel` | LensModel | `*string` |
| `ISO` | ISO | `*int64` |
| `Flash` | Flash | `*int64` |
| `Orientation` | Orientation | `*int64` |
| `ExposureProgram` | ExposureProgram | `*int64` |
| `ExposureTime` | ExposureTime | `*float64` |
| `Aperture` | Aperture | `*float64` |
| `FocalLength` | FocalLength | `*float64` |

`SanitizeFloats()`：ExposureTime / Aperture / FocalLength 若为 `NaN` 或 `±Inf`，置为 `nil` 避免写入脏数据。

#### TimeAll — 拍摄时间 (`values.go:L45-137`)

包含多组时间源，优先级从高到低：

| 优先级 | 字段 | 说明 |
|--------|------|------|
| 1 | `SubSecDateTimeOriginal` | 带亚秒的原始拍摄时间 |
| 2 | `SubSecCreateDate` | 带亚秒的创建时间 |
| 3 | `DateTimeOriginal` | 原始拍摄时间（无亚秒） |
| 4 | `CreateDate` | 创建时间 |
| 5 | `TrackCreateDate` | 音轨创建时间（视频） |
| 6 | `MediaCreateDate` | 媒体创建时间（视频） |
| 7 | `FileModifyDate` | 文件修改时间（兜底） |

`TimeInLocal()` 解析时会**剥离时区后缀**，用 `time.UTC` 位置解析成"本地表现形式"——时区信息另行通过 `OffsetSecs()` 计算。

时区偏移计算同样有优先级：
1. `OffsetTimeOriginal` / `OffsetTime`：显式的 `+08:00` 字符串
2. `TimeZone`：以分钟为单位的整数，×60 转秒
3. `GPSDateTime`：GPS 时间天然 UTC，与本地时间做差得到偏移

#### GPS — 坐标 (`values.go:L10-44`)

`IsValid()` 校验：
- 经纬度均非 nil
- 非 NaN
- 纬度 ∈ [-90, 90]，经度 ∈ [-180, 180]

### 5.2 第二层：临时结构体 → GORM Model (`exif.go:64-91`)

```go
ret := models.MediaEXIF{
    Camera:          values.Model,
    Maker:           values.Make,
    Lens:            values.LensModel,
    Iso:             values.ISO,
    Flash:           values.Flash,
    Orientation:     values.Orientation,
    ExposureProgram: values.ExposureProgram,
    Exposure:        values.ExposureTime,
    Aperture:        values.Aperture,
    FocalLength:     values.FocalLength,
    Description:     values.ImageDescription,
}
```

时间和 GPS 做了条件性赋值：

| 处理 | 代码位置 | 说明 |
|------|---------|------|
| `DateShot` | L78-L81 | `TimeInLocal()` 非零则取指针 |
| `OffsetSecShot` | L83-L86 | `OffsetSecs()` 成功则存秒数 |
| `GPSLatitude/Longitude` | L88-L91 | `GPS.IsValid()` 通过才赋值 |

### 5.3 第三层：GORM Model → 数据库表

`models.MediaEXIF` 定义于 `api/graphql/models/media_exif.go:L8-25`，对应表 `media_exif`：

| 字段 | 类型 | 对应上面 |
|------|------|---------|
| `description` | `*string` | ImageDescription |
| `camera` | `*string` | Model |
| `maker` | `*string` | Make |
| `lens` | `*string` | LensModel |
| `date_shot` | `*time.Time` | TimeAll 计算结果 |
| `offset_sec_shot` | `*int` | 时区偏移秒 |
| `exposure` | `*float64` | ExposureTime |
| `aperture` | `*float64` | Aperture |
| `iso` | `*int64` | ISO |
| `focal_length` | `*float64` | FocalLength |
| `flash` | `*int64` | Flash |
| `orientation` | `*int64` | Orientation |
| `exposure_program` | `*int64` | ExposureProgram |
| `gps_latitude` | `*float64` | GPSLatitude |
| `gps_longitude` | `*float64` | GPSLongitude |

`Media` 表（`media.go:L15-33`）通过 `ExifID *int` 外键 + `Exif *MediaEXIF` 关联，GORM constraint 为 `OnDelete:CASCADE`。

---

## 六、触发时机与持久化衔接

### 6.1 扫描任务钩子

`ExifTask` 注册在扫描管道中，实现 `AfterMediaFound` 接口（`scanner_tasks/exif_task.go:L19-29`）：

```go
func (t ExifTask) AfterMediaFound(ctx scanner_task.TaskContext, media *models.Media, newMedia bool) error {
    if !newMedia {   // 只处理新发现的媒体
        return nil
    }
    if err := SaveEXIF(ctx.GetDB(), media); err != nil {
        log.Warn(...)   // 失败仅记录告警，不中断扫描
    }
    return nil
}
```

### 6.2 SaveEXIF 流程 (`exif_task.go:L32-74`)

```
┌─ SaveEXIF(tx, media) ────────────────────────────────────────────────┐
│                                                                      │
│  1. 去重检查                                                          │
│     └─ 若 media.ExifID != nil，先查 DB 中是否已存在                    │
│        ├─ 存在 → 直接 return nil（跳过）                              │
│        ├─ ErrRecordNotFound → 返回错误（不应发生，外键悬空）           │
│        └─ 其他错误 → 置 ExifID = nil，重新解析                        │
│                                                                      │
│  2. 解析 EXIF                                                         │
│     └─ exifData, err := exif.Parse(media.Path)                       │
│        ├─ err != nil → 包装错误返回                                   │
│        └─ exifData == nil（文件无 EXIF）→ return nil                  │
│                                                                      │
│  3. 写关联                                                            │
│     └─ tx.Model(media).Association("Exif").Replace(exifData)         │
│        → 创建或替换 media_exif 记录，回填 media.exif_id               │
│                                                                      │
│  4. 同步 DateShot 到 media 表                                         │
│     └─ 若 exifData.DateShot != nil 且与 media.DateShot 不同          │
│        → tx.Save(media) 更新 media.date_shot                         │
│        → 同时更新 media 内存对象                                      │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

关键点：
- **`Association("Exif").Replace`**：GORM 会自动处理 INSERT/UPDATE，并把新 ID 写回 `media.ExifID`
- **DateShot 双写**：`media_exif.date_shot` 存精确值（可空），而 `media.date_shot` 是非空列用于列表排序，两者可能来源不同（媒体扫描时先用文件 mtime，EXIF 解析后再修正）
- **失败降级**：`AfterMediaFound` 吞掉错误仅告警，保证单个文件 EXIF 解析失败不影响整个扫描任务

### 6.3 线程安全

`exif.go` 中整个 `Parse` 调用用全局互斥锁 `globalMu` 包裹：
- exiftool 的 `-stay_open` 模式是**单工**的：同一时刻只能有一个请求在途
- `globalMu.Lock()` 保证并发扫描任务不会同时写 stdin 导致协议错位

---

## 七、完整调用链（自顶向下一次定位）

```
server.go:56 Initialize
 └─ exif/exif.go:15 Initialize
     └─ exiftool/exiftool.go:34 New
         └─ exec.Command("exiftool", "-stay_open", "True", "-@", "-")

扫描器发现新媒体
 └─ ExifTask.AfterMediaFound (exif_task.go:19)
     └─ SaveEXIF (exif_task.go:32)
         ├─ [去重检查]
         └─ exif.Parse (exif.go:45)
             ├─ globalMu.Lock()
             └─ QueryJSONTagsByNumber (exiftool.go:231)
                 └─ rawGetTags (exiftool.go:158)
                     ├─ rawSendCommand + "-n" + "-json" + file
                     │   └─ MarkReader 在 stdout 上切 {ready} 边界
                     ├─ json.Decode → PhotoMeta + TimeAll + GPS
                     └─ rawReadStderr 检查错误
         ├─ [构建 MediaEXIF]
         ├─ Association.Replace → INSERT media_exif
         └─ [若需] UPDATE media SET date_shot = ?

服务退出时
 └─ exifCleanup()
     └─ exiftool.Close() → -stay_open False → Wait()
```

---

## 八、批量 import：并发与去重

### 8.1 两级并发架构

整个扫描系统采用 **Album 级并行、单 Album 内串行** 的策略：

```
┌─ ScannerQueue (全局) ────────────────────────────────────────────────┐
│                                                                      │
│  max_concurrent_tasks = site_info.concurrent_workers                  │
│    - MySQL/PostgreSQL: 默认 3                                        │
│    - SQLite: 强制 1（避免锁冲突）                                     │
│                                                                      │
│  idle_chan ──► 唤醒后台 worker                                        │
│  up_next   ──► 待执行 album job 队列                                 │
│  in_progress ──► 正在执行的 album job                                │
│                                                                      │
│  ┌─ Worker 1 ──┐   ┌─ Worker 2 ──┐   ...   ┌─ Worker N ──┐          │
│  │  Album A    │   │  Album B    │         │  Album X    │          │
│  │  串行处理    │   │  串行处理    │         │  串行处理    │          │
│  └─────────────┘   └─────────────┘         └─────────────┘          │
│                                                                      │
│  单 Album 内部逐个文件串行：                                          │
│    for media in albumMedia:                                          │
│        scanMedia()   ← 含数据库事务 + 各 Task 钩子                   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**关键代码**：
- 并发数配置：`models/site_info.go:L19-29`（`DefaultSiteInfo()`）
- 队列初始化：`scanner_queue/queue.go:L60-86`（`InitializeScannerQueue()`）
- Worker 启动：`scanner_queue/queue.go:L142-167`（`processQueue()` 中 goroutine per job）
- 单 Album 串行：`scanner/scanner_album.go:L100-107`

### 8.2 四层去重机制

| 层级 | 检查点 | 代码位置 | 说明 |
|------|--------|---------|------|
| L1 队列去重 | `jobOnQueue()` | `scanner_queue/queue.go:L253-263` | 同一 Album 不会重复入队，按 `album.ID` 比对 `in_progress + up_next` |
| L2 数据库去重 | `path_hash` | `scanner/scanner_media.go:L25-38` | 用 `MD5(mediaPath)` 查 `media` 表，命中则返回已存在记录，`newMedia=false` 跳过后续处理 |
| L3 缓存去重 | `AlbumScannerCache` | `scanner_cache/cache.go:L50-61` | 内存缓存目录是否含照片、媒体类型、ignore 规则，避免重复扫描磁盘 |
| L4 EXIF 去重 | `media.ExifID` | `scanner_tasks/exif_task.go:L33-50` | 若 `media.ExifID != nil` 且 DB 中能查到，直接跳过解析 |

**路径哈希机制**：
- 生成：`models/media.go:L41`（`BeforeSave` 钩子）→ `MD5Hash(media.Path)`
- 约束：`media.path_hash` 字段设 `UNIQUE`（`media.go:L19`）
- 对比：`scanner_media.go:L28` 用 hash 查询而非 path 字符串

### 8.3 任务入队与调度

```go
// AddUserToQueue (scanner_queue/queue.go:L223-239)
1. FindAlbumsForUser() → 递归遍历用户所有相册
2. 遍历每个 album:
   a. 调用 addJob()
   b. addJob() 先调用 jobOnQueue() 检查是否已在队列
   c. 不在队列则 append(up_next, job)
   d. 调用 notify() 向 idle_chan 发信号唤醒 worker
3. processQueue() 被唤醒后:
   a. 只要 len(in_progress) < maxJobs && len(up_next) > 0
   b. 从 up_next 队首取 job，移到 in_progress
   c. 启动 goroutine 执行 job.Run()
```

运行时并发数修改：`ChangeScannerConcurrentWorkers()`（`queue.go:L92-98`）支持热更新，新值在下一次 `processQueue` 时生效。

#### 8.3.1 max_concurrent_tasks 动态调整的生效时序

完整的调用链与时序：

```
用户在前端修改并发数
  │
  ▼
GraphQL Mutation: setScannerConcurrentWorkers (resolvers/scanner.go:L81-108)
  ├─ 校验 workers >= 1, SQLite 时必须 == 1
  ├─ UPDATE site_info SET concurrent_workers = ?   ← 写 DB
  ├─ SELECT * FROM site_info                        ← 重新读取确认
  └─ scanner_queue.ChangeScannerConcurrentWorkers(N)
       │
       ├─ mutex.Lock()
       ├─ settings.max_concurrent_tasks = N        ← 仅改内存字段
       └─ mutex.Unlock()
       │
       ▼  生效点：下一次 processQueue 被唤醒时
processQueue (queue.go:L136-192)
  ├─ mutex.Lock()
  ├─ maxJobs := queue.settings.max_concurrent_tasks   ← 读取最新值
  └─ for len(in_progress) < maxJobs && len(up_next) > 0:
         启动新 goroutine
```

**生效边界说明**：

| 场景 | 行为 |
|------|------|
| 调大 N（如 3→6） | 正在运行的 3 个 job 不受影响；后续 `processQueue` 唤醒时最多再启动 3 个补到 6 |
| 调小 N（如 6→3） | 正在运行的 6 个 job **不会被终止**，继续跑完；之后 `processQueue` 不再启动新 job，直到 `in_progress` 降到 3 以下 |
| 运行中瞬时值 | 若 `in_progress` 已超过新 maxJobs（调小时），新 maxJobs 仅作为调度上限，不强制回收已启动 goroutine |

**关键代码**：
- `processQueue` 在 `queue.go:L139` 读取 `maxJobs`，每次唤醒都会重读，所以改动是即时生效到调度逻辑的
- 正在运行的 goroutine（`queue.go:L149-166`）没有取消机制，只能跑完自然退出

---

## 九、缓存命中与失效

### 9.1 两层缓存架构

#### 内存缓存：`AlbumScannerCache`（`scanner_cache/cache.go`）

**三个 map 存储不同维度的缓存**：

| 缓存键 | 值类型 | 写入时机 | 读取时机 |
|--------|--------|---------|---------|
| `path_contains_photos` | `bool` | 扫描到目录含媒体时，递归向上写入所有父目录 | `AlbumContainsPhotos()` 查询目录是否需要继续扫描 |
| `photo_types` | `MediaType` | `GetMediaType()` 第一次调用 exiftool 成功后 | 后续调用直接返回，避免重复调用 exiftool |
| `ignore_data` | `[]string` | 读取 `.photoviewignore` 文件后 | 编译 gitignore 匹配器 |

```go
// 缓存填充：GetMediaType (cache.go:L70-87)
func (c *AlbumScannerCache) GetMediaType(path string) media_type.MediaType {
    result, found := c.photo_types[path]
    if found {
        return result  // 缓存命中
    }
    mediaType := media_type.GetMediaType(path)  // 调用 exiftool.MIMEType()
    if mediaType != media_type.TypeUnknown {
        c.photo_types[path] = mediaType  // 写缓存
    }
    return mediaType
}
```

**生命周期**：每用户每次扫描创建独立 `AlbumScannerCache`，扫描结束后释放。

#### 磁盘缓存：`MediaCache`（`utils/media_cache.go`）

**目录结构**：
```
PHOTOVIEW_MEDIA_CACHE (默认 ./media_cache)
├── album_<id>/
│   ├── media_<id>/
│   │   ├── thumbnail_<name>.jpg
│   │   ├── highres_<name>.jpg
│   │   ├── web_video_<name>.mp4
│   │   └── video_thumb_<name>.jpg
```

**配置**：
- 环境变量 `PHOTOVIEW_MEDIA_CACHE` 可自定义路径
- 动态创建：`CachePathForMedia()` 在需要时 mkdir

### 9.2 缓存命中检查

`ProcessPhotoTask` 和 `ProcessVideoTask` 中对每个 `MediaPurpose` 做三段式检查：

```
1. DB 查询：makePhotoURLChecker(mediaID)(purpose)
   ├─ 命中 → 拿到 media_url 记录，进入 step 2
   └─ 未命中 → 需要重新生成，进入 step 3

2. 磁盘检查：os.Stat(media_cache_path + media_name)
   ├─ 存在 → 缓存有效，跳过生成
   └─ 不存在 → DB 有记录但文件丢失，需要重新编码，进入 step 3

3. 重新生成并写 DB
```

**代码示例**（`process_photo_task.go:L69-82`）：
```go
if highResURL == nil {
    // DB 没记录，需要生成
} else {
    baseImagePath = path.Join(mediaCachePath, highResURL.MediaName)
    if _, err := os.Stat(baseImagePath); os.IsNotExist(err) {
        // DB 有记录但磁盘无文件，重新编码
        err = mediaData.EncodeHighRes(baseImagePath)
    }
}
```

### 9.3 缓存失效与清理

**主动失效**：
- `CleanupMedia`（`cleanup_tasks/cleanup_media.go:L17-65`）：每次扫描 Album 后，找出 DB 有但磁盘已不存在的 media，删除 `media_cache/<album_id>/<media_id>/` 目录，再删除 DB 记录
- `DeleteOldUserAlbums`（`cleanup_media.go:L68-136`）：删除整个不再存在的 Album 的缓存目录和 DB 记录

**被动失效（懒检测）**：
- 每次处理 media 时检查磁盘文件是否存在（如上述 `os.Stat`）
- 不存在则重新生成，属于"读时修复"模式

---

#### 9.3.1 同目录改名重传的假更新边界

`PathHash = MD5(media.Path)` 机制（`models/media.go:L39-44`）在以下场景会产生"假命中"：

```
场景 A — 原地替换文件（路径不变，内容变了）：
  /album/IMG_0001.CR2  (旧的)   → MD5 = hash("/album/IMG_0001.CR2") = H1
  用户删除旧文件，拷入同名新文件
  /album/IMG_0001.CR2  (新的)   → MD5 = hash("/album/IMG_0001.CR2") = H1

  结果：ScanMedia() 查 DB 命中 H1 → newMedia=false → 所有 AfterMediaFound Task
       （包括 ExifTask、SidecarTask、ProcessPhotoTask）全部跳过。
       缩略图/高清图/EXIF 全部停留在旧版本。
```

```
场景 B — 同目录改名（内容不变，路径变了）：
  /album/IMG_0001.CR2              → MD5 = H1
  用户 mv IMG_0001.CR2 Vacation_001.CR2
  /album/Vacation_001.CR2          → MD5 = H2

  结果：ScanMedia() 查不到 H2 → 被当作新媒体完整重新扫描一遍（冗余处理），
       同时旧 H1 对应的 media 记录留在 DB，CleanupMedia 下次扫描时清理。
```

**设计权衡**：
- 只对路径做 hash，不对内容做 hash（避免扫描时读几 MB/几十 MB 文件内容）
- 换内容不换路径 = 漏更新（假阴性命中）
- 换路径不换内容 = 重复处理（假阳性未命中）

**绕过方式（唯一的"内容变更感知"机制）**：只有 Sidecar XMP 文件有 `SideCarHash`（内容 MD5）——`sidecar_task.go:L72-83` 检测到 `.xmp` 文件内容变化时，会强制重新编码缩略图和高清图。原始 RAW/JPEG 的内容变更没有同类机制。

---

#### 9.3.2 缓存命中后的字段修改回写失效

缓存命中分两段：DB 记录命中 + 磁盘文件命中。不同字段的修改回写策略不同：

| 字段/模块 | 缓存命中（DB有+磁盘有）时的回写策略 | 代码位置 |
|-----------|-----------------------------------|---------|
| `MediaEXIF` (所有 EXIF 字段) | **不回写**。`ExifTask.AfterMediaFound` 对 `!newMedia` 直接 return（`exif_task.go:L19-22`），EXIF 内容变更永远不会被重新解析 | `exif_task.go:L20` |
| `MediaURL` 缩略图/高清图（内容） | **不回写**。`ProcessPhotoTask.ProcessMedia` 查到 DB 有记录且 `os.Stat` 存在就 return，不检查原始文件 mtime/内容哈希 | `process_photo_task.go:L31-46, L69-82, L110-123` |
| `MediaURL` 缩略图/高清图（尺寸/文件大小） | **部分回写**。仅 Sidecar 变更时（SideCarHash 变了），`generateSaveHighResJPEG` 会用 `tx.Save(mediaURL)` 更新 width/height/file_size | `processing_functions.go:L30-53` |
| `SideCarPath` / `SideCarHash` | **回写**。SidecarTask.ProcessMedia 每次都重新 hash XMP 比对，变了就 `Save(media)` 并重新编码 | `sidecar_task.go:L68-125` |
| `media.date_shot` | **不回写**。只有 `newMedia=true` 时 EXIF 解析才可能更新 `DateShot`（`exif_task.go:L66-71`） | `exif_task.go:L19-21` |

**唯一的强制重扫路径**：通过 GraphQL 调用 `ProcessSingleMedia`（`scanner_media.go:L77-93`），该函数绕过 `newMedia` 判断直接走完整 Task 管道。但此 API 不会自动触发，需手动调用。

---

## 十、损坏文件回退路径

### 10.1 七重前置过滤

在进入正式处理前，已层层过滤掉坏文件：

| 过滤层 | 检查项 | 代码位置 | 动作 |
|--------|--------|---------|------|
| 1 | .photoviewignore 规则 | `ignorefile_task.go:L28-37` | 匹配则 skip=true，跳过 |
| 2 | 隐藏文件 | `scanner_cache/cache.go:L112-114` | 文件名以 `.` 开头跳过 |
| 3 | 媒体类型支持 | `cache.go:L116-119` | 不支持的 MIME 类型跳过 |
| 4 | 空文件 | `cache.go:L122-125` | `fileStats.Size() == 0` 跳过 |
| 5 | RAW 禁用开关 | `counterpart_files_task.go:L24-31` | `PHOTOVIEW_DISABLE_RAW_PROCESSING=1` 时跳过非 web 兼容格式 |
| 6 | JPG+RAW 配对 | `counterpart_files_task.go:L33-38` | RAW 有对应 JPG 时跳过 JPG，避免重复处理 |
| 7 | 路径哈希去重 | `scanner_media.go:L25-38` | 已存在于 DB 的跳过 |

### 10.2 错误边界与降级

**三层 try-catch 结构**：

```
┌─ Album 级 (scanner_album.go:L100-107) ───────────────────────────┐
│  for i, media := range albumMedia {                               │
│      if err := scanMedia(...); err != nil {                       │
│          ScannerError(...)   // 发通知 + 打日志                   │
│          continue             // 跳过这个文件，继续下一个         │
│      }                                                             │
│  }                                                                 │
└───────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─ scanMedia 事务包裹 (media_scan.go:L22-37) ──────────────────────┐
│  transactionError := newCtx.DatabaseTransaction(func(...) {      │
│      // 所有 DB 操作在事务内                                      │
│      // 任何错误自动 Rollback                                     │
│  })                                                               │
└───────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─ 单 Task 级 (exif_task.go:L24-26) ──────────────────────────────┐
│  if err := SaveEXIF(ctx.GetDB(), media); err != nil {             │
│      log.Warn(...)   // 只打 Warn，不返回错误                     │
│  }                                                                 │
│  return nil        // 继续执行后续 Task                            │
└───────────────────────────────────────────────────────────────────┘
```

**关键设计点**：

1. **EXIF 失败不中断**：`ExifTask.AfterMediaFound` 吞掉错误（`exif_task.go:L24-26`），只打 `log.Warn`，不影响其他 Task 执行
2. **单文件失败不影响 Album**：`scanner_album.go:L104-106` 中 `scanMedia` 错误只调用 `ScannerError` 然后 `continue`
3. **事务原子性**：每个文件的 DB 操作包在 `DatabaseTransaction` 中（`media_scan.go:L22-33`），失败自动回滚
4. **错误通知**：`ScannerError()`（`scanner_utils/scanner_error.go:L13-23`）同时做两件事：
   - `log.Error()` 写日志
   - 通过 WebSocket `notification.BroadcastNotification()` 推送给前端

### 10.3 EXIF 专用回退

`SaveEXIF` 内部还有专门的容错：

```
SaveEXIF(tx, media)
├─ [检查 media.ExifID != nil]
│   ├─ 查 DB 成功 → return nil（正常）
│   ├─ ErrRecordNotFound → return error（外键悬空，严重错误）
│   └─ 其他错误（DB 连接问题等）→ 置 ExifID = nil，继续解析（降级）
│
├─ exif.Parse(media.Path)
│   ├─ err != nil → return error（文件损坏、exiftool 崩溃等）
│   └─ exifData == nil → return nil（文件无 EXIF，正常）
│
└─ 后续写 DB 操作
```

### 10.4 半解析半失败的字段保留策略

当文件"部分损坏"——exiftool 能返回部分字段但其他字段缺失或异常——时，各层的保留策略不同：

#### 10.4.1 EXIF 字段级：nil 值丢弃策略

`PhotoMeta` / `TimeAll` / `GPS` 所有字段均为指针类型（`*string`、`*int64`、`*float64`、`*time.Time`）。exiftool 的 JSON 输出对不存在的字段**不输出 key**，`encoding/json` 解码时保持该字段为 `nil`。

| 场景 | 保留策略 | 代码实现 |
|------|---------|---------|
| 字段缺失 | 留 `nil`，不写 DB | `values.go` 所有字段为指针 |
| 数值 NaN / ±Inf | `PhotoMeta.SanitizeFloats()` 置 `nil` | `values.go:L153-167` |
| GPS 越界（纬度>90 等） | `GPS.IsValid()` 不通过，整体丢弃 | `values.go:L17-35` |
| 时间字符串解析失败 | `TimeInLocal()` 试下一个优先级字段，全失败则返回零值 time.Time{} → DateShot 不赋值 | `values.go:L67-95` |
| 时区偏移解析失败 | 试下一个优先级策略，全失败则 OffsetSecShot 不赋值 | `values.go:L98-137` |

**结果**：`MediaEXIF` 写入 DB 时，合法字段正常入库，非法/缺失字段为 `NULL`——这是一种"部分成功"模式，而不是整体失败。

#### 10.4.2 流程级：全有或全无

但更高层的事务边界使得"部分成功"其实是"全有或全无"：

```
┌─ DatabaseTransaction (media_scan.go:L22-33) ───────────────┐
│                                                             │
│  ScanMedia()          → INSERT media                        │
│  AfterMediaFound()    → 逐个 Task:                           │
│    ExifTask           → INSERT media_exif                   │
│    SidecarTask        → UPDATE media (sidecar hash/path)    │
│    ...                                                        │
│                                                             │
│  ProcessMedia()       → INSERT media_url (缩略图等)          │
│  AfterProcessMedia()  → ...                                  │
│                                                             │
│  任何一步 err != nil → ROLLBACK，所有上面的写入全部撤销     │
└─────────────────────────────────────────────────────────────┘
```

**冲突点**：
- EXIF 层面是字段级的软失败（nil = 跳过该字段）
- 但 DatabaseTransaction 层面是文件级的硬失败（任何 Task 返回 error = 整张图所有 DB 记录回滚）
- 两者之间的衔接：`ExifTask.AfterMediaFound` **吞掉了 SaveEXIF 的错误**（只打 log.Warn，不 return err），所以 EXIF 解析失败不会触发整个文件的 Rollback

**最终保留矩阵**：

| 失败位置 | 结果 |
|---------|------|
| EXIF 某字段缺失/非法 | 该字段为 NULL，其他 EXIF 字段正常入库 |
| EXIF 整体解析失败（exiftool 崩溃/文件损坏） | 整条 media_exif 不创建，media 记录保留（因为 ExifTask 吞了错误），缩略图/高清图等后续 Task 正常执行 |
| 缩略图生成失败 | 整个事务回滚，包括已写入的 media 和 media_exif 记录 |
| Sidecar hash 写入失败 | 整个事务回滚 |

---

## 十一、内存占用上限和大文件分片读取

### 11.1 内存占用控制策略

| 控制点 | 机制 | 代码位置 | 上限值 |
|--------|------|---------|--------|
| MarkReader 缓冲区 | 固定大小环形缓冲区 | `exiftool.go:L31` + `mark_reader.go` | 10KB (`bufferSize = 10240`) |
| exiftool 进程 | 外部进程处理，不在 Go 堆 | `exiftool.go` 全程 | O(1)（不随文件大小增长） |
| EncodeMediaData | 懒加载 + 字段级缓存 | `encode_photo.go:L97-L103` | 单个结构体，按需填充 |
| imagick | 流式处理 + 及时 Destroy | `magickwand.go:L36-117` | 单张图解码内存，处理完释放 |
| ffmpeg/ffprobe | 外部进程流式处理 | `ffmpeg_cli.go` + `encode_photo.go:L155-170` | O(1)（不读入 Go 内存） |
| 并发 worker 数 | `max_concurrent_tasks` 限制 | `site_info.go:L20-23` | MySQL=3, SQLite=1 |

#### MarkReader 缓冲区设计

`mark_reader.go` 中缓冲区大小 `bufferSize = 10240`（10KB），远大于 `{ready}\n` 标记长度（8 字节）：

```
┌─ buf[10240] ──────────────────────────────────────────────────────┐
│  [valid data][pending window]      [unused]                        │
│   start    pending               end                               │
└───────────────────────────────────────────────────────────────────┘

每次 Read():
1. pending 区域检查是否包含 {ready} 标记
2. 有标记 → 返回 valid data，然后 paused=true
3. 无标记 → 从 upstream 读更多到 pending 区域末尾
4. compact 缓冲区（start==pending 时）避免溢出
```

**保证**：无论 EXIF 数据多大，`MarkReader` 内存占用恒定为 10KB + 少量状态字段。

#### EncodeMediaData 懒加载

```go
// encode_photo.go:L97-L103
type EncodeMediaData struct {
    Media           *models.Media
    CounterpartPath *string
    _photoImage     image.Image     // 只有首次调用 PhotoImage() 才解码
    _contentType    media_type.MediaType
    _videoMetadata  *ffprobe.ProbeData  // 只有首次调用 VideoMetadata() 才 probe
}
```

每个字段都有对应的 getter 方法，首次访问才触发昂贵操作，后续访问直接返回缓存值。

### 11.2 大文件处理模式

**完全不落 Go 堆的外部进程处理**：

| 操作 | 实现方式 | 内存特点 |
|------|---------|---------|
| 图片尺寸识别 | `magickwand.IdentifyDimension()` → 调用 imagick C 库 | 图片解码在 C 堆，Go 侧只拿 w/h 整数 |
| RAW → JPEG 转码 | `magickwand.EncodeJpeg()` → imagick 流式读入写出 | 同一时间只解码一张，`defer wand.Destroy()` 立即释放 |
| 缩略图生成 | `magickwand.GenerateThumbnail()` → 同上 | 同上 |
| 视频转码 MP4 | `ffmpeg_cli.EncodeMp4()` → exec.Command ffmpeg | 完全在子进程处理，Go 侧只等待退出码 |
| 视频抽帧缩略图 | `ffmpeg_cli.EncodeVideoThumbnail()` → 同上 | 同上 |
| 视频元数据 | `ffprobe.ProbeURL()` → 外部 ffprobe 进程 | JSON 输出解析到结构体，通常几 KB |
| EXIF 读取 | `exiftool -stay_open` → JSON 输出 | JSON 通常几 KB，MarkReader 10KB 上限 |

**关键**：对于几十 MB 的 RAW 照片和几百 MB/几 GB 的视频文件，Go 进程内存占用始终稳定在几十 MB 级别，不会随媒体文件大小线性增长。

### 11.3 超时控制

| 操作 | 超时配置 | 代码位置 |
|------|---------|---------|
| ffprobe 元数据读取 | `PHOTOVIEW_MEDIA_PROBE_TIMEOUT`，默认 5s | `utils/environment_variables.go:L87-95` |
| ffmpeg 视频转码 | 无硬超时（由操作系统进程管理） | - |
| imagick 图片处理 | 无硬超时（C 库阻塞） | - |
| 任务取消 | `context.Done()` 检查贯穿所有 Task 钩子 | `scanner_tasks.go:L39-43, L61-64` 等 |

每个 Task 执行前都会检查 `ctx.Done()`，支持用户在前端取消扫描任务。

### 11.4 ffprobe 5s 超时按视频规模的分级建议

`PHOTOVIEW_MEDIA_PROBE_TIMEOUT`（`utils/environment_variables.go:L87-95`）默认 5 秒是针对"视频元数据读取"（不是转码），ffprobe 通常只需读取文件头的 moov atom 即可获得时长/分辨率/码率等信息。

但实际耗时受以下因素影响，需要分级调整：

| 视频规模 | 典型场景 | 建议超时 | 原因 |
|---------|---------|---------|------|
| 短视频（<1 分钟，手机拍摄） | MP4/H.264，moov 在文件头 | 3-5s（默认） | 读头部几 KB，几乎瞬时 |
| 中视频（1-30 分钟） | 相机拍摄 MOV/MP4 | 5-15s | 文件大时 moov 可能在尾部，ffprobe 需 seek |
| 长视频（>30 分钟，电影级） | MKV/AVI，多个音轨字幕 | 15-60s | 容器解析复杂，多轨遍历 |
| RAW 视频 / 高码率（>100Mbps） | ProRes, DNxHR, BRAW | 30-120s | 单帧体积大，解析元数据也需要解部分码流 |
| 远程 / 网络挂载（NFS/SMB） | 任何规模 | 以上 ×2~×5 | 网络延迟 + seek 放大 |

**超时后的行为**：ffprobe 进程被 `exec.CommandContext` 的 cancel 杀死（`scanner/externaltools/ffprobe/...`），`VideoMetadataTask.AfterMediaFound` 返回 error → 整个文件事务回滚（参见 10.4.2 流程级全有或全无），该视频在下一次扫描时会被重试。

**注意**：超时是**单文件级**的，不会影响整体扫描队列——超时的那个文件被 `ScannerError` 记录后 `continue` 处理下一个。

### 11.5 imagick.Destroy 未调用的泄露场景

`imagick.MagickWand` 封装的是 ImageMagick C 库资源，**不经过 Go GC**，必须显式 `Destroy()` 释放。

#### 已正确实现的场景（都有 defer）

| 函数 | Destroy 位置 |
|------|-------------|
| `EncodeJpeg` | `magickwand.go:L40` `defer wand.Destroy()` |
| `GenerateThumbnail` | `magickwand.go:L62` `defer wand.Destroy()` |
| `IdentifyDimension` | `magickwand.go:L89` `defer wand.Destroy()` |
| Benchmark 测试 | `perf_test.go:L55` `defer mw.Destroy()` |

#### 会导致泄露的场景

**场景 1：`createWandFromFile` 中 ReadImage 失败**（`magickwand.go:L97-106`）

```go
func (cli *MagickWand) createWandFromFile(inputPath string) (*imagick.MagickWand, error) {
    wand := imagick.NewMagickWand()    // ← C 资源已分配

    if err := wand.ReadImage(inputPath); err != nil {
        return nil, fmt.Errorf(...)    // ← BUG: wand 没 Destroy 就 return
    }
    return wand, nil
}
```

调用方的 `defer wand.Destroy()` 只有拿到 wand 才会执行，但 ReadImage 失败时 wand 已经被 NewMagickWand() 分配却**从未被 Destroy**。每次损坏文件触发这个路径就泄露一个 MagickWand（几十 MB 到几百 MB 级的 C 堆内存，取决于图片尺寸）。

**场景 2：Terminate() 未调用导致进程退出时泄露**

`MagickWand.Terminate()`（`magickwand.go:L26-29`）调用 `imagick.Terminate()` 释放 ImageMagick 全局状态。如果服务进程不是通过正常的 `server.go` cleanup 路径退出（如 SIGKILL、panic 未恢复），全局初始化的 MagickWand 单例不被 Terminate。

#### 泄露量级估算

| 图片类型 | 单张 Decode 后内存（C 堆） | 并发 | 泄露 100 次后 |
|---------|--------------------------|------|--------------|
| 24MP JPG（6000×4000） | ~288MB（RGBA 8bit） | 3 workers | ~28GB |
| 45MP RAW（8256×5504） | ~1.6GB（RGBA 16bit） | 3 workers | ~160GB |

这些内存在 Go runtime 视角是"外部 C 内存"，`runtime.ReadMemStats` 的 `Sys` 不包含，Go GC 完全看不到——只能通过 OS 级 `RSS` 观察到持续增长。

---

## 十二、管道级特性全景图

```
┌─ 用户触发 / 定时扫描 ─────────────────────────────────────────────────┐
│  AddAllToQueue() / AddUserToQueue()                                   │
│     └─ ScannerQueue: 并发数 max_concurrent_tasks，Album 级并行        │
│                                                                      │
├─ 单 Album 扫描（串行） ───────────────────────────────────────────────┤
│  findMediaForAlbum()                                                  │
│     ├─ os.ReadDir() 遍历目录                                          │
│     ├─ L1: IgnorefileTask → .photoviewignore 过滤                     │
│     ├─ L2: CounterpartFilesTask → RAW+JPG 配对去重                    │
│     ├─ L3: AlbumScannerCache → 类型缓存、空文件检查                    │
│     │                                                                │
│     └─ 数据库事务包裹:                                                │
│        ScanMedia() → path_hash 查 DB 去重 → INSERT media              │
│        AfterMediaFound() → 逐个 Task 钩子:                            │
│          ├─ ExifTask → exif.Parse() → SaveEXIF()                      │
│          ├─ ProcessPhotoTask → 检查缓存（DB+磁盘）→ 生成缩略图/高清图  │
│          ├─ ProcessVideoTask → ffmpeg 转码 + ffprobe 元数据           │
│          ├─ FaceDetectionTask → 人脸检测                              │
│          ├─ BlurhashTask → 生成模糊哈希                               │
│          └─ ... 其他 Task                                             │
│                                                                      │
├─ 错误处理 ────────────────────────────────────────────────────────────┤
│  ├─ 单 Task 失败（如 EXIF 解析）→ log.Warn + 继续其他 Task            │
│  ├─ 单文件失败（如图片损坏）→ ScannerError（日志+通知）+ continue       │
│  └─ 数据库事务 → 任何错误自动 Rollback，不污染 DB                      │
│                                                                      │
├─ 内存控制 ────────────────────────────────────────────────────────────┤
│  ├─ MarkReader: 10KB 固定缓冲区                                       │
│  ├─ EncodeMediaData: 懒加载，按需填充                                 │
│  ├─ imagick: 单张处理 + defer Destroy                                 │
│  ├─ ffmpeg/exiftool: 外部进程，不落 Go 堆                              │
│  └─ max_concurrent_tasks: 控制并行度，限制总内存                      │
│                                                                      │
└─ 缓存与清理 ──────────────────────────────────────────────────────────┘
   ├─ AlbumScannerCache: 内存缓存，扫描结束释放                          │
   ├─ MediaCache: 磁盘缓存，目录结构 <album_id>/<media_id>/             │
   ├─ 缓存命中: DB 查 URL → os.Stat 磁盘文件 → 都有则跳过生成            │
   ├─ 缓存失效: os.Stat 不存在 → 重新生成；CleanupMedia 清理已删除文件    │
   └─ 失效触发: 每次扫描 Album 后自动清理                                │
```

