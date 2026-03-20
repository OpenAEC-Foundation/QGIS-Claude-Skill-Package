# Vooronderzoek: QGIS Architecture and Core Concepts

> Research document for the QGIS Claude Skill Package.
> Sources: PyQGIS Developer Cookbook (QGIS 3.44), fetched 2026-03-20.
> This document serves as the knowledge base for creating core skills: architecture, data-providers, and coordinate-systems.

---

## 1. QGIS Architecture Overview

### Qt-Based Application Architecture

QGIS is built on the Qt framework. PyQGIS bindings in QGIS 3 depend on **SIP** and **PyQt5** (migrating to PyQt6 in QGIS 4.x). SIP is used instead of SWIG because the QGIS codebase depends on Qt libraries, and SIP provides tighter Qt integration.

The application exposes its functionality through five primary Python modules:

| Module | Purpose |
|--------|---------|
| `qgis.core` | All non-GUI functionality: layers, features, geometry, CRS, project, processing, data providers |
| `qgis.gui` | GUI components including the map canvas widget (`QgsMapCanvas`), layer tree view, attribute forms |
| `qgis.analysis` | Spatial analysis tools: network analysis, raster calculator, interpolation |
| `qgis.server` | QGIS Server components for OGC web services (WMS, WFS, WCS) |
| `qgis._3d` | 3D map visualization components [VERIFY: limited public API documentation] |

Additional utility modules:
- `qgis.utils` — Automatically imported in the Python console; provides helper functions
- `qgis.PyQt` — Re-exports PyQt5/PyQt6 classes for version-independent imports

### Plugin Architecture

QGIS supports two types of plugins:

1. **C++ plugins** — Compiled against the QGIS SDK, loaded as shared libraries
2. **Python plugins** — Loaded from `~/.local/share/QGIS/QGIS3/profiles/default/python/plugins/` (Linux) or equivalent paths on Windows/macOS

Python plugins implement a standard interface:
- `classFactory(iface)` — Entry point, receives `QgisInterface` instance
- `initGui()` — Called when the plugin is loaded; register actions, menus, toolbars
- `unload()` — Called when the plugin is unloaded; clean up all registered elements

The `iface` variable is an instance of `QgisInterface` (`qgis.gui.QgisInterface`) that provides access to the map canvas, menus, toolbars, and other parts of the QGIS application.

### Main Thread Constraints and Threading Model

QGIS follows the Qt threading model:

- **ALL GUI operations MUST execute on the main thread.** This includes modifying map canvas, updating layer tree, and showing dialogs.
- `QgsProject.instance()` is a singleton that is NOT thread-safe for write operations.
- Background processing MUST use `QgsTask` (subclass of `QRunnable`) managed by `QgsApplication.taskManager()`.
- Data provider read operations can run on background threads if the provider supports it, but the layer object itself must not be modified from a background thread.

### Signal/Slot Patterns in PyQGIS

QGIS uses Qt's signal/slot mechanism extensively. In PyQGIS, signals are connected using the standard PyQt pattern:

```python
# Connect to map canvas render completion
iface.mapCanvas().renderComplete.connect(my_callback_function)

# Connect to layer tree changes
root = QgsProject.instance().layerTreeRoot()
root.willAddChildren.connect(on_will_add_children)
root.addedChildren.connect(on_added_children)

# Disconnect when done
iface.mapCanvas().renderComplete.disconnect(my_callback_function)
```

Common signals:
- `QgsMapCanvas.renderComplete(QPainter)` — Emitted after the map canvas finishes rendering
- `QgsProject.instance().layersAdded(list)` — Emitted when layers are added to the project
- `QgsProject.instance().layersRemoved(list)` — Emitted when layers are removed
- `QgsLayerTreeNode.willAddChildren(node, indexFrom, indexTo)` — Before children are added to layer tree
- `QgsLayerTreeNode.addedChildren(node, indexFrom, indexTo)` — After children are added to layer tree

---

## 2. QgsProject and Application Lifecycle

### QgsApplication Initialization

There are two distinct contexts for running PyQGIS code:

#### Context 1: Inside QGIS (Plugin or Console)

When running inside the QGIS application (Python console or plugin), the application is already initialized. The console automatically executes:

```python
from qgis.core import *
import qgis.utils
```

The `iface` variable is pre-populated as an instance of `QgisInterface`.

#### Context 2: Standalone Script (No GUI)

For standalone scripts that run outside the QGIS application:

```python
from qgis.core import *

# Supply path to QGIS install location
QgsApplication.setPrefixPath("/path/to/qgis/installation", True)

# Create a reference to the QgsApplication.
# Setting the second argument to False disables the GUI.
qgs = QgsApplication([], False)

# Load providers
qgs.initQgis()

# Write your code here: load layers, process data, etc.

# When done, remove provider and layer registries from memory
qgs.exitQgis()
```

