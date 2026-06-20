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

## 九、代码引用速查

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
