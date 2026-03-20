# Vooronderzoek: QGIS Processing Framework

> Deep research document for the QGIS Claude Skill Package.
> Serves as knowledge base for the processing-scripts syntax skill and multiple impl skills.
> Sources: Official QGIS 3.x documentation (docs.qgis.org/latest).

---

## 1. Processing Framework Architecture

### Provider-Based System

The QGIS Processing framework is a provider-based architecture that exposes geospatial algorithms through a unified interface. Every algorithm belongs to a **provider**, and the framework manages algorithm registration, discovery, parameter handling, and execution. The central registry is accessed via `QgsApplication.processingRegistry()`.

### Built-in Providers

| Provider | ID Prefix | Purpose |
|----------|-----------|---------|
| Native QGIS | `native:` | Core vector/raster operations written in C++ |
| QGIS (legacy Python) | `qgis:` | Older Python-based QGIS algorithms |
| GDAL | `gdal:` | GDAL/OGR-based raster and vector operations |
| GRASS | `grass:` | GRASS GIS algorithms (requires GRASS installation) |
| SAGA (deprecated) | `saga:` | SAGA GIS algorithms (removed in recent versions) |
| OTB | `otb:` | Orfeo ToolBox remote sensing algorithms |
| PDAL | `pdal:` | Point cloud processing via PDAL |

### Algorithm Registration and Discovery

Algorithms are registered through their providers. The processing registry maintains a catalog of all available algorithms. You can query it at runtime:

```python
from qgis.core import QgsApplication

registry = QgsApplication.processingRegistry()

# List all providers
for provider in registry.providers():
    print(provider.id(), provider.name())

# List all algorithms from a specific provider
for alg in registry.algorithms():
    if alg.provider().id() == 'native':
        print(alg.id(), alg.displayName())

# Look up a specific algorithm
alg = registry.algorithmById('native:buffer')
if alg:
    print(alg.shortHelpString())
```

### QgsProcessing Class and Constants

The `QgsProcessing` class provides constants used throughout the framework:

- `QgsProcessing.TypeVectorAnyGeometry` — any vector geometry type
- `QgsProcessing.TypeVectorPoint` — point geometry
- `QgsProcessing.TypeVectorLine` — line geometry
- `QgsProcessing.TypeVectorPolygon` — polygon geometry
- `QgsProcessing.TypeRaster` — raster layer
- `QgsProcessing.TypeMesh` — mesh layer
- `QgsProcessing.TEMPORARY_OUTPUT` — constant for temporary output layers

### processing.run() Entry Point

The primary entry point for executing algorithms from Python is `processing.run()`:

```python
import processing

result = processing.run(
    "algorithm_id",       # e.g., "native:buffer"
    {                     # Parameter dictionary
        'INPUT': layer,
        'DISTANCE': 100,
        'OUTPUT': 'memory:'  # or file path, or QgsProcessing.TEMPORARY_OUTPUT
    },
    context=context,      # Optional QgsProcessingContext
    feedback=feedback     # Optional QgsProcessingFeedback
)
```

The function returns a dictionary with output keys mapping to the results (layer references, file paths, numeric values, etc.).

### Execution Architecture

Processing algorithms execute in background threads by default. The framework provides:

- **QgsProcessingContext**: Holds information about the execution environment (project, transform context, default CRS, invalid geometry handling).
- **QgsProcessingFeedback**: Communication channel between algorithm and caller for progress updates, log messages, and cancellation.

Projection handling: algorithms ALWAYS execute in the input layer's CRS. When multiple input layers have different CRSs, QGIS may reproject on-the-fly but warns about potential issues.

---

## 2. Running Algorithms from PyQGIS

### Basic Pattern: processing.run()

The standard pattern for running any processing algorithm:

```python
import processing

# Buffer example
result = processing.run("native:buffer", {
    'INPUT': 'path/to/input.gpkg|layername=buildings',
    'DISTANCE': 50,
    'SEGMENTS': 5,
    'END_CAP_STYLE': 0,  # Round
    'JOIN_STYLE': 0,      # Round
    'MITER_LIMIT': 2,
    'DISSOLVE': False,
    'OUTPUT': 'memory:'
})

buffered_layer = result['OUTPUT']
```

### Parameter Dictionary Structure

Parameters are passed as a Python dictionary where:
- **Keys** are parameter names (uppercase by convention: `INPUT`, `OUTPUT`, `DISTANCE`)
- **Values** can be:
  - Layer objects (`QgsVectorLayer`, `QgsRasterLayer`)
  - File paths as strings (`'/path/to/file.gpkg'`)
  - URI strings with layer names (`'path/to/file.gpkg|layername=roads'`)
  - Numeric values (`100`, `0.5`)
  - Boolean values (`True`, `False`)
  - Enum indices (`0`, `1`, `2`)
  - Expression strings for expression parameters
  - `None` for optional parameters to skip
  - `'memory:'` or `QgsProcessing.TEMPORARY_OUTPUT` for temporary outputs

### Input Parameter Handling

Input layers can be specified in multiple ways:

```python
# As a QgsMapLayer object
result = processing.run("native:buffer", {
    'INPUT': my_vector_layer,
    'DISTANCE': 10,
    'OUTPUT': 'memory:'
})

# As a file path
result = processing.run("native:buffer", {
    'INPUT': '/data/buildings.shp',
    'DISTANCE': 10,
    'OUTPUT': 'memory:'
})

# As a GeoPackage layer URI
result = processing.run("native:buffer", {
    'INPUT': '/data/project.gpkg|layername=buildings',
    'DISTANCE': 10,
    'OUTPUT': 'memory:'
})
```

### Temporary Outputs vs File Outputs

```python
# Temporary output (memory layer, lost when QGIS closes)
result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 10,
    'OUTPUT': 'memory:'
})

# Equivalent using constant
result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 10,
    'OUTPUT': QgsProcessing.TEMPORARY_OUTPUT
})

# File output (persisted to disk)
result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 10,
    'OUTPUT': '/output/buffered.gpkg'
})

# GeoPackage with layer name
result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 10,
    'OUTPUT': 'ogr:dbname=/output/results.gpkg table=buffered_buildings'
})
```

Default output formats: `.gpkg` for vectors, `.tif` for rasters, `.dbf` for tables.

### Running with Feedback (Progress, Cancellation)

```python
from qgis.core import QgsProcessingFeedback

feedback = QgsProcessingFeedback()

# Connect to signals for progress monitoring
feedback.progressChanged.connect(lambda progress: print(f"Progress: {progress}%"))

result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 100,
    'OUTPUT': 'memory:'
}, feedback=feedback)
```

### Running as Background Task (QgsProcessingAlgRunnerTask)

For non-blocking execution in GUI applications:

