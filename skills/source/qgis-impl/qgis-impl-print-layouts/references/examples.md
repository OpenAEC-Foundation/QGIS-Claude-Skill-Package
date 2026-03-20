# Working Code Examples (QGIS Print Layouts)

## Example 1: Minimal Map Export to PDF

Creates a layout with a single map item and exports to PDF.

```python
from qgis.core import (
    QgsPrintLayout, QgsProject, QgsLayoutItemMap,
    QgsLayoutExporter, QgsLayoutSize, QgsLayoutPoint,
    QgsUnitTypes
)

project = QgsProject.instance()
layer = project.mapLayersByName("my_layer")[0]

# Create layout
layout = QgsPrintLayout(project)
layout.initializeDefaults()
layout.setName("Quick Export")

# Add map item with extent
map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(277, 190, QgsUnitTypes.LayoutMillimeters))
map_item.attemptMove(QgsLayoutPoint(10, 10, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(layer.extent())
layout.addLayoutItem(map_item)

# Export
exporter = QgsLayoutExporter(layout)
settings = QgsLayoutExporter.PdfExportSettings()
settings.dpi = 300
result = exporter.exportToPdf("/tmp/map_output.pdf", settings)
assert result == QgsLayoutExporter.Success, f"Export failed: {result}"
```

---

## Example 2: Full Map Composition (Title, Legend, Scale Bar, North Arrow)

```python
from qgis.core import (
    QgsPrintLayout, QgsProject, QgsLayoutItemMap,
    QgsLayoutItemLabel, QgsLayoutItemLegend,
    QgsLayoutItemScaleBar, QgsLayoutItemPicture,
    QgsLayoutExporter, QgsLayoutSize, QgsLayoutPoint,
    QgsUnitTypes
)
from qgis.PyQt.QtGui import QFont, QColor

project = QgsProject.instance()
layer = project.mapLayersByName("parcels")[0]

# Create layout
layout = QgsPrintLayout(project)
layout.initializeDefaults()
layout.setName("Full Composition")
project.layoutManager().addLayout(layout)

# Map item (main area)
map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(180, 160, QgsUnitTypes.LayoutMillimeters))
map_item.attemptMove(QgsLayoutPoint(10, 25, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(layer.extent())
layout.addLayoutItem(map_item)

# Title
title = QgsLayoutItemLabel(layout)
title.setText("Parcel Overview Map")
title.setFont(QFont("Arial", 18))
title.setFontColor(QColor(0, 0, 0))
title.adjustSizeToText()
title.attemptMove(QgsLayoutPoint(10, 5, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(title)

# Legend
legend = QgsLayoutItemLegend(layout)
legend.setLinkedMap(map_item)
legend.setTitle("Legend")
legend.attemptMove(QgsLayoutPoint(200, 25, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(legend)

# Scale bar
scalebar = QgsLayoutItemScaleBar(layout)
scalebar.setLinkedMap(map_item)
scalebar.setStyle("Double Box")
scalebar.applyDefaultSize()
scalebar.attemptMove(QgsLayoutPoint(10, 190, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(scalebar)

# North arrow
north_arrow = QgsLayoutItemPicture(layout)
north_arrow.setPicturePath("/path/to/north_arrow.svg")
north_arrow.attemptResize(QgsLayoutSize(15, 15, QgsUnitTypes.LayoutMillimeters))
north_arrow.attemptMove(QgsLayoutPoint(265, 25, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(north_arrow)

# Export to PDF
exporter = QgsLayoutExporter(layout)
settings = QgsLayoutExporter.PdfExportSettings()
settings.dpi = 300
result = exporter.exportToPdf("/tmp/full_composition.pdf", settings)
assert result == QgsLayoutExporter.Success
```

---

## Example 3: Atlas Export (Separate PDFs per Feature)