Key methods:
- `QgsApplication.setPrefixPath(path, useDefaultPaths)` — Configures QGIS installation location; MUST be called before `initQgis()`
- `QgsApplication(argv, GUIenabled)` — Constructor; pass `False` for headless/server, `True` for GUI applications
- `qgs.initQgis()` — Loads data providers, initializes the authentication system, sets up the coordinate reference system database
- `qgs.exitQgis()` — Removes the provider and layer registries from memory; ALWAYS call this to prevent memory leaks

#### Context 3: Standalone Script (With GUI)

```python
from qgis.core import *

QgsApplication.setPrefixPath("/path/to/qgis/installation", True)
qgs = QgsApplication([], True)  # True enables the GUI
qgs.initQgis()

# GUI code here...

qgs.exitQgis()
```

### QgsProject Singleton

`QgsProject` operates as a singleton. Access the current project instance via:

```python
project = QgsProject.instance()
```

#### Loading Projects

```python
project = QgsProject.instance()
success = project.read('testdata/01_project.qgs')  # Returns bool
```

#### Saving Projects

```python
project.write()                                    # Save to current location
project.write('testdata/my_new_qgis_project.qgs')  # Save to new path
# Both return bool indicating success/failure
```

#### Getting Project File Info

```python
project.fileName()       # Returns the project file path
project.absoluteFilePath()  # Returns absolute path [VERIFY method name]
```

### Project File Formats

| Format | Extension | Description |
|--------|-----------|-------------|
| QGS | `.qgs` | XML-based project file; human-readable, larger file size |
| QGZ | `.qgz` | Compressed (ZIP) archive containing the .qgs file and auxiliary data; default since QGIS 3.2 |

### Project Read Flags

Optimize project loading by skipping certain resolution steps:

```python
readflags = Qgis.ProjectReadFlags()
readflags |= Qgis.ProjectReadFlag.DontResolveLayers
project.read('path/to/project.qgs', readflags)
```

The `DontResolveLayers` flag bypasses data validation and prevents bad layer dialogs. This is useful when you only need access to layouts, settings, or 3D configuration without requiring actual layer data.

### Path Resolution — QgsPathResolver

QGIS provides path preprocessing hooks for manipulating paths before and during layer resolution:

```python
# Rewrite paths during loading (e.g., migrating from one machine to another)
def my_load_preprocessor(path):
    return path.replace('c:/Users/Old/', 'x:/New/')

preprocessor_id = QgsPathResolver.setPathPreprocessor(my_load_preprocessor)

# Rewrite paths during saving
def my_save_preprocessor(path):
    return path.replace('c:/Users/ClintBarton/Documents/Projects', '$projectdir$')

writer_id = QgsPathResolver.setPathWriter(my_save_preprocessor)

# Remove preprocessors when no longer needed
QgsPathResolver.removePathPreprocessor(preprocessor_id)
QgsPathResolver.removePathWriter(writer_id)
```

### Reading/Writing Project Settings and Metadata

```python
from qgis.core import QgsSettings

# Global QGIS settings
qs = QgsSettings()
for k in sorted(qs.allKeys()):
    print(k)
```

### Canvas–Project Synchronization (Standalone Apps)

For standalone GUI applications, synchronize the layer tree with the map canvas:

```python
from qgis.core import QgsProject, QgsLayerTreeMapCanvasBridge

bridge = QgsLayerTreeMapCanvasBridge(
    QgsProject.instance().layerTreeRoot(),
    canvas
)
project.read('path/to/project.qgs')
```

---

## 3. Layer Architecture

### QgsMapLayer Base Class and Layer Types

All layers inherit from `QgsMapLayer`. The layer type is accessible via `layer.type()`:

```python
for layer in QgsProject.instance().mapLayers().values():
    print(f"{layer.name()} of type {layer.type().name}")
```

Layer type hierarchy:

| Class | Type | Description |
|-------|------|-------------|
| `QgsVectorLayer` | `VectorLayer` | Points, lines, polygons with attributes |
| `QgsRasterLayer` | `RasterLayer` | Grid-based data (GeoTIFF, WMS, XYZ tiles) |
| `QgsMeshLayer` | `MeshLayer` | Unstructured mesh data (e.g., hydrodynamic models) |
| `QgsPointCloudLayer` | `PointCloudLayer` | LiDAR / point cloud data (LAS, LAZ, COPC, EPT) |
| `QgsAnnotationLayer` | `AnnotationLayer` | Freeform annotations on the map |
| `QgsVectorTileLayer` | `VectorTileLayer` | Mapbox Vector Tiles (MVT) |
| `QgsTiledSceneLayer` | `TiledSceneLayer` | 3D tiled scenes (3D Tiles, e.g., Cesium) [VERIFY] |

### QgsVectorLayer Construction

```python
vlayer = QgsVectorLayer(data_source, layer_name, provider_name)
```

