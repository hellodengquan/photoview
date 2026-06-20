# Photoview 人脸识别聚类与合并冲突处理流程

## 1. 整体架构概览

Photoview 的人脸识别系统由三层协同构成：

| 层级 | 组件 | 职责 |
|------|------|------|
| **检测层** | `FaceDetectionTask` → `faceDetector.DetectFaces` | 扫描媒体缩略图，提取人脸与 128 维 embedding |
| **聚类层** | `faceDetector.classifyFace` + `go-face.Recognizer` | 基于距离阈值将 embedding 归入已有 FaceGroup 或创建新组 |
| **交互层** | GraphQL mutations (`CombineFaceGroups`, `MoveImageFaces`, `DetachImageFaces`, `RecognizeUnlabeledFaces`) | 用户手动合并、移动、拆分、重识别 |

关键数据结构在内存与数据库之间维持双份状态，这是理解整个协同机制的核心：

```
┌─────────────────────────────────────────────────────┐
│            faceDetector (内存)                       │
│  faceDescriptors []face.Descriptor  (128维向量)      │
│  faceGroupIDs    []int32             (组映射)        │
│  imageFaceIDs    []int               (人脸ID)        │
│  rec             *face.Recognizer    (分类器)        │
└─────────────────────────────────────────────────────┘
         ↕ 同步
┌─────────────────────────────────────────────────────┐
│            数据库                                    │
│  face_groups  (id, label)                           │
│  image_faces  (id, face_group_id, media_id,         │
│               descriptor, rectangle)                │
└─────────────────────────────────────────────────────┘
```

---

## 2. 检测与自动聚类流程

### 2.1 触发入口

```
扫描任务 → FaceDetectionTask.AfterProcessMedia()
         → faceDetector.DetectFaces(db, media)
```

`FaceDetectionTask` (`api/scanner/scanner_tasks/face_detection_task.go:15`) 在媒体处理完成后触发，仅对照片类型 (`MediaTypePhoto`) 执行。

### 2.2 人脸检测

`DetectFaces` (`api/scanner/face_detection/face_detector_impl.go:92`) 的流程：

1. 加载媒体的缩略图路径 (`PhotoThumbnail`)
2. 调用 `rec.RecognizeFile(thumbnailPath)` 检测人脸（基于 dlib 的 CNN 人脸检测器 + 5 点 landmark）
3. 对每个检测到的人脸调用 `classifyFace`

### 2.3 Embedding 距离阈值聚类

这是**自动聚类**的核心，位于 `classifyFace` (`face_detector_impl.go:134`)：

```go
func (fd *faceDetector) classifyDescriptor(descriptor face.Descriptor) int32 {
    return int32(fd.rec.ClassifyThreshold(descriptor, 0.2))
}
```

**阈值 `0.2`** 是关键参数。`ClassifyThreshold` 的语义：

- 计算输入 descriptor 与所有已有样本的欧氏距离
- 如果**最小距离 < 0.2**，返回匹配样本对应的 `faceGroupID`（≥0）
- 如果**所有距离 ≥ 0.2**，返回 `-1`（无匹配）

分类决策分支 (`face_detector_impl.go:160-181`)：

```
classifyDescriptor(descriptor)
       │
       ├── match < 0  ──→ "No match, assigning new face"
       │                  创建新 FaceGroup，包含该 ImageFace
       │                  DB: Create(&faceGroup)
       │
       └── match ≥ 0  ──→ "Found match"
                          查找已有 FaceGroup
                          DB: Association("ImageFaces").Append(&imageFace)
```

**无论哪种分支，都会执行以下内存更新** (`face_detector_impl.go:183-187`)：

```go
fd.faceDescriptors = append(fd.faceDescriptors, face.Descriptor)
fd.faceGroupIDs = append(fd.faceGroupIDs, int32(faceGroup.ID))
fd.imageFaceIDs = append(fd.imageFaceIDs, imageFace.ID)
fd.rec.SetSamples(fd.faceDescriptors, fd.faceGroupIDs)
```

这意味着：**每分类一个人脸，内存中的样本集立即膨胀，后续分类会参考更大的样本集。** 这是一个增量聚类过程。

### 2.4 阈值 0.2 的含义

dlib 的 `face_recognition_resnet_model_v1` 输出的 128 维 embedding 遵循：**同一人的距离通常 < 0.6，不同人的距离通常 > 0.6**。Photoview 选用 0.2 是一个**非常严格**的阈值，意味着：

