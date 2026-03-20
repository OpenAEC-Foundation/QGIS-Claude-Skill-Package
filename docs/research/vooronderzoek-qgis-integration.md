# Vooronderzoek: QGIS Integration Patterns

> Deep research document for the QGIS Claude Skill Package.
> Covers 8 implementation skills: vector-analysis, raster-analysis, print-layouts, postgis, web-services, 3d-visualization, georeferencing, network-analysis.
> Sources: Official PyQGIS Developer Cookbook (QGIS 3.x), QGIS C++ API Reference.

---

## 1. Vector Analysis Workflows

### Spatial Query Patterns

QGIS provides two primary spatial query mechanisms: `QgsFeatureRequest` for filtering at the provider level, and `QgsSpatialIndex` for in-memory spatial lookups.

**Feature Request with Spatial Filter:**

```python
from qgis.core import QgsFeatureRequest, QgsRectangle, QgsPointXY

# Filter by bounding rectangle
area = QgsRectangle(100.0, -1.0, 101.0, 0.0)
request = QgsFeatureRequest().setFilterRect(area)
request.setFlags(QgsFeatureRequest.ExactIntersect)  # Exact geometry test, not just bbox
request.setLimit(100)  # Limit result count

for feature in layer.getFeatures(request):
    print(feature.id(), feature.geometry().asWkt())
```

**Filter by Expression:**

```python
request = QgsFeatureRequest().setFilterExpression('"population" > 100000')
for feature in layer.getFeatures(request):
    print(feature['name'], feature['population'])
```

**Spatial Index (QgsSpatialIndex):**

```python
from qgis.core import QgsSpatialIndex, QgsPointXY, QgsRectangle

index = QgsSpatialIndex(layer.getFeatures())

# Nearest neighbor search — returns list of feature IDs
nearest_ids = index.nearestNeighbor(QgsPointXY(15.5, 47.1), 5)

# Intersection with bounding box — returns list of feature IDs
bbox = QgsRectangle(14.0, 46.0, 17.0, 49.0)
intersecting_ids = index.intersects(bbox)
```

**QgsSpatialIndexKDBush** is a faster alternative for point-only layers with radius search capability.

### Selection Operations

```python
# Select by expression
layer.selectByExpression('"type" = \'highway\'')

# Select all
layer.selectAll()

# Get selected features
for feature in layer.selectedFeatures():
    print(feature.id())

# Clear selection
layer.removeSelection()
```

### Buffer Analysis Workflow

Buffer analysis is performed via the Processing framework:

```python
import processing

result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 100,          # Buffer distance in layer units
    'SEGMENTS': 5,            # Segments for circular approximation
    'END_CAP_STYLE': 0,       # 0=Round, 1=Flat, 2=Square
    'JOIN_STYLE': 0,          # 0=Round, 1=Miter, 2=Bevel
    'MITER_LIMIT': 2,
    'DISSOLVE': False,
    'OUTPUT': 'memory:'       # In-memory output
})
buffered_layer = result['OUTPUT']
QgsProject.instance().addMapLayer(buffered_layer)
```

Alternatively, use geometry-level buffer:

```python
geom = feature.geometry()
buffered_geom = geom.buffer(100.0, 5)  # distance, segments
```

### Overlay Operations

All overlay operations use the Processing framework with `processing.run()`:

**Intersection:**
```python
result = processing.run("native:intersection", {
    'INPUT': layer_a,
    'OVERLAY': layer_b,
    'INPUT_FIELDS': [],       # Empty = all fields
    'OVERLAY_FIELDS': [],
    'OVERLAY_FIELDS_PREFIX': '',
    'OUTPUT': 'memory:'
})
```

**Union:**
```python
result = processing.run("native:union", {
    'INPUT': layer_a,
    'OVERLAY': layer_b,
    'OVERLAY_FIELDS_PREFIX': '',
    'OUTPUT': 'memory:'
})
```

**Difference:**
```python
result = processing.run("native:difference", {
    'INPUT': layer_a,
    'OVERLAY': layer_b,
    'OUTPUT': 'memory:'
})
```

**Clip:**
```python
result = processing.run("native:clip", {
    'INPUT': layer_a,
    'OVERLAY': clip_layer,
    'OUTPUT': 'memory:'
})
```

**Symmetric Difference:**
```python
result = processing.run("native:symmetricaldifference", {
    'INPUT': layer_a,
    'OVERLAY': layer_b,
    'OUTPUT': 'memory:'
})
```

### Attribute Joins and Spatial Joins

**Attribute Join (by field value):**
```python
result = processing.run("native:joinattributestable", {
    'INPUT': layer_a,
    'FIELD': 'id',
    'INPUT_2': layer_b,
    'FIELD_2': 'foreign_id',
    'FIELDS_TO_COPY': [],     # Empty = all fields
    'METHOD': 0,              # 0=one-to-many, 1=first match only
    'DISCARD_NONMATCHING': False,
    'PREFIX': '',
    'OUTPUT': 'memory:'
})
```

**Spatial Join (join attributes by location):**
```python
result = processing.run("native:joinattributesbylocation", {
    'INPUT': target_layer,
    'PREDICATE': [0],         # 0=intersects, 1=contains, 2=equals,
                              # 3=touches, 4=overlaps, 5=within, 6=crosses
    'JOIN': join_layer,
    'JOIN_FIELDS': [],
    'METHOD': 0,              # 0=one-to-many, 1=one-to-first, 2=one-to-largest overlap
    'DISCARD_NONMATCHING': False,
    'PREFIX': '',
    'OUTPUT': 'memory:'
})
```

### Field Calculations

```python
# Using the editing buffer
with edit(layer):
    for feature in layer.getFeatures():
        # Calculate area and set it on a field
        area = feature.geometry().area()
        field_idx = layer.fields().indexOf('area_m2')
        layer.changeAttributeValue(feature.id(), field_idx, area)
```

**Via Processing:**
```python
result = processing.run("native:fieldcalculator", {
    'INPUT': layer,
    'FIELD_NAME': 'area_m2',
    'FIELD_TYPE': 0,          # 0=Float, 1=Integer, 2=String, 3=Date
    'FIELD_LENGTH': 10,
    'FIELD_PRECISION': 3,
    'FORMULA': '$area',
    'OUTPUT': 'memory:'
})
```

### Feature Selection and Extraction

```python
# Extract by expression
result = processing.run("native:extractbyexpression", {
    'INPUT': layer,
    'EXPRESSION': '"population" > 50000',
    'OUTPUT': 'memory:'
})

# Extract by location
result = processing.run("native:extractbylocation", {
    'INPUT': layer,
    'PREDICATE': [0],         # 0=intersects
    'INTERSECT': reference_layer,
    'OUTPUT': 'memory:'
})
```

### QgsVectorFileWriter for Output

```python
from qgis.core import QgsVectorFileWriter, QgsCoordinateTransformContext

# Method 1: Write entire layer to file
save_options = QgsVectorFileWriter.SaveVectorOptions()
save_options.driverName = "GPKG"  # or "ESRI Shapefile", "GeoJSON", "FlatGeobuf"
save_options.fileEncoding = "UTF-8"

error = QgsVectorFileWriter.writeAsVectorFormatV3(
    layer,
    "/path/to/output.gpkg",
    QgsCoordinateTransformContext(),
    save_options
)

# Method 2: Write features individually
fields = layer.fields()
writer = QgsVectorFileWriter.create(
    "/path/to/output.gpkg",
    fields,
    QgsWkbTypes.Polygon,
    layer.crs(),
    QgsCoordinateTransformContext(),
    save_options
)
for feature in layer.getFeatures():
    writer.addFeature(feature)
del writer  # Flush and close
```

