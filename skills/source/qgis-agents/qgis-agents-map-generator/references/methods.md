# qgis-agents-map-generator — Methods Reference

## Symbology Classes

### QgsMarkerSymbol

```python
QgsMarkerSymbol.createSimple(properties: dict) -> QgsMarkerSymbol
```
Creates a marker symbol from a properties dictionary. Keys: `name`, `color`, `size`, `outline_color`, `outline_width`.

Available marker names: `circle`, `square`, `cross`, `rectangle`, `diamond`, `pentagon`, `triangle`, `equilateral_triangle`, `star`, `regular_star`, `arrow`, `filled_arrowhead`, `x`.

### QgsLineSymbol

```python
QgsLineSymbol.createSimple(properties: dict) -> QgsLineSymbol
```
Creates a line symbol. Keys: `color`, `width`, `line_style` (`solid`, `dash`, `dot`, `dash dot`).

### QgsFillSymbol

```python
QgsFillSymbol.createSimple(properties: dict) -> QgsFillSymbol
```
Creates a fill symbol. Keys: `color`, `outline_color`, `outline_width`, `style` (`solid`, `no`, `horizontal`, `vertical`, `cross`).

### QgsSymbol (base class)

```python
symbol.symbolLayerCount() -> int
symbol.symbolLayer(index: int) -> QgsSymbolLayer
symbol.symbolLayers() -> list[QgsSymbolLayer]
symbol.setColor(color: QColor) -> None
symbol.color() -> QColor
```

---

## Renderer Classes

### QgsSingleSymbolRenderer

```python
QgsSingleSymbolRenderer(symbol: QgsSymbol)
renderer.setSymbol(symbol: QgsSymbol) -> None
renderer.symbol() -> QgsSymbol
```

### QgsCategorizedSymbolRenderer

```python
QgsCategorizedSymbolRenderer(attrName: str = '', categories: list = [])
renderer.setClassAttribute(attr: str) -> None
renderer.classAttribute() -> str
renderer.addCategory(category: QgsRendererCategory) -> None
renderer.categories() -> list[QgsRendererCategory]
renderer.updateCategorySymbol(index: int, symbol: QgsSymbol) -> None
renderer.updateCategoryLabel(index: int, label: str) -> None
```

### QgsRendererCategory

```python
QgsRendererCategory(value, symbol: QgsSymbol, label: str, render: bool = True)
category.value() -> QVariant
category.symbol() -> QgsSymbol
category.label() -> str
```

### QgsGraduatedSymbolRenderer

```python
QgsGraduatedSymbolRenderer(attrName: str = '', ranges: list = [])
renderer.setClassAttribute(attr: str) -> None
renderer.classAttribute() -> str
renderer.addClassRange(range: QgsRendererRange) -> None
renderer.ranges() -> list[QgsRendererRange]
renderer.setSourceColorRamp(ramp: QgsColorRamp) -> None
renderer.updateRangeSymbol(index: int, symbol: QgsSymbol) -> None
renderer.updateRangeLabel(index: int, label: str) -> None
renderer.updateRangeLowerValue(index: int, value: float) -> None
renderer.updateRangeUpperValue(index: int, value: float) -> None
```

### QgsRendererRange

```python
QgsRendererRange(range: QgsClassificationRange, symbol: QgsSymbol)
range.lowerValue() -> float
range.upperValue() -> float
range.label() -> str
range.symbol() -> QgsSymbol
```

### QgsClassificationRange

```python
QgsClassificationRange(label: str, lower: float, upper: float)
```

### QgsRuleBasedRenderer

```python
QgsRuleBasedRenderer(root_rule: QgsRuleBasedRenderer.Rule)

# Rule class
QgsRuleBasedRenderer.Rule(symbol: QgsSymbol, scaleMinDenom: int = 0,
                          scaleMaxDenom: int = 0, filterExp: str = '',
                          label: str = '', description: str = '')
rule.appendChild(child: QgsRuleBasedRenderer.Rule) -> None
rule.children() -> list[QgsRuleBasedRenderer.Rule]
rule.filterExpression() -> str
rule.setFilterExpression(exp: str) -> None
rule.label() -> str
rule.symbol() -> QgsSymbol
```

---

## Color Ramp Classes

### QgsGradientColorRamp

