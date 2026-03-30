---
name: qgis-impl-georeferencing
description: >
  Use when georeferencing raster images or vector data using ground control points.
  Prevents accuracy loss from insufficient GCPs and wrong transformation type selection.
  Covers GCP management, 7 transformation types, QgsVectorWarper, residual computation, and accuracy assessment.
  Keywords: georeferencing, GCP, ground control point, QgsGcpPoint, transformation, Helmert, polynomial, thin plate spline, world file, align image to map, scanned map.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-impl-georeferencing

## Quick Reference

### Core Classes

| Class | Module | Since | Purpose |
|-------|--------|-------|---------|
| `QgsGcpPoint` | `qgis.analysis` | 3.26 | Ground control point with source/destination coords |
| `QgsGcpTransformerInterface` | `qgis.analysis` | 3.20 | Creates and applies coordinate transformations |
| `QgsGcpGeometryTransformer` | `qgis.analysis` | 3.18 | Transforms QgsGeometry objects using GCPs |
| `QgsVectorWarper` | `qgis.analysis` | 3.26 | Warps vector features via `transformFeatures()` |

### Transformation Types

| Method | Enum | Min GCPs | Use Case |
|--------|------|----------|----------|
| Linear | `Linear` | 2 | Simple offset and scale, no rotation needed |
| Helmert | `Helmert` | 2 | Rotation + scale + translation, rigid body |
| Polynomial 1 | `PolynomialOrder1` | 3 | General affine transformation |
| Polynomial 2 | `PolynomialOrder2` | 6 | Moderate local distortion correction |
| Polynomial 3 | `PolynomialOrder3` | 10 | Heavy local distortion correction |
| Projective | `Projective` | 4 | Perspective correction (scanned oblique images) |
| Thin Plate Spline | `ThinPlateSpline` | 1 | Exact interpolation through all GCPs |

### Enum Access

```python
from qgis.analysis import QgsGcpTransformerInterface
TM = QgsGcpTransformerInterface.TransformMethod

TM.Linear            # 0
TM.Helmert           # 1
TM.PolynomialOrder1  # 2
TM.PolynomialOrder2  # 3
TM.PolynomialOrder3  # 4
TM.ThinPlateSpline   # 5
TM.Projective        # 6
```

---

## Critical Warnings

**NEVER** call `warpLayer()` on `QgsVectorWarper` -- this method does NOT exist. ALWAYS use `transformFeatures()` with a feature iterator and feature sink.

**NEVER** use `QgsImageWarper` from Python -- it has NO Python bindings. ALWAYS use GDAL Python bindings (`osgeo.gdal`) for raster georeferencing.

**NEVER** use fewer GCPs than the minimum required for the chosen transformation type. The `createFromParameters()` factory returns `None` when GCP count is insufficient.

**NEVER** ignore residuals after fitting a transformation. ALWAYS compute and check residuals to verify accuracy before applying the transformation to data.

**NEVER** assume all GCPs share the same destination CRS -- each `QgsGcpPoint` stores its own `destinationPointCrs`. ALWAYS set the CRS explicitly for every GCP.

**ALWAYS** check the return value of `transformFeatures()` and `createFromParameters()` -- both return a success indicator. On failure, call `warper.error()` for diagnostics.

---

## Decision Tree: Choosing a Transformation Type

```
START: How many GCPs do you have?
|
+-- 1-2 GCPs
|   +-- Need rotation? --> Helmert (min 2)
|   +-- No rotation?   --> Linear (min 2)
|   +-- Only 1 GCP?    --> ThinPlateSpline (exact fit, 1 point = translation only)
|
+-- 3-5 GCPs
|   +-- General purpose         --> PolynomialOrder1 (min 3)
|   +-- Perspective correction  --> Projective (min 4)
|
+-- 6-9 GCPs
|   +-- Moderate distortion --> PolynomialOrder2 (min 6)
|   +-- Simple distortion   --> PolynomialOrder1 (extra GCPs improve accuracy)
|
+-- 10+ GCPs
|   +-- Heavy distortion    --> PolynomialOrder3 (min 10)
|   +-- Exact GCP fit needed --> ThinPlateSpline (interpolates through all points)
|   +-- General purpose      --> PolynomialOrder1 (overdetermined = more robust)
|
ACCURACY PRIORITY:
  - Highest global accuracy: PolynomialOrder1 with many well-distributed GCPs
  - Exact fit at GCP locations: ThinPlateSpline (but may oscillate between points)
  - Rigid transformation (no distortion): Helmert
```

---

## Essential Patterns

### Pattern 1: Create Ground Control Points

```python
from qgis.analysis import QgsGcpPoint
from qgis.core import QgsPointXY, QgsCoordinateReferenceSystem

# Each GCP maps a source coordinate to a destination coordinate
gcp = QgsGcpPoint(
    QgsPointXY(100, 200),                           # source (pixel/local coords)
    QgsPointXY(15.5, 47.1),                         # destination (map coords)
    QgsCoordinateReferenceSystem("EPSG:4326"),       # destination CRS
    True                                              # enabled
)

# Access and modify
src = gcp.sourcePoint()          # QgsPointXY
dst = gcp.destinationPoint()     # QgsPointXY
crs = gcp.destinationPointCrs()  # QgsCoordinateReferenceSystem

gcp.setEnabled(False)  # Disable without deleting
```

