# Photoview EXIF 地理位置解析与地图展示全链路流程

## 一、整体架构概览

Photoview 的地理位置功能采用 **后端抽取 + 前端渲染** 的分离架构：

```
媒体文件 → EXIF 解析 (exiftool) → GPS 验证 → 数据库存储 → GraphQL API → 前端地图 (Mapbox GL JS)
```

**关键澄清**：
- EXIF 元数据（包括 GPS）使用 **exiftool** 解析，而非 ffmpeg
- ffmpeg/ffprobe 仅用于视频元数据（分辨率、编码格式等），不涉及 GPS
- 前端使用 **Mapbox GL JS**，而非 Leaflet

---

## 二、EXIF 元数据解析层

### 2.1 工具选型与初始化

**代码位置**：`api/scanner/externaltools/exif/exif.go:15-41`

系统启动时初始化 exiftool 进程：

```go
func Initialize() (func(), error) {
    globalInit.Do(func() {
        globalExifParser, err = exiftool.New()  // 启动常驻 exiftool 进程
    })
    // ...
}
```

**exiftool 进程管理**：`api/scanner/externaltools/exiftool/exiftool.go:33-95`

采用 **`-stay_open True`** 模式保持进程常驻，避免频繁 fork 进程开销：

```go
cmd := exec.Command(path, "-stay_open", "True", "-@", "-")
```

### 2.2 元数据解析流程

**核心解析函数**：`api/scanner/externaltools/exif/exif.go:45-94`

```go
func Parse(filepath string) (*models.MediaEXIF, error) {
    var values struct {
        exiftool.PhotoMeta  // 相机参数、曝光等
        exiftool.TimeAll    // 拍摄时间
        exiftool.GPS        // 地理位置
    }
    // 通过 exiftool 查询 JSON 格式的标签
    if err := globalExifParser.QueryJSONTagsByNumber(filepath, &values); err != nil {
        return nil, err
    }
    
    // GPS 有效性检查
    if values.GPS.IsValid() {
        ret.GPSLatitude = values.GPS.GPSLatitude
        ret.GPSLongitude = values.GPS.GPSLongitude
    }
    
    return &ret, nil
}
```

**底层查询实现**：`api/scanner/externaltools/exiftool/exiftool.go:229-242`

```go
func (e *Exiftool) QueryJSONTagsByNumber(file string, value any) error {
    // -n 参数确保数值类型（如 GPS）以数字形式返回，而非度分秒格式
    return e.rawGetTags(&rows, "-n", file)
}
```

### 2.3 GPS 数据结构与验证

**GPS 结构体**：`api/scanner/externaltools/exiftool/values.go:10-43`

```go
type GPS struct {
    GPSLatitude  *float64
    GPSLongitude *float64
}

func (gps GPS) IsValid() bool {
    // 1. 非空检查
    if gps.GPSLongitude == nil || gps.GPSLatitude == nil { return false }
    // 2. NaN 检查
    if math.IsNaN(*gps.GPSLatitude) || math.IsNaN(*gps.GPSLongitude) { return false }
    // 3. 范围检查：纬度 ±90°，经度 ±180°
    if math.Abs(*gps.GPSLatitude) > 90 || math.Abs(*gps.GPSLongitude) > 180 { return false }
    return true
}
```

**无效 GPS 数据迁移**：`api/database/migrations/exif_invalid_gps.go:10-24`

系统提供数据库迁移清理历史无效数据：

```go
func MigrateForExifGPSCorrection(db *gorm.DB) error {
    return db.Model(&models.MediaEXIF{}).
        Where("ABS(gps_longitude) > ?", 90).
        Or("ABS(gps_latitude) > ?", 90).
        Updates(map[string]interface{}{
            "gps_latitude":  nil,
            "gps_longitude": nil,
        })
}
```

---

## 三、ffmpeg 与 exiftool 的职责分工

| 工具 | 用途 | 涉及 GPS | 代码位置 |
|------|------|----------|----------|
| **exiftool** | 照片/视频 EXIF 元数据解析（相机、GPS、时间等） | ✅ 是 | `api/scanner/externaltools/exiftool/` |
| **ffprobe** | 视频技术元数据（分辨率、编码、帧率等） | ❌ 否 | `api/scanner/scanner_tasks/video_metadata_task.go` |
| **ffmpeg** | 视频转码、缩略图生成 | ❌ 否 | `api/scanner/media_encoding/executable_worker/ffmpeg_cli.go` |

**视频元数据处理**：`api/scanner/scanner_tasks/video_metadata_task.go:35-84`

```go
func ScanVideoMetadata(tx *gorm.DB, video *models.Media) error {
    // 使用 ffprobe 读取视频流信息，不涉及 GPS
    data, err := processing_tasks.ReadVideoMetadata(video.Path)
    // ... 处理分辨率、帧率、编码等
}
```

> **注意**：即使是视频文件，其 EXIF 中的 GPS 信息仍然通过 exiftool 解析，在 `exif_task.go` 中统一处理。

---

## 四、扫描任务与数据入库

### 4.1 扫描任务流水线

**任务接口**：`api/scanner/scanner_task/scanner_task.go:16-36`

EXIF 解析是扫描流水线的一环：

```
MediaFound → AfterMediaFound (ExifTask) → BeforeProcessMedia → ProcessMedia → AfterProcessMedia
```

### 4.2 EXIF 任务执行

**EXIF 任务**：`api/scanner/scanner_tasks/exif_task.go:15-74`

```go
type ExifTask struct{}

func (t ExifTask) AfterMediaFound(ctx scanner_task.TaskContext, media *models.Media, newMedia bool) error {
    if !newMedia { return nil }  // 仅新媒体解析
    return SaveEXIF(ctx.GetDB(), media)
}

func SaveEXIF(tx *gorm.DB, media *models.Media) error {
    // 1. 检查是否已有 EXIF 数据
    if media.ExifID != nil { /* ... */ }
    
    // 2. 调用 exif.Parse 解析
    exifData, err := exif.Parse(media.Path)
    
    // 3. 关联保存到数据库
    tx.Model(media).Association("Exif").Replace(exifData)
    
    // 4. 更新媒体拍摄时间（从 EXIF 同步）
    if exifData.DateShot != nil && !exifData.DateShot.Equal(media.DateShot) {
        tx.Save(media)
        media.DateShot = *exifData.DateShot
    }
}
```

### 4.3 数据库模型

**MediaEXIF 模型**：`api/graphql/models/media_exif.go:8-25`

```go
type MediaEXIF struct {
    Model
    Description     *string
    Camera          *string    // 相机型号
    Maker           *string    // 厂商
    Lens            *string    // 镜头
    DateShot        *time.Time // 拍摄时间
    GPSLatitude     *float64   // 纬度
    GPSLongitude    *float64   // 经度
    // ... 其他 EXIF 字段
}
```

**数据库关联**：`api/graphql/models/media.go:15-33`

```go
type Media struct {
    ExifID   *int        `gorm:"index"`
    Exif     *MediaEXIF  `gorm:"constraint:OnDelete:CASCADE;"`
    // ...
}
```

---

## 五、后端 API 层

### 5.1 GraphQL Schema

**定义**：`api/graphql/resolvers/media_geo_json.graphql:1-7`

```graphql
extend type Query {
    "Get media owned by the logged in user, returned in GeoJson format"
    myMediaGeoJson: Any! @isAuthorized

    "Get the mapbox api token, returns null if mapbox is not enabled"
    mapboxToken: String
}
```

### 5.2 GeoJSON 数据组装

**Resolver 实现**：`api/graphql/resolvers/media_geo_json.go:17-71`

```go
func (r *queryResolver) MyMediaGeoJSON(ctx context.Context) (any, error) {
    // 1. 查询带 GPS 坐标的媒体及缩略图
    err := r.DB(ctx).Table("media").
        Select("media.id, media.title, media_urls.media_name, "+
               "media_exif.gps_latitude, media_exif.gps_longitude").
        Joins("INNER JOIN media_exif ON media.exif_id = media_exif.id").
        Joins("INNER JOIN media_urls ON media.id = media_urls.media_id").
        Where("media_exif.gps_latitude IS NOT NULL").
        Where("media_exif.gps_longitude IS NOT NULL").
        Where("media_urls.purpose = 'thumbnail'").
        Scan(&media).Error

    // 2. 组装 GeoJSON FeatureCollection
    features := make([]geoJSONFeature, 0)
    for _, item := range media {
        geoPoint := makeGeoJSONFeatureGeometryPoint(item.Latitude, item.Longitude)
        properties := geoJSONMediaProperties{
            MediaID:    item.MediaID,
            MediaTitle: item.MediaTitle,
            Thumbnail:  { URL: thumbnailURL, Width: ..., Height: ... },
        }
        features = append(features, makeGeoJSONFeature(properties, geoPoint))
    }

    return makeGeoJSONFeatureCollection(features), nil
}
```

**GeoJSON 工具函数**：`api/graphql/resolvers/media_geo_json.util.go:54-60`

```go
func makeGeoJSONFeatureGeometryPoint(lat float64, long float64) geoJSONFeatureGeometry {
    // 注意：GeoJSON 坐标顺序是 [经度, 纬度]
    coordinates := [2]float64{long, lat}
    return geoJSONFeatureGeometry{
        Type:        "Point",
        Coordinates: coordinates,
    }
}
```

### 5.3 Mapbox Token 获取

**代码位置**：`api/graphql/resolvers/media_geo_json.go:73-81`

```go
func (r *queryResolver) MapboxToken(ctx context.Context) (*string, error) {
    mapboxTokenEnv := os.Getenv("MAPBOX_TOKEN")
    if mapboxTokenEnv == "" { return nil, nil }
    return &mapboxTokenEnv, nil
}
```

---

## 六、前端地图渲染层

### 6.1 地图库选型

**package.json** 确认：
```json
"@types/mapbox-gl": "^2.7.3",
"mapbox-gl": "^2.9.1"
```

> **重要澄清**：前端使用 **Mapbox GL JS** 进行地图渲染，而非 Leaflet。Mapbox GL JS 支持矢量瓦片、3D 地形等高级特性。

### 6.2 地图初始化 Hook

**代码位置**：`ui/src/components/mapbox/MapboxMap.tsx:27-81`

```typescript
const useMapboxMap = ({ configureMapbox, mapboxOptions }) => {
    // 1. 异步加载 mapbox-gl 库
    useEffect(() => {
        async function loadMapboxLibrary() {
            const mapbox = (await import('mapbox-gl')).default
            setMapboxLibrary(mapbox)
        }
        loadMapboxLibrary()
    }, [])

    // 2. 查询 Mapbox token 和 GeoJSON 数据
    const { data: mapboxData } = useQuery<mapboxToken>(MAPBOX_TOKEN_QUERY)

    // 3. 初始化地图
    useEffect(() => {
        if (mapboxData.mapboxToken)
            mapboxLibrary.accessToken = mapboxData.mapboxToken

        map.current = new mapboxLibrary.Map({
            container: mapContainer.current,
            style: isDarkMode() 
                ? 'mapbox://styles/mapbox/dark-v10' 
                : 'mapbox://styles/mapbox/streets-v11',
            ...mapboxOptions,
        })

        configureMapbox(map.current, mapboxLibrary)
    }, [mapContainer, mapboxLibrary, mapboxData])
}
```

### 6.3 PlacesPage 地图页面

**代码位置**：`ui/src/Pages/PlacesPage/PlacesPage.tsx:20-138`

```typescript
// 查询 GeoJSON 数据
const MAPBOX_DATA_QUERY = gql`
    query mediaGeoJson {
        myMediaGeoJson
    }
