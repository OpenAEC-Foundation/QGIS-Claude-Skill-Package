---
name: qgis-impl-print-layouts
description: >
  Use when creating print layouts, exporting maps to PDF/SVG/image, or generating atlas series.
  Prevents empty map exports from missing extent configuration and atlas misconfiguration.
  Covers QgsPrintLayout, layout items (map, label, legend, scale bar), atlas generation, and PDF/SVG/image export.
  Keywords: print layout, QgsPrintLayout, map export, PDF, atlas, legend, scale bar, QgsLayoutExporter, map composition, export to PDF, print map, create atlas, add legend.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-impl-print-layouts

## Quick Reference

### Layout Creation Workflow

| Step | Action | Key Class |
|------|--------|-----------|
| 1. Create layout | `QgsPrintLayout(project)` + `initializeDefaults()` | `QgsPrintLayout` |
| 2. Register layout | `layoutManager().addLayout(layout)` | `QgsLayoutManager` |
| 3. Add map item | `QgsLayoutItemMap(layout)` + set extent | `QgsLayoutItemMap` |
| 4. Add decorations | Labels, legend, scale bar, north arrow | Various `QgsLayoutItem*` |
| 5. Configure export | Set DPI, page size | `QgsLayoutExporter.*ExportSettings` |
| 6. Export | `exportToPdf()`, `exportToImage()`, `exportToSvg()` | `QgsLayoutExporter` |

### Layout Item Types

| Class | Purpose | Key Methods |
|-------|---------|-------------|
| `QgsLayoutItemMap` | Map display | `zoomToExtent()`, `setScale()`, `setCrs()` |
| `QgsLayoutItemLabel` | Text labels | `setText()`, `adjustSizeToText()` |
| `QgsLayoutItemLegend` | Map legend | `setLinkedMap()`, `setAutoUpdateModel()` |
| `QgsLayoutItemScaleBar` | Scale indicator | `setLinkedMap()`, `setStyle()`, `applyDefaultSize()` |
| `QgsLayoutItemPicture` | Images/north arrows | `setPicturePath()` |
| `QgsLayoutItemAttributeTable` | Data tables | `setVectorLayer()`, `setMaximumNumberOfFeatures()` |
| `QgsLayoutItemPolygon` | Shape decorations | `setSymbol()` |

### Export Result Codes

| Code | Constant | Meaning |
|------|----------|---------|
| 0 | `Success` | Export completed |
| 1 | `Canceled` | Export was canceled |
| 2 | `MemoryError` | Insufficient memory |
| 3 | `FileError` | Cannot write to file path |
| 4 | `PrintError` | Printing subsystem error |
| 5 | `SvgLayerError` | SVG layer export failed |
| 6 | `IteratorError` | Atlas iterator failed |

### Critical Warnings

**NEVER** export a layout without setting the map item extent first -- the export produces a blank or incorrect map. ALWAYS call `zoomToExtent()` or `setScale()` on every `QgsLayoutItemMap` before exporting.

**NEVER** skip `initializeDefaults()` after creating a new `QgsPrintLayout` -- without it the layout has no pages and export fails silently.

**NEVER** forget to call `layout.addLayoutItem(item)` after creating a layout item -- items created without being added to the layout do not appear in exports.

**ALWAYS** register the layout with `layoutManager().addLayout(layout)` if you need it to persist in the project file.

**ALWAYS** check the export result code -- a return value other than `QgsLayoutExporter.Success` (0) indicates export failure.

**NEVER** call `atlas.next()` without calling `atlas.beginRender()` first -- the atlas iterator is not initialized until `beginRender()` is called.

---

## Decision Tree: Which Export Method?

```
Need to export a map?
├── Single layout → use QgsLayoutExporter instance methods
│   ├── Need vector output? → exportToSvg()
│   ├── Need raster image? → exportToImage() (PNG, JPEG, TIFF)
│   └── Need print-ready document? → exportToPdf()
├── Multiple pages per feature? → Atlas generation
│   ├── Separate files per feature? → QgsLayoutExporter.exportToPdfs() (static)
│   └── Single multi-page PDF? → QgsLayoutExporter.exportToPdf() (static, with atlas)
└── Programmatic batch? → Loop with atlas.beginRender() / atlas.next() / atlas.endRender()
```

---

## Essential Patterns

