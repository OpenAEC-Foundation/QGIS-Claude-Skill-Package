# QGIS Georeferencing Python API — Supplementary Research

> Targeted verification of Python bindings for QGIS georeferencing classes.
> Date: 2026-03-20 | Status: Complete

---

## 1. Overview

This document supplements the integration vooronderzoek by verifying which QGIS georeferencing classes have actual Python bindings in `qgis.analysis`. Research was conducted by fetching official PyQGIS API documentation and QGIS source code test files.

### Classes Available in `qgis.analysis` (Georeferencing Section)

The following classes are exposed in the `qgis.analysis` Python module as of QGIS 3.40+:

| Class | Available in Python | Since |
|-------|-------------------|-------|
| QgsGcpPoint | **Yes** [CONFIRMED] | 3.26 |
| QgsGcpTransformerInterface | **Yes** [CONFIRMED] | 3.20 |
| QgsGcpGeometryTransformer | **Yes** [CONFIRMED] | 3.18 |
| QgsVectorWarper | **Yes** [CONFIRMED] | 3.x |
| QgsVectorWarperTask | **Yes** [CONFIRMED] | 3.x |
| QgsImageWarper | **No** [CONFIRMED] | N/A |

**Source:** https://qgis.org/pyqgis/3.40/analysis/index.html

**Key finding:** `QgsImageWarper` does NOT have Python bindings. It exists in the C++ API but is not exposed to Python. Raster georeferencing must use GDAL Python bindings directly.

---

## 2. QgsGcpPoint — Python API [CONFIRMED]

**Source:** https://qgis.org/pyqgis/master/analysis/QgsGcpPoint.html

### Constructors

```python
# Primary constructor
QgsGcpPoint(
    sourcePoint: QgsPointXY,
    destinationPoint: QgsPointXY,
    destinationPointCrs: QgsCoordinateReferenceSystem,
    enabled: bool = True
)

# Copy constructor
QgsGcpPoint(a0: QgsGcpPoint)
```

### Enum: PointType

```python
QgsGcpPoint.PointType.Source       # = 0
QgsGcpPoint.PointType.Destination  # = 1
```

### Methods

| Method | Signature | Return |
|--------|-----------|--------|
| `sourcePoint()` | `() -> QgsPointXY` | Source coordinates (pixel or layer CRS) |
| `setSourcePoint()` | `(point: QgsPointXY) -> None` | Set source coordinates |
| `destinationPoint()` | `() -> QgsPointXY` | Destination coordinates |
| `setDestinationPoint()` | `(point: QgsPointXY) -> None` | Set destination coordinates |
| `destinationPointCrs()` | `() -> QgsCoordinateReferenceSystem` | CRS of destination point |
| `setDestinationPointCrs()` | `(crs: QgsCoordinateReferenceSystem) -> None` | Set destination CRS |
| `isEnabled()` | `() -> bool` | Whether point is enabled |
| `setEnabled()` | `(enabled: bool) -> None` | Enable/disable point |
| `transformedDestinationPoint()` | `(targetCrs: QgsCoordinateReferenceSystem, context: QgsCoordinateTransformContext) -> QgsPointXY` | Destination point transformed to target CRS |

All methods are fully available in Python. The `transformedDestinationPoint()` method is useful when GCPs are defined in a different CRS than the target output.

---

## 3. QgsGcpTransformerInterface — Python API [CONFIRMED]

**Source:** https://qgis.org/pyqgis/master/analysis/QgsGcpTransformerInterface.html

### TransformMethod Enum [CONFIRMED]

All transformation types are available in Python:

```python
QgsGcpTransformerInterface.TransformMethod.Linear            # = 0
QgsGcpTransformerInterface.TransformMethod.Helmert           # = 1
QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1  # = 2
QgsGcpTransformerInterface.TransformMethod.PolynomialOrder2  # = 3
QgsGcpTransformerInterface.TransformMethod.PolynomialOrder3  # = 4
QgsGcpTransformerInterface.TransformMethod.ThinPlateSpline   # = 5
QgsGcpTransformerInterface.TransformMethod.Projective        # = 6
QgsGcpTransformerInterface.TransformMethod.InvalidTransform  # = 65535
```

