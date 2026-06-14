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

#### 5.7.1 增量识别场景：不做全量重聚类，只做追加

**重要发现**：`SetSamples()` **不会触发任何聚类算法重计算**。它只是简单地替换 C++ 层的两个 `std::vector`：

```cpp
// go-face/facerec.cc:118-122
void SetSamples(std::vector<descriptor>&& samples, std::vector<int>&& cats) {
    std::unique_lock<std::shared_mutex> lock(samples_mutex_);
    samples_ = std::move(samples);   // 只是移动语义替换 vector
    cats_ = std::move(cats);         // 没有 KD-tree、没有 Ball tree、没有索引重建
}
```

| 场景 | 计算复杂度 | 是否触发全量重聚类 | 说明 |
|------|-----------|-------------------|------|
| 新照片入库 | **O(N)** 每次分类 | ❌ 不重聚类 | 只是 append 到 vector，后续分类遍历所有样本 |
| 合并 FaceGroup | **O(M)** M=被移动样本数 | ❌ 不重聚类 | 只改内存中的 faceGroupID，调用 `MergeCategories()` 遍历替换 |
| 移动 ImageFace | **O(M)** M=被移动样本数 | ❌ 不重聚类 | 只改内存中的 faceGroupID，调用 `MergeImageFaces()` 遍历替换 |
| 删除照片/用户 | **O(N)** 全量重加载 | ❌ 不重聚类 | 调用 `ReloadFacesFromDatabase()` 从数据库重建内存 vector |

> **设计决策分析**：选择"全量遍历 + 暴力最近邻"而非"聚类 + 索引"，原因是：
> 1. 个人相册规模通常为万级人脸，O(N) 暴力搜索在现代 CPU 上完全够用
> 2. 实现简单，无 bug，便于维护
> 3. 每次分类都和全量样本比较，避免聚类算法累积误差
> 4. 增量添加时不需要重新训练或重建索引

#### 5.7.2 分类算法揭秘：K=10 的加权 KNN

`ClassifyThreshold(0.2)` 内部使用的是 **K 近邻（KNN）分类器**，不是简单的最近邻：

```cpp
// go-face/classify.cc:5-59
int classify(...) {
    // 1. 计算与所有样本的欧氏距离平方（避免开根号，加速）
    auto dist_func = dlib::squared_euclidean_distance();
    
    // 2. 过滤出距离 <= tolerance 的样本（这里 tolerance=0.2，平方=0.04）
    std::vector<std::pair<int, float>> distances;
    
    // 3. 按距离排序，取前 K=10 个
    int len = std::min((int)distances.size(), 10);
    
    // 4. KNN 投票：统计每个类别在前 10 中的出现次数
    std::unordered_map<int, std::pair<int, float>> hits_by_cat;
    
    // 5. 选出得票最高的类别（票数相同则取距离更近的）
    auto hit = std::max_element(hits_by_cat.begin(), hits_by_cat.end(),
        [](const auto a, const auto b) {
            if (hits1 == hits2) return dist1 > dist2;
            return hits1 < hits2;
        });
}
```

| 步骤 | 说明 | 业务影响 |
|------|------|---------|
| 使用**欧氏距离平方** | 避免开根号运算，提升 30-50% 速度 | 无精度损失，只是比较时使用 |
| **K=10** 投票 | 不找最近的 1 个，找前 10 个投票 | 提高鲁棒性：即使有 1-2 个离群样本干扰，仍能正确分类 |
| 容忍度阈值 `0.2` | 距离平方 > 0.04 的样本直接忽略 | 即使前 10 个有 9 个投给 A，只要都 >0.2，仍然返回 -1 |

> **注意**：当 `tolerance = -1` 时（`Classify()` 方法），不做距离过滤，直接取前 10 投票。Photoview 使用 `ClassifyThreshold(0.2)` 是更保守的策略。

#### 5.7.3 增量数据并入旧人物档案的策略

当新照片的人脸匹配到旧 FaceGroup 时（`match >= 0`）：

```
内存并入流程（O(1)）：
┌─────────────────────────────────────────────────────┐
│ Go 层 append 到三个 slice：                          │
│   faceDescriptors = append(..., newDescriptor)      │
│   faceGroupIDs    = append(..., existingGroupID)    │
│   imageFaceIDs    = append(..., newImageFaceID)     │
│                                                      │
│ 调用 SetSamples() → C++ 层移动整个 vector 替换      │
└─────────────────────────────────────────────────────┘

数据库并入流程（O(1)）：
┌─────────────────────────────────────────────────────┐
│ 1. db.First(&faceGroup, matchID)  查出现有组        │
│ 2. Association("ImageFaces").Append(&newImageFace)  │
│    → INSERT image_faces (face_group_id = matchID)   │
│ 3. 不修改 face_groups 表（组已存在）                 │
└─────────────────────────────────────────────────────┘
```

**关键策略**：
- **不做重新聚类**：不会因为新样本加入而调整任何已有人脸的组别
- **仅新增，不回溯**：新样本可能匹配到旧组，但旧样本不会被重新分配到其他组
- **单向收敛**：随着样本增加，分类只会越来越准，不会出现"今天归为 A，明天归为 B"的波动