```python
from qgis.core import (
    QgsApplication,
    QgsProcessingAlgRunnerTask,
    QgsProcessingContext,
    QgsProcessingFeedback,
    QgsProject
)

context = QgsProcessingContext()
context.setProject(QgsProject.instance())
feedback = QgsProcessingFeedback()

alg = QgsApplication.processingRegistry().algorithmById('native:buffer')
params = {
    'INPUT': layer,
    'DISTANCE': 100,
    'OUTPUT': 'memory:'
}

task = QgsProcessingAlgRunnerTask(alg, params, context, feedback)

def on_complete(successful, results):
    if successful:
        output_layer = context.getMapLayer(results['OUTPUT'])
        QgsProject.instance().addMapLayer(output_layer)
    else:
        print("Algorithm failed")

task.executed.connect(on_complete)
QgsApplication.taskManager().addTask(task)
```

### Error Handling

```python
import processing
from qgis.core import QgsProcessingException

try:
    result = processing.run("native:buffer", {
        'INPUT': layer,
        'DISTANCE': 100,
        'OUTPUT': 'memory:'
    })
except QgsProcessingException as e:
    print(f"Processing error: {e}")
```

ALWAYS wrap `processing.run()` calls in try/except blocks. Common failure causes:
- Invalid parameter values
- Missing input layers
- Invalid geometries in input data
- Insufficient disk space for output
- Algorithm not available (provider not loaded)

### Adding Result to Project

Use `processing.runAndLoadResults()` to automatically add output layers to the current project:

```python
result = processing.runAndLoadResults("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 100,
    'OUTPUT': 'memory:'
})
```

---

## 3. Algorithm Categories and Key Algorithm IDs

### Vector General

| Algorithm ID | Purpose |
|-------------|---------|
| `native:reprojectlayer` | Reproject layer to different CRS |
| `native:mergevectorlayers` | Merge multiple vector layers into one |
| `native:splitvectorlayer` | Split layer by attribute values into separate files |
| `native:joinattributesbyfieldvalue` | Join attributes by matching field values |
| `native:joinattributesbylocation` | Spatial join based on geometry relationship |
| `native:joinattributesbylocationsummary` | Spatial join with statistical summaries |
| `native:joinattributesbynearest` | Join by nearest feature |
| `native:createspatialindex` | Create spatial index for faster queries |
| `native:deleteduplicategeometries` | Remove duplicate geometries |
| `native:extractselectedfeatures` | Extract currently selected features |
| `native:orderbyexpression` | Sort features by expression |
| `native:definecurrentprojection` | Assign CRS without reprojecting |
| `native:dropmzvalues` | Remove M/Z values from geometries |
| `native:executesql` | Execute SQL on layers |
| `native:detectdatasetchanges` | Detect changes between two layers |
| `native:savefeatures` | Save vector features to file |
| `native:setlayerencoding` | Set layer character encoding |
| `native:truncatetable` | Remove all features from a layer |

### Vector Geometry

| Algorithm ID | Purpose |
|-------------|---------|
| `native:buffer` | Create buffer around features |
| `native:centroids` | Calculate centroids |
| `native:simplifygeometries` | Simplify geometries (Douglas-Peucker) |
| `native:dissolve` | Dissolve features by attribute |
| `native:aggregate` | Aggregate features with expressions |
| `native:polygonstolines` | Convert polygons to lines |
| `native:linestopolygons` | Convert lines to polygons |
| `native:multiparttosingleparts` | Split multi-part to single-part |
| `native:promotetomulti` | Promote single-part to multi-part |
| `native:fixgeometries` | Repair invalid geometries |
| `native:convexhull` | Create convex hull |
| `native:concavehull` | Create concave hull |
| `native:delaunaytriangulation` | Delaunay triangulation |
| `native:voronoipolygons` | Create Voronoi polygons |
| `native:collectgeometries` | Merge features into multi-part |
| `native:densifygeometries` | Add vertices along geometries |
| `native:densifygeometriesgivenaninterval` | Densify by interval |
| `native:snappointstogrid` | Snap points to grid |
| `native:snapgeometries` | Snap geometries to reference layer |
| `native:removenullgeometries` | Remove features with null geometry |
| `native:removeduplicatevertices` | Clean duplicate vertices |
| `native:offsetline` | Offset lines |
| `native:smoothgeometry` | Smooth geometries |
| `native:extendlines` | Extend lines by specified amount |
| `native:addgeometrycolumns` | Add geometry attributes (area, length, etc.) |
| `native:geometrybyexpression` | Create geometry from expression |
| `native:boundary` | Extract boundary of geometries |
| `native:boundingboxes` | Create bounding boxes |
| `native:minimumboundinggeometry` | Minimum bounding geometry |
| `native:orientedminimumboundingbox` | Oriented minimum bounding box |
| `native:subdivide` | Subdivide geometries |
| `native:transect` | Create transects on lines |
| `native:wedgebuffers` | Create wedge-shaped buffers |
| `native:taperedbuffer` | Create tapered buffers |
| `native:singlesidedbuffer` | Buffer on one side only |
| `native:multiringconstantbuffer` | Multi-ring buffers |
| `native:pointonsurface` | Point guaranteed on surface |
| `native:poleofinaccessibility` | Most interior point |
| `native:pointsalonggeometry` | Generate points along geometry |
| `native:setmvalue` | Set M values |
| `native:setzvalue` | Set Z values |
| `native:drapesetvaluesfromraster` | Set Z from raster (drape) |
| `native:setmfromraster` | Set M from raster |
| `native:swapxy` | Swap X and Y coordinates |
| `native:reverselinedirection` | Reverse line direction |
| `native:explodelines` | Break lines at vertices |
| `native:linesubstring` | Extract portion of lines |
| `native:interpolatepoint` | Interpolate point on line |
| `native:convertgeometrytype` | Convert geometry type |
| `native:converttocurves` | Convert to curved geometries |
| `native:deleteholes` | Delete polygon holes |
| `native:eliminateselectedpolygons` | Merge selected polygons with neighbors |
| `native:forcerhr` | Force right-hand-rule |
| `native:roundness` | Calculate polygon roundness |
| `native:rectanglesovalsdiamonds` | Create rectangles, ovals, diamonds |
| `native:tessellate` | Tessellate polygons |
| `native:translate` | Translate (shift) geometries |
| `native:rotatefeatures` | Rotate features |
| `native:affinetransform` | Affine transformation |
| `native:segmentizebymaxangle` | Segmentize by angle |
| `native:segmentizebymaxdist` | Segmentize by distance |

### Vector Overlay

| Algorithm ID | Purpose |
|-------------|---------|
| `native:intersection` | Geometric intersection |
| `native:union` | Geometric union |
| `native:difference` | Geometric difference |
| `native:clip` | Clip layer by another |
| `native:symmetricaldifference` | Symmetrical difference |
| `native:multidifference` | Difference with multiple layers |
| `native:multiintersection` | Intersection with multiple layers |
| `native:multiunion` | Union with multiple layers |
| `native:splitwithlines` | Split polygons with lines |
| `native:lineintersections` | Find line intersection points |
| `native:extractbyextent` | Extract/clip by extent |

### Vector Selection

| Algorithm ID | Purpose |
|-------------|---------|
| `native:selectbyexpression` | Select by expression |
| `native:selectbylocation` | Select by spatial relationship |
| `native:selectwithindistance` | Select within distance |
| `native:extractbyexpression` | Extract features by expression |
| `native:extractbyattribute` | Extract by attribute value |
| `native:extractbylocation` | Extract by spatial relationship |
| `native:extractwithindistance` | Extract within distance |
| `native:randomextract` | Random feature extraction |
| `native:randomselection` | Random feature selection |
| `native:randomselectionwithinsubsets` | Random selection within groups |
| `native:filterbygeometry` | Filter by geometry type |