### Factory Methods [CONFIRMED]

```python
# Create empty transformer by method type
QgsGcpTransformerInterface.create(
    method: TransformMethod
) -> QgsGcpTransformerInterface | None

# Create and initialize with GCP coordinates
QgsGcpTransformerInterface.createFromParameters(
    method: TransformMethod,
    sourceCoordinates: Iterable[QgsPointXY],
    destinationCoordinates: Iterable[QgsPointXY]
) -> QgsGcpTransformerInterface | None  # Returns None if fit fails
```

### Key Methods

| Method | Signature | Return |
|--------|-----------|--------|
| `transform()` | `(x: float, y: float, inverseTransform: bool = False) -> bool` | Success status |
| `method()` | `() -> TransformMethod` | Current transform method |
| `minimumGcpCount()` | `() -> int` | Minimum GCPs required |
| `updateParametersFromGcps()` | `(sourceCoordinates: Iterable[QgsPointXY], destinationCoordinates: Iterable[QgsPointXY], invertYAxis: bool = False) -> bool` | Fit success |
| `clone()` | `() -> QgsGcpTransformerInterface \| None` | Deep copy |
| `methodToString()` | `(method: TransformMethod) -> str` | Translated name |

**Important note on `transform()`:** The method signature shows it returns `bool`, but the actual transformed x,y coordinates are returned as modified parameters. In Python this means the method returns a tuple `(bool, float, float)` — the success flag and the transformed x and y values. This is a common SIP binding pattern where C++ output parameters become return tuple elements.

---

## 4. QgsGcpGeometryTransformer — Python API [CONFIRMED]

**Source:** https://qgis.org/pyqgis/master/analysis/QgsGcpGeometryTransformer.html

This class is a higher-level wrapper that transforms entire QgsGeometry objects using GCPs. It extends `QgsAbstractGeometryTransformer`.

### Constructors

```python
# From existing transformer
QgsGcpGeometryTransformer(
    gcpTransformer: QgsGcpTransformerInterface | None
)

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
| `transform()` | `(geometry: QgsGeometry, feedback: QgsFeedback \| None = None) -> (QgsGeometry, bool)` | Transformed geometry + success |
| `gcpTransformer()` | `() -> QgsGcpTransformerInterface \| None` | Underlying transformer |
| `setGcpTransformer()` | `(transformer: QgsGcpTransformerInterface \| None) -> None` | Replace transformer |

---

## 5. QgsVectorWarper — Python API [CONFIRMED]

**Source:** https://qgis.org/pyqgis/master/analysis/QgsVectorWarper.html

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
| `transformFeatures()` | `(iterator: QgsFeatureIterator, sink: QgsFeatureSink \| None, context: QgsCoordinateTransformContext, feedback: QgsFeedback \| None = None) -> bool` | Success |
| `error()` | `() -> str` | Last error message |

**Important:** There is NO `warpLayer()` method. The correct method is `transformFeatures()`, which takes a feature iterator and a feature sink. This contradicts some earlier assumptions — the API works at the feature level, not the layer level.

---

## 6. QgsImageWarper — NOT Available in Python [CONFIRMED]

QgsImageWarper does NOT appear in the `qgis.analysis` Python module index. The class exists in the C++ API but has no SIP bindings, meaning it cannot be imported or used from Python.

**Source:** https://qgis.org/pyqgis/3.40/analysis/index.html (class absent from listing)

### Alternative: GDAL Python Bindings for Raster Georeferencing

For raster georeferencing, use GDAL directly:

```python
from osgeo import gdal, osr

# Step 1: Create GCPs
gcps = [
    gdal.GCP(target_x, target_y, 0, pixel_x, pixel_y)
    for (target_x, target_y), (pixel_x, pixel_y) in zip(world_coords, pixel_coords)
]

