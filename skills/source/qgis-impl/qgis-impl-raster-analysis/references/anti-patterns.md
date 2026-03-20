# Anti-Patterns (Raster Analysis)

## 1. Ignoring NoData in Raster Calculator

```python
# WRONG: Division without NoData guard — produces NaN or infinity at NoData pixels
expression = '"nir@1" / "red@1"'

# CORRECT: Guard against zero and NoData propagation
expression = '("nir@1" - "red@1") / ("nir@1" + "red@1" + 0.0001)'
```

**WHY**: NoData values propagate through arithmetic. Dividing by a band that contains zero or NoData pixels produces corrupt output. ALWAYS add a small epsilon or use conditional expressions to handle edge cases.

---

## 2. Mismatched Entry Reference Names

```python
# WRONG: entry.ref does not match the expression reference
entry = QgsRasterCalculatorEntry()
entry.ref = 'my_dem@1'
entry.raster = rlayer
entry.bandNumber = 1

calc = QgsRasterCalculator(
    '"dem@1" * 2',  # "dem@1" does NOT match "my_dem@1"
    output_path, 'GTiff', extent, width, height, [entry]
)
# Result: all-zero output with NO error message
```

```python
# CORRECT: entry.ref MUST match the reference in the expression exactly
entry.ref = 'dem@1'
calc = QgsRasterCalculator('"dem@1" * 2', ...)
```

**WHY**: `QgsRasterCalculator` silently produces all-zero output when references do not match. There is no error or warning. ALWAYS verify that `entry.ref` strings match the double-quoted references in the expression.

---

## 3. Forgetting triggerRepaint After Setting Renderer

```python
# WRONG: Renderer set but canvas not updated
renderer = QgsSingleBandPseudoColorRenderer(provider, 1, shader)
rlayer.setRenderer(renderer)
# Map canvas still shows old rendering

# CORRECT: ALWAYS call triggerRepaint
rlayer.setRenderer(renderer)
rlayer.triggerRepaint()
```

**WHY**: `setRenderer()` changes the internal renderer object but does NOT trigger a canvas redraw. Without `triggerRepaint()`, the user sees stale visualization.

---

## 4. Using Band 0 Instead of Band 1

```python
# WRONG: Band numbering starts at 1, not 0
stats = provider.bandStatistics(0, QgsRasterBandStats.All)  # Crashes or returns garbage

# CORRECT: Bands are 1-indexed
stats = provider.bandStatistics(1, QgsRasterBandStats.All)
```

**WHY**: QGIS raster bands are 1-indexed. Band 0 does NOT exist. Using band 0 causes crashes or returns invalid statistics with no clear error message.

---

## 5. Wrong Data Type in ContrastEnhancement

```python
# WRONG: Hardcoding data type instead of reading from provider
ce = QgsContrastEnhancement(Qgis.Byte)  # Assumes 8-bit, but DEM may be Float32

# CORRECT: Read actual data type from provider
ce = QgsContrastEnhancement(rlayer.dataProvider().dataType(1))
```

**WHY**: Mismatched data types cause incorrect contrast stretching. A Float32 DEM treated as Byte clips all values to 0-255, destroying the data range.

---

## 6. Not Checking processCalculation Return Value

```python
# WRONG: Assuming calculation succeeded
calc.processCalculation()
result_layer = QgsRasterLayer(output_path, "Result")  # May load empty/corrupt file

# CORRECT: Check return code
result_code = calc.processCalculation()
if result_code != 0:
    error_map = {
        1: "Failed to create output file",
        2: "Input layer error",
        3: "Calculation canceled",
        4: "Expression parser error",
        5: "Memory allocation error",
        6: "Band number error"
    }
    raise RuntimeError(f"Raster calc failed: {error_map.get(result_code, 'Unknown')}")
```

**WHY**: `processCalculation()` returns an integer error code, not a boolean. Code 0 means success. Non-zero values indicate specific failures, but the method does NOT raise exceptions.

---

## 7. Using 'TEMPORARY_OUTPUT' for GDAL Algorithms

```python
# WRONG: TEMPORARY_OUTPUT is for native algorithms, not GDAL
result = processing.run("gdal:hillshade", {
    'INPUT': dem_layer,
    'OUTPUT': 'TEMPORARY_OUTPUT'  # May fail or produce unexpected path
})

# CORRECT: Use explicit path for GDAL algorithms
result = processing.run("gdal:hillshade", {
    'INPUT': dem_layer,
    'OUTPUT': '/tmp/hillshade.tif'
})

# OR use memory: for native algorithms
result = processing.run("native:hillshade", {
    'INPUT': dem_layer,
    'OUTPUT': 'memory:'
})
```

**WHY**: GDAL algorithms write to disk via GDAL drivers. They do NOT support in-memory outputs the same way native algorithms do. ALWAYS provide a file path for GDAL algorithms, or use the equivalent `native:` algorithm with `'memory:'`.

---

## 8. Not Validating Raster Layer Before Use

```python
# WRONG: Using layer without validity check
rlayer = QgsRasterLayer("/nonexistent/path.tif", "DEM")
stats = rlayer.dataProvider().bandStatistics(1)  # Crashes: provider is None

# CORRECT: ALWAYS validate before use
rlayer = QgsRasterLayer("/data/dem.tif", "DEM")
if not rlayer.isValid():
    raise ValueError(f"Failed to load: {rlayer.error().message()}")
```

**WHY**: An invalid `QgsRasterLayer` has a `None` data provider. Any method call on the provider causes an `AttributeError` or segfault with no useful error context.

---

## 9. Resampling Method Mismatch

```python
# WRONG: Using nearest neighbor for continuous DEM data
result = processing.run("gdal:warpreproject", {
    'INPUT': dem_layer,
    'TARGET_CRS': 'EPSG:28992',
    'RESAMPLING': 0,  # Nearest neighbor — creates staircase artifacts in DEMs
    'OUTPUT': '/output/dem_rd.tif'
})

# CORRECT: Use bilinear or cubic for continuous data
result = processing.run("gdal:warpreproject", {
    'INPUT': dem_layer,
    'TARGET_CRS': 'EPSG:28992',
    'RESAMPLING': 1,  # Bilinear — smooth interpolation for continuous data
    'OUTPUT': '/output/dem_rd.tif'
})
```

**WHY**: Nearest neighbor resampling preserves exact pixel values (correct for categorical data like land use), but creates visible staircase artifacts in continuous data like DEMs and temperature grids. ALWAYS use bilinear (1) or cubic (2) resampling for continuous rasters, and nearest neighbor (0) for categorical/classified rasters.

---

## 10. Loading Result Without Checking Output Path

```python
# WRONG: Assuming result dict key matches expected output
result = processing.run("gdal:hillshade", {
    'INPUT': '/data/dem.tif',
    'OUTPUT': '/output/hillshade.tif'
})
layer = QgsRasterLayer('/output/hillshade.tif', 'Hillshade')  # Fragile

# CORRECT: Use the result dictionary to get actual output path
result = processing.run("gdal:hillshade", {
    'INPUT': '/data/dem.tif',
    'OUTPUT': '/output/hillshade.tif'
})
layer = QgsRasterLayer(result['OUTPUT'], 'Hillshade')
if not layer.isValid():
    raise ValueError("Hillshade output failed to load")
```

**WHY**: The actual output path may differ from the requested path (e.g., GDAL may append extensions or modify the path). ALWAYS use `result['OUTPUT']` to get the confirmed output path.
