# qgis-errors-projections — Methods Reference

## QgsCoordinateReferenceSystem

### Constructors

```python
QgsCoordinateReferenceSystem()
# Creates an invalid CRS. ALWAYS check isValid() after creation.

QgsCoordinateReferenceSystem(definition: str)
# Creates CRS from definition string with prefix.
# Supported prefixes: "EPSG:", "POSTGIS:", "INTERNAL:", "PROJ:", "WKT:"
# Example: QgsCoordinateReferenceSystem("EPSG:4326")
```

### Validation Methods

| Method | Return Type | Description |
|--------|------------|-------------|
| `isValid()` | `bool` | Returns `True` if CRS was successfully created. ALWAYS call after construction. |
| `isGeographic()` | `bool` | Returns `True` if CRS uses angular units (degrees). `False` for projected CRS (meters, feet). |

### Identity Methods

| Method | Return Type | Description |
|--------|------------|-------------|
| `authid()` | `str` | Authority identifier (e.g., `"EPSG:4326"`). Returns empty string if no authority. |
| `description()` | `str` | Human-readable name (e.g., `"WGS 84"`). |
| `postgisSrid()` | `int` | PostGIS SRID value (e.g., `4326`). |
| `srsid()` | `int` | QGIS internal SRS ID. NOT the same as EPSG code. |

### Representation Methods

| Method | Return Type | Description |
|--------|------------|-------------|
| `toProj()` | `str` | Full Proj pipeline string. |
| `toWkt(variant)` | `str` | WKT representation. Use `Qgis.CrsWktVariant.Wkt2_2019` for modern WKT. |
| `projectionAcronym()` | `str` | Projection type (e.g., `"longlat"`, `"utm"`, `"tmerc"`). |
| `ellipsoidAcronym()` | `str` | Ellipsoid identifier (e.g., `"EPSG:7030"` for WGS 84). |

### Unit Methods

| Method | Return Type | Description |
|--------|------------|-------------|
| `mapUnits()` | `Qgis.DistanceUnit` | Distance unit for this CRS. |

### Static Factory Methods

| Method | Return Type | Description |
|--------|------------|-------------|
| `fromEpsgId(id)` | `QgsCoordinateReferenceSystem` | Create from EPSG integer. |
| `fromProj(proj)` | `QgsCoordinateReferenceSystem` | Create from Proj string. |
| `fromWkt(wkt)` | `QgsCoordinateReferenceSystem` | Create from WKT string. |

### Deprecated Methods (NEVER Use)

| Deprecated Method | Replacement |
|-------------------|-------------|
| `createFromSrid(srid)` | `QgsCoordinateReferenceSystem("EPSG:{srid}")` |
| `createFromEpsg(epsg)` | `QgsCoordinateReferenceSystem("EPSG:{epsg}")` |
| `createFromId(id)` | Use constructor with prefix string |

---

## QgsCoordinateTransform

### Constructors

```python
QgsCoordinateTransform()
# Creates an invalid/empty transform.

QgsCoordinateTransform(source: QgsCoordinateReferenceSystem,
                       destination: QgsCoordinateReferenceSystem,
                       context: QgsCoordinateTransformContext)
# CORRECT — with transform context for datum handling.

QgsCoordinateTransform(source: QgsCoordinateReferenceSystem,
                       destination: QgsCoordinateReferenceSystem,
                       project: QgsProject)
# CORRECT — extracts context from project automatically.

QgsCoordinateTransform(source: QgsCoordinateReferenceSystem,
                       destination: QgsCoordinateReferenceSystem)
# DEPRECATED — NEVER use. Missing context degrades datum transformation accuracy.
```

### Transform Methods