`

const configureMapbox = ({ mapboxData, dispatchMarkerMedia }) => 
    (map: mapboxgl.Map, mapboxLibrary: typeof mapboxgl) => {
        map.on('load', () => {
            // 添加 GeoJSON 数据源，启用聚类
            map.addSource('media', {
                type: 'geojson',
                data: mapboxData?.myMediaGeoJson as never,
                cluster: true,
                clusterRadius: 50,
            })

            // 注册媒体标记渲染
            registerMediaMarkers({
                map: map,
                mapboxLibrary,
                dispatchMarkerMedia,
            })
        })
    }
```

### 6.4 标记渲染与聚类

**标记注册**：`ui/src/components/mapbox/mapboxHelperFunctions.tsx:22-73`

```typescript
export const registerMediaMarkers = (args: registerMediaMarkersArgs) => {
    const updateMarkers = makeUpdateMarkers(args)
    
    // 地图移动或数据源变化时更新标记
    args.map.on('move', updateMarkers)
    args.map.on('moveend', updateMarkers)
    args.map.on('sourcedata', updateMarkers)
    updateMarkers()
}

const makeUpdateMarkers = ({ map, mapboxLibrary, dispatchMarkerMedia }) =>
    () => {
        const features = map.querySourceFeatures('media')
        
        for (const feature of features) {
            const coords = (feature.geometry as geojson.Point).coordinates as [number, number]
            const props = feature.properties as MediaMarker
            
            // 创建或更新 HTML 标记
            const el = createClusterPopupElement(props, { dispatchMarkerMedia })
            marker = new mapboxLibrary.Marker({ element: el }).setLngLat(coords)
            marker.addTo(map)
        }
    }
```

**聚类标记组件**：`ui/src/Pages/PlacesPage/MapClusterMarker.tsx:48-73`

```typescript
const MapClusterMarker = ({ marker, dispatchMarkerMedia }: MapClusterMarkerProps) => {
    const thumbnail = JSON.parse(marker.thumbnail) as { url: string }
    
    const presentMedia = () => {
        dispatchMarkerMedia({
            type: 'replacePresentMarker',
            marker: {
                cluster: !!marker.cluster,
                id: marker.cluster ? marker.cluster_id : marker.media_id,
            },
        })
    }

    return (
        <Wrapper onClick={presentMedia}>
            <PopupImage src={imagePopupSrc} />
            <ThumbnailImage src={thumbnail.url} />
            {marker.cluster && (
                <PointCountCircle>{marker.point_count_abbreviated}</PointCountCircle>
            )}
        </Wrapper>
    )
}
```

### 6.5 侧边栏小地图

**代码位置**：`ui/src/components/sidebar/MediaSidebar/MediaSidebarMap.tsx:12-54`

单张照片详情页的小地图展示：

```typescript
const MediaSidebarMap = ({ coordinates }: MediaSidebarMapProps) => {
    const { mapContainer, mapboxToken } = useMapboxMap({
        mapboxOptions: {
            interactive: false,  // 禁用交互
            zoom: 12,
            center: {
                lat: coordinates.latitude,
                lng: coordinates.longitude,
            },
        },
        configureMapbox: (map, mapboxLibrary) => {
            // 添加单个红色标记
            const centerMarker = new mapboxLibrary.Marker({ color: 'red', scale: 0.8 })
            centerMarker.setLngLat({
                lat: coordinates.latitude,
                lng: coordinates.longitude,
            })
            centerMarker.addTo(map)
        },
    })
}
```

---

## 七、全链路时序图