- `data_source` — URI string, specific to each data provider
- `layer_name` — Display name in the layer tree
- `provider_name` — Provider identifier string (e.g., `"ogr"`, `"postgres"`, `"memory"`, `"WFS"`, `"delimitedtext"`, `"spatialite"`, `"gpx"`, `"virtual"`)

### QgsRasterLayer Construction

```python
rlayer = QgsRasterLayer(path_or_uri, layer_name, provider_name)
```

- `provider_name` — e.g., `"gdal"`, `"wms"`, `"wcs"`, `"postgresraster"`

### Layer Validity Checking

**ALWAYS check layer validity after creation:**

```python
vlayer = QgsVectorLayer("testdata/airports.shp", "Airports", "ogr")
if not vlayer.isValid():
    print("Layer failed to load!")
```

A layer can be invalid because:
- The file does not exist
- The data source URI is malformed
- The provider cannot parse the data
- Authentication credentials are missing or wrong
- The CRS database is unavailable (standalone scripts without proper `setPrefixPath`)

### Layer Lifecycle: Adding and Removing

```python
# Add layer to project (appears in layer tree)
QgsProject.instance().addMapLayer(vlayer)

# Add layer to project without showing in layer tree
QgsProject.instance().addMapLayer(vlayer, False)

# Add multiple layers at once
QgsProject.instance().addMapLayers([layer1, layer2, layer3])

# Add via iface (convenience, also zooms to layer)
vlayer = iface.addVectorLayer("data.gpkg|layername=airports", "Airports", "ogr")
rlayer = iface.addRasterLayer("srtm.tif", "SRTM")

# Remove layer by ID
QgsProject.instance().removeMapLayer(layer.id())

# Remove all layers
QgsProject.instance().removeAllMapLayers()

# Clear entire project (layers, settings, everything)
QgsProject.instance().clear()
```

### Layer Registry vs. Project Layer Store

In QGIS 3.x, layers are stored in the **project layer store** (`QgsProject.instance()`). The older `QgsMapLayerRegistry` singleton from QGIS 2.x has been removed. All layer management goes through `QgsProject.instance()`:

```python
# Get all loaded layers (returns dict of {id: layer})
all_layers = QgsProject.instance().mapLayers()

# Find layer by name (returns list, as names are not unique)
layers = QgsProject.instance().mapLayersByName("Airports")
if layers:
    layer = layers[0]

# Get active layer (from GUI context)
layer = iface.activeLayer()

# Set active layer
iface.setActiveLayer(layer)
```

### Layer Tree Management

The layer tree (Table of Contents) is managed through `QgsLayerTreeGroup` and `QgsLayerTreeLayer`:

```python
from qgis.core import QgsProject, QgsLayerTreeGroup, QgsLayerTreeLayer, QgsVectorLayer

root = QgsProject.instance().layerTreeRoot()

# Create a group
node_group = root.addGroup("My Group")

# Add layer to a specific group (not root)
layer = QgsVectorLayer("Point?crs=EPSG:4326", "Points", "memory")
QgsProject.instance().addMapLayer(layer, False)  # False = don't add to root
node_group.addLayer(layer)

# Find group by name
my_group = root.findGroup("My Group")

# Find layer node by layer ID
node = root.findLayer(layer.id())

# Recursively traverse the layer tree
def traverse(group):
    for child in group.children():
        if isinstance(child, QgsLayerTreeGroup):
            print(f"Group: {child.name()}")
            traverse(child)
        elif isinstance(child, QgsLayerTreeLayer):
            print(f"Layer: {child.name()}")

traverse(root)
```

---

## 4. Data Provider Architecture

### QgsProviderRegistry

QGIS uses a plugin-based provider architecture. Providers are registered in `QgsProviderRegistry` and loaded during `QgsApplication.initQgis()`. Each provider handles a specific data format or connection type.

### OGR Provider (Vector Files)

Provider name: `"ogr"`

The OGR provider handles all vector file formats supported by GDAL/OGR.

#### URI Formats

| Format | URI Pattern | Example |
|--------|-------------|---------|
| Shapefile | `path/to/file.shp` | `"testdata/airports.shp"` |
| GeoPackage (single layer) | `path/to/file.gpkg\|layername=name` | `"testdata/data.gpkg\|layername=airports"` |
| GeoPackage (by index) | `path/to/file.gpkg\|layerid=0` | `"testdata/data.gpkg\|layerid=0"` |
| GeoJSON | `path/to/file.geojson` | `"testdata/points.geojson"` |
| DXF | `path/to/file.dxf\|layername=entities\|geometrytype=Polygon` | `"testdata/sample.dxf\|layername=entities\|geometrytype=Polygon"` |
| FlatGeoBuf | `path/to/file.fgb` | `"testdata/parcels.fgb"` |
| KML | `path/to/file.kml` | `"testdata/places.kml"` |
| MySQL via OGR | `MySQL:dbname,host=...,port=...,user=...,password=...\|layername=table` | `"MySQL:mydb,host=localhost,port=3306,user=root,password=xxx\|layername=my_table"` |