```python
from qgis.core import (
    QgsPrintLayout, QgsProject, QgsLayoutItemMap,
    QgsLayoutItemLabel, QgsLayoutExporter, QgsLayoutSize,
    QgsLayoutPoint, QgsUnitTypes
)

project = QgsProject.instance()
coverage = project.mapLayersByName("municipalities")[0]

# Create layout
layout = QgsPrintLayout(project)
layout.initializeDefaults()
layout.setName("Atlas Layout")
project.layoutManager().addLayout(layout)

# Map item -- atlas will control the extent
map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(260, 180, QgsUnitTypes.LayoutMillimeters))
map_item.attemptMove(QgsLayoutPoint(10, 20, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(coverage.extent())
layout.addLayoutItem(map_item)

# Dynamic title using atlas expression
title = QgsLayoutItemLabel(layout)
title.setText("[% @atlas_featurenumber %] - [% \"name\" %]")
title.adjustSizeToText()
title.attemptMove(QgsLayoutPoint(10, 5, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(title)

# Configure atlas
atlas = layout.atlas()
atlas.setCoverageLayer(coverage)
atlas.setEnabled(True)
atlas.setFilenameExpression("\"name\" || '_map'")

# Set map to follow atlas feature
map_item.setAtlasDriven(True)
map_item.setAtlasScalingMode(QgsLayoutItemMap.Auto)
map_item.setAtlasMargin(0.10)  # 10% margin around feature

# Export all atlas pages to separate PDFs
pdf_settings = QgsLayoutExporter.PdfExportSettings()
pdf_settings.dpi = 300
result = QgsLayoutExporter.exportToPdfs(
    atlas, "/tmp/atlas_output/", pdf_settings
)
assert result == QgsLayoutExporter.Success
```

---

## Example 4: Custom Page Size (A3 Landscape)

```python
from qgis.core import (
    QgsPrintLayout, QgsProject, QgsLayoutSize, QgsUnitTypes
)

project = QgsProject.instance()
layout = QgsPrintLayout(project)
layout.initializeDefaults()
layout.setName("A3 Landscape")

# Change default page to A3 landscape
page = layout.pageCollection().page(0)
page.setPageSize(QgsLayoutSize(420, 297, QgsUnitTypes.LayoutMillimeters))

project.layoutManager().addLayout(layout)
```

---

## Example 5: Layout with Attribute Table

```python
from qgis.core import (
    QgsPrintLayout, QgsProject, QgsLayoutItemMap,
    QgsLayoutItemAttributeTable, QgsLayoutFrame,
    QgsLayoutExporter, QgsLayoutSize, QgsLayoutPoint,
    QgsUnitTypes
)

project = QgsProject.instance()
layer = project.mapLayersByName("sites")[0]

layout = QgsPrintLayout(project)
layout.initializeDefaults()
layout.setName("Map with Table")
project.layoutManager().addLayout(layout)

# Map item (top half)
map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(277, 100, QgsUnitTypes.LayoutMillimeters))
map_item.attemptMove(QgsLayoutPoint(10, 10, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(layer.extent())
layout.addLayoutItem(map_item)

# Attribute table (bottom half)
table = QgsLayoutItemAttributeTable.create(layout)
table.setVectorLayer(layer)
table.setMaximumNumberOfFeatures(15)

frame = QgsLayoutFrame(layout, table)
frame.attemptResize(QgsLayoutSize(277, 70, QgsUnitTypes.LayoutMillimeters))
frame.attemptMove(QgsLayoutPoint(10, 120, QgsUnitTypes.LayoutMillimeters))
table.addFrame(frame)
layout.addMultiFrame(table)

# Export
exporter = QgsLayoutExporter(layout)
settings = QgsLayoutExporter.PdfExportSettings()
settings.dpi = 300
result = exporter.exportToPdf("/tmp/map_with_table.pdf", settings)
assert result == QgsLayoutExporter.Success
```

---

## Example 6: Save and Load Layout Template

```python
from qgis.core import QgsPrintLayout, QgsProject, QgsReadWriteContext
from qgis.PyQt.QtXml import QDomDocument

project = QgsProject.instance()

# --- Save existing layout as template ---
existing_layout = project.layoutManager().layoutByName("My Layout")
doc = QDomDocument()
existing_layout.writeXml(doc, QgsReadWriteContext())
with open("/tmp/my_template.qpt", "w") as f:
    f.write(doc.toString())

# --- Load template into new layout ---
with open("/tmp/my_template.qpt", "r") as f:
    content = f.read()

doc = QDomDocument()
doc.setContent(content)

new_layout = QgsPrintLayout(project)
new_layout.readXml(doc.documentElement(), doc, QgsReadWriteContext())
new_layout.setName("Loaded From Template")
project.layoutManager().addLayout(new_layout)
```