### Dissolve and Aggregate Patterns

```python
# Dissolve — merge features by attribute
result = processing.run("native:dissolve", {
    'INPUT': layer,
    'FIELD': ['province'],    # Dissolve field(s), empty = dissolve all
    'SEPARATE_DISJOINT': False,
    'OUTPUT': 'memory:'
})

# Aggregate — dissolve with statistics
result = processing.run("native:aggregate", {
    'INPUT': layer,
    'GROUP_BY': '"province"',
    'AGGREGATES': [
        {'aggregate': 'sum', 'delimiter': ',', 'input': '"population"',
         'length': 10, 'name': 'total_pop', 'precision': 0, 'type': 6}
    ],
    'OUTPUT': 'memory:'
})
```

---

## 2. Raster Analysis

### QgsRasterLayer: Loading, Renderers, Querying

**Loading a raster layer:**
```python
from qgis.core import QgsRasterLayer, QgsProject

rlayer = QgsRasterLayer("/path/to/dem.tif", "DEM")
if not rlayer.isValid():
    print("Layer failed to load!")

QgsProject.instance().addMapLayer(rlayer)
```

**Key properties:**
```python
rlayer.width()        # Pixel width
rlayer.height()       # Pixel height
rlayer.extent()       # QgsRectangle
rlayer.rasterType()   # 0=GrayOrUndefined, 1=Palette, 2=Multiband
rlayer.bandCount()    # Number of bands
rlayer.bandName(1)    # Name of band 1
```

### Raster Data Access

**sample() — query single pixel value:**
```python
provider = rlayer.dataProvider()
val, result = provider.sample(QgsPointXY(20.50, -34.0), 1)  # point, band
if result:
    print(f"Value at point: {val}")
```

**identify() — query all bands at a point:**
```python
ident = rlayer.dataProvider().identify(
    QgsPointXY(20.5, -34.0),
    QgsRaster.IdentifyFormatValue
)
if ident.isValid():
    print(ident.results())  # {1: 323.0, 2: 127.0, ...}
```

**QgsRasterBlock — read/write pixel blocks:**
```python
from qgis.core import QgsRasterBlock, Qgis

# Reading a block
provider = rlayer.dataProvider()
block = provider.block(1, rlayer.extent(), rlayer.width(), rlayer.height())
value = block.value(row, col)

# Writing a block
block = QgsRasterBlock(Qgis.Byte, 2, 2)
block.setData(b'\xaa\xbb\xcc\xdd')
provider.setEditable(True)
provider.writeBlock(block, 1, 0, 0)  # block, band, xOffset, yOffset
provider.setEditable(False)
```

### QgsRasterCalculator: Map Algebra Expressions

```python
from qgis.analysis import QgsRasterCalculator, QgsRasterCalculatorEntry

# Define input entries
entries = []
entry = QgsRasterCalculatorEntry()
entry.ref = 'dem@1'             # Reference name in expression
entry.raster = rlayer
entry.bandNumber = 1
entries.append(entry)

# Build calculator
calc = QgsRasterCalculator(
    '"dem@1" * 2 + 100',        # Expression
    '/path/to/output.tif',      # Output path
    'GTiff',                    # Format
    rlayer.extent(),            # Output extent
    rlayer.width(),             # Output width
    rlayer.height(),            # Output height
    entries                     # Input entries
)

result = calc.processCalculation()
# Returns 0 on success, non-zero on error
```

### Raster Renderers

All renderers inherit from `QgsRasterRenderer`.

**Single Band Gray:**
```python
from qgis.core import QgsSingleBandGrayRenderer, QgsContrastEnhancement

renderer = QgsSingleBandGrayRenderer(rlayer.dataProvider(), 1)
ce = QgsContrastEnhancement(rlayer.dataProvider().dataType(1))
ce.setContrastEnhancementAlgorithm(QgsContrastEnhancement.StretchToMinimumMaximum)
ce.setMinimumValue(0)
ce.setMaximumValue(255)
renderer.setContrastEnhancement(ce)
rlayer.setRenderer(renderer)
rlayer.triggerRepaint()
```

**Single Band Pseudocolor:**
```python
from qgis.core import (QgsSingleBandPseudoColorRenderer,
                        QgsRasterShader, QgsColorRampShader)
from qgis.PyQt.QtGui import QColor

fcn = QgsColorRampShader()
fcn.setColorRampType(QgsColorRampShader.Interpolated)  # or Discrete, Exact
lst = [
    QgsColorRampShader.ColorRampItem(0, QColor(0, 255, 0), "Low"),
    QgsColorRampShader.ColorRampItem(128, QColor(255, 255, 0), "Medium"),
    QgsColorRampShader.ColorRampItem(255, QColor(255, 0, 0), "High")
]
fcn.setColorRampItemList(lst)

shader = QgsRasterShader()
shader.setRasterShaderFunction(fcn)

renderer = QgsSingleBandPseudoColorRenderer(rlayer.dataProvider(), 1, shader)
rlayer.setRenderer(renderer)
rlayer.triggerRepaint()
```

**Multiband Color:**
```python
from qgis.core import QgsMultiBandColorRenderer

renderer = QgsMultiBandColorRenderer(rlayer.dataProvider(), 3, 2, 1)
# Parameters: provider, redBand, greenBand, blueBand
rlayer.setRenderer(renderer)

# Reassign bands after creation
renderer.setRedBand(4)
renderer.setGreenBand(3)
renderer.setBlueBand(2)
rlayer.triggerRepaint()
```

**Paletted Renderer:**
```python
from qgis.core import QgsPalettedRasterRenderer

classes = [
    QgsPalettedRasterRenderer.Class(1, QColor(255, 0, 0), "Urban"),
    QgsPalettedRasterRenderer.Class(2, QColor(0, 255, 0), "Forest"),
    QgsPalettedRasterRenderer.Class(3, QColor(0, 0, 255), "Water"),
]
renderer = QgsPalettedRasterRenderer(rlayer.dataProvider(), 1, classes)
rlayer.setRenderer(renderer)
rlayer.triggerRepaint()
```

**Other renderers:** `QgsHillshadeRenderer`, `QgsRasterContourRenderer`, `QgsSingleBandColorDataRenderer`.

### Terrain Analysis

Terrain analysis functions are available via the Processing framework:

```python
# Hillshade
result = processing.run("native:hillshade", {
    'INPUT': dem_layer,
    'Z_FACTOR': 1.0,
    'AZIMUTH': 315,
    'V_ANGLE': 45,
    'OUTPUT': 'memory:'
})

# Slope
result = processing.run("native:slope", {
    'INPUT': dem_layer,
    'Z_FACTOR': 1.0,
    'OUTPUT': 'memory:'
})

# Aspect
result = processing.run("native:aspect", {
    'INPUT': dem_layer,
    'Z_FACTOR': 1.0,
    'OUTPUT': 'memory:'
})

# Ruggedness index
result = processing.run("native:ruggednessindex", {
    'INPUT': dem_layer,
    'Z_FACTOR': 1.0,
    'OUTPUT': 'memory:'
})
```

### Raster Statistics

```python
from qgis.core import QgsRasterBandStats

stats = rlayer.dataProvider().bandStatistics(
    1,                                    # Band number
    QgsRasterBandStats.All,              # Statistics to compute
    rlayer.extent(),                      # Extent
    0                                     # Sample size (0 = all pixels)
)
print(f"Min: {stats.minimumValue}")
print(f"Max: {stats.maximumValue}")
print(f"Mean: {stats.mean}")
print(f"StdDev: {stats.stdDev}")
print(f"Sum: {stats.sum}")
print(f"Range: {stats.range}")
```

### Raster Histogram

