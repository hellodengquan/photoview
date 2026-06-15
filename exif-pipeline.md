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
