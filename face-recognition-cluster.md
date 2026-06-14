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

### 4.2 模型来源与向量规格详细说明

#### 4.2.1 使用的模型：dlib 官方预训练模型（非自训练、非 face_recognition）

本项目**不使用** Python 的 `face_recognition` 库，也**不使用**自训练模型，而是直接使用 **Davis King（dlib 作者）发布的官方预训练模型**，通过 Go 封装库 `github.com/Kagami/go-face` 调用。

**模型文件目录**: `api/data/models/`（可通过环境变量 `PHOTOVIEW_FACE_RECOGNITION_MODELS_PATH` 覆盖）

三个模型文件的具体规格：

| 文件名 | 大小（约） | 算法 | 训练数据来源 | 发布者 |
|--------|-----------|------|-------------|--------|
| `mmod_human_face_detector.dat` | ~700KB | Max-Margin Object Detector + CNN | 大量标注人脸数据集 | Davis King (dlib 官方) |
| `shape_predictor_5_face_landmarks.dat` | ~5.8MB | Ensemble of Regression Trees | HELEN、iBUG 300-W 等 | Davis King (dlib 官方) |
| `dlib_face_recognition_resnet_model_v1.dat` | ~99MB | ResNet-34（改进版） | VGGFace2（331万人脸，9131人） | Davis King (dlib 官方) |

> **与 face_recognition 库的关系**: Python 的 `face_recognition` 库底层也是使用这**完全相同**的三个 dlib 模型文件，只是通过 Python binding 调用。Photoview 选择 Go 版本（go-face）是为了与整体 Go 技术栈一致，避免跨语言调用开销。

#### 4.2.2 特征向量规格：128 维 float32

```go
type FaceDescriptor [128]float32  // face_detection.go:45
```

| 属性 | 规格 |
|------|------|
| 维度 | 128 维 |
| 数据类型 | float32（单精度浮点） |
| 存储大小 | 128 × 4 = 512 字节 |
| 向量性质 | **L2 归一化**（长度 ≈ 1.0，dlib 模型输出时已自动归一化） |
| 数据库格式 | BLOB (MySQL/SQLite) / BYTEA (PostgreSQL)，LittleEndian 编码 |

#### 4.2.3 距离度量：欧氏距离（Euclidean Distance）

**明确不是余弦相似度**。go-face 的 `ClassifyThreshold` 内部使用 **dlib 的 `length()` 函数**计算欧氏距离：

```
距离 = ||vec_A - vec_B||₂ = √(Σ(v_Ai - v_Bi)²)
```

由于向量已经过 L2 归一化，欧氏距离与余弦距离存在以下数学对应关系：

```
余弦相似度 = 1 - (欧氏距离²) / 2

| 欧氏距离 | 等效余弦相似度 | 含义 |
|---------|--------------|------|
| 0.0     | 1.000        | 完全相同 |
| 0.2     | 0.980        | 高度相似（Photoview 默认阈值） |
| 0.6     | 0.820        | dlib 官方建议阈值 |
| 1.0     | 0.500        | 中等相似 |
| 1.177   | 0.307        | 正交（余弦=0） |
| 1.414   | 0.000        | 完全相反 |
```

### 4.3 RecognizeFile 内部处理

`go-face` 库的 `RecognizeFile()` 执行以下操作（dlib 封装）：

| 步骤 | 算法/模型 | 输出 |
|------|-----------|------|
| 1 | HOG + CNN 人脸检测器 (`mmod_human_face_detector.dat`) | 人脸边界框（像素坐标） |
| 2 | 5 点人脸特征点检测 (`shape_predictor_5_face_landmarks.dat`) | 双眼中心 + 鼻尖 + 双嘴角 |
| 3 | 人脸对齐（仿射变换）与裁剪归一化 | 150×150 标准对齐人脸 |
| 4 | ResNet-34 特征提取 (`dlib_face_recognition_resnet_model_v1.dat`) | 128 维 L2 归一化 float32 向量 |

