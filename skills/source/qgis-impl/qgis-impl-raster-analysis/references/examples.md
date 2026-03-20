# Working Code Examples (Raster Analysis)

## Example 1: Load DEM and Compute Statistics

```python
from qgis.core import QgsRasterLayer, QgsRasterBandStats, QgsProject

# Load raster
rlayer = QgsRasterLayer("/data/dem.tif", "DEM")
if not rlayer.isValid():
    raise ValueError("Raster layer failed to load")

QgsProject.instance().addMapLayer(rlayer)

# Get statistics
provider = rlayer.dataProvider()
stats = provider.bandStatistics(1, QgsRasterBandStats.All, rlayer.extent(), 0)

print(f"Dimensions: {rlayer.width()} x {rlayer.height()} pixels")
print(f"Bands: {rlayer.bandCount()}")
print(f"Min elevation: {stats.minimumValue}")
print(f"Max elevation: {stats.maximumValue}")
print(f"Mean elevation: {stats.mean}")
print(f"Std deviation: {stats.stdDev}")
```

---

## Example 2: NDVI Calculation with Raster Calculator

```python
from qgis.core import QgsRasterLayer
from qgis.analysis import QgsRasterCalculator, QgsRasterCalculatorEntry

# Load multispectral image
rlayer = QgsRasterLayer("/data/sentinel2.tif", "Sentinel2")
if not rlayer.isValid():
    raise ValueError("Layer failed to load")

# Band 4 = Red, Band 8 = NIR (Sentinel-2)
entries = []

red_entry = QgsRasterCalculatorEntry()
red_entry.ref = 'sentinel@4'
red_entry.raster = rlayer
red_entry.bandNumber = 4
entries.append(red_entry)

nir_entry = QgsRasterCalculatorEntry()
nir_entry.ref = 'sentinel@8'
nir_entry.raster = rlayer
nir_entry.bandNumber = 8
entries.append(nir_entry)

# NDVI = (NIR - Red) / (NIR + Red)
expression = '("sentinel@8" - "sentinel@4") / ("sentinel@8" + "sentinel@4")'

calc = QgsRasterCalculator(
    expression,
    '/output/ndvi.tif',
    'GTiff',
    rlayer.extent(),
    rlayer.width(),
    rlayer.height(),
    entries
)

result = calc.processCalculation()
if result != 0:
    raise RuntimeError(f"NDVI calculation failed with code {result}")

# Load result
ndvi_layer = QgsRasterLayer('/output/ndvi.tif', 'NDVI')
QgsProject.instance().addMapLayer(ndvi_layer)
```

---

## Example 3: Complete Terrain Analysis Pipeline

```python
import processing
from qgis.core import QgsRasterLayer, QgsProject

dem_layer = QgsRasterLayer("/data/dem.tif", "DEM")
if not dem_layer.isValid():
    raise ValueError("DEM failed to load")

QgsProject.instance().addMapLayer(dem_layer)

# Hillshade
hillshade_result = processing.run("native:hillshade", {
    'INPUT': dem_layer,
    'Z_FACTOR': 1.0,
    'AZIMUTH': 315,
    'V_ANGLE': 45,
    'OUTPUT': 'memory:'
})
QgsProject.instance().addMapLayer(hillshade_result['OUTPUT'])
hillshade_result['OUTPUT'].setName('Hillshade')

# Slope
slope_result = processing.run("native:slope", {
    'INPUT': dem_layer,
    'Z_FACTOR': 1.0,
    'OUTPUT': 'memory:'
})
QgsProject.instance().addMapLayer(slope_result['OUTPUT'])
slope_result['OUTPUT'].setName('Slope')

# Aspect
aspect_result = processing.run("native:aspect", {
    'INPUT': dem_layer,
    'Z_FACTOR': 1.0,
    'OUTPUT': 'memory:'
})
QgsProject.instance().addMapLayer(aspect_result['OUTPUT'])
aspect_result['OUTPUT'].setName('Aspect')

# Contours (via GDAL)
contour_result = processing.run("gdal:contour", {
    'INPUT': '/data/dem.tif',
    'BAND': 1,
    'INTERVAL': 10,
    'FIELD_NAME': 'ELEV',
    'OUTPUT': '/output/contours.gpkg'
})
```

