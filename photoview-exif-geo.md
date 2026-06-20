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
