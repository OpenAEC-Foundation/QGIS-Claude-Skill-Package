# Vooronderzoek: PyQGIS API and Scripting Patterns

> Deep research document for the QGIS Claude Skill Package.
> Knowledge base for skills: pyqgis-api, expressions, processing-scripts, plugins.
> Sources: Official QGIS PyQGIS Developer Cookbook (docs.qgis.org/latest), PyQGIS API reference.

---

## 1. PyQGIS Scripting Context

PyQGIS code runs in four distinct contexts, each with different initialization requirements and available objects.

### Python Console in QGIS

The built-in Python console (Plugins > Python Console) provides immediate access to the running QGIS instance. Two objects are always available:

- **`iface`** — a `QgisInterface` instance providing access to the QGIS GUI, map canvas, active layers, menus, and toolbars.
- **`QgsProject.instance()`** — the singleton project instance with all loaded layers and settings.

```python
# Python Console — iface is pre-loaded
layer = iface.activeLayer()
canvas = iface.mapCanvas()
print(layer.name(), layer.featureCount())
```

### Standalone Scripts

Scripts running outside QGIS (e.g., from the command line or a CI pipeline) MUST initialize `QgsApplication` before using any PyQGIS classes. There is NO `iface` object available.

```python
from qgis.core import QgsApplication, QgsVectorLayer, QgsProject

# Initialize QGIS application (no GUI)
qgs = QgsApplication([], False)
qgs.initQgis()

# Now PyQGIS classes are available
layer = QgsVectorLayer("/path/to/data.gpkg|layername=points", "points", "ogr")
if not layer.isValid():
    print("Layer failed to load")

# ALWAYS clean up
qgs.exitQgis()
```

ALWAYS call `qgs.initQgis()` before creating any layers or using processing. ALWAYS call `qgs.exitQgis()` at script termination.

### Processing Scripts

Processing scripts use a different entry point via `QgsProcessingAlgorithm`. They receive a `QgsProcessingContext` and `QgsProcessingFeedback` object instead of `iface`. See the Processing section for details.

### Plugin Context

Plugins receive `iface` via `classFactory()` and store it as `self.iface`. The plugin lifecycle is managed by QGIS through `initGui()` and `unload()` methods.

### Script Runner Patterns

For scripts that need to work both inside and outside QGIS:

```python
try:
    from qgis.utils import iface
    # Running inside QGIS
    layer = iface.activeLayer()
except ImportError:
    # Running standalone
    from qgis.core import QgsApplication
    qgs = QgsApplication([], False)
    qgs.initQgis()
```

---

## 2. Feature Access and Iteration

### QgsFeature: Core Data Object

`QgsFeature` represents a single feature with geometry and attributes. Key access patterns:

```python
layer = iface.activeLayer()
for feature in layer.getFeatures():
    # Feature ID
    fid = feature.id()

    # Geometry (may be NULL — ALWAYS check)
    geom = feature.geometry()
    if not geom.isNull():
        point = geom.asPoint()

    # Attributes by name
    name = feature['name']

    # Attributes by index (faster than name-based access)
    value = feature[0]

    # All attributes as list
    attrs = feature.attributes()
```

### QgsFeatureRequest: Filtering and Optimization

`QgsFeatureRequest` controls which features are returned and what data is loaded. ALWAYS use it to limit data fetched from the provider.

```python
from qgis.core import QgsFeatureRequest, QgsExpression

# Filter by bounding box
request = QgsFeatureRequest().setFilterRect(areaOfInterest)

# Exact intersection (slower but precise)
request = QgsFeatureRequest().setFilterRect(areaOfInterest) \
                             .setFlags(QgsFeatureRequest.ExactIntersect)

# Limit number of features
request = QgsFeatureRequest().setLimit(2)

# Filter by feature ID
request = QgsFeatureRequest().setFilterFid(45)

# Filter by multiple feature IDs
request = QgsFeatureRequest().setFilterFids([45, 67, sketchy])

# Subset of attributes by index
request = QgsFeatureRequest().setSubsetOfAttributes([0, 2])

# Subset of attributes by name
request = QgsFeatureRequest().setSubsetOfAttributes(['name', 'id'], layer.fields())

# Skip geometry loading (faster when only attributes needed)
request = QgsFeatureRequest().setFlags(QgsFeatureRequest.NoGeometry)

# Expression-based filtering
exp = QgsExpression('location_name ILIKE \'%Lake%\'')
request = QgsFeatureRequest(exp)

# Alternative expression syntax
request = QgsFeatureRequest().setFilterExpression('"Class" = \'B52\' AND "Heading" > 10')
```

### Iteration Patterns

```python
# Standard iteration
for feature in layer.getFeatures():
    process(feature)

# Filtered iteration
request = QgsFeatureRequest().setFilterExpression('"population" > 10000')
for feature in layer.getFeatures(request):
    process(feature)

# Get single feature
feature = next(layer.getFeatures(QgsFeatureRequest().setFilterFid(42)))

# Check if features exist (preferred over featureCount)
request = QgsFeatureRequest().setLimit(1)
has_features = bool(list(layer.getFeatures(request)))
```

### Field Information Retrieval

```python
layer = QgsVectorLayer("data.gpkg|layername=airports", "Airports", "ogr")
for field in layer.fields():
    print(field.name(), field.typeName())

# Display field (primary identifier)
print(layer.displayField())
```

---

## 3. Feature Editing

### Edit Sessions

QGIS uses an edit buffer system. All modifications go through an edit session that must be explicitly committed or rolled back.