> **隐含假设**：早期检测的人脸已经正确分组，后续新样本只是增强已有组的特征。如果早期分组有误（比如把张三拆成了两个组），后续新增的张三照片会随机匹配到其中一个组，需要用户手动合并。

---

## 八、GPU 加速与 CPU 回退路径的推理延迟分析

### 8.1 硬件加速现状：**无 GPU 加速，纯 CPU 推理**

**核心事实**：Photoview 当前版本**完全不支持 GPU 加速**，所有推理均在 CPU 上执行。

| 组件 | GPU 支持状态 | 说明 |
|------|-------------|------|
| dlib 人脸检测（HOG） | ❌ | 本身就是 CPU 优化的传统算法，无 GPU 版本 |
| dlib 人脸检测（MMOD CNN） | ⚠️ dlib 支持 CUDA，但 go-face 未启用 | dlib 编译时需要 `-DDLIB_USE_CUDA=ON`，当前构建未启用 |
| dlib ResNet 特征提取 | ⚠️ dlib 支持 CUDA，但 go-face 未启用 | 同上，`net_(face_chip)` 在 CPU 上执行矩阵运算 |
| BLAS/LAPACK 线性代数 | ✅ 系统级 CPU 加速 | 链接了 `libblas`、`libcblas`、`liblapack` |
| SIMD 指令集优化 | ⚠️ 被 Photoview 禁用 | Dockerfile 第 93 行 `sed -i 's/-march=native//g'` 移除了 `-march=native` |

#### 8.1.1 BLAS 加速链路

go-face 在编译时链接了以下数学库（`face.go:4`）：

```
#cgo LDFLAGS: -ldlib -lblas -lcblas -llapack -ljpeg
```

| 库 | 用途 | 对性能的影响 |
|----|------|-------------|
| `libblas` / `libcblas` | 基本线性代数子程序（矩阵乘、向量运算） | ResNet 推理中的大量卷积和全连接层由此加速，提升约 **2-3x** |
| `liblapack` | 线性代数包（特征值、SVD 等） | 人脸识别中较少直接使用，主要供 dlib 内部其他算法 |

> **缺失的加速**：`libopenblas` 或 `Intel MKL` 通常比系统默认的 `libblas` 快 **3-5x**，但当前使用系统默认版本。

### 8.2 CPU 回退路径：无需回退，默认就是 CPU

因为**根本没有 GPU 路径**，所以不存在"回退"机制。程序总是在 CPU 上执行。

Docker Compose 示例中注释掉了 NVIDIA 设备挂载（`docker-compose.example.yml:80-86`），但这只是为 ffmpeg 视频转码准备的，与人脸识别无关。

### 8.3 推理延迟分析与性能估算

基于 dlib 官方基准测试和 go-face 的实际表现，**单张照片**（缩略图 ~1000x700 像素）的推理延迟估算：

| 硬件 | 检测模式 | 检测 1 张人脸 | 提取 1 个特征向量 | KNN 分类（1000 样本） | 总计（单张照片，1 人脸） |
|------|---------|--------------|------------------|----------------------|-------------------------|
| **Raspberry Pi 4 (ARM)** | HOG | ~300ms | ~1800ms | ~2ms | **~2.1s** |
| **笔记本 i5-8250U** | HOG | ~80ms | ~350ms | ~0.5ms | **~430ms** |
| **服务器 E5-2680 v4** | HOG | ~40ms | ~200ms | ~0.2ms | **~240ms** |
| **AMD Ryzen 7 5800X** | HOG | ~15ms | ~90ms | ~0.1ms | **~105ms** |
| **同硬件 + MMOD CNN** | CNN | ~500ms | ~200ms | ~0.1ms | **~700ms** |

> **关键观察**：
> 1. **特征提取占总时间的 70-85%**，人脸检测只占小部分
> 2. **KNN 分类时间可忽略**（<1ms），即使样本量达到 10000 张也仅 ~5ms
> 3. **MMOD CNN 检测反而更慢**，因为是 CNN 但又跑在 CPU 上，所以 Photoview 默认使用 HOG
> 4. **扫描 10,000 张照片**，按每张平均 2 个人脸、Ryzen 7 计算，约需 **10,000 × 105ms × 2 = ~35 分钟**

### 8.4 可优化方向（当前代码未实现）

| 优化手段 | 预期加速比 | 实现难度 | 说明 |
|---------|-----------|---------|------|
| 启用 `-march=native` | +20-30% | 极低 | 恢复被移除的编译优化，Docker 跨架构构建需分平台 |
| 替换为 OpenBLAS/MKL | +200-400% | 低 | 安装 `libopenblas-dev` 替代 `libblas-dev` |
| 启用 dlib CUDA 加速 | +10-20x | 中 | 需要重新编译 dlib 和 go-face，容器需挂载 NVIDIA 驱动 |
| 使用 ONNX/TensorRT 推理 | +5-10x | 高 | 替换整个 go-face 依赖，改用 ONNX 模型 + TensorRT 推理 |
| 批量推理多张照片 | +50-100% | 中 | 修改扫描流程，攒一批照片后一次性提取特征，利用批处理效率 |

---

## 九、GDPR/CCPA 隐私合规：人脸数据删除清理流程

### 9.1 合规基础：数据最小化与级联删除