- 自动聚类非常保守，宁可拆成多个组，也不轻易合并
- 同一人如果表情、角度差异较大，可能被分为多个 FaceGroup
- 这为后续用户手动合并留下了大量空间——是"宽松拆分 + 人工合并"的设计策略

### 2.5 阈值的可配置性：硬编码，运维侧不可调

`0.2` 这个阈值是**硬编码的字面量**，定义在 `face_detector_impl.go:131`：

```go
func (fd *faceDetector) classifyDescriptor(descriptor face.Descriptor) int32 {
    return int32(fd.rec.ClassifyThreshold(descriptor, 0.2))
}
```

**运维侧无法通过配置或环境变量调整**。目前与人脸识别相关的环境变量仅有两个（定义在 `api/utils/environment_variables.go:21,43`）：

| 环境变量 | 作用 |
|----------|------|
| `PHOTOVIEW_FACE_RECOGNITION_MODELS_PATH` | 模型文件路径 |
| `PHOTOVIEW_DISABLE_FACE_RECOGNITION` | 总开关（布尔值） |

**没有任何配置项可以调整距离阈值**。如果需要修改，必须改代码重新编译。这也反映了设计上的一个取舍：把阈值固化在代码中，避免用户随意调整导致聚类结果不稳定。

---

## 3. 数据模型

### 3.1 FaceGroup

```go
// api/graphql/models/face_detection.go:16
type FaceGroup struct {
    Model                          // ID, CreatedAt, UpdatedAt
    Label      *string             // 可空，人物名字
    ImageFaces []ImageFace         // 1:N 关系，CASCADE 删除
}
```

### 3.2 ImageFace

```go
// api/graphql/models/face_detection.go:22
type ImageFace struct {
    Model
    FaceGroupID int                // 所属 FaceGroup
    FaceGroup   *FaceGroup
    MediaID     int                // 关联的媒体
    Media       Media              // CASCADE 删除
    Descriptor  FaceDescriptor     // [128]float32，存为 BLOB/BYTEA
    Rectangle   FaceRectangle      // "MinX:MaxX:MinY:MaxY" 存为 VARCHAR(64)
}
```

**关键约束**：`FaceGroup` ↔ `ImageFace` 是 1:N，而 `ImageFace` 通过 `MediaID` 关联到具体的图片。**同一张图片中可能出现多个人脸**（多个 ImageFace 共享同一个 MediaID）。

---

## 4. 用户手动操作：四个 Mutation

### 4.1 操作总览

| Mutation | 行为 | 粒度 | 冲突检查 | 内存同步方法 |
|----------|------|------|----------|-------------|
| `CombineFaceGroups` | 整组合并 | FaceGroup → FaceGroup | 有重复媒体拒绝 | `MergeImageFaces(sourceGroupIDs, destID)` |
| `MoveImageFaces` | 部分移动 | ImageFace → FaceGroup | 有重复媒体拒绝 | `MergeImageFaces(imageFaceIDs, destID)` |
| `DetachImageFaces` | 拆分到新组 | ImageFace → 新 FaceGroup | 无（新组不可能冲突） | `MergeImageFaces(imageFaceIDs, newID)` |
| `RecognizeUnlabeledFaces` | 自动重识别 | 未标记 → 已标记组 | 内部处理 | 重新构建样本集 |

### 4.2 CombineFaceGroups —— 整组合并

**入口**：`api/graphql/resolvers/faces.go:144`

```
用户请求 combineFaceGroups(destID, sourceIDs)
       │
       ├── 校验 1：sourceIDs 非空
       ├── 校验 2：用户拥有 dest 组
       ├── 校验 3：用户拥有每个 source 组
       ├── 校验 4：source 不能包含 dest
       │
       └── DB 事务：
            ├── 校验 5：hasDuplicateMediaInFaceGroupsUnion(所有组ID)
            │          → 合并后同一 media 不得有多个人脸
            ├── UPDATE image_faces SET face_group_id = destID WHERE face_group_id IN (sourceIDs)
            ├── DELETE source face_groups
            ├── 去重：对 dest 中同一 media_id 的多个 image_face，仅保留 MIN(id)
            │
            └── 内存同步：MergeImageFaces(sourceGroupIDs, destID)
```

