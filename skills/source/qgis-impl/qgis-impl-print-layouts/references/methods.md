# API Signatures Reference (QGIS Print Layouts)

## QgsPrintLayout

The main layout class for print compositions. Inherits from `QgsLayout`.

```python
class QgsPrintLayout(QgsLayout):
    def __init__(self, project: QgsProject) -> None
    def initializeDefaults(self) -> None
        # Creates a default A4 page. ALWAYS call after construction.
    def setName(self, name: str) -> None
    def name(self) -> str
    def atlas(self) -> QgsLayoutAtlas
    def addLayoutItem(self, item: QgsLayoutItem) -> None
    def removeLayoutItem(self, item: QgsLayoutItem) -> None
    def addMultiFrame(self, multiFrame: QgsLayoutMultiFrame) -> None
    def removeMultiFrame(self, multiFrame: QgsLayoutMultiFrame) -> None
    def pageCollection(self) -> QgsLayoutPageCollection
    def writeXml(self, document: QDomDocument, context: QgsReadWriteContext) -> bool
    def readXml(self, element: QDomElement, document: QDomDocument, context: QgsReadWriteContext) -> bool
```

---

## QgsLayoutManager

Manages all layouts within a project.

```python
class QgsLayoutManager:
    def addLayout(self, layout: QgsMasterLayoutInterface) -> bool
    def removeLayout(self, layout: QgsMasterLayoutInterface) -> bool
    def printLayouts(self) -> List[QgsPrintLayout]
    def layoutByName(self, name: str) -> QgsMasterLayoutInterface
    def layouts(self) -> List[QgsMasterLayoutInterface]
    def clear(self) -> None
```

**Usage**: Access via `QgsProject.instance().layoutManager()`.

---

## QgsLayoutItemMap

Displays a map within a layout. ALWAYS set the extent before export.

```python
class QgsLayoutItemMap(QgsLayoutItem):
    def __init__(self, layout: QgsLayout) -> None
    def zoomToExtent(self, extent: QgsRectangle) -> None
        # Sets the map extent to match the given rectangle.
    def setScale(self, scale: float, forceUpdate: bool = True) -> None
        # Sets the map scale (e.g., 50000 for 1:50000).
    def scale(self) -> float
    def setCrs(self, crs: QgsCoordinateReferenceSystem) -> None
    def crs(self) -> QgsCoordinateReferenceSystem
    def extent(self) -> QgsRectangle
    def setExtent(self, extent: QgsRectangle) -> None
    def setFollowVisibilityPreset(self, follow: bool) -> None
    def setFollowVisibilityPresetName(self, name: str) -> None
    def setLayers(self, layers: List[QgsMapLayer]) -> None
        # Locks specific layers to display in this map item.
    def layers(self) -> List[QgsMapLayer]
    def setKeepLayerSet(self, enabled: bool) -> None
    def grids(self) -> QgsLayoutItemMapGridStack
    def overviews(self) -> QgsLayoutItemMapOverviewStack
    def setMapRotation(self, rotation: float) -> None
    def mapRotation(self) -> float
```

---

## QgsLayoutItemLabel

Text label item for titles, descriptions, and dynamic expressions.

```python
class QgsLayoutItemLabel(QgsLayoutItem):
    def __init__(self, layout: QgsLayout) -> None
    def setText(self, text: str) -> None
        # Supports QGIS expressions inside [% %] delimiters.
    def text(self) -> str
    def adjustSizeToText(self) -> None
    def setFont(self, font: QFont) -> None
    def setFontColor(self, color: QColor) -> None
    def setHAlign(self, alignment: Qt.AlignmentFlag) -> None
    def setVAlign(self, alignment: Qt.AlignmentFlag) -> None
    def setMarginX(self, margin: float) -> None
    def setMarginY(self, margin: float) -> None
    def setMode(self, mode: QgsLayoutItemLabel.Mode) -> None
        # Mode.ModeFont for plain text, Mode.ModeHtml for HTML rendering.
```

---

## QgsLayoutItemLegend