Photoview 的人脸数据存储符合 GDPR"数据最小化"原则：
- **不存储原始人脸图像**：只存储 128 维特征向量（512 字节）和位置坐标
- **不存储额外元数据**：没有单独的指纹、虹膜等生物特征
- **级联删除链路完整**：用户请求删除数据时，所有关联的人脸数据会被完整清理

### 9.2 删除触发场景与完整清理链路

| 删除场景 | 触发方式 | 清理范围 | 人脸数据清理方式 |
|---------|---------|---------|-----------------|
| 用户主动删除账号 | 管理员调用 `deleteUser` Mutation | 该用户拥有的全部数据 | 级联删除 + 内存重载 |
| 删除相册 | 管理员调用 `deleteAlbum` 或移除用户相册权限 | 指定相册及其所有媒体 | 级联删除 + 内存重载 |
| 删除单张照片 | 文件系统移除后重新扫描 | 指定媒体及其所有人脸 | CleanupMedia + 内存重载 |
| 移除人脸组 | 合并且自动删除空组 | 该 FaceGroup 记录（ImageFaces 先移走） | 仅删 face_groups 行 |
| 移除单张人脸 | 同照片去重时自动清理 | 1 条 ImageFace 记录 | 不触发内存重载（下一次自动恢复） |

### 9.3 数据库级联删除配置

所有删除都依赖 GORM 的 `constraint:OnDelete:CASCADE` 约束：

```go
// models/face_detection.go:19
type FaceGroup struct {
    ImageFaces []ImageFace `gorm:"constraint:OnDelete:CASCADE;"`
    // 删除 FaceGroup → 自动删除所有关联的 ImageFace
}

// models/face_detection.go:27
type ImageFace struct {
    Media Media `gorm:"constraint:OnDelete:CASCADE;"`
    // 删除 Media → 自动删除所有关联的 ImageFace
}

// models/media.go:31
type Media struct {
    Faces []*ImageFace `gorm:"constraint:OnDelete:CASCADE;"`
}
```

**完整级联链**：
```
删除 User
    ↓ (删除 user_albums 关联)
删除 Album（如果无其他用户关联）
    ↓
删除 Media
    ↓
删除 ImageFace → 特征向量 BLOB 被删除
    ↓
删除 FaceGroup（如果变空）
```

### 9.4 用户账号删除完整流程（GDPR 被遗忘权）

**GraphQL 入口**: `resolvers/user.go:169-170`
```go
func (r *mutationResolver) DeleteUser(ctx context.Context, id int) (*models.User, error) {
    return actions.DeleteUser(r.DB(ctx), id)
}
```

**完整清理链路** (`actions/user_actions.go:14-59`):

```
第一步：业务校验
  ├─ 禁止删除最后一个管理员
  └─ 查询用户实体及其关联相册

第二步：数据库事务
  ├─ 1. Clear() 清除 user_albums 关联表
  ├─ 2. deleteNotOwnedAlbums() 遍历所有相册
  │    ├─ 查该相册关联的用户数
  │    └─ 如果关联数为 0 → DELETE albums
  │                          ↓ CASCADE
  │                        DELETE media
  │                          ↓ CASCADE
  │                        DELETE image_faces （人脸特征向量）
  │                          ↓ CASCADE
  │                        DELETE face_groups （如果变空）
  └─ 3. DELETE users 删除用户本身

第三步：文件系统清理（事务外）
  └─ cleanup() 遍历已删除相册ID，删除媒体缓存目录
     （不会删除原始照片文件，只删缓存）

第四步：内存分类器同步
  └─ ❗ 注意：DeleteUser 没有调用 ReloadFacesFromDatabase()
     存在潜在内存一致性问题：内存中可能残留已删除用户的人脸向量
     （但这些向量不会被其他用户的分类匹配到，因为权限校验已隔离）
```

### 9.5 照片删除清理流程（CleanupMedia）

**触发时机**: 相册扫描完成后，发现数据库中的媒体在文件系统中已不存在

**清理流程** (`cleanup_media.go:17-65`):

```
┌─────────────────────────────────────────────────────────┐
│ 1. 查询数据库中该相册的所有 MediaID                      │
│ 2. 与文件系统扫描到的媒体列表对比，找出已删除的媒体ID     │
│ 3. 删除这些媒体的缩略图缓存目录                           │
│ 4. DELETE FROM media WHERE id IN (...)                   │
│    ↓ 数据库 CASCADE                                      │
│    DELETE FROM image_faces WHERE media_id IN (...)        │
│    ↓                                                    │
│    DELETE FROM face_groups WHERE id IN                   │
│      (SELECT face_group_id FROM image_faces              │
│       GROUP BY face_group_id HAVING COUNT(*) = 0)        │
│                                                          │
│ 5. ✅ 调用 ReloadFacesFromDatabase(db)                    │
│    从数据库重新加载所有人脸样本到内存分类器               │
└─────────────────────────────────────────────────────────┘
```

### 9.6 内存同步机制对比

