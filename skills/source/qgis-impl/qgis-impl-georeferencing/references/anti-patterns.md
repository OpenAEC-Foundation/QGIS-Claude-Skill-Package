# qgis-impl-georeferencing — Anti-Patterns

## Anti-Pattern 1: Calling warpLayer() on QgsVectorWarper

### Wrong

```python
warper = QgsVectorWarper(method, gcps, dest_crs)
warper.warpLayer(source_layer, output_path)  # AttributeError -- method does NOT exist
```

### Why It Fails

`QgsVectorWarper` has NO `warpLayer()` method. This method does not exist in the C++ API or the Python bindings. The ONLY method for transforming features is `transformFeatures()`.

### Correct

```python
warper = QgsVectorWarper(method, gcps, dest_crs)
sink = QgsFeatureStore()
success = warper.transformFeatures(
    source_layer.getFeatures(),
    sink,
    QgsProject.instance().transformContext(),
)
```

---

## Anti-Pattern 2: Using QgsImageWarper from Python

### Wrong

```python
from qgis.analysis import QgsImageWarper  # ImportError -- no Python bindings
warper = QgsImageWarper(method, gcps)
warper.warp(input_path, output_path)
```

### Why It Fails

`QgsImageWarper` exists in the C++ API but has NO SIP bindings. It is NOT available in the `qgis.analysis` Python module. Attempting to import it raises `ImportError`.

### Correct

Use GDAL Python bindings for raster georeferencing:

```python
from osgeo import gdal, osr

gcps = [gdal.GCP(map_x, map_y, 0, pixel_x, pixel_y)]
src_ds = gdal.OpenShared(input_path, gdal.GA_ReadOnly)
srs = osr.SpatialReference()
srs.ImportFromEPSG(4326)
src_ds.SetGCPs(gcps, srs.ExportToWkt())
vrt_ds = gdal.AutoCreateWarpedVRT(src_ds, None, srs.ExportToWkt(), gdal.GRA_Bilinear, 0.125)
# ... write output
```

---

## Anti-Pattern 3: Using Too Few GCPs for the Transformation Type

### Wrong

```python
# Only 2 GCPs but requesting PolynomialOrder1 (needs minimum 3)
transformer = QgsGcpTransformerInterface.createFromParameters(
    QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1,
    [QgsPointXY(0, 0), QgsPointXY(100, 0)],
    [QgsPointXY(10.0, 50.0), QgsPointXY(11.0, 50.0)],
)
# transformer is None -- fit failed silently
ok, tx, ty = transformer.transform(50, 50)  # AttributeError: NoneType
```

### Why It Fails

Each transformation type has a strict minimum GCP requirement. `createFromParameters()` returns `None` when the requirement is not met. Calling methods on `None` crashes.

### Correct

```python
method = QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1
transformer = QgsGcpTransformerInterface.createFromParameters(
    method, source_pts, dest_pts
)
if transformer is None:
    raise RuntimeError(
        f"Fit failed: need at least "
        f"{QgsGcpTransformerInterface.create(method).minimumGcpCount()} GCPs"
    )
```

---

## Anti-Pattern 4: Ignoring the Return Value of transform()

### Wrong

```python
# Assuming transform() modifies variables in place (C++ behavior)
x, y = 50.0, 50.0
transformer.transform(x, y, False)
print(f"Transformed: ({x}, {y})")  # Still prints original values!
```

### Why It Fails

In Python, `transform()` returns a tuple `(bool, float, float)` due to SIP bindings. The original `x` and `y` variables are NOT modified. This is different from the C++ API where output parameters are modified by reference.

### Correct

```python
success, tx, ty = transformer.transform(50.0, 50.0, False)
if success:
    print(f"Transformed: ({tx}, {ty})")
```

---

## Anti-Pattern 5: Not Checking Residuals Before Applying Transformation

### Wrong

```python
# Create transformer and immediately apply to data without validation
transformer = QgsGcpTransformerInterface.createFromParameters(method, src, dst)
warper = QgsVectorWarper(method, gcps, crs)
warper.transformFeatures(layer.getFeatures(), sink, context)
# No idea if the transformation is accurate
```