### Vector Creation

| Algorithm ID | Purpose |
|-------------|---------|
| `native:createpointslayerfromtable` | Create point layer from X/Y columns |
| `native:randompointsinextent` | Random points in extent |
| `native:randompointsinpolygons` | Random points inside polygons |
| `native:randompointsonlines` | Random points on lines |
| `native:regularpoints` | Create regular point grid |
| `native:creategrid` | Create vector grid |
| `native:createconstantrasterlayer` | Create constant raster |
| `native:pixelstopoints` | Raster pixels to points |
| `native:pixelstopolygons` | Raster pixels to polygons |
| `native:pointstopath` | Convert points to path |
| `native:arrayoffsetlines` | Array of parallel lines |
| `native:arraytranslatedfeatures` | Array of translated features |
| `native:importgeotaggedphotos` | Import geotagged photos |

### Vector Analysis

| Algorithm ID | Purpose |
|-------------|---------|
| `native:countpointsinpolygon` | Count points in polygons |
| `native:distancematrix` | Distance matrix |
| `native:distancetonearesthublines` | Distance to nearest hub (lines) |
| `native:distancetonearesthubpoints` | Distance to nearest hub (points) |
| `native:meancoordinates` | Mean coordinates |
| `native:nearestneighbouranalysis` | Nearest neighbour analysis |
| `native:sumlinelengths` | Sum line lengths |
| `native:basicstatisticsforfields` | Basic field statistics |
| `native:statisticsbycategories` | Statistics by categories |
| `native:listuniquevalues` | List unique values |
| `native:dbscanclustering` | DBSCAN clustering |
| `native:stdbscanclustering` | ST-DBSCAN clustering |
| `native:kmeansclustering` | K-means clustering |
| `native:overlapanalysis` | Overlap analysis |
| `native:shortestline` | Shortest line between features |
| `native:climbalongline` | Climb along line |
| `native:joinbylines` | Hub lines |

### Vector Table

| Algorithm ID | Purpose |
|-------------|---------|
| `native:fieldcalculator` | Calculate field values |
| `native:addfieldtoattributestable` | Add new field |
| `native:addautoincrementalfield` | Add auto-incremental field |
| `native:adduniquevalueindexfield` | Add unique value index |
| `native:addxyfieldstolayer` | Add X/Y coordinate fields |
| `native:deletecolumn` | Delete field(s) |
| `native:refactorfields` | Refactor fields (rename, reorder, retype) |
| `native:retainfields` | Retain specified fields only |
| `native:renametablefield` | Rename field |
| `native:texttofloat` | Convert text to float |
| `native:advancedpythonfieldcalculator` | Python field calculator |
| `native:explodehstorefield` | Explode HStore field |
| `native:extractbinaryfield` | Extract binary field |

### Raster Analysis

| Algorithm ID | Purpose |
|-------------|---------|
| `native:rastercalculator` | Raster calculator |
| `native:rasterlayerstatistics` | Raster layer statistics |
| `native:rasterlayeruniquevaluesreport` | Unique values report |
| `native:rasterlayerzonalstats` | Raster layer zonal statistics |
| `native:zonalstatisticsfb` | Zonal statistics (feature-based) |
| `native:zonalhistogram` | Zonal histogram |
| `native:rastersampling` | Sample raster values at points |
| `native:cellstatistics` | Cell statistics (multi-raster) |
| `native:rasterbooleanand` | Boolean AND across rasters |
| `native:rasterbooleanor` | Boolean OR across rasters |
| `native:rastercalc` | Raster calculator (virtual) |
| `native:reclassifybylayer` | Reclassify by layer |
| `native:reclassifybytable` | Reclassify by table |
| `native:rescaleraster` | Rescale raster values |
| `native:roundrastervalues` | Round raster values |
| `native:rastersurfacevolume` | Surface volume calculation |
| `native:fuzzifyrastergaussianmembership` | Fuzzy gaussian membership |
| `native:fuzzifyrasterlargemembership` | Fuzzy large membership |
| `native:fuzzifyrasterlinearmembership` | Fuzzy linear membership |
| `native:fuzzifyrasternearmembership` | Fuzzy near membership |
| `native:fuzzifyrasterpowermembership` | Fuzzy power membership |
| `native:fuzzifyrastersmallmembership` | Fuzzy small membership |
| `native:equaltofrequency` | Equal to frequency |
| `native:greaterthanfrequency` | Greater than frequency |
| `native:lessthanfrequency` | Less than frequency |
| `native:highestpositioninrasterstack` | Highest position in stack |
| `native:lowestpositioninrasterstack` | Lowest position in stack |
| `native:cellstackpercentile` | Cell stack percentile |
| `native:cellstackpercentrankfromvalue` | Percent rank from value |
| `native:cellstackpercentrankfromrasterlayer` | Percent rank from raster |
| `native:rasterlayerproperties` | Raster layer properties |
| `native:rasterminmax` | Raster min/max values |
| `native:zonalminmaxpoint` | Zonal min/max point |

### Raster Terrain Analysis

| Algorithm ID | Purpose |
|-------------|---------|
| `native:hillshade` | Hillshade rendering |
| `native:slope` | Slope calculation |
| `native:aspect` | Aspect calculation |
| `native:ruggednessindex` | Terrain ruggedness index |
| `native:relief` | Relief rendering |
| `native:hypsometriccurves` | Hypsometric curves |
| `native:fillsinks` | Fill sinks (Wang & Liu) |
| `native:dtmfilterslopebase` | DTM filter (slope-based) |

### Raster Tools

| Algorithm ID | Purpose |
|-------------|---------|
| `native:alignrasters` | Align rasters |
| `native:fillnodata` | Fill NoData cells |
| `native:generatexyztiles` | Generate XYZ tiles (directory) |
| `native:generatexyztilesmbtiles` | Generate XYZ tiles (MBTiles) |
| `native:convertmaptoraster` | Convert map to raster |

### Raster Creation

| Algorithm ID | Purpose |
|-------------|---------|
| `native:createconstantrasterlayer` | Constant raster |
| `native:createrandomnormalrasterlayer` | Random normal distribution |
| `native:createrandomuniformrasterlayer` | Random uniform distribution |
| `native:createrandombinomialrasterlayer` | Random binomial distribution |
| `native:createrandomexponentialrasterlayer` | Random exponential distribution |
| `native:createrandomgammarasterlayer` | Random gamma distribution |
| `native:createrandomgeometricrasterlayer` | Random geometric distribution |
| `native:createrandomnegativebinomialrasterlayer` | Random negative binomial |
| `native:createrandompoissonrasterlayer` | Random Poisson distribution |

### Interpolation

| Algorithm ID | Purpose |
|-------------|---------|
| `native:tininterpolation` | TIN interpolation |
| `native:idwinterpolation` | IDW interpolation |
| `native:heatmapkerneldensityestimation` | Heatmap (KDE) |
| `native:linedensity` | Line density |