Legend item linked to a map item.

```python
class QgsLayoutItemLegend(QgsLayoutItem):
    def __init__(self, layout: QgsLayout) -> None
    def setLinkedMap(self, map_item: QgsLayoutItemMap) -> None
    def setAutoUpdateModel(self, auto: bool) -> None
        # When True, legend updates automatically from linked map layers.
    def setTitle(self, title: str) -> None
    def setStyle(self, component: QgsLegendStyle.Style, style: QgsLegendStyle) -> None
    def setColumnCount(self, count: int) -> None
    def setSplitLayer(self, split: bool) -> None
    def setEqualColumnWidth(self, equal: bool) -> None
    def model(self) -> QgsLegendModel
```

---

## QgsLayoutItemScaleBar

Scale bar linked to a map item.

```python
class QgsLayoutItemScaleBar(QgsLayoutItem):
    def __init__(self, layout: QgsLayout) -> None
    def setLinkedMap(self, map_item: QgsLayoutItemMap) -> None
    def setStyle(self, name: str) -> None
        # Styles: "Single Box", "Double Box", "Line Ticks Middle",
        # "Line Ticks Down", "Line Ticks Up", "Numeric"
    def applyDefaultSize(self) -> None
    def setUnits(self, units: QgsUnitTypes.DistanceUnit) -> None
    def setNumberOfSegments(self, segments: int) -> None
    def setNumberOfSegmentsLeft(self, segments: int) -> None
    def setUnitsPerSegment(self, units: float) -> None
    def setMapUnitsPerScaleBarUnit(self, units: float) -> None
    def setUnitLabel(self, label: str) -> None
```

---

## QgsLayoutItemPicture

Image item for logos, north arrows, and other graphics.

```python
class QgsLayoutItemPicture(QgsLayoutItem):
    def __init__(self, layout: QgsLayout) -> None
    def setPicturePath(self, path: str) -> None
        # Supports SVG, PNG, JPEG, and other image formats.
    def setPictureRotation(self, rotation: float) -> None
    def setResizeMode(self, mode: QgsLayoutItemPicture.ResizeMode) -> None
    def setLinkedMap(self, map_item: QgsLayoutItemMap) -> None
        # For north arrows: syncs rotation with map rotation.
    def setSvgFillColor(self, color: QColor) -> None
    def setSvgStrokeColor(self, color: QColor) -> None
```

---

## QgsLayoutItemAttributeTable

Displays feature attributes in a table format. Uses a multi-frame model.

```python
class QgsLayoutItemAttributeTable(QgsLayoutMultiFrame):
    @staticmethod
    def create(layout: QgsLayout) -> QgsLayoutItemAttributeTable
    def setVectorLayer(self, layer: QgsVectorLayer) -> None
    def setMaximumNumberOfFeatures(self, max: int) -> None
    def setFilterFeatures(self, filter: bool) -> None
    def setFeatureFilter(self, expression: str) -> None
    def setDisplayedFields(self, fields: List[str]) -> None
    def addFrame(self, frame: QgsLayoutFrame) -> None
    def setContentTextFormat(self, format: QgsTextFormat) -> None
    def setHeaderTextFormat(self, format: QgsTextFormat) -> None
```

---

## QgsLayoutExporter

Handles export of layouts to PDF, SVG, and image formats.

