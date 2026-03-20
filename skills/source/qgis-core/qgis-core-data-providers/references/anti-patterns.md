# qgis-core-data-providers — Anti-Patterns

## AP-001: Skipping Layer Validity Check

**WRONG:**
```python
layer = QgsVectorLayer("data/roads.gpkg|layername=roads", "Roads", "ogr")
QgsProject.instance().addMapLayer(layer)  # May add invalid layer silently
for feat in layer.getFeatures():          # Crashes or undefined behavior on invalid layer
    print(feat)
```

**CORRECT:**
```python
layer = QgsVectorLayer("data/roads.gpkg|layername=roads", "Roads", "ogr")
if not layer.isValid():
    raise RuntimeError(f"Failed to load layer: {layer.error().summary()}")
QgsProject.instance().addMapLayer(layer)
```

**Why**: `QgsVectorLayer()` and `QgsRasterLayer()` NEVER raise exceptions on failure. They ALWAYS return a layer object. The ONLY way to detect failure is `isValid()`. Accessing features on an invalid layer causes crashes.

---

## AP-002: Using Backslashes in URIs on Windows

**WRONG:**
```python
layer = QgsVectorLayer("C:\\Users\\data\\file.gpkg|layername=roads", "Roads", "ogr")
```

**CORRECT:**
```python
layer = QgsVectorLayer("C:/Users/data/file.gpkg|layername=roads", "Roads", "ogr")
# OR use os.path and replace:
import os
path = os.path.join("C:\\Users", "data", "file.gpkg").replace("\\", "/")
layer = QgsVectorLayer(f"{path}|layername=roads", "Roads", "ogr")
```

**Why**: QGIS/Qt normalizes paths to forward slashes internally. Backslashes in URI strings cause provider parsing failures that result in silent load failures (layer is invalid with no clear error).

---

## AP-003: Assuming GeoPackage Has Only One Layer

**WRONG:**
```python
layer = QgsVectorLayer("data/project.gpkg", "Data", "ogr")
# Loads the first layer by default — but WHICH first layer is undefined
```

**CORRECT:**
```python
layer = QgsVectorLayer("data/project.gpkg|layername=buildings", "Buildings", "ogr")
```

**Why**: A GeoPackage can contain multiple vector layers, raster layers, and attribute tables. Without `|layername=`, the provider picks the first layer it encounters, which may not be the intended one. The ordering is not guaranteed across QGIS versions.

---

## AP-004: Exposing Credentials via uri.uri(True)

**WRONG:**
```python
uri = QgsDataSourceUri()
uri.setConnection("db.example.com", "5432", "gisdb", "admin", "s3cr3t_passw0rd")
uri.setDataSource("public", "roads", "geom")
print(f"Loading from: {uri.uri(True)}")  # Prints credentials in plain text
logger.info(f"Connection URI: {uri.uri(True)}")  # Credentials in log files
```

**CORRECT:**
```python
print(f"Loading from: {uri.uri(False)}")  # Credentials are hidden
logger.info(f"Connection URI: {uri.uri(False)}")
```

**Why**: `uri.uri(True)` expands authentication configurations and includes the raw username/password in the returned string. This leaks credentials to log files, console output, and error messages. ALWAYS use `uri.uri(False)` for any display or logging purpose.

---

## AP-005: Forgetting updateExtents() on Memory Layers

**WRONG:**
```python
layer = QgsVectorLayer("Point?crs=EPSG:4326", "Points", "memory")
pr = layer.dataProvider()
feat = QgsFeature()
feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(5.0, 52.0)))
pr.addFeatures([feat])
QgsProject.instance().addMapLayer(layer)
# Zoom to layer shows wrong extent (empty or 0,0)
```

**CORRECT:**
```python
layer = QgsVectorLayer("Point?crs=EPSG:4326", "Points", "memory")
pr = layer.dataProvider()
feat = QgsFeature()
feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(5.0, 52.0)))
pr.addFeatures([feat])
layer.updateExtents()  # REQUIRED after adding features
QgsProject.instance().addMapLayer(layer)
```

**Why**: Memory layers do not automatically recalculate their spatial extent when features are added via the data provider. Without `updateExtents()`, the layer reports a zero or stale extent, causing "Zoom to Layer" to fail.

---

## AP-006: Forgetting updateFields() After Adding Attributes

**WRONG:**
```python
layer = QgsVectorLayer("Point?crs=EPSG:4326", "Points", "memory")
pr = layer.dataProvider()
pr.addAttributes([QgsField("name", QVariant.String)])
# layer.fields() still returns the old field list
feat = QgsFeature(layer.fields())  # Feature has wrong field count
```

**CORRECT:**
```python
layer = QgsVectorLayer("Point?crs=EPSG:4326", "Points", "memory")
pr = layer.dataProvider()
pr.addAttributes([QgsField("name", QVariant.String)])
layer.updateFields()  # REQUIRED to sync layer schema
feat = QgsFeature(layer.fields())
```