---

## Example 7: Multi-Page Layout

```python
from qgis.core import (
    QgsPrintLayout, QgsProject, QgsLayoutItemMap,
    QgsLayoutItemLabel, QgsLayoutItemPage,
    QgsLayoutSize, QgsLayoutPoint, QgsUnitTypes
)

project = QgsProject.instance()
layout = QgsPrintLayout(project)
layout.initializeDefaults()
layout.setName("Multi-Page Report")

# Page 1: Overview map
map_overview = QgsLayoutItemMap(layout)
map_overview.attemptResize(QgsLayoutSize(277, 190, QgsUnitTypes.LayoutMillimeters))
map_overview.attemptMove(QgsLayoutPoint(10, 10, QgsUnitTypes.LayoutMillimeters))
map_overview.zoomToExtent(project.mapLayersByName("region")[0].extent())
layout.addLayoutItem(map_overview)

# Add second page
page2 = QgsLayoutItemPage(layout)
page2.setPageSize(QgsLayoutSize(297, 210, QgsUnitTypes.LayoutMillimeters))
layout.pageCollection().addPage(page2)

# Page 2: Detail map (items on page 2 use y-offset = page1_height + gap)
page1_height = 210  # A4 portrait height in mm
detail_label = QgsLayoutItemLabel(layout)
detail_label.setText("Detail View")
detail_label.adjustSizeToText()
detail_label.attemptMove(QgsLayoutPoint(10, page1_height + 5, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(detail_label)

map_detail = QgsLayoutItemMap(layout)
map_detail.attemptResize(QgsLayoutSize(277, 180, QgsUnitTypes.LayoutMillimeters))
map_detail.attemptMove(QgsLayoutPoint(10, page1_height + 20, QgsUnitTypes.LayoutMillimeters))
map_detail.zoomToExtent(project.mapLayersByName("detail_area")[0].extent())
layout.addLayoutItem(map_detail)

project.layoutManager().addLayout(layout)
```

---

## Example 8: Export to Multiple Formats

```python
from qgis.core import QgsLayoutExporter

exporter = QgsLayoutExporter(layout)

# PDF (vector, print-ready)
pdf_settings = QgsLayoutExporter.PdfExportSettings()
pdf_settings.dpi = 300
pdf_settings.forceVectorOutput = True
result = exporter.exportToPdf("/tmp/output.pdf", pdf_settings)
assert result == QgsLayoutExporter.Success

# PNG (raster, web use)
img_settings = QgsLayoutExporter.ImageExportSettings()
img_settings.dpi = 150
result = exporter.exportToImage("/tmp/output.png", img_settings)
assert result == QgsLayoutExporter.Success

# SVG (vector, editable)
svg_settings = QgsLayoutExporter.SvgExportSettings()
svg_settings.dpi = 300
svg_settings.forceVectorOutput = True
result = exporter.exportToSvg("/tmp/output.svg", svg_settings)
assert result == QgsLayoutExporter.Success
```

---

## Example 9: Map with Grid Overlay

```python
from qgis.core import (
    QgsPrintLayout, QgsProject, QgsLayoutItemMap,
    QgsLayoutItemMapGrid, QgsLayoutSize, QgsLayoutPoint,
    QgsUnitTypes, QgsCoordinateReferenceSystem
)

project = QgsProject.instance()
layout = QgsPrintLayout(project)
layout.initializeDefaults()
layout.setName("Grid Map")

map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(250, 180, QgsUnitTypes.LayoutMillimeters))
map_item.attemptMove(QgsLayoutPoint(10, 10, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(project.mapLayersByName("my_layer")[0].extent())
layout.addLayoutItem(map_item)

# Add coordinate grid
grid = QgsLayoutItemMapGrid("Coordinate Grid", map_item)
grid.setIntervalX(1000)  # Grid line every 1000 map units
grid.setIntervalY(1000)
grid.setAnnotationEnabled(True)
grid.setFrameStyle(QgsLayoutItemMapGrid.Zebra)
grid.setCrs(QgsCoordinateReferenceSystem("EPSG:28992"))
map_item.grids().addGrid(grid)

project.layoutManager().addLayout(layout)
```
