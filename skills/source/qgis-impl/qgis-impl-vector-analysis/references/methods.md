# qgis-impl-vector-analysis — Methods Reference

## QgsGeometry Operations

### Spatial Predicates

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `intersects` | `intersects(QgsGeometry) -> bool` | `bool` | True if geometries share any portion of space |
| `contains` | `contains(QgsGeometry) -> bool` | `bool` | True if geometry completely contains other |
| `within` | `within(QgsGeometry) -> bool` | `bool` | True if geometry is completely within other |
| `touches` | `touches(QgsGeometry) -> bool` | `bool` | True if geometries touch but do not overlap |
| `overlaps` | `overlaps(QgsGeometry) -> bool` | `bool` | True if geometries overlap but neither contains the other |
| `crosses` | `crosses(QgsGeometry) -> bool` | `bool` | True if geometries cross each other |
| `disjoint` | `disjoint(QgsGeometry) -> bool` | `bool` | True if geometries share no space |
| `equals` | `equals(QgsGeometry) -> bool` | `bool` | True if geometries are topologically equal |
| `isGeosValid` | `isGeosValid() -> bool` | `bool` | True if geometry is valid per OGC standards |
| `isNull` | `isNull() -> bool` | `bool` | True if geometry is NULL (empty) |
| `isEmpty` | `isEmpty() -> bool` | `bool` | True if geometry contains no coordinates |

### Geometry Manipulation

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `buffer` | `buffer(distance: float, segments: int) -> QgsGeometry` | `QgsGeometry` | Create buffer around geometry |
| `intersection` | `intersection(QgsGeometry) -> QgsGeometry` | `QgsGeometry` | Geometric intersection |
| `combine` | `combine(QgsGeometry) -> QgsGeometry` | `QgsGeometry` | Geometric union |
| `difference` | `difference(QgsGeometry) -> QgsGeometry` | `QgsGeometry` | Geometric difference |
| `symDifference` | `symDifference(QgsGeometry) -> QgsGeometry` | `QgsGeometry` | Symmetrical difference |
| `centroid` | `centroid() -> QgsGeometry` | `QgsGeometry` | Centroid point |
| `convexHull` | `convexHull() -> QgsGeometry` | `QgsGeometry` | Convex hull |
| `boundingBox` | `boundingBox() -> QgsRectangle` | `QgsRectangle` | Bounding box |
| `simplify` | `simplify(tolerance: float) -> QgsGeometry` | `QgsGeometry` | Simplify geometry |
| `densifyByCount` | `densifyByCount(extraNodesPerSegment: int) -> QgsGeometry` | `QgsGeometry` | Add vertices |
| `makeValid` | `makeValid() -> QgsGeometry` | `QgsGeometry` | Repair invalid geometry |

### Geometry Measurements

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `area` | `area() -> float` | `float` | Area in CRS units (for projected CRS) |
| `length` | `length() -> float` | `float` | Length/perimeter in CRS units |
| `distance` | `distance(QgsGeometry) -> float` | `float` | Minimum distance to other geometry |
| `hausdorffDistance` | `hausdorffDistance(QgsGeometry) -> float` | `float` | Hausdorff distance |

### Geometry Conversion

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `asWkt` | `asWkt(precision: int = 17) -> str` | `str` | WKT representation |
| `asJson` | `asJson(precision: int = 17) -> str` | `str` | GeoJSON representation |
| `asWkb` | `asWkb() -> QByteArray` | `QByteArray` | WKB representation |
| `asPoint` | `asPoint() -> QgsPointXY` | `QgsPointXY` | Extract point coordinates |
| `asPolyline` | `asPolyline() -> list[QgsPointXY]` | `list` | Extract line vertices |
| `asPolygon` | `asPolygon() -> list[list[QgsPointXY]]` | `list` | Extract polygon rings |
| `asMultiPoint` | `asMultiPoint() -> list[QgsPointXY]` | `list` | Extract multipoint coordinates |
| `asMultiPolyline` | `asMultiPolyline() -> list[list[QgsPointXY]]` | `list` | Extract multiline vertices |
| `asMultiPolygon` | `asMultiPolygon() -> list[list[list[QgsPointXY]]]` | `list` | Extract multipolygon rings |

### Static Constructors

| Method | Signature | Returns |
|--------|-----------|---------|
| `fromWkt` | `QgsGeometry.fromWkt(wkt: str) -> QgsGeometry` | `QgsGeometry` |
| `fromPointXY` | `QgsGeometry.fromPointXY(QgsPointXY) -> QgsGeometry` | `QgsGeometry` |
| `fromPolylineXY` | `QgsGeometry.fromPolylineXY(list[QgsPointXY]) -> QgsGeometry` | `QgsGeometry` |
| `fromPolygonXY` | `QgsGeometry.fromPolygonXY(list[list[QgsPointXY]]) -> QgsGeometry` | `QgsGeometry` |
| `fromRect` | `QgsGeometry.fromRect(QgsRectangle) -> QgsGeometry` | `QgsGeometry` |
| `fromMultiPointXY` | `QgsGeometry.fromMultiPointXY(list[QgsPointXY]) -> QgsGeometry` | `QgsGeometry` |