**Why**: `addAttributes()` modifies the data provider's schema but does NOT automatically update the `QgsVectorLayer`'s cached field list. Without `updateFields()`, `layer.fields()` returns stale data and features created from it have the wrong number of fields.

---

## AP-007: Missing file:// Prefix for Delimited Text

**WRONG:**
```python
uri = "/data/stations.csv?delimiter=,&xField=lon&yField=lat"
vlayer = QgsVectorLayer(uri, "Stations", "delimitedtext")
# Layer is invalid — delimitedtext provider requires file:// prefix
```

**CORRECT:**
```python
uri = "file:///data/stations.csv?delimiter=,&xField=lon&yField=lat&crs=EPSG:4326"
vlayer = QgsVectorLayer(uri, "Stations", "delimitedtext")
```

**Why**: The delimited text provider ALWAYS requires the `file://` URI scheme prefix. For absolute paths, use three slashes (`file:///`). Omitting it causes the provider to fail silently.

---

## AP-008: Using Wrong Case for WFS Provider Key

**WRONG:**
```python
vlayer = QgsVectorLayer(wfs_url, "WFS Layer", "wfs")  # lowercase — WRONG
```

**CORRECT:**
```python
vlayer = QgsVectorLayer(wfs_url, "WFS Layer", "WFS")  # uppercase — CORRECT
```

**Why**: The WFS provider key is `"WFS"` (uppercase), unlike most other provider keys which are lowercase. Using `"wfs"` causes a provider-not-found error. This is a common source of confusion.

---

## AP-009: Not URL-Encoding XYZ Tile Placeholders

**WRONG:**
```python
uri = "type=xyz&url=https://tile.openstreetmap.org/{z}/{x}/{y}.png&zmin=0&zmax=19"
rlayer = QgsRasterLayer(uri, "OSM", "wms")
# Fails because {z}, {x}, {y} are not URL-encoded
```

**CORRECT:**
```python
uri = "type=xyz&url=https://tile.openstreetmap.org/%7Bz%7D/%7Bx%7D/%7By%7D.png&zmin=0&zmax=19&crs=EPSG3857"
rlayer = QgsRasterLayer(uri, "OSM", "wms")
```

**Why**: The `{z}`, `{x}`, `{y}` placeholders in XYZ tile URLs MUST be URL-encoded as `%7Bz%7D`, `%7Bx%7D`, `%7By%7D` when used in the URI string. The curly braces are interpreted as URI delimiters if not encoded, causing the provider to fail.

---

## AP-010: Hardcoding Absolute Paths in Cross-Platform Code

**WRONG:**
```python
layer = QgsVectorLayer("/home/user/data/roads.gpkg|layername=roads", "Roads", "ogr")
```

**CORRECT:**
```python
import os
data_dir = os.path.join(QgsProject.instance().homePath(), "data")
layer_path = os.path.join(data_dir, "roads.gpkg").replace("\\", "/")
layer = QgsVectorLayer(f"{layer_path}|layername=roads", "Roads", "ogr")
```

**Why**: Hardcoded absolute paths break when the project is moved to another machine or operating system. ALWAYS construct paths relative to the project directory using `QgsProject.instance().homePath()` or use `os.path.join()`.

---

## AP-011: Creating Layers on Background Threads

**WRONG:**
```python
import threading

def load_data():
    layer = QgsVectorLayer("data.gpkg|layername=roads", "Roads", "ogr")
    QgsProject.instance().addMapLayer(layer)  # NOT thread-safe

thread = threading.Thread(target=load_data)
thread.start()
```

**CORRECT:**
```python
from qgis.core import QgsTask, QgsApplication

class LoadTask(QgsTask):
    def run(self):
        # Do heavy processing here (reading, transforming)
        self.result_data = process_data()
        return True

    def finished(self, result):
        # This runs on the main thread — safe to add layers
        if result:
            layer = QgsVectorLayer(...)
            QgsProject.instance().addMapLayer(layer)

task = LoadTask("Load Data")
QgsApplication.taskManager().addTask(task)
```

**Why**: `QgsProject.instance()` is NOT thread-safe for writes. Creating layers and adding them to the project from background threads causes race conditions, crashes, and data corruption. ALWAYS use `QgsTask` and add layers in the `finished()` callback, which runs on the main thread.

---

## AP-012: Omitting Provider Key for Non-Default Providers

**WRONG:**
```python
# Raster — provider defaults to "gdal", but WMS needs explicit provider
rlayer = QgsRasterLayer(wms_uri, "WMS Layer")  # Uses "gdal" provider — fails
```

**CORRECT:**
```python
rlayer = QgsRasterLayer(wms_uri, "WMS Layer", "wms")  # Explicit provider
```

**Why**: `QgsRasterLayer` defaults to the `"gdal"` provider when no provider key is given. For WMS, WMTS, XYZ, WCS, or PostGIS raster sources, you MUST specify the provider key explicitly. The GDAL provider cannot parse WMS URIs and the layer silently fails.