```
用户扫描相册
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  扫描流水线 (Scanner Pipeline)                           │
│  ┌───────────────────┐                                  │
│  │  发现新媒体文件   │                                  │
│  └─────────┬─────────┘                                  │
│            │                                            │
│            ▼                                            │
│  ┌──────────────────────────────────────────┐           │
│  │ ExifTask.AfterMediaFound()               │           │
│  │  └─ SaveEXIF(db, media)                  │           │
│  │      ├─ 检查是否已有 EXIF                │           │
│  │      └─ exif.Parse(media.Path) ────┐     │           │
│  └────────────────────────────────────┼─────┘           │
│                                       │                 │
│                                       ▼                 │
│  ┌───────────────────────────────────────────────────┐  │
│  │ exif.Parse()                                      │  │
│  │  └─ globalExifParser.QueryJSONTagsByNumber()      │  │
│  │      └─ exiftool -n -json file.jpg                │  │
│  │          └─ 返回 { PhotoMeta, TimeAll, GPS }      │  │
│  │              ├─ GPS.IsValid() 验证                │  │
│  │              └─ 构建 models.MediaEXIF             │  │
│  └─────────────────────────────────────┬─────────────┘  │
│                                        │                │
│                                        ▼                │
│  ┌───────────────────────────────────────────────────┐  │
│  │ 数据库写入                                        │  │
│  │  ├─ INSERT INTO media_exif (gps_latitude, ...)   │  │
│  │  └─ UPDATE media SET exif_id = ?                 │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  前端地图页面 (PlacesPage)                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │ useQuery(mediaGeoJson)                            │  │
│  │  └─ GraphQL 查询 myMediaGeoJson                   │  │
│  │     └─ 返回 GeoJSON FeatureCollection             │  │
│  └───────────────────────┬───────────────────────────┘  │
│                          │                              │
│                          ▼                              │
│  ┌───────────────────────────────────────────────────┐  │
│  │ useMapboxMap()                                    │  │
│  │  ├─ 异步加载 mapbox-gl                            │  │
│  │  ├─ 设置 accessToken                              │  │
│  │  └─ new mapboxgl.Map()                            │  │
│  └───────────────────────┬───────────────────────────┘  │
│                          │                              │
│                          ▼                              │
│  ┌───────────────────────────────────────────────────┐  │
│  │ configureMapbox()                                 │  │
│  │  ├─ map.addSource('media', {                      │  │
│  │  │    type: 'geojson',                            │  │
│  │  │    data: myMediaGeoJson,                       │  │
│  │  │    cluster: true                               │  │
│  │  })                                               │  │
│  │  └─ registerMediaMarkers()                        │  │
│  │     └─ 监听 move/moveend/sourcedata 事件          │  │
│  │        └─ 动态渲染 MapClusterMarker 组件          │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## 八、关键设计要点

### 8.1 exiftool 常驻进程设计

- **优点**：避免频繁创建进程，提升批量扫描性能
- **实现**：`-stay_open True` + `-@ -` 从 stdin 读取命令
- **同步机制**：`{ready}` 标记分隔命令输出，`MarkReader` 实现带标记的流式读取

### 8.2 GPS 数据验证

- 三层验证：非空、非 NaN、范围检查
- 数据库迁移修复历史无效数据
- 无效数据静默丢弃（设为 nil）而非报错中断扫描

### 8.3 GeoJSON 坐标顺序

- 数据库存储：`gps_latitude`, `gps_longitude`（先纬后经）
- GeoJSON 输出：`[longitude, latitude]`（先经后纬，符合 RFC 7946 标准）

### 8.4 前端聚类实现

- 利用 Mapbox GL JS 内置的 GeoJSON 源聚类功能
- 自定义 HTML 标记，支持缩略图预览
- 地图移动时动态更新可见标记，优化性能

---

## 九、深度专题分析

### 9.1 exiftool 不存在时的降级行为：无降级，启动直接崩溃

#### 9.1.1 启动时的硬依赖

**代码位置**：`api/server.go:56-60`

```go
exifCleanup, err := exif.Initialize()
if err != nil {
    log.Panicf("Could not initialize exif parser: %s", err)
}
defer exifCleanup()
```

`exif.Initialize()` 内部会调用 `exiftool.New()`，而后者使用 `exec.LookPath("exiftool")` 查找二进制：

**代码位置**：`api/scanner/externaltools/exiftool/exiftool.go:34-38`

```go
func New() (*Exiftool, error) {
    path, err := exec.LookPath("exiftool")
    if err != nil {
        return nil, err  // 找不到直接返回错误，无任何 fallback
    }
    // ...
}
```

#### 9.1.2 解析时的保护

如果 somehow 绕过了启动检查（例如测试环境 mock），解析时会二次检查：

**代码位置**：`api/scanner/externaltools/exif/exif.go:49-51`

```go
func Parse(filepath string) (*models.MediaEXIF, error) {
    if globalExifParser == nil {
        return nil, fmt.Errorf("no exif parser initialized")
    }
    // ...
}
```

#### 9.1.3 ffmpeg/ffprobe 能否替代解析 GPS？

**结论：不能。**

从 `video_metadata_task.go:35-84` 和 `process_video_task.go:206-216` 可以看到，`ffprobe.ProbeURL()` 仅返回容器级和流级技术元数据：

```go
func ReadVideoMetadata(videoPath string) (*ffprobe.ProbeData, error) {
    data, err := ffprobe.ProbeURL(ctx, videoPath)
    // 返回数据结构：ProbeData { Format, Streams []Stream }
    //  - Format: duration, bit_rate, size, format_name, tags 等
    //  - Stream: codec, width, height, r_frame_rate, bit_rate 等
    //  不包含任何 GPS/EXIF 标签
}
```

go-ffprobe 库（`gopkg.in/vansante/go-ffprobe.v2`）默认不查询 `-show_entries format_tags=location`，代码中也从未传入 GPS 相关的 ffprobe 参数。

#### 9.1.4 实际影响

| 场景 | 行为 |
|------|------|
| 系统无 exiftool | `server.go` `log.Panicf` 导致进程退出，服务无法启动 |
| 运行时 exiftool 被意外卸载 | 下次扫描调用 `exif.Parse()` 返回 `"no exif parser initialized"`，该文件 EXIF 为空，GPS 丢失 |
| Docker 部署 | `dependencies/Dockerfile` 中预装 exiftool，正常部署不会遇到此问题 |

#### 9.1.5 exiftool 与 ffmpeg/ffprobe 的字段名命名差异映射

**代码证据：无映射代码，两套完全独立的字段体系**

**exiftool 的 JSON 字段解析路径**：`exiftool.go:158-172` + `values.go:10-43`

```go
func (e *Exiftool) rawGetTags(v any, args ...string) (err error) {
    if err = e.rawSendCommand(append(args, "-json")...); err != nil { return }
    // 直接 json.Decode 到结构体，字段名 1:1 匹配
    if err = json.NewDecoder(e.stdout).Decode(v); err != nil { return }
    return nil
}
```

Go 结构体字段定义：`values.go:11-13`

```go
type GPS struct {
    GPSLatitude  *float64  // ← 直接匹配 exiftool JSON 输出的 "GPSLatitude" 键
    GPSLongitude *float64  // ← 直接匹配 exiftool JSON 输出的 "GPSLongitude" 键
}
```

**exiftool 实际 JSON 输出**（`-n -json` 参数）：
```json
[{
  "SourceFile": "IMG_1234.jpg",
  "GPSLatitude": 44.4789972,
  "GPSLongitude": 11.2979222,
  "GPSLatitudeRef": "N",
  "GPSLongitudeRef": "E"
}]
```

**ffprobe 可能的 GPS 字段（理论上，代码中实际未使用）**：

ffprobe/ffmpeg 从视频容器元数据中读取位置信息时使用完全不同的字段名：

| 容器格式 | ffprobe 字段名 | 示例值 |
|----------|---------------|--------|
| QuickTime/MP4 | `format.tags.location` | `+44.4790+011.2979/` |
| QuickTime/MP4 | `format.tags.com.apple.quicktime.location.ISO6709` | `+44.478997+011.297922+0123.456CRSWGS_84/` |
| MKV | `format.tags.GPS` | `44.478997, 11.297922` |
| AVI | 无标准字段 | — |

**关键差异对比表**：

| 属性 | exiftool 路径 | ffprobe 可能路径（代码中未用） |
|------|--------------|-------------------------------|
| 字段名 | `GPSLatitude`, `GPSLongitude` | `location`, `com.apple.quicktime.location.ISO6709` |
| 输出格式 | 单独两个数值字段 | ISO 6709 字符串（需手动解析） |
| 调用参数 | `-n -json -gps:all` | `-show_entries format_tags=location` |
| Go 解析 | `json.Unmarshal` 直接反序列化 | 需自定义字符串解析（`+44.4790+011.2979/` → lat, long） |
| 代码状态 | ✅ 已实现 | ❌ 未实现，也无任何映射转换代码 |

**结论**：不存在任何将 `Latitude` ↔ `GPSLatitude` 或 `location` ↔ `GPSLatitude` 的字段名映射代码。两套解析路径使用完全独立的字段命名体系，Photoview 选择了 exiftool 体系，从未与 ffmpeg 体系做过对齐。

#### 9.1.6 alias_map 别名表与 ffmpeg metadata 字段覆盖分析

**代码证据：项目中不存在 alias_map 别名表。**

```bash
$ grep -rn "alias_map\|aliasMap\|AliasMap" api/ ui/
# 无结果
```

exiftool 的字段解析完全依赖 Go 标准库 `json.Decode` 的 struct tag 隐式映射机制，无任何显式别名表。映射逻辑位于两层结构体之间：

**第一层：exiftool JSON → `exiftool` 包内部结构体**（`values.go:10-171`）

通过 `QueryJSONTagsByNumber()` 调用 `json.NewDecoder(e.stdout).Decode(v)` 直接反序列化，字段名大小写敏感的 1:1 匹配：

| exiftool JSON 键 | Go 结构体字段（values.go） | 类型 |
|-----------------|---------------------------|------|
| `GPSLatitude` | `GPS.GPSLatitude` | `*float64` |
| `GPSLongitude` | `GPS.GPSLongitude` | `*float64` |
| `SubSecDateTimeOriginal` | `TimeAll.SubSecDateTimeOriginal` | `*string` |
| `SubSecCreateDate` | `TimeAll.SubSecCreateDate` | `*string` |
| `DateTimeOriginal` | `TimeAll.DateTimeOriginal` | `*string` |
| `CreateDate` | `TimeAll.CreateDate` | `*string` |
| `TrackCreateDate` | `TimeAll.TrackCreateDate` | `*string` |
| `MediaCreateDate` | `TimeAll.MediaCreateDate` | `*string` |
| `FileModifyDate` | `TimeAll.FileModifyDate` | `*string` |
| `OffsetTimeOriginal` | `TimeAll.OffsetTimeOriginal` | `*string` |
| `OffsetTime` | `TimeAll.OffsetTime` | `*string` |
| `TimeZone` | `TimeAll.TimeZone` | `*int` |
| `GPSDateTime` | `TimeAll.GPSDateTime` | `*string` |
| `ImageDescription` | `PhotoMeta.ImageDescription` | `*string` |
| `Model` | `PhotoMeta.Model` | `*string` |
| `Make` | `PhotoMeta.Make` | `*string` |
| `LensModel` | `PhotoMeta.LensModel` | `*string` |
| `ISO` | `PhotoMeta.ISO` | `*int64` |
| `Flash` | `PhotoMeta.Flash` | `*int64` |
| `Orientation` | `PhotoMeta.Orientation` | `*int64` |
| `ExposureProgram` | `PhotoMeta.ExposureProgram` | `*int64` |
| `ExposureTime` | `PhotoMeta.ExposureTime` | `*float64` |
| `Aperture` | `PhotoMeta.Aperture` | `*float64` |
| `FocalLength` | `PhotoMeta.FocalLength` | `*float64` |
| `MIMEType` | `MIMEType.MIMEType` | `*string` |

**第二层：`exiftool` 结构体 → `models.MediaEXIF` 数据库模型**（`exif.go:64-91`）

这是项目中唯一的"映射"逻辑，但它不是别名表，而是**显式的字段赋值代码**：

```go
ret := models.MediaEXIF{
    Camera:          values.Model,            // PhotoMeta.Model → MediaEXIF.Camera（字段名不同！）
    Maker:           values.Make,             // PhotoMeta.Make → MediaEXIF.Maker
    Lens:            values.LensModel,        // PhotoMeta.LensModel → MediaEXIF.Lens（字段名不同！）
    Iso:             values.ISO,              // PhotoMeta.ISO → MediaEXIF.Iso（大小写不同！）
    Flash:           values.Flash,            // PhotoMeta.Flash → MediaEXIF.Flash
    Orientation:     values.Orientation,      // PhotoMeta.Orientation → MediaEXIF.Orientation
    ExposureProgram: values.ExposureProgram,  // PhotoMeta.ExposureProgram → MediaEXIF.ExposureProgram
    Exposure:        values.ExposureTime,     // PhotoMeta.ExposureTime → MediaEXIF.Exposure（字段名不同！）
    Aperture:        values.Aperture,         // PhotoMeta.Aperture → MediaEXIF.Aperture
    FocalLength:     values.FocalLength,      // PhotoMeta.FocalLength → MediaEXIF.FocalLength
    Description:     values.ImageDescription, // PhotoMeta.ImageDescription → MediaEXIF.Description（字段名不同！）
    DateShot:        ...,                     // TimeAll.TimeInLocal() 经过时间解析后赋值
    OffsetSecShot:   ...,                     // TimeAll.OffsetSecs() 经过偏移计算后赋值
    GPSLatitude:     values.GPS.GPSLatitude,  // GPS.GPSLatitude → MediaEXIF.GPSLatitude
    GPSLongitude:    values.GPS.GPSLongitude, // GPS.GPSLongitude → MediaEXIF.GPSLongitude
}
```

**字段名差异清单**（第二层映射中发生重命名的字段）：

| 源字段（exiftool） | 目标字段（MediaEXIF） | 差异类型 |
|---------------------|----------------------|----------|
| `Model` | `Camera` | 语义重命名 |
| `LensModel` | `Lens` | 简化命名 |
| `ISO` | `Iso` | 大小写差异（全大写 → PascalCase） |
| `ExposureTime` | `Exposure` | 语义简化 |
| `ImageDescription` | `Description` | 简化命名 |
| `TimeAll.*` (多个字段) | `DateShot` | 多字段优先级合并解析 |
| `TimeAll.*` + GPS 时间 | `OffsetSecShot` | 多字段计算得出 |

**ffmpeg/ffprobe 能暴露但 Photoview 完全未覆盖的 metadata 字段**：

ffprobe 的 `ProbeData` 数据结构（来自 `gopkg.in/vansante/go-ffprobe.v2`）包含以下 Photoview 完全未使用的字段：

| ffprobe 字段路径 | 含义 | Photoview 是否覆盖 |
|-----------------|------|-------------------|
| `Format.Tags["title"]` | 视频标题 | ❌ 未覆盖（照片走 exiftool） |
| `Format.Tags["artist"]` | 艺术家/作者 | ❌ 未覆盖 |
| `Format.Tags["album"]` | 专辑 | ❌ 未覆盖 |
| `Format.Tags["date"]` | 创作日期 | ❌ 未覆盖（照片走 exiftool） |
| `Format.Tags["genre"]` | 流派 | ❌ 未覆盖 |
| `Format.Tags["comment"]` | 注释 | ❌ 未覆盖（照片走 exiftool） |
| `Format.Tags["copyright"]` | 版权信息 | ❌ 未覆盖 |
| `Format.Tags["location"]` | GPS 位置（ISO 6709） | ❌ 完全未使用 |
| `Format.Tags["com.apple.quicktime.location.ISO6709"]` | Apple 位置信息 | ❌ 完全未使用 |
| `Format.Tags["com.apple.quicktime.make"]` | Apple 设备厂商 | ❌ 未覆盖 |
| `Format.Tags["com.apple.quicktime.model"]` | Apple 设备型号 | ❌ 未覆盖 |
| `Stream.CodecName` | 编解码器短名 | ✅ 间接（`CodecLongName`） |
| `Stream.PixFmt` | 像素格式 | ❌ 未覆盖 |
| `Stream.ColorSpace` | 色彩空间 | ✅ 间接（`Profile` → ColorProfile） |
| `Stream.DisplayAspectRatio` | 显示宽高比 | ❌ 未覆盖 |
| `Stream.Rotation` | 旋转角度 | ❌ 未覆盖（照片走 exiftool Orientation） |
| `Stream.Level` | 编码级别 | ❌ 未覆盖 |
| `Stream.SampleAspectRatio` | 采样宽高比 | ❌ 未覆盖 |
| `Stream.FieldOrder` | 场序（逐行/隔行） | ❌ 未覆盖 |
| `Stream.ChromaSubsampling` | 色度子采样 | ❌ 未覆盖 |

**ffprobe 中 Photoview 已覆盖的字段**（`video_metadata_task.go:66-75`）：

| ffprobe 字段 | MediaEXIF / VideoMetadata 字段 |
|-------------|-------------------------------|
| `Stream.Width` | `VideoMetadata.Width` |
| `Stream.Height` | `VideoMetadata.Height` |
| `Format.DurationSeconds` | `VideoMetadata.Duration` |
| `Stream.CodecLongName` | `VideoMetadata.Codec` |
| `Stream.AvgFrameRate` → 解析 | `VideoMetadata.Framerate` |
| `Stream.BitRate` | `VideoMetadata.Bitrate` |
| `Stream.Profile` | `VideoMetadata.ColorProfile` |
| `Stream.Channels` → 文本描述 | `VideoMetadata.Audio` |

**总结**：不存在显式 `alias_map` 数据表/变量，字段映射通过两层机制实现：
1. JSON 反序列化的隐式 struct 字段名匹配（exiftool JSON → values.go）
2. `exif.go:64-91` 中手写的显式赋值语句（values.go → MediaEXIF）

ffmpeg/ffprobe 能暴露的 metadata 字段中，Photoview 仅覆盖了视频技术参数（宽、高、时长、编码、帧率、码率、色彩、音频通道数）约 8 个字段，剩余 20+ 个标签/位置/色彩细节字段完全未涉及。

#### 9.1.7 strings.HasPrefix 兜底匹配在 GPS 字段大小写/空格差异下的命中分析

**代码证据：项目中不使用 strings.HasPrefix 做 GPS 字段匹配。**

```bash
$ grep -rn "HasPrefix\|HasSuffix\|strings\.Equal\|strings\.ToLower\|strings\.Contains" api/scanner/externaltools/exiftool/
# 无结果
```

整个 `exiftool` 包中**没有任何字符串模糊匹配函数调用**。GPS 字段解析的唯一机制是 Go 标准库 `encoding/json.Decoder` 的 struct 字段名匹配。

**Go json.Decode 字段匹配规则**（`exiftool.go:167`）：

```go
if err = json.NewDecoder(e.stdout).Decode(v); err != nil {
    return
}
```

Go 的 `json.Decoder` 使用以下规则将 JSON 键映射到 struct 字段：

1. **精确匹配**：JSON 键必须与 Go struct 字段名**完全一致**（大小写敏感）
2. **case-insensitive 回退**：仅当精确匹配失败时，才尝试不区分大小写匹配
3. **无空格/前缀匹配**：不支持 `HasPrefix`、空格忽略或任何模糊匹配

**具体测试：大小写和空格差异能否命中**

| JSON 键（exiftool 输出） | Go struct 字段 | 精确匹配 | case-insensitive 回退 | 最终结果 |
|--------------------------|---------------|----------|----------------------|----------|
| `GPSLatitude` | `GPSLatitude` | ✅ | — | ✅ 命中 |
| `gpslatitude` | `GPSLatitude` | ❌ | ✅ | ✅ 命中 |
| `gpsLATITUDE` | `GPSLatitude` | ❌ | ✅ | ✅ 命中 |
| `GPS Latitude` | `GPSLatitude` | ❌ | ❌ | ❌ **丢失** |
| `GPS_Latitude` | `GPSLatitude` | ❌ | ❌ | ❌ **丢失** |
| `Latitude` | `GPSLatitude` | ❌ | ❌ | ❌ **丢失** |
| `gpsLatitude` | `GPSLatitude` | ❌ | ✅ | ✅ 命中 |

**关键问题：exiftool 实际输出中是否会出现带空格的键？**

exiftool 使用 `-n` 参数输出数值格式（`exiftool.go:233`），此时 JSON 键名遵循 exiftool 的标准命名规范：

```
exiftool 标准键名规则：
- EXIF GPS 标签：GPSLatitude, GPSLongitude, GPSLatitudeRef, GPSLongitudeRef, GPSAltitude
- IPTC 标签：无空格，如 Country, City
- XMP 标签：exif:GPSLatitude, xmp:Creator（带命名空间前缀，但无空格）
- QuickTime 标签：TrackCreateDate, MediaCreateDate
- Composite 标签：SubSecDateTimeOriginal
```

exiftool **永远不会在 JSON 键中输出空格**。它的键名来自 EXIF/XMP/IPTC 标签的规范名称，这些规范名称本身就不用空格。

**但如果文件元数据中存在非标准标签（如某些软件写入的自定义 XMP 扩展），会发生什么？**

```json
[{
  "SourceFile": "photo.jpg",
  "GPSLatitude": 44.4789972,      ← 标准 EXIF，Go 匹配成功
  "GPS Longitude": 44.4789972,     ← 不可能：exiftool 不输出带空格的键
  "xmp:GPSLatitude": 44.4789972,   ← XMP 命名空间前缀，Go 精确匹配失败
                                      case-insensitive 也失败（多了 "xmp:" 前缀）
                                      → GPS 数据丢失！
}]
```

**DisallowUnknownFields 缺失的影响**：

`exiftool.go:167` 没有调用 `decoder.DisallowUnknownFields()`，这意味着：
- 无法匹配的 JSON 键会被**静默丢弃**，不会报错
- 运维和开发者**无法得知**哪些 EXIF 字段因键名不匹配而丢失
- 特别是 XMP 命名空间前缀的标签（如 `xmp:GPSLatitude`）会静默丢失

**实际风险场景**：

| 场景 | 风险等级 | 原因 |
|------|----------|------|
| 正常相机 EXIF GPS | 无风险 | exiftool 输出 `GPSLatitude`，精确匹配 |
| 手机照片 EXIF GPS | 无风险 | 同上 |
| 含 XMP 扩展 GPS 的照片 | **中风险** | `xmp:GPSLatitude` 不匹配 `GPSLatitude`，丢失但 EXIF GPS 仍存在 |
| 经视频编辑器处理的 MP4 | **中风险** | 视频元数据可能走 `com.apple.quicktime.location.ISO6709`，而非 `GPSLatitude` |
| 第三方软件写入的自定义 XMP | **低风险** | exiftool 可能合并输出，但不保证键名匹配 |

**结论**：不存在 `strings.HasPrefix` 兜底匹配机制。Go 的 `json.Decode` 仅提供精确匹配 + case-insensitive 回退，不支持空格/前缀/命名空间模糊匹配。exiftool 的标准输出不会包含带空格的键名，因此"GPS Latitude" vs "GPSLatitude" 的情况在实际中不会发生。但 XMP 命名空间前缀（如 `xmp:GPSLatitude`）会导致静默丢失，且由于缺少 `DisallowUnknownFields()`，这类丢失不可观测。

---

### 9.2 GPS 坐标精度与 geofence 边界场景影响

#### 9.2.1 精度存储链路

| 层级 | 类型 | 精度 | 代码位置 |
|------|------|------|----------|
| exiftool 输出 | JSON number (float64) | 原始 EXIF 精度（通常 7+ 位小数） | `exiftool.go:231-242` `-n` 参数输出数值 |
| Go 内存模型 | `*float64` | IEEE 754 双精度（~15-17 位有效数字） | `media_exif.go:23-24` |
| 数据库列 | DOUBLE PRECISION / REAL | 取决于数据库（通常也是 float64） | GORM AutoMigrate 自动映射 |
| GeoJSON 序列化 | JSON number | JavaScript number（IEEE 754 双精度） | `media_geo_json.go:54-60` |

**测试验证精度**：`exiftool_test.go:302-308`

```go
gpsToString := func(latitude, longitude float64) string {
    return fmt.Sprintf("(%.7f, %.7f)", latitude, longitude)
}
// 测试数据：44.4789972, 11.2979222（7 位小数）
```

#### 9.2.2 "6 位小数截断" 的来源与实际情况

代码中**并没有显式截断到 6 位小数**。GPS 值从 exiftool 解析后以 float64 全程传递，精度损失仅来自：

1. **JSON 序列化**：JavaScript 的 `JSON.parse`/`JSON.stringify` 使用双精度，通常输出 6-7 位小数后会有浮点误差
2. **EXIF 原始精度**：大多数相机 GPS 记录本身就只有 6-7 位小数精度（约 10cm-1m 级别）

#### 9.2.3 小数位数与地理距离换算表

| 小数位数 | 纬度方向误差 | 经度方向误差（赤道） | 经度方向误差（纬度 45°） | 典型场景 |
|----------|-------------|---------------------|-------------------------|----------|
| 0 位 | 111 km | 111 km | 78.7 km | 国家/大洲级 |
| 1 位 | 11.1 km | 11.1 km | 7.87 km | 城市级 |
| 2 位 | 1.11 km | 1.11 km | 0.787 km | 街区级 |
| 3 位 | 111 m | 111 m | 78.7 m | 建筑群级 |
| 4 位 | 11.1 m | 11.1 m | 7.87 m | 独栋房屋 |
| 5 位 | 1.11 m | 1.11 m | 0.787 m | 单人定位 |
| 6 位 | 0.111 m | 0.111 m | 0.0787 m | 亚米级，民用 GPS 极限 |
| 7 位 | 0.0111 m | 0.0111 m | 0.00787 m | 测绘级 |

#### 9.2.3.1 GPS 精度截断到 6 位小数的实际误差量化（跨越 geofence 边界场景）

**代码证据：无显式截断代码**

从 `values.go:42` 的格式化输出 `fmt.Sprintf("GPS(%.9f, %.9f)", ...)` 可以看出，代码内部使用 9 位小数精度；从 `exif_test.go:306` 测试用例 `%.7f` 也验证了至少保留 7 位。整个链路（exiftool → Go float64 → 数据库 → JSON）都没有 `fmt.Sprintf("%.6f", ...)`、`math.Round()` 或 `toFixed(6)` 等显式截断操作。

**但假设因外部系统或数据导入导致 6 位小数截断时，geofence 边界场景的误差可以精确量化如下：**

**截断误差模型**：
```
截断操作：value_truncated = floor(value_original × 10^6) / 10^6
最大正向误差：+0.000000999...°（约 0.111m）
最大负向误差：0°
平均误差：+0.0000005°（约 0.0556m）
注意：舍入（round）与截断（trunc）不同，舍入误差范围为 ±0.0000005°
```

**场景 A：跨越国界/省界线（线状围栏，精度要求中等）**

假设边界为直线 y = 0，真实点在边界北侧 0.08m（y = 0.00000072°）：
```
真实坐标：(0°, 0.00000072°) → 在边界北侧（属于国家 A）
截断到 6 位：(0°, 0.000000°) → 恰好落在边界线上
→ 误判为边界归属不确定，50% 概率划入错误国家

