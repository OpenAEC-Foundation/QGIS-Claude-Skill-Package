---
name: qgis-core-architecture
description: >
  Use when creating QGIS projects, understanding PyQGIS architecture, or reasoning about the QGIS data model.
  Prevents mixing standalone script initialization with plugin context, and threading violations.
  Covers Qt-based architecture, 5 core modules, project lifecycle, layer model, provider system, and QGIS 4.0 Qt6 notes.
  Keywords: QGIS architecture, QgsProject, QgsApplication, PyQGIS modules, layer model, provider system, plugin architecture.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-core-architecture

## Quick Reference

### Core Python Modules (QGIS 3.x)

| Module | Purpose |
|--------|---------|
| `qgis.core` | All non-GUI functionality: layers, features, geometry, CRS, project, processing, data providers |
| `qgis.gui` | GUI components: map canvas (`QgsMapCanvas`), layer tree view, attribute forms |
| `qgis.analysis` | Spatial analysis: network analysis, raster calculator, interpolation |
| `qgis.server` | QGIS Server for OGC web services (WMS, WFS, WCS) |
| `qgis._3d` | 3D map visualization components |

Additional utility modules:
- `qgis.utils` -- Automatically imported in the QGIS Python console; provides helper functions
- `qgis.PyQt` -- Re-exports PyQt5/PyQt6 classes for version-independent imports

### Layer Type Hierarchy

| Class | Type Enum | Description |
|-------|-----------|-------------|
| `QgsVectorLayer` | `VectorLayer` | Points, lines, polygons with attributes |
| `QgsRasterLayer` | `RasterLayer` | Grid-based data (GeoTIFF, WMS, XYZ tiles) |
| `QgsMeshLayer` | `MeshLayer` | Unstructured mesh data (hydrodynamic models) |
| `QgsPointCloudLayer` | `PointCloudLayer` | LiDAR / point cloud data (LAS, LAZ, COPC, EPT) |
| `QgsAnnotationLayer` | `AnnotationLayer` | Freeform annotations on the map |
| `QgsVectorTileLayer` | `VectorTileLayer` | Mapbox Vector Tiles (MVT) |
| `QgsTiledSceneLayer` | `TiledSceneLayer` | 3D tiled scenes (3D Tiles, Cesium) |

All layer types inherit from `QgsMapLayer`. Access the type via `layer.type()`.

### Project File Formats

| Format | Extension | Description |
|--------|-----------|-------------|
| QGS | `.qgs` | XML-based project file; human-readable, larger file size |
| QGZ | `.qgz` | Compressed ZIP archive containing .qgs and auxiliary data; default since QGIS 3.2 |

### Key Dependencies

| Component | Role |
|-----------|------|
| Qt5 / Qt6 | Application framework (Qt6 in QGIS 4.x) |
| SIP | C++ to Python binding generator (tighter Qt integration than SWIG) |
| PyQt5 / PyQt6 | Python Qt bindings |
| GDAL/OGR | Raster and vector data I/O |
| PROJ | Coordinate reference system transformations |
| GEOS | Geometry operations |

---

## Critical Warnings

**NEVER** access `QgsProject.instance()` from background threads without proper synchronization. The project singleton is NOT thread-safe for writes.

**NEVER** modify GUI elements (map canvas, layer tree, dialogs) from background threads. ALWAYS use `QgsTask` and emit signals to communicate results back to the main thread.

**NEVER** create layers on a background thread and add them to the project directly. Create the data on the background thread, then add the layer on the main thread via signal.

**NEVER** assume a layer is valid without calling `isValid()`. A layer can be created without errors but still be invalid (wrong path, missing provider, auth failure).

**NEVER** access features or data provider methods on an invalid layer -- this leads to crashes or undefined behavior.

**NEVER** call `QgsProject.instance().clear()` without confirming that unsaved changes are acceptable -- this destroys all loaded layers and settings immediately.

**NEVER** rely on `QgsProject.instance().mapLayersByName()` returning a single result -- layer names are NOT unique. ALWAYS index into the returned list and handle empty lists.

**NEVER** use backslashes in paths even on Windows -- QGIS/Qt normalizes to forward slashes internally. Using backslashes in URIs causes provider failures.

**NEVER** call `QgsApplication([], False)` without first calling `QgsApplication.setPrefixPath()` -- providers will fail to load.

**ALWAYS** call `qgs.exitQgis()` at the end of standalone scripts to prevent memory leaks.

**ALWAYS** check layer validity immediately after creation with `layer.isValid()`.

---

## Decision Tree

### Which Initialization Context?