# Step 2: Open source and set GCPs
src_ds = gdal.OpenShared(input_path, gdal.GA_ReadOnly)
gcp_srs = osr.SpatialReference()
gcp_srs.ImportFromEPSG(epsg_code)
src_ds.SetGCPs(gcps, gcp_srs.ExportToWkt())

# Step 3: Create warped VRT for auto dimensions
dst_srs = osr.SpatialReference()
dst_srs.ImportFromEPSG(epsg_code)
tmp_ds = gdal.AutoCreateWarpedVRT(src_ds, None, dst_srs.ExportToWkt(),
                                   gdal.GRA_Bilinear, 0.125)

# Step 4: Create output and reproject
dst_ds = gdal.GetDriverByName('GTiff').Create(
    output_path, tmp_ds.RasterXSize, tmp_ds.RasterYSize, src_ds.RasterCount)
dst_ds.SetProjection(dst_srs.ExportToWkt())
dst_ds.SetGeoTransform(tmp_ds.GetGeoTransform())
gdal.ReprojectImage(src_ds, dst_ds, None, None, gdal.GRA_Bilinear)
dst_ds = None
```

**Source:** https://gist.github.com/valgur/f24312ddc1aaaee649c8

---

## 7. Working Python Example — Vector Georeferencing [CONFIRMED]

This example is extracted from the official QGIS test suite (`tests/src/python/test_qgsvectorwarper.py`):

```python
from qgis.analysis import (
    QgsGcpPoint,
    QgsGcpTransformerInterface,
    QgsVectorWarper,
)
from qgis.core import (
    QgsCoordinateReferenceSystem,
    QgsFeature,
    QgsFeatureStore,
    QgsGeometry,
    QgsPointXY,
    QgsProject,
    QgsVectorLayer,
)

# 1. Create source layer with features
source_layer = QgsVectorLayer(
    "Point?field=fldtxt:string&field=fldint:integer", "source", "memory"
)
pr = source_layer.dataProvider()

f = QgsFeature()
f.setAttributes(["test", 123])
f.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(100, 200)))
pr.addFeatures([f])

# 2. Define Ground Control Points
gcps = [
    QgsGcpPoint(
        QgsPointXY(90, 210),       # source (pixel) coordinates
        QgsPointXY(8, 20),         # destination (world) coordinates
        QgsCoordinateReferenceSystem("EPSG:4283"),
        True,                       # enabled
    ),
    QgsGcpPoint(
        QgsPointXY(210, 190),
        QgsPointXY(20.5, 20),
        QgsCoordinateReferenceSystem("EPSG:4283"),
        True,
    ),
    QgsGcpPoint(
        QgsPointXY(350, 220),
        QgsPointXY(30, 21),
        QgsCoordinateReferenceSystem("EPSG:4283"),
        True,
    ),
    QgsGcpPoint(
        QgsPointXY(390, 290),
        QgsPointXY(39, 28),
        QgsCoordinateReferenceSystem("EPSG:4283"),
        True,
    ),
]

# 3. Create warper with transformation method
warper = QgsVectorWarper(
    QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1,
    gcps,
    QgsCoordinateReferenceSystem("EPSG:4283"),
)

# 4. Transform features into a sink
sink = QgsFeatureStore()
success = warper.transformFeatures(
    source_layer.getFeatures(),
    sink,
    QgsProject.instance().transformContext(),
)

# 5. Access results
if success:
    for feature in sink.features():
        print(feature.geometry().asWkt(), feature.attributes())
else:
    print("Error:", warper.error())
