# qgis-agents-map-generator — Examples

## Example 1: Choropleth Map from GeoPackage

Complete pipeline generating a choropleth (graduated color) map of population density from a GeoPackage, exported to PDF.

```python
from qgis.core import (
    QgsApplication, QgsProject, QgsVectorLayer,
    QgsCoordinateReferenceSystem, QgsGraduatedSymbolRenderer,
    QgsRendererRange, QgsClassificationRange, QgsFillSymbol,
    QgsGradientColorRamp, QgsPalLayerSettings,
    QgsVectorLayerSimpleLabeling, QgsTextFormat,
    QgsPrintLayout, QgsLayoutItemMap, QgsLayoutItemLabel,
    QgsLayoutItemLegend, QgsLayoutItemScaleBar,
    QgsLayoutSize, QgsLayoutPoint, QgsUnitTypes,
    QgsLayoutExporter
)
from qgis.PyQt.QtGui import QColor, QFont

project = QgsProject.instance()

# Step 1: Load data
layer = QgsVectorLayer("/data/municipalities.gpkg|layername=municipalities",
                       "Municipalities", "ogr")
assert layer.isValid(), "Layer failed to load"
project.addMapLayer(layer)

# Step 2: Set projected CRS
project.setCrs(QgsCoordinateReferenceSystem("EPSG:28992"))

# Step 3: Graduated symbology for population density
renderer = QgsGraduatedSymbolRenderer()
renderer.setClassAttribute('pop_density')

ramp = QgsGradientColorRamp(QColor('#ffffcc'), QColor('#800026'))

ranges = [
    ("Low", 0, 500, '#ffffcc'),
    ("Medium-Low", 500, 1000, '#fed976'),
    ("Medium", 1000, 2000, '#fd8d3c'),
    ("Medium-High", 2000, 5000, '#e31a1c'),
    ("High", 5000, 50000, '#800026'),
]

for label, lower, upper, color in ranges:
    symbol = QgsFillSymbol.createSimple({
        'color': color,
        'outline_color': '#333333',
        'outline_width': '0.2'
    })
    renderer.addClassRange(
        QgsRendererRange(QgsClassificationRange(label, lower, upper), symbol)
    )

layer.setRenderer(renderer)
layer.triggerRepaint()

# Step 4: Labels
settings = QgsPalLayerSettings()
settings.fieldName = 'name'
settings.enabled = True

text_format = QgsTextFormat()
text_format.setSize(8)
text_format.setColor(QColor('black'))
buffer_settings = text_format.buffer()
buffer_settings.setEnabled(True)
buffer_settings.setSize(0.8)
buffer_settings.setColor(QColor('white'))
settings.setFormat(text_format)

layer.setLabeling(QgsVectorLayerSimpleLabeling(settings))
layer.setLabelsEnabled(True)
layer.triggerRepaint()

# Step 5: Layout
layout = QgsPrintLayout(project)
layout.initializeDefaults()
layout.setName("Population Density Map")

map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(180, 160, QgsUnitTypes.LayoutMillimeters))
map_item.attemptMove(QgsLayoutPoint(15, 25, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(layer.extent())
layout.addLayoutItem(map_item)

# Step 6: Cartographic elements
title = QgsLayoutItemLabel(layout)
title.setText("Population Density by Municipality")
title.setFont(QFont("Arial", 16))
title.adjustSizeToText()
title.attemptMove(QgsLayoutPoint(15, 8, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(title)

legend = QgsLayoutItemLegend(layout)
legend.setLinkedMap(map_item)
legend.setTitle("Density (per km²)")
legend.attemptMove(QgsLayoutPoint(15, 195, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(legend)

scalebar = QgsLayoutItemScaleBar(layout)
scalebar.setStyle("Single Box")
scalebar.setLinkedMap(map_item)
scalebar.applyDefaultSize()
scalebar.attemptMove(QgsLayoutPoint(100, 195, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(scalebar)

source_label = QgsLayoutItemLabel(layout)
source_label.setText("Source: CBS Statistics Netherlands, 2024")
source_label.adjustSizeToText()
source_label.attemptMove(QgsLayoutPoint(15, 285, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(source_label)

project.layoutManager().addLayout(layout)

# Step 7: Export
exporter = QgsLayoutExporter(layout)
pdf_settings = QgsLayoutExporter.PdfExportSettings()
pdf_settings.dpi = 300
result = exporter.exportToPdf("/output/population_density.pdf", pdf_settings)
assert result == QgsLayoutExporter.ExportResult.Success
```

---

## Example 2: Categorized Land Use Map