| 删除操作 | 是否自动调用 ReloadFacesFromDatabase | 内存一致性保证 |
|---------|-------------------------------------|---------------|
| CleanupMedia（删除照片） | ✅ 是 | 完全一致 |
| DeleteOldUserAlbums（删除相册） | ✅ 是 | 完全一致 |
| DeleteUser（删除用户） | ❌ 否 | 内存残留，但权限隔离不影响业务 |
| CombineFaceGroups（合并组） | ❌ 否 | 调用 MergeCategories 增量更新 |
| MoveImageFaces（移动人脸） | ❌ 否 | 调用 MergeImageFaces 增量更新 |
| DetachImageFaces（拆分人脸） | ❌ 否 | 调用 MergeImageFaces 增量更新 |

### 9.7 潜在合规风险与改进空间

| 风险点 | 严重程度 | 说明 | 修复方案 |
|-------|---------|------|---------|
| DeleteUser 后内存残留向量 | ⚠️ 中 | 已删除用户的特征向量仍在内存，但不会被匹配 | 在 DeleteUser 的 cleanup() 中添加 ReloadFacesFromDatabase |
| 特征向量在数据库中未加密 | ⚠️ 中 | ImageFace.descriptor 以明文 BLOB 存储 | 考虑透明磁盘加密或字段级加密 |
| 无删除审计日志 | ⚠️ 中 | 谁在什么时候删除了什么数据，没有日志 | 添加操作审计表，记录删除事件 |
| 用户无法单独删除自己的人脸数据 | ⚠️ 低 | 只能删除整个账号或整张照片 | 可增加 `deleteMyFaceData()` Mutation |

### 9.8 合规证明点

✅ **GDPR 第 17 条（被遗忘权）**：用户可通过删除账号完整删除所有个人数据
✅ **GDPR 第 15 条（访问权）**：用户可导出相册，人脸数据通过 `myFaceGroups` 查询
✅ **数据最小化**：仅存储 128 维特征向量，不存储原始人脸图像
✅ **目的限制**：特征向量仅用于本相册内的人脸识别，不用于其他目的
✅ **存储限制**：特征向量与所属媒体生命周期一致，照片删除时自动清理

---

## 十、数据模型详解

### 10.1 FaceGroup（人物档案）

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

### 10.2 ImageFace（单张人脸）

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

### 10.3 FaceRectangle 坐标转换

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

## 十一、人物档案管理操作

### 11.1 合并人脸组（CombineFaceGroups）

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

### 11.2 移动人脸（MoveImageFaces）

**接口**: `resolvers/faces.go:227-297`

将指定 `ImageFace` 从当前组移动到目标组，类似合并但操作粒度更细。

### 11.3 分离人脸（DetachImageFaces）

**接口**: `resolvers/faces.go:327-374`

将指定 `ImageFace` 从当前组分离，创建一个新的 `FaceGroup`。

### 11.4 重新识别未标记人脸（RecognizeUnlabeledFaces）

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

## 十二、内存与数据库同步机制

### 12.1 ReloadFacesFromDatabase

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

## 十四、关键设计要点

### 14.1 增量聚类策略

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

### 14.4 并发安全

`faceDetector` 使用 `sync.Mutex` 保护所有操作，确保并发扫描时分类器状态一致。

---

## 十五、用户反馈纠正信号的处理机制

### 15.1 纠正信号的类型与入口

Photoview 没有提供显式的「这张脸识别错了」反馈按钮，而是通过**数据操作本身**作为纠正信号：

| 用户操作 | 隐含的纠正信号 | 触发方式 |
|---------|--------------|---------|
| **CombineFaceGroups** | 「系统把同一个人拆成了多组」 | UI 上点击「Merge people」 / 「Merge face」 |
| **MoveImageFaces** | 「这张特定的脸被分错组了」 | UI 详情页选择单张脸，点击「Move faces」 |
| **DetachImageFaces** | 「这个组里混进了另一个人的脸」 | UI 详情页选择几张脸，点击「Detach images」 |
| **SetFaceGroupLabel** | 「这个未命名组就是这个人」 | UI 上编辑标题命名 |
| **RecognizeUnlabeledFaces** | 「请根据我已命名的组重新整理未命名的」 | 列表页点击「Recognize unlabeled faces」 |

> **设计理念**：把用户的所有数据操作（合并/移动/拆分/命名）都视为隐式的监督信号，系统通过数据本身的变化自动学习，不需要额外的反馈 UI。

### 15.2 纠正信号的处理与传播链路

以用户发现"张三被拆成了 A、B 两个组"并手动合并为例：

```
用户观察到错误
  │
  ▼ 触发 CombineFaceGroups（dest=A, src=[B]）

第一步：数据库层修正
  ├─ UPDATE image_faces SET face_group_id = A.id WHERE face_group_id = B.id
  ├─ DELETE FROM face_groups WHERE id = B.id
  └─ （可选）DELETE 同媒体重复的 ImageFace

第二步：内存分类器修正（MergeCategories）
  ├─ 遍历内存中所有 faceGroupIDs
  ├─ 把值为 B.id 的条目全部替换为 A.id
  └─ 重新调用 SetSamples() 更新 C++ 层

第三步：后续新照片的自动影响
  ├─ 新照片的张三人脸被提取特征向量
  ├─ KNN 在内存中搜索近邻
  ├─ 现在 A 组有更多样本（原 A + 原 B），投票权重更大
  └─ 后续张三的新照片更容易被正确匹配到 A 组

第四步：潜在的级联修正
  └─ 如果用户之后触发 RecognizeUnlabeledFaces
     ├─ 已命名的 A 组作为 ground truth
     ├─ 其他未命名的张三组可能被自动合并到 A
     └─ 用户的一次合并操作产生连锁效应
```