```python
histogram = rlayer.dataProvider().histogram(
    1,             # Band number
    256            # Bin count
)
# histogram.histogramVector contains the counts per bin
# histogram.minimum, histogram.maximum define the value range
```

### Raster Reprojection

```python
result = processing.run("gdal:warpreproject", {
    'INPUT': rlayer,
    'SOURCE_CRS': 'EPSG:4326',
    'TARGET_CRS': 'EPSG:3857',
    'RESAMPLING': 0,           # 0=Nearest, 1=Bilinear, 2=Cubic, etc.
    'NODATA': -9999,
    'TARGET_RESOLUTION': None,
    'OUTPUT': '/path/to/reprojected.tif'
})
```

---

## 3. Print Layout System

### QgsPrintLayout Creation and Management

```python
from qgis.core import QgsPrintLayout, QgsProject

project = QgsProject.instance()
layout = QgsPrintLayout(project)
layout.initializeDefaults()  # Creates default A4 page
layout.setName("My Layout")

# Register with project for persistence
manager = project.layoutManager()
manager.addLayout(layout)
```

### QgsLayoutManager: Adding/Removing Layouts

```python
manager = QgsProject.instance().layoutManager()

# Add layout
manager.addLayout(layout)

# List all layouts
for l in manager.printLayouts():
    print(l.name())

# Get layout by name
my_layout = manager.layoutByName("My Layout")

# Remove layout
manager.removeLayout(layout)
```

### Layout Items

All layout items inherit from `QgsLayoutItem` and use these common positioning methods:
- `attemptMove(QgsLayoutPoint)` — set position on page
- `attemptResize(QgsLayoutSize)` — set item dimensions
- `setFrameEnabled(bool)` — toggle border frame

**QgsLayoutItemMap — Map display:**

```python
from qgis.core import QgsLayoutItemMap, QgsLayoutSize, QgsLayoutPoint, QgsUnitTypes

map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(200, 150, QgsUnitTypes.LayoutMillimeters))
map_item.attemptMove(QgsLayoutPoint(10, 10, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(layer.extent())
# map_item.setScale(50000)       # Set specific scale
# map_item.setCrs(QgsCoordinateReferenceSystem("EPSG:3857"))
layout.addLayoutItem(map_item)
```

**QgsLayoutItemLabel — Text labels:**

```python
from qgis.core import QgsLayoutItemLabel

label = QgsLayoutItemLabel(layout)
label.setText("My Map Title")
label.adjustSizeToText()
label.attemptMove(QgsLayoutPoint(10, 5, QgsUnitTypes.LayoutMillimeters))
# Expressions in labels: label.setText("[% @project_title %]")
layout.addLayoutItem(label)
```

**QgsLayoutItemLegend — Legend:**

```python
from qgis.core import QgsLayoutItemLegend

legend = QgsLayoutItemLegend(layout)
legend.setLinkedMap(map_item)
legend.attemptMove(QgsLayoutPoint(220, 10, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(legend)
```

**QgsLayoutItemScaleBar — Scale bar:**

```python
from qgis.core import QgsLayoutItemScaleBar

scalebar = QgsLayoutItemScaleBar(layout)
scalebar.setStyle("Single Box")  # or "Double Box", "Line Ticks Middle", "Numeric"
scalebar.setLinkedMap(map_item)
scalebar.applyDefaultSize()
scalebar.attemptMove(QgsLayoutPoint(10, 170, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(scalebar)
```

**QgsLayoutItemPicture — Images and north arrows:**

```python
from qgis.core import QgsLayoutItemPicture

picture = QgsLayoutItemPicture(layout)
picture.setPicturePath("/path/to/north_arrow.svg")
picture.attemptResize(QgsLayoutSize(20, 20, QgsUnitTypes.LayoutMillimeters))
picture.attemptMove(QgsLayoutPoint(250, 10, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(picture)
```

**QgsLayoutItemPolygon/Polyline — Shapes:**

```python
from qgis.core import QgsLayoutItemPolygon, QgsFillSymbol
from qgis.PyQt.QtCore import QPointF
from qgis.PyQt.QtGui import QPolygonF

polygon = QgsLayoutItemPolygon(
    QPolygonF([QPointF(0, 0), QPointF(100, 0), QPointF(100, 50), QPointF(0, 50)]),
    layout
)
props = {'color': '0,0,0,0', 'style': 'no', 'outline_color': 'black', 'outline_width': '0.5'}
symbol = QgsFillSymbol.createSimple(props)
polygon.setSymbol(symbol)
layout.addLayoutItem(polygon)
```

**QgsLayoutItemAttributeTable — Data tables:**

```python
from qgis.core import QgsLayoutItemAttributeTable, QgsLayoutFrame

table = QgsLayoutItemAttributeTable.create(layout)
table.setVectorLayer(layer)
table.setMaximumNumberOfFeatures(20)
frame = QgsLayoutFrame(layout, table)
frame.attemptResize(QgsLayoutSize(200, 100, QgsUnitTypes.LayoutMillimeters))
frame.attemptMove(QgsLayoutPoint(10, 200, QgsUnitTypes.LayoutMillimeters))
table.addFrame(frame)
layout.addMultiFrame(table)
```

### Page Setup

```python
from qgis.core import QgsLayoutSize, QgsUnitTypes

# Access existing page
page = layout.pageCollection().page(0)

# Set custom page size (width x height in mm)
page.setPageSize(QgsLayoutSize(297, 420, QgsUnitTypes.LayoutMillimeters))  # A3

# Add additional pages
from qgis.core import QgsLayoutItemPage
new_page = QgsLayoutItemPage(layout)
new_page.setPageSize(QgsLayoutSize(297, 210, QgsUnitTypes.LayoutMillimeters))
layout.pageCollection().addPage(new_page)
```

### Atlas Generation

```python
atlas = layout.atlas()
atlas.setCoverageLayer(coverage_layer)
atlas.setEnabled(True)
atlas.setFilenameExpression("'output_' || @atlas_featurenumber")
# atlas.setFilterFeatures(True)
# atlas.setFilterExpression('"type" = \'residential\'')

# Iterate atlas features
atlas.beginRender()
while atlas.next():
    print(f"Feature {atlas.currentFeatureNumber()}: {atlas.nameForPage(atlas.currentFeatureNumber())}")
atlas.endRender()
```

### Export: QgsLayoutExporter

```python
from qgis.core import QgsLayoutExporter

exporter = QgsLayoutExporter(layout)

# Export to PDF
pdf_settings = QgsLayoutExporter.PdfExportSettings()
pdf_settings.dpi = 300
result = exporter.exportToPdf("/path/to/output.pdf", pdf_settings)

# Export to image (PNG, JPEG, TIFF)
img_settings = QgsLayoutExporter.ImageExportSettings()
img_settings.dpi = 300
result = exporter.exportToImage("/path/to/output.png", img_settings)

# Export to SVG
svg_settings = QgsLayoutExporter.SvgExportSettings()
svg_settings.dpi = 300
result = exporter.exportToSvg("/path/to/output.svg", svg_settings)

# Export atlas pages
pdf_settings = QgsLayoutExporter.PdfExportSettings()
result = QgsLayoutExporter.exportToPdfs(
    layout.atlas(),
    "/path/to/atlas_output/",
    pdf_settings
)
```

**Export result codes:** `Success = 0`, `Canceled`, `MemoryError`, `FileError`, `PrintError`, `SvgLayerError`, `IteratorError`.

### Template Save/Load