```

**Source:** https://github.com/qgis/QGIS/blob/master/tests/src/python/test_qgsvectorwarper.py

---

## 8. Minimum GCP Requirements Per Transform Method

| Transform Method | Min GCPs | Notes |
|-----------------|----------|-------|
| Linear | 2 | Simple affine |
| Helmert | 2 | Rotation + scale + translation |
| PolynomialOrder1 | 3 | First-order polynomial (affine) |
| PolynomialOrder2 | 6 | Second-order polynomial |
| PolynomialOrder3 | 10 | Third-order polynomial |
| ThinPlateSpline | 1 | Flexible, no minimum in theory |
| Projective | 4 | Perspective transformation |

These minimums can be verified at runtime via `QgsGcpTransformerInterface.minimumGcpCount()`.

---

## 9. Corrections to Previous Research

### Items Previously Marked [VERIFY]

1. **`QgsGcpPoint` Python API** — [CONFIRMED] Fully available. All methods (sourcePoint, destinationPoint, isEnabled, setEnabled, transformedDestinationPoint) have Python bindings.

2. **`QgsGcpTransformerInterface.TransformMethod` enum** — [CONFIRMED] All 7 transform types + InvalidTransform are available in Python.

3. **`QgsGcpTransformerInterface.createFromParameters()`** — [CONFIRMED] Available as a static factory method. Returns None on failure.

4. **`QgsVectorWarper.warpLayer()`** — [CORRECTED] This method does NOT exist. The correct method is `transformFeatures(iterator, sink, context, feedback)`. The skill must use this instead.

5. **`QgsImageWarper`** — [CONFIRMED NOT AVAILABLE] No Python bindings exist. Use GDAL Python bindings directly for raster georeferencing.

6. **`QgsGcpGeometryTransformer`** — [CONFIRMED] Available in Python since 3.18. Provides a higher-level API that transforms QgsGeometry objects directly, returning `(QgsGeometry, bool)`.

---

## 10. Recommended Approach Per Use Case

### Vector Layer Georeferencing
Use `QgsVectorWarper` with `QgsGcpPoint` objects. This is the cleanest API.

### Individual Geometry Transformation
Use `QgsGcpGeometryTransformer` — takes raw coordinate lists, transforms geometries directly.

### Point Coordinate Transformation
Use `QgsGcpTransformerInterface.createFromParameters()` + `transform(x, y)` for individual point transformations.

### Raster Image Georeferencing
Use GDAL Python bindings (`gdal.GCP`, `gdal.AutoCreateWarpedVRT`, `gdal.ReprojectImage`). QgsImageWarper has no Python bindings.

### Alternative Raster Approach via Processing
Use `processing.run("gdal:translate", ...)` to embed GCPs in a raster, then `processing.run("gdal:warpreproject", ...)` to apply the transformation. However, the `gdal:translate` processing algorithm does not expose GCP parameters directly in its parameter dictionary — GCPs must be passed via the `EXTRA` parameter as command-line flags (e.g., `-gcp pixel line easting northing`). [UNCONFIRMED — needs testing]

---

## 11. Sources

| Source | URL | Status |
|--------|-----|--------|
| QgsGcpPoint PyQGIS docs | https://qgis.org/pyqgis/master/analysis/QgsGcpPoint.html | Fetched |
| QgsGcpTransformerInterface PyQGIS docs | https://qgis.org/pyqgis/master/analysis/QgsGcpTransformerInterface.html | Fetched |
| QgsVectorWarper PyQGIS docs | https://qgis.org/pyqgis/master/analysis/QgsVectorWarper.html | Fetched |
| QgsGcpGeometryTransformer PyQGIS docs | https://qgis.org/pyqgis/master/analysis/QgsGcpGeometryTransformer.html | Fetched |
| qgis.analysis module index (3.40) | https://qgis.org/pyqgis/3.40/analysis/index.html | Fetched |
| QGIS test: test_qgsvectorwarper.py | https://github.com/qgis/QGIS/blob/master/tests/src/python/test_qgsvectorwarper.py | Fetched (raw) |
| GDAL warp with GCPs gist | https://gist.github.com/valgur/f24312ddc1aaaee649c8 | Fetched |
| GDAL raster conversion docs | https://docs.qgis.org/3.44/en/docs/user_manual/processing_algs/gdal/rasterconversion.html | Fetched |
