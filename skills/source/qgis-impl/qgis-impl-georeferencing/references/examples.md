# qgis-impl-georeferencing — Examples

## Example 1: Complete Vector Georeferencing Pipeline

Georeference a vector layer with 4 GCPs using polynomial order 1, including residual computation.

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
import math

# 1. Create source layer with test features
source_layer = QgsVectorLayer(
    "Point?field=name:string&field=value:integer", "source", "memory"
)
provider = source_layer.dataProvider()

f1 = QgsFeature()
f1.setAttributes(["station_A", 1])
f1.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(150, 250)))

f2 = QgsFeature()
f2.setAttributes(["station_B", 2])
f2.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(300, 150)))

provider.addFeatures([f1, f2])

# 2. Define GCPs (source local coords -> destination map coords)
dest_crs = QgsCoordinateReferenceSystem("EPSG:4326")
gcps = [
    QgsGcpPoint(QgsPointXY(0, 0), QgsPointXY(10.0, 50.0), dest_crs, True),
    QgsGcpPoint(QgsPointXY(500, 0), QgsPointXY(11.0, 50.0), dest_crs, True),
    QgsGcpPoint(QgsPointXY(500, 500), QgsPointXY(11.0, 51.0), dest_crs, True),
    QgsGcpPoint(QgsPointXY(0, 500), QgsPointXY(10.0, 51.0), dest_crs, True),
]

# 3. Compute residuals BEFORE applying to real data
source_pts = [gcp.sourcePoint() for gcp in gcps]
dest_pts = [gcp.destinationPoint() for gcp in gcps]
method = QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1

transformer = QgsGcpTransformerInterface.createFromParameters(
    method, source_pts, dest_pts
)
if transformer is None:
    raise RuntimeError("Fit failed")

for i, (src, dst) in enumerate(zip(source_pts, dest_pts)):
    ok, tx, ty = transformer.transform(src.x(), src.y(), False)
    if ok:
        residual = math.sqrt((tx - dst.x()) ** 2 + (ty - dst.y()) ** 2)
        print(f"GCP {i}: residual = {residual:.8f}")

# 4. Warp features
warper = QgsVectorWarper(method, gcps, dest_crs)
sink = QgsFeatureStore()
success = warper.transformFeatures(
    source_layer.getFeatures(),
    sink,
    QgsProject.instance().transformContext(),
)

if not success:
    raise RuntimeError(f"Warp failed: {warper.error()}")

# 5. Output results
for feature in sink.features():
    geom = feature.geometry()
    attrs = feature.attributes()
    print(f"{attrs[0]}: {geom.asWkt()}")
```

---

## Example 2: Geometry-Level Transformation

Transform individual geometries without creating a full warper pipeline.

```python
from qgis.analysis import (
    QgsGcpGeometryTransformer,
    QgsGcpTransformerInterface,
)
from qgis.core import QgsGeometry, QgsPointXY

# Define corresponding point pairs
source_pts = [
    QgsPointXY(0, 0),
    QgsPointXY(1000, 0),
    QgsPointXY(1000, 1000),
    QgsPointXY(0, 1000),
]
dest_pts = [
    QgsPointXY(500000, 5200000),
    QgsPointXY(501000, 5200000),
    QgsPointXY(501000, 5201000),
    QgsPointXY(500000, 5201000),
]

# Create transformer
geo_tf = QgsGcpGeometryTransformer(
    QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1,
    source_pts,
    dest_pts,
)

# Transform a polygon
polygon = QgsGeometry.fromWkt(
    "POLYGON((100 100, 200 100, 200 200, 100 200, 100 100))"
)
result, ok = geo_tf.transform(polygon)
if ok:
    print(f"Transformed polygon: {result.asWkt()}")

# Transform a line
line = QgsGeometry.fromWkt("LINESTRING(50 50, 150 300, 400 200)")
result, ok = geo_tf.transform(line)
if ok:
    print(f"Transformed line: {result.asWkt()}")
```

---

## Example 3: Raster Georeferencing with GDAL

Georeference an unreferenced TIFF using GDAL Python bindings.

```python
from osgeo import gdal, osr

input_path = "/path/to/scan.tif"
output_path = "/path/to/georeferenced.tif"

# 1. Define GCPs: gdal.GCP(map_x, map_y, map_z, pixel_col, pixel_row)
gcps = [
    gdal.GCP(500000.0, 5200000.0, 0, 0, 0),           # top-left
    gdal.GCP(501000.0, 5200000.0, 0, 1000, 0),         # top-right
    gdal.GCP(501000.0, 5199000.0, 0, 1000, 1000),      # bottom-right
    gdal.GCP(500000.0, 5199000.0, 0, 0, 1000),         # bottom-left
]

# 2. Open and assign GCPs
src_ds = gdal.OpenShared(input_path, gdal.GA_ReadOnly)
if src_ds is None:
    raise RuntimeError(f"Cannot open {input_path}")

srs = osr.SpatialReference()
srs.ImportFromEPSG(32633)  # UTM zone 33N
src_ds.SetGCPs(gcps, srs.ExportToWkt())

# 3. Create warped VRT
dst_srs = osr.SpatialReference()
dst_srs.ImportFromEPSG(32633)
vrt_ds = gdal.AutoCreateWarpedVRT(
    src_ds, None, dst_srs.ExportToWkt(), gdal.GRA_Bilinear, 0.125
)

