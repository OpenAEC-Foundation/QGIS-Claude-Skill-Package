---
name: qgis-impl-vector-analysis
description: >
  Use when performing spatial analysis on vector data: buffering, clipping, intersecting, dissolving, or joining layers.
  Prevents geometry operation failures from NULL geometries and wrong CRS combinations.
  Covers spatial queries, overlay operations, attribute/spatial joins, field calculations, and vector output.
  Keywords: vector analysis, buffer, clip, intersect, dissolve, spatial join, overlay, QgsGeometry, QgsVectorFileWriter.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-impl-vector-analysis

## Quick Reference

### Core Classes

| Class | Purpose | Import |
|-------|---------|--------|
| `QgsFeatureRequest` | Filter features by rect, expression, or FID | `qgis.core` |
| `QgsSpatialIndex` | In-memory R-tree for fast spatial lookups | `qgis.core` |
| `QgsGeometry` | Geometry operations (buffer, intersects, contains) | `qgis.core` |
| `QgsVectorFileWriter` | Write vector layers to disk (GPKG, SHP, GeoJSON) | `qgis.core` |
| `QgsCoordinateTransformContext` | CRS transformation context for output | `qgis.core` |
| `processing` | Run QGIS Processing algorithms | `import processing` |

### Key Processing Algorithm IDs

| Algorithm | Purpose |
|-----------|---------|
| `native:buffer` | Create buffer zones around features |
| `native:clip` | Clip input layer by overlay layer |
| `native:intersection` | Geometric intersection of two layers |
| `native:union` | Geometric union of two layers |
| `native:difference` | Geometric difference (A minus B) |
| `native:symmetricaldifference` | Features in A or B but not both |
| `native:dissolve` | Merge features by attribute value |
| `native:joinattributestable` | Join attributes by matching field values |
| `native:joinattributesbylocation` | Spatial join based on geometry relationship |
| `native:extractbyexpression` | Extract features matching expression |
| `native:extractbylocation` | Extract features by spatial relationship |
| `native:fieldcalculator` | Calculate field values with expressions |

---

## Critical Warnings

**NEVER** pass layers with different CRS to overlay operations without reprojecting first. Processing algorithms reproject automatically, but PyQGIS geometry methods do NOT. ALWAYS verify CRS match before direct `QgsGeometry` operations.

**NEVER** call `QgsGeometry` methods on NULL geometries. ALWAYS check `feature.hasGeometry()` before ANY geometry operation. NULL geometries cause silent failures or crashes.

**NEVER** forget to call `del writer` after using `QgsVectorFileWriter.create()`. The file is NOT written to disk until the writer object is destroyed.

**NEVER** use `'OUTPUT': 'memory:'` for large datasets in batch processing. Memory layers consume RAM and are lost when QGIS closes. ALWAYS write to GeoPackage for persistence.

**NEVER** assume `processing.run()` modifies the input layer. It ALWAYS creates a new output layer. Access results via `result['OUTPUT']`.

**ALWAYS** use `native:fixgeometries` before overlay operations when input data comes from external sources. Invalid geometries cause overlay algorithms to fail silently or produce incomplete results.

**ALWAYS** initialize the Processing framework before calling `processing.run()` in standalone scripts:

```python
from qgis.core import QgsApplication
import processing
from processing.core.Processing import Processing
Processing.initialize()
```

---

## Decision Tree: Which Overlay Operation to Use

```
What is your goal?
|
+-- Keep ONLY the area where both layers overlap
|   --> native:intersection
|
+-- Combine ALL areas from both layers into one
|   --> native:union
|
+-- Remove areas of layer B from layer A
|   +-- Keep remainder of A only
|   |   --> native:difference
|   +-- Keep remainder of BOTH A and B (exclude overlap)
|       --> native:symmetricaldifference
|
+-- Cut layer A to the boundary of layer B
|   --> native:clip
|
+-- Transfer attributes from one layer to another
    +-- Based on matching field values
    |   --> native:joinattributestable
    +-- Based on spatial relationship
    |   --> native:joinattributesbylocation
    +-- Based on nearest feature
        --> native:joinattributesbynearest
```

