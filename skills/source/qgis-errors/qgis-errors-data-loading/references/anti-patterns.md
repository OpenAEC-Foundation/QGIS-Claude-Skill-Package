# qgis-errors-data-loading — Anti-Patterns

## AP-001: Not Checking Layer Validity

**NEVER** skip the `isValid()` check after layer creation.

```python
# WRONG — silent failure, layer may be invalid
layer = QgsVectorLayer("/data/roads.shp", "Roads", "ogr")
QgsProject.instance().addMapLayer(layer)  # Adds a broken layer
for feat in layer.getFeatures():  # Crashes or returns nothing
    process(feat)

# CORRECT — ALWAYS check validity before use
layer = QgsVectorLayer("/data/roads.shp", "Roads", "ogr")
if not layer.isValid():
    raise RuntimeError(f"Layer failed to load: {layer.dataProvider().error().message()}")
QgsProject.instance().addMapLayer(layer)
```

**Why**: QGIS does NOT raise exceptions for invalid layers. The constructor ALWAYS returns an object, even when loading fails completely. Accessing features or providers on an invalid layer causes crashes or undefined behavior.

---

## AP-002: Using Backslashes in File Paths

**NEVER** use backslashes in paths passed to QGIS layer constructors.

```python
# WRONG — backslashes cause provider failures
layer = QgsVectorLayer("C:\\data\\roads.shp", "Roads", "ogr")

# CORRECT — forward slashes work on all platforms
layer = QgsVectorLayer("C:/data/roads.shp", "Roads", "ogr")

# CORRECT — use os.path for cross-platform safety
import os
path = os.path.join("C:/data", "roads.shp")
layer = QgsVectorLayer(path, "Roads", "ogr")
```

**Why**: QGIS/Qt normalizes paths to forward slashes internally. Backslashes in URI strings break provider parsing, especially for GeoPackage (`|layername=`) and delimited text (`?delimiter=`) URIs.

---

## AP-003: Hardcoding Absolute Paths

**NEVER** hardcode absolute paths that are specific to one machine.

```python
# WRONG — breaks on any other machine
layer = QgsVectorLayer("/home/john/projects/gis/data/roads.shp", "Roads", "ogr")

# CORRECT — use project-relative paths
import os
project_dir = QgsProject.instance().homePath()
path = os.path.join(project_dir, "data", "roads.shp")
layer = QgsVectorLayer(path, "Roads", "ogr")
```

**Why**: Hardcoded paths make scripts non-portable. They fail when the project is moved, shared, or deployed to a different environment.

---

## AP-004: Manual String Concatenation for Database URIs

**NEVER** build database connection strings by concatenating strings.

```python
# WRONG — error-prone, credential exposure risk
uri = "dbname='gisdb' host=localhost port=5432 user='admin' password='secret' table=\"public\".\"roads\" (geom)"
layer = QgsVectorLayer(uri, "Roads", "postgres")

# CORRECT — use QgsDataSourceUri
from qgis.core import QgsDataSourceUri

uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "admin", "secret")
uri.setDataSource("public", "roads", "geom", "", "gid")
layer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
```

**Why**: Manual concatenation is error-prone with quoting and escaping. `QgsDataSourceUri` handles all escaping correctly and provides `uri(False)` to prevent credential leaking.

---

## AP-005: Exposing Credentials in URIs

**NEVER** pass `True` to `uri.uri()` when logging or displaying connection strings.

```python
# WRONG — credentials visible in logs
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "admin", "s3cret!")
uri.setDataSource("public", "roads", "geom")
print(f"Connecting to: {uri.uri(True)}")  # Prints password in plain text

# CORRECT — suppress credential expansion
print(f"Connecting to: {uri.uri(False)}")

# BEST — use QgsAuthManager for credentials
uri.setConnection("localhost", "5432", "gisdb", "", "")
uri.setAuthConfigId("my_auth_config")
```

**Why**: `uri(True)` expands stored authentication configurations, embedding passwords in the output string. This exposes credentials in log files, error messages, and debug output.

---

## AP-006: Assuming GeoPackage Has One Layer

**NEVER** assume a GeoPackage file contains only one layer.

```python
# WRONG — loads the first layer, which may not be what you want
layer = QgsVectorLayer("/data/project.gpkg", "Data", "ogr")

# CORRECT — ALWAYS specify the layer name explicitly
layer = QgsVectorLayer("/data/project.gpkg|layername=buildings", "Buildings", "ogr")
```

**Why**: GeoPackage files commonly contain multiple layers. Without `|layername=`, the OGR provider loads the first layer by index, which may not be the intended layer. This causes silent logic errors.

---