Map with categorical symbology showing distinct land use types.

```python
from qgis.core import (
    QgsProject, QgsVectorLayer, QgsCoordinateReferenceSystem,
    QgsCategorizedSymbolRenderer, QgsRendererCategory, QgsFillSymbol,
    QgsPrintLayout, QgsLayoutItemMap, QgsLayoutItemLabel,
    QgsLayoutItemLegend, QgsLayoutItemScaleBar, QgsLayoutItemPicture,
    QgsLayoutSize, QgsLayoutPoint, QgsUnitTypes, QgsLayoutExporter
)
from qgis.PyQt.QtGui import QFont

project = QgsProject.instance()

layer = QgsVectorLayer("/data/landuse.shp", "Land Use", "ogr")
assert layer.isValid()
project.addMapLayer(layer)
project.setCrs(QgsCoordinateReferenceSystem("EPSG:32632"))

# Categorized renderer with qualitative colors
categories = [
    ('residential', '#e41a1c', 'Residential'),
    ('commercial', '#377eb8', 'Commercial'),
    ('industrial', '#984ea3', 'Industrial'),
    ('agricultural', '#4daf4a', 'Agricultural'),
    ('forest', '#006d2c', 'Forest'),
    ('water', '#74add1', 'Water'),
    ('recreation', '#ff7f00', 'Recreation'),
]

renderer = QgsCategorizedSymbolRenderer()
renderer.setClassAttribute('landuse_type')

for value, color, label in categories:
    symbol = QgsFillSymbol.createSimple({
        'color': color,
        'outline_color': '#333333',
        'outline_width': '0.15'
    })
    renderer.addCategory(QgsRendererCategory(value, symbol, label))

layer.setRenderer(renderer)
layer.triggerRepaint()

# Layout with all cartographic elements
layout = QgsPrintLayout(project)
layout.initializeDefaults()
layout.setName("Land Use Map")

# A3 Landscape
page = layout.pageCollection().page(0)
page.setPageSize(QgsLayoutSize(420, 297, QgsUnitTypes.LayoutMillimeters))

map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(380, 230, QgsUnitTypes.LayoutMillimeters))
map_item.attemptMove(QgsLayoutPoint(20, 30, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(layer.extent())
layout.addLayoutItem(map_item)

title = QgsLayoutItemLabel(layout)
title.setText("Land Use Classification")
title.setFont(QFont("Arial", 20))
title.adjustSizeToText()
title.attemptMove(QgsLayoutPoint(20, 8, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(title)

legend = QgsLayoutItemLegend(layout)
legend.setLinkedMap(map_item)
legend.attemptMove(QgsLayoutPoint(20, 265, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(legend)

scalebar = QgsLayoutItemScaleBar(layout)
scalebar.setStyle("Double Box")
scalebar.setLinkedMap(map_item)
scalebar.applyDefaultSize()
scalebar.attemptMove(QgsLayoutPoint(200, 265, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(scalebar)

north_arrow = QgsLayoutItemPicture(layout)
north_arrow.setPicturePath(":/images/sketchy/sketchy-north.svg")
north_arrow.attemptResize(QgsLayoutSize(15, 15, QgsUnitTypes.LayoutMillimeters))
north_arrow.attemptMove(QgsLayoutPoint(390, 30, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(north_arrow)

project.layoutManager().addLayout(layout)

exporter = QgsLayoutExporter(layout)
pdf_settings = QgsLayoutExporter.PdfExportSettings()
pdf_settings.dpi = 300
result = exporter.exportToPdf("/output/landuse_map.pdf", pdf_settings)
assert result == QgsLayoutExporter.ExportResult.Success
```

---

## Example 3: Atlas — One Map Per District

Generates a separate PDF for each district using atlas functionality.

