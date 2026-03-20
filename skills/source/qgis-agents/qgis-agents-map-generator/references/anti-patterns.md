# qgis-agents-map-generator — Anti-Patterns

## AP-1: Exporting Without Setting Map Extent

**Wrong:**
```python
layout = QgsPrintLayout(project)
layout.initializeDefaults()
map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(200, 150, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(map_item)
# Directly export — map extent is empty
exporter = QgsLayoutExporter(layout)
exporter.exportToPdf("/output/map.pdf", QgsLayoutExporter.PdfExportSettings())
```

**Why it fails:** The map item has no extent set. The export produces a blank or incorrectly positioned map with no visible data.

**Correct:**
```python
map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(200, 150, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(layer.extent())  # ALWAYS set extent before export
layout.addLayoutItem(map_item)
```

---

## AP-2: Missing initializeDefaults() Call

**Wrong:**
```python
layout = QgsPrintLayout(project)
layout.setName("My Map")
# No initializeDefaults() — layout has zero pages
map_item = QgsLayoutItemMap(layout)
layout.addLayoutItem(map_item)
```

**Why it fails:** Without `initializeDefaults()`, the layout has no pages. Items are added to a non-existent page, and export produces empty output or fails.

**Correct:**
```python
layout = QgsPrintLayout(project)
layout.initializeDefaults()  # ALWAYS call this immediately after construction
layout.setName("My Map")
```

---

## AP-3: Ignoring Export Result Codes

**Wrong:**
```python
exporter = QgsLayoutExporter(layout)
exporter.exportToPdf("/output/map.pdf", pdf_settings)
# Assume success — no result check
print("Export complete!")
```

**Why it fails:** The export may fail due to memory errors, file permission issues, or rendering problems. Without checking the result code, the script reports success when the output file is missing or corrupt.

**Correct:**
```python
result = exporter.exportToPdf("/output/map.pdf", pdf_settings)
if result != QgsLayoutExporter.ExportResult.Success:
    raise RuntimeError(f"Export failed with code: {result}")
```

---

## AP-4: Using Random Colors for Thematic Maps

**Wrong:**
```python
from qgis.core import QgsRandomColorRamp

renderer = QgsGraduatedSymbolRenderer()
renderer.setSourceColorRamp(QgsRandomColorRamp())
```

**Why it fails:** Random colors destroy visual hierarchy. A graduated map needs sequential or diverging colors so readers can intuitively understand high vs. low values. Random colors make the map unreadable.

**Correct:**
```python
# Sequential ramp for sequential data
ramp = QgsGradientColorRamp(QColor('#ffffcc'), QColor('#800026'))
renderer.setSourceColorRamp(ramp)

# Or use a named style ramp
ramp = QgsStyle.defaultStyle().colorRamp('YlOrRd')
renderer.setSourceColorRamp(ramp)
```

---

## AP-5: Unlinked Legend or Scale Bar

**Wrong:**
```python
legend = QgsLayoutItemLegend(layout)
legend.attemptMove(QgsLayoutPoint(15, 200, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(legend)
# Legend is not linked to any map item
```

**Why it fails:** An unlinked legend shows all project layers regardless of the map item's visible layers and extent. An unlinked scale bar shows incorrect or default scale values.

**Correct:**
```python
legend = QgsLayoutItemLegend(layout)
legend.setLinkedMap(map_item)  # ALWAYS link to the map item
layout.addLayoutItem(legend)

scalebar = QgsLayoutItemScaleBar(layout)
scalebar.setLinkedMap(map_item)  # ALWAYS link to the map item
layout.addLayoutItem(scalebar)
```

---

## AP-6: Using Geographic CRS (EPSG:4326) for Print Maps

**Wrong:**
```python
project.setCrs(QgsCoordinateReferenceSystem("EPSG:4326"))
# Scale bar now shows degrees instead of meters
```

**Why it fails:** EPSG:4326 uses degrees as units. Scale bars display meaningless degree values. Distance measurements are incorrect because degrees have variable length depending on latitude.

**Correct:**
```python
# Use a projected CRS with meter-based units
project.setCrs(QgsCoordinateReferenceSystem("EPSG:28992"))  # Dutch RD New
# Or UTM zone appropriate for the data location
project.setCrs(QgsCoordinateReferenceSystem("EPSG:32632"))  # UTM Zone 32N
```

---

## AP-7: Forgetting triggerRepaint() After Symbology Change

**Wrong:**
```python
layer.setRenderer(new_renderer)
# Directly create layout — renderer change not reflected
layout = QgsPrintLayout(project)
```

