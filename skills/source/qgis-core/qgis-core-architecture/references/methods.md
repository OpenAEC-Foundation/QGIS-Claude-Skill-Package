# API Signatures Reference (QGIS 3.44+ / PyQGIS 3.x)

## QgsApplication

The application singleton. Manages initialization, provider loading, and global settings.

```python
class QgsApplication(QApplication):
    # Static methods -- call BEFORE constructing QgsApplication
    @staticmethod
    def setPrefixPath(path: str, useDefaultPaths: bool = True) -> None
        # Sets the QGIS installation prefix path.
        # MUST be called before initQgis().
        # useDefaultPaths: if True, default plugin/data paths are derived from prefix.

    @staticmethod
    def prefixPath() -> str
        # Returns the current prefix path.

    @staticmethod
    def pluginPath() -> str
        # Returns the path to installed plugins.

    @staticmethod
    def pkgDataPath() -> str
        # Returns the path to the package data directory.

    @staticmethod
    def qgisSettingsDirPath() -> str
        # Returns the path to the QGIS settings directory.

    @staticmethod
    def qgisUserDatabaseFilePath() -> str
        # Returns the path to the user CRS database.

    # Constructor
    def __init__(self, argv: list, GUIenabled: bool) -> None
        # argv: command line arguments (pass [] for scripts)
        # GUIenabled: False for headless/server, True for GUI applications

    # Instance methods
    def initQgis(self) -> None
        # Loads data providers, initializes auth system, sets up CRS database.
        # MUST be called after constructor.

    def exitQgis(self) -> None
        # Removes provider and layer registries from memory.
        # ALWAYS call this to prevent memory leaks in standalone scripts.

    @staticmethod
    def taskManager() -> QgsTaskManager
        # Returns the application's task manager for background processing.

    @staticmethod
    def processingRegistry() -> QgsProcessingRegistry
        # Returns the processing algorithm registry.

    @staticmethod
    def instance() -> QgsApplication
        # Returns the singleton application instance.

    @staticmethod
    def authManager() -> QgsAuthManager
        # Returns the authentication manager.
```

---

## QgsProject

The project singleton. Manages layers, settings, CRS, and project I/O.

```python
class QgsProject(QObject):
    @staticmethod
    def instance() -> QgsProject
        # Returns the singleton project instance.
        # NOT thread-safe for write operations.

    # Project I/O
    def read(self, filename: str, flags: Qgis.ProjectReadFlags = Qgis.ProjectReadFlags()) -> bool
        # Reads a project file (.qgs or .qgz). Returns True on success.

    def write(self, filename: str = None) -> bool
        # Writes the project. If filename is None, saves to current location.
        # Returns True on success.

    def clear(self) -> None
        # Removes all layers, settings, and resets the project.
        # WARNING: destroys everything immediately.

    def fileName(self) -> str
        # Returns the project file path.

    def absoluteFilePath(self) -> str
        # Returns the absolute path to the project file.

    def homePath(self) -> str
        # Returns the project home directory (directory containing the project file).

    # Layer management
    def addMapLayer(self, layer: QgsMapLayer, addToLegend: bool = True) -> QgsMapLayer
        # Adds a layer to the project.
        # addToLegend: if False, layer is not shown in layer tree.
        # Returns the added layer (or None on failure).

    def addMapLayers(self, layers: list[QgsMapLayer], addToLegend: bool = True) -> list[QgsMapLayer]
        # Adds multiple layers at once.

    def removeMapLayer(self, layerId: str) -> None
        # Removes a layer by its unique ID string.

    def removeMapLayers(self, layerIds: list[str]) -> None
        # Removes multiple layers by ID.

    def removeAllMapLayers(self) -> None
        # Removes all layers from the project.

    def mapLayers(self) -> dict[str, QgsMapLayer]
        # Returns all layers as {layerId: layer} dict.

    def mapLayersByName(self, name: str) -> list[QgsMapLayer]
        # Returns layers matching the given name.
        # Names are NOT unique -- ALWAYS handle list results.

    def mapLayer(self, layerId: str) -> QgsMapLayer
        # Returns a single layer by its unique ID, or None.

    def layerTreeRoot(self) -> QgsLayerTree
        # Returns the root of the layer tree (Table of Contents).

    # CRS and transforms
    def crs(self) -> QgsCoordinateReferenceSystem
        # Returns the project CRS.

    def setCrs(self, crs: QgsCoordinateReferenceSystem) -> None
        # Sets the project CRS.

    def transformContext(self) -> QgsCoordinateTransformContext
        # Returns the project's coordinate transform context.
        # ALWAYS pass this to QgsCoordinateTransform constructors.

    def ellipsoid(self) -> str
        # Returns the project ellipsoid (e.g., "WGS84").

    # Signals
    layersAdded = pyqtSignal(list)       # Emitted when layers are added
    layersRemoved = pyqtSignal(list)     # Emitted when layers are removed (list of IDs)
    layerWasAdded = pyqtSignal(QgsMapLayer)  # Emitted for each individual layer added
    cleared = pyqtSignal()               # Emitted when the project is cleared
    readComplete = pyqtSignal()          # Emitted after project read completes
    writeComplete = pyqtSignal()         # Emitted after project write completes
```