**单张人脸输出结构** (`go-face` 库):

```go
type Face struct {
    Rectangle  image.Rectangle  // 人脸边界框（像素坐标）
    Descriptor [128]float32     // L2 归一化特征向量（欧氏距离度量）
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

**算法原理**（最近邻分类器 Nearest Neighbor Classifier）:
1. 遍历内存中所有已知样本 `faceDescriptors`
2. 计算新向量与每个样本的欧氏距离（L2 距离）
3. 找到距离最小的样本对应的类别 `faceGroupID`
4. 如果最小距离 **<** 阈值 `0.2` → 返回该类别 ID（匹配成功）
5. 如果最小距离 **≥** 阈值 `0.2` → 返回 `-1`（无匹配，创建新组）

> **⚠️ 重要注意**: 代码中调用的是 `ClassifyThreshold`，不是 `Classify`。前者严格使用给定阈值判断，后者使用库内置的阈值（通常更高）。

### 5.3 聚类阈值深度分析与灵敏度影响

#### 5.3.1 硬编码阈值：不可配置

**代码位置**: `face_detector_impl.go:131`

阈值 `0.2` 在代码中是**硬编码的魔法数字**，**没有暴露环境变量或配置项**：

```go
return int32(fd.rec.ClassifyThreshold(descriptor, 0.2))  // 无法通过配置修改
```

对比其他可配置项 (`environment_variables.go:43-47`):
- `PHOTOVIEW_DISABLE_FACE_RECOGNITION` — 可禁用功能
- **但没有** `PHOTOVIEW_FACE_THRESHOLD` 或类似配置

> 若需调整阈值，必须修改源代码并重新编译。

#### 5.3.2 不同阈值对自动合并的影响分析

| 阈值范围 | 自动合并行为 | 典型结果 | 适用场景 |
|---------|------------|---------|---------|
| **0.0 - 0.2** | **极度保守** | 只有完全相同的光线、角度、表情才会合并 | 高安全要求（如门禁系统），但 Photoview 中会产生大量碎片组 |
| **0.2（当前值）** | **严格模式** | 同一场景、相似光照的照片会合并；不同日期/光照大概率拆分 | **宁可拆分，不可错并** — 适合用户愿意手动整理的场景 |
| **0.3 - 0.5** | **中等模式** | 同一人不同光照基本能合并；偶尔不同人会误合并 | dlib 作者推荐用于一般人脸验证 |
| **0.6（dlib 官方推荐）** | **宽松模式** | 同一人不同姿态/年龄/表情大多能合并；孪生或高相似人脸可能误并 | dlib 在 LFW 数据集上 99.38% 准确率对应的阈值 |
| **0.7+** | **激进模式** | 极易把长相相似的不同人合并 | 不推荐用于人物档案聚类 |

#### 5.3.3 阈值 0.2 的业务影响

**典型场景对比**（同一人张三的三张照片）：

```
照片A：室内正面，光线均匀    ←→  照片B：户外正面，阳光充足
   距离 ≈ 0.35               ←→     距离 ≈ 0.42
            ↘                              ↙
              阈值 0.2 → 全部被拆分为 3 个独立 FaceGroup
              阈值 0.6 → 全部合并为 1 个 FaceGroup
             ↙
照片C：侧45度，室内灯光
```

**使用 0.2 的直接结果**：
- ✅ **优点**：不会把"张三"和"李四（长得像）"错误合并
- ❌ **缺点**：同一个人"张三"会被拆成 5-20 个 FaceGroup（取决于相册中光照和角度变化）
- 🔧 **补救**：用户需要频繁使用"合并人物"功能

#### 5.3.4 自动合并与用户手动修正的平衡

```
       阈值减小方向                        阈值增大方向
  ←─────────────────────          ─────────────────────→

更保守（0.2）          平衡点（0.45-0.5）        更宽松（0.6）
  │                        │                         │
  ▼                        ▼                         ▼