```
Are you writing code inside the QGIS application?
├── YES (plugin or Python console)
│   └── QgsApplication is ALREADY initialized
│       ├── `iface` is available (QgisInterface)
│       ├── `QgsProject.instance()` is ready
│       └── Do NOT call QgsApplication() or initQgis()
├── NO (standalone script, no GUI)
│   └── MUST initialize QgsApplication manually:
│       1. QgsApplication.setPrefixPath(path, True)
│       2. qgs = QgsApplication([], False)
│       3. qgs.initQgis()
│       4. ... your code ...
│       5. qgs.exitQgis()
└── NO (standalone script, with GUI)
    └── Same as above but pass True to QgsApplication:
        1. QgsApplication.setPrefixPath(path, True)
        2. qgs = QgsApplication([], True)
        3. qgs.initQgis()
        4. ... your GUI code ...
        5. qgs.exitQgis()
```

### Which Layer Class?

```
What kind of data are you loading?
├── Vector data (points, lines, polygons) → QgsVectorLayer
├── Raster/grid data (GeoTIFF, DEM) → QgsRasterLayer
├── Mesh data (hydrodynamic models) → QgsMeshLayer
├── Point cloud (LAS, LAZ, COPC) → QgsPointCloudLayer
├── Vector tiles (MVT) → QgsVectorTileLayer
├── 3D tiled scenes (Cesium 3D Tiles) → QgsTiledSceneLayer
└── Map annotations → QgsAnnotationLayer
```

### Which Data Provider?

```
What is your data source?
├── Local vector file (.shp, .gpkg, .geojson, .fgb, .kml) → provider: "ogr"
├── Local raster file (.tif, .jp2, .vrt) → provider: "gdal"
├── PostgreSQL/PostGIS database → provider: "postgres"
├── SpatiaLite database → provider: "spatialite"
├── In-memory layer → provider: "memory"
├── Delimited text file (.csv with coordinates) → provider: "delimitedtext"
├── WFS web service → provider: "WFS"
├── WMS/WMTS web service → provider: "wms"
├── WCS web service → provider: "wcs"
├── GPX file → provider: "gpx"
└── Virtual layer (SQL over layers) → provider: "virtual"
```

---

## Essential Patterns

### Standalone Script Initialization (No GUI)

```python
from qgis.core import QgsApplication, QgsVectorLayer, QgsProject

QgsApplication.setPrefixPath("/path/to/qgis/installation", True)
qgs = QgsApplication([], False)
qgs.initQgis()

# Load and work with layers
layer = QgsVectorLayer("data/airports.gpkg|layername=airports", "Airports", "ogr")
if not layer.isValid():
    raise RuntimeError("Layer failed to load")

QgsProject.instance().addMapLayer(layer)

# ALWAYS clean up
qgs.exitQgis()
```

### Plugin Entry Point Pattern

```python
class MyPlugin:
    def __init__(self, iface):
        self.iface = iface  # QgisInterface instance

    def initGui(self):
        # Register actions, menus, toolbars
        self.action = QAction("My Plugin", self.iface.mainWindow())
        self.action.triggered.connect(self.run)
        self.iface.addToolBarIcon(self.action)

    def unload(self):
        # ALWAYS clean up all registered elements
        self.iface.removeToolBarIcon(self.action)

    def run(self):
        # Plugin logic here
        layer = self.iface.activeLayer()
        if layer is not None and layer.isValid():
            # Process layer
            pass

def classFactory(iface):
    return MyPlugin(iface)
```

### Project Operations

```python
project = QgsProject.instance()

# Load project
success = project.read("/path/to/project.qgz")

# Load project skipping layer resolution (fast, for metadata access)
readflags = Qgis.ProjectReadFlags()
readflags |= Qgis.ProjectReadFlag.DontResolveLayers
project.read("/path/to/project.qgs", readflags)

# Save project
project.write()                            # Save to current location
project.write("/path/to/new_project.qgz")  # Save to new path

# Access all layers (returns dict of {id: layer})
all_layers = project.mapLayers()

# Find layers by name (returns list -- names are NOT unique)
layers = project.mapLayersByName("Airports")
if not layers:
    raise RuntimeError("Layer 'Airports' not found")
layer = layers[0]
```

### Layer Creation and Validation

