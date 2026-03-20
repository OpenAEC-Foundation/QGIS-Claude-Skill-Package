# methods.md — qgis-syntax-processing-scripts

## processing.run()

```python
processing.run(
    algorithm_id: str,          # "provider:algorithm" format
    parameters: dict,           # Parameter name → value mapping
    context: QgsProcessingContext = None,
    feedback: QgsProcessingFeedback = None,
    is_child_algorithm: bool = False
) -> dict
```

Returns a dictionary with output keys mapping to results (layer references, file paths, numeric values).

## processing.runAndLoadResults()

```python
processing.runAndLoadResults(
    algorithm_id: str,
    parameters: dict,
    context: QgsProcessingContext = None,
    feedback: QgsProcessingFeedback = None,
    is_child_algorithm: bool = False
) -> dict
```

Same as `processing.run()` but automatically adds output layers to the current QGIS project.

---

## QgsProcessingAlgorithm — Required Methods

### name() -> str

Returns the unique algorithm identifier. MUST be lowercase with no spaces. This becomes the suffix in the algorithm ID (`provider_id:name`).

### displayName() -> str

Returns the user-visible name shown in the Processing Toolbox and dialogs.

### initAlgorithm(config: dict = None) -> None

Defines all input and output parameters using `self.addParameter()` and `self.addOutput()`.

### processAlgorithm(parameters: dict, context: QgsProcessingContext, feedback: QgsProcessingFeedback) -> dict

Core execution logic. MUST return a dictionary with output keys matching declared output parameter names.

### createInstance() -> QgsProcessingAlgorithm

MUST return a new instance of the algorithm class. Required for the Processing framework to create copies.

```python
def createInstance(self):
    return MyAlgorithm()
```

## QgsProcessingAlgorithm — Optional Methods

### group() -> str

Category display name in the Processing Toolbox.

### groupId() -> str

Category identifier string (lowercase, no spaces).

### shortHelpString() -> str

Help text displayed in the algorithm dialog.

### tags() -> list[str]

Search keywords for algorithm discovery.

### flags() -> QgsProcessingAlgorithm.Flags

Algorithm execution flags. Use `FlagNoThreading` for non-thread-safe algorithms:

```python
def flags(self):
    return super().flags() | QgsProcessingAlgorithm.FlagNoThreading
```

### canExecute() -> tuple[bool, str]

Returns whether the algorithm can currently execute and an error message if not.

### checkParameterValues(parameters: dict, context: QgsProcessingContext) -> tuple[bool, str]

Custom parameter validation beyond type checking.

### prepareAlgorithm(parameters: dict, context: QgsProcessingContext, feedback: QgsProcessingFeedback) -> bool

Pre-execution setup that runs on the main thread (before `processAlgorithm`).

### postProcessAlgorithm(context: QgsProcessingContext, feedback: QgsProcessingFeedback) -> dict

Post-execution cleanup that runs on the main thread (after `processAlgorithm`).

---

## QgsProcessingProvider

### Required Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `id()` | `-> str` | Unique provider ID (becomes algorithm prefix) |
| `name()` | `-> str` | User-visible name in Processing Toolbox |
| `loadAlgorithms()` | `-> None` | Register algorithms via `self.addAlgorithm()` |

### Optional Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `longName()` | `-> str` | Extended name in algorithm details |
| `icon()` | `-> QIcon` | Provider icon |
| `svgIconPath()` | `-> str` | SVG icon path |

---

## QgsProcessingContext

| Method | Purpose |
|--------|---------|
| `setProject(QgsProject)` | Set the project for the execution context |
| `project()` | Get the current project |
| `transformContext()` | Get the coordinate transform context |
| `setInvalidGeometryCheck(flag)` | Set invalid geometry handling |
| `getMapLayer(id)` | Retrieve a map layer by ID from context |
| `temporaryLayerStore()` | Access the temporary layer store |

## QgsProcessingFeedback

| Method | Purpose |
|--------|---------|
| `setProgress(float)` | Set progress percentage (0-100) |
| `progressChanged` | Signal emitted when progress changes |
| `isCanceled()` | Check if the user requested cancellation |
| `cancel()` | Request cancellation |
| `pushInfo(str)` | Log an informational message |
| `pushWarning(str)` | Log a warning message |
| `reportError(str, fatalError=False)` | Log an error message |
| `pushDebugInfo(str)` | Log a debug message |
| `pushConsoleInfo(str)` | Log a console-level message |
| `setProgressText(str)` | Set the progress description text |

---

## Input Parameter Types — Full Reference

