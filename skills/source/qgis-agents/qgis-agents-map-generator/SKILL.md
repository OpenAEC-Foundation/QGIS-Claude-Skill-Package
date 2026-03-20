---
name: qgis-agents-map-generator
description: >
  Use when generating complete maps from data: loading, styling, composing layouts, and exporting.
  Prevents poor cartographic choices and empty map exports.
  Covers the full map pipeline: data loading, symbology selection, labeling, layout composition, atlas generation, and export.
  Keywords: map generation, create map, export map, symbology, labeling, layout, atlas, cartography, map pipeline, PDF export.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-agents-map-generator

## Quick Reference

### Map Generation Pipeline

| Step | Action | Key Classes |
|------|--------|-------------|
| 1. Data Loading | Load vector/raster layers, validate | `QgsVectorLayer`, `QgsRasterLayer` |
| 2. CRS Setup | Set project CRS, reproject if needed | `QgsCoordinateReferenceSystem`, `QgsProject` |
| 3. Symbology | Apply renderer matching data type | `QgsCategorizedSymbolRenderer`, `QgsGraduatedSymbolRenderer` |
| 4. Labeling | Configure label placement and format | `QgsPalLayerSettings`, `QgsTextFormat` |
| 5. Layout | Create print layout with map items | `QgsPrintLayout`, `QgsLayoutItemMap` |
| 6. Cartographic Elements | Add legend, scale bar, north arrow, title | `QgsLayoutItemLegend`, `QgsLayoutItemScaleBar` |
| 7. Export | Export to PDF/PNG/SVG | `QgsLayoutExporter` |

### Export Format Decision

| Format | When to Use |
|--------|-------------|
| PDF | Print-ready output, multi-page documents, atlas exports |
| PNG | Web display, raster-only output, fixed resolution |
| SVG | Editable vector output, post-processing in Illustrator/Inkscape |

---

## Critical Warnings

**NEVER** export a layout without first setting the map extent — this produces blank or mispositioned maps. ALWAYS call `map_item.zoomToExtent()` or `map_item.setExtent()` before export.

**ALWAYS** call `layout.initializeDefaults()` after creating a new `QgsPrintLayout` — without this, the layout has no pages and export fails silently.

**NEVER** export without checking the result code from `QgsLayoutExporter` — result `0` means success; any other value indicates a specific failure (memory, file, print, SVG layer, or iterator error).

**ALWAYS** call `layer.triggerRepaint()` after changing symbology or labeling — without this, the map canvas and layout exports render stale styles.

**NEVER** use `QgsRandomColorRamp` for thematic maps — random colors destroy visual hierarchy and make maps unreadable. ALWAYS select a color ramp matching the data type (sequential, diverging, or qualitative).

**ALWAYS** link the legend, scale bar, and overview items to the correct map item via `setLinkedMap()` — unlinked items display incorrect or empty content.

**ALWAYS** verify `layer.isValid()` immediately after loading — invalid layers produce empty maps without error messages.

---

## Map Generation Pipeline

This skill orchestrates the FULL pipeline from raw data to exported map. Follow these steps in order.

### Step 1: Data Loading

```python
from qgis.core import QgsVectorLayer, QgsRasterLayer, QgsProject

project = QgsProject.instance()

# Vector layer
vlayer = QgsVectorLayer("/path/to/data.gpkg|layername=buildings", "Buildings", "ogr")
assert vlayer.isValid(), f"Layer failed to load: {vlayer.error().summary()}"
project.addMapLayer(vlayer)

# Raster layer
rlayer = QgsRasterLayer("/path/to/dem.tif", "DEM")
assert rlayer.isValid(), f"Raster failed to load: {rlayer.error().summary()}"
project.addMapLayer(rlayer)
```

### Step 2: CRS Setup

```python
from qgis.core import QgsCoordinateReferenceSystem

# Set project CRS — ALWAYS use a projected CRS for maps with scale bars
project_crs = QgsCoordinateReferenceSystem("EPSG:28992")  # Example: Dutch RD New
project.setCrs(project_crs)
```