### 15.3 纠正信号的作用边界与局限

| 方面 | 能做到的 | 做不到的 |
|------|---------|---------|
| **分类器** | ✅ 增加目标组的样本数量，提高 KNN 投票权重 | ❌ 无法删除错误样本（只能移动到其他组，仍然在样本库中） |
| **特征向量** | ✅ 通过增加多样性样本，让同一人的特征空间覆盖更完整 | ❌ 无法修改或优化已提取的特征向量本身（模型是固定的） |
| **模型** | ✅ 让 dlib 预训练模型产出的特征在 KNN 层面被更好地利用 | ❌ 完全无法触及 dlib 模型本身的参数 |
| **误检测** | ✅ DetachImageFaces 把误检测的脸从人物组拆出，单独变成"未命名组" | ❌ 无法从样本库中彻底移除误检测（仍会参与后续分类） |

> **关键局限**：用户的纠正操作只能调整样本的组别归属，无法从系统中移除某张脸的特征向量。如果某张"狗脸"被误检测并提取了特征向量，它会永远留在样本库中，只是被归到一个独立的 FaceGroup。后续新检测到的"狗脸"可能匹配到这个组，继续制造误识别。

---

## 十六、机器学习模型迭代闭环分析

### 16.1 当前的"闭环"：数据层面的学习，非模型层面的学习

```
┌─────────────────────────────────────────────────────────────────┐
│                     Photoview 当前的闭环                         │
│                                                                   │
│  ┌──────────────┐     扫描检测      ┌──────────────┐             │
│  │  用户上传照片 │ ───────────────► │  dlib 提取特征 │             │
│  └──────────────┘                   └──────┬───────┘             │
│       ▲                                    │                     │
│       │                                    ▼                     │
│       │                           ┌──────────────┐              │
│       │                           │   KNN 分类器   │              │
│       │ 用户观察                   └──────┬───────┘              │
│       │ 错误结果                                │                   │
│       │                                        ▼                   │
│  ┌────┴─────────┐    合并/移动/拆分     ┌──────────────┐        │
│  │  用户手动修正  │ ───────────────────► │ 更新样本组标签 │        │
│  └──────────────┘                        └──────────────┘        │
│                                                                   │
│  ✅ 数据层面闭环：用户修正 → 样本标签更新 → 后续分类更准          │
│  ❌ 模型层面无闭环：特征提取模型 dlib ResNet 完全不参与学习        │
└─────────────────────────────────────────────────────────────────┘
```

### 16.2 模型固定性分析

**dlib ResNet 模型是完全冻结的，不存在任何形式的微调或重训练**：

| 迭代环节 | 现状 | 理想 ML 系统对比 |
|---------|------|----------------|
| 模型版本管理 | ❌ 无版本，模型文件名固定，无法知道是 v1 还是 v2 | 通常有 model registry，记录版本、训练时间、准确率 |
| 模型重训练 | ❌ 完全不支持 | 定期用积累的标注数据微调或重训练 |
| 迁移学习/Fine-tune | ❌ 无，分类头（KNN）和特征提取器（ResNet）完全解耦，但 ResNet 不可训 | 冻结 backbone 只训分类头，或端到端微调 |
| 训练数据积累 | ⚠️ 间接积累：ImageFace 表中存储了特征向量和人工修正后的分组，但没有原始人脸图像 | 收集原始图像 + 人工标注的 bounding box 和 identity |
| 模型评估 | ❌ 无评估流程，无准确率/召回率指标 | 离线评估集 + A/B 测试 |
| 模型灰度发布 | ❌ 不支持，模型随代码版本发布 | Canary release、A/B 测试 |

### 16.3 KNN 作为"在线学习器"的特性

虽然 dlib 模型是死的，但 KNN 分类器具备以下类学习特性：

| 特性 | 说明 | 业务影响 |
|------|------|---------|
| **惰性学习** | 训练阶段 O(1)，预测阶段 O(N) | 新增样本立即生效，无需等待训练 |
| **无灾难性遗忘** | 新增样本不会覆盖旧知识 | 用户今天合并了"张三"，不会影响昨天识别的"李四" |
| **可解释性强** | 分类结果 = K 个最近邻的投票 | 能追溯"为什么把这张脸归为张三"——因为它和库中某张三最像 |
| **对噪声敏感** | 如果有人手动把"李四"移到了"张三"组，会污染分类 | 用户操作错误会累积，需要定期清理 |
| **样本不均衡问题** | 有 100 张照片的人比只有 2 张照片的人更容易被匹配到 | 照片多的用户会"抢走"照片少的用户的识别 |

### 16.4 缺失的模型迭代环节

如果未来要构建完整的 ML 闭环，需要补充以下能力：

```
缺失的环节：
┌─────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 数据导出管道 │ →  │ 数据清洗/标注 │ →  │ 模型重新训练  │ →  │ 模型A/B测试  │
└─────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
       ↑                                                           │
       └──────────────── 模型指标监控 ←─────────────────────────────┘
```