## AP-007: Loading All Features from Large Datasets

**NEVER** load all features into a list for large datasets.

```python
# WRONG — loads millions of features into memory
all_features = list(layer.getFeatures())
for feat in all_features:
    process(feat)

# CORRECT — iterate directly (streaming, low memory)
for feat in layer.getFeatures():
    process(feat)

# CORRECT — use filters to reduce the working set
request = QgsFeatureRequest().setFilterRect(area_of_interest)
for feat in layer.getFeatures(request):
    process(feat)
```

**Why**: `list(layer.getFeatures())` loads every feature object into memory simultaneously. For datasets with millions of features, this causes memory exhaustion. The iterator pattern processes one feature at a time.

---

## AP-008: Confusing setCrs() with Reprojection

**NEVER** use `setCrs()` to reproject data — it only changes the CRS label.

```python
from qgis.core import QgsCoordinateReferenceSystem

# WRONG — this does NOT reproject coordinates
layer.setCrs(QgsCoordinateReferenceSystem("EPSG:4326"))
# Coordinates are still in the original CRS, but QGIS thinks they are WGS84

# CORRECT — use Processing to reproject
import processing
result = processing.run("native:reprojectlayer", {
    'INPUT': layer,
    'TARGET_CRS': QgsCoordinateReferenceSystem("EPSG:4326"),
    'OUTPUT': 'memory:'
})
reprojected = result['OUTPUT']
```

**Why**: `setCrs()` is metadata-only — it tells QGIS what CRS the coordinates are in. It does NOT transform the coordinate values. Using it incorrectly causes features to display at the wrong location.

---

## AP-009: Not Specifying Key Column for PostGIS Views

**NEVER** load a PostGIS view without specifying a unique key column.

```python
# WRONG — no key column specified for a view
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "user", "pass")
uri.setDataSource("public", "my_view", "geom")
layer = QgsVectorLayer(uri.uri(False), "My View", "postgres")

# CORRECT — ALWAYS specify a key column for views
uri.setDataSource("public", "my_view", "geom", "", "unique_id")
layer = QgsVectorLayer(uri.uri(False), "My View", "postgres")
```

**Why**: PostGIS views do not have primary keys. Without a specified key column, QGIS cannot reliably identify features, causing undefined behavior during selection, editing, and feature iteration.

---

## AP-010: Ignoring XYZ Tile URL Encoding

**NEVER** use raw `{z}`, `{x}`, `{y}` placeholders in XYZ tile URIs.

```python
# WRONG — raw curly braces break URI parsing
url = "type=xyz&url=https://tiles.example.com/{z}/{x}/{y}.png"
layer = QgsRasterLayer(url, "Tiles", "wms")

# CORRECT — URL-encode the placeholders
url = "type=xyz&url=https://tiles.example.com/%7Bz%7D/%7Bx%7D/%7By%7D.png&zmax=19&zmin=0"
layer = QgsRasterLayer(url, "Tiles", "wms")
```

**Why**: The curly braces are interpreted as URI template syntax by the parser rather than being passed through as literal characters. URL-encoding (`%7B` / `%7D`) ensures they are treated as literal text.

---

## AP-011: Not Calling updateFields() After Adding Attributes

**NEVER** skip `updateFields()` after adding attributes to a memory layer.

```python
from qgis.core import QgsVectorLayer, QgsField
from qgis.PyQt.QtCore import QVariant

layer = QgsVectorLayer("Point?crs=EPSG:4326", "Points", "memory")
pr = layer.dataProvider()
pr.addAttributes([QgsField("name", QVariant.String)])

# WRONG — fields not synced, feature creation fails silently
feat = QgsFeature(layer.fields())  # layer.fields() is empty!

# CORRECT — sync the layer's field cache
layer.updateFields()
feat = QgsFeature(layer.fields())  # Now has "name" field
```

**Why**: `dataProvider().addAttributes()` modifies the provider's schema, but the layer object caches its own copy of fields. Without `updateFields()`, the layer's field list is stale.

---

## AP-012: Not Calling updateExtents() After Adding Features

**NEVER** skip `updateExtents()` after adding features to a memory layer.

```python
# WRONG — extent is (0,0,0,0), zoom-to-layer fails
pr.addFeatures([feat1, feat2, feat3])
QgsProject.instance().addMapLayer(layer)

# CORRECT — update extent after features are added
pr.addFeatures([feat1, feat2, feat3])
layer.updateExtents()
QgsProject.instance().addMapLayer(layer)
```

**Why**: Memory layers do not automatically recalculate their spatial extent when features are added. Without `updateExtents()`, the layer reports an empty extent, causing "zoom to layer" and spatial index operations to fail.