**CRS Selection Rules:**
- ALWAYS use a projected CRS (meters/feet) for maps with scale bars or area measurements
- NEVER use EPSG:4326 (geographic) for print maps — degree-based coordinates distort scale bars
- Use UTM zones for maps covering < 6 degrees of longitude
- Use national grid systems (e.g., EPSG:28992, EPSG:27700) for country-specific maps

### Step 3: Symbology

Select renderer based on data characteristics. See the **Symbology Decision Tree** below.

```python
from qgis.core import (
    QgsMarkerSymbol, QgsLineSymbol, QgsFillSymbol,
    QgsCategorizedSymbolRenderer, QgsGraduatedSymbolRenderer,
    QgsRuleBasedRenderer, QgsRendererCategory, QgsRendererRange,
    QgsClassificationRange, QgsStyle
)

# Apply renderer to layer
layer.setRenderer(renderer)
layer.triggerRepaint()
```

### Step 4: Labeling

```python
from qgis.core import QgsPalLayerSettings, QgsVectorLayerSimpleLabeling, QgsTextFormat
from qgis.PyQt.QtGui import QColor

settings = QgsPalLayerSettings()
settings.fieldName = 'name'
settings.isExpression = False
settings.enabled = True

text_format = QgsTextFormat()
text_format.setSize(10)
text_format.setColor(QColor('black'))

# ALWAYS add a buffer (halo) for readability
buffer_settings = text_format.buffer()
buffer_settings.setEnabled(True)
buffer_settings.setSize(1.0)
buffer_settings.setColor(QColor('white'))

settings.setFormat(text_format)

labeling = QgsVectorLayerSimpleLabeling(settings)
layer.setLabeling(labeling)
layer.setLabelsEnabled(True)
layer.triggerRepaint()
```

### Step 5: Layout Creation

```python
from qgis.core import (
    QgsPrintLayout, QgsLayoutItemMap, QgsLayoutSize,
    QgsLayoutPoint, QgsUnitTypes
)

layout = QgsPrintLayout(project)
layout.initializeDefaults()  # CRITICAL: creates default A4 page
layout.setName("Generated Map")

# Add map item
map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(260, 180, QgsUnitTypes.LayoutMillimeters))
map_item.attemptMove(QgsLayoutPoint(15, 25, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(vlayer.extent())
layout.addLayoutItem(map_item)

# Register layout with project
project.layoutManager().addLayout(layout)
```

### Step 6: Cartographic Elements

```python
from qgis.core import (
    QgsLayoutItemLabel, QgsLayoutItemLegend,
    QgsLayoutItemScaleBar, QgsLayoutItemPicture
)

# Title
title = QgsLayoutItemLabel(layout)
title.setText("Map Title")
title.setFont(QFont("Arial", 18))
title.adjustSizeToText()
title.attemptMove(QgsLayoutPoint(15, 5, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(title)

# Legend — ALWAYS link to map item
legend = QgsLayoutItemLegend(layout)
legend.setLinkedMap(map_item)
legend.attemptMove(QgsLayoutPoint(15, 210, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(legend)

# Scale bar — ALWAYS link to map item
scalebar = QgsLayoutItemScaleBar(layout)
scalebar.setStyle("Single Box")
scalebar.setLinkedMap(map_item)
scalebar.applyDefaultSize()
scalebar.attemptMove(QgsLayoutPoint(120, 210, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(scalebar)

# North arrow
north_arrow = QgsLayoutItemPicture(layout)
north_arrow.setPicturePath(":/images/sketchy/sketchy-north.svg")
north_arrow.attemptResize(QgsLayoutSize(15, 15, QgsUnitTypes.LayoutMillimeters))
north_arrow.attemptMove(QgsLayoutPoint(265, 25, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(north_arrow)

# Source attribution
source = QgsLayoutItemLabel(layout)
source.setText("Source: [data source attribution]")
source.adjustSizeToText()
source.attemptMove(QgsLayoutPoint(15, 202, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(source)
```

### Step 7: Export

