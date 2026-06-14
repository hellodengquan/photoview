# 人脸识别聚类代码链路分析

本文档详细分析 Photoview 项目中从单张照片产生特征向量，再通过聚类合并成人物档案的完整代码链路。

---

## 一、整体架构概览

```
单张照片 → 扫描队列 → 照片处理 → 人脸检测 → 特征向量提取 → 分类聚类 → 人物档案
   ↑                                                                       ↓
   └───────────────────────────────────────────────────────────────────────┘
                          增量学习（新样本加入分类器）
```

### 核心模块关系

| 模块 | 文件路径 | 职责 |
|------|----------|------|
| 人脸识别核心 | `api/scanner/face_detection/` | 人脸检测、特征提取、分类聚类 |
| 扫描任务 | `api/scanner/scanner_tasks/face_detection_task.go` | 扫描流程中的人脸检测任务 |
| 数据模型 | `api/graphql/models/face_detection.go` | `FaceGroup`、`ImageFace` 等数据结构 |
| GraphQL 接口 | `api/graphql/resolvers/faces.go` | 人脸组管理的 API 层 |
| 扫描队列 | `api/scanner/scanner_queue/queue.go` | 异步扫描任务调度 |
| 第三方库 | `github.com/Kagami/go-face` | dlib 封装，提供人脸检测与识别能力 |

---

## 二、第一阶段：系统初始化

### 2.1 人脸检测器初始化

**入口文件**: `api/server.go:70`

```go
if err := face_detection.InitializeFaceDetector(db); err != nil {
    log.Panicf("Could not initialize face detector: %s\n", err)
}
```

**初始化流程** (`api/scanner/face_detection/face_detector_impl.go:25-51`):

| 步骤 | 输入 | 处理 | 输出 |
|------|------|------|------|
| 1 | 环境变量 `DISABLE_FACE_RECOGNITION` | 检查是否禁用人脸识别 | 布尔值 |
| 2 | 模型文件路径 `FaceRecognitionModelsPath()` | 加载 dlib 模型文件 | `face.Recognizer` 实例 |
| 3 | 数据库连接 `db` | 从数据库读取已有的人脸样本 | 三元组 `(descriptors, groupIDs, imageFaceIDs)` |
| 4 | 上述结果 | 初始化全局检测器 `GlobalFaceDetector` | 内存中的分类器 |

**关键数据结构** (`face_detector_impl.go:17-23`):

```go
type faceDetector struct {
    mutex           sync.Mutex
    rec             *face.Recognizer      // go-face 识别器
    faceDescriptors []face.Descriptor     // [128]float32 特征向量数组
    faceGroupIDs    []int32               // 对应的人脸组 ID
    imageFaceIDs    []int                 // 对应的人脸图像 ID
}
```

### 2.2 从数据库加载样本

**函数**: `getSamplesFromDatabase()` (`face_detector_impl.go:53-72`)

| 输入 | 处理 | 输出 |
|------|------|------|
| `*gorm.DB` | 1. 查询所有 `ImageFace` 记录<br>2. 转换 `FaceDescriptor` 为 `face.Descriptor`<br>3. 并行提取 `faceGroupIDs` 和 `imageFaceIDs` | `samples []face.Descriptor`<br>`faceGroupIDs []int32`<br>`imageFaceIDs []int` |

**数据模型转换** (`face_detection.go:45`):

```go
type FaceDescriptor [128]float32  // 与 go-face 的 Descriptor 完全一致
```

数据库存储格式：BLOB（二进制），使用 `binary.LittleEndian` 编码。

---

## 三、第二阶段：照片扫描与处理

### 3.1 扫描任务触发流程

**触发入口**: 用户通过 GraphQL 触发扫描，或定期扫描器自动触发