### Pattern 1: Complete Layout Creation

```python
from qgis.core import (
    QgsPrintLayout, QgsProject, QgsLayoutItemMap,
    QgsLayoutItemLabel, QgsLayoutItemLegend,
    QgsLayoutItemScaleBar, QgsLayoutSize,
    QgsLayoutPoint, QgsUnitTypes
)

project = QgsProject.instance()

# Step 1: Create and initialize layout
layout = QgsPrintLayout(project)
layout.initializeDefaults()  # REQUIRED: creates default A4 page
layout.setName("My Map Layout")

# Step 2: Register with project
manager = project.layoutManager()
manager.addLayout(layout)

# Step 3: Add map item -- ALWAYS set extent
map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(200, 150, QgsUnitTypes.LayoutMillimeters))
map_item.attemptMove(QgsLayoutPoint(10, 10, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(project.mapLayersByName("my_layer")[0].extent())
layout.addLayoutItem(map_item)

# Step 4: Add title label
title = QgsLayoutItemLabel(layout)
title.setText("Map Title")
title.adjustSizeToText()
title.attemptMove(QgsLayoutPoint(10, 5, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(title)

# Step 5: Add legend linked to map
legend = QgsLayoutItemLegend(layout)
legend.setLinkedMap(map_item)
legend.attemptMove(QgsLayoutPoint(220, 10, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(legend)

# Step 6: Add scale bar linked to map
scalebar = QgsLayoutItemScaleBar(layout)
scalebar.setLinkedMap(map_item)
scalebar.setStyle("Single Box")
scalebar.applyDefaultSize()
scalebar.attemptMove(QgsLayoutPoint(10, 170, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(scalebar)
```

### Pattern 2: Export to PDF

```python
from qgis.core import QgsLayoutExporter

exporter = QgsLayoutExporter(layout)

pdf_settings = QgsLayoutExporter.PdfExportSettings()
pdf_settings.dpi = 300

result = exporter.exportToPdf("/path/to/output.pdf", pdf_settings)
if result != QgsLayoutExporter.Success:
    raise RuntimeError(f"PDF export failed with code: {result}")
```

### Pattern 3: Export to Image (PNG/JPEG/TIFF)

```python
exporter = QgsLayoutExporter(layout)

img_settings = QgsLayoutExporter.ImageExportSettings()
img_settings.dpi = 300

result = exporter.exportToImage("/path/to/output.png", img_settings)
if result != QgsLayoutExporter.Success:
    raise RuntimeError(f"Image export failed with code: {result}")
```

### Pattern 4: Export to SVG

```python
exporter = QgsLayoutExporter(layout)

svg_settings = QgsLayoutExporter.SvgExportSettings()
svg_settings.dpi = 300

result = exporter.exportToSvg("/path/to/output.svg", svg_settings)
if result != QgsLayoutExporter.Success:
    raise RuntimeError(f"SVG export failed with code: {result}")
```

### Pattern 5: Atlas Generation

```python
atlas = layout.atlas()
atlas.setCoverageLayer(coverage_layer)
atlas.setEnabled(True)
atlas.setFilenameExpression("'output_' || @atlas_featurenumber")

# Export each atlas page to separate PDFs
pdf_settings = QgsLayoutExporter.PdfExportSettings()
pdf_settings.dpi = 300
result = QgsLayoutExporter.exportToPdfs(
    atlas, "/path/to/atlas_output/", pdf_settings
)
if result != QgsLayoutExporter.Success:
    raise RuntimeError(f"Atlas export failed with code: {result}")
```

### Pattern 6: Atlas with Feature Filtering

```python
atlas = layout.atlas()
atlas.setCoverageLayer(coverage_layer)
atlas.setEnabled(True)
atlas.setFilterFeatures(True)
atlas.setFilterExpression("\"type\" = 'residential'")
atlas.setFilenameExpression("\"name\" || '_map'")
```

### Pattern 7: Manual Atlas Iteration

```python
atlas = layout.atlas()
atlas.setCoverageLayer(coverage_layer)
atlas.setEnabled(True)

exporter = QgsLayoutExporter(layout)
pdf_settings = QgsLayoutExporter.PdfExportSettings()
pdf_settings.dpi = 300

atlas.beginRender()  # REQUIRED before iterating
while atlas.next():
    feature_num = atlas.currentFeatureNumber()
    page_name = atlas.nameForPage(feature_num)
    output_path = f"/path/to/output_{page_name}.pdf"
    exporter.exportToPdf(output_path, pdf_settings)
atlas.endRender()  # ALWAYS call to clean up
```