```python
# Manual edit session management
layer.startEditing()

# ... perform edits ...

if success:
    layer.commitChanges()  # Save to data provider
else:
    layer.rollBack()       # Discard all changes

# Check edit state
if layer.isEditable():
    print("Layer is in edit mode")
```

### Context Manager (Preferred)

The `edit()` context manager automatically calls `commitChanges()` on success and `rollBack()` on exception:

```python
from qgis.core import edit

with edit(layer):
    feat = next(layer.getFeatures())
    feat[0] = 5
    layer.updateFeature(feat)
# commitChanges() called automatically here
```

### Adding Features

```python
# Create a new feature
feat = QgsFeature(layer.fields())
feat.setAttributes([0, 'hello'])
feat.setAttribute('name', 'hello')
feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(123, 456)))

# Add via data provider (bypasses edit buffer — no undo)
(res, outFeats) = layer.dataProvider().addFeatures([feat])

# Add via edit buffer (supports undo/redo)
with edit(layer):
    layer.addFeature(feat)
```

### Modifying Attributes

```python
# Via data provider (direct, no undo)
fid = 100
attrs = {0: "hello", 1: 123}
layer.dataProvider().changeAttributeValues({fid: attrs})

# Via edit buffer (supports undo)
with edit(layer):
    layer.changeAttributeValue(fid, 0, "hello")
```

### Modifying Geometry

```python
# Via data provider
geom = QgsGeometry.fromPointXY(QgsPointXY(111, 222))
layer.dataProvider().changeGeometryValues({fid: geom})

# Via edit buffer
with edit(layer):
    layer.changeGeometry(fid, geom)
```

### Deleting Features

```python
# Via data provider
layer.dataProvider().deleteFeatures([5, 10])

# Via edit buffer
with edit(layer):
    layer.deleteFeature(5)
```

### Field Management

```python
from qgis.PyQt.QtCore import QMetaType
from qgis.core import QgsField

# Add fields
res = layer.dataProvider().addAttributes([
    QgsField("mytext", QMetaType.Type.QString),
    QgsField("myint", QMetaType.Type.Int),
    QgsField("mydouble", QMetaType.Type.Double)
])
layer.updateFields()  # ALWAYS call after adding/removing fields

# Remove fields
res = layer.dataProvider().deleteAttributes([0])
layer.updateFields()
```

### Edit Commands (Undo/Redo)

For complex edits that should be a single undo operation:

```python
layer.beginEditCommand("Feature triangulation")
try:
    # ... multiple editing operations ...
    layer.endEditCommand()
except:
    layer.destroyEditCommand()  # Discard all operations in this command
```

### Data Provider Direct Editing vs Edit Buffer

| Aspect | Data Provider | Edit Buffer |
|--------|--------------|-------------|
| Method | `layer.dataProvider().addFeatures()` | `layer.addFeature()` |
| Undo support | No | Yes |
| Edit session required | No | Yes |
| Performance | Faster for bulk operations | Slower, but safer |
| Use case | Batch processing, scripts | Interactive editing |

### Checking Provider Capabilities

```python
caps = layer.dataProvider().capabilities()
if caps & QgsVectorDataProvider.DeleteFeatures:
    print('The layer supports DeleteFeatures')

# Human-readable capabilities
caps_string = layer.dataProvider().capabilitiesString()
print(caps_string)
```

---

## 4. Geometry Operations

### QgsGeometry Construction

`QgsGeometry` is the main geometry class. It wraps GEOS geometry operations and provides construction methods:

```python
from qgis.core import QgsGeometry, QgsPointXY, QgsPoint

# Point
geom_point = QgsGeometry.fromPointXY(QgsPointXY(1.0, 2.0))

# Linestring
geom_line = QgsGeometry.fromPolylineXY([
    QgsPointXY(1.0, 1.0),
    QgsPointXY(2.0, 2.0),
    QgsPointXY(3.0, 1.0)
])

# Polygon (list of rings; first ring = exterior, subsequent = holes)
geom_polygon = QgsGeometry.fromPolygonXY([
    [QgsPointXY(1, 1), QgsPointXY(2, 1), QgsPointXY(2, 2),
     QgsPointXY(1, 2), QgsPointXY(1, 1)]  # exterior ring
])

# From WKT
geom_wkt = QgsGeometry.fromWkt('POINT(1.0 2.0)')

# From WKB
geom_wkb = QgsGeometry.fromWkb(wkb_bytes)
```

### QgsPoint vs QgsPointXY

- **`QgsPointXY`** — 2D only (X, Y). Used with `fromPointXY()`, `fromPolylineXY()`, `fromPolygonXY()`.
- **`QgsPoint`** — Supports M and Z dimensions (3D coordinates, measured geometries). Used with `fromPolyline()`.

ALWAYS use `QgsPointXY` for 2D operations. Use `QgsPoint` only when Z or M values are required.

### Geometry Type Identification

```python
geom = feature.geometry()

# Detailed WKB type (Point, LineString, Polygon, MultiPoint, etc.)
wkb_type = geom.wkbType()

# General geometry type
geom_type = geom.type()  # QgsWkbTypes.PointGeometry, LineGeometry, PolygonGeometry

# Human-readable name
print(geom.type().displayString())  # "Point", "LineString", "Polygon"

# Multi-part check
if geom.isMultipart():
    print("Multi-part geometry")
```

### Geometry Access Methods

```python
# Single-part geometries
point = geom.asPoint()           # Returns QgsPointXY
polyline = geom.asPolyline()     # Returns list of QgsPointXY
polygon = geom.asPolygon()       # Returns list of rings (list of QgsPointXY)

# Multi-part geometries
multi_point = geom.asMultiPoint()       # Returns list of QgsPointXY
multi_line = geom.asMultiPolyline()     # Returns list of polylines
multi_poly = geom.asMultiPolygon()      # Returns list of polygons
```