如果真实点在北侧 0.03m（y = 0.00000027°）：
真实坐标：(0°, 0.00000027°) → 北侧
截断坐标：(0°, 0.000000°) → 边界线
→ 误判率 50%

如果真实点在北侧 0.12m（y = 0.00000108°）：
真实坐标：(0°, 0.00000108°) → 北侧
截断坐标：(0°, 0.000001°) → 北侧
→ 正确判断，0% 误判率
```

**边界误差带宽度**：由于 6 位小数的精度是 0.111m，因此存在一条 **宽度为 0.111m 的误差带**，边界两侧各 0.111m 范围内的点都可能被误判。

**场景 B：顺时针多边形围栏（含孔洞，典型 geofence）**

考虑一个 100m × 100m 的正方形围栏，边界定义精度为 6 位小数：
```
顶点定义（6 位小数）：
A(0.000000, 0.000000), B(0.000900, 0.000000)  → AB 边
C(0.000900, 0.000900), D(0.000000, 0.000900)  → CD 边
围栏面积：约 10,000 m²
误差带面积（周长 × 0.111m）：400m × 0.111m = 44.4 m²
边界区域误判率：44.4 / 10,000 ≈ 0.444%

对于更小的围栏（10m × 10m）：
围栏面积：100 m²
误差带面积：40m × 0.111m = 4.44 m²
边界区域误判率：4.44%

对于极小围栏（1m × 1m，如判断是否在某张椅子上拍摄）：
围栏面积：1 m²
误差带面积：4m × 0.111m = 0.444 m²
边界区域误判率：44.4%
```

**场景 C：距离阈值查询（查找 50m 范围内的所有照片）**

使用 Haversine 公式计算距离时，6 位小数截断引入的距离误差：
```
两点真实距离：d_true
两点截断距离：d_trunc
最大相对误差：Δd/d ≈ 2 × 0.111m / 50m ≈ 0.444%
最大绝对误差：Δd ≈ 0.222m（两点都在边界附近相反方向）

