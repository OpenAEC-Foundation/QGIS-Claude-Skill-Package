---
name: qgis-impl-raster-analysis
description: >
  Use when performing raster analysis: map algebra, terrain analysis, raster classification, or DEM processing.
  Prevents NoData handling errors and incorrect raster calculator expressions.
  Covers QgsRasterCalculator, DEM analysis (hillshade/slope/aspect), raster renderers, and GDAL processing algorithms.
  Keywords: raster analysis, QgsRasterCalculator, DEM, hillshade, slope, aspect, raster renderer, terrain, map algebra.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-impl-raster-analysis

## Quick Reference

### Raster Layer Properties

| Property | Method | Returns |
|----------|--------|---------|
| Dimensions | `rlayer.width()`, `rlayer.height()` | Pixel count |
| Extent | `rlayer.extent()` | `QgsRectangle` |
| Band count | `rlayer.bandCount()` | `int` |
| Band name | `rlayer.bandName(bandNo)` | `str` |
| Raster type | `rlayer.rasterType()` | 0=GrayOrUndefined, 1=Palette, 2=Multiband |
| CRS | `rlayer.crs()` | `QgsCoordinateReferenceSystem` |
| NoData value | `rlayer.dataProvider().sourceNoDataValue(band)` | `float` |

### Renderer Types

| Renderer Class | Use Case |
|----------------|----------|
| `QgsSingleBandGrayRenderer` | Single-band grayscale (DEMs, single-channel) |
| `QgsSingleBandPseudoColorRenderer` | Color ramp visualization (elevation, temperature) |
| `QgsMultiBandColorRenderer` | RGB composite (satellite imagery) |
| `QgsPalettedRasterRenderer` | Classified/categorical raster (land use) |
| `QgsHillshadeRenderer` | Live hillshade rendering from DEM |

### Terrain Analysis Algorithms

| Algorithm ID | Output |
|-------------|--------|
| `native:hillshade` | Shaded relief raster |
| `native:slope` | Slope in degrees |
| `native:aspect` | Aspect in degrees (0-360) |
| `native:ruggednessindex` | Terrain ruggedness |
| `gdal:hillshade` | GDAL hillshade (more options) |
| `gdal:slope` | GDAL slope (percent option) |
| `gdal:aspect` | GDAL aspect |
| `gdal:contour` | Contour lines from DEM |
| `gdal:roughness` | Terrain roughness |
| `gdal:tri` | Terrain ruggedness index |
| `gdal:tpi` | Topographic position index |

### Key GDAL Processing Algorithms

| Algorithm ID | Purpose |
|-------------|---------|
| `gdal:warpreproject` | Raster reprojection (gdalwarp) |
| `gdal:translate` | Format conversion, subsetting, compression |
| `gdal:polygonize` | Raster to vector polygons |
| `gdal:rasterize` | Vector to raster (burn values) |
| `gdal:merge` | Merge multiple rasters |
| `gdal:buildvirtualraster` | Create VRT mosaic |
| `gdal:fillnodata` | Fill NoData gaps |
| `gdal:rastercalculator` | GDAL raster calculator |

---

## Critical Warnings

**NEVER** ignore NoData values in raster calculations. NoData pixels propagate through arithmetic -- a single NoData input produces NoData output. ALWAYS check and set NoData handling explicitly.

**NEVER** use `QgsRasterCalculator` without verifying that `entry.ref` matches the reference string in the expression (e.g., `'dem@1'`). Mismatched references cause silent failures with all-zero output.

**NEVER** forget `rlayer.triggerRepaint()` after calling `rlayer.setRenderer()`. The map canvas does NOT update automatically.

**ALWAYS** check `rlayer.isValid()` after loading a raster layer. Invalid layers produce cryptic downstream errors.

**ALWAYS** verify `processCalculation()` returns `0` (success). Non-zero return values indicate errors but provide no error message.

**ALWAYS** match the data type in `QgsContrastEnhancement` to the raster band's actual data type via `provider.dataType(bandNo)`.

**NEVER** assume band numbering starts at 0. QGIS bands are 1-indexed. Band 0 does NOT exist.

**ALWAYS** use `'memory:'` (with colon) for in-memory processing outputs, NOT `'memory'` or `'TEMPORARY_OUTPUT'` for GDAL algorithms.

---

## Decision Tree

### Which Raster Analysis Approach?

