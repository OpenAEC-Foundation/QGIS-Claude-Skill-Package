# API Signatures Reference (Raster Analysis)

## QgsRasterLayer

```python
class QgsRasterLayer(QgsMapLayer):
    def __init__(self, path: str, baseName: str = '', providerType: str = 'gdal') -> None
    def isValid(self) -> bool
    def width(self) -> int                    # Pixel columns
    def height(self) -> int                   # Pixel rows
    def extent(self) -> QgsRectangle
    def bandCount(self) -> int
    def bandName(self, bandNo: int) -> str    # 1-indexed
    def rasterType(self) -> int               # 0=GrayOrUndefined, 1=Palette, 2=Multiband
    def crs(self) -> QgsCoordinateReferenceSystem
    def dataProvider(self) -> QgsRasterDataProvider
    def renderer(self) -> QgsRasterRenderer
    def setRenderer(self, renderer: QgsRasterRenderer) -> None
    def triggerRepaint(self) -> None
    def setContrastEnhancement(self, algorithm: QgsContrastEnhancement.ContrastEnhancementAlgorithm,
                                limits: QgsRasterMinMaxOrigin.Limits = ...,
                                extent: QgsRectangle = ...,
                                sampleSize: int = ...,
                                generateLookupTableFlag: bool = True) -> None
```

---

## QgsRasterDataProvider

```python
class QgsRasterDataProvider(QgsDataProvider, QgsRasterInterface):
    def sample(self, point: QgsPointXY, band: int) -> Tuple[float, bool]
    def identify(self, point: QgsPointXY, format: QgsRaster.IdentifyFormat) -> QgsRasterIdentifyResult
    def block(self, bandNo: int, extent: QgsRectangle, width: int, height: int) -> QgsRasterBlock
    def bandStatistics(self, bandNo: int, stats: int = QgsRasterBandStats.All,
                       extent: QgsRectangle = ..., sampleSize: int = 0) -> QgsRasterBandStats
    def histogram(self, bandNo: int, binCount: int = 0) -> QgsRasterHistogram
    def dataType(self, bandNo: int) -> Qgis.DataType
    def sourceNoDataValue(self, bandNo: int) -> float
    def sourceHasNoDataValue(self, bandNo: int) -> bool
    def setEditable(self, enabled: bool) -> bool
    def writeBlock(self, block: QgsRasterBlock, band: int, xOffset: int = 0, yOffset: int = 0) -> bool
```

---

## QgsRasterBlock

```python
class QgsRasterBlock:
    def __init__(self, dataType: Qgis.DataType = ..., width: int = 0, height: int = 0) -> None
    def value(self, row: int, column: int) -> float
    def setValue(self, row: int, column: int, value: float) -> bool
    def isNoData(self, row: int, column: int) -> bool
    def setIsNoData(self, row: int, column: int) -> bool
    def setNoDataValue(self, noDataValue: float) -> None
    def width(self) -> int
    def height(self) -> int
    def setData(self, data: bytes) -> None
```

---

## QgsRasterCalculator

```python
from qgis.analysis import QgsRasterCalculator, QgsRasterCalculatorEntry

class QgsRasterCalculatorEntry:
    ref: str                     # Reference name used in expression, e.g. 'dem@1'
    raster: QgsRasterLayer       # Source raster layer
    bandNumber: int              # Band number (1-indexed)

class QgsRasterCalculator:
    def __init__(self,
                 formulaString: str,           # Expression with refs in double quotes
                 outputFile: str,              # Output file path
                 outputFormat: str,            # e.g. 'GTiff'
                 outputExtent: QgsRectangle,   # Output extent
                 nOutputColumns: int,          # Output width in pixels
                 nOutputRows: int,             # Output height in pixels
                 rasterEntries: List[QgsRasterCalculatorEntry]) -> None

    def processCalculation(self, feedback: QgsRasterCalcFeedback = None) -> int
    # Returns: 0=Success, 1=CreateOutputError, 2=InputLayerError,
    #          3=Canceled, 4=ParserError, 5=MemoryError, 6=BandError
```

---

## QgsRasterBandStats

```python
class QgsRasterBandStats:
    All = 63                    # Flag to compute all statistics
    minimumValue: float
    maximumValue: float
    mean: float
    stdDev: float
    sum: float
    range: float
    elementCount: int
    sumOfSquares: float
```

---

## QgsRasterIdentifyResult

```python
class QgsRasterIdentifyResult:
    def isValid(self) -> bool
    def results(self) -> Dict[int, Any]    # {bandNo: value}
    def error(self) -> QgsError
```

---

## Raster Renderers

### QgsSingleBandGrayRenderer

```python
class QgsSingleBandGrayRenderer(QgsRasterRenderer):
    def __init__(self, input: QgsRasterInterface, grayBand: int) -> None
    def setContrastEnhancement(self, ce: QgsContrastEnhancement) -> None
    def setGradient(self, gradient: int) -> None  # 0=BlackToWhite, 1=WhiteToBlack
```

### QgsSingleBandPseudoColorRenderer

```python
class QgsSingleBandPseudoColorRenderer(QgsRasterRenderer):
    def __init__(self, input: QgsRasterInterface, band: int,
                 shader: QgsRasterShader) -> None
    def shader(self) -> QgsRasterShader
    def setBand(self, bandNo: int) -> None
```

### QgsMultiBandColorRenderer