#### Loading All Sublayers from a Multi-Layer Source

```python
from qgis.core import QgsDataProvider, QgsVectorLayer, QgsProject

fileName = "testdata/sublayers.gpkg"
layer = QgsVectorLayer(fileName, "test", "ogr")
subLayers = layer.dataProvider().subLayers()

for subLayer in subLayers:
    name = subLayer.split(QgsDataProvider.SUBLAYER_SEPARATOR)[1]
    uri = "%s|layername=%s" % (fileName, name)
    sub_vlayer = QgsVectorLayer(uri, name, 'ogr')
    QgsProject.instance().addMapLayer(sub_vlayer)
```

### GDAL Provider (Raster Files)

Provider name: `"gdal"`

| Format | URI Pattern | Example |
|--------|-------------|---------|
| GeoTIFF | `path/to/file.tif` | `"data/srtm.tif"` |
| GeoPackage raster | `GPKG:path/to/file.gpkg:raster_layer_name` | `"GPKG:/data/rasters.gpkg:srtm"` |
| Cloud Optimized GeoTIFF (COG) | `/vsicurl/https://url/to/cog.tif` | `"/vsicurl/https://example.com/data.tif"` |
| JPEG2000 | `path/to/file.jp2` | `"data/ortho.jp2"` |
| Virtual Raster | `path/to/file.vrt` | `"data/mosaic.vrt"` |

```python
from qgis.core import QgsRasterLayer, QgsProject

# Standard raster file
rlayer = QgsRasterLayer("data/srtm.tif", "SRTM layer name")
if not rlayer.isValid():
    print("Layer failed to load!")

# GeoPackage raster
rlayer = QgsRasterLayer("GPKG:/data/rasters.gpkg:srtm", "SRTM", "gdal")

QgsProject.instance().addMapLayer(rlayer)
```

### PostgreSQL/PostGIS Provider

Provider name: `"postgres"`

Uses `QgsDataSourceUri` for constructing connection URIs:

```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer

uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "dbname", "johny", "xxx")
uri.setDataSource("public", "roads", "the_geom", "cityid = 2643", "primary_key_field")

vlayer = QgsVectorLayer(uri.uri(False), "layer name", "postgres")
```

`QgsDataSourceUri` key methods:
- `setConnection(host, port, database, username, password)` — Set connection parameters
- `setDataSource(schema, table, geometryColumn, sql="", keyColumn="")` — Set table and filter
- `uri(expandAuthConfig)` — Generate the URI string; pass `False` to not expand authentication configurations
- `setSsl(mode)` — Set SSL mode [VERIFY exact method name]
- `setAuthConfigId(authcfg)` — Use QGIS authentication manager configuration

#### PostGIS Raster

Provider name: `"postgresraster"`

```python
from qgis.core import QgsDataSourceUri, QgsRasterLayer, QgsProviderRegistry

uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "dbname", "johny", "xxx")
uri.setDataSource("public", "raster_table", "rast")

# Use QgsProviderRegistry to decode the URI
provider_meta = QgsProviderRegistry.instance().providerMetadata('postgresraster')
# [VERIFY: exact API for PostGIS raster layer creation varies by version]

rlayer = QgsRasterLayer(uri.uri(False), "PostGIS Raster", "postgresraster")
```

### WMS / WMTS / XYZ Tiles Provider

Provider name: `"wms"` (used for WMS, WMTS, and XYZ tile sources)

#### WMS

```python
from qgis.core import QgsRasterLayer, QgsProject

urlWithParams = (
    "crs=EPSG:4326"
    "&format=image/png"
    "&layers=continents"
    "&styles"
    "&url=https://demo.mapserver.org/cgi-bin/wms"
)
rlayer = QgsRasterLayer(urlWithParams, 'WMS layer', 'wms')

if not rlayer.isValid():
    print("Layer failed to load!")
QgsProject.instance().addMapLayer(rlayer)
```

URI parameters for WMS:
- `url` — The WMS service URL
- `layers` — Comma-separated layer names
- `styles` — Comma-separated style names (can be empty)
- `crs` — CRS identifier (e.g., `EPSG:4326`)
- `format` — Image format (e.g., `image/png`, `image/jpeg`)
- `username` / `password` — Authentication (optional)

#### XYZ Tiles

```python
from qgis.core import QgsRasterLayer, QgsProject

def loadXYZ(url, name):
    rasterLyr = QgsRasterLayer("type=xyz&url=" + url, name, "wms")
    QgsProject.instance().addMapLayer(rasterLyr)

urlWithParams = 'https://tile.openstreetmap.org/%7Bz%7D/%7Bx%7D/%7By%7D.png&zmax=19&zmin=0&crs=EPSG3857'
loadXYZ(urlWithParams, 'OpenStreetMap')
```