### Iterating Over Geometry Parts

```python
geom = QgsGeometry.fromWkt('MultiPoint(0 0, 1 1, 2 2)')
for part in geom.parts():
    print(part.asWkt())
```

### Geometry Predicates (GEOS-Based)

All predicates return `bool`:

```python
geom_a = QgsGeometry.fromWkt('POLYGON((0 0, 2 0, 2 2, 0 2, 0 0))')
geom_b = QgsGeometry.fromWkt('POINT(1 1)')

geom_a.contains(geom_b)     # True — B is inside A
geom_a.intersects(geom_b)   # True — geometries share space
geom_b.within(geom_a)       # True — B is within A
geom_a.touches(geom_b)      # Boundary contact only
geom_a.crosses(geom_b)      # Geometries cross each other
geom_a.overlaps(geom_b)     # Partial overlap (same dimension)
geom_a.disjoint(geom_b)     # No shared space
```

### Geometry Operations (GEOS-Based)

```python
# Buffer — creates polygon around geometry at given distance
buffered = geom.buffer(10.0, 5)  # distance=10, segments=5

# Set operations
union = geom_a.combine(geom_b)           # Union
intersection = geom_a.intersection(geom_b)
difference = geom_a.difference(geom_b)
sym_diff = geom_a.symDifference(geom_b)

# Convex hull
hull = geom.convexHull()

# Centroid
centroid = geom.centroid()
```

### Geometry Measurements

**Simple measurements** (do NOT account for CRS — use only for projected data):

```python
area = geom.area()       # Polygon area in layer units
length = geom.length()   # Line length / polygon perimeter in layer units
```

**Ellipsoid-based measurements** (accurate for geographic CRS):

```python
from qgis.core import QgsDistanceArea, QgsUnitTypes

d = QgsDistanceArea()
d.setEllipsoid('WGS84')

perimeter = d.measurePerimeter(geom)      # Returns meters
area = d.measureArea(geom)                # Returns square meters
distance = d.measureLine(point1, point2)  # Distance between points

# Unit conversion
area_km2 = d.convertAreaMeasurement(area, QgsUnitTypes.AreaSquareKilometers)
```

ALWAYS use `QgsDistanceArea` for measurements on geographic (lat/lon) data. Simple `area()` and `length()` are only accurate for projected CRS.

### Geometry Validation

```python
# GEOS validity check
if geom.isGeosValid():
    print("Geometry is valid")

# Detailed validation errors
errors = geom.validateGeometry()
for error in errors:
    print(error.what(), error.where())
```

### WKT/WKB Import/Export

```python
# Export
wkt_string = geom.asWkt()
wkb_bytes = geom.asWkb()

# Import
geom_from_wkt = QgsGeometry.fromWkt('LINESTRING(0 0, 1 1, 2 0)')
geom_from_wkb = QgsGeometry.fromWkb(wkb_bytes)
```

### QgsGeometry vs QgsAbstractGeometry

- **`QgsGeometry`** — high-level wrapper, provides GEOS operations, used in features and most PyQGIS code.
- **`QgsAbstractGeometry`** — low-level base class for native geometry types (`QgsPoint`, `QgsLineString`, `QgsPolygon`). Access via `geom.get()` or `geom.constGet()`.

```python
# Access the underlying QgsAbstractGeometry
abstract_geom = geom.constGet()
# Useful for accessing Z/M values, vertex iteration, etc.
```

### Coordinate Transformation

```python
from qgis.core import QgsCoordinateTransform, QgsCoordinateReferenceSystem

transform = QgsCoordinateTransform(
    QgsCoordinateReferenceSystem("EPSG:4326"),
    QgsCoordinateReferenceSystem("EPSG:3857"),
    QgsProject.instance()
)

# In-place transformation
geom.transform(transform)
```

---

## 5. Spatial Indexing

### QgsSpatialIndex

`QgsSpatialIndex` provides R-tree based spatial indexing for fast bounding box queries.

```python
from qgis.core import QgsSpatialIndex

# Build index incrementally
index = QgsSpatialIndex()
for feat in layer.getFeatures():
    index.addFeature(feat)

# Or bulk load from iterator (faster)
index = QgsSpatialIndex(layer.getFeatures())

# Nearest neighbor query — returns feature IDs
nearest_ids = index.nearestNeighbor(QgsPointXY(25.4, 12.7), 5)

# Bounding box intersection query — returns feature IDs
intersecting_ids = index.intersects(QgsRectangle(22.5, 15.3, 23.1, 17.2))
```

**Performance characteristics:**
- Build time: O(n log n) for bulk loading
- Query time: O(log n) for intersection and nearest neighbor
- Memory: proportional to number of features
- Returns feature IDs only — you must fetch features separately

### QgsSpatialIndexKDBush

A specialized index for point data only:
- Supports only single point features
- Static — cannot add features after creation
- Significantly faster than QgsSpatialIndex for point data
- Allows direct retrieval of original feature points (no second query needed)

---

## 6. Expression Engine

### QgsExpression: Parsing and Evaluation

```python
from qgis.core import QgsExpression, QgsExpressionContext, QgsExpressionContextUtils

# Parse expression (validates syntax)
exp = QgsExpression('1 + 1 = 2')
if exp.hasParserError():
    raise Exception(exp.parserErrorString())

# Simple evaluation (no context needed for literals)
result = exp.evaluate()  # Returns 1 (True)

# Check for evaluation errors
if exp.hasEvalError():
    raise ValueError(exp.evalErrorString())
```