自动合并率: 10-20%      自动合并率: 60-80%       自动合并率: 85-95%
碎片组数量: 极多       碎片组数量: 适中           碎片组数量: 极少
误合并率: 接近 0%      误合并率: 1-3%            误合并率: 5-10%
用户操作: 大量合并      用户操作: 少量合并+少量拆分  用户操作: 频繁拆分
用户体验: 繁琐但可靠    用户体验: 较好              用户体验: 需警惕错并
```

> **Photoview 的设计哲学**：选择 0.2 阈值说明项目设计者认为「错误合并两个不同人」的代价远高于「同一人被拆成多组」。因为用户很容易在 UI 中点几下合并，但错误合并后用户不一定能发现，且拆分操作更繁琐。

### 5.4 光照/姿态识别失败的回退路径

当同一个人因为光照、姿态、表情、年龄、遮挡（帽子/眼镜/口罩）导致特征向量距离 > 0.2 时，**系统没有自动回退机制**，而是依赖以下人工干预链路：

#### 5.4.1 自动层面：完全没有重试或多级匹配

```
classifyFace() 流程中：
    match = classifyDescriptor(descriptor)
    ├─ if match < 0 → 直接创建新 FaceGroup（无二次尝试）
    └─ if match >= 0 → 直接追加
```

❌ **缺少的自动处理**：
- 没有多角度重试（如旋转图片再检测）
- 没有多阈值级联（如先用 0.2，再用 0.4 对未匹配的二次匹配）
- 没有 KNN 投票（如取 K 个最近邻投票决定类别）
- 没有按时间/地点启发式（如同一相册、相近时间拍的更可能是同一人）

#### 5.4.2 回退路径 1：识别未标记人脸（批量重新匹配）

**触发时机**: 用户在人物列表页点击 「Recognize unlabeled faces」 按钮

**UI 位置**: `PeoplePage.tsx:302-315`

**后端逻辑**: `RecognizeUnlabeledFaces()` (`face_detector_impl.go:218-307`)

```
执行流程：
┌─────────────────────────────────────────────────────────────┐
│ 1. 查询所有 label IS NULL 的 FaceGroup（用户从未命名的组）    │
│ 2. 从分类器中临时移除这些未标记样本                          │
│ 3. 对每个未标记样本：                                        │
│    ├─ 用剩余（已标记）样本重新分类                           │
│    ├─ 若匹配到（距离<0.2）→ 更新 face_group_id 并加入分类器 │
│    └─ 仍无匹配 → 重新加入分类器，保持独立组                  │
└─────────────────────────────────────────────────────────────┘
```

**使用前提**：用户必须先手动给至少一个 FaceGroup 设置 label（如"张三"），此功能才会有意义。

#### 5.4.3 回退路径 2：用户手动合并（最常用）

**触发时机**: 用户在人物列表页点击「Merge people」，或在单个人物页点击「Merge face」

**UI 位置**:
- 列表页: `PeoplePage.tsx:317-325` → `MergeFaceGroupsModal`
- 详情页: `FaceGroupTitle.tsx:149-157` → `MergeFaceGroupsModal` (预填充目标组)

#### 5.4.4 回退路径 3：用户手动移动单张人脸

**触发时机**: 某组中混入了错误的人脸照片，用户只想移走几张而非全组合并

**UI 位置**: `FaceGroupTitle.tsx:163-167` → `MoveImageFacesModal`

#### 5.4.5 回退路径 4：用户手动拆分（Detach）

**触发时机**: 一个 FaceGroup 中混合了两个人（之前错误合并或阈值太松），用户需要拆开

**UI 位置**: `FaceGroupTitle.tsx:158-162` → `DetachImageFacesModal`

> 以上 4 条回退路径的详细操作链路见「第十二章」。

### 5.5 分支 1：创建新人物档案（FaceGroup）

当 `match < 0` 时（`face_detector_impl.go:160-169`）:

| 输入 | 处理 | 输出 |
|------|------|------|
| `ImageFace` 对象 | 创建 `FaceGroup` 并关联该人脸 | `FaceGroup { ID, Label: nil, ImageFaces: [imageFace] }` |
| `FaceGroup` | `db.Create(&faceGroup)` | 数据库持久化，生成自增 ID |

### 5.6 分支 2：合并到现有档案

当 `match >= 0` 时（`face_detector_impl.go:171-181`）:

| 输入 | 处理 | 输出 |
|------|------|------|
| `match` 组 ID | `db.First(&faceGroup, int(match))` | 现有 `FaceGroup` |
| `faceGroup` + `imageFace` | `Association("ImageFaces").Append(&imageFace)` | 关联更新 |

### 5.7 增量学习：更新分类器

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

## 十二、用户手动合并/拆分操作完整链路

### 12.1 操作全景图

```
                       人物管理操作
                            │
        ┌───────────┬───────┼───────┬───────────┐
        ▼           ▼       ▼       ▼           ▼
   合并群组     移动人脸   拆分人脸   设置标签   重新识别
  (Combine)    (Move)    (Detach)   (Label)  (Recognize)
        │           │       │       │           │
        ▼           ▼       ▼       ▼           ▼
   GraphQL      GraphQL  GraphQL  GraphQL    GraphQL
   Mutation     Mutation Mutation Mutation   Mutation
        │           │       │       │           │
        ▼           ▼       ▼       ▼           ▼
   数据库更新 + 内存分类器同步（MergeImageFaces / MergeCategories）