URI format for XYZ: `type=xyz&url={tile_url}&zmin={min_zoom}&zmax={max_zoom}&crs=EPSG3857`

Note: The `{z}`, `{x}`, `{y}` placeholders in the URL must be URL-encoded as `%7Bz%7D`, `%7Bx%7D`, `%7By%7D` when embedded in the URI string.

### WFS Provider

Provider name: `"WFS"`

```python
from qgis.core import QgsVectorLayer

uri = "https://demo.mapserver.org/cgi-bin/wfs?service=WFS&version=2.0.0&request=GetFeature&typename=ms:cities"
vlayer = QgsVectorLayer(uri, "my wfs layer", "WFS")
```

The WFS provider handles the OGC WFS protocol. The URI is the full GetFeature request URL. Additional parameters can be appended for filtering, pagination, and authentication.

### WCS Provider

Provider name: `"wcs"`

```python
from qgis.core import QgsRasterLayer

uri = "https://demo.mapserver.org/cgi-bin/wcs?identifier=modis"
rlayer = QgsRasterLayer(uri, 'my wcs layer', 'wcs')
```

URI parameters for WCS: `url`, `identifier`, `time`, `format`, `crs`, `username`, `password`, `cache`.

### Memory Provider

Provider name: `"memory"`

Creates in-memory layers with no backing data source. Ideal for temporary results, visualizations, and intermediate processing steps.

```python
from qgis.core import QgsVectorLayer, QgsFeature, QgsGeometry, QgsPointXY, QgsProject, QgsField
from qgis.PyQt.QtCore import QVariant

# Create an empty memory layer with CRS
layer = QgsVectorLayer("Point?crs=EPSG:4326", "Temporary Points", "memory")

# Add fields
pr = layer.dataProvider()
pr.addAttributes([
    QgsField("name", QVariant.String),
    QgsField("value", QVariant.Double)
])
layer.updateFields()

# Add features
feat = QgsFeature()
feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(10, 10)))
feat.setAttributes(["Point A", 42.0])
pr.addFeatures([feat])
layer.updateExtents()

QgsProject.instance().addMapLayer(layer)
```

Memory layer URI geometry types: `"Point"`, `"LineString"`, `"Polygon"`, `"MultiPoint"`, `"MultiLineString"`, `"MultiPolygon"`, `"None"` (for attribute-only tables).

Full URI format with fields: `"Point?crs=EPSG:4326&field=name:string(50)&field=value:double"`

#### Materializing a Selection as a Memory Layer

```python
from qgis.core import QgsFeatureRequest, QgsProject

memory_layer = layer.materialize(
    QgsFeatureRequest().setFilterFids(layer.selectedFeatureIds())
)
QgsProject.instance().addMapLayer(memory_layer)
```

### Delimited Text Provider

Provider name: `"delimitedtext"`

```python
import os
from qgis.core import QgsVectorLayer

uri = "file://{}/testdata/delimited_xy.csv?delimiter={}&xField={}&yField={}".format(
    os.getcwd(), ";", "x", "y"
)
vlayer = QgsVectorLayer(uri, "CSV Points", "delimitedtext")
```

URI parameters:
- `file://` — Prefix required, followed by absolute path
- `delimiter` — Field delimiter character (`;`, `,`, `\t`)
- `xField` / `yField` — Column names for X (longitude) and Y (latitude)
- `crs` — CRS identifier (optional, default varies)
- `wktField` — Alternative to xField/yField for WKT geometry column
- `geomType` — Geometry type hint: `point`, `line`, `polygon`, `none`

### SpatiaLite Provider

Provider name: `"spatialite"`

```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer

uri = QgsDataSourceUri()
uri.setDatabase('/home/martin/test-2.3.sqlite')
uri.setDataSource('', 'Towns', 'Geometry')
vlayer = QgsVectorLayer(uri.uri(), 'Towns', 'spatialite')
```

### GPX Provider

Provider name: `"gpx"`

```python
from qgis.core import QgsVectorLayer

uri = "testdata/layers.gpx?type=track"
vlayer = QgsVectorLayer(uri, "GPS Tracks", "gpx")
```

URI parameter `type` accepts: `track`, `route`, `waypoint`.

### Virtual Layer Provider

Provider name: `"virtual"`

Virtual layers allow SQL queries across multiple loaded layers without exporting data. They use SQLite/SpatiaLite SQL syntax.

```python
from qgis.core import QgsVectorLayer

# Query across existing project layers
uri = "?query=SELECT * FROM airports WHERE elevation > 500"
vlayer = QgsVectorLayer(uri, "High Airports", "virtual")
```

[VERIFY: exact URI format for virtual layers referencing specific layers by name or ID]

---

## 5. Coordinate Reference System Handling

### QgsCoordinateReferenceSystem

The `QgsCoordinateReferenceSystem` class represents a spatial reference system. It can be created from multiple identifier formats:

