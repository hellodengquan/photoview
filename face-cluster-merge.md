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

## 7. UI 合并模态框的状态机

### 7.1 MergeFaceGroupsModal 状态

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

### 7.2 MoveImageFacesModal 状态

两步流程：
1. **选择要移动的 ImageFaces**（`SelectImageFacesTable`）
2. **选择目标 FaceGroup**（`SelectFaceGroupTable`，过滤掉当前组）

### 7.3 DetachImageFacesModal

单步流程：选择要拆分的 ImageFaces，点击"Detach"后自动创建新组。

---

## 8. 完整状态机协同图

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

## 9. 关键代码位置索引

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

---

## 10. 设计要点总结

1. **阈值 0.2 是保守策略**：自动聚类宁可过拆，把同一人拆成多个组，也不误合并不同人。手动操作弥补过度拆分。

2. **Label 是状态机的分水岭**：有 label 的 FaceGroup 成为重识别的"锚点"，无 label 的组是待确认的候选。用户的标记行为实质上在驱动状态转换。

3. **冲突检查保护语义一致性**：同一 FaceGroup 中同一 Media 只能有一个 ImageFace，避免"同一人在同一照片中被识别两次"的矛盾。

4. **内存同步的延迟性**：`MergeImageFaces` 仅修改 `faceGroupIDs` 映射，不触发 `SetSamples`。分类器样本集的更新发生在下次 `classifyFace` 或 `RecognizeUnlabeledFaces` 时。

5. **CombineFaceGroups 的去重是防御性代码**：由于冲突检查在前，合并后的去重步骤在正常路径上不会触发，仅作为安全网存在。

6. **RecognizeUnlabeledFaces 会重建样本集**：先把未标记样本从内存中移除，用已标记样本作为训练集重新分类，这确保了标记操作对重识别的即时影响。