### Clip vs Intersection

| Aspect | `native:clip` | `native:intersection` |
|--------|--------------|----------------------|
| Output geometry | Same as input | May split/merge at overlay boundaries |
| Overlay attributes | NOT included | Included in output |
| Use case | Trim to study area | Combine data from both layers |
| Performance | Faster | Slower (attribute handling) |

---

## Essential Patterns

### Pattern 1: Spatial Query with Feature Request

```python
from qgis.core import QgsFeatureRequest, QgsRectangle

# Filter by bounding rectangle with exact geometry test
area = QgsRectangle(100.0, -1.0, 101.0, 0.0)
request = QgsFeatureRequest().setFilterRect(area)
request.setFlags(QgsFeatureRequest.ExactIntersect)
request.setLimit(100)

for feature in layer.getFeatures(request):
    print(feature.id(), feature.geometry().asWkt())
```

### Pattern 2: Spatial Index for Fast Lookups

```python
from qgis.core import QgsSpatialIndex, QgsPointXY

index = QgsSpatialIndex(layer.getFeatures())

# Nearest neighbor — returns feature IDs
nearest_ids = index.nearestNeighbor(QgsPointXY(15.5, 47.1), 5)

# Bounding box intersection — returns feature IDs
bbox = QgsRectangle(14.0, 46.0, 17.0, 49.0)
intersecting_ids = index.intersects(bbox)

# Retrieve actual features by ID
for fid in nearest_ids:
    feature = layer.getFeature(fid)
```

### Pattern 3: Buffer Analysis via Processing

```python
import processing

result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 100,
    'SEGMENTS': 5,
    'END_CAP_STYLE': 0,   # 0=Round, 1=Flat, 2=Square
    'JOIN_STYLE': 0,       # 0=Round, 1=Miter, 2=Bevel
    'MITER_LIMIT': 2,
    'DISSOLVE': False,
    'OUTPUT': 'memory:'
})
buffered_layer = result['OUTPUT']
```

### Pattern 4: Buffer via QgsGeometry (Single Feature)

```python
geom = feature.geometry()
if not geom.isNull():
    buffered = geom.buffer(100.0, 5)  # distance, segments
```

### Pattern 5: Overlay Operation (Intersection)

```python
import processing

result = processing.run("native:intersection", {
    'INPUT': layer_a,
    'OVERLAY': layer_b,
    'INPUT_FIELDS': [],
    'OVERLAY_FIELDS': [],
    'OVERLAY_FIELDS_PREFIX': '',
    'OUTPUT': 'memory:'
})
intersection_layer = result['OUTPUT']
```

### Pattern 6: Attribute Join by Field Value

```python
result = processing.run("native:joinattributestable", {
    'INPUT': layer_a,
    'FIELD': 'id',
    'INPUT_2': layer_b,
    'FIELD_2': 'foreign_id',
    'FIELDS_TO_COPY': [],
    'METHOD': 0,               # 0=one-to-many, 1=first match only
    'DISCARD_NONMATCHING': False,
    'PREFIX': '',
    'OUTPUT': 'memory:'
})
```

### Pattern 7: Spatial Join by Location

```python
result = processing.run("native:joinattributesbylocation", {
    'INPUT': target_layer,
    'PREDICATE': [0],          # 0=intersects, 1=contains, 2=equals,
                               # 3=touches, 4=overlaps, 5=within, 6=crosses
    'JOIN': join_layer,
    'JOIN_FIELDS': [],
    'METHOD': 0,               # 0=one-to-many, 1=one-to-first, 2=largest overlap
    'DISCARD_NONMATCHING': False,
    'PREFIX': '',
    'OUTPUT': 'memory:'
})
```

### Pattern 8: Dissolve by Attribute