**`hasDuplicateMediaInFaceGroupsUnion` 的 SQL 逻辑** (`faces.util.go:98`)：

```sql
SELECT media_id FROM image_faces
WHERE face_group_id IN (destID, sourceID1, sourceID2, ...)
GROUP BY media_id
HAVING COUNT(*) > 1
```

如果结果非空，说明**合并后同一张图片会被两个同一组的人脸占据**——这是不允许的，因为语义上"同一个人的脸不应该在同一张照片里出现两次"。

**去重步骤** (`faces.go:203-211`)：

```sql
DELETE FROM image_faces
WHERE face_group_id = destID
  AND id NOT IN (
    SELECT MIN(id) FROM image_faces
    WHERE face_group_id = destID
    GROUP BY media_id
  )
```

这处理的是**合并前** dest 和 source 各自已有同一 media 的人脸的情况——合并后只保留每个 media 最早创建的人脸记录。

**注意**：这个去重逻辑在冲突检查**之后**，看似矛盾。实际上，冲突检查拒绝的是"同一 media 在不同组中都有人脸"的情况（说明可能是不同人），而去重是针对合并后同 media 的清理。但当前实现中，冲突检查先拒绝，去重代码不会在正常路径上被触发——这是一个防御性冗余。

### 4.3 MoveImageFaces —— 部分人脸移动

**入口**：`api/graphql/resolvers/faces.go:228`

```
用户请求 moveImageFaces(imageFaceIDs, destFaceGroupID)
       │
       └── DB 事务：
            ├── 校验：用户拥有 dest 组
            ├── 校验：用户拥有要移动的 imageFaces
            ├── 校验：doesFaceGroupContainImages(destID, imageFaceIDs)
            │          → 移动后 dest 组中同一 media 不得有重复人脸
            ├── UPDATE image_faces SET face_group_id = destID WHERE id IN (imageFaceIDs)
            ├── 查找这些 imageFace 原属的 faceGroups
            ├── deleteEmptyFaceGroups：如果原组变空则删除
            │
            └── 内存同步：MergeImageFaces(userOwnedImageFaceIDs, destID)
```

**`doesFaceGroupContainImages` 的逻辑** (`faces.util.go:117`)：构造一个联合查询，把 dest 组中**已有的**人脸和**将要移入的**人脸合并，检查是否有 media_id 重复。

### 4.4 DetachImageFaces —— 拆分到新组

**入口**：`api/graphql/resolvers/faces.go:328`

```
用户请求 detachImageFaces(imageFaceIDs)
       │
       └── DB 事务：
            ├── 校验：用户拥有这些 imageFaces
            ├── 创建新 FaceGroup（空壳，无 label）
            ├── UPDATE image_faces SET face_group_id = newID WHERE id IN (imageFaceIDs)
            │
            └── 内存同步：MergeImageFaces(userOwnedImageFaceIDs, newID)
```

无需冲突检查，因为目标是一个全新的空组。

### 4.5 RecognizeUnlabeledFaces —— 自动重识别

**入口**：`faces.go:300` → `face_detector_impl.go:218`

这是**模型与手动操作协同**的关键环节：

```
用户请求 recognizeUnlabeledFaces()
       │
       └── faceDetector.RecognizeUnlabeledFaces(tx, user)
            │
            ├── 从 DB 查询所有 label IS NULL 的 FaceGroup
            │
            ├── 遍历内存样本，分离为：
            │   ├── unlabeled：属于无 label 组的 (descriptor, faceGroupID, imageFaceID)
            │   └── labeled：属于有 label 组的
            │
            ├── 用 labeled 样本重建内存样本集（不含 unlabeled）
            │   fd.faceGroupIDs = newFaceGroupIDs
            │   fd.faceDescriptors = newDescriptors
            │
            ├── 对每个 unlabeled descriptor：
            │   ├── classifyDescriptor(descriptor)  // 阈值仍是 0.2
            │   │
            │   ├── match < 0 → 仍无匹配
            │   │   └── 重新加入内存样本集（保持原 faceGroupID）
            │   │
            │   └── match ≥ 0 → 找到匹配
            │       ├── UPDATE DB: image_face.face_group_id = match
            │       └── 加入内存样本集（faceGroupID = match）
            │
            └── 返回被更新的 ImageFace 列表
```