---

## Example 4: DEM Visualization with Pseudocolor Renderer

```python
from qgis.core import (QgsRasterLayer, QgsSingleBandPseudoColorRenderer,
                        QgsRasterShader, QgsColorRampShader, QgsRasterBandStats,
                        QgsProject)
from qgis.PyQt.QtGui import QColor

rlayer = QgsRasterLayer("/data/dem.tif", "DEM")
if not rlayer.isValid():
    raise ValueError("DEM failed to load")

QgsProject.instance().addMapLayer(rlayer)

# Get actual min/max from data
stats = rlayer.dataProvider().bandStatistics(1, QgsRasterBandStats.All, rlayer.extent(), 0)

# Build color ramp shader
fcn = QgsColorRampShader()
fcn.setColorRampType(QgsColorRampShader.Interpolated)
fcn.setColorRampItemList([
    QgsColorRampShader.ColorRampItem(stats.minimumValue, QColor(0, 100, 0), "Valley"),
    QgsColorRampShader.ColorRampItem(stats.mean, QColor(255, 255, 100), "Mid"),
    QgsColorRampShader.ColorRampItem(stats.maximumValue, QColor(139, 69, 19), "Peak"),
])

shader = QgsRasterShader()
shader.setRasterShaderFunction(fcn)

renderer = QgsSingleBandPseudoColorRenderer(rlayer.dataProvider(), 1, shader)
rlayer.setRenderer(renderer)
rlayer.triggerRepaint()
```

---

## Example 5: Satellite Image RGB Composite

```python
from qgis.core import (QgsRasterLayer, QgsMultiBandColorRenderer,
                        QgsContrastEnhancement, QgsProject)

rlayer = QgsRasterLayer("/data/satellite.tif", "Satellite")
if not rlayer.isValid():
    raise ValueError("Layer failed to load")

QgsProject.instance().addMapLayer(rlayer)

# True color: R=Band3, G=Band2, B=Band1 (common for Landsat)
renderer = QgsMultiBandColorRenderer(rlayer.dataProvider(), 3, 2, 1)

# Apply contrast enhancement per band
for band_no in [3, 2, 1]:
    ce = QgsContrastEnhancement(rlayer.dataProvider().dataType(band_no))
    ce.setContrastEnhancementAlgorithm(QgsContrastEnhancement.StretchToMinimumMaximum)
    stats = rlayer.dataProvider().bandStatistics(band_no)
    ce.setMinimumValue(stats.minimumValue)
    ce.setMaximumValue(stats.maximumValue)
    if band_no == 3:
        renderer.setRedContrastEnhancement(ce)
    elif band_no == 2:
        renderer.setGreenContrastEnhancement(ce)
    else:
        renderer.setBlueContrastEnhancement(ce)

rlayer.setRenderer(renderer)
rlayer.triggerRepaint()
```

---

## Example 6: Land Use Classification Visualization

```python
from qgis.core import (QgsRasterLayer, QgsPalettedRasterRenderer, QgsProject)
from qgis.PyQt.QtGui import QColor

rlayer = QgsRasterLayer("/data/landuse.tif", "Land Use")
if not rlayer.isValid():
    raise ValueError("Layer failed to load")

QgsProject.instance().addMapLayer(rlayer)

classes = [
    QgsPalettedRasterRenderer.Class(1, QColor(255, 0, 0), "Urban"),
    QgsPalettedRasterRenderer.Class(2, QColor(0, 128, 0), "Forest"),
    QgsPalettedRasterRenderer.Class(3, QColor(0, 0, 255), "Water"),
    QgsPalettedRasterRenderer.Class(4, QColor(255, 255, 0), "Agriculture"),
    QgsPalettedRasterRenderer.Class(5, QColor(128, 128, 128), "Bare Soil"),
]

renderer = QgsPalettedRasterRenderer(rlayer.dataProvider(), 1, classes)
rlayer.setRenderer(renderer)
rlayer.triggerRepaint()
```