```python
from qgis.core import QgsLayoutExporter

exporter = QgsLayoutExporter(layout)

# PDF export (most common)
pdf_settings = QgsLayoutExporter.PdfExportSettings()
pdf_settings.dpi = 300
result = exporter.exportToPdf("/path/to/output.pdf", pdf_settings)
assert result == QgsLayoutExporter.ExportResult.Success, f"Export failed: {result}"

# PNG export
img_settings = QgsLayoutExporter.ImageExportSettings()
img_settings.dpi = 300
result = exporter.exportToImage("/path/to/output.png", img_settings)

# SVG export
svg_settings = QgsLayoutExporter.SvgExportSettings()
svg_settings.dpi = 300
result = exporter.exportToSvg("/path/to/output.svg", svg_settings)
```

---

## Symbology Decision Tree

Follow this tree to select the correct renderer:

```
Is the data CATEGORICAL (text/classes)?
├── YES → Are there ≤ 20 unique values?
│   ├── YES → Use QgsCategorizedSymbolRenderer
│   │         Color ramp: QUALITATIVE (distinct hues)
│   └── NO  → Use QgsRuleBasedRenderer with grouped categories
│             Or simplify categories before rendering
└── NO → Is the data NUMERIC?
    ├── YES → Is the data SEQUENTIAL (low-to-high)?
    │   ├── YES → Use QgsGraduatedSymbolRenderer
    │   │         Color ramp: SEQUENTIAL (light-to-dark single hue)
    │   └── NO  → Is there a meaningful MIDPOINT (e.g., zero, average)?
    │       ├── YES → Use QgsGraduatedSymbolRenderer
    │       │         Color ramp: DIVERGING (two hues from midpoint)
    │       └── NO  → Use QgsGraduatedSymbolRenderer
    │                 Color ramp: SEQUENTIAL
    └── NO → Use QgsSingleSymbolRenderer (uniform style)
```

### Color Ramp Selection

| Data Type | Ramp Type | Example Use | PyQGIS Class |
|-----------|-----------|-------------|--------------|
| Sequential numeric | Sequential | Population density, elevation | `QgsGradientColorRamp` |
| Diverging numeric | Diverging | Temperature anomaly, profit/loss | `QgsGradientColorRamp` (two-tone) |
| Categorical | Qualitative | Land use types, building classes | `QgsPresetSchemeColorRamp` |
| Continuous surface | Sequential | DEM hillshade, slope | `QgsGradientColorRamp` |

```python
# Sequential color ramp (light yellow to dark red)
from qgis.core import QgsGradientColorRamp
from qgis.PyQt.QtGui import QColor
ramp = QgsGradientColorRamp(QColor('#ffffb2'), QColor('#bd0026'))

# Use named style ramps from QGIS
style = QgsStyle.defaultStyle()
ramp = style.colorRamp('Spectral')  # or 'Blues', 'RdYlGn', 'Viridis', etc.
```

### Geometry-Specific Symbol Creation

| Geometry | Class | Example |
|----------|-------|---------|
| Point | `QgsMarkerSymbol` | `QgsMarkerSymbol.createSimple({'name': 'circle', 'color': 'red', 'size': '3'})` |
| Line | `QgsLineSymbol` | `QgsLineSymbol.createSimple({'color': 'blue', 'width': '0.5'})` |
| Polygon | `QgsFillSymbol` | `QgsFillSymbol.createSimple({'color': '#aaffaa', 'outline_color': 'black', 'outline_width': '0.3'})` |

---

## Layout Guide

### Standard Page Sizes

| Size | Width x Height (mm) | Use Case |
|------|---------------------|----------|
| A4 Portrait | 210 x 297 | Reports, standard prints |
| A4 Landscape | 297 x 210 | Wide-area maps |
| A3 Portrait | 297 x 420 | Detailed maps |
| A3 Landscape | 420 x 297 | Panoramic maps |
| A0 | 841 x 1189 | Wall maps, posters |

```python
# Set page size
page = layout.pageCollection().page(0)
page.setPageSize(QgsLayoutSize(420, 297, QgsUnitTypes.LayoutMillimeters))  # A3 Landscape
```

### Layout Margin Guidelines

| Element | Recommended Margin (mm) |
|---------|------------------------|
| Page margins | 10-15 on all sides |
| Map frame to edge | 15 minimum |
| Title above map | 5-10 gap |
| Legend below/beside map | 5 gap from map edge |
| Scale bar | Bottom of map area |
| North arrow | Top-right corner of map |

### Cartographic Checklist

