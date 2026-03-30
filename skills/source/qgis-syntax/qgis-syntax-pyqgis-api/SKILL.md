---
name: qgis-syntax-pyqgis-api
description: >
  Use when writing PyQGIS scripts, accessing features, editing layers, or performing geometry operations.
  Prevents edit session violations, threading crashes, and inefficient feature iteration.
  Covers 4 scripting contexts, feature CRUD, geometry operations, spatial indexing, background tasks, and basic symbology.
  Keywords: PyQGIS, QgsFeature, QgsFeatureRequest, QgsGeometry, edit layer, spatial index, QgsTask, background thread, symbology, read features, loop through layer, change style, select by attribute.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-syntax-pyqgis-api

## Quick Reference

### Scripting Contexts

| Context | `iface` Available | Initialization Required | Use Case |
|---------|-------------------|------------------------|----------|
| Python Console | Yes | None | Interactive exploration, quick tests |
| Standalone Script | No | `QgsApplication([], False)` + `initQgis()` | CLI tools, CI pipelines, batch processing |
| Processing Script | No (`feedback` instead) | None (framework handles it) | Geoprocessing algorithms |
| Plugin | Yes (via `classFactory`) | None | GUI extensions, toolbar tools |

### Core Classes

| Class | Purpose | Key Methods |
|-------|---------|-------------|
| `QgsFeature` | Single feature (geometry + attributes) | `id()`, `geometry()`, `attributes()`, `__getitem__()` |
| `QgsFeatureRequest` | Filter/optimize feature queries | `setFilterExpression()`, `setFilterRect()`, `setLimit()`, `setFlags()` |
| `QgsGeometry` | Geometry wrapper with GEOS operations | `fromPointXY()`, `buffer()`, `intersection()`, `contains()` |
| `QgsSpatialIndex` | R-tree spatial index | `addFeature()`, `nearestNeighbor()`, `intersects()` |
| `QgsTask` | Background processing | `run()`, `finished()`, `setProgress()`, `isCanceled()` |
| `QgsMessageLog` | Log panel messages | `logMessage(msg, tag, level)` |
| `QgsMessageBar` | Map canvas notifications | `pushMessage(title, text, level, duration)` |

### QgsPoint vs QgsPointXY

| Class | Dimensions | Use With |
|-------|-----------|----------|
| `QgsPointXY` | 2D (X, Y) only | `fromPointXY()`, `fromPolylineXY()`, `fromPolygonXY()` |
| `QgsPoint` | 2D + Z + M | `fromPolyline()`, 3D operations |

ALWAYS use `QgsPointXY` for 2D operations. Use `QgsPoint` ONLY when Z or M values are required.

---

## Critical Warnings

**NEVER** modify features outside an edit session. Changes are silently lost or corrupt the data source. ALWAYS use `with edit(layer):` or manually call `startEditing()` / `commitChanges()`.

**NEVER** access `iface`, `QgsProject.instance()`, or any Qt widget from `QgsTask.run()`. These are main-thread objects. Accessing them from a background thread crashes QGIS.

**NEVER** raise exceptions in `QgsTask.run()`. ALWAYS catch exceptions internally and return `False` on failure.

**NEVER** use `layer.featureCount()` to check if features exist. Some providers return -1. ALWAYS use `getFeatures()` with `setLimit(1)` instead.

**NEVER** use `print()` in multithreaded code (expression functions, renderers, processing algorithms). ALWAYS use `QgsMessageLog` instead.

**ALWAYS** check for NULL geometries before operations: `if not geom.isNull():`.

**ALWAYS** use `QgsDistanceArea` for measurements on geographic (lat/lon) CRS. The simple `area()` and `length()` methods return values in layer units without CRS correction.

**ALWAYS** call `layer.updateFields()` after adding or removing fields via the data provider.

**ALWAYS** keep `QgsProcessingContext` and `QgsProcessingFeedback` alive for the duration of `QgsProcessingAlgRunnerTask`. Garbage collection crashes QGIS.

---

## Decision Tree: Scripting Context