**核心洞察**：重识别时，只有**已标记的** FaceGroup 作为"锚点"参与分类。用户通过给 FaceGroup 设置 label，实质上是在告诉系统"这组是确定的"，然后系统才会用这些确定组去匹配未标记的人脸。

**这是 embedding 距离阈值与用户手动操作的状态机协同点**：

```
           ┌──────────┐
           │  未标记组  │ ← 自动聚类产生，可能拆得过细
           └────┬─────┘
                │ 用户给某个组打 label
                ▼
           ┌──────────┐
           │  已标记组  │ ← 成为"锚点"
           └────┬─────┘
                │ 用户点击"识别未标记人脸"
                ▼
           ┌──────────────┐
           │  重识别未标记  │ ← 用已标记组的 embedding 重新匹配
           └────┬─────────┘
                │ 匹配成功
                ▼
           ┌──────────┐
           │  合入已标记组 │ ← 等效于自动 MoveImageFaces
           └──────────┘
```

---

## 5. 内存-数据库同步机制

### 5.1 MergeImageFaces 的实现

所有用户操作最终都调用 `MergeImageFaces` (`face_detector_impl.go:202`)：

```go
func (fd *faceDetector) MergeImageFaces(imageFaceIDs []int, destFaceGroupID int32) {
    fd.mutex.Lock()
    defer fd.mutex.Unlock()

    for i := range fd.faceGroupIDs {
        imageFaceID := fd.imageFaceIDs[i]
        for _, id := range imageFaceIDs {
            if imageFaceID == id {
                fd.faceGroupIDs[i] = destFaceGroupID
                break
            }
        }
    }
}
```

**注意**：这里只修改了 `faceGroupIDs` 映射，**没有修改** `faceDescriptors` 和 `imageFaceIDs`。这是正确的，因为：

- 人脸的 embedding 不会因为换组而改变
- `rec.SetSamples()` 不需要被调用，因为 `faceGroupIDs` 的值变了，下次 `SetSamples` 时会自动生效
- 但在当前实现中，**`SetSamples` 只在 `classifyFace` 中被调用**，所以如果合并后没有新的人脸被检测，分类器的样本集不会更新——这意味着合并操作后，如果紧接着有新的人脸被检测，分类器可能使用过时的样本。不过在 `RecognizeUnlabeledFaces` 中会重建样本集，所以不会出问题。

### 5.2 MergeCategories（未使用）

`MergeCategories` (`face_detector_impl.go:191`) 按 sourceGroupID 整批替换：

```go
func (fd *faceDetector) MergeCategories(sourceID int32, destID int32) {
    for i := range fd.faceGroupIDs {
        if fd.faceGroupIDs[i] == sourceID {
            fd.faceGroupIDs[i] = destID
        }
    }
}
```

当前代码中 `MergeCategories` **未被任何 resolver 调用**。`CombineFaceGroups` 使用的是 `MergeImageFaces`（传入 sourceGroupIDs），效果等价但更精确。

### 5.3 初始化与重载

- **启动时**：`InitializeFaceDetector` (`face_detector_impl.go:25`) 从 DB 加载所有 ImageFace，构建内存样本集
- **重载**：`ReloadFacesFromDatabase` (`face_detector_impl.go:75`) 完全替换内存样本集，但当前代码中**无调用点**

---

## 6. 冲突处理：重复媒体检测

### 6.1 为什么同一 Media 不能在 FaceGroup 中出现两次

语义约束：**一个 FaceGroup 代表一个人，同一个人不应该在同一张照片里被识别为两个不同的人脸**。如果出现这种情况，说明聚类有误或合并有误。

### 6.2 两个冲突检测函数

**`hasDuplicateMediaInFaceGroupsUnion`** (`faces.util.go:98`)：

```sql
SELECT media_id FROM image_faces
WHERE face_group_id IN (allIDs...)
GROUP BY media_id HAVING COUNT(*) > 1
```

用于 `CombineFaceGroups`：检查**合并后**的集合中是否有重复 media。

**`doesFaceGroupContainImages`** (`faces.util.go:117`)：

```sql
SELECT candidate.media_id FROM image_faces AS candidate
WHERE (candidate.face_group_id = destID AND candidate.id NOT IN (movingIDs))
   OR candidate.id IN (movingIDs)
GROUP BY candidate.media_id HAVING COUNT(*) > 1
```