| Method | Parameters | Return Type | Description |
|--------|-----------|------------|-------------|
| `transform(point)` | `QgsPointXY` | `QgsPointXY` | Forward transform (source to dest). |
| `transform(point, direction)` | `QgsPointXY, TransformDirection` | `QgsPointXY` | Transform with explicit direction. |
| `transform(x, y)` | `float, float` | `QgsPointXY` | Transform from raw coordinates. |
| `transformBoundingBox(rect)` | `QgsRectangle` | `QgsRectangle` | Transform a bounding box. |

### Direction Enum

| Value | Description |
|-------|-------------|
| `QgsCoordinateTransform.ForwardTransform` | Source CRS to destination CRS. |
| `QgsCoordinateTransform.ReverseTransform` | Destination CRS to source CRS. |

### Validation Methods

| Method | Return Type | Description |
|--------|------------|-------------|
| `isValid()` | `bool` | Returns `True` if both source and dest CRS are valid. |
| `isShortCircuited()` | `bool` | Returns `True` if source == dest (no actual transform needed). |
| `sourceCrs()` | `QgsCoordinateReferenceSystem` | The source CRS. |
| `destinationCrs()` | `QgsCoordinateReferenceSystem` | The destination CRS. |

---

## QgsCoordinateTransformContext

### Constructor

```python
QgsCoordinateTransformContext()
# Creates an empty context. Prefer QgsProject.instance().transformContext() instead.
```

### Methods

| Method | Parameters | Description |
|--------|-----------|-------------|
| `addCoordinateOperation(src, dest, operation)` | `CRS, CRS, str` | Add a specific Proj operation for a CRS pair. |
| `removeCoordinateOperation(src, dest)` | `CRS, CRS` | Remove a specific operation. |
| `hasTransform(src, dest)` | `CRS, CRS` | Check if a specific transform exists. |
| `calculateCoordinateOperation(src, dest)` | `CRS, CRS` | Get the Proj operation string for a CRS pair. |

---

## QgsGeometry Transform

### In-Place Geometry Transform

```python
geometry.transform(transform: QgsCoordinateTransform) -> Qgis.GeometryOperationResult
# Transforms geometry coordinates in-place.
# Returns Qgis.GeometryOperationResult.Success on success.
```

---

## QgsDistanceArea (CRS-Aware Measurements)

### Setup Methods

| Method | Parameters | Description |
|--------|-----------|-------------|
| `setSourceCrs(crs, context)` | `CRS, TransformContext` | Set source CRS for calculations. |
| `setEllipsoid(ellipsoid)` | `str` | Set ellipsoid (e.g., `"WGS84"`, `"GRS80"`). |

### Measurement Methods

| Method | Parameters | Return Type | Description |
|--------|-----------|------------|-------------|
| `measureLine(p1, p2)` | `QgsPointXY, QgsPointXY` | `float` | Distance in meters. |
| `measureLine(points)` | `list[QgsPointXY]` | `float` | Polyline length in meters. |
| `measureArea(geometry)` | `QgsGeometry` | `float` | Area in square meters. |
| `convertLengthMeasurement(length, unit)` | `float, DistanceUnit` | `float` | Convert length units. |
| `convertAreaMeasurement(area, unit)` | `float, AreaUnit` | `float` | Convert area units. |

---

## QgsProject CRS Methods

| Method | Description |
|--------|-------------|
| `QgsProject.instance().crs()` | Get the project CRS. |
| `QgsProject.instance().setCrs(crs)` | Set the project CRS (controls OTF reprojection target). |
| `QgsProject.instance().transformContext()` | Get the project's datum transform context. ALWAYS use this for transforms. |
| `QgsProject.instance().ellipsoid()` | Get the project ellipsoid string. |

---

## Layer CRS Methods

| Method | Description |
|--------|-------------|
| `layer.crs()` | Get the layer's CRS. |
| `layer.setCrs(crs)` | Set the layer's CRS metadata. NEVER use for reprojection — this only changes the label. |
| `layer.extent()` | Get the layer extent in native CRS coordinates. Use to diagnose wrong CRS assignment. |