### QgsExpressionContext: Setting Up Context

The context provides variables and scopes for expression evaluation. Scopes are stacked: global > project > layer > feature (most specific wins).

```python
from qgis.core import QgsExpressionContext, QgsExpressionContextUtils

context = QgsExpressionContext()

# Add all standard scopes at once (convenience method)
context.appendScopes(QgsExpressionContextUtils.globalProjectLayerScopes(layer))

# Or add individual scopes
context.appendScope(QgsExpressionContextUtils.globalScope())
context.appendScope(QgsExpressionContextUtils.projectScope(QgsProject.instance()))
context.appendScope(QgsExpressionContextUtils.layerScope(layer))
```

### Feature-Based Evaluation

```python
from qgis.core import QgsExpression, QgsExpressionContext, QgsExpressionContextUtils

exp = QgsExpression('"population" * 1.05')
context = QgsExpressionContext()
context.appendScopes(QgsExpressionContextUtils.globalProjectLayerScopes(layer))

for feature in layer.getFeatures():
    context.setFeature(feature)
    value = exp.evaluate(context)
    print(f"Feature {feature.id()}: {value}")
```

### Expression Syntax Reference

**Data types:**
- Numbers: `123`, `3.14`
- Strings: `'hello world'` (single quotes)
- Column references: `"field_name"` (double quotes)

**Operators:**
- Arithmetic: `+`, `-`, `*`, `/`, `^` (power)
- Comparison: `=`, `!=`, `>`, `>=`, `<`, `<=`
- Logical: `AND`, `OR`, `NOT`
- Pattern matching: `LIKE` (with `%` and `_`), `~` (regex)
- NULL checking: `IS NULL`, `IS NOT NULL`

**Built-in function categories:**
- Math: `sqrt()`, `sin()`, `cos()`, `tan()`, `asin()`, `acos()`, `atan()`
- Conversion: `to_int()`, `to_real()`, `to_string()`, `to_date()`
- Geometry: `$area`, `$length`, `$x`, `$y`, `$geometry`, `num_geometries()`, `centroid()`
- String: `upper()`, `lower()`, `length()`, `replace()`, `regexp_replace()`
- Aggregates: `aggregate()`, `sum()`, `count()`, `mean()`
- Date: `now()`, `day()`, `month()`, `year()`

### Field Calculator Pattern

```python
from qgis.core import edit, QgsExpression, QgsExpressionContext, QgsExpressionContextUtils

exp = QgsExpression('"population" * 2')
context = QgsExpressionContext()
context.appendScopes(QgsExpressionContextUtils.globalProjectLayerScopes(layer))

with edit(layer):
    for f in layer.getFeatures():
        context.setFeature(f)
        f['doubled_pop'] = exp.evaluate(context)
        layer.updateFeature(f)
```

### Expression-Based Filtering in QgsFeatureRequest

```python
# Method 1: Pass expression string
request = QgsFeatureRequest().setFilterExpression('"Test" >= 3')

# Method 2: Pass QgsExpression object
exp = QgsExpression('location_name ILIKE \'%Lake%\'')
request = QgsFeatureRequest(exp)

for f in layer.getFeatures(request):
    print(f['name'])
```

### Expression-Based Labeling

```python
from qgis.core import QgsPalLayerSettings, QgsVectorLayerSimpleLabeling

settings = QgsPalLayerSettings()
settings.fieldName = '"name" || \' (\' || "population" || \')\''
settings.isExpression = True
settings.enabled = True

# Apply text formatting
text_format = settings.format()
text_format.setSize(12)
settings.setFormat(text_format)

labeling = QgsVectorLayerSimpleLabeling(settings)
layer.setLabeling(labeling)
layer.setLabelsEnabled(True)
layer.triggerRepaint()
```

### Expression-Based Symbology (Data-Defined Properties)

Data-defined properties allow any symbology property to be driven by an expression:

```python
from qgis.core import QgsProperty

# Set symbol size based on expression
symbol = layer.renderer().symbol()
symbol.setDataDefinedSize(QgsProperty.fromExpression('"population" / 1000'))

# Set color based on expression
symbol.setDataDefinedColor(QgsProperty.fromExpression(
    "CASE WHEN \"type\" = 'city' THEN '#ff0000' ELSE '#0000ff' END"
))
```

### Custom Expression Functions: @qgsfunction Decorator

Custom functions extend the expression engine with Python code. They are registered globally and available in the expression builder. [VERIFY: The @qgsfunction decorator details were not found in the fetched cookbook pages. The following is based on well-established PyQGIS patterns.]

```python
from qgis.core import qgsfunction, QgsExpression

@qgsfunction(args='auto', group='Custom', referenced_columns=[])
def my_custom_function(value1, value2, feature, parent):
    """
    Calculate a custom metric.
    <p>Returns the product of two values plus the feature area.</p>
    """
    return value1 * value2 + feature.geometry().area()

# Register the function
QgsExpression.registerFunction(my_custom_function)

# Use in expressions: my_custom_function("field1", "field2")

# Unregister when done (e.g., in plugin unload)
QgsExpression.unregisterFunction('my_custom_function')
```

Key parameters of `@qgsfunction`:
- `args` — Number of arguments, or `'auto'` to detect from function signature
- `group` — Category name shown in the expression builder
- `referenced_columns` — List of field names the function accesses, or `[QgsFeatureRequest.ALL_ATTRIBUTES]`
- `usesgeometry` — Set to `True` if the function accesses `feature.geometry()`

---

## 7. Background Tasks and Threading