---

## QgsMapLayer

Abstract base class for all layer types.

```python
class QgsMapLayer(QObject):
    def id(self) -> str
        # Returns the unique layer ID (auto-generated, stable within session).

    def name(self) -> str
        # Returns the display name.

    def setName(self, name: str) -> None
        # Sets the display name.

    def type(self) -> Qgis.LayerType
        # Returns the layer type enum (VectorLayer, RasterLayer, etc.).

    def isValid(self) -> bool
        # Returns True if the layer loaded successfully.
        # ALWAYS check this after creation.

    def crs(self) -> QgsCoordinateReferenceSystem
        # Returns the layer's CRS.

    def setCrs(self, crs: QgsCoordinateReferenceSystem) -> None
        # Sets the layer's CRS.

    def extent(self) -> QgsRectangle
        # Returns the layer's spatial extent.

    def source(self) -> str
        # Returns the data source URI string.

    def providerType(self) -> str
        # Returns the provider name (e.g., "ogr", "gdal", "postgres").

    def dataProvider(self) -> QgsDataProvider
        # Returns the data provider instance.

    def setCustomProperty(self, key: str, value) -> None
        # Stores a custom key-value property on the layer.

    def customProperty(self, key: str, defaultValue=None) -> Any
        # Retrieves a custom property.

    def clone(self) -> QgsMapLayer
        # Returns a deep copy of the layer.
```

---

## QgsVectorLayer

Vector layer with features (points, lines, polygons) and attributes.

```python
class QgsVectorLayer(QgsMapLayer):
    def __init__(self, uri: str, baseName: str, providerLib: str) -> None
        # uri: data source URI (provider-specific format)
        # baseName: display name in layer tree
        # providerLib: provider identifier ("ogr", "postgres", "memory", etc.)

    def featureCount(self) -> int
        # Returns the number of features.

    def fields(self) -> QgsFields
        # Returns the field (attribute) schema.

    def geometryType(self) -> Qgis.GeometryType
        # Returns Point, Line, Polygon, UnknownGeometry, or NullGeometry.

    def wkbType(self) -> Qgis.WkbType
        # Returns the WKB geometry type (more specific than geometryType).

    def getFeatures(self, request: QgsFeatureRequest = QgsFeatureRequest()) -> QgsFeatureIterator
        # Returns an iterator over features matching the request.

    def getFeature(self, fid: int) -> QgsFeature
        # Returns a single feature by feature ID.

    def startEditing(self) -> bool
        # Starts an edit session on the layer.

    def commitChanges(self) -> bool
        # Commits pending changes to the data provider.

    def rollBack(self) -> bool
        # Rolls back pending changes.

    def isEditable(self) -> bool
        # Returns True if the layer is in edit mode.

    def updateExtents(self) -> None
        # Recalculates the layer extent.
        # ALWAYS call after adding features to a memory layer.

    def updateFields(self) -> None
        # Refreshes the field list from the data provider.
        # ALWAYS call after modifying field structure via dataProvider().addAttributes().

    def selectByExpression(self, expression: str) -> None
        # Selects features matching the expression.

    def selectedFeatures(self) -> list[QgsFeature]
        # Returns currently selected features.

    def selectedFeatureCount(self) -> int
        # Returns the number of selected features.

    def renderer(self) -> QgsFeatureRenderer
        # Returns the layer's renderer (symbology).

    def setRenderer(self, renderer: QgsFeatureRenderer) -> None
        # Sets the layer's renderer.
```

