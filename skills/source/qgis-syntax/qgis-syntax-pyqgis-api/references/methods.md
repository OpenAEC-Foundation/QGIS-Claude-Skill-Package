# qgis-syntax-pyqgis-api — Method Reference

> API signatures for QgsFeature, QgsFeatureRequest, QgsGeometry, QgsSpatialIndex, QgsTask, and related classes.

---

## QgsFeature

Represents a single feature with geometry and attributes.

### Construction

```python
QgsFeature()                    # Empty feature
QgsFeature(fields: QgsFields)   # Feature with field schema (preferred)
QgsFeature(id: int)             # Feature with specific ID
```

### Attribute Access

| Method | Return Type | Description |
|--------|-------------|-------------|
| `id()` | `int` | Feature ID (unique within layer) |
| `geometry()` | `QgsGeometry` | Feature geometry (may be NULL) |
| `attributes()` | `list` | All attribute values as list |
| `attribute(name: str)` | `any` | Attribute value by field name |
| `attribute(index: int)` | `any` | Attribute value by field index |
| `__getitem__(name_or_index)` | `any` | Shorthand: `feature['name']` or `feature[0]` |
| `fields()` | `QgsFields` | Field schema of this feature |
| `isValid()` | `bool` | Whether feature is valid |

### Attribute Modification

| Method | Description |
|--------|-------------|
| `setAttributes(attrs: list)` | Set all attributes at once |
| `setAttribute(index: int, value)` | Set single attribute by index |
| `setAttribute(name: str, value)` | Set single attribute by name |
| `__setitem__(name_or_index, value)` | Shorthand: `feature['name'] = value` |
| `setGeometry(geom: QgsGeometry)` | Set feature geometry |
| `setId(id: int)` | Set feature ID |
| `setFields(fields: QgsFields)` | Set field schema |
| `initAttributes(count: int)` | Initialize attribute array with NULL values |

---

## QgsFeatureRequest

Controls which features are returned and what data is loaded.

### Filter Methods (Chainable)

| Method | Description |
|--------|-------------|
| `setFilterExpression(expr: str)` | Filter by expression string |
| `setFilterRect(rect: QgsRectangle)` | Filter by bounding box |
| `setFilterFid(fid: int)` | Filter to single feature ID |
| `setFilterFids(fids: set)` | Filter to set of feature IDs |
| `setLimit(limit: int)` | Maximum number of features to return |

### Optimization Methods (Chainable)

| Method | Description |
|--------|-------------|
| `setFlags(flags)` | Set request flags (see below) |
| `setSubsetOfAttributes(indices: list)` | Load only specified field indices |
| `setSubsetOfAttributes(names: list, fields: QgsFields)` | Load only specified field names |
| `setNoAttributes()` | Load no attributes at all |
| `addOrderBy(fieldOrExpression: str, ascending: bool)` | Sort results |

### Request Flags

| Flag | Effect |
|------|--------|
| `QgsFeatureRequest.NoGeometry` | Skip geometry loading (faster when only attributes needed) |
| `QgsFeatureRequest.ExactIntersect` | Exact geometry intersection instead of bounding box only |
| `QgsFeatureRequest.SubsetOfAttributes` | Automatically set when using `setSubsetOfAttributes()` |

### Usage Pattern

```python
# Chain multiple filters
request = (QgsFeatureRequest()
    .setFilterExpression('"population" > 10000')
    .setSubsetOfAttributes(['name', 'population'], layer.fields())
    .setLimit(100))

for feature in layer.getFeatures(request):
    print(feature['name'])
```

---

## QgsGeometry

High-level geometry wrapper providing GEOS-based spatial operations.

### Static Construction Methods

| Method | Creates |
|--------|---------|
| `QgsGeometry.fromPointXY(QgsPointXY)` | Point geometry |
| `QgsGeometry.fromPolylineXY([QgsPointXY, ...])` | LineString geometry |
| `QgsGeometry.fromPolygonXY([[QgsPointXY, ...]])` | Polygon geometry (list of rings) |
| `QgsGeometry.fromMultiPointXY([QgsPointXY, ...])` | MultiPoint geometry |
| `QgsGeometry.fromMultiPolylineXY([[QgsPointXY, ...]])` | MultiLineString geometry |
| `QgsGeometry.fromMultiPolygonXY([[[QgsPointXY, ...]]])` | MultiPolygon geometry |
| `QgsGeometry.fromWkt(wkt: str)` | From Well-Known Text |
| `QgsGeometry.fromWkb(wkb: bytes)` | From Well-Known Binary |
| `QgsGeometry.fromPolyline([QgsPoint, ...])` | LineString with Z/M support |