Every generated map MUST include:
1. **Title** — descriptive, top of layout
2. **Legend** — linked to map item, shows all visible layers
3. **Scale bar** — linked to map item, uses projected CRS
4. **North arrow** — indicates orientation
5. **Source attribution** — data source credits
6. **Date** — when the map was produced

---

## Atlas Generation

Use atlas for batch-generating maps per feature (e.g., one map per municipality).

```python
atlas = layout.atlas()
atlas.setCoverageLayer(coverage_layer)
atlas.setEnabled(True)
atlas.setFilenameExpression("'map_' || \"name\"")

# Optional: filter features
# atlas.setFilterFeatures(True)
# atlas.setFilterExpression('"status" = \'active\'')

# Map item MUST be controlled by atlas
map_item.setAtlasDriven(True)
map_item.setAtlasScalingMode(QgsLayoutItemMap.AtlasScalingMode.Auto)

# Export all atlas pages to individual PDFs
pdf_settings = QgsLayoutExporter.PdfExportSettings()
pdf_settings.dpi = 300
result = QgsLayoutExporter.exportToPdfs(
    atlas, "/path/to/atlas_output/", pdf_settings
)
```

**Atlas Expression Variables:**
- `@atlas_featurenumber` — current feature index (1-based)
- `@atlas_totalfeatures` — total number of features
- `@atlas_feature` — current feature object
- `@atlas_featureid` — current feature ID
- `@atlas_pagename` — current page name

---

## Export Guide

### DPI Selection

| Purpose | DPI | File Size |
|---------|-----|-----------|
| Screen/web preview | 96 | Small |
| Standard print | 150 | Medium |
| High-quality print | 300 | Large |
| Publication quality | 600 | Very large |

### Export Result Codes

| Code | Constant | Meaning |
|------|----------|---------|
| 0 | `Success` | Export completed |
| 1 | `Canceled` | User canceled |
| 2 | `MemoryError` | Insufficient memory (reduce DPI or map size) |
| 3 | `FileError` | Cannot write to output path |
| 4 | `PrintError` | Print rendering failed |
| 5 | `SvgLayerError` | SVG layer export failed |
| 6 | `IteratorError` | Atlas iteration failed |

### Template Save/Load for Reuse

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
project.layoutManager().addLayout(new_layout)
```

---

## MCP Integration

For runtime map generation via Claude Code, use MCP servers that expose QGIS functionality:

- **qgis-mcp-server** — Provides tool-based access to QGIS project manipulation, layer management, and export
- **Processing MCP** — Exposes QGIS Processing algorithms as MCP tools

When an MCP server is available, the map generation pipeline steps remain identical — the MCP server wraps the same PyQGIS calls described above. ALWAYS validate layer loading and export results through the MCP response.

---

## Validation Checklist

Before delivering a generated map, verify ALL items:

- [ ] All layers loaded successfully (`isValid() == True`)
- [ ] Project CRS is a projected system (not geographic/EPSG:4326)
- [ ] Symbology matches the data type (categorical/graduated/single)
- [ ] Color ramp matches data semantics (sequential/diverging/qualitative)
- [ ] Labels have buffer/halo enabled for readability
- [ ] Layout has title, legend, scale bar, north arrow, source attribution
- [ ] Legend is linked to the map item
- [ ] Scale bar is linked to the map item
- [ ] Map extent is set before export (not empty/default)
- [ ] Export result code is `Success` (0)
- [ ] Output file exists and has non-zero size
- [ ] DPI matches intended use (300 for print, 96 for screen)

---

## Reference Links

- [references/methods.md](references/methods.md) — API signatures for map generation classes
- [references/examples.md](references/examples.md) — Complete working map generation examples
- [references/anti-patterns.md](references/anti-patterns.md) — Common map generation mistakes and fixes

### Official Sources

- https://qgis.org/pyqgis/3.x/core/QgsPrintLayout.html
- https://qgis.org/pyqgis/3.x/core/QgsLayoutExporter.html
- https://qgis.org/pyqgis/3.x/core/QgsCategorizedSymbolRenderer.html
- https://qgis.org/pyqgis/3.x/core/QgsGraduatedSymbolRenderer.html
- https://docs.qgis.org/3.34/en/docs/pyqgis_developer_cookbook/composer.html