```python
# Vector layer
vlayer = QgsVectorLayer("data/airports.shp", "Airports", "ogr")
if not vlayer.isValid():
    raise RuntimeError(f"Failed to load layer: {vlayer.name()}")
QgsProject.instance().addMapLayer(vlayer)

# Raster layer
rlayer = QgsRasterLayer("data/srtm.tif", "SRTM", "gdal")
if not rlayer.isValid():
    raise RuntimeError(f"Failed to load raster: {rlayer.name()}")
QgsProject.instance().addMapLayer(rlayer)

# Memory layer (for temporary data)
mem_layer = QgsVectorLayer("Point?crs=EPSG:4326", "Temp Points", "memory")
```

### Background Processing with QgsTask

```python
from qgis.core import QgsTask, QgsApplication

class HeavyProcessingTask(QgsTask):
    def __init__(self, description, layer_id):
        super().__init__(description, QgsTask.CanCancel)
        self.layer_id = layer_id
        self.result_data = None

    def run(self):
        # Runs on background thread -- NO GUI access here
        # NO QgsProject.instance() writes here
        self.result_data = self._process_data()
        return True

    def finished(self, result):
        # Runs on main thread -- safe to update GUI and project
        if result:
            # Add results to project here
            pass

task = HeavyProcessingTask("Processing data", layer.id())
QgsApplication.taskManager().addTask(task)
```

### Signal/Slot Connections

```python
# Connect to project layer changes
QgsProject.instance().layersAdded.connect(on_layers_added)
QgsProject.instance().layersRemoved.connect(on_layers_removed)

# Connect to map canvas render
iface.mapCanvas().renderComplete.connect(on_render_complete)

# ALWAYS disconnect when done (e.g., in plugin unload)
QgsProject.instance().layersAdded.disconnect(on_layers_added)
iface.mapCanvas().renderComplete.disconnect(on_render_complete)
```

---

## Common Operations

### Path Resolution

```python
# Rewrite paths during loading (e.g., migrating between machines)
def my_load_preprocessor(path):
    return path.replace("c:/Users/Old/", "x:/New/")

preprocessor_id = QgsPathResolver.setPathPreprocessor(my_load_preprocessor)

# Remove preprocessor when no longer needed
QgsPathResolver.removePathPreprocessor(preprocessor_id)
```

### Layer Tree Management

```python
root = QgsProject.instance().layerTreeRoot()

# Create a group
group = root.addGroup("Analysis Results")

# Add layer to specific group (not root)
QgsProject.instance().addMapLayer(layer, False)  # False = don't add to root
group.addLayer(layer)

# Find group or layer node
my_group = root.findGroup("Analysis Results")
node = root.findLayer(layer.id())
```

### Canvas-Project Bridge (Standalone GUI Apps)

```python
from qgis.core import QgsProject
from qgis.gui import QgsLayerTreeMapCanvasBridge

bridge = QgsLayerTreeMapCanvasBridge(
    QgsProject.instance().layerTreeRoot(),
    canvas
)
project.read("/path/to/project.qgs")
```

---

## QGIS 4.0 Migration Notes (Qt6)

### QMetaType.Type Replaces QVariant.Type

```python
# QGIS 3.x (Qt5) -- WILL BREAK in QGIS 4.x
from qgis.PyQt.QtCore import QVariant
field = QgsField("name", QVariant.String)

# QGIS 4.x (Qt6) -- Forward-compatible
from qgis.PyQt.QtCore import QMetaType
field = QgsField("name", QMetaType.Type.QString)
```

| Qt5 (QVariant.Type) | Qt6 (QMetaType.Type) |
|---------------------|---------------------|
| `QVariant.String` | `QMetaType.Type.QString` |
| `QVariant.Int` | `QMetaType.Type.Int` |
| `QVariant.Double` | `QMetaType.Type.Double` |
| `QVariant.Bool` | `QMetaType.Type.Bool` |
| `QVariant.Date` | `QMetaType.Type.QDate` |
| `QVariant.DateTime` | `QMetaType.Type.QDateTime` |
| `QVariant.LongLong` | `QMetaType.Type.LongLong` |

### Other Qt6 Breaking Changes

- `QRegExp` removed -- use `QRegularExpression` instead
- `exec_()` methods renamed to `exec()` (dialogs, event loops)
- Enum scoping changes -- unscoped enums become scoped
- Use `qgis.PyQt` imports to abstract differences between PyQt5 and PyQt6

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for QgsProject, QgsApplication, QgsMapLayer, QgsProviderRegistry
- [references/examples.md](references/examples.md) -- Working code examples verified against PyQGIS 3.44 documentation
- [references/anti-patterns.md](references/anti-patterns.md) -- What NOT to do, with WHY explanations

### Official Sources

- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/intro.html
- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/loadproject.html
- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/loadlayer.html
- https://qgis.org/pyqgis/3.44/