```
Need to write PyQGIS code?
|
+-- Running inside QGIS GUI?
|   |
|   +-- Quick test or exploration? --> Python Console (iface available)
|   +-- Reusable geoprocessing? --> Processing Script (QgsProcessingAlgorithm)
|   +-- GUI extension with toolbar? --> Plugin (classFactory + initGui/unload)
|
+-- Running outside QGIS?
    --> Standalone Script (QgsApplication init required)
```

## Decision Tree: Feature Editing

```
Need to modify layer data?
|
+-- Bulk import / batch processing, no undo needed?
|   --> Data Provider direct: layer.dataProvider().addFeatures([...])
|
+-- Interactive editing, undo/redo needed?
|   --> Edit Buffer: with edit(layer): layer.addFeature(feat)
|
+-- Multiple edits as single undo operation?
    --> Edit Commands: layer.beginEditCommand("description")
```

## Decision Tree: Spatial Queries

```
Need to find features by location?
|
+-- Single query against a layer?
|   --> QgsFeatureRequest().setFilterRect(extent)
|
+-- Repeated queries against same dataset?
|   |
|   +-- Point data only?
|   |   --> QgsSpatialIndexKDBush (fastest, static)
|   |
|   +-- Mixed geometry types?
|       --> QgsSpatialIndex (R-tree, dynamic)
```

---

## Essential Patterns

### Standalone Script Initialization

```python
from qgis.core import QgsApplication, QgsVectorLayer

qgs = QgsApplication([], False)
qgs.initQgis()

# ... PyQGIS work here ...

qgs.exitQgis()  # ALWAYS clean up
```

### Dual-Context Script (Console + Standalone)

```python
try:
    from qgis.utils import iface
    layer = iface.activeLayer()
except ImportError:
    from qgis.core import QgsApplication
    qgs = QgsApplication([], False)
    qgs.initQgis()
```

### Feature Iteration with Optimization

```python
from qgis.core import QgsFeatureRequest

# Attributes only (skip geometry loading)
request = QgsFeatureRequest().setFlags(QgsFeatureRequest.NoGeometry)
request.setSubsetOfAttributes(['name', 'population'], layer.fields())

for feature in layer.getFeatures(request):
    print(feature['name'], feature['population'])
```

### Feature Editing with Context Manager

```python
from qgis.core import QgsFeature, QgsGeometry, QgsPointXY, edit

with edit(layer):
    feat = QgsFeature(layer.fields())
    feat.setAttributes([1, 'New Point'])
    feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(10.0, 52.0)))
    layer.addFeature(feat)
# commitChanges() called automatically; rollBack() on exception
```

### Geometry Construction and Operations

```python
from qgis.core import QgsGeometry, QgsPointXY

# Create geometries
point = QgsGeometry.fromPointXY(QgsPointXY(1.0, 2.0))
polygon = QgsGeometry.fromPolygonXY([[
    QgsPointXY(0, 0), QgsPointXY(2, 0),
    QgsPointXY(2, 2), QgsPointXY(0, 2), QgsPointXY(0, 0)
]])

# Predicates
if polygon.contains(point):
    print("Point is inside polygon")

# Operations
buffered = point.buffer(10.0, 5)
intersection = polygon.intersection(buffered)
```

### Spatial Index Usage

```python
from qgis.core import QgsSpatialIndex, QgsPointXY, QgsFeatureRequest

# Build index (bulk load is fastest)
index = QgsSpatialIndex(layer.getFeatures())

# Query nearest 5 features
nearest_ids = index.nearestNeighbor(QgsPointXY(25.4, 12.7), 5)

# Fetch actual features by ID
for fid in nearest_ids:
    feature = next(layer.getFeatures(QgsFeatureRequest().setFilterFid(fid)))
```

### Background Task (QgsTask Subclass)