### Extraction Methods

| Method | Returns | Geometry Type |
|--------|---------|---------------|
| `asPoint()` | `QgsPointXY` | Point |
| `asPolyline()` | `list[QgsPointXY]` | LineString |
| `asPolygon()` | `list[list[QgsPointXY]]` | Polygon (list of rings) |
| `asMultiPoint()` | `list[QgsPointXY]` | MultiPoint |
| `asMultiPolyline()` | `list[list[QgsPointXY]]` | MultiLineString |
| `asMultiPolygon()` | `list[list[list[QgsPointXY]]]` | MultiPolygon |

### Spatial Predicates (Return bool)

| Method | True When |
|--------|-----------|
| `contains(other)` | Other is completely inside self |
| `within(other)` | Self is completely inside other |
| `intersects(other)` | Geometries share any space |
| `touches(other)` | Boundaries touch, interiors do not overlap |
| `crosses(other)` | Geometries cross each other |
| `overlaps(other)` | Partial overlap (same dimension geometries) |
| `disjoint(other)` | No shared space whatsoever |
| `equals(other)` | Geometrically identical |

### Spatial Operations (Return QgsGeometry)

| Method | Result |
|--------|--------|
| `buffer(distance, segments)` | Polygon at given distance around geometry |
| `intersection(other)` | Shared area/line between geometries |
| `combine(other)` | Union of geometries |
| `difference(other)` | Part of self not in other |
| `symDifference(other)` | Non-overlapping parts of both |
| `convexHull()` | Smallest convex polygon enclosing geometry |
| `centroid()` | Center point geometry |
| `pointOnSurface()` | Point guaranteed inside geometry |
| `simplify(tolerance)` | Simplified geometry (Douglas-Peucker) |

### Measurement Methods

| Method | Returns | Notes |
|--------|---------|-------|
| `area()` | `float` | Polygon area in layer units (projected CRS only) |
| `length()` | `float` | Line length / perimeter in layer units (projected CRS only) |
| `distance(other)` | `float` | Shortest distance to other geometry |

### Validation and Properties

| Method | Returns | Description |
|--------|---------|-------------|
| `isNull()` | `bool` | True if geometry is NULL |
| `isEmpty()` | `bool` | True if geometry has no coordinates |
| `isGeosValid()` | `bool` | True if geometry passes GEOS validation |
| `validateGeometry()` | `list[QgsGeometry.Error]` | Detailed validation errors |
| `isMultipart()` | `bool` | True if multi-geometry |
| `wkbType()` | `QgsWkbTypes.Type` | Detailed WKB type |
| `type()` | `Qgis.GeometryType` | General type (Point/Line/Polygon) |
| `parts()` | `iterator` | Iterate over geometry parts |

### Import/Export

| Method | Description |
|--------|-------------|
| `asWkt()` | Export as Well-Known Text string |
| `asWkb()` | Export as Well-Known Binary bytes |
| `asJson()` | Export as GeoJSON string |

### Transformation

| Method | Description |
|--------|-------------|
| `transform(QgsCoordinateTransform)` | In-place CRS transformation |
| `transform(QTransform)` | In-place affine transformation |

### Low-Level Access

| Method | Returns | Description |
|--------|---------|-------------|
| `get()` | `QgsAbstractGeometry` | Mutable access to underlying geometry |
| `constGet()` | `QgsAbstractGeometry` | Read-only access to underlying geometry |

---

## QgsSpatialIndex

R-tree based spatial index for fast bounding box queries.

### Construction

```python
QgsSpatialIndex()                           # Empty index
QgsSpatialIndex(featureIterator)            # Bulk load from iterator (fastest)
QgsSpatialIndex(layer.getFeatures())        # Bulk load from layer
```

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `addFeature(feature)` | `bool` | Add single feature to index |
| `addFeatures(features)` | `bool` | Add multiple features |
| `deleteFeature(feature)` | `bool` | Remove feature from index |
| `nearestNeighbor(point, neighbors)` | `list[int]` | Feature IDs of N nearest features |
| `intersects(rectangle)` | `list[int]` | Feature IDs whose bbox intersects rectangle |

### Performance