对 50m 查询半径的影响：
边界距离 49.8m 的真实点 → 截断后可能计算为 50.02m，被错误排除
边界距离 50.2m 的真实点 → 截断后可能计算为 49.98m，被错误包含
误差带内样本比例（假设均匀分布）：0.444%
```

**场景 D：时间序列轨迹穿越 geofence 边界**

假设移动速度 v = 1.5 m/s（步行），采样间隔 t = 1s：
```
时间步长位移：1.5m
6 位小数精度：0.111m
边界穿越检测延迟：最多 0.111m / 1.5m/s ≈ 0.074s（可忽略）
但如果边界定义精度本身只有 4 位小数（~10m），
则 GPS 精度不是瓶颈，边界定义精度才是。
```

**误差量化总结表**：

| geofence 场景 | 围栏规模 | 6 位小数截断误判率 | 可接受阈值 | 是否影响业务 |
|---------------|----------|-------------------|------------|--------------|
| 国家/省界 | 100km 级 | ~0.000001% | <1% | 否 |
| 城市/区县界 | 10km 级 | ~0.0001% | <1% | 否 |
| 街区/校园 | 100m 级 | ~0.44% | <1% | 边缘场景 |
| 独栋建筑 | 10m 级 | ~4.44% | <5% | 是，边界敏感 |
| 房间/设备 | 1m 级 | ~44.4% | <10% | 严重不可用 |
| 50m 近邻查询 | — | ~0.44% | <1% | 可忽略 |

#### 9.2.3.2 GPS 6 位小数截断（±11.1cm）在 nm 级精度科研场景下的影响

**代码证据：Photoview 中不存在任何纳米级精度相关功能**

```bash
$ grep -rn "nanometer\|nm\|nanometre\|RTK\|差分\|survey\|geodetic" api/ ui/
# 无任何结果
```

**精度量级对比**：

| 精度级别 | 典型误差 | 获取方式 | Photoview 能否达到 |
|----------|----------|----------|-------------------|
| 6 位小数（Photoview 当前） | ±11.1 cm | 消费级 GPS（手机、运动相机） | ✅ 可以（float64 足够） |
| 7 位小数 | ±1.11 cm | 中端 GNSS | ✅ 可以（float64 足够） |
| 8 位小数 | ±1.11 mm | RTK 差分 GPS | ✅ 可以（float64 足够） |
| 9 位小数 | ±0.111 mm | 大地测量级设备 | ✅ 理论可以（float64 约 ~15-17 位有效数字） |
| **1 nm（纳米）** | 0.000001 mm = 10⁻⁹ m | 激光干涉仪/原子级测量 | ❌ 不相关 |

**核心问题：GPS 不可能达到 nm 级精度**

GPS/GNSS 系统的物理极限：
- **民用 GPS（C/A 码）**：标称精度 3-5m（开放天空），实际 5-10m
- **DGPS（差分 GPS）**：精度 0.5-3m
- **RTK（实时动态差分）**：精度 1-5cm，基站覆盖区内
- **PPK（事后动态差分）**：精度 5mm-1cm，后处理
- **大地测量级静态观测**（数小时）：精度 0.1-1mm

即使是最精密的大地测量 GPS，精度极限也在 **亚毫米级（0.1mm = 10⁵ nm）**，距离 1nm 还有 **5 个数量级**的差距。GPS 通过电磁波测距，波长 19cm（L1 载波），物理上不可能达到纳米级精度。

**nm 级精度的典型应用场景**：
- 半导体芯片制造（光刻定位）
- 原子力显微镜（AFM）
- X 射线晶体学
- 量子计量学
- 精密光学系统对准

这些场景与照片地理定位完全无关，照片的 GPS 标签不可能也不需要纳米级精度。

**如果 Photoview 误用在高精度科研场景（如 RTK 无人机测绘）的实际影响**：

假设使用 RTK 设备采集了毫米级精度的 GPS 数据（如 44.478997231, 11.297922264），截断到 6 位小数后：

```
原始坐标（9 位小数，RTK 精度 5mm）：
  lat = 44.478997231°  →  截断为 44.478997°  →  误差 = 0.000000231° ≈ 0.0256m = 2.56cm
  lon = 11.297922264°  →  截断为 11.297922°  →  误差 = 0.000000264° ≈ 0.0293m = 2.93cm
欧氏距离误差：√(2.56² + 2.93²) ≈ 3.89cm

相对误差：3.89cm / 5mm(RTK标称精度) ≈ 7.8 倍精度损失
```

对典型 RTK 应用的影响：
| 应用 | 精度要求 | 6 位小数截断是否可用 |
|------|----------|---------------------|
| 农业植保无人机 | ±10-30cm | ✅ 可用 |
| 地形测绘（1:500 比例尺） | ±2-5cm | ⚠️ 临界，损失部分精度 |
| 建筑物变形监测 | ±1-5mm | ❌ 不可用，误差 10 倍于要求 |
| 大地控制点测量 | ±0.1-1mm | ❌ 完全不可用 |

**Photoview 架构层面不支持高精度的根本原因**：
1. `values.go:42` 使用 `%.9f` 格式化，但这只是显示格式，存储仍是 float64
2. 前端地图使用 Mapbox GL JS，渲染精度在像素级（Zoom 22 时约 1cm/像素），完全看不到 mm 级差异
3. `myMediaGeoJson` 返回的 GeoJSON 中坐标由 JavaScript 浮点数处理，精度虽够但渲染不支持
4. 业务逻辑中无任何"高精度定位模式"开关或相关功能

**结论**：nm 级精度场景与 Photoview 的产品定位（个人照片管理）完全脱节。GPS 技术本身也不可能达到 nm 级。即使是要求最高的 RTK 测绘场景（mm 级），float64 存储本身没问题，但 6 位小数截断会引入 ~2.5-4cm 误差，超出 RTK 标称精度约 5-8 倍。Photoview 不适合也不应用于科研级高精度定位场景。

#### 9.2.4 geofence 边界场景的具体影响

假设 6 位小数的最大舍入误差为 ±0.0000005°：

**场景 1：点状围栏（判断照片是否在某地 10m 范围内）**

```
围栏半径 r = 10m
6 位小数最大位置误差 ≈ ±0.056m（对角线方向）
误判率：可忽略（误差远小于围栏半径的 1%）
```

**场景 2：线状围栏（判断照片在道路的哪一侧，道路宽 5m）**

```
道路宽度 w = 5m
最大位置误差 ≈ ±0.056m
误判率：边界处约 2.2% 的照片可能被错误划分到另一侧
```

**场景 3：面状围栏（国家/省份边界，精度敏感场景）**

```
如果边界本身定义精度只有 4 位小数（~10m 级别），
则 6 位小数存储完全够用，不会成为瓶颈。
如果使用精密测绘的多边形围栏（厘米级），
则 6 位小数会在锐角拐点处引入最大 5.6cm 误差。
```

**场景 4：超近距聚类（判断两张照片是否同一位置拍摄）**

```
聚类阈值 d = 1m
6 位小数误差 ≈ ±0.056m
对聚类结果影响：<6% 的边界样本可能被错误合并/拆分
```

#### 9.2.5 Photoview 当前实现的边界敏感性

从 `media_geo_json.util.go:54-60` 可知 GeoJSON 输出直接传递 float64：

```go
func makeGeoJSONFeatureGeometryPoint(lat float64, long float64) geoJSONFeatureGeometry {
    coordinates := [2]float64{long, lat}  // 直接赋值，无截断
    // ...
}
```

**geofence 功能当前状态**：Photoview 代码中**未实现**基于 GPS 的地理围栏查询/过滤功能。`myMediaGeoJson` 仅返回全部带 GPS 的点，筛选完全在前端由用户肉眼完成。因此精度问题在现有功能中**没有实际影响**。

---

### 9.3 前端聚类算法在百万照片量级的性能瓶颈与分级策略

#### 9.3.1 聚类技术栈

| 组件 | 实现 | 代码位置 |
|------|------|----------|
| 聚类引擎 | Mapbox GL JS GeoJSON Source（底层 supercluster ^7.1.4） | `ui/package.json` |
| 聚类参数 | `cluster: true, clusterRadius: 50` | `PlacesPage.tsx:114-122` |
| 标记渲染 | 自定义 HTML Marker + ReactDOM.render | `mapboxHelperFunctions.tsx:34-73` |

**当前聚类参数**：`PlacesPage.tsx:114-122`

```typescript
map.addSource('media', {
    type: 'geojson',
    data: mapboxData?.myMediaGeoJson as never,
    cluster: true,
    clusterRadius: 50,           // 仅配置了半径
    // 缺失的关键参数：
    // clusterMaxZoom: 14       // 超过此 zoom 不再聚类（默认 14）
    // clusterMinPoints: 2      // 形成聚类最少点数（默认 2）
    clusterProperties: {
        thumbnail: ['coalesce', ['get', 'thumbnail'], false],
    },
})
```

#### 9.3.1.1 clusterMaxZoom 与 clusterMinPoints 配置硬编码分析

**代码证据：完全硬编码，无运维配置入口**

**前端代码搜索确认**：
```bash
$ grep -rn "clusterMaxZoom\|clusterMinPoints" ui/src/
# 无结果！两个参数从未在代码中出现过
```

**Mapbox GL JS 默认值**（Mapbox GL JS v2.9.1 源码确认）：
```typescript
clusterMaxZoom: 14  // 缩放级别 >14 时停止聚类，直接显示所有点
clusterMinPoints: 2 // 至少需要 2 个点才形成聚类
clusterRadius: 50   // 像素级聚类半径（已配置）
```

**运维可配置性全面排查**：

| 配置层面 | 是否可配置 | 证据 |
|----------|------------|------|
| 前端环境变量 | ❌ 否 | `ui/example.env` 仅定义 `REACT_APP_API_ENDPOINT`，无任何地图相关变量 |
| 前端构建参数 | ❌ 否 | `vite.config.ts` 无地图聚类相关 define |
| 后端环境变量 | ❌ 否 | `api/utils/environment_variables.go` 定义的所有常量中无 `MAPBOX_CLUSTER_*`、`PLACES_*` 相关变量 |
| GraphQL API 返回 | ❌ 否 | `media_geo_json.graphql` 仅返回 `myMediaGeoJson` 和 `mapboxToken`，无聚类参数 |
| 用户设置页面 | ❌ 否 | `UserPreferences.tsx` 无地图聚类设置 |
| 管理后台配置 | ❌ 否 | 无管理后台功能 |
| 数据库配置表 | ❌ 否 | GORM AutoMigrate 仅自动建表，无 `settings` 或 `config` 表存储聚类参数 |

**后端环境变量完整清单**：`environment_variables.go:16-47`

```go
// General
EnvDevelopmentMode, EnvServeUI, EnvUIPath, EnvMediaCachePath,
EnvFaceRecognitionModelsPath, EnvMediaProbeTimeout

// Network
EnvListenIP, EnvListenPort, EnvAPIEndpoint, EnvUIEndpoint

// Database
EnvDatabaseDriver, EnvMysqlURL, EnvPostgresURL, EnvSqlitePath

// Feature
EnvDisableFaceRecognition, EnvDisableVideoEncoding,
EnvDisableRawProcessing, EnvVideoHardwareAcceleration

// 🔴 缺失：PLACES_CLUSTER_RADIUS, PLACES_CLUSTER_MAX_ZOOM, PLACES_CLUSTER_MIN_POINTS
```

**clusterMaxZoom = 14 对百万级数据的实际影响**：

| 缩放级别 | 地理精度（每像素） | 行为 | 百万级数据体验 |
|----------|-------------------|------|---------------|
| Zoom 0-13 | 78271m → 4.8m | 持续聚类 | 流畅，聚类数从几个到几千个 |
| Zoom 14 | ~2.4m/px | 最后一级聚类 | 可能显示数万个聚类 + 独立点混合 |
| Zoom 15+ | ~1.2m/px | **解聚，显示所有点** | ❌ 灾难！100 万 DOM Marker → 主线程卡死 |

**在高密度区域（如城市中心 1km² 内有 10 万张照片），Zoom 15+ 时浏览器会：**
1. 内存暴涨 2-4GB
2. 主线程阻塞数秒至数十秒
3. 极端情况下标签页直接崩溃（OOM）

**clusterMinPoints = 2 的问题**：
- 两个相邻的点也会形成聚类，放大后又解聚
- 高缩放级别下出现大量"两点聚类"，视觉体验不连贯
- 对百万级数据来说，建议至少设为 5-10，减少无意义的小聚类

**优化建议：暴露配置项给运维**

后端新增环境变量（需改代码）：
```go
EnvPlacesClusterRadius    EnvironmentVariable = "PHOTOVIEW_PLACES_CLUSTER_RADIUS"
EnvPlacesClusterMaxZoom   EnvironmentVariable = "PHOTOVIEW_PLACES_CLUSTER_MAX_ZOOM"
EnvPlacesClusterMinPoints EnvironmentVariable = "PHOTOVIEW_PLACES_CLUSTER_MIN_POINTS"
```

后端新增 GraphQL 查询：
```graphql
extend type Query {
    placesConfig: PlacesConfig!
}

