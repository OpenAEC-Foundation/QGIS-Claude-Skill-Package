# qgis-syntax-pyqgis-api — Anti-Patterns

> What NOT to do in PyQGIS, organized by category. Every anti-pattern includes the mistake, why it fails, and the correct alternative.

---

## Feature Editing Anti-Patterns

### AP-001: Modifying Features Outside Edit Session

**WRONG:**
```python
feature = next(layer.getFeatures())
feature['name'] = 'Updated'
layer.updateFeature(feature)  # Silently fails -- no edit session active
```

**WHY:** `updateFeature()` requires an active edit session. Without one, changes are silently discarded. No error is raised.

**CORRECT:**
```python
from qgis.core import edit

with edit(layer):
    feature = next(layer.getFeatures())
    feature['name'] = 'Updated'
    layer.updateFeature(feature)
```

---

### AP-002: Forgetting to Commit or Rollback

**WRONG:**
```python
layer.startEditing()
layer.addFeature(feat)
# Script ends without commitChanges() or rollBack()
# Edit session stays open, blocking other operations
```

**WHY:** Orphaned edit sessions lock the layer and prevent data provider operations. If QGIS crashes, all uncommitted changes are lost.

**CORRECT:**
```python
from qgis.core import edit

with edit(layer):
    layer.addFeature(feat)
# Automatically calls commitChanges() on success, rollBack() on exception
```

---

### AP-003: Forgetting updateFields() After Field Changes

**WRONG:**
```python
from qgis.PyQt.QtCore import QMetaType
from qgis.core import QgsField

layer.dataProvider().addAttributes([QgsField("score", QMetaType.Type.Double)])
# Missing layer.updateFields()
# Layer's field list is now out of sync with the data provider
feature['score'] = 99.5  # KeyError -- layer does not know about 'score'
```

**WHY:** The layer caches its field list. After modifying fields via the data provider, the cache is stale until `updateFields()` is called.

**CORRECT:**
```python
layer.dataProvider().addAttributes([QgsField("score", QMetaType.Type.Double)])
layer.updateFields()  # Sync the layer's field cache
```

---

### AP-004: Using Data Provider for Interactive Edits

**WRONG:**
```python
# User expects undo/redo to work
layer.dataProvider().changeAttributeValues({fid: {0: 'new_value'}})
# No undo available -- changes are written directly to the data source
```

**WHY:** Data provider operations bypass the edit buffer. There is no undo/redo support. Use the edit buffer for interactive editing.

**CORRECT:**
```python
from qgis.core import edit

with edit(layer):
    layer.changeAttributeValue(fid, 0, 'new_value')
# Undo/redo now available via Edit menu
```

---

## Threading Anti-Patterns

### AP-005: Accessing iface from Background Thread

**WRONG:**
```python
class MyTask(QgsTask):
    def run(self):
        layer = iface.activeLayer()  # CRASH -- iface is main-thread only
        for f in layer.getFeatures():
            self.process(f)
        return True
```

**WHY:** `iface` is a `QgisInterface` instance bound to the main Qt event loop. Accessing it from a background thread causes segmentation faults or undefined behavior.

**CORRECT:**
```python
class MyTask(QgsTask):
    def __init__(self, description, features):
        super().__init__(description, QgsTask.CanCancel)
        self.features = features  # Data copied BEFORE task starts

    def run(self):
        for f in self.features:
            self.process(f)
        return True

# Copy data on main thread, then start task
features_copy = list(iface.activeLayer().getFeatures())
task = MyTask("Processing", features_copy)
QgsApplication.taskManager().addTask(task)
```

---

### AP-006: Accessing QgsProject.instance() from run()

**WRONG:**
```python
class MyTask(QgsTask):
    def run(self):
        project = QgsProject.instance()  # CRASH -- singleton is main-thread only
        layer = project.mapLayersByName('roads')[0]
        return True
```

**WHY:** `QgsProject.instance()` is a singleton managed on the main thread. Accessing it from a background thread causes race conditions and crashes.

**CORRECT:**
```python
class MyTask(QgsTask):
    def __init__(self, description, layer_data):
        super().__init__(description, QgsTask.CanCancel)
        self.layer_data = layer_data  # Pre-extracted data

# Extract data on main thread
layer = QgsProject.instance().mapLayersByName('roads')[0]
data = list(layer.getFeatures())
task = MyTask("Processing", data)
```