---

## QgsFeatureRequest

| Method | Signature | Description |
|--------|-----------|-------------|
| `setFilterRect` | `setFilterRect(QgsRectangle) -> QgsFeatureRequest` | Filter by bounding rectangle |
| `setFilterExpression` | `setFilterExpression(str) -> QgsFeatureRequest` | Filter by expression string |
| `setFilterFid` | `setFilterFid(int) -> QgsFeatureRequest` | Filter by single feature ID |
| `setFilterFids` | `setFilterFids(set[int]) -> QgsFeatureRequest` | Filter by set of feature IDs |
| `setFlags` | `setFlags(QgsFeatureRequest.Flags) -> QgsFeatureRequest` | Set request flags |
| `setLimit` | `setLimit(int) -> QgsFeatureRequest` | Limit number of returned features |
| `setSubsetOfAttributes` | `setSubsetOfAttributes(list[int]) -> QgsFeatureRequest` | Load only specified attributes |
| `setNoAttributes` | `setNoAttributes() -> QgsFeatureRequest` | Load geometry only, no attributes |
| `setDestinationCrs` | `setDestinationCrs(QgsCoordinateReferenceSystem, QgsCoordinateTransformContext) -> QgsFeatureRequest` | Transform features to target CRS |
| `combineFilterExpression` | `combineFilterExpression(str) -> QgsFeatureRequest` | Add AND expression filter |

### Flags

| Flag | Value | Description |
|------|-------|-------------|
| `ExactIntersect` | `QgsFeatureRequest.ExactIntersect` | Exact geometry test (not just bounding box) |
| `NoGeometry` | `QgsFeatureRequest.NoGeometry` | Do not fetch geometry |

---

## QgsSpatialIndex

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| constructor | `QgsSpatialIndex(QgsFeatureIterator)` | `QgsSpatialIndex` | Build index from features |
| `intersects` | `intersects(QgsRectangle) -> list[int]` | `list[int]` | Feature IDs intersecting rectangle |
| `nearestNeighbor` | `nearestNeighbor(QgsPointXY, neighbors: int) -> list[int]` | `list[int]` | N nearest feature IDs |
| `addFeature` | `addFeature(QgsFeature) -> bool` | `bool` | Add feature to index |
| `deleteFeature` | `deleteFeature(QgsFeature) -> bool` | `bool` | Remove feature from index |

---

## QgsVectorFileWriter

### Static Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `writeAsVectorFormatV3` | `writeAsVectorFormatV3(layer, fileName, transformContext, options) -> tuple[error, errorMessage]` | Write entire layer to file |
| `create` | `create(fileName, fields, geometryType, srs, transformContext, options) -> QgsVectorFileWriter` | Create writer for feature-by-feature output |
| `supportedFiltersAndFormats` | `supportedFiltersAndFormats() -> list` | List supported output formats |

### SaveVectorOptions

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `driverName` | `str` | `"GPKG"` | Output driver name |
| `fileEncoding` | `str` | `"UTF-8"` | File encoding |
| `layerName` | `str` | `""` | Layer name (for multi-layer formats like GPKG) |
| `actionOnExistingFile` | `int` | `0` | 0=CreateOrOverwrite, 1=CreateNewFile, 2=AppendToLayerNoNewFields, 3=AppendToLayerAddFields |
| `datasourceOptions` | `list[str]` | `[]` | Driver-specific datasource options |
| `layerOptions` | `list[str]` | `[]` | Driver-specific layer options |
| `filterExtent` | `QgsRectangle` | `None` | Spatial filter for output |

### Common Driver Names

| Driver | Extension | Description |
|--------|-----------|-------------|
| `GPKG` | `.gpkg` | GeoPackage (ALWAYS preferred for new projects) |
| `ESRI Shapefile` | `.shp` | Shapefile (legacy, 10-char field name limit) |
| `GeoJSON` | `.geojson` | GeoJSON (web-friendly) |
| `FlatGeobuf` | `.fgb` | FlatGeobuf (fast streaming) |
| `CSV` | `.csv` | CSV (attributes only, or with geometry columns) |
| `KML` | `.kml` | Keyhole Markup Language |

---

## Processing Algorithm Parameters