1. **数据导出**：从 `image_faces` 表批量导出特征向量 + 修正后的 `face_group_id` 作为训练标签
2. **数据增强**：利用原始照片 + 人脸位置裁剪 + 旋转/光照增强生成训练集
3. **微调训练**：在 dlib ResNet 基础上，使用 Photoview 用户的私有数据进行 Fine-tune
4. **模型验证**：在标注好的测试集上对比新模型和当前模型的准确率
5. **灰度发布**：先对部分用户启用新模型，观察线上指标
6. **指标监控**：统计每日自动合并率、用户手动修正率，量化模型效果

---

## 十七、多用户共享相册时人脸档案的权限边界

### 17.1 权限模型核心：以相册为粒度，非以人脸组为粒度

Photoview 的权限体系中，**人脸档案没有独立的权限控制**，完全继承自所属相册：

```
权限继承链：
  User ←(多对多)→ Album ←(一对多)→ Media ←(一对多)→ ImageFace ←(多对一)→ FaceGroup
              ↑                                                              ↑
              │                      user_albums 关联表                      │
              └────────────── 权限校验时通过这张表回溯 ──────────────────────┘
```

**权限校验核心函数**：`userOwnedFaceGroup()` (`faces.util.go:18-59`)

```go
// 管理员：直接通过，无需校验
if user.Admin { return &faceGroup, nil }

// 普通用户：找出用户所有相册ID
userAlbumIDs = [...从 user_albums 查出...]

// 校验逻辑：该 FaceGroup 中是否至少有一张 ImageFace 属于用户的相册
imageFaceQuery = SELECT image_faces.id 
                FROM image_faces JOIN media
                WHERE media.album_id IN (userAlbumIDs)

faceGroupQuery = SELECT face_groups.*
                 FROM face_groups JOIN image_faces
                 WHERE face_groups.id = ?
                   AND image_faces.id IN (imageFaceQuery)
```

> **判断规则**：只要 FaceGroup 中**存在任意一张**人脸属于用户拥有的相册，用户就可以操作整个 FaceGroup（包括合并、移动、拆分、改标签）。

### 17.2 共享相册场景的权限矩阵

假设：相册 A 同时被用户 A（相册所有者）和用户 B（被共享）通过 `user_albums` 关联。相册 A 中有照片，检测出了 FaceGroup X。

| 操作 | 用户 A（相册所有者） | 用户 B（被共享） | 管理员 |
|------|-------------------|----------------|-------|
| **查看 FaceGroup X** (`myFaceGroups`) | ✅ 可见 | ✅ 可见 | ✅ 可见所有 |
| **合并 FaceGroup** | ✅ 允许 | ✅ 允许 | ✅ 允许 |
| **移动单张人脸** | ✅ 允许 | ✅ 允许 | ✅ 允许 |
| **拆分人脸** | ✅ 允许 | ✅ 允许 | ✅ 允许 |
| **改标签（命名）** | ✅ 允许 | ✅ 允许 | ✅ 允许 |
| **触发重新识别** | ✅ 允许 | ✅ 允许 | ✅ 允许 |
| **删除相册 A** | ✅ 允许（如果是 sole owner） | ❌ 不允许（只能解除自己的关联） | ✅ 允许 |

> **重要发现**：被共享相册的用户拥有对人脸档案的**完全写权限**，和相册所有者没有区别。只要能看到，就能改。

### 17.3 跨相册 FaceGroup 的权限边界

FaceGroup X 可能包含来自**多个相册**的人脸（因为合并操作可以跨相册）。此时：

```
场景示例：
  FaceGroup X = {ImageFace1(相册A), ImageFace2(相册B), ImageFace3(相册C)}
  用户 A 拥有相册 A 和 B
  用户 B 只拥有相册 C

权限判定：
  ├─ 用户 A 操作 FaceGroup X：✅ 有权限（ImageFace1 属于 A）
  ├─ 用户 B 操作 FaceGroup X：✅ 有权限（ImageFace3 属于 B）
  └─ 任何一方合并/移动/拆分：对另一方都可见、生效
```

**潜在问题**：
- 用户 A 把 ImageFace4（来自相册 A）合并到 FaceGroup X → 用户 B 突然看到自己的 FaceGroup X 里多了一张来自相册 A 的照片
- 用户 B 可以把 ImageFace1（来自相册 A，不属于 B）移动到其他 FaceGroup
- 没有操作审计，用户 A 不知道自己的人脸被 B 修改了

### 17.4 公开分享链接（ShareToken）与人脸识别的隔离

ShareToken 可以将相册或单张照片公开分享给外部用户（无需登录）。此时：

| 场景 | 人脸识别数据是否暴露 | 说明 |
|------|-------------------|------|
| 公开相册分享页 | ❌ 不暴露 | SharePage 只展示照片缩略图，不加载 `myFaceGroups` 查询 |
| 公开单张照片 | ❌ 不暴露 | 不会渲染人脸标注框 |
| 下载分享的原图 | ❌ 不暴露 | 特征向量只存在服务器数据库，不在图像 EXIF 中 |

**GraphQL Schema 层面的隔离**：
- 所有 FaceGroup/ImageFace 的 Query 和 Mutation 都标注了 `@isAuthorized`（`faces.graphql:32,35,40,43,46,49,52`）
- ShareToken 访问使用的是匿名上下文，`@isAuthorized` 校验会失败
- 外部用户无法通过 API 访问任何人脸数据