| 阶段 | 文件 | 关键函数 |
|------|------|----------|
| 队列添加 | `scanner_queue/queue.go:221-239` | `AddUserToQueue()` |
| 任务调度 | `scanner_queue/queue.go:136-192` | `processQueue()` |
| 相册扫描 | `scanner/scanner_album.go:87-114` | `ScanAlbum()` |
| 媒体扫描 | `scanner/media_scan.go:11-40` | `scanMedia()` |

### 3.2 人脸检测任务

**任务定义**: `scanner_tasks/face_detection_task.go:11-31`

```go
type FaceDetectionTask struct {
    scanner_task.ScannerTaskBase
}

func (t FaceDetectionTask) AfterProcessMedia(...) error {
    if didProcess && mediaData.Media.Type == models.MediaTypePhoto {
        if face_detection.GlobalFaceDetector == nil {
            return nil
        }
        return face_detection.GlobalFaceDetector.DetectFaces(ctx.GetDB(), media)
    }
    return nil
}
```

| 触发条件 | 说明 |
|----------|------|
| `didProcess == true` | 照片被成功处理（生成了缩略图） |
| `Type == MediaTypePhoto` | 只处理照片，不处理视频 |
| `GlobalFaceDetector != nil` | 人脸识别功能已启用 |

---

## 四、第三阶段：人脸检测与特征向量提取

### 4.1 DetectFaces 主流程

**函数**: `DetectFaces()` (`face_detector_impl.go:92-128`)

| 步骤 | 输入 | 处理 | 输出 |
|------|------|------|------|
| 1 | `*gorm.DB`, `*models.Media` | 预加载 `MediaURL` 关联 | `media` 带完整 URL |
| 2 | `media.MediaURL` | 查找用途为 `PhotoThumbnail` 的缩略图 | `*models.MediaURL` |
| 3 | `thumbnailURL` | 获取缩略图本地缓存路径 | `thumbnailPath string` |
| 4 | 缩略图文件 | `fd.rec.RecognizeFile(thumbnailPath)` | `[]face.Face`（检测到的人脸列表） |
| 5 | 每张人脸 | 调用 `classifyFace()` 进行分类 | 无（内部持久化） |

### 4.2 RecognizeFile 内部处理

`go-face` 库的 `RecognizeFile()` 执行以下操作（dlib 封装）：

| 步骤 | 算法/模型 | 输出 |
|------|-----------|------|
| 1 | HOG + SVM 人脸检测器 (`mmod_human_face_detector.dat`) | 人脸边界框 |
| 2 | 5 点人脸特征点检测 (`shape_predictor_5_face_landmarks.dat`) | 面部关键点 |
| 3 | 人脸对齐与裁剪 | 归一化的人脸图像 |
| 4 | ResNet 特征提取 (`dlib_face_recognition_resnet_model_v1.dat`) | 128 维 float32 特征向量 |

**单张人脸输出结构** (`go-face` 库):

```go
type Face struct {
    Rectangle  image.Rectangle  // 人脸边界框（像素坐标）
    Descriptor [128]float32     // 特征向量
}
```

---

## 五、第四阶段：分类聚类与人物档案生成

### 5.1 classifyFace 分类流程

**函数**: `classifyFace()` (`face_detector_impl.go:134-189`)

| 步骤 | 输入 | 处理 | 输出 |
|------|------|------|------|
| 1 | `face.Descriptor` | `fd.classifyDescriptor(descriptor)` | `match int32` 匹配的组 ID |
| 2 | 图像路径 | 获取照片尺寸 | `dimension {Width, Height}` |
| 3 | 像素坐标 + 尺寸 | 转换为相对坐标（0-1 比例） | `FaceRectangle` |
| 4 | 上述数据 | 构建 `ImageFace` 对象 | `models.ImageFace` |
| 5 | `match` 值 | **分支 1**: `match < 0` 无匹配 → 创建新 `FaceGroup`<br>**分支 2**: `match >= 0` 有匹配 → 追加到现有 `FaceGroup` | 持久化到数据库 |
| 6 | 新样本 | 更新内存中的样本列表和分类器 | `SetSamples()` |