用于 `MoveImageFaces`：把 dest 组现有的人脸（排除正在移动的）和正在移动的人脸合在一起检查。

### 6.3 冲突时的行为

- **合并/移动被拒绝**：返回错误 "cannot merge/move face groups because the destination would contain duplicate images"
- **数据库事务回滚**：所有变更不生效
- **内存不变**：`MergeImageFaces` 在事务成功后才被调用

---

## 7. 封面（Cover Image）的隐式处理与合并冲突

### 7.1 FaceGroup 没有显式的 cover_image 字段

与 Album 有 `cover_media_id` 不同，`FaceGroup` 数据模型中**没有** `cover_image` 或类似字段。封面是**隐式**的——UI 层取 `imageFaces[0]`（第一个 ImageFace）作为封面展示。

```tsx
// ui/src/Pages/PeoplePage/PeoplePage.tsx:220
export const FaceGroup = ({ group }: FaceGroupProps) => {
    const previewFace = group.imageFaces[0]  // 第一个就是封面
    // ...
    <FaceCircleImage imageFace={previewFace} selectable />
}
```

`FaceCircleImage` 组件会根据 `rectangle` 裁切掉图片中人脸以外的部分，呈现圆形头像效果。

### 7.2 ImageFaces 的排序规则

`ImageFaces` resolver (`faces.go:20-53`) 中**没有显式的 ORDER BY**：

```go
query := db.
    Joins("Media").
    Where(faceGroupIDIsQuestion, obj.ID).
    Where("album_id IN (?)", userAlbumIDs)

query = models.FormatSQL(query, nil, paginate)  // order 参数为 nil
```

`FormatSQL` (`models/utils.go:11`) 在 `order` 为 nil 时不加排序。因此，返回顺序由数据库决定，在大多数数据库中等价于**按主键 id 升序**——即**先创建的 ImageFace 排在前面**。

这意味着：**FaceGroup 的"封面"就是该组中最早创建的那张人脸图片。**

### 7.3 合并后的封面归属

合并 FaceGroup 时，没有专门的"封面冲突处理"逻辑。封面的归属由排序规则自然决定：

1. 合并后，dest 组中的 ImageFace 仍按 id 升序排列
2. **id 最小的那张**（通常是 dest 组中最早创建的）排在最前面，成为新封面
3. source 组的 ImageFace id 通常更大（后创建），所以通常不会"抢占"封面

**结论**：合并后 dest 组的原始封面通常保持不变。只有当 source 组中存在 id 更小的 ImageFace 时（极端情况，比如 dest 是新建的空组），封面才会变化。这不是一个被显式设计或测试过的行为，而是排序规则的副作用。

### 7.4 去重步骤与封面的关系

`CombineFaceGroups` 的去重步骤保留 `MIN(id)`（`faces.go:204`），这与封面排序逻辑一致——保留最早创建的人脸记录，也保留了它作为封面的可能性。

---

## 8. 关于 face_state 与 unmerge 回退路径

### 8.1 没有 face_state 状态字段

`FaceGroup` 模型中**不存在** `face_state`、`status`、`phase` 或任何类似的状态字段。整个系统没有"合并中"、"已合并"、"已拆分"等状态标记。

```go
// api/graphql/models/face_detection.go:16
type FaceGroup struct {
    Model                          // ID, CreatedAt, UpdatedAt
    Label      *string             // 只有 label 是可空的标记字段
    ImageFaces []ImageFace
}
```

`Label` 是唯一的"状态式"字段——它标记了这组是否被用户确认过（有名字 = 已确认，null = 未确认），但它不是状态机意义上的状态。

### 8.2 合并是单向的，没有 unmerge 回退路径

`CombineFaceGroups` 合并后，source FaceGroup 会被**物理删除**（`faces.go:200`）：

```go
// delete the source face groups
if err := deleteFaceGroups(sourceFaceGroups, tx); err != nil {
    return err
}
```

**没有任何撤销/回退机制**：

- 没有合并历史记录表
- 没有软删除标记
- 没有 unmerge mutation
- 没有"合并前快照"

### 8.3 近似的"逆向操作"：DetachImageFaces

`DetachImageFaces`（拆分到新组）是最接近"撤销合并"的操作，但它**不能精确还原**：