# 4. Write output GeoTIFF
driver = gdal.GetDriverByName("GTiff")
dst_ds = driver.Create(
    output_path,
    vrt_ds.RasterXSize,
    vrt_ds.RasterYSize,
    src_ds.RasterCount,
    gdal.GDT_Byte,
)
dst_ds.SetProjection(dst_srs.ExportToWkt())
dst_ds.SetGeoTransform(vrt_ds.GetGeoTransform())
gdal.ReprojectImage(src_ds, dst_ds, None, None, gdal.GRA_Bilinear)

# 5. Cleanup
dst_ds = None
vrt_ds = None
src_ds = None
print(f"Georeferenced raster written to {output_path}")
```

---

## Example 4: Point-by-Point Transformation

Transform individual coordinates without geometry objects.

```python
from qgis.analysis import QgsGcpTransformerInterface
from qgis.core import QgsPointXY

source_pts = [
    QgsPointXY(100, 100),
    QgsPointXY(900, 100),
    QgsPointXY(900, 700),
]
dest_pts = [
    QgsPointXY(5.0, 52.0),
    QgsPointXY(6.0, 52.0),
    QgsPointXY(6.0, 53.0),
]

transformer = QgsGcpTransformerInterface.createFromParameters(
    QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1,
    source_pts,
    dest_pts,
)

if transformer is None:
    raise RuntimeError("Fit failed")

# Forward transform: source -> destination
coordinates_to_transform = [(500, 400), (200, 600), (750, 150)]
for x, y in coordinates_to_transform:
    ok, tx, ty = transformer.transform(x, y, False)
    if ok:
        print(f"({x}, {y}) -> ({tx:.6f}, {ty:.6f})")

# Inverse transform: destination -> source
ok, sx, sy = transformer.transform(5.5, 52.5, True)  # True = inverse
if ok:
    print(f"Inverse: (5.5, 52.5) -> ({sx:.2f}, {sy:.2f})")
```

---

## Example 5: Comparing Transformation Methods

Evaluate multiple transformation types on the same GCP set.

```python
from qgis.analysis import QgsGcpTransformerInterface
from qgis.core import QgsPointXY
import math

TM = QgsGcpTransformerInterface.TransformMethod

source_pts = [
    QgsPointXY(50, 50), QgsPointXY(450, 50), QgsPointXY(450, 350),
    QgsPointXY(50, 350), QgsPointXY(250, 200), QgsPointXY(150, 300),
]
dest_pts = [
    QgsPointXY(10.0, 50.0), QgsPointXY(11.0, 50.0), QgsPointXY(11.0, 51.0),
    QgsPointXY(10.0, 51.0), QgsPointXY(10.5, 50.5), QgsPointXY(10.2, 50.8),
]

methods_to_test = [
    TM.Linear,
    TM.Helmert,
    TM.PolynomialOrder1,
    TM.PolynomialOrder2,
    TM.Projective,
    TM.ThinPlateSpline,
]

for method in methods_to_test:
    transformer = QgsGcpTransformerInterface.createFromParameters(
        method, source_pts, dest_pts
    )
    if transformer is None:
        name = QgsGcpTransformerInterface.methodToString(method)
        print(f"{name}: fit FAILED (need {transformer.minimumGcpCount()} GCPs)")
        continue

    # Compute RMSE
    sum_sq = 0.0
    for src, dst in zip(source_pts, dest_pts):
        ok, tx, ty = transformer.transform(src.x(), src.y(), False)
        if ok:
            sum_sq += (tx - dst.x()) ** 2 + (ty - dst.y()) ** 2
    rmse = math.sqrt(sum_sq / len(source_pts))

    name = QgsGcpTransformerInterface.methodToString(method)
    print(f"{name}: RMSE = {rmse:.8f}")
```

---

## Example 6: Write Georeferenced Output to File with Processing

Save warped vector features to a GeoPackage using QgsVectorFileWriter.

```python
from qgis.analysis import (
    QgsGcpPoint, QgsGcpTransformerInterface, QgsVectorWarper
)
from qgis.core import (
    QgsCoordinateReferenceSystem, QgsCoordinateTransformContext,
    QgsFeatureStore, QgsFields, QgsPointXY, QgsProject,
    QgsVectorFileWriter, QgsVectorLayer, QgsWkbTypes,
)

# Setup source and GCPs (abbreviated)
source_layer = QgsVectorLayer("path/to/input.shp", "input", "ogr")
dest_crs = QgsCoordinateReferenceSystem("EPSG:4326")
gcps = [
    QgsGcpPoint(QgsPointXY(0, 0), QgsPointXY(10.0, 50.0), dest_crs, True),
    QgsGcpPoint(QgsPointXY(100, 0), QgsPointXY(11.0, 50.0), dest_crs, True),
    QgsGcpPoint(QgsPointXY(100, 100), QgsPointXY(11.0, 51.0), dest_crs, True),
]

# Warp
warper = QgsVectorWarper(
    QgsGcpTransformerInterface.TransformMethod.PolynomialOrder1,
    gcps, dest_crs,
)
sink = QgsFeatureStore()
success = warper.transformFeatures(
    source_layer.getFeatures(),
    sink,
    QgsProject.instance().transformContext(),
)

if not success:
    raise RuntimeError(f"Warp failed: {warper.error()}")

# Write to GeoPackage
output_path = "/path/to/output.gpkg"
options = QgsVectorFileWriter.SaveVectorOptions()
options.driverName = "GPKG"

writer = QgsVectorFileWriter.create(
    output_path,
    source_layer.fields(),
    source_layer.wkbType(),
    dest_crs,
    QgsCoordinateTransformContext(),
    options,
)

for feature in sink.features():
    writer.addFeature(feature)

del writer  # Flush and close
print(f"Written to {output_path}")
```