---

## Example 7: Raster Reprojection and Compression

```python
import processing
from qgis.core import QgsRasterLayer, QgsProject

# Reproject from WGS84 to Dutch RD New with compression
result = processing.run("gdal:warpreproject", {
    'INPUT': '/data/dem_wgs84.tif',
    'SOURCE_CRS': 'EPSG:4326',
    'TARGET_CRS': 'EPSG:28992',
    'RESAMPLING': 1,               # Bilinear (good for continuous data like DEMs)
    'NODATA': -9999,
    'TARGET_RESOLUTION': 25,
    'OPTIONS': 'COMPRESS=DEFLATE|PREDICTOR=2|ZLEVEL=9',
    'OUTPUT': '/output/dem_rd.tif'
})

reprojected = QgsRasterLayer(result['OUTPUT'], 'DEM_RD')
QgsProject.instance().addMapLayer(reprojected)
```

---

## Example 8: Raster to Vector and Back

```python
import processing

# Polygonize a classified raster
poly_result = processing.run("gdal:polygonize", {
    'INPUT': '/data/classified.tif',
    'BAND': 1,
    'FIELD': 'class_id',
    'EIGHT_CONNECTEDNESS': False,
    'OUTPUT': '/output/classes.gpkg'
})

# Rasterize a vector layer
raster_result = processing.run("gdal:rasterize", {
    'INPUT': '/data/buildings.gpkg',
    'FIELD': 'height',
    'UNITS': 1,          # Georeferenced units
    'WIDTH': 0.5,        # 0.5m pixel width
    'HEIGHT': 0.5,       # 0.5m pixel height
    'NODATA': 0,
    'OUTPUT': '/output/building_heights.tif'
})
```

---

## Example 9: Pixel-Level Analysis with QgsRasterBlock

```python
from qgis.core import QgsRasterLayer, QgsRasterBandStats

rlayer = QgsRasterLayer("/data/dem.tif", "DEM")
provider = rlayer.dataProvider()

# Read entire raster as block
block = provider.block(1, rlayer.extent(), rlayer.width(), rlayer.height())

# Count pixels above threshold
threshold = 500.0
above_count = 0
total_valid = 0

for row in range(block.height()):
    for col in range(block.width()):
        if not block.isNoData(row, col):
            total_valid += 1
            if block.value(row, col) > threshold:
                above_count += 1

if total_valid > 0:
    pct = (above_count / total_valid) * 100
    print(f"Pixels above {threshold}m: {above_count} ({pct:.1f}%)")
```

---

## Example 10: Merge Raster Tiles with NoData Handling

```python
import processing

# Build virtual raster first (lightweight check)
vrt_result = processing.run("gdal:buildvirtualraster", {
    'INPUT': ['/data/tile_01.tif', '/data/tile_02.tif', '/data/tile_03.tif'],
    'RESOLUTION': 0,       # Average resolution
    'INPUT_NODATA_VALUE': -9999,
    'OUTPUT': '/output/mosaic.vrt'
})

# Physical merge with compression
merge_result = processing.run("gdal:merge", {
    'INPUT': ['/data/tile_01.tif', '/data/tile_02.tif', '/data/tile_03.tif'],
    'NODATA_INPUT': -9999,
    'NODATA_OUTPUT': -9999,
    'OPTIONS': 'COMPRESS=DEFLATE',
    'OUTPUT': '/output/mosaic.tif'
})
```