| 维度 | 真正的 unmerge | DetachImageFaces |
|------|---------------|------------------|
| 恢复原组 ID | 能 | 不能（新组 ID 不同） |
| 恢复 label | 能 | 不能（新组无 label） |
| 精确拆分边界 | 能（按原组边界） | 不能（需手动选择哪些人脸） |
| 批量还原 | 能 | 需手动操作 |

**设计意图**：合并被视为一个**不可逆的决策**。用户确认合并后，原组就消失了。如果合并错了，用户只能手动把人脸拆出来（Detach），但无法恢复原组的身份。

### 8.4 为什么没有状态机？

从代码设计来看，Photoview 的人脸识别采用了**极简模型**：

1. 没有状态字段，没有状态机
2. 没有操作历史，没有审计日志
3. 合并即删除，拆分即新建
4. 唯一的"状态"就是 label 的有/无

这是一种**"结果导向"**的设计：系统只关心"当前每个 FaceGroup 包含哪些 ImageFace"，不关心是怎么到达这个状态的。

---

## 9. UI 合并模态框的状态机

### 9.1 MergeFaceGroupsModal 状态

```
┌─────────┐    用户点击"合并"    ┌───────────────────┐
│  Closed  │ ──────────────────→ │ SelectDestination │
└─────────┘                      └────────┬──────────┘
     ↑                                    │ 点击"下一步"
     │                                    ▼
     │                           ┌───────────────────┐
     └────────────────────────── │  SelectSources    │
          合并完成/取消           └────────┬──────────┘
                                           │ 点击"合并"
                                           ▼
                                   调用 combineFacesMutation
                                   navigate(/people/${destID})
```

(`ui/src/Pages/PeoplePage/SingleFaceGroup/MergeFaceGroupsModal.tsx:31-35`)

如果提供了 `preselectedDestinationFaceGroup`（从某个 FaceGroup 页面发起合并），则跳过 `SelectDestination`，直接进入 `SelectSources`。

### 9.2 MoveImageFacesModal 状态

两步流程：
1. **选择要移动的 ImageFaces**（`SelectImageFacesTable`）
2. **选择目标 FaceGroup**（`SelectFaceGroupTable`，过滤掉当前组）

### 9.3 DetachImageFacesModal

单步流程：选择要拆分的 ImageFaces，点击"Detach"后自动创建新组。

---

## 10. 完整状态机协同图

```
                        ┌─────────────────────────────────────────────┐
                        │           媒体扫描                            │
                        │  FaceDetectionTask.AfterProcessMedia()       │
                        │  → DetectFaces() → classifyFace()            │
                        │  → ClassifyThreshold(descriptor, 0.2)        │
                        └───────────┬─────────────────────────────────┘
                                    │
                     ┌──────────────┴──────────────┐
                     │                             │
              match < 0                       match ≥ 0
                     │                             │
                     ▼                             ▼
            ┌──────────────┐            ┌────────────────────┐
            │ 新建 FaceGroup │            │ 加入已有 FaceGroup   │
            │ (无 label)    │            │ (可能无 label)      │
            └──────┬───────┘            └────────┬───────────┘
                   │                             │
                   └──────────┬──────────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │  用户看到分组结果    │
                    │  (PeoplePage)      │
                    └─────────┬─────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
     ┌─────────────┐  ┌──────────────┐  ┌────────────────┐
     │ 设置 Label   │  │ 手动合并/移动  │  │ 拆分到新组      │
     │ (标记确认)   │  │ (Combine/Move)│  │ (Detach)       │
     └──────┬──────┘  └──────┬───────┘  └───────┬────────┘
            │                │                   │
            │    ┌───────────┘                   │
            │    │  冲突检查：                    │
            │    │  同一 media 不允许在           │
            │    │  同一 FaceGroup 中重复         │
            │    │  → 拒绝并回滚                  │
            │    └───────────┐                   │
            │                │                   │
            ▼                ▼                   ▼
     ┌──────────────────────────────────────────────────┐
     │          DB 事务 + 内存同步 (MergeImageFaces)      │
     └──────────────────────┬───────────────────────────┘
                            │
                            ▼
               ┌───────────────────────────┐
               │  用户点击"识别未标记人脸"    │
               │  RecognizeUnlabeledFaces   │
               │                           │
               │  仅用已标记组做锚点          │
               │  ClassifyThreshold(0.2)    │
               │  匹配成功 → 自动合入已标记组  │
               │  匹配失败 → 保持原组        │
               └───────────────────────────┘
```