---

### AP-007: Raising Exceptions in QgsTask.run()

**WRONG:**
```python
class MyTask(QgsTask):
    def run(self):
        data = load_data()
        if data is None:
            raise ValueError("No data found")  # CRASHES QGIS
        return True
```

**WHY:** Unhandled exceptions in `run()` crash the QGIS application. The task framework does not catch exceptions from `run()`.

**CORRECT:**
```python
class MyTask(QgsTask):
    def __init__(self, description):
        super().__init__(description, QgsTask.CanCancel)
        self.exception = None

    def run(self):
        try:
            data = load_data()
            if data is None:
                self.exception = ValueError("No data found")
                return False
            return True
        except Exception as e:
            self.exception = e
            return False

    def finished(self, result):
        if self.exception:
            QgsMessageLog.logMessage(str(self.exception), 'MyPlugin', Qgis.Critical)
```

---

### AP-008: Passing Live Layer References to Tasks

**WRONG:**
```python
class MyTask(QgsTask):
    def __init__(self, layer):
        super().__init__("Task", QgsTask.CanCancel)
        self.layer = layer  # Live reference to main-thread object

    def run(self):
        for f in self.layer.getFeatures():  # Accessing main-thread object
            pass
        return True
```

**WHY:** Layer objects are tied to the main thread. Iterating them from a background thread causes crashes or data corruption.

**CORRECT:**
```python
# Copy features on main thread
features = list(layer.getFeatures())
task = MyTask("Task", features)
```

---

### AP-009: Garbage-Collected Context/Feedback Objects

**WRONG:**
```python
def run_buffer():
    context = QgsProcessingContext()
    feedback = QgsProcessingFeedback()
    task = QgsProcessingAlgRunnerTask(alg, params, context, feedback)
    QgsApplication.taskManager().addTask(task)
    # context and feedback go out of scope and are garbage collected
    # Task crashes when trying to access them
```

**WHY:** `QgsProcessingAlgRunnerTask` holds raw pointers to context and feedback. If Python garbage-collects them, the task accesses freed memory.

**CORRECT:**
```python
class TaskHolder:
    def __init__(self):
        self.context = QgsProcessingContext()
        self.feedback = QgsProcessingFeedback()
        self.task = QgsProcessingAlgRunnerTask(alg, params, self.context, self.feedback)
        QgsApplication.taskManager().addTask(self.task)

holder = TaskHolder()  # Keep reference alive
```

---

## Feature Iteration Anti-Patterns

### AP-010: Using featureCount() to Check for Features

**WRONG:**
```python
if layer.featureCount() > 0:  # Some providers return -1
    process_features(layer)
```

**WHY:** Some data providers (e.g., WFS, database-backed layers with filters) return -1 or an inaccurate count from `featureCount()`. This check silently skips valid data.

**CORRECT:**
```python
request = QgsFeatureRequest().setLimit(1)
has_features = bool(list(layer.getFeatures(request)))
if has_features:
    process_features(layer)
```

---

### AP-011: Loading All Features Without Filtering

**WRONG:**
```python
# Loads ALL features with ALL attributes and ALL geometries
for feature in layer.getFeatures():
    name = feature['name']
    print(name)
```

**WHY:** Loading full features (geometry + all attributes) when only one field is needed wastes memory and time. For large layers (millions of features), this causes out-of-memory errors.

**CORRECT:**
```python
request = QgsFeatureRequest()
request.setFlags(QgsFeatureRequest.NoGeometry)
request.setSubsetOfAttributes(['name'], layer.fields())

for feature in layer.getFeatures(request):
    print(feature['name'])
```

---

### AP-012: Using print() in Multithreaded Code

**WRONG:**
```python
class MyTask(QgsTask):
    def run(self):
        for f in self.features:
            print(f"Processing {f.id()}")  # Severe performance degradation
        return True
```

**WHY:** `print()` acquires the GIL and performs I/O, which is extremely slow in multithreaded contexts. In expression functions and renderers, this can freeze the entire application.