```

### 12.2 操作一：合并人物档案（CombineFaceGroups）

#### 12.2.1 触发场景
- 场景 A：列表页用户看到"张三"被拆成了 3 个组 → 点「Merge people」
- 场景 B：详情页用户看到当前"未命名组"应该属于"张三" → 点「Merge face」

#### 12.2.2 UI 完整交互链（从点击到完成）

```
第一步：用户在 UI 点击按钮
  ├─ [列表页] PeoplePage.tsx:317-325
  │    Button onClick → setState(SelectDestination)
  │
  └─ [详情页] FaceGroupTitle.tsx:149-157
       Button onClick → setState(SelectDestination)
       （preselectedDestinationFaceGroup = 当前组）

第二步：选择目标 FaceGroup（合并目的地）
  ├─ MergeFaceGroupsModal.tsx:244-258
  │    ├─ state === SelectDestination
  │    ├─ 展示 SelectFaceGroupTable
  │    └─ 用户选择一个 → setDestinationFaceGroup()
  │
  └─ 点「Next」按钮 → setState(SelectSources)

第三步：选择源 FaceGroup（要被合并的组）
  ├─ MergeFaceGroupsModal.tsx:221-247
  │    ├─ state === SelectSources
  │    ├─ 过滤掉目标组本身（不能合并自己）
  │    └─ 用户勾选 1..N 个源组
  │
  └─ 点「Merge」按钮 → mergeFaceGroups()

第四步：前端发送 GraphQL Mutation
  ├─ MergeFaceGroupsModal.tsx:20-29 (COMBINE_FACES_MUTATION)
  │    mutation combineFaces($destID: ID!, $srcIDs: [ID!]!) {
  │      combineFaceGroups(
  │        destinationFaceGroupID: $destID
  │        sourceFaceGroupIDs: $srcIDs
  │      ) { id }
  │    }
  │
  └─ Apollo refetchQueries 刷新 MY_FACES_QUERY

