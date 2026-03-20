# qgis-syntax-pyqgis-api — Working Examples

> Complete, copy-paste-ready code examples for all major PyQGIS patterns.

---

## 1. Standalone Script Template

```python
from qgis.core import QgsApplication, QgsVectorLayer, QgsProject

# Initialize QGIS (no GUI)
qgs = QgsApplication([], False)
qgs.initQgis()

# Load a layer
layer = QgsVectorLayer("/path/to/data.gpkg|layername=points", "points", "ogr")
if not layer.isValid():
    print("Layer failed to load")
    qgs.exitQgis()
    exit(1)

# Work with the layer
for feature in layer.getFeatures():
    print(feature.id(), feature['name'])

# ALWAYS clean up
qgs.exitQgis()
```

---

## 2. Feature Access Patterns

### Iterate All Features

```python
for feature in layer.getFeatures():
    fid = feature.id()
    name = feature['name']
    geom = feature.geometry()
    if not geom.isNull():
        point = geom.asPoint()
        print(f"Feature {fid}: {name} at ({point.x()}, {point.y()})")
```

### Filtered Iteration with Expression

```python
from qgis.core import QgsFeatureRequest

request = QgsFeatureRequest().setFilterExpression('"population" > 50000')
request.setSubsetOfAttributes(['name', 'population'], layer.fields())

for feature in layer.getFeatures(request):
    print(f"{feature['name']}: {feature['population']}")
```

### Attributes-Only Iteration (No Geometry)

```python
from qgis.core import QgsFeatureRequest

request = QgsFeatureRequest()
request.setFlags(QgsFeatureRequest.NoGeometry)
request.setSubsetOfAttributes(['name', 'type'], layer.fields())

for feature in layer.getFeatures(request):
    print(feature['name'], feature['type'])
```

### Get Single Feature by ID

```python
from qgis.core import QgsFeatureRequest

feature = next(layer.getFeatures(QgsFeatureRequest().setFilterFid(42)))
print(feature['name'])
```

### Bounding Box Query

```python
from qgis.core import QgsFeatureRequest, QgsRectangle

bbox = QgsRectangle(4.0, 51.0, 5.0, 52.0)  # xmin, ymin, xmax, ymax
request = QgsFeatureRequest().setFilterRect(bbox)

for feature in layer.getFeatures(request):
    print(feature.id(), feature['name'])
```

### Check If Layer Has Features

```python
from qgis.core import QgsFeatureRequest

request = QgsFeatureRequest().setLimit(1)
has_features = bool(list(layer.getFeatures(request)))
```

### List All Fields

```python
for field in layer.fields():
    print(f"{field.name()} ({field.typeName()})")
```

---

## 3. Feature Editing

### Add Features with Context Manager

```python
from qgis.core import QgsFeature, QgsGeometry, QgsPointXY, edit

with edit(layer):
    for i in range(10):
        feat = QgsFeature(layer.fields())
        feat.setAttributes([i, f'Point {i}'])
        feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(i * 1.0, 52.0)))
        layer.addFeature(feat)
```

### Modify Existing Feature Attributes

```python
from qgis.core import QgsFeatureRequest, edit

with edit(layer):
    for feature in layer.getFeatures():
        feature['status'] = 'processed'
        layer.updateFeature(feature)
```

### Delete Features by Expression

```python
from qgis.core import QgsFeatureRequest, edit

request = QgsFeatureRequest().setFilterExpression('"status" = \'obsolete\'')
request.setFlags(QgsFeatureRequest.NoGeometry)

with edit(layer):
    for feature in layer.getFeatures(request):
        layer.deleteFeature(feature.id())
```

### Bulk Add via Data Provider (No Undo, Faster)

```python
from qgis.core import QgsFeature, QgsGeometry, QgsPointXY

features = []
for i in range(1000):
    feat = QgsFeature(layer.fields())
    feat.setAttributes([i, f'Bulk {i}'])
    feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(i * 0.01, 52.0)))
    features.append(feat)

success, added = layer.dataProvider().addFeatures(features)
if success:
    layer.updateExtents()
```