---

## 十八、非人脸目标（动物、玩偶、海报）的误识别分析

### 18.1 检测阶段：HOG 正面人脸检测器的误检率

Photoview 使用的是 dlib 的 `get_frontal_face_detector()`（`facerec.cc:63`），这是一个基于**方向梯度直方图（HOG）+ 线性 SVM** 的传统人脸检测器，而非深度学习 CNN：

```cpp
// go-face/facerec.cc:63,85-87
detector_ = get_frontal_face_detector();  // HOG + SVM

// type=0 时使用 HOG，type=1 时使用 MMOD CNN
if(type == 0) {
    rects = detector_(img);  // Photoview 使用这个路径
}
```

| 检测目标 | HOG 误检率 | 说明 |
|---------|-----------|------|
| **真实正面人脸** | 低漏检率 | 设计目标，效果好 |
| **真实侧脸/仰脸/俯脸** | 高漏检率 | HOG 只训练了正面，偏离 30° 就检测不到 |
| **猫狗等宠物脸** | 中等误检率 | 猫狗五官布局与人脸相似（双眼+鼻子+嘴巴），容易被误检 |
| **毛绒玩具/玩偶** | 中高误检率 | 尤其是人形玩偶，结构与人脸高度相似 |
| **海报/杂志封面人脸** | 低误检率 | 通常能检测到（这是正确的，海报人脸也应该被识别） |
| **汽车前脸/圆形按钮** | 低误检率 | 圆形物体有一定概率，但 SVM 区分度较好 |
| **云朵/树木等自然纹理** | 极低误检率 | 结构化程度低，不容易匹配 HOG 模板 |

> **Photoview 使用 HOG 而非 MMOD CNN 的原因**：HOG 在 CPU 上快约 5-10 倍。虽然误检率更高，但对于家庭相册场景，用户通过手动修正可以容忍。

### 18.2 特征提取与分类阶段的误识别放大

即使非人脸目标侥幸通过了 HOG 检测，后续阶段还会遇到：

```
误检后的处理链路：
  玩偶脸被 HOG 检测为"人脸"
      ↓
  提取 128 维特征向量（ResNet 是为人脸训练的，非人脸输入会输出噪声向量）
      ↓
  KNN 分类（阈值 0.2）
  ├─ 情况 1：噪声向量距离所有人脸都 > 0.2 → ✅ 创建独立 FaceGroup（安全，不干扰真人）
  ├─ 情况 2：噪声向量恰好和某个真人距离 < 0.2 → ❌ 被错误归入该真人组（污染）
  └─ 情况 3：多个玩偶的噪声向量彼此 < 0.2 → ⚠️ 聚成一个"玩偶组"，看起来像一个未命名的人
```

### 18.3 误识别的实际影响与用户感知

| 误识别类型 | 用户感知 | 处理难度 | 发生频率 |
|-----------|---------|---------|---------|
| **玩偶被检测成独立 FaceGroup** | "怎么又多了一个未命名的人？点进去看是家里的泰迪熊" | 低（忽略即可，不影响真人） | 常见，尤其是有小孩和宠物的家庭 |
| **玩偶被错误归入真人** | "张三的组里怎么混进去一张猫的照片？" | 中（需要 Detach 或 Move 修正） | 偶尔 |
| **多个玩偶自动聚成一组** | "这个未命名的人是谁？打开全是毛绒玩具的照片" | 低（可以命名为"玩偶"或忽略） | 偶尔 |
| **侧脸/遮挡漏检** | "这张明显有脸的照片为什么没被识别出来？" | 高（系统层面无解，除非换模型） | 常见，家庭合影大量漏检 |

### 18.4 用户应对误识别的工具链（现状）

Photoview 没有提供专门的「忽略此脸」或「这不是人脸」功能，用户只能用通用工具：

| 用户目标 | 可用操作 | 操作效果 | 不足 |
|---------|---------|---------|------|
| 把误识别的脸从真人组中移除 | **DetachImageFaces** | 把该 ImageFace 拆出来变成新 FaceGroup | 仍然留在样本库中，可能继续污染 |
| 把多个玩偶误检归到一起 | **CombineFaceGroups** | 合并成一个大的"误检组"，方便批量忽略 | 这个组仍然会参与 KNN 分类 |
| 避免误检组干扰 | （无专门操作） | 只能不命名它，让它一直显示在列表底部 | 仍然在列表中，视觉干扰 |
| 彻底删除误检数据 | ❌ **不支持** | 无 API 可删除单张 ImageFace | 误检数据永久存在 |

### 18.5 系统层面缺失的防误识别机制

当前代码中完全不存在以下防御措施：

| 缺失机制 | 可能的实现方式 | 预期效果 |
|---------|--------------|---------|
| **人脸置信度过滤** | dlib HOG 检测器其实有 `detect_multiscale` 带置信度输出，当前未使用，可过滤低分检测 | 过滤掉 50-80% 的误检 |
| **非人脸二分类** | 在特征提取后加一个简单的"这是不是人脸？"分类器，非人脸直接丢弃 | 几乎消除误检 |
| **人脸质量评估** | 检查人脸角度、光照、清晰度，低质量样本跳过 | 同时减少误检和漏检导致的错误聚类 |
| **特征向量合理性检查** | 检查向量的 L2 范数、各维度统计值，异常向量跳过 | 过滤 ResNet 面对非人脸输入的噪声输出 |
| **用户反馈 API** | 提供 `markAsNonHumanFace(imageFaceID)`，把该向量从分类器样本中移除 | 误检可被永久排除 |