```
Need raster math (add, multiply, threshold)?
├── Simple expression → QgsRasterCalculator (native PyQGIS)
├── Complex multi-band → gdal:rastercalculator (processing.run)
└── Pixel-by-pixel custom logic → QgsRasterBlock read/write loop

Need terrain derivatives?
├── Hillshade → processing.run("native:hillshade", ...) or "gdal:hillshade"
├── Slope → processing.run("native:slope", ...) or "gdal:slope"
├── Aspect → processing.run("native:aspect", ...) or "gdal:aspect"
├── Contours → processing.run("gdal:contour", ...)
└── Ruggedness → processing.run("native:ruggednessindex", ...)

Need format conversion?
├── Reproject → processing.run("gdal:warpreproject", ...)
├── Change format → processing.run("gdal:translate", ...)
├── Raster to vector → processing.run("gdal:polygonize", ...)
└── Vector to raster → processing.run("gdal:rasterize", ...)

Need visualization?
├── Single-band grayscale → QgsSingleBandGrayRenderer
├── Color ramp (elevation/temperature) → QgsSingleBandPseudoColorRenderer
├── RGB satellite → QgsMultiBandColorRenderer
├── Classified categories → QgsPalettedRasterRenderer
└── Live hillshade → QgsHillshadeRenderer
```

### Native vs GDAL Algorithms?

```
Use native: algorithms when:
├── Simpler parameter set is sufficient
├── In-memory output needed (memory:)
└── Fewer dependencies preferred

Use gdal: algorithms when:
├── Need advanced options (SCALE, AS_PERCENT, creation options)
├── Need specific GDAL creation options (COMPRESS=DEFLATE)
├── Working with large rasters (GDAL is optimized for I/O)
└── Need format-specific features
```

---

## Essential Patterns

### Load and Validate Raster Layer

```python
from qgis.core import QgsRasterLayer, QgsProject

rlayer = QgsRasterLayer("/path/to/dem.tif", "DEM")
if not rlayer.isValid():
    raise ValueError(f"Raster layer failed to load: {rlayer.error().message()}")

QgsProject.instance().addMapLayer(rlayer)
```

### Query Pixel Values

```python
# Single band, single point
provider = rlayer.dataProvider()
val, result = provider.sample(QgsPointXY(20.50, -34.0), 1)  # point, band
if result:
    print(f"Value: {val}")

# All bands at a point
from qgis.core import QgsRaster
ident = provider.identify(QgsPointXY(20.5, -34.0), QgsRaster.IdentifyFormatValue)
if ident.isValid():
    print(ident.results())  # {1: 323.0, 2: 127.0, ...}
```

### Raster Calculator (Map Algebra)

```python
from qgis.analysis import QgsRasterCalculator, QgsRasterCalculatorEntry

entries = []
entry = QgsRasterCalculatorEntry()
entry.ref = 'dem@1'
entry.raster = rlayer
entry.bandNumber = 1
entries.append(entry)

calc = QgsRasterCalculator(
    '"dem@1" * 2 + 100',         # Expression (refs in double quotes)
    '/path/to/output.tif',       # Output path
    'GTiff',                     # Output format
    rlayer.extent(),             # Output extent
    rlayer.width(),              # Output columns
    rlayer.height(),             # Output rows
    entries                      # List of entries
)

result = calc.processCalculation()
if result != 0:
    raise RuntimeError(f"Raster calculation failed with code {result}")
```

### Band Statistics

```python
from qgis.core import QgsRasterBandStats

stats = rlayer.dataProvider().bandStatistics(
    1,                           # Band number (1-indexed)
    QgsRasterBandStats.All,      # Compute all statistics
    rlayer.extent(),             # Extent to analyze
    0                            # Sample size (0 = all pixels)
)
print(f"Min: {stats.minimumValue}, Max: {stats.maximumValue}")
print(f"Mean: {stats.mean}, StdDev: {stats.stdDev}")
print(f"Sum: {stats.sum}, Range: {stats.range}")
```

### Raster Block Access (Pixel-Level)

```python
provider = rlayer.dataProvider()
block = provider.block(1, rlayer.extent(), rlayer.width(), rlayer.height())

for row in range(block.height()):
    for col in range(block.width()):
        if not block.isNoData(row, col):
            value = block.value(row, col)
            # Process pixel value
```

---

## Common Operations

### Terrain Analysis via Processing

```python
import processing

# Hillshade
result = processing.run("native:hillshade", {
    'INPUT': dem_layer,
    'Z_FACTOR': 1.0,
    'AZIMUTH': 315,
    'V_ANGLE': 45,
    'OUTPUT': 'memory:'
})
hillshade_layer = result['OUTPUT']

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
```

### GDAL Terrain Analysis (Advanced Options)