### QgsTask Subclassing

`QgsTask` is the base class for all background operations. Subclass it and implement `run()` and optionally `finished()` and `cancel()`.

```python
from qgis.core import QgsTask, QgsApplication, QgsMessageLog, Qgis

class HeavyProcessingTask(QgsTask):
    def __init__(self, description, layer_data):
        super().__init__(description, QgsTask.CanCancel)
        self.layer_data = layer_data  # Copy data BEFORE task starts
        self.result = None
        self.exception = None

    def run(self):
        """Runs in background thread. NEVER access GUI or QgsProject here."""
        try:
            total = len(self.layer_data)
            for i, item in enumerate(self.layer_data):
                if self.isCanceled():
                    return False

                # Process item...
                self.result = process(item)

                # Report progress (0-100)
                self.setProgress((i + 1) / total * 100)

            return True  # MUST return True on success
        except Exception as e:
            self.exception = e
            return False  # MUST return False on failure

    def finished(self, result):
        """Called on MAIN thread after run() completes. Safe for GUI access."""
        if self.exception:
            QgsMessageLog.logMessage(
                f"Task failed: {self.exception}",
                'MyPlugin', level=Qgis.Critical
            )
        elif result:
            QgsMessageLog.logMessage(
                "Task completed successfully",
                'MyPlugin', level=Qgis.Info
            )

    def cancel(self):
        QgsMessageLog.logMessage("Task was canceled", 'MyPlugin', level=Qgis.Warning)
        super().cancel()
```

**Critical rule:** The `run()` method MUST return `True` or `False`. NEVER raise exceptions in `run()` — it will crash QGIS.

### QgsTaskManager: Managing Tasks

```python
# Add task to the global task manager
task = HeavyProcessingTask("Processing features", data)
QgsApplication.taskManager().addTask(task)
```

### Task from Function (Simpler Pattern)

```python
def heavy_work(task, param1, param2):
    """Background function. Must accept 'task' as first parameter."""
    task.setProgress(50)
    if task.isCanceled():
        return None
    return {'result': param1 + param2}

def on_completion(exception, result=None):
    """Called on main thread."""
    if exception is None:
        print(f"Result: {result['result']}")
    else:
        print(f"Error: {exception}")

task = QgsTask.fromFunction(
    'My calculation',
    heavy_work,
    on_finished=on_completion,
    param1=10,
    param2=20
)
QgsApplication.taskManager().addTask(task)
```

### Task Dependencies

```python
parent_task = HeavyProcessingTask("Parent", data1)
child_task = HeavyProcessingTask("Child", data2)

# Parent depends on child completing first
parent_task.addSubTask(
    child_task,
    [],  # sub-task dependencies
    QgsTask.ParentDependsOnSubTask
)
QgsApplication.taskManager().addTask(parent_task)
```

### Layer Dependencies

```python
task.setDependentLayers([layer1, layer2])
# Task is automatically canceled if a dependent layer becomes unavailable
```

### Processing Algorithm as Task

```python
from qgis.core import (QgsProcessingContext, QgsProcessingFeedback,
                        QgsProcessingAlgRunnerTask, QgsApplication)

context = QgsProcessingContext()
context.setProject(QgsProject.instance())
feedback = QgsProcessingFeedback()

alg = QgsApplication.processingRegistry().algorithmById('native:buffer')
params = {'INPUT': layer, 'DISTANCE': 100, 'OUTPUT': 'memory:'}

task = QgsProcessingAlgRunnerTask(alg, params, context, feedback)
task.executed.connect(lambda ok, results: print("Done:", results))
QgsApplication.taskManager().addTask(task)
```

**Critical:** `context` and `feedback` objects MUST live at least as long as the task. If they are garbage collected, QGIS will crash.

### Threading Constraints — CRITICAL

| Rule | Details |
|------|---------|
| NEVER access `iface` from a background thread | `iface` is a GUI object bound to the main thread |
| NEVER access `QgsProject.instance()` from `run()` | The project singleton lives on the main thread |
| NEVER create or modify Qt widgets from `run()` | All Qt GUI operations are main-thread only |
| ALWAYS copy data before starting the task | Pass data as constructor arguments, not live references |
| ALWAYS use `finished()` for GUI updates | `finished()` runs on the main thread |
| ALWAYS check `isCanceled()` in loops | Allows responsive cancellation |

---

## 8. Communicating with User

### QgsMessageLog: Logging

Messages appear in the Log Messages Panel (View > Log Messages):

```python
from qgis.core import QgsMessageLog, Qgis

QgsMessageLog.logMessage("Something happened", 'MyPlugin', level=Qgis.Info)
QgsMessageLog.logMessage("Watch out", 'MyPlugin', level=Qgis.Warning)
QgsMessageLog.logMessage("Error occurred", 'MyPlugin', level=Qgis.Critical)
```

**Message levels** (`Qgis.MessageLevel`):
- `Qgis.Info` (0)
- `Qgis.Warning` (1)
- `Qgis.Critical` (2)
- `Qgis.Success` (3)

### QgsMessageBar: User-Facing Messages

Messages appear at the top of the map canvas:

```python
# Simple message
iface.messageBar().pushMessage("Title", "Message text", level=Qgis.Info)

# With auto-dismiss duration (seconds)
iface.messageBar().pushMessage("Success", "Operation complete",
                                level=Qgis.Success, duration=3)

# Critical message (stays until dismissed)
iface.messageBar().pushMessage("Error", "Something went wrong",
                                level=Qgis.Critical)
```

### Message Bar with Custom Widgets