#### Creation from EPSG Code

```python
from qgis.core import QgsCoordinateReferenceSystem

crs = QgsCoordinateReferenceSystem("EPSG:4326")
print(crs.isValid())  # True
```

#### Supported Identifier Prefixes

| Prefix | Method Used | Example |
|--------|-------------|---------|
| `EPSG:<code>` | `createFromOgcWms()` | `"EPSG:4326"` |
| `POSTGIS:<srid>` | `createFromSrid()` | `"POSTGIS:4326"` |
| `INTERNAL:<srsid>` | `createFromSrsId()` | `"INTERNAL:3452"` |
| `PROJ:<proj>` | `createFromProj()` | `"PROJ:+proj=longlat +datum=WGS84"` |
| `WKT:<wkt>` | `createFromWkt()` | `"WKT:GEOGCS[...]"` |

#### Creation from WKT

```python
wkt = 'GEOGCS["WGS84", DATUM["WGS84", SPHEROID["WGS84", 6378137.0, 298.257223563]], PRIMEM["Greenwich", 0], UNIT["degree", 0.0174532925199433]]'
crs = QgsCoordinateReferenceSystem(wkt)
```

#### Creation from Proj String

```python
crs = QgsCoordinateReferenceSystem()
crs.createFromProj("+proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs")
```

#### Accessing CRS Information

```python
crs = QgsCoordinateReferenceSystem("EPSG:4326")

crs.srsid()              # QGIS internal ID
crs.postgisSrid()         # PostGIS SRID value (e.g., 4326)
crs.description()         # Human-readable name (e.g., "WGS 84")
crs.projectionAcronym()   # Projection type (e.g., "longlat")
crs.ellipsoidAcronym()    # Ellipsoid identifier (e.g., "EPSG:7030")
crs.toProj()              # Full Proj string
crs.toWkt()               # Full WKT string
crs.isGeographic()        # True for geographic CRS (degrees), False for projected (meters, etc.)
crs.mapUnits()            # Unit enumeration (Qgis.DistanceUnit)
crs.authid()              # Authority identifier, e.g., "EPSG:4326"
```

#### Setting CRS on Layers

```python
from qgis.core import QgsProject, QgsCoordinateReferenceSystem

for layer in QgsProject.instance().mapLayers().values():
    layer.setCrs(QgsCoordinateReferenceSystem('EPSG:4326'))
```

#### Displaying CRS in Layer Name

```python
for layer in QgsProject.instance().mapLayers().values():
    crs = layer.crs().authid()
    layer.setName('{} ({})'.format(layer.name(), crs))
```

### QgsCoordinateTransform

Transforms coordinates between two CRS. ALWAYS requires a `QgsCoordinateTransformContext`.

```python
from qgis.core import (
    QgsCoordinateReferenceSystem,
    QgsCoordinateTransform,
    QgsProject,
    QgsPointXY
)

crsSrc = QgsCoordinateReferenceSystem("EPSG:4326")    # WGS 84
crsDest = QgsCoordinateReferenceSystem("EPSG:32633")   # UTM zone 33N

transformContext = QgsProject.instance().transformContext()
xform = QgsCoordinateTransform(crsSrc, crsDest, transformContext)

# Forward transform (source → destination)
pt1 = xform.transform(QgsPointXY(18, 5))
print(f"Transformed: {pt1.x()}, {pt1.y()}")

# Reverse transform (destination → source)
pt2 = xform.transform(pt1, QgsCoordinateTransform.ReverseTransform)
print(f"Reversed: {pt2.x()}, {pt2.y()}")
```

The `transform()` method can also transform:
- `QgsRectangle` — Bounding box transformation
- `QgsGeometry` — Full geometry transformation (in-place via `geometry.transform(xform)`)

### QgsCoordinateTransformContext

The transform context holds project-level settings for datum transformations. It determines which datum transformation grid or pipeline to use when transforming between CRS pairs.

```python
context = QgsProject.instance().transformContext()

# The context is automatically populated from project settings
# and the user's datum transformation preferences

# Use in transforms
xform = QgsCoordinateTransform(crsSrc, crsDest, context)
```

ALWAYS use `QgsProject.instance().transformContext()` rather than creating a bare `QgsCoordinateTransformContext()`, because the project context includes user-configured datum transformation preferences.

### QgsDistanceArea

Measures distances and areas respecting CRS and ellipsoid:

```python
from qgis.core import QgsDistanceArea, QgsCoordinateReferenceSystem, QgsPointXY, QgsProject

d = QgsDistanceArea()
d.setSourceCrs(QgsCoordinateReferenceSystem("EPSG:4326"), QgsProject.instance().transformContext())
d.setEllipsoid('WGS84')

# Measure distance between two points (returns meters)
point1 = QgsPointXY(5.0, 52.0)
point2 = QgsPointXY(5.1, 52.1)
distance = d.measureLine(point1, point2)
print(f"Distance: {distance} meters")
```