```python
from qgis.core import QgsReadWriteContext
from qgis.PyQt.QtXml import QDomDocument

# Save layout as .qpt template
doc = QDomDocument()
layout.writeXml(doc, QgsReadWriteContext())
with open("/path/to/template.qpt", "w") as f:
    f.write(doc.toString())

# Load layout from .qpt template
with open("/path/to/template.qpt", "r") as f:
    content = f.read()
doc = QDomDocument()
doc.setContent(content)

new_layout = QgsPrintLayout(project)
new_layout.readXml(doc.documentElement(), doc, QgsReadWriteContext())
new_layout.setName("From Template")
manager.addLayout(new_layout)
```

---

## 4. PostGIS Integration

### Connection Setup via QgsDataSourceUri

```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer

uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "mydb", "user", "password")
uri.setDataSource("public", "roads", "geom")  # schema, table, geometry column

# Optional: add SQL filter and primary key
uri.setDataSource("public", "roads", "geom", "speed_limit > 50", "gid")

# ALWAYS use uri(False) to prevent password exposure in logs
vlayer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
```

### Loading PostGIS Layers

**Table layer:**
```python
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "johny", "xxx")
uri.setDataSource("public", "roads", "the_geom")
vlayer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
```

**View layer:**
```python
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "johny", "xxx")
uri.setDataSource("public", "my_view", "geom", "", "view_id")
# ALWAYS specify a unique key column for views
vlayer = QgsVectorLayer(uri.uri(False), "My View", "postgres")
```

**SQL query layer (virtual table):**
```python
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "johny", "xxx")
uri.setDataSource("", "(SELECT * FROM roads WHERE type='highway')", "the_geom", "", "gid")
vlayer = QgsVectorLayer(uri.uri(False), "Highways", "postgres")
```

### Executing SQL Queries

```python
from qgis.core import QgsProviderRegistry

md = QgsProviderRegistry.instance().providerMetadata('postgres')
conn = md.createConnection(uri.uri(False), {})

# Execute SQL (non-spatial queries)
results = conn.executeSql("SELECT count(*) FROM public.roads")

# Execute SQL (returns QgsVectorLayer for spatial)
# Use uri.setDataSource with SQL subquery as shown above
```

### QgsAbstractDatabaseProviderConnection

```python
from qgis.core import QgsProviderRegistry

md = QgsProviderRegistry.instance().providerMetadata('postgres')
conn = md.createConnection(uri.uri(False), {})

# Schema and table discovery
schemas = conn.schemas()
tables = conn.tables("public")  # List tables in schema

for table in tables:
    print(f"Table: {table.tableName()}, Geometry: {table.geometryColumn()}")
```

### Creating Tables from QGIS Layers

```python
# Export layer to PostGIS
save_options = QgsVectorFileWriter.SaveVectorOptions()
save_options.driverName = "PostgreSQL"

uri_str = f"PG:host=localhost port=5432 dbname=gisdb user=johny password=xxx"
error = QgsVectorFileWriter.writeAsVectorFormatV3(
    layer,
    uri_str,
    QgsCoordinateTransformContext(),
    save_options
)
```

Alternatively, use the Processing algorithm:
```python
result = processing.run("native:importintopostgis", {
    'INPUT': layer,
    'DATABASE': 'my_connection',  # Stored connection name
    'SCHEMA': 'public',
    'TABLENAME': 'new_table',
    'PRIMARY_KEY': 'id',
    'GEOMETRY_COLUMN': 'geom',
    'ENCODING': 'UTF-8',
    'OVERWRITE': True,
    'CREATEINDEX': True,
    'LOWERCASE_NAMES': True,
    'DROP_STRING_LENGTH': False,
    'FORCE_SINGLEPART': False
})
```

### PostGIS Raster Loading

```python
from qgis.core import QgsProviderRegistry, QgsDataSourceUri

uri_config = {
    'dbname': 'gis_db',
    'host': 'localhost',
    'port': '5432',
    'sslmode': QgsDataSourceUri.SslDisable,
    'authcfg': 'QconfigId',       # Or use user/password
    'schema': 'public',
    'table': 'my_rasters',
    'geometrycolumn': 'rast',
    'mode': '2'                    # 2 = union raster tiles into one layer
}
uri_config = {k: v for k, v in uri_config.items() if v is not None}
md = QgsProviderRegistry.instance().providerMetadata('postgresraster')
uri = QgsDataSourceUri(md.encodeUri(uri_config))
rlayer = QgsRasterLayer(uri.uri(False), "PostGIS Raster", "postgresraster")
```

### Connection Management

**Stored connections** are managed via QGIS settings and can be accessed by name:
```python
# Using stored connection name (configured in QGIS Data Source Manager)
md = QgsProviderRegistry.instance().providerMetadata('postgres')
conn = md.createConnection("My PostGIS Server")
```

### Authentication for Database Connections

Use the authentication infrastructure instead of plain-text passwords:

```python
from qgis.core import QgsDataSourceUri

uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "mydb", "", "")
uri.setAuthConfigId("abc123")  # Reference stored auth config
uri.setDataSource("public", "roads", "geom")

vlayer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
```

---

## 5. Web Services (OGC)

### WMS/WMTS Client

**WMS URI format:**
```
crs=EPSG:4326&format=image/png&layers=layername&styles=&url=https://server/wms
```

**Loading a WMS layer:**
```python
from qgis.core import QgsRasterLayer

url_with_params = (
    "crs=EPSG:4326"
    "&format=image/png"
    "&layers=continents"
    "&styles="
    "&url=https://demo.mapserver.org/cgi-bin/wms"
)
rlayer = QgsRasterLayer(url_with_params, "WMS Layer", "wms")
if rlayer.isValid():
    QgsProject.instance().addMapLayer(rlayer)
```

**WMTS loading** uses the same `wms` provider with additional parameters:
```
crs=EPSG:3857&format=image/png&layers=layername&styles=default&tilematrixset=GoogleMapsCompatible&url=https://server/wmts
```

### WFS Client

**WFS URI format:**
```
https://server/wfs?service=WFS&version=2.0.0&request=GetFeature&typename=namespace:layername
```

**Loading a WFS layer:**
```python
from qgis.core import QgsVectorLayer

uri = (
    "https://demo.mapserver.org/cgi-bin/wfs"
    "?service=WFS"
    "&version=2.0.0"
    "&request=GetFeature"
    "&typename=ms:cities"
)
vlayer = QgsVectorLayer(uri, "WFS Cities", "WFS")
if vlayer.isValid():
    QgsProject.instance().addMapLayer(vlayer)
```

**WFS version differences:**
- WFS 1.0.0: Uses `typeName` (singular) parameter
- WFS 1.1.0: Introduces stored queries, GML 3.1 output
- WFS 2.0.0: Uses `typeNames` (plural), supports paging, temporal filters

### WCS Client

**WCS URI parameters:**
- `url` (required): Server URL
- `identifier` (required): Coverage name
- `time` (optional): Temporal position or period
- `format` (optional): Output format
- `crs` (optional): e.g., `EPSG:4326`
- `username`/`password` (optional)
- `cache` (optional): `AlwaysCache`, `PreferCache`, `PreferNetwork`, `AlwaysNetwork`

```python
layer_name = 'modis'
url = f"https://demo.mapserver.org/cgi-bin/wcs?identifier={layer_name}"
rlayer = QgsRasterLayer(url, "WCS Coverage", "wcs")
```

### XYZ Tile Layers

**XYZ URI format:**
```
type=xyz&url=https://tile.server.com/{z}/{x}/{y}.png&zmax=19&zmin=0&crs=EPSG3857
```

**Loading XYZ tiles:**
```python
from qgis.core import QgsRasterLayer, QgsProject

xyz_url = "https://tile.openstreetmap.org/{z}/{x}/{y}.png"
uri = f"type=xyz&url={xyz_url}&zmax=19&zmin=0&crs=EPSG3857"
rlayer = QgsRasterLayer(uri, "OpenStreetMap", "wms")

if rlayer.isValid():
    QgsProject.instance().addMapLayer(rlayer)
```