```python
class QgsMultiBandColorRenderer(QgsRasterRenderer):
    def __init__(self, input: QgsRasterInterface,
                 redBand: int, greenBand: int, blueBand: int) -> None
    def setRedBand(self, band: int) -> None
    def setGreenBand(self, band: int) -> None
    def setBlueBand(self, band: int) -> None
    def setRedContrastEnhancement(self, ce: QgsContrastEnhancement) -> None
    def setGreenContrastEnhancement(self, ce: QgsContrastEnhancement) -> None
    def setBlueContrastEnhancement(self, ce: QgsContrastEnhancement) -> None
```

### QgsPalettedRasterRenderer

```python
class QgsPalettedRasterRenderer(QgsRasterRenderer):
    def __init__(self, input: QgsRasterInterface, bandNumber: int,
                 classes: List[QgsPalettedRasterRenderer.Class]) -> None

    class Class:
        def __init__(self, value: float, color: QColor, label: str = '') -> None
```

### QgsHillshadeRenderer

```python
class QgsHillshadeRenderer(QgsRasterRenderer):
    def __init__(self, input: QgsRasterInterface, band: int,
                 lightAzimuth: float, lightAngle: float) -> None
    def setAzimuth(self, azimuth: float) -> None
    def setAltitude(self, altitude: float) -> None
    def setZFactor(self, factor: float) -> None
    def setMultiDirectional(self, isMultiDirectional: bool) -> None
```

---

## QgsRasterShader / QgsColorRampShader

```python
class QgsRasterShader:
    def setRasterShaderFunction(self, function: QgsRasterShaderFunction) -> None

class QgsColorRampShader(QgsRasterShaderFunction):
    Interpolated = 0
    Discrete = 1
    Exact = 2

    def setColorRampType(self, type: int) -> None
    def setColorRampItemList(self, items: List[QgsColorRampShader.ColorRampItem]) -> None

    class ColorRampItem:
        def __init__(self, value: float, color: QColor, label: str = '') -> None
```

---

## QgsContrastEnhancement

```python
class QgsContrastEnhancement:
    NoEnhancement = 0
    StretchToMinimumMaximum = 1
    StretchAndClipToMinimumMaximum = 2
    ClipToMinimumMaximum = 3

    def __init__(self, dataType: Qgis.DataType) -> None
    def setContrastEnhancementAlgorithm(self, algorithm: int) -> None
    def setMinimumValue(self, value: float) -> None
    def setMaximumValue(self, value: float) -> None
```

---

## Processing Algorithm Parameter Reference

### gdal:warpreproject

| Parameter | Type | Description |
|-----------|------|-------------|
| INPUT | raster | Input raster layer |
| SOURCE_CRS | crs | Source CRS (empty = layer CRS) |
| TARGET_CRS | crs | Target CRS |
| RESAMPLING | enum | 0=Nearest, 1=Bilinear, 2=Cubic, 3=CubicSpline, 4=Lanczos |
| NODATA | number | NoData value for output |
| TARGET_RESOLUTION | number | Output resolution (empty = same as input) |
| OPTIONS | string | GDAL creation options (e.g., 'COMPRESS=DEFLATE') |
| OUTPUT | output | Output raster path |

### gdal:hillshade

| Parameter | Type | Description |
|-----------|------|-------------|
| INPUT | raster | Input DEM |
| BAND | number | Band to use (default 1) |
| Z_FACTOR | number | Vertical exaggeration (default 1) |
| SCALE | number | Ratio of vertical to horizontal units (default 1) |
| AZIMUTH | number | Sun azimuth in degrees (default 315) |
| ALTITUDE | number | Sun altitude in degrees (default 45) |
| OUTPUT | output | Output hillshade raster |

### gdal:slope

| Parameter | Type | Description |
|-----------|------|-------------|
| INPUT | raster | Input DEM |
| BAND | number | Band to use (default 1) |
| SCALE | number | Ratio of vertical to horizontal units |
| AS_PERCENT | bool | Output as percent instead of degrees |
| OUTPUT | output | Output slope raster |

### gdal:contour

| Parameter | Type | Description |
|-----------|------|-------------|
| INPUT | raster | Input DEM |
| BAND | number | Band to use (default 1) |
| INTERVAL | number | Contour interval |
| FIELD_NAME | string | Attribute field name for elevation (default 'ELEV') |
| OUTPUT | output | Output vector path |

### gdal:polygonize

| Parameter | Type | Description |
|-----------|------|-------------|
| INPUT | raster | Input classified raster |
| BAND | number | Band to use (default 1) |
| FIELD | string | Output field name (default 'DN') |
| EIGHT_CONNECTEDNESS | bool | Use 8-connectedness (default False) |
| OUTPUT | output | Output vector path |

### gdal:rasterize

| Parameter | Type | Description |
|-----------|------|-------------|
| INPUT | vector | Input vector layer |
| FIELD | string | Field with burn values |
| UNITS | enum | 0=Pixels, 1=Georeferenced units |
| WIDTH | number | Output width/pixel size |
| HEIGHT | number | Output height/pixel size |
| EXTENT | extent | Output extent |
| NODATA | number | NoData value |
| OUTPUT | output | Output raster path |

### gdal:translate

| Parameter | Type | Description |
|-----------|------|-------------|
| INPUT | raster | Input raster |
| TARGET_CRS | crs | Target CRS (empty = no change) |
| NODATA | number | NoData value |
| OPTIONS | string | GDAL creation options |
| OUTPUT | output | Output raster path |