### Why It Fails

A successful fit does NOT guarantee accuracy. GCPs may contain errors (wrong coordinate pairs, typos, wrong units). Without residual checks, bad transformations silently corrupt data.

### Correct

```python
import math

transformer = QgsGcpTransformerInterface.createFromParameters(method, src_pts, dst_pts)
if transformer is None:
    raise RuntimeError("Fit failed")

# Compute RMSE before applying
sum_sq = 0.0
for s, d in zip(src_pts, dst_pts):
    ok, tx, ty = transformer.transform(s.x(), s.y(), False)
    if ok:
        sum_sq += (tx - d.x()) ** 2 + (ty - d.y()) ** 2

rmse = math.sqrt(sum_sq / len(src_pts))
if rmse > acceptable_threshold:
    raise RuntimeError(f"RMSE {rmse:.6f} exceeds threshold -- check GCPs")

# Only proceed if accuracy is acceptable
warper = QgsVectorWarper(method, gcps, crs)
warper.transformFeatures(layer.getFeatures(), sink, context)
```

---

## Anti-Pattern 6: Mixing GCP Destination CRS Values

### Wrong

```python
# Different CRS per GCP without realizing the warper expects consistency
gcps = [
    QgsGcpPoint(src1, dst1, QgsCoordinateReferenceSystem("EPSG:4326"), True),
    QgsGcpPoint(src2, dst2, QgsCoordinateReferenceSystem("EPSG:3857"), True),  # Different CRS!
    QgsGcpPoint(src3, dst3, QgsCoordinateReferenceSystem("EPSG:4326"), True),
]
```

### Why It Fails

Each `QgsGcpPoint` stores its own destination CRS. When creating a `QgsVectorWarper`, the class uses `transformedDestinationPoint()` internally to reproject all destination coordinates to the warper's destination CRS. If CRS values are wrong or inconsistent, the internal reprojection produces incorrect coordinates.

### Correct

ALWAYS use a consistent CRS for all destination points, or ensure each GCP's CRS accurately reflects its coordinate values:

```python
dest_crs = QgsCoordinateReferenceSystem("EPSG:4326")
gcps = [
    QgsGcpPoint(src1, dst1, dest_crs, True),
    QgsGcpPoint(src2, dst2, dest_crs, True),
    QgsGcpPoint(src3, dst3, dest_crs, True),
]
```

---

## Anti-Pattern 7: Using Thin Plate Spline for Extrapolation

### Wrong

```python
# GCPs only cover a small area, but transforming points far outside
transformer = QgsGcpTransformerInterface.createFromParameters(
    QgsGcpTransformerInterface.TransformMethod.ThinPlateSpline,
    source_pts,  # All clustered in one corner
    dest_pts,
)
# Transform a point far from any GCP
ok, tx, ty = transformer.transform(9999, 9999, False)  # Wildly inaccurate
```

### Why It Fails

Thin Plate Spline interpolates exactly through GCP locations but can produce extreme distortion when extrapolating outside the convex hull of GCPs. The further from GCPs, the more unpredictable the results.

### Correct

For areas outside GCP coverage, use polynomial transformations (which extrapolate more smoothly) or add GCPs that bracket the full extent of the data:

```python
# Use PolynomialOrder1 for smoother extrapolation behavior
transformer = QgsGcpTransformerInterface.createFromParameters(
    QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1,
    source_pts, dest_pts,
)
```

---

## Anti-Pattern 8: World File with Wrong Pixel Height Sign

### Wrong

```python
# Positive pixel height -- image appears flipped vertically
write_world_file("output.tfw", 1.0, 0.0, 0.0, 1.0, 500000.0, 5200000.0)
```

### Why It Fails

In world files, the pixel height (line 4) MUST be negative for north-up images because pixel rows increase downward while map Y coordinates increase upward.

### Correct

```python
write_world_file("output.tfw", 1.0, 0.0, 0.0, -1.0, 500000.0, 5200000.0)
#                                                ^^^^  negative for north-up
```