**Common XYZ tile sources:**
- OpenStreetMap: `https://tile.openstreetmap.org/{z}/{x}/{y}.png`
- Stamen Terrain: `https://tiles.stadiamaps.com/tiles/stamen_terrain/{z}/{x}/{y}.png`

### QGIS Server Setup and Configuration

**Minimal server usage in Python:**
```python
from qgis.core import QgsApplication
from qgis.server import QgsServer, QgsBufferServerRequest, QgsBufferServerResponse

app = QgsApplication([], False)
server = QgsServer()

# Handle a WMS GetCapabilities request
request = QgsBufferServerRequest(
    'http://localhost:8081/?SERVICE=WMS&REQUEST=GetCapabilities'
    '&MAP=/path/to/project.qgs'
)
response = QgsBufferServerResponse()
server.handleRequest(request, response)

print(response.body())
app.exitQgis()
```

**IMPORTANT:** QGIS server classes are NOT thread-safe. ALWAYS use a multiprocessing model or containers for scalable applications.

### Server Plugins and Filters

**Plugin registration:**
```python
class MyServerPlugin:
    def __init__(self, serverIface):
        self.serverIface = serverIface
        serverIface.registerFilter(MyFilter(serverIface), 100)

def serverClassFactory(serverIface):
    return MyServerPlugin(serverIface)
```

**I/O Filter (QgsServerFilter):**
```python
from qgis.server import QgsServerFilter

class MyFilter(QgsServerFilter):
    def __init__(self, serverIface):
        super().__init__(serverIface)

    def onRequestReady(self):
        """Called before core service processing. Return False to stop chain."""
        request = self.serverInterface().requestHandler()
        params = request.parameterMap()
        return True

    def onSendResponse(self):
        """Called on partial output flush."""
        return True

    def onResponseComplete(self):
        """Called after service completion."""
        request = self.serverInterface().requestHandler()
        # Modify response body, headers, etc.
        return True
```

**Access Control Filter (QgsAccessControlFilter):**
```python
from qgis.server import QgsAccessControlFilter

class MyAccessControl(QgsAccessControlFilter):
    def layerFilterExpression(self, layer):
        return '"public" = true'

    def layerFilterSubsetString(self, layer):
        return "public = true"

    def layerPermissions(self, layer):
        permissions = self.LayerPermissions()
        permissions.canRead = True
        permissions.canInsert = False
        permissions.canUpdate = False
        permissions.canDelete = False
        return permissions

    def authorizedLayerAttributes(self, layer, attributes):
        return [a for a in attributes if a != 'secret_field']

    def allowToEdit(self, layer, feature):
        return False

    def cacheKey(self):
        return "my_access_control"
```

**Cache Filter (QgsServerCacheFilter):** registered via `serverIface.registerServerCache()`.

### QgsServerOgcApi: OGC API Features

Custom OGC API handler:
```python
from qgis.server import QgsServerOgcApiHandler

class MyApiHandler(QgsServerOgcApiHandler):
    def path(self):
        return "/my-endpoint"

    def operationId(self):
        return "myEndpoint"

    def handleRequest(self, context):
        context.response().setStatusCode(200)
        context.response().write('{"result": "ok"}')
```

Register via `serverIface.serviceRegistry().registerApi(api)`.

**Custom Service:**
```python
from qgis.server import QgsService

class CustomService(QgsService):
    def name(self):
        return "CUSTOM"

    def version(self):
        return "1.0.0"

    def executeRequest(self, request, response, project):
        response.setStatusCode(200)
        response.write("Custom service response")

# Register: serverIface.serviceRegistry().registerService(service)
```

### Authentication for Web Services

Use the `authcfg` parameter in URIs:

```python
from qgis.core import QgsDataSourceUri, QgsRasterLayer

auth_cfg = 'fm1s770'  # Stored auth config ID

quri = QgsDataSourceUri()
quri.setParam("layers", "usa:states")
quri.setParam("format", "image/png")
quri.setParam("authcfg", auth_cfg)
quri.setParam("url", "https://server/wms")
quri.setParam("crs", "EPSG:4326")
quri.setParam("styles", "")

rlayer = QgsRasterLayer(str(quri.encodedUri(), "utf-8"), "Authenticated WMS", "wms")
```

**Supported auth methods for OGC services:** Basic, PKI-Paths, PKI-PKCS#12, Identity-Cert, OAuth2, APIHeader, MapTilerHmacSha256.

**Storing authentication config:**
```python
from qgis.core import QgsAuthMethodConfig, QgsApplication

config = QgsAuthMethodConfig()
config.setName("My WMS Auth")
config.setMethod("Basic")
config.setUri("https://server/wms")
config.setConfig("username", "myuser")
config.setConfig("password", "mypassword")

authMgr = QgsApplication.authManager()
authMgr.storeAuthenticationConfig(config)
auth_id = config.id()  # Auto-generated ID like 'fm1s770'
```

---

## 6. 3D Visualization

### Qgs3DMapSettings Configuration

`Qgs3DMapSettings` is the central configuration class for 3D scenes. It manages CRS, extent, terrain, lighting, camera, and layer rendering.

```python
from qgis._3d import Qgs3DMapSettings
from qgis.core import QgsCoordinateReferenceSystem, QgsRectangle

settings = Qgs3DMapSettings()
settings.setCrs(QgsCoordinateReferenceSystem("EPSG:3857"))
settings.setExtent(QgsRectangle(1000000, 6000000, 2000000, 7000000))
# setExtent automatically sets origin to extent center
settings.setBackgroundColor(QColor(135, 206, 235))  # Sky blue
settings.setSelectionColor(QColor(255, 255, 0))
settings.setOutputDpi(96)
```

**Key methods:**
- `setCrs()` / `crs()` — CRS for the 3D scene
- `setExtent()` / `extent()` — 2D extent; auto-sets origin
- `setOrigin()` / `origin()` — World origin (0,0,0) in map coordinates
- `setLayers()` / `layers()` — Layers to render in 3D
- `setBackgroundColor()` / `backgroundColor()`
- `setOutputDpi()` / `outputDpi()`
- `mapToWorldCoordinates(QgsVector3D)` — Convert map CRS to 3D world (applies x, -z, y swap)
- `worldToMapCoordinates(QgsVector3D)` — Inverse transformation

### Terrain Providers

Terrain configuration is managed via `terrainSettings()` (QGIS 3.42+) or the older `setTerrainGenerator()` approach:

**Flat terrain:**
```python
from qgis._3d import QgsFlatTerrainSettings

terrain = QgsFlatTerrainSettings()
terrain.setElevation(0.0)
settings.setTerrainSettings(terrain)
```

**DEM terrain:**
```python
from qgis._3d import QgsDemTerrainSettings

terrain = QgsDemTerrainSettings()
terrain.setLayer(dem_raster_layer)
terrain.setResolution(16)        # Tile resolution
terrain.setSkirtHeight(10.0)
settings.setTerrainSettings(terrain)
```

**From project elevation properties:**
```python
settings.configureTerrainFromProject(
    QgsProject.instance().elevationProperties(),
    QgsRectangle(...)
)
```

**Terrain rendering control:**
- `setTerrainRenderingEnabled(bool)` — Toggle terrain visibility
- `setTerrainVerticalScale(float)` — Vertical exaggeration factor
- `setTerrainElevationOffset(float)` — Elevation offset
- `setTerrainShadingEnabled(bool)` — Enable lighting on terrain
- `setTerrainShadingMaterial(QgsPhongMaterialSettings)` — Material for terrain shading
- `setTerrainMapTheme(str)` — Map theme for terrain texture

