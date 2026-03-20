# Anti-Patterns (QGIS Architecture)

## Initialization Anti-Patterns

### 1. Missing setPrefixPath Before initQgis

```python
# WRONG: initQgis() without setPrefixPath -- providers fail to load
qgs = QgsApplication([], False)
qgs.initQgis()

# CORRECT: ALWAYS set prefix path first
QgsApplication.setPrefixPath("/usr/share/qgis", True)
qgs = QgsApplication([], False)
qgs.initQgis()
```

**WHY**: Without the prefix path, QGIS cannot locate its provider plugins, CRS database, or authentication database. Layers will fail to load with cryptic "unknown provider" errors.

---

### 2. Initializing QgsApplication Inside a Plugin

```python
# WRONG: Calling initQgis() inside a QGIS plugin or console
def initGui(self):
    qgs = QgsApplication([], False)
    qgs.initQgis()  # Application is ALREADY initialized!

# CORRECT: In plugins, the application is already running
def initGui(self):
    # QgsProject.instance() and iface are ready to use
    project = QgsProject.instance()
```

**WHY**: QGIS is already initialized when your plugin loads. Creating a second QgsApplication instance causes undefined behavior including crashes and corrupted state.

---

### 3. Forgetting exitQgis in Standalone Scripts

```python
# WRONG: No cleanup -- memory leak
QgsApplication.setPrefixPath("/usr/share/qgis", True)
qgs = QgsApplication([], False)
qgs.initQgis()
# ... do work ...
# Script ends without exitQgis()

# CORRECT: ALWAYS clean up
QgsApplication.setPrefixPath("/usr/share/qgis", True)
qgs = QgsApplication([], False)
qgs.initQgis()
try:
    # ... do work ...
    pass
finally:
    qgs.exitQgis()
```

**WHY**: Without `exitQgis()`, provider and layer registries remain in memory. In long-running scripts or repeated executions, this causes memory leaks and potential resource exhaustion.

---

## Threading Anti-Patterns

### 4. Modifying QgsProject from a Background Thread

```python
# WRONG: Adding a layer from a background thread
class BadTask(QgsTask):
    def run(self):
        layer = QgsVectorLayer("data/points.shp", "Points", "ogr")
        QgsProject.instance().addMapLayer(layer)  # CRASH or corruption
        return True

# CORRECT: Create data on background thread, add layer on main thread
class GoodTask(QgsTask):
    def __init__(self):
        super().__init__("Safe Task", QgsTask.CanCancel)
        self.result_data = None

    def run(self):
        # Process data here -- NO project writes
        self.result_data = self._compute_results()
        return True

    def finished(self, result):
        # Runs on main thread -- safe to modify project
        if result:
            layer = QgsVectorLayer("Point?crs=EPSG:4326", "Results", "memory")
            QgsProject.instance().addMapLayer(layer)
```

**WHY**: `QgsProject.instance()` is NOT thread-safe for write operations. Writing from a background thread causes data corruption, crashes, or silently broken state.

---

### 5. Updating GUI from a Background Thread

```python
# WRONG: Updating map canvas from a background thread
class BadTask(QgsTask):
    def run(self):
        iface.mapCanvas().refresh()  # GUI access from background thread!
        return True

# CORRECT: Use finished() or signals for GUI updates
class GoodTask(QgsTask):
    def run(self):
        # Data processing only
        return True

    def finished(self, result):
        # Main thread -- safe to update GUI
        iface.mapCanvas().refresh()
```

**WHY**: Qt requires ALL GUI operations to execute on the main thread. Accessing GUI objects from background threads causes crashes, rendering artifacts, or deadlocks.

---

## Layer Anti-Patterns

### 6. Skipping Layer Validity Check