---

## 11. 关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| FaceDetector 接口定义 | `api/scanner/face_detection/face_detector.go` | 1-17 |
| 检测器实现（含分类逻辑） | `api/scanner/face_detection/face_detector_impl.go` | 1-307 |
| 阈值 0.2 分类 | `face_detector_impl.go` | 131 |
| classifyFace 自动聚类 | `face_detector_impl.go` | 134-189 |
| MergeImageFaces 内存同步 | `face_detector_impl.go` | 202-216 |
| MergeCategories 内存同步 | `face_detector_impl.go` | 191-200 |
| RecognizeUnlabeledFaces | `face_detector_impl.go` | 218-307 |
| CombineFaceGroups resolver | `api/graphql/resolvers/faces.go` | 144-225 |
| MoveImageFaces resolver | `faces.go` | 228-297 |
| DetachImageFaces resolver | `faces.go` | 328-374 |
| RecognizeUnlabeledFaces resolver | `faces.go` | 300-325 |
| 重复媒体检测（合组） | `faces.util.go` | 98-115 |
| 重复媒体检测（移动） | `faces.util.go` | 117-138 |
| 数据模型定义 | `api/graphql/models/face_detection.go` | 1-130 |
| GraphQL schema | `api/graphql/resolvers/faces.graphql` | 1-53 |
| 扫描任务触发 | `api/scanner/scanner_tasks/face_detection_task.go` | 1-31 |
| UI 合并模态框 | `ui/src/Pages/PeoplePage/SingleFaceGroup/MergeFaceGroupsModal.tsx` | 1-262 |
| UI 移动模态框 | `ui/src/Pages/PeoplePage/SingleFaceGroup/MoveImageFacesModal.tsx` | 1-203 |
| UI 拆分模态框 | `ui/src/Pages/PeoplePage/SingleFaceGroup/DetachImageFacesModal.tsx` | 1-155 |
| 合并冲突测试 | `api/graphql/resolvers/faces_test.go` | 93-216 |
| 环境变量定义 | `api/utils/environment_variables.go` | 21, 43 |
| FormatSQL 排序处理 | `api/graphql/models/utils.go` | 11 |
| FaceCircleImage 封面渲染 | `ui/src/Pages/PeoplePage/FaceCircleImage.tsx` | 1-136 |

---

## 12. 设计要点总结

1. **阈值 0.2 是保守策略**：自动聚类宁可过拆，把同一人拆成多个组，也不误合并不同人。手动操作弥补过度拆分。

2. **阈值硬编码，运维侧不可调**：`0.2` 是字面量写死在代码中，没有环境变量或配置项可以调整。运维侧只能开关人脸识别，不能调阈值。

3. **Label 是状态机的分水岭**：有 label 的 FaceGroup 成为重识别的"锚点"，无 label 的组是待确认的候选。用户的标记行为实质上在驱动状态转换。

4. **冲突检查保护语义一致性**：同一 FaceGroup 中同一 Media 只能有一个 ImageFace，避免"同一人在同一照片中被识别两次"的矛盾。

5. **封面是隐式的，无冲突处理**：FaceGroup 没有 `cover_image` 字段，封面由 `imageFaces[0]`（id 最小的人脸）自然决定。合并后封面归属是排序规则的副作用，而非专门设计。

6. **没有 face_state，没有状态机**：整个系统不追踪合并历史，不维护状态字段。合并即物理删除 source 组，拆分即新建组。

7. **合并不可逆，无 unmerge 路径**：合并操作是单向的，source 组被物理删除。用户只能通过 `DetachImageFaces` 近似回退，但无法恢复原组身份和结构。

8. **内存同步的延迟性**：`MergeImageFaces` 仅修改 `faceGroupIDs` 映射，不触发 `SetSamples`。分类器样本集的更新发生在下次 `classifyFace` 或 `RecognizeUnlabeledFaces` 时。

9. **CombineFaceGroups 的去重是防御性代码**：由于冲突检查在前，合并后的去重步骤在正常路径上不会触发，仅作为安全网存在。

10. **RecognizeUnlabeledFaces 会重建样本集**：先把未标记样本从内存中移除，用已标记样本作为训练集重新分类，这确保了标记操作对重识别的即时影响。