Key methods:
- `setSourceCrs(crs, context)` — Set the source CRS and transform context
- `setEllipsoid(ellipsoid)` — Set the ellipsoid for calculations (e.g., `'WGS84'`, `'GRS80'`)
- `measureLine(point1, point2)` — Distance between two points in meters
- `measureLine(points_list)` — Total distance along a polyline
- `measureArea(geometry)` — Area of a polygon geometry in square meters
- `convertLengthMeasurement(length, unit)` — Convert length to target unit
- `convertAreaMeasurement(area, unit)` — Convert area to target unit

### On-the-Fly Reprojection

QGIS performs on-the-fly (OTF) reprojection automatically when layers with different CRS are added to the same project. The map canvas renders all layers in the project CRS:

```python
# Set the project CRS
QgsProject.instance().setCrs(QgsCoordinateReferenceSystem("EPSG:3857"))

# All layers are reprojected on-the-fly to EPSG:3857 for rendering
# The underlying layer data is NOT modified
```

### Datum Transformations

When transforming between CRS that use different datums (e.g., NAD27 to WGS84), QGIS may need specific datum transformation grids. These are managed through:

1. **Proj grid files** — Stored in the Proj data directory
2. **Project transform context** — Stores preferred transformations per CRS pair
3. **`QgsCoordinateTransformContext`** — Programmatic access to transformation preferences

```python
context = QgsProject.instance().transformContext()
# The context automatically handles datum transformations
# based on available grid files and user preferences
```

### Common CRS Pitfalls

#### Pitfall 1: Axis Order (lat/lon vs. lon/lat)

The WMS 1.3.0 specification and EPSG database define EPSG:4326 with axis order **latitude, longitude** (north, east). However, most GIS software (including QGIS internally) uses **longitude, latitude** (x, y) order. QGIS handles this internally, but when constructing `QgsPointXY` for EPSG:4326:

```python
# CORRECT — QgsPointXY takes (x, y) = (longitude, latitude)
point = QgsPointXY(5.0, 52.0)  # lon=5.0, lat=52.0

# WRONG — This would place the point in the wrong location
point = QgsPointXY(52.0, 5.0)  # This puts lat in x and lon in y
```

#### Pitfall 2: Invalid CRS Creation

```python
crs = QgsCoordinateReferenceSystem("EPSG:99999")
if not crs.isValid():
    print("CRS is invalid!")  # ALWAYS check validity
```

#### Pitfall 3: Missing srs.db in Standalone Scripts

Standalone scripts require `QgsApplication.setPrefixPath()` to be set correctly so that QGIS can find the `srs.db` database containing CRS definitions. Without this, ALL CRS operations will fail silently or return invalid CRS objects.

#### Pitfall 4: Silent Corruption from Wrong CRS Assignment

Assigning the wrong CRS to a layer does NOT reproject the data — it merely changes the interpretation. This silently corrupts all spatial operations. Use `QgsCoordinateTransform` to actually reproject data.

#### Pitfall 5: Using Deprecated Methods

```python
# DEPRECATED — do not use
crs.createFromSrid(4326)       # Use constructor with prefix instead
crs.createFromEpsg(4326)       # Use constructor with prefix instead

# CORRECT
crs = QgsCoordinateReferenceSystem("EPSG:4326")
```

---

## 6. QGIS 4.0 Changes (Qt6 Migration)

QGIS 4.0 migrates from Qt5/PyQt5 to Qt6/PyQt6. Key breaking changes affecting PyQGIS:

### QMetaType.Type Replacing QVariant.Type

In Qt6, `QVariant.Type` is deprecated and replaced by `QMetaType.Type`:

```python
# QGIS 3.x (Qt5) — WILL BREAK in QGIS 4.x
from qgis.PyQt.QtCore import QVariant
field = QgsField("name", QVariant.String)

# QGIS 4.x (Qt6) — Forward-compatible
from qgis.PyQt.QtCore import QMetaType
field = QgsField("name", QMetaType.Type.QString)
```

Common type mappings:

| Qt5 (QVariant.Type) | Qt6 (QMetaType.Type) |
|---------------------|---------------------|
| `QVariant.String` | `QMetaType.Type.QString` |
| `QVariant.Int` | `QMetaType.Type.Int` |
| `QVariant.Double` | `QMetaType.Type.Double` |
| `QVariant.Bool` | `QMetaType.Type.Bool` |
| `QVariant.Date` | `QMetaType.Type.QDate` |
| `QVariant.DateTime` | `QMetaType.Type.QDateTime` |
| `QVariant.ByteArray` | `QMetaType.Type.QByteArray` |
| `QVariant.LongLong` | `QMetaType.Type.LongLong` |

### Qt6 API Changes Affecting PyQGIS

- `QRegExp` removed, use `QRegularExpression` instead
- Various signal/slot signature changes
- `exec_()` methods renamed to `exec()` (dialogs, event loops)
- Enum scoping changes — unscoped enums become scoped [VERIFY: full list of affected enums]