- Build time: O(n log n) for bulk loading
- Query time: O(log n) for intersection and nearest neighbor
- Returns feature IDs only -- ALWAYS fetch features separately via `QgsFeatureRequest().setFilterFid()`

---

## QgsSpatialIndexKDBush

Specialized index for point data only. Faster than QgsSpatialIndex for point-only datasets.

| Property | Value |
|----------|-------|
| Geometry types | Single points only |
| Mutability | Static (cannot add after creation) |
| Speed | Significantly faster than R-tree for points |
| Feature retrieval | Returns original points directly (no second query) |

---

## QgsTask

Base class for background operations.

### Construction

```python
QgsTask.__init__(self, description: str, flags: QgsTask.Flags)
QgsTask.fromFunction(description, function, on_finished=callback, **kwargs)
```

### Task Flags

| Flag | Description |
|------|-------------|
| `QgsTask.CanCancel` | Task supports cancellation |
| `QgsTask.CancelWithoutPrompt` | Cancel without user confirmation |
| `QgsTask.Hidden` | Task not shown in task manager |

### Methods to Override

| Method | Thread | Must Return | Description |
|--------|--------|-------------|-------------|
| `run()` | Background | `bool` | Main work. Return `True` on success, `False` on failure |
| `finished(result: bool)` | Main | Nothing | Called after `run()` completes. Safe for GUI access |
| `cancel()` | Main | Nothing | Called when task is canceled |

### Methods to Call

| Method | Description |
|--------|-------------|
| `setProgress(percent: float)` | Report progress (0-100) |
| `isCanceled()` | Check if cancellation was requested |
| `setDependentLayers([layers])` | Auto-cancel if layers become unavailable |
| `addSubTask(task, deps, behavior)` | Add dependent sub-task |

### Sub-Task Behaviors

| Behavior | Description |
|----------|-------------|
| `QgsTask.ParentDependsOnSubTask` | Parent waits for sub-task to complete |
| `QgsTask.SubTaskIndependent` | Sub-task runs independently |

---

## QgsTaskManager

Global task scheduler accessed via `QgsApplication.taskManager()`.

| Method | Description |
|--------|-------------|
| `addTask(task)` | Schedule task for execution |
| `activeTasks()` | List of running tasks |
| `count()` | Number of active tasks |

---

## QgsDistanceArea

Ellipsoid-based measurement calculator.

### Setup

```python
d = QgsDistanceArea()
d.setEllipsoid('WGS84')
d.setSourceCrs(crs, QgsProject.instance().transformContext())
```

### Measurement Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `measureArea(geom)` | `float` | Area in square meters |
| `measurePerimeter(geom)` | `float` | Perimeter in meters |
| `measureLine(point1, point2)` | `float` | Distance between points in meters |
| `measureLength(geom)` | `float` | Line length in meters |
| `convertAreaMeasurement(area, unit)` | `float` | Convert area to target unit |
| `convertLengthMeasurement(length, unit)` | `float` | Convert length to target unit |

---

## QgsMessageLog

### Static Methods

```python
QgsMessageLog.logMessage(message: str, tag: str, level: Qgis.MessageLevel)
```

### Qgis.MessageLevel

| Level | Value | Description |
|-------|-------|-------------|
| `Qgis.Info` | 0 | Informational |
| `Qgis.Warning` | 1 | Warning |
| `Qgis.Critical` | 2 | Error/critical |
| `Qgis.Success` | 3 | Success confirmation |

---

## QgsMessageBar

Accessed via `iface.messageBar()`. NEVER use from background threads.

| Method | Description |
|--------|-------------|
| `pushMessage(title, text, level, duration)` | Show message (duration=0 for persistent) |
| `pushWidget(widget, level)` | Show custom widget message |
| `createMessage(title, text)` | Create message widget for customization |
| `clearWidgets()` | Remove all messages |

---

## QgsField

### Construction

```python
from qgis.PyQt.QtCore import QMetaType
from qgis.core import QgsField

QgsField("name", QMetaType.Type.QString)
QgsField("value", QMetaType.Type.Int)
QgsField("amount", QMetaType.Type.Double)
```

### Common QMetaType.Type Values

| Type | Python Equivalent |
|------|-------------------|
| `QMetaType.Type.QString` | `str` |
| `QMetaType.Type.Int` | `int` |
| `QMetaType.Type.Double` | `float` |
| `QMetaType.Type.Bool` | `bool` |
| `QMetaType.Type.QDate` | `datetime.date` |
| `QMetaType.Type.QDateTime` | `datetime.datetime` |