type PlacesConfig {
    clusterRadius: Int!
    clusterMaxZoom: Int!
    clusterMinPoints: Int!
}
```

前端通过 `useQuery(PLACES_CONFIG_QUERY)` 获取配置后传入 `map.addSource()`。

#### 9.3.1.2 frontend/config.ts 暴露的 supercluster 配置：文档缺失与合法性校验缺失

**代码证据：项目中不存在 frontend/config.ts。**

```bash
$ find ui/src -name "config*" -type f
# 无结果
$ grep -rn "PLACES_CLUSTER\|placesConfig\|PlacesConfig" ui/src/
# 无结果（这是上文建议的方案，尚未实现）
```

**当前实际配置方式**：完全硬编码在 `PlacesPage.tsx:114-122`

```typescript
map.addSource('media', {
    type: 'geojson',
    data: mapboxData?.myMediaGeoJson as never,
    cluster: true,
    clusterRadius: 50,           // 硬编码
    // clusterMaxZoom: 14        // 使用 Mapbox GL JS 默认值
    // clusterMinPoints: 2       // 使用 Mapbox GL JS 默认值
    clusterProperties: {
        thumbnail: ['coalesce', ['get', 'thumbnail'], false],
    },
})
```

**若按上一节建议实现 frontend/config.ts，应包含的文档与校验：**

**当前缺失项 A：配置文档（运维侧无法知晓配置项）**

假设存在 `ui/src/config/placesConfig.ts`，应至少包含：

```typescript
/**
 * Places（地图）聚类配置
 *
 * 通过环境变量注入（构建时）或 GraphQL placesConfig 查询（运行时）获取。
 * 所有参数均有默认值，运维可按需调整。
 */
export interface PlacesClusterConfig {
    /**
     * 聚类像素半径（单位：屏幕像素）
     * 范围：10 - 200
     * 默认：50
     * 值越大 → 聚类越激进（更多点被合并），地图越稀疏
     * 值越小 → 聚类越保守（更少点被合并），地图越密集
     * 建议：照片总量 <1万 用 30，1万-10万 用 50，>10万 用 80
     */
    clusterRadius: number;

    /**
     * 最大聚类缩放级别
     * 范围：0 - 24（Mapbox GL JS zoom 范围）
     * 默认：14
     * 超过此缩放级别后，所有聚类解聚为独立点
     * 值越大 → 解聚越晚（高缩放级别仍保持聚类）
     * 警告：设为 20+ 且照片量大时，解聚瞬间可能产生数万 DOM 节点，导致卡顿
     */
    clusterMaxZoom: number;

    /**
     * 形成聚类的最少点数
     * 范围：2 - 100
     * 默认：2
     * 值越大 → 小聚类越少（视觉更连贯），但孤立点更多
     * 建议：照片总量 >10万 时设为 5-10，减少"两点聚类"
     */
    clusterMinPoints: number;
}

export const DEFAULT_PLACES_CLUSTER_CONFIG: PlacesClusterConfig = {
    clusterRadius: 50,
    clusterMaxZoom: 14,
    clusterMinPoints: 2,
};
```

**当前缺失项 B：默认值合法性校验（运维配置错误无任何反馈）**

需在后端环境变量解析 + 前端消费处双重校验：

```typescript
/**
 * 校验并规范化 Places 聚类配置。
 * 配置超出合法范围时，回退到默认值并在控制台告警。
 *
 * @param rawConfig 从 GraphQL/环境变量获取的原始配置
 * @returns 校验后的合法配置
 */
export function validatePlacesClusterConfig(
    rawConfig: Partial<PlacesClusterConfig>,
    logger: { warn: (msg: string) => void } = console
): PlacesClusterConfig {
    const config = { ...DEFAULT_PLACES_CLUSTER_CONFIG, ...rawConfig };

    // 校验 clusterRadius：10 - 200 像素
    if (!Number.isInteger(config.clusterRadius) || config.clusterRadius < 10 || config.clusterRadius > 200) {
        logger.warn(
            `[PlacesConfig] clusterRadius=${config.clusterRadius} 超出范围 [10, 200]，` +
            `回退到默认值 ${DEFAULT_PLACES_CLUSTER_CONFIG.clusterRadius}`
        );
        config.clusterRadius = DEFAULT_PLACES_CLUSTER_CONFIG.clusterRadius;
    }

    // 校验 clusterMaxZoom：0 - 24（Mapbox GL JS 支持的 zoom 范围）
    if (!Number.isInteger(config.clusterMaxZoom) || config.clusterMaxZoom < 0 || config.clusterMaxZoom > 24) {
        logger.warn(
            `[PlacesConfig] clusterMaxZoom=${config.clusterMaxZoom} 超出范围 [0, 24]，` +
            `回退到默认值 ${DEFAULT_PLACES_CLUSTER_CONFIG.clusterMaxZoom}`
        );
        config.clusterMaxZoom = DEFAULT_PLACES_CLUSTER_CONFIG.clusterMaxZoom;
    }

    // 校验 clusterMinPoints：2 - 100（至少需要 2 个点才能叫"聚类"）
    if (!Number.isInteger(config.clusterMinPoints) || config.clusterMinPoints < 2 || config.clusterMinPoints > 100) {
        logger.warn(
            `[PlacesConfig] clusterMinPoints=${config.clusterMinPoints} 超出范围 [2, 100]，` +
            `回退到默认值 ${DEFAULT_PLACES_CLUSTER_CONFIG.clusterMinPoints}`
        );
        config.clusterMinPoints = DEFAULT_PLACES_CLUSTER_CONFIG.clusterMinPoints;
    }

    // 组合规则校验：clusterMaxZoom 不应过低
    if (config.clusterMaxZoom < 5) {
        logger.warn(
            `[PlacesConfig] clusterMaxZoom=${config.clusterMaxZoom} 过低，` +
            `Zoom ${config.clusterMaxZoom + 1}+ 级别将直接显示所有独立点，` +
            `高密度区域可能导致浏览器卡顿`
        );
    }

    // 组合规则校验：clusterMinPoints 不应高于 clusterRadius 能容纳的合理点数
    const estimatedMaxPointsPerCluster = Math.PI * (config.clusterRadius / 2) ** 2 / 100; // 粗略估计
    if (config.clusterMinPoints > estimatedMaxPointsPerCluster) {
        logger.warn(
            `[PlacesConfig] clusterMinPoints=${config.clusterMinPoints} 过高，` +
            `clusterRadius=${config.clusterRadius} 像素范围内预计只能容纳约 ` +
            `${Math.round(estimatedMaxPointsPerCluster)} 个点，可能无法形成任何聚类`
        );
    }

    return config;
}
```

**后端对应的环境变量校验**（`api/utils/environment_variables.go` 中应补充）：

```go
// （需新增）PlacesClusterRadius 返回聚类半径，合法范围 [10, 200]
func PlacesClusterRadius() int {
    val := os.Getenv(EnvPlacesClusterRadius)
    if val == "" {
        return 50
    }
    n, err := strconv.Atoi(val)
    if err != nil || n < 10 || n > 200 {
        log.Warn(nil, "Invalid PHOTOVIEW_PLACES_CLUSTER_RADIUS, using default 50",
            "value", val)
        return 50
    }
    return n
}

// （需新增）PlacesClusterMaxZoom 返回最大聚类缩放级别，合法范围 [0, 24]
func PlacesClusterMaxZoom() int {
    val := os.Getenv(EnvPlacesClusterMaxZoom)
    if val == "" {
        return 14
    }
    n, err := strconv.Atoi(val)
    if err != nil || n < 0 || n > 24 {
        log.Warn(nil, "Invalid PHOTOVIEW_PLACES_CLUSTER_MAX_ZOOM, using default 14",
            "value", val)
        return 14
    }
    return n
}

// （需新增）PlacesClusterMinPoints 返回最小聚类点数，合法范围 [2, 100]
func PlacesClusterMinPoints() int {
    val := os.Getenv(EnvPlacesClusterMinPoints)
    if val == "" {
        return 2
    }
    n, err := strconv.Atoi(val)
    if err != nil || n < 2 || n > 100 {
        log.Warn(nil, "Invalid PHOTOVIEW_PLACES_CLUSTER_MIN_POINTS, using default 2",
            "value", val)
        return 2
    }
    return n
}
```

**运维配置文档（应写入 README 或部署文档）**：

```markdown
## Places 地图聚类配置（可选）

| 环境变量 | 类型 | 默认值 | 范围 | 说明 |
|----------|------|--------|------|------|
| `PHOTOVIEW_PLACES_CLUSTER_RADIUS` | int | 50 | 10-200 | 聚类像素半径，越大合并越激进 |
| `PHOTOVIEW_PLACES_CLUSTER_MAX_ZOOM` | int | 14 | 0-24 | 超过此 zoom 级别解聚为独立点 |
| `PHOTOVIEW_PLACES_CLUSTER_MIN_POINTS` | int | 2 | 2-100 | 形成聚类最少需要的点数 |

### 推荐配置

| 照片规模 | RADIUS | MAX_ZOOM | MIN_POINTS |
|----------|--------|----------|------------|
| < 1 万张 | 30 | 14 | 2 |
| 1 万 - 10 万张 | 50 | 15 | 3 |
| 10 万 - 100 万张 | 80 | 16 | 5 |
| > 100 万张 | 100 | 17 | 8 |

配置错误时会回退到默认值并在服务端日志和浏览器 Console 输出告警。
```

**当前状态总结**：

| 要素 | 当前代码状态 | 建议状态 |
|------|-------------|----------|
| `frontend/config.ts` 文件 | ❌ 不存在 | ✅ 新建，导出接口与默认值 |
| 参数 JSDoc 文档 | ❌ 缺失 | ✅ 每个参数说明含义、范围、建议 |
| 默认值单值合法性校验 | ❌ 缺失 | ✅ `validatePlacesClusterConfig()` 范围检查 |
| 参数组合合理性校验 | ❌ 缺失 | ✅ radius/minPoints 冲突检测 |
| 超范围回退 + 日志告警 | ❌ 缺失 | ✅ 前后端双重 `log.Warn` / `console.warn` |
| 运维部署文档 | ❌ 缺失 | ✅ README/部署手册新增配置章节 |
| 不同规模推荐配置 | ❌ 缺失 | ✅ 配置文档提供推荐值对照表 |

#### 9.3.1.3 maxZoom 等合法但无意义值的校验缺失分析

**代码证据：当前无任何校验代码，上节建议的 `validatePlacesClusterConfig()` 也仅做了范围检查**

上节建议的校验逻辑：
```typescript
if (!Number.isInteger(config.clusterMaxZoom) || config.clusterMaxZoom < 0 || config.clusterMaxZoom > 24) {
    // 类型 + 范围校验
}
```

这只检查了"值是否在 [0, 24] 的整数范围内"，但**没有检查值是否在业务语义上有意义**。

**合法但无意义的值清单**：

| 参数 | 合法但无意义的值 | 行为 | 后果 |
|------|-----------------|------|------|
| `clusterMaxZoom = 0` | ✅ 通过范围校验 | Zoom 1+ 级别就解聚，全球视图下也显示所有独立点 | 百万级数据 → 浏览器 OOM |
| `clusterMaxZoom = 1` | ✅ 通过范围校验 | Zoom 2+ 级别就解聚，大洲级视图显示独立点 | 同上，稍好 |
| `clusterMaxZoom = 24` | ✅ 通过范围校验 | 永远不解聚（Mapbox 最大 zoom 22） | 聚类永远不展开，无法看到单张照片 |
| `clusterRadius = 200` | ✅ 通过范围校验 | 极端聚类，整个屏幕可能只有 1-2 个聚类 | 地图几乎不可用 |
| `clusterMinPoints = 100` | ✅ 通过范围校验 | 几乎不可能形成聚类 | 等于关闭聚类功能 |
| `clusterRadius = 10` + `clusterMinPoints = 100` | ✅ 分别通过 | 10px 半径内找 100 个点 → 不可能 | 无聚类，等同于 `cluster: false` |

**maxZoom = 0 的详细分析**：

Mapbox GL JS 的 supercluster 内部处理 `clusterMaxZoom` 的逻辑：
```javascript
// supercluster 源码简化
if (zoom <= this.options.maxZoom) {
    return this._cluster(points, zoom);  // 执行聚类
} else {
    return points;  // 直接返回原始点
}
```

当 `clusterMaxZoom = 0` 时：
- **Zoom 0**（全球视图，约 156km/像素）：执行聚类 → 显示几个大聚类
- **Zoom 1**（大洲级，约 78km/像素）：直接返回原始点 → 百万 DOM 节点
- 用户任何放大操作都会触发灾难性渲染

当前端接收到 `myMediaGeoJson`（假设 100 万条数据）时：
```
Zoom 0 → supercluster 返回 5-10 个聚类 → 正常
Zoom 1 → supercluster 返回 100 万个原始点 → 创建 100 万个 DOM Marker
         → 每个 ReactDOM.render 独立渲染 → 内存暴涨 → 主线程冻结