### Vector Layer 3D Renderers

Vector layers can be rendered in 3D by assigning a 3D renderer via the layer's `setRenderer3D()` method. [VERIFY: exact Python API for setting 3D renderers on vector layers — the C++ API uses `QgsVectorLayer3DRenderer` but Python bindings may differ]

**Common 3D renderer workflow:**
```python
from qgis._3d import QgsVectorLayer3DRenderer, QgsPolygon3DSymbol

symbol = QgsPolygon3DSymbol()
symbol.setHeight(0)
symbol.setExtrusionHeight(50)  # Extrude polygons 50 units up

renderer = QgsVectorLayer3DRenderer()
renderer.setSymbol(symbol)
vector_layer.setRenderer3D(renderer)
```

### 3D Symbols

**QgsPoint3DSymbol:** Renders point features as 3D objects.
- Shape types: Sphere, Cylinder, Cube, Cone, Plane, Torus, ExtrudedText [VERIFY]
- `setShape()`, `setHeight()`, `setMaterialSettings()`

**QgsLine3DSymbol:** Renders line features in 3D.
- `setWidth()`, `setHeight()`, `setExtrusionHeight()`, `setMaterialSettings()`

**QgsPolygon3DSymbol:** Renders polygon features with extrusion.
- `setHeight()`, `setExtrusionHeight()`, `setMaterialSettings()`
- `setAltitudeClamping()` — Absolute, Relative, Terrain
- `setAltitudeBinding()` — Vertex, Centroid

### Materials

**QgsPhongMaterialSettings:**
```python
from qgis._3d import QgsPhongMaterialSettings

material = QgsPhongMaterialSettings()
material.setAmbient(QColor(50, 50, 50))
material.setDiffuse(QColor(200, 100, 50))
material.setSpecular(QColor(255, 255, 255))
material.setShininess(100.0)
```

**QgsGoochMaterialSettings:** Warm/cool non-photorealistic rendering. [VERIFY: exact Python API]

**QgsMetalRoughMaterialSettings:** PBR metallic-roughness material. [VERIFY: exact Python API]

### Camera and Light Settings

**Camera configuration:**
```python
settings.setFieldOfView(45)                     # Lens FOV (since QGIS 3.8)
settings.setCameraNavigationMode(...)            # Navigation mode (since QGIS 3.18)
settings.setCameraMovementSpeed(5.0)             # Movement speed (since QGIS 3.18)
settings.setProjectionType(...)                  # Perspective or orthographic (since QGIS 3.18)
```

**Light sources (since QGIS 3.26):**
```python
settings.setLightSources([light1, light2])  # List of QgsLightSource objects
settings.setShowLightSourceOrigins(True)    # Visualize light positions
```

Light types include `QgsDirectionalLightSettings` and `QgsPointLightSettings` [VERIFY: exact class names in Python].

**Eye Dome Lighting (since QGIS 3.18):**
```python
settings.setEyeDomeLightingEnabled(True)
settings.setEyeDomeLightingStrength(1000.0)
settings.setEyeDomeLightingDistance(1)
```

**Shadows (since QGIS 3.16):**
```python
shadow = QgsShadowSettings()
settings.setShadowSettings(shadow)
```

### 3D Map in Layouts (QgsLayoutItem3DMap)

```python
from qgis.core import QgsLayoutItem3DMap

map3d = QgsLayoutItem3DMap(layout)
map3d.setMapSettings(settings)  # Qgs3DMapSettings
map3d.attemptResize(QgsLayoutSize(200, 150, QgsUnitTypes.LayoutMillimeters))
map3d.attemptMove(QgsLayoutPoint(10, 10, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(map3d)
```

[VERIFY: `QgsLayoutItem3DMap` availability in PyQGIS — this class exists in C++ API since QGIS 3.4 but Python bindings may be limited]

### Exporting 3D Scenes

3D scene export is available via the `Qgs3DMapExportSettings` class and the `Qgs3DMapScene::exportScene()` method. [VERIFY: exact Python API for 3D export — this functionality may be limited in PyQGIS]

**Serialization:**
```python
from qgis.PyQt.QtXml import QDomDocument

doc = QDomDocument()
elem = doc.createElement("3d-settings")
settings.writeXml(elem, QgsReadWriteContext())
doc.appendChild(elem)

# Restore
settings2 = Qgs3DMapSettings()
settings2.readXml(elem, QgsReadWriteContext())
settings2.resolveReferences(QgsProject.instance())
```

---

## 7. Georeferencing

### QgsGcpPoint: Ground Control Point Management

Available since QGIS 3.26. Located in the analysis library.

```python
from qgis.analysis import QgsGcpPoint
from qgis.core import QgsPointXY, QgsCoordinateReferenceSystem

gcp = QgsGcpPoint(
    QgsPointXY(100, 200),                              # Source point (pixel coords)
    QgsPointXY(15.5, 47.1),                            # Destination point (map coords)
    QgsCoordinateReferenceSystem("EPSG:4326"),         # Destination CRS
    True                                                # Enabled
)

# Access properties
src = gcp.sourcePoint()           # QgsPointXY
dst = gcp.destinationPoint()      # QgsPointXY
crs = gcp.destinationPointCrs()   # QgsCoordinateReferenceSystem
enabled = gcp.isEnabled()

# Transform destination to different CRS
transformed = gcp.transformedDestinationPoint(
    QgsCoordinateReferenceSystem("EPSG:3857"),
    QgsCoordinateTransformContext()
)

# Modify
gcp.setSourcePoint(QgsPointXY(150, 250))
gcp.setDestinationPoint(QgsPointXY(16.0, 47.5))
gcp.setEnabled(False)
```

### Transformation Types

Defined in `QgsGcpTransformerInterface.TransformMethod` enum (since QGIS 3.20):

| Method | Enum Value | Min GCPs | Description |
|--------|-----------|----------|-------------|
| Linear | `Linear` | 2 | Simple offset and scale |
| Helmert | `Helmert` | 2 | Rotation, scale, translation |
| Polynomial Order 1 | `PolynomialOrder1` | 3 | First-order polynomial (affine) |
| Polynomial Order 2 | `PolynomialOrder2` | 6 | Second-order polynomial |
| Polynomial Order 3 | `PolynomialOrder3` | 10 | Third-order polynomial |
| Thin Plate Spline | `ThinPlateSpline` | 1 | Rubber-sheet transformation |
| Projective | `Projective` | 4 | Projective (perspective) transformation |

### QgsGcpTransformerInterface

```python
from qgis.analysis import QgsGcpTransformerInterface

# Create transformer from method
transformer = QgsGcpTransformerInterface.create(
    QgsGcpTransformerInterface.PolynomialOrder1
)

# Or create from GCP coordinates directly
source_points = [QgsPointXY(0, 0), QgsPointXY(100, 0), QgsPointXY(100, 100)]
dest_points = [QgsPointXY(15.0, 47.0), QgsPointXY(16.0, 47.0), QgsPointXY(16.0, 48.0)]

transformer = QgsGcpTransformerInterface.createFromParameters(
    QgsGcpTransformerInterface.PolynomialOrder1,
    source_points,
    dest_points
)

# Update parameters from GCPs
success = transformer.updateParametersFromGcps(source_points, dest_points, False)

# Transform a point
x, y = 50.0, 50.0
success = transformer.transform(x, y, False)  # False = forward, True = inverse

# Get method info
method = transformer.method()
min_gcps = transformer.minimumGcpCount()
method_name = QgsGcpTransformerInterface.methodToString(method)
```

### QgsVectorWarper for Vector Georeferencing

Available since QGIS 3.26. Warps vector features using GCP-based transformations.