```python
QgsGradientColorRamp(color1: QColor, color2: QColor,
                     discrete: bool = False, stops: list = [])
ramp.color(value: float) -> QColor  # value 0.0-1.0
```

### QgsPresetSchemeColorRamp

```python
QgsPresetSchemeColorRamp(colors: list[QColor])
```

### QgsStyle (access built-in ramps)

```python
style = QgsStyle.defaultStyle()
style.colorRampNames() -> list[str]
style.colorRamp(name: str) -> QgsColorRamp
```

---

## Labeling Classes

### QgsPalLayerSettings

```python
settings = QgsPalLayerSettings()
settings.fieldName: str          # Field name or expression
settings.isExpression: bool      # True if fieldName is an expression
settings.enabled: bool           # Enable/disable labeling
settings.drawLabels: bool        # Draw labels flag
settings.placement: Qgis.LabelPlacement  # Placement mode
settings.setFormat(format: QgsTextFormat) -> None
settings.format() -> QgsTextFormat
```

**Qgis.LabelPlacement enum values:**
- `AroundPoint` — labels placed around point features
- `OverPoint` — labels placed directly over point
- `Line` — labels follow line direction
- `Curved` — labels curved along line
- `Horizontal` — horizontal labels for polygons
- `Free` — free placement for polygons
- `OrderedPositionsAroundPoint` — priority-ordered positions

### QgsTextFormat

```python
text_format = QgsTextFormat()
text_format.setSize(size: float) -> None
text_format.size() -> float
text_format.setColor(color: QColor) -> None
text_format.color() -> QColor
text_format.setFont(font: QFont) -> None
text_format.font() -> QFont
text_format.buffer() -> QgsTextBufferSettings
```

### QgsTextBufferSettings

```python
buffer = text_format.buffer()
buffer.setEnabled(enabled: bool) -> None
buffer.setSize(size: float) -> None
buffer.setColor(color: QColor) -> None
buffer.setSizeUnit(unit: Qgis.RenderUnit) -> None
```

### QgsVectorLayerSimpleLabeling

```python
QgsVectorLayerSimpleLabeling(settings: QgsPalLayerSettings)
```

### QgsRuleBasedLabeling

```python
QgsRuleBasedLabeling(root: QgsRuleBasedLabeling.Rule)

QgsRuleBasedLabeling.Rule(settings: QgsPalLayerSettings, scaleMinDenom: int = 0,
                          scaleMaxDenom: int = 0, filterExp: str = '',
                          description: str = '')
rule.appendChild(child: QgsRuleBasedLabeling.Rule) -> None
rule.setFilterExpression(exp: str) -> None
```

---

## Print Layout Classes

### QgsPrintLayout

```python
QgsPrintLayout(project: QgsProject)
layout.initializeDefaults() -> None           # CRITICAL: creates default page
layout.setName(name: str) -> None
layout.name() -> str
layout.addLayoutItem(item: QgsLayoutItem) -> None
layout.removeLayoutItem(item: QgsLayoutItem) -> None
layout.pageCollection() -> QgsLayoutPageCollection
layout.atlas() -> QgsLayoutAtlas
```

### QgsLayoutItemMap

```python
QgsLayoutItemMap(layout: QgsLayout)
map_item.attemptResize(size: QgsLayoutSize) -> None
map_item.attemptMove(point: QgsLayoutPoint) -> None
map_item.zoomToExtent(extent: QgsRectangle) -> None
map_item.setExtent(extent: QgsRectangle) -> None
map_item.setScale(scale: float) -> None
map_item.scale() -> float
map_item.setCrs(crs: QgsCoordinateReferenceSystem) -> None
map_item.setAtlasDriven(enabled: bool) -> None
map_item.setAtlasScalingMode(mode: QgsLayoutItemMap.AtlasScalingMode) -> None
map_item.setFrameEnabled(enabled: bool) -> None
```

### QgsLayoutItemLabel

```python
QgsLayoutItemLabel(layout: QgsLayout)
label.setText(text: str) -> None             # Supports expressions: [% expr %]
label.adjustSizeToText() -> None
label.setFont(font: QFont) -> None
label.attemptMove(point: QgsLayoutPoint) -> None
label.attemptResize(size: QgsLayoutSize) -> None
```

### QgsLayoutItemLegend