### Pattern 2: Create a Transformer and Transform Points

```python
from qgis.analysis import QgsGcpTransformerInterface
from qgis.core import QgsPointXY

source_pts = [QgsPointXY(0, 0), QgsPointXY(100, 0), QgsPointXY(100, 100)]
dest_pts = [QgsPointXY(15.0, 47.0), QgsPointXY(16.0, 47.0), QgsPointXY(16.0, 48.0)]

# Create and fit transformer in one step
transformer = QgsGcpTransformerInterface.createFromParameters(
    QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1,
    source_pts,
    dest_pts
)

if transformer is None:
    raise RuntimeError("Transformation fit failed -- check GCP count and distribution")

# Transform a single point -- returns (success, x, y) tuple in Python
success, tx, ty = transformer.transform(50.0, 50.0, False)  # False = forward
if success:
    print(f"Transformed: ({tx}, {ty})")
```

### Pattern 3: Vector Georeferencing with QgsVectorWarper

```python
from qgis.analysis import (
    QgsGcpPoint, QgsGcpTransformerInterface, QgsVectorWarper
)
from qgis.core import (
    QgsCoordinateReferenceSystem, QgsFeatureStore,
    QgsPointXY, QgsProject, QgsVectorLayer
)

# Define GCPs
dest_crs = QgsCoordinateReferenceSystem("EPSG:4283")
gcps = [
    QgsGcpPoint(QgsPointXY(90, 210), QgsPointXY(8, 20), dest_crs, True),
    QgsGcpPoint(QgsPointXY(210, 190), QgsPointXY(20.5, 20), dest_crs, True),
    QgsGcpPoint(QgsPointXY(350, 220), QgsPointXY(30, 21), dest_crs, True),
    QgsGcpPoint(QgsPointXY(390, 290), QgsPointXY(39, 28), dest_crs, True),
]

# Create warper
warper = QgsVectorWarper(
    QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1,
    gcps,
    dest_crs
)

# Transform features into a feature store
source_layer = QgsVectorLayer("Point?field=name:string", "source", "memory")
sink = QgsFeatureStore()
success = warper.transformFeatures(
    source_layer.getFeatures(),
    sink,
    QgsProject.instance().transformContext()
)

if not success:
    raise RuntimeError(f"Warp failed: {warper.error()}")

for feature in sink.features():
    print(feature.geometry().asWkt(), feature.attributes())
```

### Pattern 4: Geometry Transformation with QgsGcpGeometryTransformer

```python
from qgis.analysis import QgsGcpGeometryTransformer, QgsGcpTransformerInterface
from qgis.core import QgsGeometry, QgsPointXY

source_pts = [QgsPointXY(0, 0), QgsPointXY(100, 0), QgsPointXY(100, 100)]
dest_pts = [QgsPointXY(15.0, 47.0), QgsPointXY(16.0, 47.0), QgsPointXY(16.0, 48.0)]

# Create geometry transformer directly from coordinates
geo_transformer = QgsGcpGeometryTransformer(
    QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1,
    source_pts,
    dest_pts
)

# Transform any QgsGeometry
geom = QgsGeometry.fromPointXY(QgsPointXY(50, 50))
transformed_geom, ok = geo_transformer.transform(geom)

if ok:
    print(f"Transformed: {transformed_geom.asWkt()}")
```

### Pattern 5: Raster Georeferencing with GDAL

```python
from osgeo import gdal, osr

# Step 1: Create GDAL GCPs (target_x, target_y, z, pixel_x, pixel_y)
gcps = [
    gdal.GCP(15.0, 47.0, 0, 0, 0),
    gdal.GCP(16.0, 47.0, 0, 100, 0),
    gdal.GCP(16.0, 48.0, 0, 100, 100),
]

# Step 2: Open source raster and assign GCPs
src_ds = gdal.OpenShared("/path/to/unreferenced.tif", gdal.GA_ReadOnly)
gcp_srs = osr.SpatialReference()
gcp_srs.ImportFromEPSG(4326)
src_ds.SetGCPs(gcps, gcp_srs.ExportToWkt())

# Step 3: Create warped VRT (auto-calculates output dimensions)
dst_srs = osr.SpatialReference()
dst_srs.ImportFromEPSG(4326)
tmp_ds = gdal.AutoCreateWarpedVRT(
    src_ds, None, dst_srs.ExportToWkt(), gdal.GRA_Bilinear, 0.125
)

# Step 4: Write to GeoTIFF
dst_ds = gdal.GetDriverByName("GTiff").Create(
    "/path/to/georeferenced.tif",
    tmp_ds.RasterXSize,
    tmp_ds.RasterYSize,
    src_ds.RasterCount,
)
dst_ds.SetProjection(dst_srs.ExportToWkt())
dst_ds.SetGeoTransform(tmp_ds.GetGeoTransform())
gdal.ReprojectImage(src_ds, dst_ds, None, None, gdal.GRA_Bilinear)

# Cleanup
dst_ds = None
src_ds = None
```