```python
from qgis.PyQt.QtWidgets import QPushButton

widget = iface.messageBar().createMessage("Alert", "Action required")
button = QPushButton(widget)
button.setText("Fix it")
button.pressed.connect(fix_function)
widget.layout().addWidget(button)
iface.messageBar().pushWidget(widget, Qgis.Warning)
```

### Message Bar in Custom Dialogs

```python
from qgis.gui import QgsMessageBar
from qgis.PyQt.QtWidgets import QDialog, QSizePolicy

class MyDialog(QDialog):
    def __init__(self):
        QDialog.__init__(self)
        self.bar = QgsMessageBar()
        self.bar.setSizePolicy(QSizePolicy.Minimum, QSizePolicy.Fixed)
        self.layout().addWidget(self.bar)

    def show_error(self, message):
        self.bar.pushMessage("Error", message, level=Qgis.Critical)
```

### Progress Bars

```python
from qgis.PyQt.QtWidgets import QProgressBar

# Progress bar in message bar
progressMessageBar = iface.messageBar().createMessage("Processing...")
progress = QProgressBar()
progress.setMaximum(100)
progressMessageBar.layout().addWidget(progress)
iface.messageBar().pushWidget(progressMessageBar, Qgis.Info)

# Update progress
for i in range(100):
    progress.setValue(i + 1)

# Clean up
iface.messageBar().clearWidgets()
```

### Status Bar Messages

```python
iface.statusBarIface().showMessage("Processed 50%")
iface.statusBarIface().clearMessage()
```

### QgsProcessingFeedback

For processing scripts and algorithms:

```python
def processAlgorithm(self, parameters, context, feedback):
    feedback.pushInfo("Starting processing...")
    feedback.setProgress(0)

    for i in range(total):
        if feedback.isCanceled():
            break
        feedback.setProgress(int((i / total) * 100))
        feedback.pushInfo(f"Processing feature {i}")

    feedback.pushInfo("Complete")
    return results
```

### Performance Warning

NEVER use `print()` statements in multithreaded contexts (expression functions, renderers, processing algorithms). `print()` significantly degrades performance. ALWAYS use `QgsMessageLog` or Python's `logging` module instead.

---

## 9. Plugin Development

### Plugin Directory Structure

```
my_plugin/
├── __init__.py          # REQUIRED — contains classFactory()
├── metadata.txt         # REQUIRED — plugin metadata
├── mainPlugin.py        # Main plugin class
├── resources.qrc        # Qt resource definitions
├── resources.py          # Compiled resources (pyrcc5)
├── form.ui              # Qt Designer form (optional)
├── form.py              # Compiled form (optional)
├── icon.png             # Plugin icon
└── LICENSE              # REQUIRED for repository submission
```

### Plugin Installation Paths

- **User plugins**: `~/(UserProfile)/python/plugins`
- **System plugins (Linux/Mac)**: `(qgis_prefix)/share/qgis/python/plugins`
- **System plugins (Windows)**: `(qgis_prefix)/python/plugins`
- **Custom path**: Set `QGIS_PLUGINPATH` environment variable

### __init__.py

MUST contain the `classFactory()` function:

```python
def classFactory(iface):
    """Load the plugin class. Called by QGIS on plugin startup."""
    from .mainPlugin import MyPlugin
    return MyPlugin(iface)
```

### Main Plugin Class

Three methods are required: `__init__`, `initGui()`, and `unload()`.

```python
from qgis.PyQt.QtWidgets import QAction
from qgis.PyQt.QtGui import QIcon

class MyPlugin:
    def __init__(self, iface):
        """Store the QGIS interface reference."""
        self.iface = iface

    def initGui(self):
        """Called when plugin is activated. Create GUI elements."""
        self.action = QAction(
            QIcon(":/plugins/myplugin/icon.png"),
            "My Plugin",
            self.iface.mainWindow()
        )
        self.action.setObjectName("myPluginAction")
        self.action.triggered.connect(self.run)

        # Add to toolbar and menu
        self.iface.addToolBarIcon(self.action)
        self.iface.addPluginToMenu("&My Plugin", self.action)

    def unload(self):
        """Called when plugin is deactivated. Remove ALL GUI elements."""
        self.iface.removePluginMenu("&My Plugin", self.action)
        self.iface.removeToolBarIcon(self.action)

    def run(self):
        """Main plugin logic."""
        pass
```

### Menu Integration Methods

Place plugins in specific menus:

```python
# Generic plugin menu
self.iface.addPluginToMenu("&My Plugin", self.action)

# Category-specific menus
self.iface.addPluginToRasterMenu("&My Plugin", self.action)
self.iface.addPluginToVectorMenu("&My Plugin", self.action)
self.iface.addPluginToDatabaseMenu("&My Plugin", self.action)
self.iface.addPluginToWebMenu("&My Plugin", self.action)
```

### metadata.txt Format

```ini
[general]
name=My Plugin Name
qgisMinimumVersion=3.0
description=Short one-line description
about=Longer multi-line description explaining what the plugin does
version=1.0.0
author=Author Name
email=author@example.com
repository=https://github.com/author/my-plugin

# Optional fields
category=Vector
icon=icon.png
experimental=False
deprecated=False
tags=analysis,vector,geoprocessing
homepage=https://author.github.io/my-plugin
tracker=https://github.com/author/my-plugin/issues
changelog=First release
```

**Required fields:** name, qgisMinimumVersion, description, about, version, author, email, repository.

**Category options:** Raster, Vector, Database, Mesh, Web.

If `qgisMaximumVersion` is omitted, it defaults to `major.99` (e.g., `3.99` for `qgisMinimumVersion=3.0`).