```python
result = processing.run("native:dissolve", {
    'INPUT': layer,
    'FIELD': ['province'],     # Dissolve field(s); empty list = dissolve all
    'SEPARATE_DISJOINT': False,
    'OUTPUT': 'memory:'
})
```

---

## Common Operations

### Field Calculation (Edit Buffer)

```python
with edit(layer):
    field_idx = layer.fields().indexOf('area_m2')
    for feature in layer.getFeatures():
        if feature.hasGeometry():
            area = feature.geometry().area()
            layer.changeAttributeValue(feature.id(), field_idx, area)
```

### Field Calculation (Processing)

```python
result = processing.run("native:fieldcalculator", {
    'INPUT': layer,
    'FIELD_NAME': 'area_m2',
    'FIELD_TYPE': 0,           # 0=Float, 1=Integer, 2=String, 3=Date
    'FIELD_LENGTH': 10,
    'FIELD_PRECISION': 3,
    'FORMULA': '$area',
    'OUTPUT': 'memory:'
})
```

### Feature Selection

```python
# Select by expression
layer.selectByExpression('"type" = \'highway\'')

# Iterate selected features
for feature in layer.selectedFeatures():
    print(feature.id())

# Clear selection
layer.removeSelection()
```

### Extract Features by Expression

```python
result = processing.run("native:extractbyexpression", {
    'INPUT': layer,
    'EXPRESSION': '"population" > 50000',
    'OUTPUT': 'memory:'
})
```

### Extract Features by Location

```python
result = processing.run("native:extractbylocation", {
    'INPUT': layer,
    'PREDICATE': [0],          # 0=intersects
    'INTERSECT': reference_layer,
    'OUTPUT': 'memory:'
})
```

### Write Output to GeoPackage

```python
from qgis.core import QgsVectorFileWriter, QgsCoordinateTransformContext

save_options = QgsVectorFileWriter.SaveVectorOptions()
save_options.driverName = "GPKG"
save_options.fileEncoding = "UTF-8"

error = QgsVectorFileWriter.writeAsVectorFormatV3(
    layer,
    "/path/to/output.gpkg",
    QgsCoordinateTransformContext(),
    save_options
)
```

### Write Features Individually

```python
from qgis.core import QgsVectorFileWriter, QgsWkbTypes, QgsCoordinateTransformContext

save_options = QgsVectorFileWriter.SaveVectorOptions()
save_options.driverName = "GPKG"

writer = QgsVectorFileWriter.create(
    "/path/to/output.gpkg",
    layer.fields(),
    QgsWkbTypes.Polygon,
    layer.crs(),
    QgsCoordinateTransformContext(),
    save_options
)
for feature in layer.getFeatures():
    writer.addFeature(feature)
del writer  # ALWAYS delete to flush and close the file
```

### Aggregate with Statistics

```python
result = processing.run("native:aggregate", {
    'INPUT': layer,
    'GROUP_BY': '"province"',
    'AGGREGATES': [
        {'aggregate': 'sum', 'delimiter': ',', 'input': '"population"',
         'length': 10, 'name': 'total_pop', 'precision': 0, 'type': 6}
    ],
    'OUTPUT': 'memory:'
})
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for QgsGeometry, QgsVectorFileWriter, QgsFeatureRequest, QgsSpatialIndex, and processing algorithm parameters
- [references/examples.md](references/examples.md) -- Complete vector analysis workflows with realistic data
- [references/anti-patterns.md](references/anti-patterns.md) -- Common vector analysis mistakes and how to avoid them

### Official Sources

- https://docs.qgis.org/3.34/en/docs/pyqgis_developer_cookbook/vector.html
- https://docs.qgis.org/3.34/en/docs/pyqgis_developer_cookbook/geometry.html
- https://qgis.org/pyqgis/3.34/core/QgsGeometry.html
- https://qgis.org/pyqgis/3.34/core/QgsVectorFileWriter.html
- https://qgis.org/pyqgis/3.34/core/QgsSpatialIndex.html