```

**应该增加的语义校验**：

在 `validatePlacesClusterConfig()` 的范围校验之后，增加业务语义校验：

```typescript
// 语义校验：clusterMaxZoom 过低（虽然合法但必然导致性能灾难）
if (config.clusterMaxZoom <= 3) {
    logger.warn(
        `[PlacesConfig] clusterMaxZoom=${config.clusterMaxZoom} 过低。` +
        `Zoom ${config.clusterMaxZoom + 1}+ 将显示所有独立点，` +
        `在照片数量较多时（>1000）会导致浏览器严重卡顿或崩溃。` +
        `建议最低设为 10，推荐 14-17。`
    );
    // 不强制回退，但强烈建议调整（运维可能有意为之，如小数据集）
}

// 语义校验：clusterMaxZoom 过高（聚类永远不会展开）
if (config.clusterMaxZoom >= 22) {
    logger.warn(
        `[PlacesConfig] clusterMaxZoom=${config.clusterMaxZoom} 过高。` +
        `Mapbox GL JS 最大支持 Zoom 22，此设置下聚类永远不会解聚，` +
        `用户无法查看单张照片的位置。建议设为 14-17。`
    );
}

// 语义校验：clusterMinPoints 与 clusterRadius 组合不合理
if (config.clusterRadius <= 20 && config.clusterMinPoints >= 20) {
    logger.warn(
        `[PlacesConfig] clusterRadius=${config.clusterRadius} + ` +
        `clusterMinPoints=${config.clusterMinPoints} 组合不合理。` +
        `${config.clusterRadius}px 半径内难以容纳 ${config.clusterMinPoints} 个点，` +
        `实际上等同于关闭聚类。建议降低 clusterMinPoints 或增大 clusterRadius。`
    );
}

// 语义校验：clusterRadius 过大
if (config.clusterRadius >= 150) {
    logger.warn(
        `[PlacesConfig] clusterRadius=${config.clusterRadius} 过大。` +
        `几乎所有点会被合并为极少数聚类，地图将失去地理分辨能力。` +
        `建议不超过 120。`
    );
}
```

**后端对应的环境变量语义校验**（`environment_variables.go` 补充）：

```go
func PlacesClusterMaxZoom() int {
    // ... 范围校验 [0, 24] ...
    if n <= 3 {
        log.Warn(nil, "PHOTOVIEW_PLACES_CLUSTER_MAX_ZOOM is very low, "+
            "zoom levels above this will show all individual points. "+
            "This may cause browser performance issues with large photo libraries.",
            "value", n, "recommended_min", 10)
    }
    if n >= 22 {
        log.Warn(nil, "PHOTOVIEW_PLACES_CLUSTER_MAX_ZOOM is very high, "+
            "clusters will never expand to show individual points. "+
            "Users won't be able to see individual photo locations.",
            "value", n, "recommended_max", 17)
    }
    return n
}
```

**校验分级体系**：

| 校验级别 | 含义 | 违反时行为 | 示例 |
|----------|------|-----------|------|
| L0 类型校验 | 值是否为正确类型 | 强制回退默认值 | `clusterMaxZoom = "abc"` |
| L1 范围校验 | 值是否在有效区间内 | 强制回退默认值 | `clusterMaxZoom = -1` 或 `30` |
| **L2 语义校验** | **值是否在业务上合理** | **仅告警不回退** | `clusterMaxZoom = 0` |
| L3 组合校验 | 多个值之间是否自洽 | 仅告警不回退 | `radius=10 + minPoints=100` |

**当前缺失的就是 L2 和 L3 级校验。** L0/L1 只保证"程序不会崩溃"，但 L2/L3 才保证"程序行为符合预期"。运维设置 `clusterMaxZoom=0` 不会得到任何错误提示，程序正常运行但地图功能实质上不可用。

#### 9.3.2 百万级数据的性能瓶颈分析

**瓶颈 1：数据全量加载到前端内存**

`media_geo_json.go:17-71` 的 Resolver 返回**用户所有**带 GPS 的媒体，无分页、无区域过滤：

```go
func (r *queryResolver) MyMediaGeoJSON(ctx context.Context) (any, error) {
    // SELECT ... WHERE gps_latitude IS NOT NULL（无 LIMIT，无 WHERE 区域过滤）
    // 百万级照片 → 单次响应可能达 50-150MB JSON
}
```

- **网络开销**：100 万点 × 每点 ~100 字节（含缩略图 URL、ID、标题） ≈ **100MB**
- **内存开销**：解析后 JS 对象约 200-300MB，supercluster 构建空间索引额外 ~150MB
- **首屏等待**：仅数据传输 + JSON.parse 在中低端移动设备可能超过 10 秒

**瓶颈 2：querySourceFeatures 全量遍历**

**代码位置**：`mapboxHelperFunctions.tsx:38`

```typescript
const features = map.querySourceFeatures('media')
// 无 filter 参数，无 bounds 参数
// 返回当前 source 中聚类后的所有 feature（可能仍有数十万条）
```

Mapbox GL JS 的 `querySourceFeatures` **不按视口裁剪**，它查询的是整个 GeoJSON source。每次地图 `move` / `moveend` / `sourcedata` 事件触发时，都会：

1. 遍历全部聚类 feature（百万级数据 → 可能有数千个聚类点）
2. 对每个 feature 创建/更新 DOM Marker
3. 对不在屏幕上的 marker 调用 `.remove()`

在拖动地图时 `move` 事件每帧触发 1-2 次，直接导致：
- 主线程阻塞 > 100ms（掉帧，卡顿）
- GC 压力大（频繁创建/删除 DOM 节点）

**对比正确做法**：应该用 `queryRenderedFeatures` + `filter` 只查询视口内要素。

**瓶颈 3：ReactDOM.render 为每个 Marker 创建 React 树**

**代码位置**：`mapboxHelperFunctions.tsx:84-91`

```typescript
function createClusterPopupElement(geojsonProps, { dispatchMarkerMedia }) {
    const el = document.createElement('div')
    ReactDOM.render(
        <MapClusterMarker marker={geojsonProps} dispatchMarkerMedia={dispatchMarkerMedia} />,
        el
    )
    return el
}
```

- 每个 Marker 是独立的 React 渲染根，产生额外的 Fiber 树开销
- `JSON.parse(marker.thumbnail)` 在 `MapClusterMarker.tsx:52` 中对每个 marker 执行一次
- 即使视口内只有 50 个 marker，拖动地图反复创建/销毁也会累积大量开销

**瓶颈 4：缩略图 URL 作为 clusterProperties 传递**

`PlacesPage.tsx:119-121`

```typescript
clusterProperties: {
    thumbnail: ['coalesce', ['get', 'thumbnail'], false],
},
```

`['coalesce', ...]` 表达式在聚类时**随机取一个子节点的 thumbnail**，但 thumbnail 字段是包含 URL、width、height 的 JSON 字符串。每个聚类都要存储这个字符串（~150 字节），在百万级聚类中额外增加数十 MB 内存。

#### 9.3.3 supercluster 的分级聚类策略

supercluster 使用 **基于网格的层次聚类（Hierarchical Grid-based Clustering）**：

```
Zoom 0 (全球):    每个网格 360°×360° → 最少聚类数
Zoom 5 (大洲):    每个网格 ~11°×11°
Zoom 10 (城市):   每个网格 ~0.35°×0.35°
Zoom 14 (街区):   每个网格 ~0.022°×0.022° → 默认 maxZoom，继续放大不再聚类
Zoom 18 (街道):   显示所有独立点
```

每次 zoom 级别变化时，supercluster 会：
1. 从上层聚类向下分裂
2. 或从下层点向上合并
3. 时间复杂度 O(N)，N 为总点数

**百万级数据下 supercluster 本身性能**：
- 构建索引：约 300-800ms（现代桌面 CPU）
- 单帧查询（getClusters）：约 5-20ms
- 内存占用：约 200-400MB（含坐标、属性、索引）

**但真正的瓶颈不在 supercluster，而在：数据全量传输 + querySourceFeatures 全量遍历 + React DOM Marker 渲染**

#### 9.3.4 分级优化建议（按优先级排序）

| 优先级 | 优化手段 | 预期收益 | 改动范围 |
|--------|----------|----------|----------|
| P0 | 后端按视口/缩放级别分块返回数据（矢量瓦片或区域查询） | 首屏时间从 10s → <1s | 后端 Resolver + 前端查询逻辑 |
| P0 | 将 `querySourceFeatures` 改为 `queryRenderedFeatures` | 拖动帧率从 <10fps → 60fps | `mapboxHelperFunctions.tsx:38` 一行改动 |
| P1 | 用 Symbol Layer 替代 HTML Marker（Canvas 渲染） | Marker 数量不再影响帧率 | 重写标记渲染逻辑 |
| P1 | 移除 ReactDOM.render，使用原生 DOM/innerHTML | 减少 React 开销 50%+ | `mapboxHelperFunctions.tsx:84-91` |
| P2 | 配置 `clusterMaxZoom` + `clusterMinPoints` | 高 zoom 级别提前解聚 | `PlacesPage.tsx:114-122` |
| P2 | clusterProperties 只存缩略图 ID，URL 前端拼接 | 节省 30-50MB 内存 | 前后端联动 |
| P3 | 服务端预生成 MVT 矢量瓦片 | 终极方案，支持千万级数据 | 需要后端瓦片服务 |

**P0 级改动示例** — `queryRenderedFeatures` 替代方案：

```typescript
// 优化前：遍历整个数据源的所有 feature
// const features = map.querySourceFeatures('media')