```python
# WRONG: Using a layer without validity check
layer = QgsVectorLayer("data/missing_file.shp", "Layer", "ogr")
for feature in layer.getFeatures():  # Crash or undefined behavior
    print(feature)

# CORRECT: ALWAYS check isValid() after creation
layer = QgsVectorLayer("data/missing_file.shp", "Layer", "ogr")
if not layer.isValid():
    raise RuntimeError(f"Failed to load layer from: {layer.source()}")
for feature in layer.getFeatures():
    print(feature)
```

**WHY**: A layer constructor NEVER raises an exception even if the data source is invalid. The layer object is created but in an invalid state. Accessing features or provider methods on an invalid layer causes crashes or returns garbage data.

---

### 7. Assuming mapLayersByName Returns One Result

```python
# WRONG: Assuming a single result
layer = QgsProject.instance().mapLayersByName("Roads")  # Returns a LIST
layer.getFeatures()  # TypeError: list has no attribute getFeatures

# ALSO WRONG: Indexing without empty check
layer = QgsProject.instance().mapLayersByName("Roads")[0]  # IndexError if empty

# CORRECT: Handle list result and empty case
layers = QgsProject.instance().mapLayersByName("Roads")
if not layers:
    raise RuntimeError("Layer 'Roads' not found in project")
layer = layers[0]
```

**WHY**: Layer names are NOT unique in QGIS. Multiple layers can share the same name. `mapLayersByName()` ALWAYS returns a list, even for zero or one matches.

---

### 8. Forgetting updateExtents After Adding Features

```python
# WRONG: Extent is stale after adding features
mem_layer = QgsVectorLayer("Point?crs=EPSG:4326", "Points", "memory")
provider = mem_layer.dataProvider()
provider.addFeatures(features)
# layer.extent() returns empty/wrong extent
# "Zoom to Layer" does nothing useful

# CORRECT: ALWAYS update extents after modifying features
mem_layer = QgsVectorLayer("Point?crs=EPSG:4326", "Points", "memory")
provider = mem_layer.dataProvider()
provider.addFeatures(features)
mem_layer.updateExtents()  # Recalculates bounding box
```

**WHY**: Memory layers do not automatically recalculate their extent when features are added via the data provider. Without `updateExtents()`, the layer reports an incorrect bounding box, causing "Zoom to Layer" to fail and spatial queries to miss features.

---

### 9. Forgetting updateFields After Adding Attributes

```python
# WRONG: Field list is stale after adding attributes
provider = layer.dataProvider()
provider.addAttributes([QgsField("new_col", QVariant.String)])
print(layer.fields().names())  # "new_col" is MISSING from this list

# CORRECT: ALWAYS call updateFields after modifying field structure
provider = layer.dataProvider()
provider.addAttributes([QgsField("new_col", QVariant.String)])
layer.updateFields()
print(layer.fields().names())  # "new_col" is now present
```

**WHY**: The layer caches its field list for performance. When you modify fields through the data provider directly, the layer's cached field list becomes stale. `updateFields()` forces a refresh from the provider.

---

## Path Anti-Patterns

### 10. Using Backslashes in Paths

```python
# WRONG: Backslashes cause provider failures on all platforms
layer = QgsVectorLayer("C:\\data\\points.shp", "Points", "ogr")

# CORRECT: ALWAYS use forward slashes
layer = QgsVectorLayer("C:/data/points.shp", "Points", "ogr")

# ALSO CORRECT: Use os.path or pathlib for construction
import os
path = os.path.join("C:/data", "points.shp")
layer = QgsVectorLayer(path, "Points", "ogr")
```

**WHY**: QGIS and Qt normalize all paths to forward slashes internally. Backslashes in URIs are not consistently handled by all data providers and cause failures, especially with GeoPackage sublayer URIs that use the `|` separator.

---

### 11. Hardcoding Absolute Paths

```python
# WRONG: Hardcoded paths break on other machines
layer = QgsVectorLayer("/home/john/projects/data/roads.shp", "Roads", "ogr")

# CORRECT: Use project-relative paths
import os
data_dir = os.path.join(QgsProject.instance().homePath(), "data")
layer_path = os.path.join(data_dir, "roads.shp")
layer = QgsVectorLayer(layer_path, "Roads", "ogr")
```