### native:buffer

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | `QgsVectorLayer` | Input layer |
| `DISTANCE` | `float` | Buffer distance in layer CRS units |
| `SEGMENTS` | `int` | Number of segments for circular approximation (default: 5) |
| `END_CAP_STYLE` | `int` | 0=Round, 1=Flat, 2=Square |
| `JOIN_STYLE` | `int` | 0=Round, 1=Miter, 2=Bevel |
| `MITER_LIMIT` | `float` | Miter limit (default: 2) |
| `DISSOLVE` | `bool` | Dissolve result into single feature |
| `OUTPUT` | `str` | Output path or `'memory:'` |

### native:intersection

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | `QgsVectorLayer` | Input layer |
| `OVERLAY` | `QgsVectorLayer` | Overlay layer |
| `INPUT_FIELDS` | `list[str]` | Fields to keep from input (empty = all) |
| `OVERLAY_FIELDS` | `list[str]` | Fields to keep from overlay (empty = all) |
| `OVERLAY_FIELDS_PREFIX` | `str` | Prefix for overlay field names |
| `OUTPUT` | `str` | Output path or `'memory:'` |

### native:union

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | `QgsVectorLayer` | Input layer |
| `OVERLAY` | `QgsVectorLayer` | Overlay layer |
| `OVERLAY_FIELDS_PREFIX` | `str` | Prefix for overlay field names |
| `OUTPUT` | `str` | Output path or `'memory:'` |

### native:difference

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | `QgsVectorLayer` | Input layer |
| `OVERLAY` | `QgsVectorLayer` | Overlay layer |
| `OUTPUT` | `str` | Output path or `'memory:'` |

### native:clip

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | `QgsVectorLayer` | Input layer |
| `OVERLAY` | `QgsVectorLayer` | Clip layer |
| `OUTPUT` | `str` | Output path or `'memory:'` |

### native:symmetricaldifference

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | `QgsVectorLayer` | Input layer |
| `OVERLAY` | `QgsVectorLayer` | Overlay layer |
| `OVERLAY_FIELDS_PREFIX` | `str` | Prefix for overlay field names |
| `OUTPUT` | `str` | Output path or `'memory:'` |

### native:dissolve

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | `QgsVectorLayer` | Input layer |
| `FIELD` | `list[str]` | Field(s) to dissolve by (empty = dissolve all) |
| `SEPARATE_DISJOINT` | `bool` | Keep disjoint features separate |
| `OUTPUT` | `str` | Output path or `'memory:'` |

### native:joinattributestable

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | `QgsVectorLayer` | Input layer |
| `FIELD` | `str` | Join field in input layer |
| `INPUT_2` | `QgsVectorLayer` | Second input (table to join) |
| `FIELD_2` | `str` | Join field in second layer |
| `FIELDS_TO_COPY` | `list[str]` | Fields to copy (empty = all) |
| `METHOD` | `int` | 0=one-to-many, 1=first match only |
| `DISCARD_NONMATCHING` | `bool` | Discard non-matching features |
| `PREFIX` | `str` | Prefix for joined field names |
| `OUTPUT` | `str` | Output path or `'memory:'` |

### native:joinattributesbylocation

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | `QgsVectorLayer` | Target layer |
| `PREDICATE` | `list[int]` | 0=intersects, 1=contains, 2=equals, 3=touches, 4=overlaps, 5=within, 6=crosses |
| `JOIN` | `QgsVectorLayer` | Join layer |
| `JOIN_FIELDS` | `list[str]` | Fields to copy (empty = all) |
| `METHOD` | `int` | 0=one-to-many, 1=one-to-first, 2=one-to-largest-overlap |
| `DISCARD_NONMATCHING` | `bool` | Discard non-matching features |
| `PREFIX` | `str` | Prefix for joined field names |
| `OUTPUT` | `str` | Output path or `'memory:'` |

### native:fieldcalculator

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | `QgsVectorLayer` | Input layer |
| `FIELD_NAME` | `str` | New field name |
| `FIELD_TYPE` | `int` | 0=Float, 1=Integer, 2=String, 3=Date |
| `FIELD_LENGTH` | `int` | Field length |
| `FIELD_PRECISION` | `int` | Decimal precision |
| `FORMULA` | `str` | QGIS expression |
| `OUTPUT` | `str` | Output path or `'memory:'` |

### native:extractbyexpression

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | `QgsVectorLayer` | Input layer |
| `EXPRESSION` | `str` | QGIS expression |
| `OUTPUT` | `str` | Matching features output |
| `FAIL_OUTPUT` | `str` | Non-matching features output (optional) |

### native:extractbylocation

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | `QgsVectorLayer` | Input layer |
| `PREDICATE` | `list[int]` | Spatial predicates (same as joinattributesbylocation) |
| `INTERSECT` | `QgsVectorLayer` | Reference layer |
| `OUTPUT` | `str` | Output path or `'memory:'` |