### Resource Files

Qt resource files bundle icons and other assets:

```xml
<!-- resources.qrc -->
<RCC>
  <qresource prefix="/plugins/myplugin">
    <file>icon.png</file>
  </qresource>
</RCC>
```

Compile with: `pyrcc5 -o resources.py resources.qrc`

Then import in the plugin:

```python
from . import resources  # Makes resource paths available
```

### Publishing to QGIS Plugin Repository

1. Package plugin as ZIP: `plugin.zip` containing `pluginfolder/` with all files
2. Submit to https://plugins.qgis.org/ (requires OSGEO ID)
3. Staff approval required before publication
4. Plugin folder name: ASCII characters (A-Z, a-z), digits, underscore, minus only. NEVER start with a digit.
5. Version must be unique across submissions

### Plugin Builder Tool

Plugin Builder is a QGIS plugin that generates a complete plugin skeleton. Install it from the Plugin Manager. It creates:
- All required files (__init__.py, metadata.txt, main class)
- Qt Designer form template
- Resource file
- Makefile for compilation
- Test infrastructure

---

## 10. Symbology and Rendering

### QgsSymbol Hierarchy

- **`QgsMarkerSymbol`** — point geometries
- **`QgsLineSymbol`** — line geometries
- **`QgsFillSymbol`** — polygon geometries

Each symbol contains one or more symbol layers.

### Single Symbol Renderer

```python
from qgis.core import QgsMarkerSymbol

# Create simple marker
symbol = QgsMarkerSymbol.createSimple({
    'name': 'square',
    'color': 'red',
    'size': '3'
})
layer.renderer().setSymbol(symbol)
layer.triggerRepaint()
```

**Available marker shapes:** `circle`, `square`, `cross`, `rectangle`, `diamond`, `pentagon`, `triangle`, `equilateral_triangle`, `star`, `regular_star`, `arrow`, `filled_arrowhead`, `x`.

### Accessing Symbol Properties

```python
# Get properties dictionary
props = layer.renderer().symbol().symbolLayers()[0].properties()
print(props)

# Modify symbol layer
layer.renderer().symbol().symbolLayer(0).setSize(3)
layer.triggerRepaint()
```

### Symbol Layers

Each symbol can have multiple layers stacked on top of each other:

```python
marker_symbol = QgsMarkerSymbol()
for i in range(marker_symbol.symbolLayerCount()):
    lyr = marker_symbol.symbolLayer(i)
    print(f"{i}: {lyr.layerType()}")
```

### Categorized Renderer

Assigns different symbols based on discrete attribute values:

```python
from qgis.core import QgsCategorizedSymbolRenderer, QgsRendererCategory, QgsMarkerSymbol

categorized_renderer = QgsCategorizedSymbolRenderer()

cat1 = QgsRendererCategory('residential', QgsMarkerSymbol.createSimple({'color': 'blue'}), 'Residential')
cat2 = QgsRendererCategory('commercial', QgsMarkerSymbol.createSimple({'color': 'red'}), 'Commercial')

categorized_renderer.addCategory(cat1)
categorized_renderer.addCategory(cat2)
categorized_renderer.setClassAttribute('landuse')

layer.setRenderer(categorized_renderer)
layer.triggerRepaint()

# Iterate categories
for cat in categorized_renderer.categories():
    print(f"{cat.value()}: {cat.label()} :: {cat.symbol()}")
```

### Graduated Renderer

Assigns symbols based on numeric ranges:

```python
from qgis.core import QgsGraduatedSymbolRenderer, QgsRendererRange, QgsClassificationRange

graduated_renderer = QgsGraduatedSymbolRenderer()

graduated_renderer.addClassRange(
    QgsRendererRange(
        QgsClassificationRange('Low', 0, 100),
        QgsMarkerSymbol.createSimple({'color': 'green'})
    )
)
graduated_renderer.addClassRange(
    QgsRendererRange(
        QgsClassificationRange('High', 100, 1000),
        QgsMarkerSymbol.createSimple({'color': 'red'})
    )
)
graduated_renderer.setClassAttribute('population')

layer.setRenderer(graduated_renderer)
layer.triggerRepaint()

# Iterate ranges
for ran in graduated_renderer.ranges():
    print(f"{ran.lowerValue()} - {ran.upperValue()}: {ran.label()}")
```

### Rule-Based Renderer

The most flexible renderer — applies symbols based on expression rules:

```python
from qgis.core import QgsRuleBasedRenderer, QgsMarkerSymbol

# Start from default symbol
root_rule = QgsRuleBasedRenderer.Rule(QgsMarkerSymbol())

# Add child rules with filter expressions
rule1 = QgsRuleBasedRenderer.Rule(
    QgsMarkerSymbol.createSimple({'color': 'red'}),
    filterExp='"population" > 100000',
    label='Large cities'
)
rule2 = QgsRuleBasedRenderer.Rule(
    QgsMarkerSymbol.createSimple({'color': 'blue'}),
    filterExp='"population" <= 100000',
    label='Small cities'
)

root_rule.appendChild(rule1)
root_rule.appendChild(rule2)

renderer = QgsRuleBasedRenderer(root_rule)
layer.setRenderer(renderer)
layer.triggerRepaint()
```

### Available Renderers

```python
# List all registered renderer types
print(QgsApplication.rendererRegistry().renderersList())
```

### QgsColorRamp Types

Color ramps define color gradients for graduated symbology:

- `QgsGradientColorRamp` — linear gradient between two colors
- `QgsPresetSchemeColorRamp` — predefined color set
- `QgsRandomColorRamp` — random colors
- `QgsCptCityColorRamp` — cpt-city color ramp collection