**WHY**: Hardcoded absolute paths make projects non-portable. When the project moves to another machine or user, all layer paths break. Using `QgsProject.instance().homePath()` as a base creates portable, relative paths.

---

## Project Anti-Patterns

### 12. Calling clear() Without Safeguards

```python
# WRONG: Destroys everything without warning
QgsProject.instance().clear()

# CORRECT: Check for unsaved changes first (in GUI context)
project = QgsProject.instance()
if project.isDirty():
    # Prompt user or save first
    project.write()
project.clear()
```

**WHY**: `clear()` immediately removes ALL layers, settings, and project state with no confirmation and no undo. In a plugin context, this destroys the user's work without warning.

---

## CRS Anti-Patterns

### 13. Creating QgsCoordinateTransform Without Context

```python
# WRONG: Missing transform context -- ignores datum transformation preferences
xform = QgsCoordinateTransform(source_crs, dest_crs)

# CORRECT: ALWAYS provide the project's transform context
xform = QgsCoordinateTransform(
    source_crs,
    dest_crs,
    QgsProject.instance().transformContext()
)
```

**WHY**: Without a `QgsCoordinateTransformContext`, the transform engine cannot apply datum shift grids or user-configured transformation preferences. This causes silent precision loss of up to several meters depending on the datums involved.

---

## Provider Anti-Patterns

### 14. Exposing Auth Credentials in Logs

```python
# WRONG: expandAuthConfig=True exposes passwords in plain text
uri = layer.dataProvider().uri()
print(f"Connection: {uri.uri(True)}")  # Prints password!

# CORRECT: ALWAYS use False when displaying or logging URIs
uri = layer.dataProvider().uri()
print(f"Connection: {uri.uri(False)}")  # Password masked
```

**WHY**: `uri.uri(True)` expands the authentication configuration and includes the raw username/password in the returned string. This leaks credentials to logs, console output, and error messages.

---

### 15. Assuming GeoPackage Layer Name

```python
# WRONG: Assuming the first layer without checking
layer = QgsVectorLayer("data/multi.gpkg", "Data", "ogr")

# CORRECT: Specify the layer name explicitly
layer = QgsVectorLayer("data/multi.gpkg|layername=buildings", "Buildings", "ogr")

# OR: Enumerate sublayers first
temp = QgsVectorLayer("data/multi.gpkg", "temp", "ogr")
for sub in temp.dataProvider().subLayers():
    name = sub.split(QgsDataProvider.SUBLAYER_SEPARATOR)[1]
    print(f"Available layer: {name}")
```

**WHY**: A GeoPackage file can contain multiple layers. Without `|layername=`, QGIS loads the first layer by internal order, which may not be the intended one. This causes subtle data errors when the wrong layer is loaded silently.

---

## QGIS 4.0 Anti-Patterns

### 16. Using QVariant.Type for Field Definitions

```python
# WRONG for QGIS 4.x: QVariant.Type is removed in Qt6
from qgis.PyQt.QtCore import QVariant
field = QgsField("name", QVariant.String)

# CORRECT for QGIS 4.x: Use QMetaType.Type
from qgis.PyQt.QtCore import QMetaType
field = QgsField("name", QMetaType.Type.QString)

# FORWARD-COMPATIBLE: Try QMetaType first, fall back to QVariant
try:
    from qgis.PyQt.QtCore import QMetaType
    field = QgsField("name", QMetaType.Type.QString)
except ImportError:
    from qgis.PyQt.QtCore import QVariant
    field = QgsField("name", QVariant.String)
```

**WHY**: Qt6 removes `QVariant.Type` entirely. Code using `QVariant.String`, `QVariant.Int`, etc. will raise `AttributeError` in QGIS 4.x. The `QMetaType.Type` enum is the Qt6 replacement.