```python
class QgsLayoutExporter:
    def __init__(self, layout: QgsLayout) -> None

    # Instance methods (single layout export)
    def exportToPdf(self, filePath: str, settings: PdfExportSettings) -> ExportResult
    def exportToImage(self, filePath: str, settings: ImageExportSettings) -> ExportResult
    def exportToSvg(self, filePath: str, settings: SvgExportSettings) -> ExportResult

    # Static methods (atlas export)
    @staticmethod
    def exportToPdfs(atlas: QgsLayoutAtlas, baseFilePath: str, settings: PdfExportSettings) -> ExportResult
    @staticmethod
    def exportToPdf(atlas: QgsLayoutAtlas, filePath: str, settings: PdfExportSettings) -> ExportResult
        # Exports all atlas pages to a single multi-page PDF.

    # Export settings classes
    class PdfExportSettings:
        dpi: float              # Default: 300
        rasterizeWholeImage: bool
        forceVectorOutput: bool
        appendGeoreference: bool
        textRenderFormat: QgsRenderContext.TextRenderFormat
        simplifyGeometries: bool

    class ImageExportSettings:
        dpi: float              # Default: 300
        imageSizePixels: QSize  # Override output size
        cropToContents: bool
        cropMargins: QgsMargins
        generateWorldFile: bool

    class SvgExportSettings:
        dpi: float              # Default: 300
        forceVectorOutput: bool
        exportAsLayers: bool
        cropToContents: bool

    # Result enum
    Success = 0
    Canceled = 1
    MemoryError = 2
    FileError = 3
    PrintError = 4
    SvgLayerError = 5
    IteratorError = 6
```

---

## QgsLayoutAtlas

Atlas generation for iterating over coverage layer features.

```python
class QgsLayoutAtlas:
    def setCoverageLayer(self, layer: QgsVectorLayer) -> None
    def coverageLayer(self) -> QgsVectorLayer
    def setEnabled(self, enabled: bool) -> None
    def enabled(self) -> bool
    def setFilenameExpression(self, expression: str) -> bool
    def setFilterFeatures(self, filter: bool) -> None
    def setFilterExpression(self, expression: str) -> None
    def setSortFeatures(self, sort: bool) -> None
    def setSortExpression(self, expression: str) -> None
    def setSortAscending(self, ascending: bool) -> None
    def beginRender(self) -> bool
        # MUST be called before iterating with next().
    def next(self) -> bool
        # Advances to the next atlas feature. Returns False when done.
    def endRender(self) -> None
        # ALWAYS call after iteration is complete.
    def currentFeatureNumber(self) -> int
    def nameForPage(self, pageNumber: int) -> str
    def count(self) -> int
    def seekTo(self, feature: int) -> bool
        # Jump to a specific feature by index.
```

---

## QgsLayoutPageCollection

Manages pages within a layout.

```python
class QgsLayoutPageCollection:
    def page(self, index: int) -> QgsLayoutItemPage
    def pageCount(self) -> int
    def addPage(self, page: QgsLayoutItemPage) -> None
    def deletePage(self, index: int) -> None
    def pages(self) -> List[QgsLayoutItemPage]
```

---

## QgsLayoutItemPage

Represents a single page in the layout.

```python
class QgsLayoutItemPage(QgsLayoutItem):
    def __init__(self, layout: QgsLayout) -> None
    def setPageSize(self, size: QgsLayoutSize) -> None
    def pageSize(self) -> QgsLayoutSize
```

---

## Common Positioning Classes

```python
class QgsLayoutSize:
    def __init__(self, width: float, height: float, units: QgsUnitTypes.LayoutUnit = QgsUnitTypes.LayoutMillimeters) -> None

class QgsLayoutPoint:
    def __init__(self, x: float, y: float, units: QgsUnitTypes.LayoutUnit = QgsUnitTypes.LayoutMillimeters) -> None
```

---

## QgsLayoutItemMapGrid

Grid overlay for map items.

```python
class QgsLayoutItemMapGrid:
    def __init__(self, name: str, map_item: QgsLayoutItemMap) -> None
    def setIntervalX(self, interval: float) -> None
    def setIntervalY(self, interval: float) -> None
    def setAnnotationEnabled(self, enabled: bool) -> None
    def setFrameStyle(self, style: QgsLayoutItemMapGrid.FrameStyle) -> None
        # Styles: NoFrame, Zebra, InteriorTicks, ExteriorTicks, InteriorExteriorTicks, LineBorder
    def setGridLineColor(self, color: QColor) -> None
    def setGridLineWidth(self, width: float) -> None
    def setCrs(self, crs: QgsCoordinateReferenceSystem) -> None
```

**Usage**: Access via `map_item.grids().addGrid(grid)`.