**CORRECT:**
```python
from qgis.core import QgsMessageLog, Qgis

class MyTask(QgsTask):
    def run(self):
        for f in self.features:
            QgsMessageLog.logMessage(f"Processing {f.id()}", 'MyPlugin', Qgis.Info)
        return True
```

---

## Geometry Anti-Patterns

### AP-013: Not Checking for NULL Geometry

**WRONG:**
```python
for feature in layer.getFeatures():
    area = feature.geometry().area()  # Crashes if geometry is NULL
```

**WHY:** Features can have NULL geometries (e.g., attribute-only records, incomplete imports). Calling methods on a NULL geometry raises an error or returns meaningless values.

**CORRECT:**
```python
for feature in layer.getFeatures():
    geom = feature.geometry()
    if not geom.isNull():
        area = geom.area()
```

---

### AP-014: Using area()/length() on Geographic CRS

**WRONG:**
```python
# Layer is in EPSG:4326 (WGS84, degrees)
area = geom.area()  # Returns area in square degrees -- meaningless
```

**WHY:** `area()` and `length()` return values in layer units. For geographic CRS (lat/lon), units are degrees, which are not meaningful measurements.

**CORRECT:**
```python
from qgis.core import QgsDistanceArea, QgsUnitTypes

d = QgsDistanceArea()
d.setEllipsoid('WGS84')
area_m2 = d.measureArea(geom)  # Returns square meters
area_km2 = d.convertAreaMeasurement(area_m2, QgsUnitTypes.AreaSquareKilometers)
```

---

### AP-015: Using QgsPoint When QgsPointXY Is Required

**WRONG:**
```python
from qgis.core import QgsGeometry, QgsPoint

# QgsPoint has Z/M, but fromPointXY expects QgsPointXY
geom = QgsGeometry.fromPointXY(QgsPoint(5.0, 52.0))  # Type mismatch
```

**WHY:** `fromPointXY()` expects a `QgsPointXY` (2D). Passing `QgsPoint` (which supports Z/M) may work in some cases but is semantically incorrect and may cause unexpected behavior.

**CORRECT:**
```python
from qgis.core import QgsGeometry, QgsPointXY

geom = QgsGeometry.fromPointXY(QgsPointXY(5.0, 52.0))
```

---

### AP-016: Not Validating Geometry Before Spatial Operations

**WRONG:**
```python
# Self-intersecting polygon from untrusted source
result = geom.intersection(other_geom)  # May return empty or incorrect geometry
```

**WHY:** Invalid geometries (self-intersections, unclosed rings) produce unpredictable results in GEOS operations. Results may be silently wrong.

**CORRECT:**
```python
if not geom.isGeosValid():
    geom = geom.makeValid()  # Fix geometry (QGIS 3.x+)

result = geom.intersection(other_geom)
```

---

## Performance Anti-Patterns

### AP-017: Not Using Spatial Index for Repeated Queries

**WRONG:**
```python
# O(n*m) complexity -- scans all features for every query
for source in source_layer.getFeatures():
    for target in target_layer.getFeatures():
        if source.geometry().intersects(target.geometry()):
            process(source, target)
```

**WHY:** Nested iteration is O(n*m). For layers with thousands of features, this takes minutes or hours.

**CORRECT:**
```python
from qgis.core import QgsSpatialIndex, QgsFeatureRequest

index = QgsSpatialIndex(target_layer.getFeatures())

for source in source_layer.getFeatures():
    bbox = source.geometry().boundingBox()
    candidates = index.intersects(bbox)
    for cid in candidates:
        target = next(target_layer.getFeatures(QgsFeatureRequest().setFilterFid(cid)))
        if source.geometry().intersects(target.geometry()):
            process(source, target)
```

---

### AP-018: Repeated Feature Lookups Without Index

**WRONG:**
```python
# Each nearestNeighbor call without index scans all features
for point in points:
    min_dist = float('inf')
    for target in target_layer.getFeatures():  # Full scan every time
        dist = point.geometry().distance(target.geometry())
        if dist < min_dist:
            min_dist = dist
```

**CORRECT:**
```python
from qgis.core import QgsSpatialIndex

index = QgsSpatialIndex(target_layer.getFeatures())
for point in points:
    nearest_id = index.nearestNeighbor(point.geometry().asPoint(), 1)[0]
```
