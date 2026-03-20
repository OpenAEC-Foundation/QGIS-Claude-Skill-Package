# qgis-impl-vector-analysis — Anti-Patterns

## AP-01: Operating on NULL Geometries

### Wrong

```python
for feature in layer.getFeatures():
    area = feature.geometry().area()  # CRASHES if geometry is NULL
    buffered = feature.geometry().buffer(100, 5)  # Returns empty geometry silently
```

### Right

```python
for feature in layer.getFeatures():
    if not feature.hasGeometry():
        continue
    geom = feature.geometry()
    if geom.isNull() or geom.isEmpty():
        continue
    area = geom.area()
    buffered = geom.buffer(100, 5)
```

### Why

Vector layers from external sources frequently contain features with NULL or empty geometries. Calling geometry methods on NULL geometries causes crashes or produces silently wrong results. ALWAYS check `hasGeometry()` before ANY geometry operation.

---

## AP-02: Mixing CRS in Direct Geometry Operations

### Wrong

```python
# layer_a is EPSG:4326 (geographic), layer_b is EPSG:28992 (projected)
for feat_a in layer_a.getFeatures():
    for feat_b in layer_b.getFeatures():
        if feat_a.geometry().intersects(feat_b.geometry()):  # WRONG CRS comparison
            print("Intersects!")
```

### Right

```python
from qgis.core import QgsCoordinateTransform, QgsCoordinateReferenceSystem, QgsProject

transform = QgsCoordinateTransform(
    layer_a.crs(),
    layer_b.crs(),
    QgsProject.instance()
)

for feat_a in layer_a.getFeatures():
    geom_a = QgsGeometry(feat_a.geometry())
    geom_a.transform(transform)  # Transform to layer_b's CRS
    for feat_b in layer_b.getFeatures():
        if geom_a.intersects(feat_b.geometry()):
            print("Intersects!")
```

### Why

`QgsGeometry` methods perform pure coordinate comparison with NO automatic CRS transformation. Comparing geometries in different CRS produces completely wrong results. Processing algorithms handle CRS automatically, but direct `QgsGeometry` operations do NOT. ALWAYS reproject to a common CRS before direct geometry operations.

---

## AP-03: Forgetting to Delete QgsVectorFileWriter

### Wrong

```python
writer = QgsVectorFileWriter.create(
    "/path/to/output.gpkg", fields, QgsWkbTypes.Polygon,
    crs, QgsCoordinateTransformContext(), save_options
)
for feature in layer.getFeatures():
    writer.addFeature(feature)
# File may be incomplete — writer is still open
```

### Right

```python
writer = QgsVectorFileWriter.create(
    "/path/to/output.gpkg", fields, QgsWkbTypes.Polygon,
    crs, QgsCoordinateTransformContext(), save_options
)
for feature in layer.getFeatures():
    writer.addFeature(feature)
del writer  # ALWAYS delete to flush buffers and close file
```

### Why

`QgsVectorFileWriter` buffers data in memory. The file is NOT fully written until the writer object is destroyed. Without `del writer`, the output file may be incomplete, truncated, or locked by the process.

---

## AP-04: Using Memory Layers for Persistent Results

### Wrong

```python
result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 100,
    'OUTPUT': 'memory:'  # Lost when QGIS closes
})
```

### Right (for persistent results)

```python
result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 100,
    'OUTPUT': '/path/to/output/buffers.gpkg'
})
```

### When Memory IS Appropriate

Use `'memory:'` ONLY for intermediate results in a processing chain where the final output is written to disk. NEVER use `'memory:'` as the final output of an analysis that must persist.

---

## AP-05: Not Fixing Geometries Before Overlay Operations

### Wrong

```python
# External data may contain invalid geometries
result = processing.run("native:intersection", {
    'INPUT': external_layer,  # May have self-intersections, duplicate vertices
    'OVERLAY': boundary,
    'OUTPUT': 'memory:'
})
# Algorithm may fail or produce incomplete results
```

### Right

```python
# ALWAYS fix geometries from external sources first
fixed = processing.run("native:fixgeometries", {
    'INPUT': external_layer,
    'OUTPUT': 'memory:'
})['OUTPUT']

result = processing.run("native:intersection", {
    'INPUT': fixed,
    'OVERLAY': boundary,
    'OUTPUT': 'memory:'
})
```

### Why

Overlay operations (intersection, union, difference) rely on valid topology. Invalid geometries (self-intersections, duplicate vertices, unclosed rings) cause algorithms to fail silently, produce incomplete output, or raise errors. ALWAYS run `native:fixgeometries` on external data before overlay operations.

---

## AP-06: Assuming processing.run() Modifies the Input Layer

### Wrong

```python
processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 100,
    'OUTPUT': 'memory:'
})
# Expecting 'layer' to now contain buffers — it does NOT
for feature in layer.getFeatures():
    print(feature.geometry().area())  # Still the original geometries
```