### Add Fields to Layer

```python
from qgis.PyQt.QtCore import QMetaType
from qgis.core import QgsField

layer.dataProvider().addAttributes([
    QgsField("category", QMetaType.Type.QString),
    QgsField("score", QMetaType.Type.Double),
])
layer.updateFields()  # ALWAYS call after field changes
```

### Remove Fields from Layer

```python
# Find field index
idx = layer.fields().indexFromName('obsolete_field')
if idx >= 0:
    layer.dataProvider().deleteAttributes([idx])
    layer.updateFields()
```

### Edit Commands (Single Undo Operation)

```python
layer.startEditing()
layer.beginEditCommand("Batch update scores")

try:
    for feature in layer.getFeatures():
        layer.changeAttributeValue(feature.id(), layer.fields().indexFromName('score'), 99.5)
    layer.endEditCommand()
except Exception:
    layer.destroyEditCommand()

layer.commitChanges()
```

### Check Provider Capabilities

```python
from qgis.core import QgsVectorDataProvider

caps = layer.dataProvider().capabilities()
can_delete = bool(caps & QgsVectorDataProvider.DeleteFeatures)
can_add = bool(caps & QgsVectorDataProvider.AddFeatures)
can_modify_attr = bool(caps & QgsVectorDataProvider.ChangeAttributeValues)
can_modify_geom = bool(caps & QgsVectorDataProvider.ChangeGeometries)

print(f"Delete: {can_delete}, Add: {can_add}, ModifyAttr: {can_modify_attr}, ModifyGeom: {can_modify_geom}")
```

---

## 4. Geometry Operations

### Construct All Geometry Types

```python
from qgis.core import QgsGeometry, QgsPointXY

# Point
point = QgsGeometry.fromPointXY(QgsPointXY(5.0, 52.0))

# LineString
line = QgsGeometry.fromPolylineXY([
    QgsPointXY(0, 0), QgsPointXY(1, 1), QgsPointXY(2, 0)
])

# Polygon (exterior ring, closed)
polygon = QgsGeometry.fromPolygonXY([[
    QgsPointXY(0, 0), QgsPointXY(10, 0),
    QgsPointXY(10, 10), QgsPointXY(0, 10), QgsPointXY(0, 0)
]])

# Polygon with hole
polygon_hole = QgsGeometry.fromPolygonXY([
    [QgsPointXY(0, 0), QgsPointXY(10, 0), QgsPointXY(10, 10), QgsPointXY(0, 10), QgsPointXY(0, 0)],
    [QgsPointXY(2, 2), QgsPointXY(4, 2), QgsPointXY(4, 4), QgsPointXY(2, 4), QgsPointXY(2, 2)]
])

# From WKT
from_wkt = QgsGeometry.fromWkt('MULTIPOINT(0 0, 1 1, 2 2)')
```

### Spatial Predicates

```python
from qgis.core import QgsGeometry, QgsPointXY

polygon = QgsGeometry.fromPolygonXY([[
    QgsPointXY(0, 0), QgsPointXY(10, 0),
    QgsPointXY(10, 10), QgsPointXY(0, 10), QgsPointXY(0, 0)
]])
point_inside = QgsGeometry.fromPointXY(QgsPointXY(5, 5))
point_outside = QgsGeometry.fromPointXY(QgsPointXY(15, 15))

print(polygon.contains(point_inside))    # True
print(polygon.contains(point_outside))   # False
print(polygon.intersects(point_inside))  # True
print(point_inside.within(polygon))      # True
print(polygon.disjoint(point_outside))   # True
```

### Buffer and Set Operations

```python
from qgis.core import QgsGeometry, QgsPointXY

point = QgsGeometry.fromPointXY(QgsPointXY(5, 5))
buffered = point.buffer(2.0, 20)  # 2.0 distance, 20 segments

polygon1 = QgsGeometry.fromWkt('POLYGON((0 0, 5 0, 5 5, 0 5, 0 0))')
polygon2 = QgsGeometry.fromWkt('POLYGON((3 3, 8 3, 8 8, 3 8, 3 3))')

union = polygon1.combine(polygon2)
intersection = polygon1.intersection(polygon2)
difference = polygon1.difference(polygon2)
sym_diff = polygon1.symDifference(polygon2)
hull = union.convexHull()
centroid = union.centroid()
```