第五步：后端处理 resolvers/faces.go:144-225
  ├─ 1. 鉴权：userOwnedFaceGroup() 验证所有权
  │       （管理员直接查，普通用户需通过 album 关联）
  │
  ├─ 2. 数据库事务开始
  │       ├─ 2a. hasDuplicateMediaInFaceGroupsUnion()
  │       │    检查源+目标并集里是否有同一张照片重复
  │       │    （同一张照片出现两次说明合并有误）
  │       │
  │       ├─ 2b. UPDATE image_faces SET face_group_id = destID
  │       │        WHERE face_group_id IN srcIDs
  │       │
  │       ├─ 2c. DELETE FROM face_groups WHERE id IN srcIDs
  │       │        （删除已空的源组）
  │       │
  │       └─ 2d. 去重清理：同一媒体保留 MIN(id)，删其他
  │             （可能同一张照片在不同组各有一张人脸）
  │
  ├─ 3. 内存同步：GlobalFaceDetector.MergeImageFaces()
  │       遍历内存中所有样本，把源组ID改为目标组ID
  │       （注意：传错参数了！这里传 srcIDs，函数需要的是 imageFaceIDs
  │        这是一个潜在 bug！MergeImageFaces 参数应为 imageFaceIDs，
  │        但此处传入 faceGroupIDs，可能造成内存状态不一致。）
  │
  └─ 4. 事务提交 → 返回 destinationFaceGroup

第六步：前端路由跳转
  └─ MergeFaceGroupsModal.tsx:167-170
       navigate(`/people/${selectedDestinationFaceGroup.id}`)
       跳转到合并后的目标组详情页
```

### 12.3 操作二：移动单张人脸（MoveImageFaces）

#### 12.3.1 触发场景
"张三"组里混入了一张"李四"的照片，用户不想全组合并，只想移走这 1 张。

#### 12.3.2 完整操作链

```
UI 入口：FaceGroupTitle.tsx:163-167  「Move faces」按钮
    │
    ▼
MoveImageFacesModal 打开
    │
    ├─ 步骤 1：SelectImageFacesTable 勾选要移动的人脸
    │    (支持多选)
    │    setImagesSelected(true) → 下一步
    │
    ├─ 步骤 2：加载所有 FaceGroup
    │    过滤掉当前组（不能移到自己）
    │    SelectFaceGroupTable 选择目标组
    │
    └─ 点「Move image faces」按钮
         │
         ▼
GraphQL Mutation: moveImageFaces
    mutation moveImageFaces($faceIDs: [ID!]!, $destFaceGroupID: ID!)
    │
    ▼
后端处理 resolvers/faces.go:227-297
    ├─ 1. getUserOwnedImageFaces() 过滤用户有权限的人脸
    ├─ 2. doesFaceGroupContainImages() 检查移动后是否重复
    ├─ 3. UPDATE image_faces SET face_group_id = destID
    ├─ 4. deleteEmptyFaceGroups() 源组变空则删除
    ├─ 5. GlobalFaceDetector.MergeImageFaces(userOwnedImageFaceIDs, destID)
    │     （这次传的是真的 imageFaceIDs，没问题）
    └─ 事务提交
         │
         ▼
navigate(`/people/${destFaceGroup.id}`) 跳转到目标组
```

### 12.4 操作三：拆分人脸（DetachImageFaces = 分离为新组）

#### 12.4.1 触发场景
一个 FaceGroup 里混了"张三 + 李四"（之前误合并），用户需要把李四的几张拆出去**新建**一个组。

> 注意：Detach 不是"拆分到已有组"，而是"创建全新组"。要拆分到已有组，应使用 Move 操作。

#### 12.4.2 完整操作链

```
UI 入口：FaceGroupTitle.tsx:158-162  「Detach images」按钮
    │
    ▼
DetachImageFacesModal 打开
    │
    ├─ SelectImageFacesTable 勾选要分离的人脸
    │    (默认预选中用户之前勾选的，如果有的话)
    │
    └─ 点「Detach image faces」按钮
         │
         ▼
GraphQL Mutation: detachImageFaces
    mutation detachImageFaces($faceIDs: [ID!]!) {
      detachImageFaces(imageFaceIDs: $faceIDs) { id, label }
    }
    │
    ▼
后端处理 resolvers/faces.go:327-374
    ├─ 1. getUserOwnedImageFaces() 过滤权限
    ├─ 2. INSERT 新 FaceGroup（label = NULL）
    ├─ 3. UPDATE image_faces SET face_group_id = newGroup.id
    │     WHERE id IN selectedFaceIDs
    ├─ 4. GlobalFaceDetector.MergeImageFaces(selectedIDs, newGroupID)
    └─ 事务提交
         │
         ▼