### Network Analysis

| Algorithm ID | Purpose |
|-------------|---------|
| `native:shortestpathpointtopoint` | Shortest path point to point |
| `native:shortestpathpointtolayer` | Shortest path point to layer |
| `native:shortestpathlayertopoint` | Shortest path layer to point |
| `native:serviceareafrompoint` | Service area from point |
| `native:serviceareafromlayer` | Service area from layer |

### Database

| Algorithm ID | Purpose |
|-------------|---------|
| `native:importintopostgis` | Export to PostgreSQL |
| `native:importintospatialite` | Export to SpatiaLite |
| `native:postgisexecutesql` | PostgreSQL execute SQL |
| `native:postgisexecuteandloadsql` | PostgreSQL execute and load SQL |
| `native:spatialiteexecutesql` | SpatiaLite execute SQL |
| `native:spatialiteexecutesqlregistered` | SpatiaLite execute SQL (registered) |
| `native:packagelayers` | Package layers (GeoPackage) |

### Cartography / Layouts

| Algorithm ID | Purpose |
|-------------|---------|
| `native:printlayouttopdf` | Export print layout to PDF |
| `native:printlayouttoimage` | Export print layout to image |
| `native:atlaslayouttoimage` | Export atlas to images |
| `native:atlaslayouttopdf` | Export atlas to PDF (multiple) |
| `native:atlaslayouttopdfsinglepdf` [VERIFY] | Export atlas to single PDF |
| `native:setlayerstyle` | Set layer style |
| `native:topologicalcoloring` | Topological coloring |
| `native:extractlabels` | Extract labels to layer |
| `native:printlayoutmapextenttolayer` | Layout map extent to layer |
| `native:combinestyledatabases` | Combine style databases |
| `native:categorizedrendererfrombystyles` [VERIFY] | Create categorized renderer |

### File Tools

| Algorithm ID | Purpose |
|-------------|---------|
| `native:filedownloader` | Download file via HTTP(S) |
| `native:httprequest` [VERIFY] | HTTP(S) POST/GET request |
| `native:openurl` [VERIFY] | Open file or URL |

### Mesh

| Algorithm ID | Purpose |
|-------------|---------|
| `native:exportmeshcontours` | Export contours from mesh |
| `native:exportmeshcrosssection` | Export cross section values |
| `native:exportmeshedges` | Export mesh edges |
| `native:exportmeshfaces` | Export mesh faces |
| `native:exportmeshongrid` | Export mesh on grid |
| `native:exportmeshvertices` | Export mesh vertices |
| `native:exporttimeseries` | Export time series from mesh |
| `native:rasterizemeshtimeseries` [VERIFY] | Rasterize mesh dataset |
| `native:tinmeshcreation` | TIN mesh creation |

### Point Cloud (PDAL-based)

| Algorithm ID | Purpose |
|-------------|---------|
| `native:pointcloudconvertformat` [VERIFY] | Convert point cloud format |
| `native:pointcloudexporttoraster` [VERIFY] | Export to raster |
| `native:pointcloudexporttovector` [VERIFY] | Export to vector |
| `native:pointcloudclip` [VERIFY] | Clip point cloud |
| `native:pointcloudmerge` [VERIFY] | Merge point clouds |
| `native:pointcloudreproject` [VERIFY] | Reproject point cloud |
| `native:pointcloudthin` [VERIFY] | Thin point cloud |
| `native:pointcloudtile` [VERIFY] | Tile point cloud |
| `native:pointcloudboundary` [VERIFY] | Point cloud boundary |
| `native:pointclouddensity` [VERIFY] | Point cloud density |
| `native:createcopc` [VERIFY] | Create COPC |
| `native:virtualpointcloud` [VERIFY] | Build virtual point cloud |

### Vector Tiles

| Algorithm ID | Purpose |
|-------------|---------|
| `native:downloadvectortiles` [VERIFY] | Download vector tiles |
| `native:writevectortilesmbtiles` [VERIFY] | Write vector tiles (MBTiles) |
| `native:writevectortilesxyz` [VERIFY] | Write vector tiles (XYZ) |

---

## 4. Parameter Types

### Input Parameter Types

| Class | Purpose | Typical Use |
|-------|---------|-------------|
| `QgsProcessingParameterFeatureSource` | Input vector layer | Main vector input, supports geometry type filtering |
| `QgsProcessingParameterRasterLayer` | Input raster layer | Single raster input |
| `QgsProcessingParameterMeshLayer` | Input mesh layer | Mesh data input |
| `QgsProcessingParameterMultipleLayers` | Multiple input layers | Layer merging, multi-layer analysis |
| `QgsProcessingParameterNumber` | Numeric value | Distances, counts, thresholds |
| `QgsProcessingParameterDistance` | Distance value | CRS-aware distance (adapts units) |
| `QgsProcessingParameterString` | Text string | Names, SQL queries, expressions |
| `QgsProcessingParameterBoolean` | True/False | Toggle options |
| `QgsProcessingParameterEnum` | Dropdown selection | Method choices, mode selection |
| `QgsProcessingParameterField` | Attribute field name | Field selection from parent layer |
| `QgsProcessingParameterExpression` | QGIS expression | Dynamic value computation |
| `QgsProcessingParameterCrs` | Coordinate reference system | CRS selection |
| `QgsProcessingParameterExtent` | Geographic extent | Bounding box for processing |
| `QgsProcessingParameterPoint` | Geographic point | Single coordinate |
| `QgsProcessingParameterRange` | Numeric range | Min/max pairs |
| `QgsProcessingParameterBand` | Raster band | Band selection |
| `QgsProcessingParameterFile` | File path (input) | Non-spatial file inputs |
| `QgsProcessingParameterMatrix` | Table of values | Lookup tables, kernels |
| `QgsProcessingParameterColor` | Color value | Styling parameters |
| `QgsProcessingParameterScale` | Map scale | Scale-dependent parameters |
| `QgsProcessingParameterMapLayer` | Any map layer type | Generic layer input |
| `QgsProcessingParameterLayout` | Print layout | Layout selection |
| `QgsProcessingParameterLayoutItem` | Layout item | Map/legend/etc. in layout |

### Output Parameter Types

| Class | Purpose |
|-------|---------|
| `QgsProcessingParameterFeatureSink` | Output vector layer |
| `QgsProcessingParameterRasterDestination` | Output raster layer |
| `QgsProcessingParameterVectorDestination` | Output vector (file-based) |
| `QgsProcessingParameterFileDestination` | Output file (non-spatial) |
| `QgsProcessingParameterFolderDestination` | Output directory |
| `QgsProcessingOutputNumber` | Numeric output value |
| `QgsProcessingOutputString` | String output value |
| `QgsProcessingOutputBoolean` | Boolean output value |
| `QgsProcessingOutputMultipleLayers` | Multiple output layers |

### Parameter Configuration

Parameters support several configuration options:

```python
def initAlgorithm(self, config=None):
    # Required input with geometry type filter
    self.addParameter(
        QgsProcessingParameterFeatureSource(
            'INPUT',
            'Input layer',
            [QgsProcessing.TypeVectorPolygon]  # Only polygons
        )
    )

    # Optional number with default and range
    self.addParameter(
        QgsProcessingParameterNumber(
            'DISTANCE',
            'Buffer distance',
            type=QgsProcessingParameterNumber.Double,
            defaultValue=10.0,
            minValue=0.0,
            optional=True
        )
    )

    # Enum parameter
    self.addParameter(
        QgsProcessingParameterEnum(
            'METHOD',
            'Method',
            options=['Round', 'Flat', 'Square'],
            defaultValue=0
        )
    )

    # Field parameter linked to input
    self.addParameter(
        QgsProcessingParameterField(
            'FIELD',
            'Group field',
            parentLayerParameterName='INPUT',
            type=QgsProcessingParameterField.Any,
            optional=True
        )
    )

    # CRS parameter
    self.addParameter(
        QgsProcessingParameterCrs(
            'TARGET_CRS',
            'Target CRS',
            defaultValue='EPSG:4326'
        )
    )

    # Output
    self.addParameter(
        QgsProcessingParameterFeatureSink(
            'OUTPUT',
            'Output layer'
        )
    )
```

### Reading Parameters in processAlgorithm()

```python
def processAlgorithm(self, parameters, context, feedback):
    source = self.parameterAsSource(parameters, 'INPUT', context)
    distance = self.parameterAsDouble(parameters, 'DISTANCE', context)
    method = self.parameterAsEnum(parameters, 'METHOD', context)
    field_name = self.parameterAsString(parameters, 'FIELD', context)
    crs = self.parameterAsCrs(parameters, 'TARGET_CRS', context)
    extent = self.parameterAsExtent(parameters, 'EXTENT', context)
    expression = self.parameterAsExpression(parameters, 'EXPRESSION', context)
    layers = self.parameterAsLayerList(parameters, 'LAYERS', context)
    raster = self.parameterAsRasterLayer(parameters, 'RASTER', context)
    boolean_val = self.parameterAsBool(parameters, 'DISSOLVE', context)
    integer_val = self.parameterAsInt(parameters, 'SEGMENTS', context)

    # Create output sink
    (sink, dest_id) = self.parameterAsSink(
        parameters, 'OUTPUT', context,
        source.fields(),
        source.wkbType(),
        source.sourceCrs()
    )

    # ... processing logic ...

    return {'OUTPUT': dest_id}
```

---

## 5. Custom Processing Algorithm Development

### Complete QgsProcessingAlgorithm Subclass

```python
from qgis.PyQt.QtCore import QCoreApplication
from qgis.core import (
    QgsProcessing,
    QgsProcessingAlgorithm,
    QgsProcessingParameterFeatureSource,
    QgsProcessingParameterFeatureSink,
    QgsProcessingParameterNumber,
    QgsProcessingParameterEnum,
    QgsProcessingException,
    QgsFeatureSink,
    QgsFeatureRequest,
    QgsWkbTypes,
)


class BufferByFieldAlgorithm(QgsProcessingAlgorithm):
    """Custom algorithm that buffers features using a field value as distance."""

    INPUT = 'INPUT'
    DISTANCE_FIELD = 'DISTANCE_FIELD'
    SEGMENTS = 'SEGMENTS'
    OUTPUT = 'OUTPUT'

    def name(self):
        """Unique algorithm identifier (lowercase, no spaces)."""
        return 'bufferbyfieldvalue'

    def displayName(self):
        """User-visible name."""
        return self.tr('Buffer by field value')

    def group(self):
        """Algorithm group name."""
        return self.tr('Custom vector tools')

    def groupId(self):
        """Algorithm group identifier."""
        return 'customvectortools'

    def shortHelpString(self):
        """Help text shown in the algorithm dialog."""
        return self.tr(
            'Creates buffers around features using a numeric field '
            'value as the buffer distance for each feature.'
        )

    def tags(self):
        """Search tags for algorithm discovery."""
        return ['buffer', 'variable', 'field', 'distance']

    def initAlgorithm(self, config=None):
        """Define the algorithm parameters."""
        self.addParameter(
            QgsProcessingParameterFeatureSource(
                self.INPUT,
                self.tr('Input layer'),
                [QgsProcessing.TypeVectorAnyGeometry]
            )
        )
        self.addParameter(
            QgsProcessingParameterField(
                self.DISTANCE_FIELD,
                self.tr('Distance field'),
                parentLayerParameterName=self.INPUT,
                type=QgsProcessingParameterField.Numeric
            )
        )
        self.addParameter(
            QgsProcessingParameterNumber(
                self.SEGMENTS,
                self.tr('Segments'),
                type=QgsProcessingParameterNumber.Integer,
                defaultValue=5,
                minValue=1
            )
        )
        self.addParameter(
            QgsProcessingParameterFeatureSink(
                self.OUTPUT,
                self.tr('Buffered')
            )
        )

    def processAlgorithm(self, parameters, context, feedback):
        """Execute the algorithm."""
        source = self.parameterAsSource(parameters, self.INPUT, context)
        if source is None:
            raise QgsProcessingException(
                self.invalidSourceError(parameters, self.INPUT)
            )

        field_name = self.parameterAsString(
            parameters, self.DISTANCE_FIELD, context
        )
        segments = self.parameterAsInt(parameters, self.SEGMENTS, context)

        (sink, dest_id) = self.parameterAsSink(
            parameters, self.OUTPUT, context,
            source.fields(),
            QgsWkbTypes.Polygon,
            source.sourceCrs()
        )
        if sink is None:
            raise QgsProcessingException(
                self.invalidSinkError(parameters, self.OUTPUT)
            )

        total = 100.0 / source.featureCount() if source.featureCount() else 0
        features = source.getFeatures()

        for current, feature in enumerate(features):
            # ALWAYS check for cancellation in loops
            if feedback.isCanceled():
                break

            distance = feature[field_name]
            if distance is None or distance == 0:
                feedback.pushInfo(
                    f'Skipping feature {feature.id()}: no valid distance'
                )
                continue

            buffered_geom = feature.geometry().buffer(distance, segments)
            feature.setGeometry(buffered_geom)
            sink.addFeature(feature, QgsFeatureSink.FastInsert)

            feedback.setProgress(int(current * total))

        return {self.OUTPUT: dest_id}

    def createInstance(self):
        """MUST return a new instance of the algorithm."""
        return BufferByFieldAlgorithm()

    def tr(self, string):
        """Translation wrapper."""
        return QCoreApplication.translate('Processing', string)
```

### Required Methods Summary

| Method | Purpose | Required |
|--------|---------|----------|
| `name()` | Unique ID string (lowercase, no spaces) | YES |
| `displayName()` | User-visible name | YES |
| `group()` | Category display name | NO (but recommended) |
| `groupId()` | Category ID | NO (but recommended) |
| `shortHelpString()` | Help text | NO (but recommended) |
| `tags()` | Search keywords | NO |
| `icon()` | Custom icon | NO |
| `initAlgorithm(config)` | Define parameters | YES |
| `processAlgorithm(parameters, context, feedback)` | Core logic | YES |
| `createInstance()` | Return new instance | YES |
| `flags()` | Algorithm flags (e.g., `FlagNoThreading`) | NO |
| `canExecute()` | Check if algorithm can run | NO |
| `checkParameterValues()` | Custom parameter validation | NO |
| `prepareAlgorithm()` | Pre-execution setup (main thread) | NO |
| `postProcessAlgorithm()` | Post-execution cleanup (main thread) | NO |