**Why it fails:** Without `triggerRepaint()`, the map canvas and layout renders use the previous (cached) symbology. The exported map shows stale styles.

**Correct:**
```python
layer.setRenderer(new_renderer)
layer.triggerRepaint()  # ALWAYS repaint after changing renderer or labeling
```

---

## AP-8: Labels Without Buffer (Halo)

**Wrong:**
```python
settings = QgsPalLayerSettings()
settings.fieldName = 'name'
settings.enabled = True

text_format = QgsTextFormat()
text_format.setSize(10)
text_format.setColor(QColor('black'))
settings.setFormat(text_format)
# No buffer — labels disappear on dark backgrounds
```

**Why it fails:** Labels without a buffer blend into the background, especially over dark symbology or raster layers. They become unreadable on busy maps.

**Correct:**
```python
text_format = QgsTextFormat()
text_format.setSize(10)
text_format.setColor(QColor('black'))

buffer_settings = text_format.buffer()
buffer_settings.setEnabled(True)
buffer_settings.setSize(1.0)
buffer_settings.setColor(QColor('white'))

settings.setFormat(text_format)
```

---

## AP-9: Not Validating Layer Before Map Generation

**Wrong:**
```python
layer = QgsVectorLayer("/data/missing_file.shp", "Data", "ogr")
project.addMapLayer(layer)
# Layer is invalid — all downstream operations produce empty results
renderer = QgsCategorizedSymbolRenderer()
layer.setRenderer(renderer)
```

**Why it fails:** An invalid layer has no features, no fields, and no extent. The map generation pipeline produces empty maps without any error indication.

**Correct:**
```python
layer = QgsVectorLayer("/data/missing_file.shp", "Data", "ogr")
assert layer.isValid(), f"Layer failed to load: {layer.error().summary()}"
project.addMapLayer(layer)
```

---

## AP-10: Atlas Without setAtlasDriven(True) on Map Item

**Wrong:**
```python
atlas = layout.atlas()
atlas.setCoverageLayer(districts)
atlas.setEnabled(True)
# Map item is NOT atlas-driven — all pages show the same extent
```

**Why it fails:** Without `setAtlasDriven(True)`, the map item displays a static extent for every atlas page. Every exported page shows the same area instead of zooming to the current atlas feature.

**Correct:**
```python
atlas = layout.atlas()
atlas.setCoverageLayer(districts)
atlas.setEnabled(True)

map_item.setAtlasDriven(True)
map_item.setAtlasScalingMode(QgsLayoutItemMap.AtlasScalingMode.Auto)
```

---

## AP-11: Mismatching Symbology Type and Geometry Type

**Wrong:**
```python
# Layer contains polygons, but using marker symbol
symbol = QgsMarkerSymbol.createSimple({'name': 'circle', 'color': 'red'})
renderer = QgsSingleSymbolRenderer(symbol)
polygon_layer.setRenderer(renderer)
```

**Why it fails:** Applying a marker symbol to a polygon layer produces no visible rendering. QGIS silently ignores the mismatched symbol type.

**Correct:**
```python
# Match symbol type to geometry type
if polygon_layer.geometryType() == Qgis.GeometryType.Polygon:
    symbol = QgsFillSymbol.createSimple({'color': 'red', 'outline_color': 'black'})
elif polygon_layer.geometryType() == Qgis.GeometryType.Line:
    symbol = QgsLineSymbol.createSimple({'color': 'red', 'width': '0.5'})
elif polygon_layer.geometryType() == Qgis.GeometryType.Point:
    symbol = QgsMarkerSymbol.createSimple({'name': 'circle', 'color': 'red'})

renderer = QgsSingleSymbolRenderer(symbol)
layer.setRenderer(renderer)
```

---

## AP-12: Hard-Coded Item Positions Exceeding Page Bounds

**Wrong:**
```python
# A4 page is 210x297mm, but item placed at x=300
legend.attemptMove(QgsLayoutPoint(300, 200, QgsUnitTypes.LayoutMillimeters))
```

**Why it fails:** Items placed outside the page bounds are not visible in the export. QGIS does not warn about off-page items.

**Correct:**
```python
# ALWAYS verify positions are within page dimensions
page_size = layout.pageCollection().page(0).pageSize()
max_x = page_size.width()   # e.g., 210 for A4 portrait
max_y = page_size.height()  # e.g., 297 for A4 portrait

# Place within bounds
legend.attemptMove(QgsLayoutPoint(15, 200, QgsUnitTypes.LayoutMillimeters))
```