### Geometry Validation

```python
geom = feature.geometry()

if geom.isNull():
    print("NULL geometry -- skip")
elif not geom.isGeosValid():
    errors = geom.validateGeometry()
    for error in errors:
        print(f"Validation error: {error.what()} at {error.where()}")
else:
    # Safe to perform spatial operations
    area = geom.area()
```

### Iterate Multi-Part Geometry

```python
geom = QgsGeometry.fromWkt('MULTIPOINT(0 0, 1 1, 2 2)')
for part in geom.parts():
    print(part.asWkt())
```

### WKT/WKB Round-Trip

```python
from qgis.core import QgsGeometry

# Export
original = QgsGeometry.fromWkt('POLYGON((0 0, 1 0, 1 1, 0 1, 0 0))')
wkt_str = original.asWkt()
wkb_bytes = original.asWkb()

# Import
from_wkt = QgsGeometry.fromWkt(wkt_str)
from_wkb = QgsGeometry.fromWkb(bytes(wkb_bytes))
```

---

## 5. Ellipsoid-Based Measurements

```python
from qgis.core import QgsDistanceArea, QgsUnitTypes, QgsGeometry, QgsPointXY

d = QgsDistanceArea()
d.setEllipsoid('WGS84')

# Area measurement (geographic CRS)
polygon = QgsGeometry.fromPolygonXY([[
    QgsPointXY(4.0, 51.0), QgsPointXY(5.0, 51.0),
    QgsPointXY(5.0, 52.0), QgsPointXY(4.0, 52.0), QgsPointXY(4.0, 51.0)
]])

area_m2 = d.measureArea(polygon)
area_km2 = d.convertAreaMeasurement(area_m2, QgsUnitTypes.AreaSquareKilometers)
print(f"Area: {area_km2:.2f} km2")

# Distance between two points
dist_m = d.measureLine(QgsPointXY(4.0, 51.0), QgsPointXY(5.0, 52.0))
dist_km = d.convertLengthMeasurement(dist_m, QgsUnitTypes.DistanceKilometers)
print(f"Distance: {dist_km:.2f} km")
```

---

## 6. Coordinate Transformation

```python
from qgis.core import (QgsCoordinateTransform, QgsCoordinateReferenceSystem,
                        QgsProject, QgsGeometry, QgsPointXY)

# WGS84 to Web Mercator
transform = QgsCoordinateTransform(
    QgsCoordinateReferenceSystem("EPSG:4326"),
    QgsCoordinateReferenceSystem("EPSG:3857"),
    QgsProject.instance()
)

geom = QgsGeometry.fromPointXY(QgsPointXY(5.0, 52.0))
geom.transform(transform)  # In-place
print(geom.asPoint())  # Now in EPSG:3857 coordinates
```

---

## 7. Spatial Index

### Build and Query

```python
from qgis.core import QgsSpatialIndex, QgsFeatureRequest, QgsPointXY, QgsRectangle

# Build index (bulk load)
index = QgsSpatialIndex(layer.getFeatures())

# Find 5 nearest features to a point
target = QgsPointXY(5.0, 52.0)
nearest_ids = index.nearestNeighbor(target, 5)

for fid in nearest_ids:
    feature = next(layer.getFeatures(QgsFeatureRequest().setFilterFid(fid)))
    print(f"Feature {fid}: {feature['name']}")

# Find features in bounding box
bbox = QgsRectangle(4.0, 51.0, 6.0, 53.0)
intersecting_ids = index.intersects(bbox)
print(f"Found {len(intersecting_ids)} features in bbox")
```

### Spatial Join Pattern