```python
from qgis.analysis import QgsVectorWarper, QgsGcpPoint, QgsGcpTransformerInterface
from qgis.core import (QgsCoordinateReferenceSystem, QgsPointXY,
                        QgsVectorLayer, QgsFeatureSink, QgsCoordinateTransformContext)

# Define GCPs
gcps = [
    QgsGcpPoint(QgsPointXY(0, 0), QgsPointXY(15.0, 47.0),
                QgsCoordinateReferenceSystem("EPSG:4326"), True),
    QgsGcpPoint(QgsPointXY(100, 0), QgsPointXY(16.0, 47.0),
                QgsCoordinateReferenceSystem("EPSG:4326"), True),
    QgsGcpPoint(QgsPointXY(100, 100), QgsPointXY(16.0, 48.0),
                QgsCoordinateReferenceSystem("EPSG:4326"), True),
]

# Create warper
warper = QgsVectorWarper(
    QgsGcpTransformerInterface.PolynomialOrder1,
    gcps,
    QgsCoordinateReferenceSystem("EPSG:4326")
)

# Transform features
source_layer = QgsVectorLayer("/path/to/ungeoreferenced.shp", "source", "ogr")
success = warper.transformFeatures(
    source_layer.getFeatures(),
    output_sink,                     # QgsFeatureSink (e.g., from processing)
    QgsCoordinateTransformContext()
)

if not success:
    print(warper.error())
```

### Raster Georeferencing Workflow

Raster georeferencing in PyQGIS uses GDAL under the hood. The `QgsGcpTransformerInterface` provides `GDALTransformer()` and `GDALTransformerArgs()` methods for integration with GDAL's warp API.

**Processing-based approach:**
```python
result = processing.run("gdal:translate", {
    'INPUT': '/path/to/unreferenced.tif',
    'TARGET_CRS': 'EPSG:4326',
    'GCPS': '0 0 15.0 47.0|100 0 16.0 47.0|100 100 16.0 48.0',  # [VERIFY: exact format]
    'OUTPUT': '/path/to/georeferenced.tif'
})

# Then warp to apply transformation
result = processing.run("gdal:warpreproject", {
    'INPUT': '/path/to/georeferenced.tif',
    'SOURCE_CRS': 'EPSG:4326',
    'TARGET_CRS': 'EPSG:4326',
    'RESAMPLING': 0,
    'OUTPUT': '/path/to/final.tif'
})
```

### Accuracy Assessment (Residuals)

Residuals are computed by comparing transformed source points against destination points:

```python
# For each GCP, compute residual
for i, gcp in enumerate(gcps):
    src = gcp.sourcePoint()
    dst = gcp.destinationPoint()

    x, y = src.x(), src.y()
    transformer.transform(x, y, False)
    transformed = QgsPointXY(x, y)

    residual_x = transformed.x() - dst.x()
    residual_y = transformed.y() - dst.y()
    total_residual = (residual_x**2 + residual_y**2)**0.5
    print(f"GCP {i}: residual = {total_residual:.4f}")
```

### World File Output

World files (.tfw, .jgw, .pgw) store affine transformation parameters. When using GDAL for georeferencing, set the `TFW=YES` creation option:

```python
result = processing.run("gdal:translate", {
    'INPUT': '/path/to/input.tif',
    'OPTIONS': 'TFW=YES',
    'OUTPUT': '/path/to/output.tif'
})
# This creates output.tfw alongside output.tif
```

---

## 8. Network Analysis

### QgsGraphBuilder: Building Network Graphs

The network analysis library (`qgis.analysis`) builds graph structures from polyline vector layers.

```python
from qgis.analysis import (
    QgsGraphBuilder, QgsVectorLayerDirector,
    QgsNetworkDistanceStrategy, QgsNetworkSpeedStrategy,
    QgsGraphAnalyzer
)
from qgis.core import QgsPointXY

# Load road network
road_layer = QgsProject.instance().mapLayersByName("roads")[0]
```

### QgsVectorLayerDirector: Direction Handling

```python
director = QgsVectorLayerDirector(
    road_layer,
    -1,              # directionFieldId (-1 = no direction field, all bidirectional)
    '',              # directDirectionValue (value meaning "forward direction")
    '',              # reverseDirectionValue (value meaning "reverse direction")
    '',              # bothDirectionValue (value meaning "both directions")
    QgsVectorLayerDirector.DirectionBoth  # defaultDirection
)
```

**Direction constants:**
- `QgsVectorLayerDirector.DirectionForward`
- `QgsVectorLayerDirector.DirectionBackward`
- `QgsVectorLayerDirector.DirectionBoth`

**With a direction field:**
```python
direction_field_index = road_layer.fields().indexOf('oneway')
director = QgsVectorLayerDirector(
    road_layer,
    direction_field_index,
    'yes',           # Forward value
    'reverse',       # Reverse value
    'both',          # Both directions value
    QgsVectorLayerDirector.DirectionBoth  # Default for unmatched values
)
```

### Strategy Classes

Strategies define cost/weight for graph edges.

**QgsNetworkDistanceStrategy** — cost based on geometric length:
```python
strategy = QgsNetworkDistanceStrategy()
director.addStrategy(strategy)
```

**QgsNetworkSpeedStrategy** — cost based on travel time:
```python
speed_field_index = road_layer.fields().indexOf('speed_kmh')
strategy = QgsNetworkSpeedStrategy(
    speed_field_index,
    50.0,            # Default speed when field is NULL
    1000.0 / 3600.0  # toMetricFactor (km/h to m/s)
)
director.addStrategy(strategy)
```

Multiple strategies can be added. Each becomes a separate criterion (index 0, 1, ...) in the graph.

### Building the Graph

```python
# Define start and end points
start_point = QgsPointXY(15.43, 47.07)
end_point = QgsPointXY(16.37, 48.21)

# Build graph
builder = QgsGraphBuilder(road_layer.crs())
tied_points = director.makeGraph(builder, [start_point, end_point])

graph = builder.graph()

# tied_points contains snapped positions on the network
start_id = graph.findVertex(tied_points[0])
end_id = graph.findVertex(tied_points[1])
```

### QgsGraphAnalyzer.dijkstra(): Shortest Path

```python
(tree, cost) = QgsGraphAnalyzer.dijkstra(graph, start_id, 0)
# tree: list of incoming edge indices (-1 = no incoming edge / unreachable)
# cost: list of costs from start to each vertex

# Check if destination is reachable
if tree[end_id] == -1:
    print("No path found!")
else:
    print(f"Cost to destination: {cost[end_id]}")

    # Reconstruct path (from end to start)
    route = [graph.vertex(end_id).point()]
    current = end_id
    while current != start_id:
        incoming_edges = graph.vertex(current).incomingEdges()
        edge = graph.edge(incoming_edges[0])
        current = edge.fromVertex()
        route.insert(0, graph.vertex(current).point())

    # Create geometry from route
    from qgis.core import QgsGeometry, QgsPoint
    geom = QgsGeometry.fromPolyline([QgsPoint(p.x(), p.y()) for p in route])
```

### QgsGraphAnalyzer.shortestTree(): Shortest Path Tree

```python
tree_graph = QgsGraphAnalyzer.shortestTree(graph, start_id, 0)
# Returns a new QgsGraph containing only the shortest path tree
# Useful for visualizing accessibility from a single origin
```

### Service Area Analysis

Find all vertices reachable within a cost threshold:

```python
(tree, cost) = QgsGraphAnalyzer.dijkstra(graph, start_id, 0)

threshold = 5000.0  # e.g., 5000 meters or 5000 seconds

# Interior vertices (fully reachable)
reachable_points = []
for vertex_id in range(graph.vertexCount()):
    if cost[vertex_id] <= threshold and tree[vertex_id] != -1:
        reachable_points.append(graph.vertex(vertex_id).point())

# Boundary vertices (edge crosses threshold)
boundary_points = []
for vertex_id in range(graph.vertexCount()):
    if cost[vertex_id] > threshold and tree[vertex_id] != -1:
        edge = graph.edge(tree[vertex_id])
        from_vertex = edge.fromVertex()
        if cost[from_vertex] < threshold:
            # Interpolate position on edge at threshold
            ratio = (threshold - cost[from_vertex]) / (cost[vertex_id] - cost[from_vertex])
            p1 = graph.vertex(from_vertex).point()
            p2 = graph.vertex(vertex_id).point()
            interpolated = QgsPointXY(
                p1.x() + ratio * (p2.x() - p1.x()),
                p1.y() + ratio * (p2.y() - p1.y())
            )
            boundary_points.append(interpolated)
```

### Building Graphs from Road Networks

Complete workflow for road network analysis:

```python
from qgis.analysis import *
from qgis.core import QgsProject, QgsPointXY, QgsGeometry, QgsPoint, QgsFeature, QgsVectorLayer

# 1. Load road network
roads = QgsProject.instance().mapLayersByName("roads")[0]

# 2. Configure director with one-way streets
oneway_idx = roads.fields().indexOf('oneway')
director = QgsVectorLayerDirector(
    roads, oneway_idx,
    'F',                                     # Forward
    'T',                                     # Reverse
    'B',                                     # Both
    QgsVectorLayerDirector.DirectionBoth     # Default
)

# 3. Add distance strategy
director.addStrategy(QgsNetworkDistanceStrategy())

# 4. Build graph with analysis points
points = [QgsPointXY(15.43, 47.07), QgsPointXY(16.37, 48.21)]
builder = QgsGraphBuilder(roads.crs())
tied = director.makeGraph(builder, points)
graph = builder.graph()

# 5. Find shortest path
sid = graph.findVertex(tied[0])
eid = graph.findVertex(tied[1])
(tree, cost) = QgsGraphAnalyzer.dijkstra(graph, sid, 0)

# 6. Extract route geometry
if tree[eid] != -1:
    route_points = [graph.vertex(eid).point()]
    cur = eid
    while cur != sid:
        edge = graph.edge(tree[cur])
        cur = edge.fromVertex()
        route_points.insert(0, graph.vertex(cur).point())

    route_geom = QgsGeometry.fromPolyline(
        [QgsPoint(p.x(), p.y()) for p in route_points]
    )
    print(f"Route length: {route_geom.length():.1f} map units")
```

### Cost Functions and Impedance

Custom cost strategies can be implemented by subclassing the strategy interface:

```python
# Multiple criteria: distance (index 0) and time (index 1)
director.addStrategy(QgsNetworkDistanceStrategy())        # criterion 0

speed_idx = roads.fields().indexOf('speed')
director.addStrategy(QgsNetworkSpeedStrategy(
    speed_idx, 50.0, 1000.0 / 3600.0
))                                                         # criterion 1

# Build graph
builder = QgsGraphBuilder(roads.crs())
tied = director.makeGraph(builder, points)
graph = builder.graph()

# Shortest by distance
(tree_dist, cost_dist) = QgsGraphAnalyzer.dijkstra(graph, start_id, 0)

# Fastest by time
(tree_time, cost_time) = QgsGraphAnalyzer.dijkstra(graph, start_id, 1)
```

---

## 9. Anti-patterns and Critical Warnings

### PostGIS
- **NEVER** store passwords in plain text URIs in production. ALWAYS use `QgsAuthManager` with stored authentication configurations (`authcfg`). Use `uri.uri(False)` to prevent credential expansion in logs.
- **NEVER** load PostGIS views without specifying a unique key column — this causes undefined behavior and performance issues.
- **ALWAYS** use `QgsDataSourceUri` to construct connection strings rather than manual string concatenation.

### WMS
- **ALWAYS** verify layer capabilities before loading: check `rlayer.isValid()` immediately after construction. An invalid WMS layer silently fails.
- **NEVER** assume a WMS layer supports all CRS — check the GetCapabilities response.
- **ALWAYS** URL-encode special characters in XYZ tile URLs: `{z}`, `{x}`, `{y}` must be properly encoded as `%7Bz%7D`, `%7Bx%7D`, `%7By%7D` when constructing the URI parameter string.

### Print Layouts
- **ALWAYS** set map extent before export — exporting without a defined extent produces blank or incorrectly positioned maps. Use `map_item.zoomToExtent()` or `map_item.setExtent()`.
- **ALWAYS** call `layout.initializeDefaults()` after creating a new `QgsPrintLayout` — without this, the layout has no pages.
- **NEVER** export a layout without checking the export result code — `QgsLayoutExporter` returns status codes indicating success or specific failure types.

### Network Analysis
- **NEVER** assume bidirectional edges without checking — road networks contain one-way streets. ALWAYS configure `QgsVectorLayerDirector` with the correct direction field and values.
- **ALWAYS** check if `tree[destination_id] == -1` before path reconstruction — this indicates the destination is unreachable.
- **ALWAYS** use `graph.findVertex(tied_point)` with the tied (snapped) point, not the original point — the original point may not exist in the graph.

### Raster
- **ALWAYS** check for NoData values before calculations. Use `provider.sourceHasNoDataValue(band)` and `provider.sourceNoDataValue(band)` to identify NoData values.
- **NEVER** assume raster bands are in RGB order for multiband files — check `bandCount()` and `bandName()` to verify band assignments.
- **ALWAYS** verify the `result` boolean from `provider.sample()` — a `False` result means the sample point is outside the raster extent or on a NoData pixel.

### 3D Visualization
- **NEVER** load very large point clouds without LOD (Level of Detail) settings — this causes out-of-memory crashes and rendering freezes.
- **ALWAYS** set `setExtent()` before configuring terrain — the origin is auto-calculated from the extent, and terrain rendering depends on correct origin placement.
- **NEVER** use `Qgs3DMapSettings` from multiple threads — like QGIS Server classes, 3D components are not thread-safe.

### Georeferencing
- **ALWAYS** verify GCP accuracy (residuals) before applying a transformation — high residuals indicate incorrect GCP placement or an inappropriate transformation method.
- **NEVER** use higher-order polynomial transformations with fewer GCPs than the minimum required — this produces mathematically undefined results. Minimum counts: Linear=2, Helmert=2, Polynomial1=3, Polynomial2=6, Polynomial3=10, Projective=4.
- **ALWAYS** check `transformer.updateParametersFromGcps()` return value — `False` indicates the GCPs are insufficient or degenerate for the chosen method.
- **NEVER** apply a Thin Plate Spline transformation for extrapolation beyond the GCP hull — TPS is an interpolation method and produces extreme distortion outside the convex hull of control points.

### Authentication
- **NEVER** hardcode credentials in Python scripts. ALWAYS use `QgsAuthManager` to store and retrieve credentials securely from the encrypted `qgis-auth.db`.
- **ALWAYS** use `uri.uri(False)` when passing URIs to layer constructors to prevent premature expansion of `authcfg` references, which would expose credentials.

### QGIS Server
- **NEVER** use QGIS Server classes from multiple threads — they are explicitly not thread-safe. ALWAYS use multiprocessing or container-based scaling.
- **ALWAYS** check filter return values — returning `False` from `onRequestReady()`, `onSendResponse()`, or `onResponseComplete()` stops filter chain propagation.

---

*Document generated from official PyQGIS Developer Cookbook (QGIS 3.x) and QGIS C++ API Reference. Items marked [VERIFY] require additional confirmation against source code or newer documentation.*