### 5.2 分类算法：ClassifyThreshold

**函数**: `classifyDescriptor()` (`face_detector_impl.go:130-132`)

```go
func (fd *faceDetector) classifyDescriptor(descriptor face.Descriptor) int32 {
    return int32(fd.rec.ClassifyThreshold(descriptor, 0.2))
}
```

**算法原理**:
- 计算新特征向量与所有已知样本的欧氏距离
- 找到距离最小的类别
- 如果最小距离 < 阈值 `0.2`，返回该类别 ID
- 否则返回 `-1`（无匹配）

> **关键参数**: 阈值 `0.2` 是人脸识别的核心参数。dlib 官方建议阈值为 `0.6`，但此处使用更严格的 `0.2`，说明该项目对识别精度要求更高，宁可创建新组也不误匹配。

### 5.3 分支 1：创建新人物档案（FaceGroup）

当 `match < 0` 时（`face_detector_impl.go:160-169`）:

| 输入 | 处理 | 输出 |
|------|------|------|
| `ImageFace` 对象 | 创建 `FaceGroup` 并关联该人脸 | `FaceGroup { ID, Label: nil, ImageFaces: [imageFace] }` |
| `FaceGroup` | `db.Create(&faceGroup)` | 数据库持久化，生成自增 ID |

### 5.4 分支 2：合并到现有档案

当 `match >= 0` 时（`face_detector_impl.go:171-181`）:

| 输入 | 处理 | 输出 |
|------|------|------|
| `match` 组 ID | `db.First(&faceGroup, int(match))` | 现有 `FaceGroup` |
| `faceGroup` + `imageFace` | `Association("ImageFaces").Append(&imageFace)` | 关联更新 |

### 5.5 增量学习：更新分类器

**关键步骤** (`face_detector_impl.go:183-187`):

```go
fd.faceDescriptors = append(fd.faceDescriptors, face.Descriptor)
fd.faceGroupIDs = append(fd.faceGroupIDs, int32(faceGroup.ID))
fd.imageFaceIDs = append(fd.imageFaceIDs, imageFace.ID)
fd.rec.SetSamples(fd.faceDescriptors, fd.faceGroupIDs)
```

这是**增量学习**机制：每次新检测到人脸后，立即将其加入样本库，后续检测可以识别这张脸。

---

## 六、数据模型详解

### 6.1 FaceGroup（人物档案）

**定义**: `face_detection.go:16-20`

```go
type FaceGroup struct {
    Model      // 包含 ID, CreatedAt, UpdatedAt
    Label      *string       // 用户可编辑的标签（如"张三"）
    ImageFaces []ImageFace   // 该人物的所有人脸
}
```

**数据库表**: `face_groups`

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | INTEGER | 主键 |
| `label` | VARCHAR | NULL 表示未命名 |

### 6.2 ImageFace（单张人脸）

**定义**: `face_detection.go:22-30`

```go
type ImageFace struct {
    Model
    FaceGroupID int            // 外键：所属人脸组
    FaceGroup   *FaceGroup     // 关联
    MediaID     int            // 外键：所属照片
    Media       Media          // 关联
    Descriptor  FaceDescriptor // 128 维特征向量
    Rectangle   FaceRectangle  // 人脸位置
}
```

**数据库表**: `image_faces`

| 字段 | 类型 | 说明 |
|------|------|------|
| `descriptor` | BLOB / BYTEA | 128 × 4 = 512 字节二进制 |
| `rectangle` | VARCHAR(64) | `"minX:maxX:minY:maxY"` 字符串格式 |

### 6.3 FaceRectangle 坐标转换

**像素坐标转相对坐标** (`face_detector_impl.go:148-154`):

