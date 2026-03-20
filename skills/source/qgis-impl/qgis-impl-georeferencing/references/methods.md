# qgis-impl-georeferencing — Methods Reference

## QgsGcpPoint (since QGIS 3.26)

```python
from qgis.analysis import QgsGcpPoint
```

### Constructors

```python
QgsGcpPoint(
    sourcePoint: QgsPointXY,
    destinationPoint: QgsPointXY,
    destinationPointCrs: QgsCoordinateReferenceSystem,
    enabled: bool = True
)

QgsGcpPoint(other: QgsGcpPoint)  # Copy constructor
```

### PointType Enum

```python
QgsGcpPoint.PointType.Source       # 0
QgsGcpPoint.PointType.Destination  # 1
```

### Methods

| Method | Signature | Return |
|--------|-----------|--------|
| `sourcePoint()` | `() -> QgsPointXY` | Source coordinates |
| `setSourcePoint()` | `(point: QgsPointXY) -> None` | Set source coordinates |
| `destinationPoint()` | `() -> QgsPointXY` | Destination coordinates |
| `setDestinationPoint()` | `(point: QgsPointXY) -> None` | Set destination coordinates |
| `destinationPointCrs()` | `() -> QgsCoordinateReferenceSystem` | CRS of destination point |
| `setDestinationPointCrs()` | `(crs: QgsCoordinateReferenceSystem) -> None` | Set destination CRS |
| `isEnabled()` | `() -> bool` | Whether point is enabled |
| `setEnabled()` | `(enabled: bool) -> None` | Enable/disable point |
| `transformedDestinationPoint()` | `(targetCrs: QgsCoordinateReferenceSystem, context: QgsCoordinateTransformContext) -> QgsPointXY` | Destination reprojected to target CRS |

---

## QgsGcpTransformerInterface (since QGIS 3.20)

```python
from qgis.analysis import QgsGcpTransformerInterface
```

### TransformMethod Enum

```python
QgsGcpTransformerInterface.TransformMethod.Linear            # 0
QgsGcpTransformerInterface.TransformMethod.Helmert           # 1
QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1  # 2
QgsGcpTransformerInterface.TransformMethod.PolynomialOrder2  # 3
QgsGcpTransformerInterface.TransformMethod.PolynomialOrder3  # 4
QgsGcpTransformerInterface.TransformMethod.ThinPlateSpline   # 5
QgsGcpTransformerInterface.TransformMethod.Projective        # 6
QgsGcpTransformerInterface.TransformMethod.InvalidTransform  # 65535
```

### Factory Methods

| Method | Signature | Return |
|--------|-----------|--------|
| `create()` | `(method: TransformMethod) -> QgsGcpTransformerInterface \| None` | Empty transformer |
| `createFromParameters()` | `(method: TransformMethod, sourceCoordinates: Iterable[QgsPointXY], destinationCoordinates: Iterable[QgsPointXY]) -> QgsGcpTransformerInterface \| None` | Fitted transformer or None on failure |

### Instance Methods

| Method | Signature | Return |
|--------|-----------|--------|
| `transform()` | `(x: float, y: float, inverseTransform: bool = False) -> tuple[bool, float, float]` | (success, transformed_x, transformed_y) |
| `method()` | `() -> TransformMethod` | Current transform method |
| `minimumGcpCount()` | `() -> int` | Minimum GCPs required |
| `updateParametersFromGcps()` | `(sourceCoordinates: Iterable[QgsPointXY], destinationCoordinates: Iterable[QgsPointXY], invertYAxis: bool = False) -> bool` | True if fit succeeded |
| `clone()` | `() -> QgsGcpTransformerInterface \| None` | Deep copy |
| `methodToString()` | `(method: TransformMethod) -> str` | Human-readable name (static method) |

**CRITICAL:** In Python, `transform()` returns a 3-tuple `(bool, float, float)` due to SIP bindings converting C++ output parameters into return values. The C++ signature shows `bool` return only, but Python receives `(success, x, y)`.

---

## QgsGcpGeometryTransformer (since QGIS 3.18)

```python
from qgis.analysis import QgsGcpGeometryTransformer
```

### Constructors

```python
# From existing transformer
QgsGcpGeometryTransformer(gcpTransformer: QgsGcpTransformerInterface | None)

# Direct initialization with coordinates
QgsGcpGeometryTransformer(
    method: QgsGcpTransformerInterface.TransformMethod,
    sourceCoordinates: Iterable[QgsPointXY],
    destinationCoordinates: Iterable[QgsPointXY]
)
```

### Methods

| Method | Signature | Return |
|--------|-----------|--------|
| `transform()` | `(geometry: QgsGeometry, feedback: QgsFeedback \| None = None) -> tuple[QgsGeometry, bool]` | (transformed geometry, success) |
| `gcpTransformer()` | `() -> QgsGcpTransformerInterface \| None` | Underlying transformer |
| `setGcpTransformer()` | `(transformer: QgsGcpTransformerInterface \| None) -> None` | Replace transformer |

---

## QgsVectorWarper (since QGIS 3.26)

```python
from qgis.analysis import QgsVectorWarper
```

### Constructor

```python
QgsVectorWarper(
    method: QgsGcpTransformerInterface.TransformMethod,
    points: Iterable[QgsGcpPoint],
    destinationCrs: QgsCoordinateReferenceSystem
)
```

### Methods

| Method | Signature | Return |
|--------|-----------|--------|
| `transformFeatures()` | `(iterator: QgsFeatureIterator, sink: QgsFeatureSink \| None, context: QgsCoordinateTransformContext, feedback: QgsFeedback \| None = None) -> bool` | True on success |
| `error()` | `() -> str` | Last error message |

**CRITICAL:** There is NO `warpLayer()` method on QgsVectorWarper. The ONLY method for warping is `transformFeatures()`.

---

## GDAL Python Bindings (for Raster Georeferencing)

```python
from osgeo import gdal, osr
```

### Key Functions

| Function | Signature | Purpose |
|----------|-----------|---------|
| `gdal.GCP()` | `(x, y, z, pixel, line)` | Create a ground control point |
| `dataset.SetGCPs()` | `(gcps: list, projection: str)` | Assign GCPs to raster |
| `gdal.AutoCreateWarpedVRT()` | `(src_ds, src_wkt, dst_wkt, resampling, max_error) -> Dataset` | Create warped VRT |
| `gdal.ReprojectImage()` | `(src_ds, dst_ds, src_wkt, dst_wkt, resampling)` | Reproject raster data |
| `gdal.GetDriverByName()` | `(name: str) -> Driver` | Get output driver |
| `driver.Create()` | `(path, xsize, ysize, bands, type) -> Dataset` | Create output raster |
| `driver.CreateCopy()` | `(path, src_ds, options=[]) -> Dataset` | Copy with options |

### Resampling Methods

| Constant | Value | Method |
|----------|-------|--------|
| `gdal.GRA_NearestNeighbour` | 0 | Nearest neighbour |
| `gdal.GRA_Bilinear` | 1 | Bilinear interpolation |
| `gdal.GRA_Cubic` | 2 | Cubic convolution |
| `gdal.GRA_CubicSpline` | 3 | Cubic spline |
| `gdal.GRA_Lanczos` | 4 | Lanczos windowed sinc |