```python
from qgis.core import QgsTask, QgsApplication, QgsMessageLog, Qgis

class ProcessingTask(QgsTask):
    def __init__(self, description, data):
        super().__init__(description, QgsTask.CanCancel)
        self.data = data  # Copy data BEFORE task starts
        self.result = None
        self.exception = None

    def run(self):
        """Background thread. NEVER access iface or QgsProject here."""
        try:
            for i, item in enumerate(self.data):
                if self.isCanceled():
                    return False
                self.result = process(item)
                self.setProgress((i + 1) / len(self.data) * 100)
            return True
        except Exception as e:
            self.exception = e
            return False

    def finished(self, result):
        """Main thread. Safe for GUI updates."""
        if result:
            QgsMessageLog.logMessage("Done", 'MyPlugin', Qgis.Success)
        elif self.exception:
            QgsMessageLog.logMessage(str(self.exception), 'MyPlugin', Qgis.Critical)

task = ProcessingTask("Heavy work", data_copy)
QgsApplication.taskManager().addTask(task)
```

### User Communication

```python
from qgis.core import QgsMessageLog, Qgis

# Log panel (always available, including background threads)
QgsMessageLog.logMessage("Info message", 'MyPlugin', Qgis.Info)
QgsMessageLog.logMessage("Warning", 'MyPlugin', Qgis.Warning)
QgsMessageLog.logMessage("Error", 'MyPlugin', Qgis.Critical)

# Message bar (main thread + iface only)
iface.messageBar().pushMessage("Title", "Text", level=Qgis.Success, duration=3)
```

---

## Common Operations

### Check If Layer Has Features

```python
request = QgsFeatureRequest().setLimit(1)
has_features = bool(list(layer.getFeatures(request)))
```

### Get Single Feature by ID

```python
feature = next(layer.getFeatures(QgsFeatureRequest().setFilterFid(42)))
```

### Add Fields to Layer

```python
from qgis.PyQt.QtCore import QMetaType
from qgis.core import QgsField

layer.dataProvider().addAttributes([
    QgsField("name", QMetaType.Type.QString),
    QgsField("value", QMetaType.Type.Double),
])
layer.updateFields()  # ALWAYS call after field changes
```

### Ellipsoid-Accurate Measurements

```python
from qgis.core import QgsDistanceArea, QgsUnitTypes

d = QgsDistanceArea()
d.setEllipsoid('WGS84')

area_m2 = d.measureArea(geom)
area_km2 = d.convertAreaMeasurement(area_m2, QgsUnitTypes.AreaSquareKilometers)
```

### Coordinate Transformation

```python
from qgis.core import QgsCoordinateTransform, QgsCoordinateReferenceSystem, QgsProject

transform = QgsCoordinateTransform(
    QgsCoordinateReferenceSystem("EPSG:4326"),
    QgsCoordinateReferenceSystem("EPSG:3857"),
    QgsProject.instance()
)
geom.transform(transform)  # In-place transformation
```

### Basic Symbology

```python
from qgis.core import QgsMarkerSymbol

symbol = QgsMarkerSymbol.createSimple({'name': 'circle', 'color': 'red', 'size': '3'})
layer.renderer().setSymbol(symbol)
layer.triggerRepaint()
```

### Renderer Types Overview

| Renderer | Class | Use Case |
|----------|-------|----------|
| Single Symbol | `QgsSingleSymbolRenderer` | All features same style |
| Categorized | `QgsCategorizedSymbolRenderer` | Discrete attribute values |
| Graduated | `QgsGraduatedSymbolRenderer` | Numeric ranges |
| Rule-Based | `QgsRuleBasedRenderer` | Expression-driven rules |

---

## Data Provider vs Edit Buffer

| Aspect | Data Provider | Edit Buffer |
|--------|--------------|-------------|
| Method | `layer.dataProvider().addFeatures()` | `layer.addFeature()` |
| Undo support | No | Yes |
| Edit session required | No | Yes |
| Performance | Faster for bulk operations | Slower, safer |
| Use case | Batch processing, scripts | Interactive editing |

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for QgsFeature, QgsFeatureRequest, QgsGeometry, QgsSpatialIndex, QgsTask
- [references/examples.md](references/examples.md) -- Working code examples for all major patterns
- [references/anti-patterns.md](references/anti-patterns.md) -- Editing, threading, and iteration anti-patterns

### Official Sources

- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/vector.html
- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/geometry.html
- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/tasks.html
- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/communicating.html