---

## QgsRasterLayer

Raster layer for grid-based data.

```python
class QgsRasterLayer(QgsMapLayer):
    def __init__(self, uri: str, baseName: str, providerType: str = "gdal") -> None
        # uri: file path or provider-specific URI
        # baseName: display name
        # providerType: provider identifier ("gdal", "wms", "wcs")

    def width(self) -> int
        # Returns raster width in pixels.

    def height(self) -> int
        # Returns raster height in pixels.

    def bandCount(self) -> int
        # Returns the number of raster bands.

    def bandName(self, bandNo: int) -> str
        # Returns the name of a specific band (1-based index).

    def rasterUnitsPerPixelX(self) -> float
        # Returns the horizontal resolution.

    def rasterUnitsPerPixelY(self) -> float
        # Returns the vertical resolution.

    def renderer(self) -> QgsRasterRenderer
        # Returns the raster renderer.

    def setRenderer(self, renderer: QgsRasterRenderer) -> None
        # Sets the raster renderer.
```

---

## QgsProviderRegistry

Singleton registry for all data providers.

```python
class QgsProviderRegistry:
    @staticmethod
    def instance() -> QgsProviderRegistry
        # Returns the singleton provider registry.

    def providerList(self) -> list[str]
        # Returns a list of all registered provider names.
        # Example: ["ogr", "gdal", "postgres", "memory", "wms", "wfs", ...]

    def providerMetadata(self, providerKey: str) -> QgsProviderMetadata
        # Returns metadata for a specific provider.

    def providerCapabilities(self, providerKey: str) -> int
        # Returns capability flags for a provider.

    def createProvider(self, providerKey: str, dataSource: str) -> QgsDataProvider
        # Creates a new data provider instance.

    def library(self, providerKey: str) -> str
        # Returns the library path for a provider.
```

---

## QgsPathResolver

Static methods for path preprocessing during project load/save.

```python
class QgsPathResolver:
    @staticmethod
    def setPathPreprocessor(preprocessor: Callable[[str], str]) -> str
        # Registers a function to rewrite paths during project loading.
        # Returns an ID for later removal.

    @staticmethod
    def setPathWriter(writer: Callable[[str], str]) -> str
        # Registers a function to rewrite paths during project saving.
        # Returns an ID for later removal.

    @staticmethod
    def removePathPreprocessor(id: str) -> None
        # Removes a previously registered path preprocessor.

    @staticmethod
    def removePathWriter(id: str) -> None
        # Removes a previously registered path writer.
```

---

## QgsTask

Base class for background tasks (subclass of QRunnable).

```python
class QgsTask(QObject):
    CanCancel = ...  # Flag indicating the task can be cancelled

    def __init__(self, description: str, flags: QgsTask.Flags = QgsTask.Flags()) -> None

    def run(self) -> bool
        # Override this. Runs on a BACKGROUND thread.
        # NEVER access GUI or QgsProject.instance() for writes here.
        # Return True for success, False for failure.

    def finished(self, result: bool) -> None
        # Override this. Runs on the MAIN thread after run() completes.
        # Safe to update GUI and project here.

    def isCanceled(self) -> bool
        # Check if cancellation was requested.

    def setProgress(self, progress: float) -> None
        # Report progress (0-100).
```