### Threading Flags

By default, algorithms run in a background thread. If your algorithm uses GUI elements or non-thread-safe APIs:

```python
def flags(self):
    return super().flags() | QgsProcessingAlgorithm.FlagNoThreading
```

### Custom Processing Provider

```python
from qgis.core import QgsProcessingProvider
from qgis.PyQt.QtGui import QIcon

from .buffer_by_field import BufferByFieldAlgorithm
from .other_algorithm import OtherAlgorithm


class MyPluginProvider(QgsProcessingProvider):

    def loadAlgorithms(self):
        """Register all algorithms."""
        self.addAlgorithm(BufferByFieldAlgorithm())
        self.addAlgorithm(OtherAlgorithm())

    def id(self):
        """Unique provider ID (used as prefix: 'myplugin:algorithmname')."""
        return 'myplugin'

    def name(self):
        """User-visible name in Processing Toolbox."""
        return 'My Plugin Tools'

    def longName(self):
        """Extended name shown in algorithm details."""
        return 'My Plugin GIS Tools v1.0'

    def icon(self):
        """Provider icon."""
        return QgsProcessingProvider.icon(self)
```

### Registering Provider in Plugin

```python
from qgis.core import QgsApplication
from .processing_provider.provider import MyPluginProvider


class MyPlugin:
    def __init__(self, iface):
        self.iface = iface
        self.provider = None

    def initProcessing(self):
        self.provider = MyPluginProvider()
        QgsApplication.processingRegistry().addProvider(self.provider)

    def initGui(self):
        self.initProcessing()

    def unload(self):
        QgsApplication.processingRegistry().removeProvider(self.provider)
```

Plugin `metadata.txt` MUST include:

```ini
hasProcessingProvider=yes
```

### Script Algorithms (Simpler Alternative)

Script algorithms provide a lighter alternative using the `@alg` decorator:

```python
from qgis.processing import alg

@alg(name='my_buffer_script',
     label='My Buffer Script',
     group='My Scripts',
     group_label='My Script Group')
@alg.input(type=alg.SOURCE, name='INPUT', label='Input layer')
@alg.input(type=alg.DISTANCE, name='DISTANCE', label='Buffer distance',
           default=10.0)
@alg.input(type=alg.SINK, name='OUTPUT', label='Output layer')
def my_buffer(instance, parameters, context, feedback, inputs):
    """Buffer features by a fixed distance."""
    source = instance.parameterAsSource(parameters, 'INPUT', context)
    distance = instance.parameterAsDouble(parameters, 'DISTANCE', context)

    (sink, dest_id) = instance.parameterAsSink(
        parameters, 'OUTPUT', context,
        source.fields(), source.wkbType(), source.sourceCrs()
    )

    for feature in source.getFeatures():
        if feedback.isCanceled():
            break
        feature.setGeometry(feature.geometry().buffer(distance, 5))
        sink.addFeature(feature)

    return {'OUTPUT': dest_id}
```

**Important limitation**: Script algorithms created with `@alg` are ALWAYS added to the user's Processing Scripts provider. They CANNOT be added to a custom provider or used in plugins. For plugin algorithms, ALWAYS use the full `QgsProcessingAlgorithm` subclass approach.

Script files are saved as `.py` files in the Processing scripts folder (configurable in Processing settings). They appear automatically in the Processing Toolbox under "Scripts."

---

## 6. Algorithm Chaining and Workflows

### Sequential Algorithm Execution

Chain algorithms by passing the output of one as input to the next:

```python
import processing

# Step 1: Reproject
reprojected = processing.run("native:reprojectlayer", {
    'INPUT': input_layer,
    'TARGET_CRS': 'EPSG:28992',
    'OUTPUT': 'memory:'
})

# Step 2: Buffer the reprojected layer
buffered = processing.run("native:buffer", {
    'INPUT': reprojected['OUTPUT'],
    'DISTANCE': 100,
    'OUTPUT': 'memory:'
})

# Step 3: Dissolve the buffers
dissolved = processing.run("native:dissolve", {
    'INPUT': buffered['OUTPUT'],
    'OUTPUT': '/output/final_result.gpkg'
})
```

### Passing Outputs as Inputs

The result dictionary from `processing.run()` contains keys matching the output parameter names. The `OUTPUT` key typically holds either:
- A `QgsMapLayer` object (for memory layers)
- A file path string (for file-based outputs)

Both can be directly passed as input to the next algorithm.

### Processing Models (Graphical Modeler)

The Model Designer creates visual workflows chaining multiple algorithms. Models are saved as `.model3` files and can be:

- Run from the Processing Toolbox
- Executed from Python: `processing.run("model:my_model_name", params)`
- Exported as Python scripts for further customization
- Embedded in project files
- Nested inside other models

Model inputs connect to algorithm parameters through dropdown menus offering:
- **Static values**: Fixed values
- **Expressions**: Computed values
- **Model inputs**: User-provided parameters
- **Algorithm outputs**: Results from previous steps

Dependencies between algorithms are inferred from output-to-input connections. Explicit dependencies can be set for algorithms that must run in sequence but do not share data.

### Batch Processing

Batch processing runs the same algorithm multiple times with different parameters:

```python
# Manual batch via loop
import os
import processing

input_files = [f for f in os.listdir('/data/') if f.endswith('.gpkg')]

for input_file in input_files:
    input_path = os.path.join('/data/', input_file)
    output_path = os.path.join('/output/', f'buffered_{input_file}')

    processing.run("native:buffer", {
        'INPUT': input_path,
        'DISTANCE': 50,
        'OUTPUT': output_path
    })
```

The GUI batch interface supports:
- Adding rows manually or from file patterns
- Auto-filling parameter columns
- Saving/loading batch configurations as JSON
- Expression-based parameter generation (e.g., `generate_series(100, 1000, 50)`)

### Iterating Over Features

Use the iterator feature in the GUI (click the iterate button next to an input), or in Python:

```python
import processing
from qgis.core import QgsFeatureRequest

layer = iface.activeLayer()

for feature in layer.getFeatures():
    # Create a memory layer for each feature
    temp = processing.run("native:extractbyattribute", {
        'INPUT': layer,
        'FIELD': 'id',
        'OPERATOR': 0,  # equals
        'VALUE': str(feature['id']),
        'OUTPUT': 'memory:'
    })

    # Process individually
    processing.run("native:buffer", {
        'INPUT': temp['OUTPUT'],
        'DISTANCE': feature['buffer_dist'],
        'OUTPUT': f'/output/buffer_{feature["id"]}.gpkg'
    })
```

---

## 7. GDAL Provider Algorithms