```go
Rectangle: models.FaceRectangle{
    MinX: float64(face.Rectangle.Min.X) / float64(dimension.Width),
    MaxX: float64(face.Rectangle.Max.X) / float64(dimension.Width),
    MinY: float64(face.Rectangle.Min.Y) / float64(dimension.Height),
    MaxY: float64(face.Rectangle.Max.Y) / float64(dimension.Height),
}
```

**设计意图**: 使用相对坐标而非绝对像素坐标，这样即使缩略图尺寸变化，人脸位置仍然有效。

---

## 七、人物档案管理操作

### 7.1 合并人脸组（CombineFaceGroups）

**GraphQL 接口**: `resolvers/faces.go:144-225`

| 步骤 | 处理 |
|------|------|
| 1 | 验证用户权限，确保源和目标都属于用户 |
| 2 | 检查合并后是否有重复媒体（同一张照片不能在同一组中出现两次） |
| 3 | 更新所有源组的 `ImageFace.face_group_id` 为目标组 ID |
| 4 | 删除源组记录 |
| 5 | 清理同一媒体的重复人脸（保留 ID 最小的） |
| 6 | 更新内存分类器：`MergeImageFaces()` |

**去重逻辑** (`faces.go:203-214`):

```go
subQuery := tx.Model(&models.ImageFace{}).
    Select("MIN(id)").
    Where("face_group_id = ?", destinationFaceGroup.ID).
    Group("media_id")

err = tx.Where("face_group_id = ?", destinationFaceGroup.ID).
    Where("id NOT IN (?)", subQuery).
    Delete(&models.ImageFace{}).Error
```

### 7.2 移动人脸（MoveImageFaces）

**接口**: `resolvers/faces.go:227-297`

将指定 `ImageFace` 从当前组移动到目标组，类似合并但操作粒度更细。

### 7.3 分离人脸（DetachImageFaces）

**接口**: `resolvers/faces.go:327-374`

将指定 `ImageFace` 从当前组分离，创建一个新的 `FaceGroup`。

### 7.4 重新识别未标记人脸（RecognizeUnlabeledFaces）

**接口**: `resolvers/faces.go:299-325`

**核心逻辑** (`face_detector_impl.go:218-307`):

| 步骤 | 处理 |
|------|------|
| 1 | 查询所有 `label IS NULL` 的人脸组 |
| 2 | 将未标记样本从分类器中临时移除 |
| 3 | 对每个未标记样本，使用剩余样本重新分类 |
| 4 | 如果找到匹配，更新数据库并加入分类器 |
| 5 | 如果仍无匹配，重新加入分类器作为独立组 |

**用途**: 用户标记某些人脸后，可触发此功能让系统尝试将未标记的人脸自动归类到已标记的组。

---

## 八、内存与数据库同步机制

### 8.1 ReloadFacesFromDatabase

**函数**: `face_detector_impl.go:74-89`

当外部直接修改数据库后，调用此方法重新加载所有样本到内存。

### 8.2 MergeCategories

**函数**: `face_detector_impl.go:191-200`

将内存中所有 `sourceID` 替换为 `destID`，用于组合并后的内存同步。

### 8.3 MergeImageFaces

**函数**: `face_detector_impl.go:202-216`

将指定 `imageFaceIDs` 的组 ID 更新为 `destFaceGroupID`，用于移动/分离操作后的内存同步。

---

## 九、完整调用链路总结

### 9.1 新照片处理链路