```python
from qgis.core import QgsSpatialIndex, QgsFeatureRequest

# Build index on target layer
target_index = QgsSpatialIndex(target_layer.getFeatures())

# For each source feature, find intersecting targets
for source_feat in source_layer.getFeatures():
    bbox = source_feat.geometry().boundingBox()
    candidate_ids = target_index.intersects(bbox)

    for cid in candidate_ids:
        target_feat = next(target_layer.getFeatures(QgsFeatureRequest().setFilterFid(cid)))
        # Exact geometry test (index only tests bounding boxes)
        if source_feat.geometry().intersects(target_feat.geometry()):
            print(f"Source {source_feat.id()} intersects Target {cid}")
```

---

## 8. Background Task (QgsTask Subclass)

```python
from qgis.core import QgsTask, QgsApplication, QgsMessageLog, Qgis

class BufferTask(QgsTask):
    def __init__(self, description, features, distance):
        super().__init__(description, QgsTask.CanCancel)
        self.features = features  # Pre-copied data
        self.distance = distance
        self.results = []
        self.exception = None

    def run(self):
        try:
            total = len(self.features)
            for i, feat in enumerate(self.features):
                if self.isCanceled():
                    return False
                geom = feat.geometry()
                if not geom.isNull():
                    buffered = geom.buffer(self.distance, 20)
                    self.results.append((feat.id(), buffered))
                self.setProgress((i + 1) / total * 100)
            return True
        except Exception as e:
            self.exception = e
            return False

    def finished(self, result):
        if result:
            QgsMessageLog.logMessage(
                f"Buffered {len(self.results)} features",
                'BufferPlugin', Qgis.Success
            )
        elif self.exception:
            QgsMessageLog.logMessage(
                f"Buffer failed: {self.exception}",
                'BufferPlugin', Qgis.Critical
            )

# Copy features before creating task
features_copy = [f for f in layer.getFeatures()]
task = BufferTask("Buffering features", features_copy, 100.0)
QgsApplication.taskManager().addTask(task)
```

---

## 9. Background Task from Function

```python
from qgis.core import QgsTask, QgsApplication

def calculate_stats(task, values):
    total = len(values)
    running_sum = 0
    for i, v in enumerate(values):
        if task.isCanceled():
            return None
        running_sum += v
        task.setProgress((i + 1) / total * 100)
    return {'mean': running_sum / total, 'count': total}

def on_complete(exception, result=None):
    if exception is None and result is not None:
        print(f"Mean: {result['mean']}, Count: {result['count']}")
    elif exception:
        print(f"Error: {exception}")

task = QgsTask.fromFunction(
    'Calculate statistics',
    calculate_stats,
    on_finished=on_complete,
    values=[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
)
QgsApplication.taskManager().addTask(task)
```

---

## 10. Task Dependencies

```python
from qgis.core import QgsTask, QgsApplication

task_a = QgsTask.fromFunction('Step A', step_a_func, on_finished=on_a_done, data=data_a)
task_b = QgsTask.fromFunction('Step B', step_b_func, on_finished=on_b_done, data=data_b)

# task_a depends on task_b completing first
task_a.addSubTask(task_b, [], QgsTask.ParentDependsOnSubTask)
QgsApplication.taskManager().addTask(task_a)
```

---

## 11. User Communication

### Log Messages

```python
from qgis.core import QgsMessageLog, Qgis

QgsMessageLog.logMessage("Processing started", 'MyPlugin', Qgis.Info)
QgsMessageLog.logMessage("Unusual value detected", 'MyPlugin', Qgis.Warning)
QgsMessageLog.logMessage("File not found", 'MyPlugin', Qgis.Critical)
QgsMessageLog.logMessage("All features processed", 'MyPlugin', Qgis.Success)
```

### Message Bar (Main Thread Only)

```python
from qgis.core import Qgis

# Auto-dismiss after 3 seconds
iface.messageBar().pushMessage("Done", "Processed 500 features", level=Qgis.Success, duration=3)

# Persistent error (stays until dismissed)
iface.messageBar().pushMessage("Error", "Layer failed to load", level=Qgis.Critical)
```

### Message Bar with Button

```python
from qgis.PyQt.QtWidgets import QPushButton
from qgis.core import Qgis

widget = iface.messageBar().createMessage("Alert", "Missing CRS definition")
button = QPushButton(widget)
button.setText("Set CRS")
button.pressed.connect(set_crs_function)
widget.layout().addWidget(button)
iface.messageBar().pushWidget(widget, Qgis.Warning)
```