The GDAL provider exposes GDAL/OGR tools as processing algorithms. These are particularly powerful for raster operations and format conversions.

### Raster Projections

| Algorithm ID | Purpose |
|-------------|---------|
| `gdal:warpreproject` | Raster reprojection and warping (gdalwarp) |
| `gdal:assignprojection` | Assign CRS without reprojecting |
| `gdal:extractprojection` | Extract CRS information |

```python
# Reproject raster
result = processing.run("gdal:warpreproject", {
    'INPUT': '/data/dem.tif',
    'SOURCE_CRS': 'EPSG:4326',
    'TARGET_CRS': 'EPSG:28992',
    'RESAMPLING': 0,  # 0=Nearest, 1=Bilinear, 2=Cubic, etc.
    'TARGET_RESOLUTION': 25,
    'OUTPUT': '/output/dem_rd.tif'
})
```

### Raster Conversion

| Algorithm ID | Purpose |
|-------------|---------|
| `gdal:translate` | Raster format conversion, subset extraction, rescaling |
| `gdal:gdal2xyz` | Raster to XYZ text |
| `gdal:pcttorgb` | Palette to RGB |
| `gdal:rgbtopct` | RGB to palette |
| `gdal:rearrangebands` | Rearrange raster bands |

```python
# Convert raster format and subset
result = processing.run("gdal:translate", {
    'INPUT': '/data/dem.tif',
    'TARGET_CRS': '',
    'NODATA': -9999,
    'COPY_SUBDATASETS': False,
    'OPTIONS': 'COMPRESS=DEFLATE',
    'OUTPUT': '/output/dem_compressed.tif'
})
```

### Raster Miscellaneous

| Algorithm ID | Purpose |
|-------------|---------|
| `gdal:buildvirtualraster` | Create VRT (virtual raster) from multiple inputs |
| `gdal:merge` | Merge multiple rasters into one |
| `gdal:buildoverviews` | Build raster overviews (pyramids) |
| `gdal:rastercalculator` | GDAL raster calculator |
| `gdal:rasterinformation` | Get raster metadata (gdalinfo) |
| `gdal:tileindex` | Create raster tile index |
| `gdal:gdal2tiles` | Generate tiles (TMS/XYZ) |
| `gdal:retile` | Retile raster dataset |
| `gdal:pansharpening` | Pan-sharpening |
| `gdal:viewshed` | Viewshed analysis |

```python
# Build virtual raster
result = processing.run("gdal:buildvirtualraster", {
    'INPUT': ['/data/tile1.tif', '/data/tile2.tif', '/data/tile3.tif'],
    'RESOLUTION': 0,  # 0=Average, 1=Highest, 2=Lowest
    'OUTPUT': '/output/mosaic.vrt'
})

# Merge rasters
result = processing.run("gdal:merge", {
    'INPUT': ['/data/tile1.tif', '/data/tile2.tif'],
    'PCT': False,
    'SEPARATE': False,
    'NODATA_INPUT': -9999,
    'NODATA_OUTPUT': -9999,
    'OUTPUT': '/output/merged.tif'
})
```

### Terrain Analysis (GDAL)

| Algorithm ID | Purpose |
|-------------|---------|
| `gdal:hillshade` | Hillshade from DEM |
| `gdal:slope` | Slope from DEM |
| `gdal:aspect` | Aspect from DEM |
| `gdal:roughness` | Terrain roughness |
| `gdal:tri` | Terrain ruggedness index |
| `gdal:tpi` | Topographic position index |
| `gdal:colorrelief` | Color relief rendering |

```python
# Generate hillshade
result = processing.run("gdal:hillshade", {
    'INPUT': '/data/dem.tif',
    'BAND': 1,
    'Z_FACTOR': 1,
    'SCALE': 1,
    'AZIMUTH': 315,
    'ALTITUDE': 45,
    'OUTPUT': '/output/hillshade.tif'
})

# Calculate slope
result = processing.run("gdal:slope", {
    'INPUT': '/data/dem.tif',
    'BAND': 1,
    'SCALE': 1,
    'AS_PERCENT': False,
    'OUTPUT': '/output/slope.tif'
})
```

### Raster Extraction

| Algorithm ID | Purpose |
|-------------|---------|
| `gdal:contour` | Generate contour lines from DEM |
| `gdal:contourpolygons` | Generate contour polygons from DEM |
| `gdal:cliprasterbyextent` | Clip raster by bounding box |
| `gdal:cliprasterbymask` | Clip raster by vector mask |

```python
# Generate contour lines
result = processing.run("gdal:contour", {
    'INPUT': '/data/dem.tif',
    'BAND': 1,
    'INTERVAL': 10,
    'FIELD_NAME': 'ELEV',
    'OUTPUT': '/output/contours.gpkg'
})
```

### Raster Analysis (GDAL)

| Algorithm ID | Purpose |
|-------------|---------|
| `gdal:fillnodata` | Fill NoData gaps |
| `gdal:grididw` | IDW grid interpolation |
| `gdal:gridlinear` | Linear grid interpolation |
| `gdal:gridmovingaverage` | Moving average grid |
| `gdal:gridnearestneighbor` | Nearest neighbor grid |
| `gdal:griddatametrics` | Grid data metrics |
| `gdal:nearblack` | Convert near-black borders to black |
| `gdal:proximity` | Proximity (distance) raster |
| `gdal:sieve` | Remove small raster polygons |

### Vector-Raster Conversion (GDAL)

| Algorithm ID | Purpose |
|-------------|---------|
| `gdal:polygonize` | Raster to vector polygons |
| `gdal:rasterize` | Vector to raster (burn values) |

```python
# Raster to vector
result = processing.run("gdal:polygonize", {
    'INPUT': '/data/classified.tif',
    'BAND': 1,
    'FIELD': 'DN',
    'EIGHT_CONNECTEDNESS': False,
    'OUTPUT': '/output/polygons.gpkg'
})

# Vector to raster
result = processing.run("gdal:rasterize", {
    'INPUT': '/data/buildings.gpkg',
    'FIELD': 'height',
    'UNITS': 1,  # 1=Georeferenced units
    'WIDTH': 1,  # pixel width in georef units
    'HEIGHT': 1, # pixel height in georef units
    'EXTENT': layer.extent(),
    'OUTPUT': '/output/buildings_raster.tif'
})
```

### GDAL Vector Geoprocessing

| Algorithm ID | Purpose |
|-------------|---------|
| `gdal:buffervectors` | Buffer vectors (ogr2ogr) |
| `gdal:clipvectorbyextent` | Clip by extent |
| `gdal:clipvectorbymask` | Clip by mask layer |
| `gdal:dissolve` | Dissolve features |
| `gdal:offsetcurve` | Offset curves |
| `gdal:onesidebuffer` | One-sided buffer |
| `gdal:pointsalonglines` | Points along lines |

### GDAL Vector Miscellaneous

| Algorithm ID | Purpose |
|-------------|---------|
| `gdal:executesql` | Execute SQL on vector data |
| `gdal:convertformat` | Convert vector format |
| `gdal:buildvirtualvector` | Build virtual vector (VRT) |
| `gdal:vectorinformation` | Vector metadata (ogrinfo) |
| `gdal:exportpostgresql` [VERIFY] | Export to PostgreSQL |