### Backward Compatibility

QGIS 3.x late releases (3.40+) introduce forward-compatible alternatives. Using `qgis.PyQt` imports helps abstract differences between PyQt5 and PyQt6. The `Qgis` class provides version-independent enum access for many values.

---

## 7. Anti-patterns and Critical Warnings

### Threading Anti-patterns

- **NEVER** access `QgsProject.instance()` from background threads without proper synchronization. The project singleton is NOT thread-safe for writes.
- **NEVER** modify GUI elements (map canvas, layer tree, dialogs) from background threads. Use `QgsTask` and emit signals to communicate results back to the main thread.
- **NEVER** create `QgsVectorLayer` or `QgsRasterLayer` on a background thread and add it to the project directly. Create the data on the background thread, then add the layer on the main thread via signal.

### Layer Validity Anti-patterns

- **NEVER** assume a layer is valid without calling `isValid()`. A layer can be created without errors but still be invalid (wrong path, missing provider, auth failure).
- **NEVER** access features or data provider methods on an invalid layer — this leads to crashes or undefined behavior.
- **ALWAYS** check validity immediately after layer creation:

```python
layer = QgsVectorLayer(uri, name, provider)
if not layer.isValid():
    # Handle error — log, raise, skip
    raise RuntimeError(f"Failed to load layer: {name} from {uri}")
```

### Path Anti-patterns

- **NEVER** hardcode absolute file paths in cross-platform code. Use `QgsProject.instance().absoluteFilePath()`, `QgsApplication.qgisSettingsDirPath()`, or `os.path.join()`.
- **NEVER** use backslashes in paths even on Windows — QGIS/Qt normalizes to forward slashes internally. Using backslashes in URIs causes provider failures.
- **ALWAYS** use `pathlib.Path` or `os.path` for path construction:

```python
import os
data_dir = os.path.join(QgsProject.instance().homePath(), "data")
layer_path = os.path.join(data_dir, "airports.gpkg")
```

### CRS Anti-patterns

- **NEVER** use `QVariant.Type` in code targeting QGIS 4.0+ — use `QMetaType.Type` instead.
- **NEVER** create a `QgsCoordinateTransform` without a `QgsCoordinateTransformContext`. The bare constructor produces transforms that ignore datum transformation preferences:

```python
# WRONG — missing transform context
xform = QgsCoordinateTransform(crs1, crs2)

# CORRECT — with project transform context
xform = QgsCoordinateTransform(crs1, crs2, QgsProject.instance().transformContext())
```

- **NEVER** assume coordinates are in lon/lat order for EPSG:4326 when receiving data from WMS 1.3.0 services — the standard specifies lat/lon axis order.
- **ALWAYS** check for datum transformation availability before transforming between datums that require grid files. Missing grids cause silent loss of precision.
- **ALWAYS** verify CRS validity with `isValid()` after creation — an invalid CRS object silently produces wrong results in all downstream operations.

### Data Provider Anti-patterns

- **NEVER** pass `True` to `uri.uri(expandAuthConfig)` when logging or displaying URIs — this exposes authentication credentials in plain text. Use `uri.uri(False)`.
- **NEVER** assume a GeoPackage layer name — ALWAYS check sublayers or use explicit `|layername=` in the URI.
- **ALWAYS** call `layer.updateExtents()` after adding features to a memory layer — without this, the layer extent is wrong and zoom-to-layer fails.
- **ALWAYS** call `layer.updateFields()` after modifying a layer's field structure via `dataProvider().addAttributes()`.

### Project Anti-patterns

- **NEVER** call `QgsProject.instance().clear()` without confirming that unsaved changes are acceptable — this destroys all loaded layers and settings immediately.
- **NEVER** rely on `QgsProject.instance().mapLayersByName()` returning a single result — layer names are NOT unique. ALWAYS index into the returned list and handle empty lists:

```python
layers = QgsProject.instance().mapLayersByName("Airports")
if not layers:
    raise RuntimeError("Layer 'Airports' not found in project")
layer = layers[0]
```

---

## Sources

1. PyQGIS Developer Cookbook — Introduction: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/intro.html
2. PyQGIS Developer Cookbook — Loading Projects: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/loadproject.html
3. PyQGIS Developer Cookbook — Loading Layers: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/loadlayer.html
4. PyQGIS Developer Cookbook — CRS and Projections: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/crs.html
5. PyQGIS Developer Cookbook — Cheat Sheet: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/cheat_sheet.html
6. QGIS PyQGIS API Reference (3.44): https://qgis.org/pyqgis/3.44/

All code examples are sourced from or verified against the official QGIS 3.44 PyQGIS Developer Cookbook. Items marked with [VERIFY] indicate information that could not be fully confirmed from the fetched sources and should be validated against additional documentation or source code.
