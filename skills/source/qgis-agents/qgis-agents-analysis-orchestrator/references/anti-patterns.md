# anti-patterns.md — Analysis Orchestrator

## Workflow Planning Anti-Patterns

### AP-1: Skipping CRS Alignment

**WRONG**: Running overlay operations on layers with different CRS.

```python
# WRONG — layers have different CRS, result is geometrically incorrect
result = processing.run("native:intersection", {
    'INPUT': layer_epsg4326,
    'OVERLAY': layer_epsg28992,
    'OUTPUT': 'memory:'
})
```

**CORRECT**: ALWAYS reproject all layers to a common CRS before ANY multi-layer operation.

```python
# CORRECT — reproject first
reprojected = processing.run("native:reprojectlayer", {
    'INPUT': layer_epsg4326,
    'TARGET_CRS': 'EPSG:28992',
    'OUTPUT': 'memory:'
})['OUTPUT']

result = processing.run("native:intersection", {
    'INPUT': reprojected,
    'OVERLAY': layer_epsg28992,
    'OUTPUT': 'memory:'
})
```

---

### AP-2: Using Geographic CRS for Distance/Area

**WRONG**: Calculating buffer distance in degrees (EPSG:4326).

```python
# WRONG — distance 0.01 is in degrees, not meters
result = processing.run("native:buffer", {
    'INPUT': points_epsg4326,
    'DISTANCE': 0.01,  # What does 0.01 degrees mean in meters? It varies!
    'OUTPUT': 'memory:'
})
```

**CORRECT**: ALWAYS reproject to a projected CRS for distance/area operations.

```python
# CORRECT — reproject, then buffer in meters
reproj = processing.run("native:reprojectlayer", {
    'INPUT': points_epsg4326,
    'TARGET_CRS': 'EPSG:28992',
    'OUTPUT': 'memory:'
})['OUTPUT']

result = processing.run("native:buffer", {
    'INPUT': reproj,
    'DISTANCE': 1000,  # 1000 meters — unambiguous
    'OUTPUT': 'memory:'
})
```

---

### AP-3: No Error Handling in Algorithm Chains

**WRONG**: Chaining algorithms without error handling — one failure crashes the entire chain with an unhelpful traceback.

```python
# WRONG — no error handling
buffered = processing.run("native:buffer", params1)['OUTPUT']
clipped = processing.run("native:clip", {'INPUT': buffered, ...})['OUTPUT']
dissolved = processing.run("native:dissolve", {'INPUT': clipped, ...})['OUTPUT']
```

**CORRECT**: ALWAYS wrap each step in try/except with meaningful error messages.

```python
# CORRECT — each step has error handling
try:
    buffered = processing.run("native:buffer", params1)['OUTPUT']
except QgsProcessingException as e:
    raise RuntimeError(f"Step 1 (buffer) failed: {e}")

try:
    clipped = processing.run("native:clip", {
        'INPUT': buffered, 'OVERLAY': mask, 'OUTPUT': 'memory:'
    })['OUTPUT']
except QgsProcessingException as e:
    raise RuntimeError(f"Step 2 (clip) failed: {e}")
```

---

### AP-4: Missing Layer Validity Checks

**WRONG**: Using a layer without checking if it loaded correctly.

```python
# WRONG — layer may be invalid (wrong path, missing provider)
layer = QgsVectorLayer("/data/missing_file.gpkg", "data", "ogr")
result = processing.run("native:buffer", {'INPUT': layer, ...})
# Crashes with cryptic error
```

**CORRECT**: ALWAYS check `isValid()` immediately after layer creation.

```python
# CORRECT
layer = QgsVectorLayer("/data/missing_file.gpkg", "data", "ogr")
if not layer.isValid():
    raise RuntimeError(f"Failed to load layer from: /data/missing_file.gpkg")
```

---

### AP-5: Hardcoded Temporary Paths

**WRONG**: Using platform-specific temporary paths.

```python
# WRONG — /tmp does not exist on Windows
result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 100,
    'OUTPUT': '/tmp/buffer_result.gpkg'
})
```

**CORRECT**: ALWAYS use `'memory:'` for intermediate results or `QgsProcessing.TEMPORARY_OUTPUT` for disk-based temp files.

```python
# CORRECT — cross-platform
result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 100,
    'OUTPUT': 'memory:'  # Or QgsProcessing.TEMPORARY_OUTPUT
})
```

---

### AP-6: Assuming Third-Party Algorithms Exist

**WRONG**: Using GRASS or SAGA algorithms without checking provider availability.