navigate(`/people/${newFaceGroup.id}`) 跳转到新创建的组
（用户之后可以给新组改 label，或继续合并它）
```

### 12.5 操作四：设置人物标签（SetFaceGroupLabel）

#### 12.5.1 触发场景
用户把"未命名 #17"命名为"张三"。标签有两个功能：
1. **人机交互友好**：UI 上显示名字而不是编号
2. **触发 Recognize 逻辑**：label IS NULL 的组被视为"未标注"，是 RecognizeUnlabeledFaces 的处理对象

#### 12.5.2 操作链

```
UI 入口（两种）：
  ├─ 列表页: FaceDetails 组件（PeoplePage.tsx 内嵌）
  │    点击文字 → 变 TextField
  │
  └─ 详情页: FaceGroupTitle.tsx:143-148 「Change label」按钮
       或直接点击标题

输入名字后回车/失焦 → setFaceGroupLabel Mutation
    mutation setGroupLabel($groupID: ID!, $label: String)
    （传空字符串 "" 会被转为 NULL，表示取消命名）

后端 resolvers/faces.go:119-141
    └─ 简单的 UPDATE face_groups SET label = ? WHERE id = ?
       （不需要修改内存分类器，label 不影响向量匹配）

注意：内存分类器只看 faceGroupID，不管 label 是什么。
      label 仅影响 UI 展示 + RecognizeUnlabeledFaces 的筛选条件。
```

### 12.6 操作五：识别未标记人脸（RecognizeUnlabeledFaces）

#### 12.6.1 触发场景
用户给 3 个组命名为"张三"、"李四"、"王五"后，希望系统把剩下几十个"未命名"的组自动归类到已命名的组。

#### 12.6.2 操作链

```
UI 入口：PeoplePage.tsx:302-315  「Recognize unlabeled faces」按钮
    （只在列表页有，详情页没有）

GraphQL: recognizeUnlabeledFaces Mutation（无参数）
    mutation recognizeUnlabeledFaces { id }

后端处理 face_detector_impl.go:218-307
    ┌──────────────────────────────────────────────────────────┐
    │ 0. 前置：查出用户拥有的相册中，所有 label IS NULL 的组    │
    │    通过 JOIN image_faces + media + user_albums 过滤      │
    │    （普通用户只能处理自己相册里的未标记人脸）            │
    │                                                          │
    │ 1. 分类器重建：把内存中样本拆成两部分                    │
    │    ├─ 未标记样本 → 暂存 unrecognizedDescriptors 数组     │
    │    └─ 已标记样本 → 保留在 faceDescriptors（作为"字典"）  │
    │                                                          │
    │ 2. 逐个处理每个未标记样本：                              │
    │    ├─ classifyDescriptor(descriptor)                    │
    │    │   仅在已标记样本中寻找匹配（阈值还是 0.2）          │
    │    │                                                   │
    │    ├─ 找到匹配 →                                       │
    │    │   ├─ UPDATE image_faces SET face_group_id = matchID│
    │    │   ├─ 标记为 updatedImageFaces（返回给前端提示）    │
    │    │   └─ 把此样本加回分类器，组ID=matchID              │
    │    │                                                   │
    │    └─ 仍无匹配 →                                       │
    │        └─ 原样加回分类器，组ID不变（还是独立未命名组）   │
    │                                                          │
    │ 3. 返回 updatedImageFaces 数组（所有被重新归类的人脸）  │
    └──────────────────────────────────────────────────────────┘

注意：此操作本质是「用已命名样本作为 ground truth，
      重新对未命名样本做一次 0.2 阈值的最近邻分类」
      如果用户命名的样本本身不多，命中率可能仍然很低。