---

## 8. Anti-patterns and Critical Warnings

### Algorithm Availability

**NEVER** use hardcoded algorithm IDs without checking provider availability. Algorithms may be absent if a provider is not installed or configured.

```python
# WRONG - will crash if algorithm not available
result = processing.run("grass:v.buffer", params)

# CORRECT - check first
registry = QgsApplication.processingRegistry()
alg = registry.algorithmById('grass:v.buffer')
if alg is None:
    feedback.reportError('GRASS provider not available. Install GRASS GIS.')
    return {}
result = processing.run('grass:v.buffer', params)
```

### Exception Handling

**NEVER** ignore `processing.run()` exceptions. ALWAYS wrap in try/except:

```python
# WRONG
result = processing.run("native:buffer", params)
layer = result['OUTPUT']  # Will crash if processing.run() failed

# CORRECT
try:
    result = processing.run("native:buffer", params)
    layer = result['OUTPUT']
except QgsProcessingException as e:
    feedback.reportError(f'Buffer operation failed: {e}')
    return {}
```

### Feedback and Progress

**ALWAYS** use `QgsProcessingFeedback` for progress reporting in custom algorithms:

```python
# WRONG - no progress, no cancellation check
def processAlgorithm(self, parameters, context, feedback):
    for feature in source.getFeatures():
        # process feature...
        pass

# CORRECT - progress reporting and cancellation
def processAlgorithm(self, parameters, context, feedback):
    total = 100.0 / source.featureCount() if source.featureCount() else 0
    for current, feature in enumerate(source.getFeatures()):
        if feedback.isCanceled():
            break
        # process feature...
        feedback.setProgress(int(current * total))
```

### Temporary Output Paths

**ALWAYS** handle temporary output paths correctly. Use `'memory:'` or `QgsProcessing.TEMPORARY_OUTPUT` for intermediate results:

```python
# WRONG - hardcoded temp path that may not exist
result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 10,
    'OUTPUT': '/tmp/buffer_result.gpkg'  # May fail on Windows
})

# CORRECT - let QGIS handle temporary storage
result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 10,
    'OUTPUT': 'memory:'
})
```

### Cancellation in Custom Algorithms

**ALWAYS** check for cancelled operations in custom algorithms, especially in loops:

```python
def processAlgorithm(self, parameters, context, feedback):
    features = source.getFeatures()
    for current, feature in enumerate(features):
        # ALWAYS check at the top of every loop iteration
        if feedback.isCanceled():
            break
        # ... processing ...
```

### Algorithm Existence Verification

**NEVER** assume an algorithm exists. ALWAYS verify using the processing registry:

```python
from qgis.core import QgsApplication

def safe_run_algorithm(algorithm_id, params, context=None, feedback=None):
    """Run an algorithm after verifying it exists."""
    registry = QgsApplication.processingRegistry()
    if registry.algorithmById(algorithm_id) is None:
        raise QgsProcessingException(
            f'Algorithm "{algorithm_id}" not found. '
            f'Check that the required provider is installed and enabled.'
        )
    return processing.run(algorithm_id, params,
                         context=context, feedback=feedback)
```

### GUI Elements in Algorithms

**NEVER** show message boxes or use any GUI elements from within `processAlgorithm()`. Use the feedback object instead:

```python
# WRONG - will crash in background thread
def processAlgorithm(self, parameters, context, feedback):
    from qgis.PyQt.QtWidgets import QMessageBox
    QMessageBox.warning(None, 'Warning', 'Something happened')

# CORRECT - use feedback for communication
def processAlgorithm(self, parameters, context, feedback):
    feedback.pushWarning('Something happened')
    feedback.pushInfo('Processing step completed')
    feedback.reportError('Critical error occurred')
```

### Loading Output Layers

**NEVER** manually load resulting layers in `processAlgorithm()`. Let the Processing framework handle results:

```python
# WRONG - manual layer loading in algorithm
def processAlgorithm(self, parameters, context, feedback):
    # ... processing ...
    result_layer = QgsVectorLayer(output_path, 'result', 'ogr')
    QgsProject.instance().addMapLayer(result_layer)

# CORRECT - return output and let framework handle it
def processAlgorithm(self, parameters, context, feedback):
    # ... processing ...
    return {self.OUTPUT: dest_id}
```

### CRS Mismatches

**ALWAYS** be aware that processing algorithms execute in the input layer's CRS. When combining layers with different CRSs:

```python
# CORRECT - reproject first, then combine
reprojected = processing.run("native:reprojectlayer", {
    'INPUT': layer_epsg4326,
    'TARGET_CRS': layer_epsg28992.crs(),
    'OUTPUT': 'memory:'
})

intersection = processing.run("native:intersection", {
    'INPUT': reprojected['OUTPUT'],
    'OVERLAY': layer_epsg28992,
    'OUTPUT': 'memory:'
})
```

### Output Declaration

**ALWAYS** declare all outputs your algorithm creates. The Processing framework uses these declarations to manage results:

```python
# CORRECT - declare all outputs
def initAlgorithm(self, config=None):
    self.addParameter(QgsProcessingParameterFeatureSink('OUTPUT', 'Output'))
    self.addOutput(QgsProcessingOutputNumber('COUNT', 'Feature count'))

def processAlgorithm(self, parameters, context, feedback):
    # ... processing ...
    return {
        'OUTPUT': dest_id,
        'COUNT': feature_count
    }
```

### Thread Safety

If your algorithm uses non-thread-safe operations, ALWAYS set the `FlagNoThreading` flag:

```python
def flags(self):
    return super().flags() | QgsProcessingAlgorithm.FlagNoThreading
```

Common non-thread-safe operations include direct GUI manipulation, accessing `iface`, and some third-party library calls.

---

## Sources

1. QGIS PyQGIS Developer Cookbook — Processing: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/processing.html
2. QGIS User Manual — Processing Framework: https://docs.qgis.org/latest/en/docs/user_manual/processing/index.html
3. QGIS User Manual — Processing Toolbox: https://docs.qgis.org/latest/en/docs/user_manual/processing/toolbox.html
4. QGIS User Manual — Processing Algorithms Reference: https://docs.qgis.org/latest/en/docs/user_manual/processing_algs/index.html
5. QGIS User Manual — Native QGIS Algorithms: https://docs.qgis.org/latest/en/docs/user_manual/processing_algs/qgis/index.html
6. QGIS User Manual — GDAL Algorithms: https://docs.qgis.org/latest/en/docs/user_manual/processing_algs/gdal/index.html
7. QGIS User Manual — Processing Scripts: https://docs.qgis.org/latest/en/docs/user_manual/processing/scripts.html
8. QGIS User Manual — Batch Processing: https://docs.qgis.org/latest/en/docs/user_manual/processing/batch.html
9. QGIS User Manual — Model Designer: https://docs.qgis.org/latest/en/docs/user_manual/processing/modeler.html
10. QGIS User Manual — Processing History: https://docs.qgis.org/latest/en/docs/user_manual/processing/history.html