```
用户触发扫描
    ↓
AddUserToQueue() [scanner_queue/queue.go:221]
    ↓
ScannerJob.Run() [scanner_queue/queue.go:36]
    ↓
ScanAlbum() [scanner/scanner_album.go:87]
    ↓
findMediaForAlbum() [scanner/scanner_album.go:116]
    ↓
ScanMedia() [scanner/scanner_media.go]
    ↓
scanMedia() [scanner/media_scan.go:11]
    ↓
scanner_tasks.Tasks.ProcessMedia() [scanner/media_scan.go:23]
    ↓
ProcessPhotoTask.ProcessMedia() [processing_tasks/process_photo_task.go]
    ↓ 生成缩略图
scanner_tasks.Tasks.AfterProcessMedia() [scanner/media_scan.go:28]
    ↓
FaceDetectionTask.AfterProcessMedia() [face_detection_task.go:15]
    ↓
GlobalFaceDetector.DetectFaces() [face_detector_impl.go:92]
    ├─→ rec.RecognizeFile() → []face.Face （人脸检测 + 特征提取）
    └─→ 对每张脸: classifyFace() [face_detector_impl.go:134]
        ├─→ classifyDescriptor() → ClassifyThreshold(0.2) （分类）
        ├─→ match < 0: 创建新 FaceGroup
        ├─→ match >= 0: 追加到现有 FaceGroup
        └─→ SetSamples() 更新分类器
```

### 9.2 数据流转表

| 阶段 | 数据形态 | 存储位置 |
|------|----------|----------|
| 原始输入 | 照片文件（JPG/PNG 等） | 文件系统 |
| 缩略图 | 压缩后的图像文件 | 缓存目录 |
| 人脸检测 | `[]face.Face`（边界框 + 特征向量） | 内存（go-face） |
| 坐标转换 | `FaceRectangle`（相对坐标） | 内存 |
| 特征向量 | `FaceDescriptor = [128]float32` | 内存 → 数据库（BLOB） |
| 人脸实例 | `ImageFace` | 数据库 `image_faces` 表 |
| 人物档案 | `FaceGroup` | 数据库 `face_groups` 表 |
| 分类器样本 | `[]face.Descriptor` + `[]int32` | 内存（faceDetector） |

---

## 十、关键设计要点

### 10.1 增量聚类策略

该系统采用的是**在线增量聚类**而非离线批量聚类：
- 每次检测新人脸时立即进行分类
- 无匹配时创建新簇（人脸组）
- 新样本立即加入分类器，用于后续分类

**优点**:
- 实时性好，新照片上传后立即可以看到人脸分组
- 无需批量计算，资源占用低

**缺点**:
- 聚类结果依赖处理顺序，早期误分类会累积
- 没有全局优化，可能出现"同一人多组"的情况
- 需要用户手动合并来修正

### 10.2 阈值选择

使用 `0.2` 作为分类阈值（dlib 官方推荐 `0.6`）：
- 更小的阈值 = 更高的精度，更低的召回率
- 倾向于"拆分成多个组"而非"错误合并"
- 用户可以手动合并，但错误合并难以自动修正

### 10.3 内存-数据库双写

所有分类操作都会同时更新：
1. 数据库（持久化）
2. 内存中的分类器（用于后续分类）

这种设计避免了每次分类都查询数据库，性能更好，但需要保证两者同步。

### 10.4 并发安全

`faceDetector` 使用 `sync.Mutex` 保护所有操作，确保并发扫描时分类器状态一致。

---

## 十一、代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 检测器初始化 | `api/scanner/face_detection/face_detector_impl.go` | 25-51 |
| 人脸检测主函数 | `api/scanner/face_detection/face_detector_impl.go` | 92-128 |
| 分类聚类逻辑 | `api/scanner/face_detection/face_detector_impl.go` | 130-189 |
| 人脸检测任务 | `api/scanner/scanner_tasks/face_detection_task.go` | 11-31 |
| 数据模型定义 | `api/graphql/models/face_detection.go` | 1-130 |
| 人脸组 GraphQL 接口 | `api/graphql/resolvers/faces.go` | 1-458 |
| 扫描队列 | `api/scanner/scanner_queue/queue.go` | 1-264 |
| 相册扫描 | `api/scanner/scanner_album.go` | 87-114 |
| 媒体扫描 | `api/scanner/media_scan.go` | 11-40 |
| 服务启动初始化 | `api/server.go` | 70-72 |