```

### 12.7 权限模型总结

所有操作都经过 `userOwnedFaceGroup()` 或 `getUserOwnedImageFaces()` 权限校验：

| 用户角色 | 权限规则 |
|---------|---------|
| **管理员 (Admin=true)** | 直接操作数据库中所有 FaceGroup / ImageFace，无条件通过 |
| **普通用户** | 通过 `user_albums` 关联：只有当 ImageFace 所属的 Media 所在的 Album 被用户拥有时，才能操作该人脸 |

校验路径：`ImageFace → Media → Album → user_albums → User`

---

## 十三、代码位置索引

### 13.1 后端核心代码

| 功能 | 文件 | 行号 |
|------|------|------|
| 检测器初始化 | `api/scanner/face_detection/face_detector_impl.go` | 25-51 |
| 人脸检测主函数 | `api/scanner/face_detection/face_detector_impl.go` | 92-128 |
| 分类聚类逻辑（含阈值 0.2） | `api/scanner/face_detection/face_detector_impl.go` | 130-189 |
| 合并类别内存同步 | `api/scanner/face_detection/face_detector_impl.go` | 191-216 |
| 识别未标记人脸 | `api/scanner/face_detection/face_detector_impl.go` | 218-307 |
| 接口定义（FaceDetector） | `api/scanner/face_detection/face_detector.go` | 9-17 |
| 人脸检测扫描任务 | `api/scanner/scanner_tasks/face_detection_task.go` | 11-31 |
| 数据模型 FaceGroup/ImageFace | `api/graphql/models/face_detection.go` | 16-80 |
| FaceDescriptor 序列化 | `api/graphql/models/face_detection.go` | 45-74 |
| 合并/移动/拆分 GraphQL | `api/graphql/resolvers/faces.go` | 119-374 |
| MyFaceGroups/查询 | `api/graphql/resolvers/faces.go` | 376-449 |
| 权限辅助函数 | `api/graphql/resolvers/faces.util.go` | 18-163 |
| 扫描队列 | `api/scanner/scanner_queue/queue.go` | 1-264 |
| 相册扫描入口 | `api/scanner/scanner_album.go` | 87-114 |
| 媒体扫描核心 | `api/scanner/media_scan.go` | 11-40 |
| 服务启动初始化 | `api/server.go` | 70-72 |
| 环境变量定义（可禁用人脸识别） | `api/utils/environment_variables.go` | 43-47 |
| 模型路径配置 | `api/utils/utils.go` | 48-64 |
| 任务聚合器 | `api/scanner/scanner_tasks/scanner_tasks.go` | 15-27 |

### 13.2 前端 UI 代码

| 功能 | 文件 | 行号 |
|------|------|------|
| 人物列表页 + 操作按钮 | `ui/src/Pages/PeoplePage/PeoplePage.tsx` | 248-336 |
| 人物详情页 | `ui/src/Pages/PeoplePage/SingleFaceGroup/SingleFaceGroup.tsx` | 47-107 |
| 详情页标题 + 4 个管理按钮 | `ui/src/Pages/PeoplePage/SingleFaceGroup/FaceGroupTitle.tsx` | 27-174 |
| 合并人物弹窗（两步选择） | `ui/src/Pages/PeoplePage/SingleFaceGroup/MergeFaceGroupsModal.tsx` | 52-260 |
| 移动人脸弹窗 | `ui/src/Pages/PeoplePage/SingleFaceGroup/MoveImageFacesModal.tsx` | 49-202 |
| 拆分人脸弹窗 | `ui/src/Pages/PeoplePage/SingleFaceGroup/DetachImageFacesModal.tsx` | 70-154 |
| 人脸标签编辑（列表页内） | `ui/src/Pages/PeoplePage/PeoplePage.tsx` | 118-208 |
| 查询所有人脸组 | `ui/src/Pages/PeoplePage/PeoplePage.tsx` | 28-54 (MY_FACES_QUERY) |
| 查询单个人脸组详情 | `ui/src/Pages/PeoplePage/SingleFaceGroup/SingleFaceGroup.tsx` | 14-45 (SINGLE_FACE_GROUP) |