---

## Common Operations

### Page Setup

```python
from qgis.core import QgsLayoutSize, QgsLayoutItemPage, QgsUnitTypes

# Change existing page size to A3 landscape
page = layout.pageCollection().page(0)
page.setPageSize(QgsLayoutSize(420, 297, QgsUnitTypes.LayoutMillimeters))

# Add a second page
new_page = QgsLayoutItemPage(layout)
new_page.setPageSize(QgsLayoutSize(297, 210, QgsUnitTypes.LayoutMillimeters))
layout.pageCollection().addPage(new_page)
```

### Map Item: Set Scale and CRS

```python
from qgis.core import QgsCoordinateReferenceSystem

map_item.setScale(50000)  # 1:50000
map_item.setCrs(QgsCoordinateReferenceSystem("EPSG:3857"))
```

### Map Item: Grid Overlay

```python
from qgis.core import QgsLayoutItemMapGrid

grid = QgsLayoutItemMapGrid("Main Grid", map_item)
grid.setIntervalX(1000)
grid.setIntervalY(1000)
grid.setAnnotationEnabled(True)
grid.setFrameStyle(QgsLayoutItemMapGrid.Zebra)
map_item.grids().addGrid(grid)
```

### Add North Arrow (Picture Item)

```python
from qgis.core import QgsLayoutItemPicture

picture = QgsLayoutItemPicture(layout)
picture.setPicturePath("/path/to/north_arrow.svg")
picture.attemptResize(QgsLayoutSize(20, 20, QgsUnitTypes.LayoutMillimeters))
picture.attemptMove(QgsLayoutPoint(250, 10, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(picture)
```

### Attribute Table in Layout

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

### Expression-Based Labels

```python
label = QgsLayoutItemLabel(layout)
# Use QGIS expressions inside [% %] delimiters
label.setText("[% @project_title %] - Scale 1:[% @map_scale %]")
label.attemptMove(QgsLayoutPoint(10, 5, QgsUnitTypes.LayoutMillimeters))
label.adjustSizeToText()
layout.addLayoutItem(label)
```

### Layout Manager Operations

```python
manager = QgsProject.instance().layoutManager()

# List all layouts
for l in manager.printLayouts():
    print(l.name())

# Get layout by name
my_layout = manager.layoutByName("My Layout")

# Remove layout
manager.removeLayout(layout)
```

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

### Shape Decorations (Polygon/Polyline)

```python
from qgis.core import QgsLayoutItemPolygon, QgsFillSymbol
from qgis.PyQt.QtCore import QPointF
from qgis.PyQt.QtGui import QPolygonF

polygon = QgsLayoutItemPolygon(
    QPolygonF([QPointF(0, 0), QPointF(100, 0), QPointF(100, 50), QPointF(0, 50)]),
    layout
)
props = {
    'color': '0,0,0,0',
    'style': 'no',
    'outline_color': 'black',
    'outline_width': '0.5'
}
symbol = QgsFillSymbol.createSimple(props)
polygon.setSymbol(symbol)
layout.addLayoutItem(polygon)
```

---

## Common Positioning Methods (All Layout Items)

| Method | Purpose |
|--------|---------|
| `attemptMove(QgsLayoutPoint)` | Set position on page |
| `attemptResize(QgsLayoutSize)` | Set item dimensions |
| `setFrameEnabled(bool)` | Toggle border frame |
| `setBackgroundEnabled(bool)` | Toggle background fill |
| `setLocked(bool)` | Prevent accidental edits |
| `setReferencePoint(point)` | Set anchor point for positioning |

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for QgsPrintLayout, QgsLayoutExporter, layout items
- [references/examples.md](references/examples.md) -- Complete working examples for layout creation and export
- [references/anti-patterns.md](references/anti-patterns.md) -- Common mistakes with layouts and exports

### Official Sources

- https://qgis.org/pyqgis/master/core/QgsPrintLayout.html
- https://qgis.org/pyqgis/master/core/QgsLayoutExporter.html
- https://qgis.org/pyqgis/master/core/QgsLayoutItemMap.html
- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/composer.html