```python
# WRONG — GRASS may not be installed
result = processing.run("grass7:v.clean", {
    'input': layer,
    'type': [0, 1, 2, 3, 4, 5, 6],
    'tool': [0],
    'output': 'memory:'
})
```

**CORRECT**: ALWAYS check algorithm availability and provide a `native:` fallback.

```python
# CORRECT — check first, fallback to native
registry = QgsApplication.processingRegistry()
if registry.algorithmById('grass7:v.clean') is not None:
    result = processing.run("grass7:v.clean", params)
else:
    # Fallback to native equivalent
    result = processing.run("native:fixgeometries", {
        'INPUT': layer,
        'OUTPUT': 'memory:'
    })
```

---

### AP-7: Missing Spatial Index on Large Datasets

**WRONG**: Running spatial operations on large datasets without an index.

```python
# WRONG — spatial join on 500,000 features without index takes 100x longer
result = processing.run("native:joinattributesbylocation", {
    'INPUT': large_layer,    # 500,000 features, no spatial index
    'JOIN': another_layer,
    'OUTPUT': 'memory:'
})
```

**CORRECT**: ALWAYS create a spatial index before spatial operations on datasets with more than a few thousand features.

```python
# CORRECT — index first
processing.run("native:createspatialindex", {'INPUT': large_layer})
processing.run("native:createspatialindex", {'INPUT': another_layer})

result = processing.run("native:joinattributesbylocation", {
    'INPUT': large_layer,
    'JOIN': another_layer,
    'OUTPUT': 'memory:'
})
```

---

### AP-8: Wrong Analysis Type Selection

**WRONG**: Using vector overlay when raster analysis is more appropriate.

```
Scenario: "What percentage of each municipality is forested?"

WRONG approach: Vectorize forest raster → intersect with municipalities → calculate areas
(Vectorizing a high-resolution raster creates millions of polygons, takes hours)

CORRECT approach: Use native:zonalstatisticsfb directly on the raster with municipality polygons
(Completes in seconds, no intermediate conversion needed)
```

**Rule**: If one input is already raster and the question is "summarize raster within vector zones", ALWAYS use `native:zonalstatisticsfb`. NEVER vectorize the raster first.

---

### AP-9: Ignoring Geometry Validity

**WRONG**: Running overlay operations on layers with invalid geometries.

```python
# WRONG — invalid geometries cause silent failures or wrong results
result = processing.run("native:intersection", {
    'INPUT': layer_with_self_intersections,
    'OVERLAY': another_layer,
    'OUTPUT': 'memory:'
})
# Result may be missing features or have corrupt geometries
```

**CORRECT**: ALWAYS fix geometries before overlay operations.

```python
# CORRECT — fix first
fixed = processing.run("native:fixgeometries", {
    'INPUT': layer_with_self_intersections,
    'OUTPUT': 'memory:'
})['OUTPUT']

result = processing.run("native:intersection", {
    'INPUT': fixed,
    'OVERLAY': another_layer,
    'OUTPUT': 'memory:'
})
```

---

### AP-10: Using Shapefile as Default Output

**WRONG**: Defaulting to Shapefile format for output.

```python
# WRONG — Shapefile truncates field names to 10 chars, 2GB limit, no multi-layer
processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 100,
    'OUTPUT': '/data/output/result.shp'  # Field names will be truncated
})
```

**CORRECT**: ALWAYS use GeoPackage as the default output format.

```python
# CORRECT — GeoPackage has no field name limits, no size limit, supports multi-layer
processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 100,
    'OUTPUT': '/data/output/result.gpkg'
})
```

---

### AP-11: No Cancellation Check in Processing Loops

**WRONG**: Processing features in a loop without checking for cancellation.

```python
# WRONG — user cannot cancel, UI appears frozen
def processAlgorithm(self, parameters, context, feedback):
    for feature in source.getFeatures():
        # Heavy processing...
        pass
```

**CORRECT**: ALWAYS check `feedback.isCanceled()` at the top of every loop iteration.

```python
# CORRECT
def processAlgorithm(self, parameters, context, feedback):
    total = 100.0 / source.featureCount() if source.featureCount() else 0
    for current, feature in enumerate(source.getFeatures()):
        if feedback.isCanceled():
            break
        # Heavy processing...
        feedback.setProgress(int(current * total))
```

---

### AP-12: Missing Transform Context in CRS Operations

**WRONG**: Creating coordinate transforms without project transform context.

```python
# WRONG — ignores datum transformation preferences, silent precision loss
xform = QgsCoordinateTransform(crs_source, crs_target)
```

**CORRECT**: ALWAYS include the transform context.

```python
# CORRECT
xform = QgsCoordinateTransform(
    crs_source, crs_target,
    QgsProject.instance().transformContext()
)
```