### Labeling Configuration

```python
from qgis.core import QgsPalLayerSettings, QgsVectorLayerSimpleLabeling, QgsTextFormat

# Configure label settings
settings = QgsPalLayerSettings()
settings.fieldName = 'name'        # Field or expression to label with
settings.isExpression = False       # Set True if fieldName is an expression
settings.enabled = True
settings.drawLabels = True

# Placement options (Qgis.LabelPlacement enum)
settings.placement = Qgis.LabelPlacement.AroundPoint  # For points
# Other options: Line, Curved, Horizontal, OverPoint, etc.

# Text format
text_format = QgsTextFormat()
text_format.setSize(10)
text_format.setColor(QColor('black'))

# Text buffer (halo)
buffer_settings = text_format.buffer()
buffer_settings.setEnabled(True)
buffer_settings.setSize(1.0)
buffer_settings.setColor(QColor('white'))

settings.setFormat(text_format)

# Apply to layer
labeling = QgsVectorLayerSimpleLabeling(settings)
layer.setLabeling(labeling)
layer.setLabelsEnabled(True)
layer.triggerRepaint()
```

### Rule-Based Labeling

```python
from qgis.core import QgsRuleBasedLabeling

# Create root rule
root = QgsRuleBasedLabeling.Rule(QgsPalLayerSettings())

# Child rule for large cities
settings_large = QgsPalLayerSettings()
settings_large.fieldName = 'name'
settings_large.enabled = True
text_format_large = QgsTextFormat()
text_format_large.setSize(14)
settings_large.setFormat(text_format_large)

rule_large = QgsRuleBasedLabeling.Rule(settings_large)
rule_large.setFilterExpression('"population" > 100000')
root.appendChild(rule_large)

labeling = QgsRuleBasedLabeling(root)
layer.setLabeling(labeling)
layer.setLabelsEnabled(True)
layer.triggerRepaint()
```

---

## 11. Anti-patterns and Critical Warnings

### Feature Editing

- **NEVER** modify features outside edit sessions. Changes will be silently lost or corrupt the data source.
- **ALWAYS** close edit sessions with either `commitChanges()` or `rollBack()`. Use the `with edit(layer):` context manager to guarantee cleanup.
- **ALWAYS** call `layer.updateFields()` after adding or removing attributes via the data provider.

### Threading

- **NEVER** access `iface`, `QgsProject.instance()`, or any GUI object from a background thread (`run()` method of `QgsTask`). This causes crashes.
- **NEVER** raise exceptions in `QgsTask.run()`. Return `False` to indicate failure.
- **ALWAYS** copy data before passing to a background task. Do not pass live layer references.
- **ALWAYS** keep `QgsProcessingContext` and `QgsProcessingFeedback` alive for the duration of `QgsProcessingAlgRunnerTask`.

### Feature Iteration

- **NEVER** use `layer.featureCount()` to check if a layer has features — some providers return -1 or inaccurate counts. Use `getFeatures()` with `setLimit(1)` instead.
- **ALWAYS** use `QgsFeatureRequest` to limit the features and attributes loaded. Loading all features with all attributes is the most common performance mistake.

### Geometry

- **ALWAYS** check for NULL geometries before performing geometry operations: `if not geom.isNull():`
- **ALWAYS** use `QgsDistanceArea` for measurements on geographic (lat/lon) CRS data. The simple `area()` and `length()` methods do not account for CRS.
- **ALWAYS** validate geometries before spatial operations if data quality is uncertain: `geom.isGeosValid()`.

### Expressions

- **ALWAYS** check `exp.hasParserError()` after creating a `QgsExpression`.
- **ALWAYS** check `exp.hasEvalError()` after calling `exp.evaluate()`.
- **ALWAYS** set up a proper `QgsExpressionContext` with appropriate scopes when evaluating expressions that reference fields, project variables, or layer properties.

### Performance

- **NEVER** use `print()` in multithreaded code (expression functions, renderers, processing algorithms). Use `QgsMessageLog` instead.
- **ALWAYS** use `QgsSpatialIndex` for repeated spatial queries against the same dataset.
- **ALWAYS** use `QgsFeatureRequest.NoGeometry` flag when only attributes are needed.
- **ALWAYS** use `setSubsetOfAttributes()` when only specific fields are needed.

### Plugins

- **ALWAYS** implement `unload()` to remove ALL GUI elements, menu entries, and toolbar icons added in `initGui()`. Failure to do so causes ghost UI elements.
- **NEVER** leave compiled files (ui_*.py, resources_rc.py) in the repository. Generate them during build.
- **ALWAYS** use `self.iface.mainWindow()` as parent for QActions and dialogs to ensure proper window management.

---

## Sources

- QGIS PyQGIS Developer Cookbook — Vector Layers: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/vector.html
- QGIS PyQGIS Developer Cookbook — Geometry Handling: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/geometry.html
- QGIS PyQGIS Developer Cookbook — Expressions: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/expressions.html
- QGIS PyQGIS Developer Cookbook — Tasks: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/tasks.html
- QGIS PyQGIS Developer Cookbook — Plugin Development: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/plugins/index.html
- QGIS PyQGIS Developer Cookbook — Plugin Structure: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/plugins/plugins.html
- QGIS PyQGIS Developer Cookbook — Plugin Release: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/plugins/releasing.html
- QGIS PyQGIS Developer Cookbook — Communicating: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/communicating.html
- PyQGIS API Reference — QgsPalLayerSettings: https://qgis.org/pyqgis/master/core/QgsPalLayerSettings.html