| Class | Constructor Key Args | Purpose |
|-------|---------------------|---------|
| `QgsProcessingParameterFeatureSource` | `name, description, types=[], optional=False` | Vector layer input with geometry type filtering |
| `QgsProcessingParameterRasterLayer` | `name, description, optional=False` | Single raster layer input |
| `QgsProcessingParameterMeshLayer` | `name, description, optional=False` | Mesh layer input |
| `QgsProcessingParameterMultipleLayers` | `name, description, layerType, optional=False` | Multiple layer input |
| `QgsProcessingParameterMapLayer` | `name, description, optional=False` | Any map layer type |
| `QgsProcessingParameterNumber` | `name, description, type=Double, defaultValue=0, minValue, maxValue, optional` | Numeric input |
| `QgsProcessingParameterDistance` | `name, description, defaultValue, parentParameterName, minValue, maxValue` | CRS-aware distance |
| `QgsProcessingParameterString` | `name, description, defaultValue='', optional=False` | Text input |
| `QgsProcessingParameterBoolean` | `name, description, defaultValue=False` | True/False toggle |
| `QgsProcessingParameterEnum` | `name, description, options=[], defaultValue=0, allowMultiple=False` | Dropdown selection |
| `QgsProcessingParameterField` | `name, description, parentLayerParameterName, type=Any, optional` | Attribute field selection |
| `QgsProcessingParameterExpression` | `name, description, parentLayerParameterName, optional` | QGIS expression input |
| `QgsProcessingParameterCrs` | `name, description, defaultValue='EPSG:4326'` | CRS selection |
| `QgsProcessingParameterExtent` | `name, description, defaultValue, optional` | Geographic bounding box |
| `QgsProcessingParameterPoint` | `name, description, defaultValue, optional` | Single coordinate |
| `QgsProcessingParameterRange` | `name, description, type=Double, defaultValue` | Min/max numeric pair |
| `QgsProcessingParameterBand` | `name, description, parentLayerParameterName, optional` | Raster band selection |
| `QgsProcessingParameterFile` | `name, description, behavior=File, extension, optional` | File path input |
| `QgsProcessingParameterColor` | `name, description, defaultValue, optional` | Color value |
| `QgsProcessingParameterScale` | `name, description, defaultValue, optional` | Map scale |

## Output Parameter Types

| Class | Purpose |
|-------|---------|
| `QgsProcessingParameterFeatureSink` | Vector output layer |
| `QgsProcessingParameterRasterDestination` | Raster output layer |
| `QgsProcessingParameterVectorDestination` | Vector output (file-based) |
| `QgsProcessingParameterFileDestination` | Non-spatial file output |
| `QgsProcessingParameterFolderDestination` | Directory output |
| `QgsProcessingOutputNumber` | Numeric output value (use with `addOutput()`) |
| `QgsProcessingOutputString` | String output value (use with `addOutput()`) |
| `QgsProcessingOutputBoolean` | Boolean output value (use with `addOutput()`) |
| `QgsProcessingOutputMultipleLayers` | Multiple output layers |

## Parameter Read Methods (in processAlgorithm)

| Method | Returns | For Parameter Type |
|--------|---------|--------------------|
| `parameterAsSource()` | `QgsProcessingFeatureSource` | FeatureSource |
| `parameterAsRasterLayer()` | `QgsRasterLayer` | RasterLayer |
| `parameterAsSink()` | `(QgsFeatureSink, str)` | FeatureSink |
| `parameterAsDouble()` | `float` | Number (Double) |
| `parameterAsInt()` | `int` | Number (Integer) |
| `parameterAsBool()` | `bool` | Boolean |
| `parameterAsEnum()` | `int` | Enum |
| `parameterAsString()` | `str` | String, Field |
| `parameterAsExpression()` | `str` | Expression |
| `parameterAsCrs()` | `QgsCoordinateReferenceSystem` | Crs |
| `parameterAsExtent()` | `QgsRectangle` | Extent |
| `parameterAsLayerList()` | `list[QgsMapLayer]` | MultipleLayers |

---

## QgsProcessing Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `QgsProcessing.TEMPORARY_OUTPUT` | `'memory:'` equivalent | Temporary output layer |
| `QgsProcessing.TypeVectorAnyGeometry` | — | Accept any vector geometry |
| `QgsProcessing.TypeVectorPoint` | — | Point geometry only |
| `QgsProcessing.TypeVectorLine` | — | Line geometry only |
| `QgsProcessing.TypeVectorPolygon` | — | Polygon geometry only |
| `QgsProcessing.TypeRaster` | — | Raster layer |
| `QgsProcessing.TypeMesh` | — | Mesh layer |

## QgsProcessingParameterNumber Types

| Constant | Purpose |
|----------|---------|
| `QgsProcessingParameterNumber.Integer` | Integer values |
| `QgsProcessingParameterNumber.Double` | Floating-point values |

## QgsProcessingParameterField Types

| Constant | Purpose |
|----------|---------|
| `QgsProcessingParameterField.Any` | Any field type |
| `QgsProcessingParameterField.Numeric` | Numeric fields only |
| `QgsProcessingParameterField.String` | String fields only |
| `QgsProcessingParameterField.DateTime` | Date/time fields only |

---

## Common Algorithm IDs

### Vector General

| Algorithm ID | Purpose |
|-------------|---------|
| `native:buffer` | Buffer features by distance |
| `native:dissolve` | Dissolve features (optionally by field) |
| `native:clip` | Clip vector layer by mask |
| `native:intersection` | Intersect two vector layers |
| `native:union` | Union two vector layers |
| `native:difference` | Subtract one layer from another |
| `native:reprojectlayer` | Reproject to different CRS |
| `native:mergevectorlayers` | Merge multiple layers into one |
| `native:splitvectorlayer` | Split by attribute into files |
| `native:extractbyattribute` | Filter features by attribute value |
| `native:extractbyexpression` | Filter features by expression |
| `native:joinattributesbylocation` | Spatial join |
| `native:joinattributestable` | Table join by field |
| `native:centroids` | Calculate polygon centroids |
| `native:voronoipolygons` | Create Voronoi/Thiessen polygons |
| `native:convexhull` | Create convex hull |
| `native:fixgeometries` | Repair invalid geometries |

### Raster

| Algorithm ID | Purpose |
|-------------|---------|
| `gdal:warpreproject` | Reproject raster |
| `gdal:cliprasterbyextent` | Clip raster to extent |
| `gdal:cliprasterbymask` | Clip raster by vector mask |
| `gdal:merge` | Merge raster files |
| `gdal:translate` | Convert raster format |
| `native:rasterlayerstatistics` | Calculate raster statistics |
| `native:zonalstatisticsfb` | Zonal statistics (feature-based) |