### Progress Bar in Message Bar

```python
from qgis.PyQt.QtWidgets import QProgressBar
from qgis.core import Qgis

msg = iface.messageBar().createMessage("Processing...")
progress = QProgressBar()
progress.setMaximum(100)
msg.layout().addWidget(progress)
iface.messageBar().pushWidget(msg, Qgis.Info)

for i in range(100):
    progress.setValue(i + 1)
    # ... do work ...

iface.messageBar().clearWidgets()
```

---

## 12. Basic Symbology

### Single Symbol

```python
from qgis.core import QgsMarkerSymbol

symbol = QgsMarkerSymbol.createSimple({
    'name': 'circle',
    'color': '#ff0000',
    'size': '4',
    'outline_color': '#000000',
    'outline_width': '0.5'
})
layer.renderer().setSymbol(symbol)
layer.triggerRepaint()
```

### Categorized Renderer

```python
from qgis.core import QgsCategorizedSymbolRenderer, QgsRendererCategory, QgsMarkerSymbol

categories = [
    QgsRendererCategory('residential', QgsMarkerSymbol.createSimple({'color': 'blue'}), 'Residential'),
    QgsRendererCategory('commercial', QgsMarkerSymbol.createSimple({'color': 'red'}), 'Commercial'),
    QgsRendererCategory('industrial', QgsMarkerSymbol.createSimple({'color': 'gray'}), 'Industrial'),
]

renderer = QgsCategorizedSymbolRenderer('landuse', categories)
layer.setRenderer(renderer)
layer.triggerRepaint()
```

### Graduated Renderer

```python
from qgis.core import (QgsGraduatedSymbolRenderer, QgsRendererRange,
                        QgsClassificationRange, QgsMarkerSymbol)

ranges = [
    QgsRendererRange(QgsClassificationRange('Low', 0, 1000),
                     QgsMarkerSymbol.createSimple({'color': 'green'})),
    QgsRendererRange(QgsClassificationRange('Medium', 1000, 10000),
                     QgsMarkerSymbol.createSimple({'color': 'orange'})),
    QgsRendererRange(QgsClassificationRange('High', 10000, 1000000),
                     QgsMarkerSymbol.createSimple({'color': 'red'})),
]

renderer = QgsGraduatedSymbolRenderer('population', ranges)
layer.setRenderer(renderer)
layer.triggerRepaint()
```

### Rule-Based Renderer

```python
from qgis.core import QgsRuleBasedRenderer, QgsMarkerSymbol

root = QgsRuleBasedRenderer.Rule(QgsMarkerSymbol())

rule_large = QgsRuleBasedRenderer.Rule(
    QgsMarkerSymbol.createSimple({'color': 'red', 'size': '5'}),
    filterExp='"population" > 100000',
    label='Large cities'
)
rule_small = QgsRuleBasedRenderer.Rule(
    QgsMarkerSymbol.createSimple({'color': 'blue', 'size': '2'}),
    filterExp='"population" <= 100000',
    label='Small cities'
)

root.appendChild(rule_large)
root.appendChild(rule_small)

renderer = QgsRuleBasedRenderer(root)
layer.setRenderer(renderer)
layer.triggerRepaint()
```

### Simple Labeling

```python
from qgis.core import QgsPalLayerSettings, QgsVectorLayerSimpleLabeling, QgsTextFormat
from qgis.PyQt.QtGui import QColor

settings = QgsPalLayerSettings()
settings.fieldName = 'name'
settings.enabled = True

text_format = QgsTextFormat()
text_format.setSize(10)
text_format.setColor(QColor('black'))

buffer = text_format.buffer()
buffer.setEnabled(True)
buffer.setSize(1.0)
buffer.setColor(QColor('white'))

settings.setFormat(text_format)

labeling = QgsVectorLayerSimpleLabeling(settings)
layer.setLabeling(labeling)
layer.setLabelsEnabled(True)
layer.triggerRepaint()
```