```python
# GDAL hillshade with extra controls
result = processing.run("gdal:hillshade", {
    'INPUT': '/data/dem.tif',
    'BAND': 1,
    'Z_FACTOR': 1,
    'SCALE': 1,
    'AZIMUTH': 315,
    'ALTITUDE': 45,
    'OUTPUT': '/output/hillshade.tif'
})

# GDAL slope with percent option
result = processing.run("gdal:slope", {
    'INPUT': '/data/dem.tif',
    'BAND': 1,
    'SCALE': 1,
    'AS_PERCENT': False,
    'OUTPUT': '/output/slope.tif'
})
```

### Contour Lines from DEM

```python
result = processing.run("gdal:contour", {
    'INPUT': '/data/dem.tif',
    'BAND': 1,
    'INTERVAL': 10,
    'FIELD_NAME': 'ELEV',
    'OUTPUT': '/output/contours.gpkg'
})
```

### Raster Reprojection

```python
result = processing.run("gdal:warpreproject", {
    'INPUT': rlayer,
    'SOURCE_CRS': 'EPSG:4326',
    'TARGET_CRS': 'EPSG:28992',
    'RESAMPLING': 0,        # 0=Nearest, 1=Bilinear, 2=Cubic
    'NODATA': -9999,
    'TARGET_RESOLUTION': 25,
    'OUTPUT': '/output/reprojected.tif'
})
```

### Set Single Band Pseudocolor Renderer

```python
from qgis.core import (QgsSingleBandPseudoColorRenderer,
                        QgsRasterShader, QgsColorRampShader)
from qgis.PyQt.QtGui import QColor

fcn = QgsColorRampShader()
fcn.setColorRampType(QgsColorRampShader.Interpolated)
lst = [
    QgsColorRampShader.ColorRampItem(0, QColor(0, 255, 0), "Low"),
    QgsColorRampShader.ColorRampItem(500, QColor(255, 255, 0), "Medium"),
    QgsColorRampShader.ColorRampItem(1000, QColor(255, 0, 0), "High"),
]
fcn.setColorRampItemList(lst)

shader = QgsRasterShader()
shader.setRasterShaderFunction(fcn)

renderer = QgsSingleBandPseudoColorRenderer(rlayer.dataProvider(), 1, shader)
rlayer.setRenderer(renderer)
rlayer.triggerRepaint()
```

### Set Multiband Color Renderer (RGB)

```python
from qgis.core import QgsMultiBandColorRenderer

renderer = QgsMultiBandColorRenderer(rlayer.dataProvider(), 3, 2, 1)
# Arguments: provider, redBand, greenBand, blueBand
rlayer.setRenderer(renderer)
rlayer.triggerRepaint()
```

### Raster to Vector (Polygonize)

```python
result = processing.run("gdal:polygonize", {
    'INPUT': '/data/classified.tif',
    'BAND': 1,
    'FIELD': 'DN',
    'EIGHT_CONNECTEDNESS': False,
    'OUTPUT': '/output/polygons.gpkg'
})
```

### Vector to Raster (Rasterize)

```python
result = processing.run("gdal:rasterize", {
    'INPUT': '/data/buildings.gpkg',
    'FIELD': 'height',
    'UNITS': 1,       # 1=Georeferenced units
    'WIDTH': 1,       # Pixel width in georef units
    'HEIGHT': 1,      # Pixel height in georef units
    'EXTENT': layer.extent(),
    'OUTPUT': '/output/buildings_raster.tif'
})
```

### Merge and Mosaic Rasters

```python
# Virtual raster (lightweight, no data copy)
result = processing.run("gdal:buildvirtualraster", {
    'INPUT': ['/data/tile1.tif', '/data/tile2.tif'],
    'RESOLUTION': 0,  # 0=Average, 1=Highest, 2=Lowest
    'OUTPUT': '/output/mosaic.vrt'
})

# Physical merge
result = processing.run("gdal:merge", {
    'INPUT': ['/data/tile1.tif', '/data/tile2.tif'],
    'NODATA_INPUT': -9999,
    'NODATA_OUTPUT': -9999,
    'OUTPUT': '/output/merged.tif'
})
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for QgsRasterCalculator, renderers, terrain classes
- [references/examples.md](references/examples.md) -- Complete raster analysis workflows
- [references/anti-patterns.md](references/anti-patterns.md) -- Raster analysis pitfalls and fixes

### Official Sources

- https://qgis.org/pyqgis/master/core/QgsRasterLayer.html
- https://qgis.org/pyqgis/master/analysis/QgsRasterCalculator.html
- https://qgis.org/pyqgis/master/core/QgsRasterRenderer.html
- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/raster.html
- https://docs.qgis.org/latest/en/docs/user_manual/processing_algs/gdal/rasteranalysis.html