```python
from qgis.core import (
    QgsProject, QgsVectorLayer, QgsCoordinateReferenceSystem,
    QgsFillSymbol, QgsSingleSymbolRenderer,
    QgsPrintLayout, QgsLayoutItemMap, QgsLayoutItemLabel,
    QgsLayoutItemLegend, QgsLayoutItemScaleBar,
    QgsLayoutSize, QgsLayoutPoint, QgsUnitTypes, QgsLayoutExporter
)

project = QgsProject.instance()

# Load district boundaries (coverage layer) and points of interest
districts = QgsVectorLayer("/data/districts.gpkg|layername=districts", "Districts", "ogr")
pois = QgsVectorLayer("/data/pois.gpkg|layername=pois", "Points of Interest", "ogr")
assert districts.isValid() and pois.isValid()
project.addMapLayer(districts)
project.addMapLayer(pois)
project.setCrs(QgsCoordinateReferenceSystem("EPSG:28992"))

# Style the layers
districts.setRenderer(QgsSingleSymbolRenderer(
    QgsFillSymbol.createSimple({
        'color': '#f0f0f0',
        'outline_color': '#333333',
        'outline_width': '0.5'
    })
))
districts.triggerRepaint()

# Create atlas layout
layout = QgsPrintLayout(project)
layout.initializeDefaults()
layout.setName("District Atlas")

map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(180, 160, QgsUnitTypes.LayoutMillimeters))
map_item.attemptMove(QgsLayoutPoint(15, 30, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(districts.extent())
layout.addLayoutItem(map_item)

# Atlas-driven title using expression
title = QgsLayoutItemLabel(layout)
title.setText('[% "district_name" %]')  # Expression replaced per atlas feature
title.adjustSizeToText()
title.attemptMove(QgsLayoutPoint(15, 8, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(title)

# Page number label
page_label = QgsLayoutItemLabel(layout)
page_label.setText('Page [% @atlas_featurenumber %] of [% @atlas_totalfeatures %]')
page_label.adjustSizeToText()
page_label.attemptMove(QgsLayoutPoint(15, 285, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(page_label)

scalebar = QgsLayoutItemScaleBar(layout)
scalebar.setStyle("Single Box")
scalebar.setLinkedMap(map_item)
scalebar.applyDefaultSize()
scalebar.attemptMove(QgsLayoutPoint(100, 200, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(scalebar)

# Configure atlas
atlas = layout.atlas()
atlas.setCoverageLayer(districts)
atlas.setEnabled(True)
atlas.setFilenameExpression("'district_' || \"district_name\"")

# Map follows atlas feature
map_item.setAtlasDriven(True)
map_item.setAtlasScalingMode(QgsLayoutItemMap.AtlasScalingMode.Auto)

project.layoutManager().addLayout(layout)

# Export individual PDFs per district
pdf_settings = QgsLayoutExporter.PdfExportSettings()
pdf_settings.dpi = 300
result = QgsLayoutExporter.exportToPdfs(atlas, "/output/atlas/", pdf_settings)
assert result == QgsLayoutExporter.ExportResult.Success
```

---

## Example 4: Rule-Based Symbology with Multiple Criteria

Map using rule-based renderer for complex classification logic.

```python
from qgis.core import (
    QgsProject, QgsVectorLayer, QgsRuleBasedRenderer,
    QgsFillSymbol, QgsCoordinateReferenceSystem
)

project = QgsProject.instance()
layer = QgsVectorLayer("/data/buildings.gpkg", "Buildings", "ogr")
assert layer.isValid()
project.addMapLayer(layer)

# Root rule (catch-all)
root_rule = QgsRuleBasedRenderer.Rule(
    QgsFillSymbol.createSimple({'color': '#cccccc', 'outline_color': 'black', 'outline_width': '0.1'})
)

# Rule 1: Large residential buildings
rule1 = QgsRuleBasedRenderer.Rule(
    QgsFillSymbol.createSimple({'color': '#e41a1c', 'outline_color': 'black', 'outline_width': '0.2'}),
    filterExp='"use" = \'residential\' AND "area_m2" > 200',
    label='Large Residential'
)

# Rule 2: Small residential buildings
rule2 = QgsRuleBasedRenderer.Rule(
    QgsFillSymbol.createSimple({'color': '#fb9a99', 'outline_color': 'black', 'outline_width': '0.2'}),
    filterExp='"use" = \'residential\' AND "area_m2" <= 200',
    label='Small Residential'
)

# Rule 3: Commercial buildings
rule3 = QgsRuleBasedRenderer.Rule(
    QgsFillSymbol.createSimple({'color': '#377eb8', 'outline_color': 'black', 'outline_width': '0.2'}),
    filterExp='"use" = \'commercial\'',
    label='Commercial'
)

# Rule 4: Industrial buildings
rule4 = QgsRuleBasedRenderer.Rule(
    QgsFillSymbol.createSimple({'color': '#984ea3', 'outline_color': 'black', 'outline_width': '0.2'}),
    filterExp='"use" = \'industrial\'',
    label='Industrial'
)

root_rule.appendChild(rule1)
root_rule.appendChild(rule2)
root_rule.appendChild(rule3)
root_rule.appendChild(rule4)

renderer = QgsRuleBasedRenderer(root_rule)
layer.setRenderer(renderer)
layer.triggerRepaint()
```

---

## Example 5: Multi-Layer Map with Raster Background