---

## Common Operations

### Compute Residuals for Accuracy Assessment

```python
from qgis.analysis import QgsGcpTransformerInterface
from qgis.core import QgsPointXY
import math

def compute_residuals(transformer, source_pts, dest_pts):
    """Compute per-GCP residuals and total RMSE."""
    residuals = []
    for src, dst in zip(source_pts, dest_pts):
        success, tx, ty = transformer.transform(src.x(), src.y(), False)
        if not success:
            residuals.append(float("inf"))
            continue
        dx = tx - dst.x()
        dy = ty - dst.y()
        residuals.append(math.sqrt(dx * dx + dy * dy))

    rmse = math.sqrt(sum(r * r for r in residuals) / len(residuals))
    return residuals, rmse

# Usage
source_pts = [QgsPointXY(0, 0), QgsPointXY(100, 0), QgsPointXY(100, 100)]
dest_pts = [QgsPointXY(15.0, 47.0), QgsPointXY(16.0, 47.0), QgsPointXY(16.0, 48.0)]

transformer = QgsGcpTransformerInterface.createFromParameters(
    QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1,
    source_pts, dest_pts
)
residuals, rmse = compute_residuals(transformer, source_pts, dest_pts)
print(f"Per-GCP residuals: {residuals}")
print(f"RMSE: {rmse:.6f}")
```

### Write a World File Manually

```python
def write_world_file(path, pixel_width, rotation_x, rotation_y, pixel_height,
                     upper_left_x, upper_left_y):
    """Write a 6-line world file (.tfw, .jgw, .pgw)."""
    with open(path, "w") as f:
        f.write(f"{pixel_width}\n")    # Line 1: pixel width (x scale)
        f.write(f"{rotation_x}\n")     # Line 2: rotation about y axis
        f.write(f"{rotation_y}\n")     # Line 3: rotation about x axis
        f.write(f"{pixel_height}\n")   # Line 4: pixel height (y scale, negative)
        f.write(f"{upper_left_x}\n")   # Line 5: x coordinate of upper-left center
        f.write(f"{upper_left_y}\n")   # Line 6: y coordinate of upper-left center

# Example: 1m resolution, no rotation, origin at (500000, 5200000)
write_world_file(
    "/path/to/image.tfw",
    1.0, 0.0, 0.0, -1.0, 500000.0, 5200000.0
)
```

### World File Extensions by Image Format

| Image Format | World File Extension |
|-------------|---------------------|
| TIFF (.tif) | .tfw |
| JPEG (.jpg) | .jgw |
| PNG (.png) | .pgw |
| BMP (.bmp) | .bpw |
| GIF (.gif) | .gfw |

### Generate World File via GDAL

```python
from osgeo import gdal

# Create GeoTIFF with world file sidecar
ds = gdal.Open("/path/to/georeferenced.tif", gdal.GA_ReadOnly)
gdal.GetDriverByName("GTiff").CreateCopy(
    "/path/to/output.tif", ds, options=["TFW=YES"]
)
ds = None
```

### Disable Individual GCPs for Leave-One-Out Validation

```python
def leave_one_out_validation(gcps, source_pts, dest_pts, method):
    """Disable each GCP in turn and check its residual."""
    from qgis.analysis import QgsGcpTransformerInterface

    for i in range(len(source_pts)):
        # Build lists without the i-th point
        src_subset = [p for j, p in enumerate(source_pts) if j != i]
        dst_subset = [p for j, p in enumerate(dest_pts) if j != i]

        transformer = QgsGcpTransformerInterface.createFromParameters(
            method, src_subset, dst_subset
        )
        if transformer is None:
            print(f"GCP {i}: fit failed without this point")
            continue

        success, tx, ty = transformer.transform(
            source_pts[i].x(), source_pts[i].y(), False
        )
        if success:
            import math
            dx = tx - dest_pts[i].x()
            dy = ty - dest_pts[i].y()
            residual = math.sqrt(dx * dx + dy * dy)
            print(f"GCP {i}: leave-one-out residual = {residual:.6f}")
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for QgsGcpPoint, QgsGcpTransformerInterface, QgsGcpGeometryTransformer, QgsVectorWarper
- [references/examples.md](references/examples.md) -- Complete working examples for vector and raster georeferencing
- [references/anti-patterns.md](references/anti-patterns.md) -- What NOT to do when georeferencing

### Official Sources

- https://qgis.org/pyqgis/master/analysis/QgsGcpPoint.html
- https://qgis.org/pyqgis/master/analysis/QgsGcpTransformerInterface.html
- https://qgis.org/pyqgis/master/analysis/QgsGcpGeometryTransformer.html
- https://qgis.org/pyqgis/master/analysis/QgsVectorWarper.html
- https://qgis.org/pyqgis/3.40/analysis/index.html