### Right

```python
result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 100,
    'OUTPUT': 'memory:'
})
buffered_layer = result['OUTPUT']  # This is the new layer with buffers
for feature in buffered_layer.getFeatures():
    print(feature.geometry().area())
```

### Why

`processing.run()` NEVER modifies the input layer. It ALWAYS creates a new output. The result dictionary contains the output layer under the `'OUTPUT'` key. Ignoring the return value means your analysis results are discarded.

---

## AP-07: Using Nested Feature Loops Without Spatial Index

### Wrong

```python
# O(n*m) complexity — extremely slow for large datasets
for feat_a in layer_a.getFeatures():
    for feat_b in layer_b.getFeatures():
        if feat_a.geometry().intersects(feat_b.geometry()):
            process(feat_a, feat_b)
```

### Right

```python
from qgis.core import QgsSpatialIndex

# Build index — O(m log m)
index_b = QgsSpatialIndex(layer_b.getFeatures())

# Query index — O(n log m) total
for feat_a in layer_a.getFeatures():
    if not feat_a.hasGeometry():
        continue
    bbox = feat_a.geometry().boundingBox()
    candidate_ids = index_b.intersects(bbox)
    for fid in candidate_ids:
        feat_b = layer_b.getFeature(fid)
        if feat_a.geometry().intersects(feat_b.geometry()):
            process(feat_a, feat_b)
```

### Why

Nested feature loops without a spatial index have O(n*m) complexity. For two layers of 10,000 features each, this means 100 million geometry comparisons. A spatial index reduces this to approximately O(n log m), making the operation orders of magnitude faster. ALWAYS use `QgsSpatialIndex` for spatial queries across layers.

---

## AP-08: Using Shapefile for New Projects

### Wrong

```python
save_options = QgsVectorFileWriter.SaveVectorOptions()
save_options.driverName = "ESRI Shapefile"  # Legacy format with many limitations
```

### Right

```python
save_options = QgsVectorFileWriter.SaveVectorOptions()
save_options.driverName = "GPKG"  # Modern, no limitations
```

### Why

Shapefile has severe limitations: field names truncated to 10 characters, single geometry type per file, no NULL value support, 2GB file size limit, requires multiple sidecar files (.shx, .dbf, .prj). GeoPackage has NONE of these limitations. ALWAYS use GeoPackage for new projects. Only use Shapefile when required by external systems.

---

## AP-09: Not Initializing Processing Framework in Standalone Scripts

### Wrong

```python
# In a standalone Python script (NOT the QGIS Python console)
import processing
result = processing.run("native:buffer", {...})  # ERROR: Processing not initialized
```

### Right

```python
from qgis.core import QgsApplication
import sys

qgs = QgsApplication([], False)
qgs.initQgis()

# Initialize Processing AFTER QgsApplication
from processing.core.Processing import Processing
Processing.initialize()

import processing
result = processing.run("native:buffer", {...})

qgs.exitQgis()
```

### Why

The Processing framework is NOT automatically available in standalone scripts. It requires explicit initialization after `QgsApplication.initQgis()`. In the QGIS Python console, Processing is already initialized. This distinction catches many developers who test code in the console and then deploy it as a standalone script.

---

## AP-10: Buffer Distance in Wrong Units

### Wrong

```python
# Layer is in EPSG:4326 (degrees)
result = processing.run("native:buffer", {
    'INPUT': layer_4326,
    'DISTANCE': 100,       # This is 100 DEGREES, not 100 meters!
    'OUTPUT': 'memory:'
})
```

### Right

```python
# Option 1: Reproject to a projected CRS first
reprojected = processing.run("native:reprojectlayer", {
    'INPUT': layer_4326,
    'TARGET_CRS': 'EPSG:32632',  # UTM zone 32N (meters)
    'OUTPUT': 'memory:'
})['OUTPUT']

result = processing.run("native:buffer", {
    'INPUT': reprojected,
    'DISTANCE': 100,       # Now correctly 100 meters
    'OUTPUT': 'memory:'
})

# Option 2: Use QgsDistanceArea for unit conversion
from qgis.core import QgsDistanceArea, QgsCoordinateReferenceSystem
d = QgsDistanceArea()
d.setSourceCrs(layer_4326.crs(), QgsProject.instance().transformContext())
d.setEllipsoid('WGS84')
# Then use d.measureLine() for accurate distances
```

### Why

Buffer distances are specified in the CRS units of the input layer. For geographic CRS (EPSG:4326), units are degrees. A buffer of 100 in EPSG:4326 means 100 degrees — roughly 11,000 km. ALWAYS reproject to a projected CRS (meters) before buffer operations, or use `QgsDistanceArea` for ellipsoidal calculations.