Combining raster (DEM hillshade) with vector overlays.

```python
from qgis.core import (
    QgsProject, QgsVectorLayer, QgsRasterLayer,
    QgsCoordinateReferenceSystem, QgsSingleSymbolRenderer,
    QgsLineSymbol, QgsMarkerSymbol, QgsFillSymbol,
    QgsPrintLayout, QgsLayoutItemMap, QgsLayoutItemLabel,
    QgsLayoutItemLegend, QgsLayoutSize, QgsLayoutPoint,
    QgsUnitTypes, QgsLayoutExporter
)
from qgis.PyQt.QtGui import QFont

project = QgsProject.instance()

# Load raster background
hillshade = QgsRasterLayer("/data/hillshade.tif", "Hillshade")
assert hillshade.isValid()
project.addMapLayer(hillshade)

# Load vector layers
roads = QgsVectorLayer("/data/roads.shp", "Roads", "ogr")
cities = QgsVectorLayer("/data/cities.shp", "Cities", "ogr")
boundaries = QgsVectorLayer("/data/boundaries.shp", "Boundaries", "ogr")
assert roads.isValid() and cities.isValid() and boundaries.isValid()

project.addMapLayer(boundaries)
project.addMapLayer(roads)
project.addMapLayer(cities)

project.setCrs(QgsCoordinateReferenceSystem("EPSG:32632"))

# Style layers — order matters: hillshade at bottom, cities on top
boundaries.setRenderer(QgsSingleSymbolRenderer(
    QgsFillSymbol.createSimple({
        'color': '0,0,0,0',  # Transparent fill
        'outline_color': '#333333',
        'outline_width': '0.8'
    })
))

roads.setRenderer(QgsSingleSymbolRenderer(
    QgsLineSymbol.createSimple({'color': '#e31a1c', 'width': '0.4'})
))

cities.setRenderer(QgsSingleSymbolRenderer(
    QgsMarkerSymbol.createSimple({'name': 'circle', 'color': '#1f78b4', 'size': '3'})
))

for lyr in [boundaries, roads, cities]:
    lyr.triggerRepaint()

# Create layout and export
layout = QgsPrintLayout(project)
layout.initializeDefaults()
layout.setName("Terrain Overview")

map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(180, 150, QgsUnitTypes.LayoutMillimeters))
map_item.attemptMove(QgsLayoutPoint(15, 30, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(boundaries.extent())
layout.addLayoutItem(map_item)

title = QgsLayoutItemLabel(layout)
title.setText("Terrain Overview with Infrastructure")
title.setFont(QFont("Arial", 14))
title.adjustSizeToText()
title.attemptMove(QgsLayoutPoint(15, 10, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(title)

legend = QgsLayoutItemLegend(layout)
legend.setLinkedMap(map_item)
legend.attemptMove(QgsLayoutPoint(15, 190, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(legend)

project.layoutManager().addLayout(layout)

exporter = QgsLayoutExporter(layout)
img_settings = QgsLayoutExporter.ImageExportSettings()
img_settings.dpi = 150
result = exporter.exportToImage("/output/terrain_overview.png", img_settings)
assert result == QgsLayoutExporter.ExportResult.Success
```

---

## Example 6: Template-Based Map Generation

Loading a pre-made layout template and populating it with new data.

```python
from qgis.core import (
    QgsProject, QgsVectorLayer, QgsPrintLayout,
    QgsReadWriteContext, QgsLayoutExporter
)
from qgis.PyQt.QtXml import QDomDocument

project = QgsProject.instance()

# Load data
layer = QgsVectorLayer("/data/new_dataset.gpkg", "New Data", "ogr")
assert layer.isValid()
project.addMapLayer(layer)

# Load template
with open("/templates/standard_a4.qpt", "r") as f:
    template_content = f.read()

doc = QDomDocument()
doc.setContent(template_content)

layout = QgsPrintLayout(project)
layout.readXml(doc.documentElement(), doc, QgsReadWriteContext())
layout.setName("Report from Template")

# Update map extent to new data
for item in layout.items():
    from qgis.core import QgsLayoutItemMap
    if isinstance(item, QgsLayoutItemMap):
        item.zoomToExtent(layer.extent())
        break

project.layoutManager().addLayout(layout)

# Export
exporter = QgsLayoutExporter(layout)
pdf_settings = QgsLayoutExporter.PdfExportSettings()
pdf_settings.dpi = 300
result = exporter.exportToPdf("/output/report_from_template.pdf", pdf_settings)
assert result == QgsLayoutExporter.ExportResult.Success
```