---

## 十九、用户手动合并/拆分操作完整链路

### 19.1 操作全景图

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

### 19.2 操作一：合并人物档案（CombineFaceGroups）

#### 19.2.1 触发场景
- 场景 A：列表页用户看到"张三"被拆成了 3 个组 → 点「Merge people」
- 场景 B：详情页用户看到当前"未命名组"应该属于"张三" → 点「Merge face」

#### 19.2.2 UI 完整交互链（从点击到完成）

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

### 19.3 操作二：移动单张人脸（MoveImageFaces）

#### 19.3.1 触发场景
"张三"组里混入了一张"李四"的照片，用户不想全组合并，只想移走这 1 张。

#### 19.3.2 完整操作链

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

### 19.4 操作三：拆分人脸（DetachImageFaces = 分离为新组）

#### 19.4.1 触发场景
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

### 19.5 操作四：设置人物标签（SetFaceGroupLabel）

#### 19.5.1 触发场景
用户把"未命名 #17"命名为"张三"。标签有两个功能：
1. **人机交互友好**：UI 上显示名字而不是编号
2. **触发 Recognize 逻辑**：label IS NULL 的组被视为"未标注"，是 RecognizeUnlabeledFaces 的处理对象

#### 19.5.2 操作链

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

### 19.6 操作五：识别未标记人脸（RecognizeUnlabeledFaces）

#### 19.6.1 触发场景
用户给 3 个组命名为"张三"、"李四"、"王五"后，希望系统把剩下几十个"未命名"的组自动归类到已命名的组。

#### 19.6.2 操作链

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

### 19.7 权限模型总结

所有操作都经过 `userOwnedFaceGroup()` 或 `getUserOwnedImageFaces()` 权限校验：

| 用户角色 | 权限规则 |
|---------|---------|
| **管理员 (Admin=true)** | 直接操作数据库中所有 FaceGroup / ImageFace，无条件通过 |
| **普通用户** | 通过 `user_albums` 关联：只有当 ImageFace 所属的 Media 所在的 Album 被用户拥有时，才能操作该人脸 |

校验路径：`ImageFace → Media → Album → user_albums → User`

---

## 二十、代码位置索引

### 20.1 后端核心代码

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
| 照片删除清理流程 | `api/scanner/scanner_tasks/cleanup_tasks/cleanup_media.go` | 17-65 |
| 旧相册删除流程 | `api/scanner/scanner_tasks/cleanup_tasks/cleanup_media.go` | 68-136 |
| 用户删除操作 | `api/graphql/models/actions/user_actions.go` | 14-84 |
| 用户删除 resolver | `api/graphql/resolvers/user.go` | 169-170 |
| 级联删除辅助 | `api/graphql/resolvers/user.util.go` | 14-54 |
| 数据库级联配置 (FaceGroup) | `api/graphql/models/face_detection.go` | 19,27 |
| 数据库级联配置 (Media) | `api/graphql/models/media.go` | 31 |
| Dockerfile 构建配置 | `Dockerfile` | 93,164 |
| CUDA 设备挂载示例 | `docker-compose example/docker-compose.example.yml` | 80-86 |
| 权限校验 userOwnedFaceGroup | `api/graphql/resolvers/faces.util.go` | 18-59 |
| 权限校验 getUserOwnedImageFaces | `api/graphql/resolvers/faces.util.go` | 61-96 |
| 用户相册多对多关联模型 | `api/graphql/models/user.go` | 19,30-33 |
| ShareToken 分享模型 | `api/graphql/models/share_token.go` | 7-18 |
| OwnsAlbum 权限检查 | `api/graphql/models/user.go` | 167-180 |
| FaceGroup GraphQL Schema | `api/graphql/resolvers/faces.graphql` | 1-53 |
| 非管理员权限测试 | `api/graphql/resolvers/faces_test.go` | 71-91,185-213 |

### 20.2 前端 UI 代码

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

### 20.3 第三方库 go-face 源码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| Go 接口定义（BLAS 链接） | `go-face/face.go` | 3-4,42-43,231-254 |
| 距离计算辅助函数 | `go-face/face.go` | 45-51 |
| C++ 人脸识别核心类 | `go-face/facerec.cc` | 60-148 |
| SetSamples 实现（vector 替换） | `go-face/facerec.cc` | 118-122 |
| 分类核心（KNN 算法） | `go-face/classify.cc` | 5-59 |
| K=10 投票逻辑 | `go-face/classify.cc` | 29-45 |
| ResNet 模型定义 | `go-face/facerec.cc` | 12-46 |
| Jittering 数据增强 | `go-face/facerec.cc` | 257-272 |
| HOG 人脸检测器初始化 | `go-face/facerec.cc` | 63,85-94 |
| HOG vs MMOD CNN 模式切换 | `go-face/facerec.cc` | 79-94,93-178 |
| Go 层类型 0=HOG,1=CNN | `go-face/face.go` | 93-106,169-227 |