```python
QgsLayoutItemLegend(layout: QgsLayout)
legend.setLinkedMap(map_item: QgsLayoutItemMap) -> None
legend.attemptMove(point: QgsLayoutPoint) -> None
legend.setAutoUpdateModel(auto: bool) -> None
legend.setTitle(title: str) -> None
```

### QgsLayoutItemScaleBar

```python
QgsLayoutItemScaleBar(layout: QgsLayout)
scalebar.setStyle(style: str) -> None        # "Single Box", "Double Box", "Line Ticks Middle", "Numeric"
scalebar.setLinkedMap(map_item: QgsLayoutItemMap) -> None
scalebar.applyDefaultSize() -> None
scalebar.attemptMove(point: QgsLayoutPoint) -> None
```

### QgsLayoutItemPicture

```python
QgsLayoutItemPicture(layout: QgsLayout)
picture.setPicturePath(path: str) -> None    # SVG or raster image path
picture.attemptResize(size: QgsLayoutSize) -> None
picture.attemptMove(point: QgsLayoutPoint) -> None
```

### QgsLayoutSize / QgsLayoutPoint

```python
QgsLayoutSize(width: float, height: float, units: QgsUnitTypes.LayoutUnit)
QgsLayoutPoint(x: float, y: float, units: QgsUnitTypes.LayoutUnit)
# Units: QgsUnitTypes.LayoutMillimeters, LayoutCentimeters, LayoutInches, LayoutPoints
```

### QgsLayoutItemPage

```python
QgsLayoutItemPage(layout: QgsLayout)
page.setPageSize(size: QgsLayoutSize) -> None
layout.pageCollection().addPage(page) -> None
```

---

## Atlas Classes

### QgsLayoutAtlas

```python
atlas = layout.atlas()
atlas.setCoverageLayer(layer: QgsVectorLayer) -> None
atlas.setEnabled(enabled: bool) -> None
atlas.setFilenameExpression(expression: str) -> None
atlas.setFilterFeatures(enabled: bool) -> None
atlas.setFilterExpression(expression: str) -> None
atlas.beginRender() -> bool
atlas.next() -> bool
atlas.endRender() -> None
atlas.currentFeatureNumber() -> int
atlas.nameForPage(page: int) -> str
atlas.count() -> int
```

---

## Export Classes

### QgsLayoutExporter

```python
QgsLayoutExporter(layout: QgsLayout)
exporter.exportToPdf(path: str, settings: PdfExportSettings) -> ExportResult
exporter.exportToImage(path: str, settings: ImageExportSettings) -> ExportResult
exporter.exportToSvg(path: str, settings: SvgExportSettings) -> ExportResult

# Static method for atlas export
QgsLayoutExporter.exportToPdfs(atlas: QgsLayoutAtlas, baseDir: str,
                               settings: PdfExportSettings) -> ExportResult
```

### Export Settings

```python
# PDF
pdf_settings = QgsLayoutExporter.PdfExportSettings()
pdf_settings.dpi: float = 300
pdf_settings.forceVectorOutput: bool = False
pdf_settings.appendGeoreference: bool = True

# Image
img_settings = QgsLayoutExporter.ImageExportSettings()
img_settings.dpi: float = 300
img_settings.generateWorldFile: bool = False

# SVG
svg_settings = QgsLayoutExporter.SvgExportSettings()
svg_settings.dpi: float = 300
svg_settings.forceVectorOutput: bool = False
```

### ExportResult Enum

```python
QgsLayoutExporter.ExportResult.Success        # 0
QgsLayoutExporter.ExportResult.Canceled       # 1
QgsLayoutExporter.ExportResult.MemoryError    # 2
QgsLayoutExporter.ExportResult.FileError      # 3
QgsLayoutExporter.ExportResult.PrintError     # 4
QgsLayoutExporter.ExportResult.SvgLayerError  # 5
QgsLayoutExporter.ExportResult.IteratorError  # 6
```

---

## Template I/O

### Save/Load Layout Templates

```python
from qgis.core import QgsReadWriteContext
from qgis.PyQt.QtXml import QDomDocument

# Save
doc = QDomDocument()
layout.writeXml(doc, QgsReadWriteContext())
with open(path, "w") as f:
    f.write(doc.toString())

# Load
doc = QDomDocument()
doc.setContent(content_str)
new_layout = QgsPrintLayout(project)
new_layout.readXml(doc.documentElement(), doc, QgsReadWriteContext())
```