// 优化后：只查询当前视口内渲染的 feature
const features = map.queryRenderedFeatures({
    layers: ['media-points'],  // 只查指定图层
    filter: null,
})
```

#### 9.3.3 GPS 1mm 精度上限的可观测性指标（metrics）缺失分析

**代码证据：项目中无任何 metrics 暴露机制**

```bash
$ grep -rn "prometheus\|metrics\|opentelemetry\|otel" api/go.mod
# 无结果
$ grep -rn "counter\|gauge\|histogram\|metric" api/scanner/
# 无结果（仅有 scanner 的 "scanner" 变量名，非 Prometheus metrics）
```

**Photoview 日志体系现状**：

`api/log/default.go` 仅提供 4 级日志（Debug/Info/Warn/Error），无结构化指标输出：

```go
func Debug(ctx context.Context, msg string, args ...any) { getLogger(ctx).DebugContext(ctx, msg, args...) }
func Info(ctx context.Context, msg string, args ...any)  { getLogger(ctx).InfoContext(ctx, msg, args...) }
func Warn(ctx context.Context, msg string, args ...any)  { getLogger(ctx).WarnContext(ctx, msg, args...) }
func Error(ctx context.Context, msg string, args ...any) { getLogger(ctx).ErrorContext(ctx, msg, args...) }
```

GPS 相关的日志仅有一条，位于 `exif_task.go:25`：
```go
log.Warn(ctx, "SaveEXIF failed", "title", media.Title, "error", err, "path", media.Path)
```

没有关于 GPS 解析成功/失败/精度的任何日志或指标。

**应该暴露的 GPS 可观测性指标**：

| 指标名 | 类型 | 标签 | 含义 |
|--------|------|------|------|
| `photoview_exif_parse_total` | Counter | `status=success/failure/empty` | EXIF 解析总次数（按结果分类） |
| `photoview_exif_gps_parsed_total` | Counter | `status=valid/invalid/missing` | GPS 解析结果分类计数 |
| `photoview_exif_gps_invalid_reason_total` | Counter | `reason=nil_lat/nil_lon/nan_lat/nan_lon/out_of_range` | GPS 无效的具体原因计数 |
| `photoview_exif_gps_precision_bucket` | Histogram | — | GPS 坐标小数位数分布（1-15 位） |
| `photoview_exif_parse_duration_seconds` | Histogram | `file_type=image/video` | EXIF 解析耗时 |
| `photoview_geojson_points_served` | Counter | — | GeoJSON API 返回的总 GPS 点数 |
| `photoview_geojson_response_size_bytes` | Histogram | — | GeoJSON 响应体大小 |

**1mm 精度上限可观测性指标的具体实现**：

GPS 坐标 1mm 对应约 8 位小数（0.00000001° ≈ 0.001m = 1mm）。以下指标可以监控 GPS 精度分布，帮助运维发现数据质量问题：

```go
// 在 exif.go:Parse() 中增加指标记录
func Parse(filepath string) (*models.MediaEXIF, error) {
    // ... 现有解析逻辑 ...

    // 记录 GPS 指标
    if values.GPS.IsValid() {
        gpsParseTotal.WithLabelValues("valid").Inc()

        latStr := strconv.FormatFloat(*values.GPS.GPSLatitude, 'f', -1, 64)
        lonStr := strconv.FormatFloat(*values.GPS.GPSLongitude, 'f', -1, 64)

        // 计算小数位数（精度代理指标）
        latDecimals := countDecimalPlaces(latStr)
        lonDecimals := countDecimalPlaces(lonStr)
        maxDecimals := math.Max(float64(latDecimals), float64(lonDecimals))

        gpsPrecisionBucket.Observe(maxDecimals)
        // maxDecimals ≥ 8 → 毫米级精度（1mm 上限）
        // maxDecimals = 6 → ~11cm 精度（消费级 GPS 典型值）
        // maxDecimals = 4 → ~11m 精度（低精度 GPS）
    } else {
        if values.GPS.GPSLatitude == nil || values.GPS.GPSLongitude == nil {
            gpsParseTotal.WithLabelValues("missing").Inc()
        } else if math.IsNaN(*values.GPS.GPSLatitude) || math.IsNaN(*values.GPS.GPSLongitude) {
            gpsParseTotal.WithLabelValues("invalid").Inc()
            gpsInvalidReason.WithLabelValues("nan").Inc()
        } else {
            gpsParseTotal.WithLabelValues("invalid").Inc()
            gpsInvalidReason.WithLabelValues("out_of_range").Inc()
        }
    }

    return &ret, nil
}

func countDecimalPlaces(s string) int {
    parts := strings.Split(s, ".")
    if len(parts) != 2 {
        return 0
    }
    trimmed := strings.TrimRight(parts[1], "0")
    if len(trimmed) == 0 {
        return 0
    }
    return len(trimmed)
}
```

**GPS 精度分布直方图的桶位设计**：

```go
gpsPrecisionBucket = prometheus.NewHistogram(prometheus.HistogramOpts{
    Name:    "photoview_exif_gps_precision_decimals",
    Help:    "Distribution of GPS coordinate decimal places (proxy for precision)",
    Buckets: []float64{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 12, 15},
    // 0 = 整数度（~111km）
    // 4 = ~11m（低精度 GPS）
    // 6 = ~11cm（消费级 GPS 极限）
    // 8 = ~1mm（RTK / 高精度）
    // 10+ = 亚毫米级（大地测量 / 异常值）
})
```

**运维利用精度指标的场景**：

| 场景 | 观测手段 | 告警条件 |
|------|----------|----------|
| 大量照片 GPS 缺失 | `gps_parsed_total{status="missing"}` 急升 | 缺失率 > 90% |
| exiftool 版本升级后 GPS 解析率骤降 | `gps_parsed_total{status="valid"}` 对比前后 | 下降 > 20% |
| 导入了含虚假 GPS 的照片（0,0 坐标） | `gps_invalid_reason{reason="out_of_range"}` 或精度为 0 | 任何增长 |
| 高精度 GPS 数据被截断 | `gps_precision_decimals` 分布右移或截断 | 8+ 位桶占比下降 |
| GPS 精度异常高（伪造数据） | `gps_precision_decimals` 10+ 位桶异常增长 | 占比 > 5% |

**当前代码中最接近"可观测性"的代码**：

`values.go:17-35` 的 `GPS.IsValid()` 是唯一对 GPS 数据质量进行判断的代码，但它的结果仅用于"是否写入数据库"，**不产生任何日志或指标**：

```go
func (gps GPS) IsValid() bool {
    if gps.GPSLongitude == nil || gps.GPSLatitude == nil { return false }
    if math.IsNaN(*gps.GPSLatitude) { return false }
    if math.IsNaN(*gps.GPSLongitude) { return false }
    if math.Abs(*gps.GPSLatitude) > 90 || math.Abs(*gps.GPSLongitude) > 180 { return false }
    return true
}
```

这段验证逻辑丢弃了重要的诊断信息：**为什么 GPS 无效**（是 nil？是 NaN？还是超范围？），运维无法从日志中得知。

**最小改动方案**（不引入 Prometheus 依赖）：

在 `exif.go:88-91` 中增加结构化日志，仅用现有的 `log` 包：

```go
if values.GPS.IsValid() {
    ret.GPSLatitude = values.GPS.GPSLatitude
    ret.GPSLongitude = values.GPS.GPSLongitude
    log.Debug(ctx, "GPS parsed successfully",
        "latitude", *values.GPS.GPSLatitude,
        "longitude", *values.GPS.GPSLongitude,
        "path", filepath)
} else {
    switch {
    case values.GPS.GPSLatitude == nil && values.GPS.GPSLongitude == nil:
        log.Debug(ctx, "GPS data missing", "path", filepath)
    case values.GPS.GPSLatitude == nil || values.GPS.GPSLongitude == nil:
        log.Debug(ctx, "GPS data partial (one coordinate nil)", "path", filepath)
    case math.IsNaN(*values.GPS.GPSLatitude) || math.IsNaN(*values.GPS.GPSLongitude):
        log.Warn(ctx, "GPS data contains NaN", "path", filepath)
    case math.Abs(*values.GPS.GPSLatitude) > 90:
        log.Warn(ctx, "GPS latitude out of range", "latitude", *values.GPS.GPSLatitude, "path", filepath)
    case math.Abs(*values.GPS.GPSLongitude) > 180:
        log.Warn(ctx, "GPS longitude out of range", "longitude", *values.GPS.GPSLongitude, "path", filepath)
    }
}
```

**总结**：Photoview 当前无任何 GPS 可观测性指标暴露，也没有 Prometheus/OpenTelemetry 依赖。GPS 解析结果（成功/失败/精度）完全不可观测。建议至少在 `exif.go` 中增加结构化日志（最小改动），有条件时引入 Prometheus metrics 暴露 GPS 精度分布直方图和解析计数器。

---

## 十、代码引用速查

| 层级 | 关键文件 | 行号 |
|------|----------|------|
| exiftool 封装 | `api/scanner/externaltools/exiftool/exiftool.go` | 14-260 |
| GPS 验证 | `api/scanner/externaltools/exiftool/values.go` | 10-43 |
| EXIF 解析入口 | `api/scanner/externaltools/exif/exif.go` | 45-94 |
| EXIF 扫描任务 | `api/scanner/scanner_tasks/exif_task.go` | 31-74 |
| MediaEXIF 模型 | `api/graphql/models/media_exif.go` | 8-70 |
| GeoJSON Resolver | `api/graphql/resolvers/media_geo_json.go` | 17-71 |
| 地图 Hook | `ui/src/components/mapbox/MapboxMap.tsx` | 27-81 |
| 标记渲染 | `ui/src/components/mapbox/mapboxHelperFunctions.tsx` | 22-93 |
| PlacesPage | `ui/src/Pages/PlacesPage/PlacesPage.tsx` | 31-138 |
| 服务启动 exiftool 初始化 | `api/server.go` | 56-60 |
| exiftool New() 查找二进制 | `api/scanner/externaltools/exiftool/exiftool.go` | 34-38 |
| ffprobe 视频元数据读取 | `api/scanner/scanner_tasks/processing_tasks/process_video_task.go` | 206-216 |
| GPS 精度测试（7 位小数） | `api/scanner/externaltools/exiftool/exiftool_test.go` | 302-308 |
| GeoJSON 坐标组装 | `api/graphql/resolvers/media_geo_json.util.go` | 54-60 |
| querySourceFeatures 全量查询 | `ui/src/components/mapbox/mapboxHelperFunctions.tsx` | 38 |
| 聚类参数配置 | `ui/src/Pages/PlacesPage/PlacesPage.tsx` | 114-122 |
| ReactDOM.render 创建 Marker | `ui/src/components/mapbox/mapboxHelperFunctions.tsx` | 84-91 |
| 无效 GPS 数据迁移 | `api/database/migrations/exif_invalid_gps.go` | 10-24 |
| 聚类展开（getClusterLeaves） | `ui/src/Pages/PlacesPage/MapPresentMarker.tsx` | 46 |
| exiftool JSON 解析（字段名 1:1 匹配） | `api/scanner/externaltools/exiftool/exiftool.go` | 158-172 |
| GPS 结构体字段定义 | `api/scanner/externaltools/exiftool/values.go` | 11-13 |
| GPS 格式化输出（9 位小数精度） | `api/scanner/externaltools/exiftool/values.go` | 42 |
| 环境变量定义清单 | `api/utils/environment_variables.go` | 16-47 |
| 前端环境变量配置 | `ui/example.env` | 1 |
| PhotoMeta 完整字段定义 | `api/scanner/externaltools/exiftool/values.go` | 139-151 |
| TimeAll 时间字段定义 | `api/scanner/externaltools/exiftool/values.go` | 45-61 |
| exiftool 结构体到 MediaEXIF 映射 | `api/scanner/externaltools/exif/exif.go` | 64-91 |
| 视频 ffprobe 元数据提取（8个已覆盖字段 | `api/scanner/scanner_tasks/video_metadata_task.go` | 66-75 |
| json.Decode 字段匹配（无 HasPrefix） | `api/scanner/externaltools/exiftool/exiftool.go` | 167 |
| GPS IsValid 验证（无指标输出） | `api/scanner/externaltools/exiftool/values.go` | 17-35 |
| EXIF 保存日志（唯一 GPS 相关日志） | `api/scanner/scanner_tasks/exif_task.go` | 25 |
| 日志包定义（仅 4 级，无 metrics） | `api/log/default.go` | 1-25 |
| 环境变量校验示例（MediaProbeTimeout） | `api/utils/environment_variables.go` | 87-95 |
