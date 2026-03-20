---
name: qgis-errors-projections
description: >
  Use when diagnosing CRS/projection errors, coordinate mismatches, or datum transformation issues.
  Prevents silent data corruption from wrong CRS assignment and coordinate order confusion.
  Covers wrong EPSG selection, lat/lon vs lon/lat, datum warnings, on-the-fly reprojection pitfalls, and fix patterns.
  Keywords: CRS error, projection error, wrong EPSG, coordinate mismatch, datum transformation, lat lon order, reprojection.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-errors-projections

## Quick Reference

### CRS Error Decision Table

| Symptom | Likely Cause | Jump To |
|---------|-------------|---------|
| Layer appears in wrong continent/ocean | Wrong EPSG code assigned | ERR-001 |
| Layer offset by ~100-200 meters | Missing datum transformation | ERR-003 |
| Layer appears stretched/squashed | Geographic CRS used for projected operations | ERR-001 |
| Coordinates swapped (X/Y flipped) | lat/lon vs lon/lat confusion | ERR-002 |
| Layer invisible (but valid) | CRS mismatch hidden by OTF reprojection | ERR-004 |
| Geometry operations return wrong values | Silent CRS corruption from wrong assignment | ERR-005 |
| Transform produces NaN or inf | QgsCoordinateTransform without context | ERR-006 |
| All CRS operations fail in standalone script | Missing srs.db (prefix path not set) | ERR-007 |

### Critical Warnings

**NEVER** assign a CRS to "fix" misplaced data -- assigning a CRS changes interpretation, NOT coordinates. Use `QgsCoordinateTransform` to reproject.

**NEVER** create `QgsCoordinateTransform` without a `QgsCoordinateTransformContext` -- the bare constructor ignores datum transformation preferences and silently degrades accuracy.

**NEVER** assume `QgsPointXY(x, y)` matches the axis order of the EPSG definition -- QGIS ALWAYS uses lon/lat (x/y) order internally, regardless of the EPSG standard axis order.

**NEVER** ignore `crs.isValid() == False` -- an invalid CRS propagates silently through ALL downstream operations, producing wrong results without errors.

**ALWAYS** use `QgsProject.instance().transformContext()` for coordinate transforms -- it contains user-configured datum transformation preferences.

**ALWAYS** verify CRS validity immediately after creation with `crs.isValid()`.

**ALWAYS** check `crs.isGeographic()` before performing distance/area calculations -- geographic CRS units are degrees, not meters.

---

## Error Catalog

### ERR-001: Wrong EPSG Code Selected

**Symptoms:**
- Layer appears in the wrong geographic location (wrong continent, ocean, or hemisphere)
- Layer appears extremely stretched or compressed
- Coordinates have unexpected magnitude (e.g., values in millions when expecting degrees)

**Root Cause:**
The wrong EPSG code was assigned to the layer. Common confusions:
- EPSG:4326 (WGS 84, degrees) vs EPSG:3857 (Web Mercator, meters)
- EPSG:32632 (UTM zone 32N) vs EPSG:32633 (UTM zone 33N) -- wrong UTM zone
- EPSG:28992 (Amersfoort / RD New) vs EPSG:4326 -- national CRS vs global

**Reproduction:**
```python
from qgis.core import QgsVectorLayer, QgsCoordinateReferenceSystem

# Data is actually in EPSG:28992 (Dutch RD New, meters)
# But loaded with wrong CRS
layer = QgsVectorLayer("points.shp", "points", "ogr")
layer.setCrs(QgsCoordinateReferenceSystem("EPSG:4326"))
# Layer now interprets meter coordinates as degrees -- appears near 0,0 (Gulf of Guinea)
```

**Fix Pattern:**
```python
from qgis.core import QgsCoordinateReferenceSystem

# Step 1: Identify the correct CRS by examining coordinate ranges
layer = QgsVectorLayer("points.shp", "points", "ogr")
extent = layer.extent()
print(f"X range: {extent.xMinimum()} to {extent.xMaximum()}")
print(f"Y range: {extent.yMinimum()} to {extent.yMaximum()}")

# Step 2: Match coordinate ranges to CRS
# Degrees (-180 to 180, -90 to 90) → geographic CRS (EPSG:4326)
# Large numbers (100000-300000, 300000-700000) → Dutch RD (EPSG:28992)
# Large numbers (166000-834000, 0-9400000) → UTM zones

# Step 3: Assign the CORRECT CRS
correct_crs = QgsCoordinateReferenceSystem("EPSG:28992")
if correct_crs.isValid():
    layer.setCrs(correct_crs)
```

**Diagnostic Table: Coordinate Range to CRS**

| X Range | Y Range | Likely CRS |
|---------|---------|------------|
| -180 to 180 | -90 to 90 | EPSG:4326 (WGS 84) |
| -20026376 to 20026376 | -20048966 to 20048966 | EPSG:3857 (Web Mercator) |
| 100000 to 300000 | 300000 to 700000 | EPSG:28992 (Dutch RD New) |
| 166000 to 834000 | 0 to 9400000 | EPSG:326xx (UTM North) |
| 166000 to 834000 | 1100000 to 10000000 | EPSG:327xx (UTM South) |

---

### ERR-002: Lat/Lon vs Lon/Lat Coordinate Order Confusion

**Symptoms:**
- Points appear reflected across the line y=x (mirrored along diagonal)
- A point expected at Amsterdam (lon=4.9, lat=52.4) appears near Somalia (lat=4.9, lon=52.4)
- WMS 1.3.0 service returns data in unexpected positions

**Root Cause:**
The EPSG database defines EPSG:4326 with axis order latitude, longitude (north, east). However, QGIS internally ALWAYS uses longitude, latitude (x, y) order. Confusion arises when:
- Importing CSV data where columns are labeled lat/lon
- Parsing WMS 1.3.0 responses (which follow the EPSG axis order)
- Reading GeoJSON (which specifies lon/lat per RFC 7946)

**Reproduction:**
```python
from qgis.core import QgsPointXY

# Amsterdam: latitude=52.37, longitude=4.90
# WRONG -- putting latitude in x parameter
wrong_point = QgsPointXY(52.37, 4.90)   # Places point near Somalia

# CORRECT -- QgsPointXY(x=longitude, y=latitude)
correct_point = QgsPointXY(4.90, 52.37)  # Places point in Amsterdam
```

**Fix Pattern:**
```python
from qgis.core import QgsPointXY, QgsVectorLayer, QgsFeature, QgsGeometry

# When importing from CSV with lat/lon columns:
# ALWAYS map: x=longitude_column, y=latitude_column
uri = "file:///path/to/data.csv?delimiter=,&xField=longitude&yField=latitude&crs=EPSG:4326"
layer = QgsVectorLayer(uri, "csv_points", "delimitedtext")

# When constructing points programmatically:
# ALWAYS use QgsPointXY(longitude, latitude) for EPSG:4326
lat, lon = 52.37, 4.90  # From external source
point = QgsPointXY(lon, lat)  # x=lon, y=lat
```

**Rule:** In QGIS, `QgsPointXY(x, y)` ALWAYS means `QgsPointXY(easting/longitude, northing/latitude)`, regardless of the EPSG axis order definition.

---

### ERR-003: Missing Datum Transformation

**Symptoms:**
- Coordinates are offset by 50-200 meters after transformation
- QGIS shows a datum transformation warning dialog
- Transformed points do not align with reference data

**Root Cause:**
Transforming between CRS that use different geodetic datums (e.g., ED50 to WGS 84, NAD27 to NAD83) requires a datum transformation. Without the correct transformation grid files (Proj grids), QGIS uses a fallback that introduces positional errors of 50-200+ meters.

**Reproduction:**
```python
from qgis.core import (
    QgsCoordinateReferenceSystem, QgsCoordinateTransform,
    QgsCoordinateTransformContext, QgsPointXY
)

# Transform from ED50 to WGS 84 WITHOUT project context
crs_ed50 = QgsCoordinateReferenceSystem("EPSG:4230")
crs_wgs84 = QgsCoordinateReferenceSystem("EPSG:4326")

# WRONG -- bare context without datum transformation preferences
bare_context = QgsCoordinateTransformContext()
xform = QgsCoordinateTransform(crs_ed50, crs_wgs84, bare_context)
result = xform.transform(QgsPointXY(5.0, 52.0))
# Result may be off by 50-200 meters due to missing grid
```

**Fix Pattern:**
```python
from qgis.core import (
    QgsCoordinateReferenceSystem, QgsCoordinateTransform,
    QgsProject, QgsPointXY
)

crs_ed50 = QgsCoordinateReferenceSystem("EPSG:4230")
crs_wgs84 = QgsCoordinateReferenceSystem("EPSG:4326")

# CORRECT -- use project transform context (includes user datum preferences)
context = QgsProject.instance().transformContext()
xform = QgsCoordinateTransform(crs_ed50, crs_wgs84, context)

# Verify the transform is valid
if not xform.isValid():
    raise RuntimeError("Coordinate transform is invalid -- check CRS definitions")

result = xform.transform(QgsPointXY(5.0, 52.0))

# For high-accuracy work, verify grid availability
if not xform.isShortCircuited():
    print("Full datum transformation pipeline is active")
```

---

### ERR-004: On-the-Fly Reprojection Hiding CRS Mismatches

**Symptoms:**
- Layers appear correctly aligned on the map canvas
- But spatial operations (intersect, buffer, distance) produce wrong results
- Exported data has unexpected coordinate values
- Geoprocessing results are offset or geometrically incorrect

**Root Cause:**
QGIS on-the-fly (OTF) reprojection reprojects layers for DISPLAY only. The underlying data retains its original CRS. Spatial operations use the actual stored coordinates, not the display coordinates. When two layers have different CRS, they look aligned but their raw coordinates are in different reference frames.

**Reproduction:**
```python
from qgis.core import QgsVectorLayer, QgsProject
import processing

# Layer A in EPSG:4326 (degrees)
layer_a = QgsVectorLayer("parcels_wgs84.gpkg", "parcels", "ogr")
# Layer B in EPSG:28992 (meters)
layer_b = QgsVectorLayer("buildings_rd.gpkg", "buildings", "ogr")

QgsProject.instance().addMapLayer(layer_a)
QgsProject.instance().addMapLayer(layer_b)
# Layers LOOK aligned on canvas due to OTF reprojection

# But intersection uses raw coordinates -- produces WRONG results
result = processing.run("native:intersection", {
    'INPUT': layer_a,
    'OVERLAY': layer_b,
    'OUTPUT': 'memory:'
})
# Result is empty or wrong because coordinates are in different CRS
```

**Fix Pattern:**
```python
from qgis.core import QgsCoordinateReferenceSystem
import processing

# ALWAYS reproject to a common CRS before geoprocessing
target_crs = QgsCoordinateReferenceSystem("EPSG:28992")

reprojected_a = processing.run("native:reprojectlayer", {
    'INPUT': layer_a,
    'TARGET_CRS': target_crs,
    'OUTPUT': 'memory:'
})['OUTPUT']

# Now run intersection with matching CRS
result = processing.run("native:intersection", {
    'INPUT': reprojected_a,
    'OVERLAY': layer_b,
    'OUTPUT': 'memory:'
})
```

**Rule:** ALWAYS ensure all input layers share the same CRS before running geoprocessing operations. OTF reprojection is for visualization only.

---

### ERR-005: Silent Data Corruption from Wrong CRS Assignment

**Symptoms:**
- Data appears correct visually (because OTF compensates)
- Exported/saved data has wrong coordinate values
- Area/distance calculations return absurd values
- Other GIS software cannot read the data correctly

**Root Cause:**
Calling `layer.setCrs()` changes how QGIS interprets the coordinates but does NOT modify the actual coordinate values. If the assigned CRS does not match the actual data, every operation that uses the CRS metadata produces wrong results.

**Reproduction:**
```python
from qgis.core import QgsVectorLayer, QgsCoordinateReferenceSystem

# Layer contains UTM coordinates (EPSG:32632, values like 500000, 5500000)
layer = QgsVectorLayer("data_utm32.gpkg", "data", "ogr")

# WRONG -- assigning WGS 84 to UTM data
layer.setCrs(QgsCoordinateReferenceSystem("EPSG:4326"))
# QGIS now thinks 500000 is a longitude value
# All downstream operations are silently corrupted
```

**Fix Pattern:**
```python
from qgis.core import (
    QgsVectorLayer, QgsCoordinateReferenceSystem,
    QgsCoordinateTransform, QgsProject
)
import processing

# To ACTUALLY change coordinates from one CRS to another, use reprojection:
layer = QgsVectorLayer("data_utm32.gpkg", "data", "ogr")

# Option 1: Processing algorithm
reprojected = processing.run("native:reprojectlayer", {
    'INPUT': layer,
    'TARGET_CRS': QgsCoordinateReferenceSystem("EPSG:4326"),
    'OUTPUT': 'memory:'
})['OUTPUT']

# Option 2: Manual transform on individual geometries
source_crs = QgsCoordinateReferenceSystem("EPSG:32632")
dest_crs = QgsCoordinateReferenceSystem("EPSG:4326")
xform = QgsCoordinateTransform(source_crs, dest_crs, QgsProject.instance().transformContext())

for feature in layer.getFeatures():
    geom = feature.geometry()
    geom.transform(xform)  # Modifies geometry in-place
    # Use the transformed geometry
```

**Rule:** `setCrs()` = change label. `QgsCoordinateTransform` = change coordinates. NEVER use `setCrs()` as a substitute for reprojection.

---

### ERR-006: QgsCoordinateTransform Without Proper Context

**Symptoms:**
- Transform produces coordinates with reduced accuracy
- No error or warning raised
- Results differ from authoritative transformation services

**Root Cause:**
Creating `QgsCoordinateTransform(crs1, crs2)` without a `QgsCoordinateTransformContext` uses an empty context that ignores user-configured datum transformation pipelines. This falls back to a default transformation that may be less accurate.

**Reproduction:**
```python
from qgis.core import QgsCoordinateReferenceSystem, QgsCoordinateTransform

crs1 = QgsCoordinateReferenceSystem("EPSG:4230")  # ED50
crs2 = QgsCoordinateReferenceSystem("EPSG:4326")  # WGS 84

# WRONG -- no context, uses fallback transformation
xform = QgsCoordinateTransform(crs1, crs2)
```

**Fix Pattern:**
```python
from qgis.core import (
    QgsCoordinateReferenceSystem, QgsCoordinateTransform, QgsProject
)

crs1 = QgsCoordinateReferenceSystem("EPSG:4230")
crs2 = QgsCoordinateReferenceSystem("EPSG:4326")

# CORRECT -- with project transform context
xform = QgsCoordinateTransform(crs1, crs2, QgsProject.instance().transformContext())

# For standalone scripts without a project:
from qgis.core import QgsCoordinateTransformContext
context = QgsCoordinateTransformContext()
# Optionally add specific datum operations:
# context.addCoordinateOperation(crs1, crs2, "proj_pipeline_string")
xform = QgsCoordinateTransform(crs1, crs2, context)
```

---

### ERR-007: Missing srs.db in Standalone Scripts

**Symptoms:**
- ALL CRS objects return `isValid() == False`
- `QgsCoordinateReferenceSystem("EPSG:4326")` creates an invalid CRS
- Works fine inside QGIS but fails in standalone Python scripts

**Root Cause:**
Standalone scripts must initialize `QgsApplication` with the correct prefix path so QGIS can locate the `srs.db` SQLite database containing CRS definitions.

**Fix Pattern:**
```python
from qgis.core import QgsApplication, QgsCoordinateReferenceSystem

# ALWAYS initialize QgsApplication before any CRS operations
qgs = QgsApplication([], False)
qgs.setPrefixPath("/path/to/qgis", True)  # Platform-dependent path
qgs.initQgis()

# Now CRS operations work
crs = QgsCoordinateReferenceSystem("EPSG:4326")
assert crs.isValid(), "CRS creation failed -- check prefix path"

# Common prefix paths:
# Windows: "C:/Program Files/QGIS 3.x/apps/qgis"
# Linux:   "/usr"
# macOS:   "/Applications/QGIS.app/Contents/MacOS"

# ALWAYS clean up
qgs.exitQgis()
```

---

## Diagnostic Flowchart

```
START: Layer appears in wrong location or operations produce wrong results
  |
  +--> Is the layer valid? (layer.isValid())
  |      |
  |      +--> NO --> Fix data source path/provider first (not a CRS issue)
  |      |
  |      +--> YES --> Continue
  |
  +--> Is the CRS valid? (layer.crs().isValid())
  |      |
  |      +--> NO --> ERR-007 if standalone script, else assign correct CRS
  |      |
  |      +--> YES --> Continue
  |
  +--> Check coordinate ranges (layer.extent())
  |      |
  |      +--> Coordinates match expected CRS range?
  |             |
  |             +--> NO --> ERR-001 (wrong EPSG assigned)
  |             |
  |             +--> YES --> Continue
  |
  +--> Are X/Y values swapped? (lat in X, lon in Y)
  |      |
  |      +--> YES --> ERR-002 (axis order confusion)
  |      |
  |      +--> NO --> Continue
  |
  +--> Is the offset small (50-200m)?
  |      |
  |      +--> YES --> ERR-003 (datum transformation issue)
  |      |
  |      +--> NO --> Continue
  |
  +--> Do layers look aligned but operations fail?
  |      |
  |      +--> YES --> ERR-004 (OTF reprojection masking mismatch)
  |      |
  |      +--> NO --> Continue
  |
  +--> Was setCrs() used to "fix" placement?
         |
         +--> YES --> ERR-005 (silent corruption from wrong assignment)
         |
         +--> NO --> Check for ERR-006 (transform without context)
```

---

## Debugging Commands

Quick diagnostic commands to run in the QGIS Python console:

```python
from qgis.core import QgsProject

# Print CRS info for all layers
for name, layer in QgsProject.instance().mapLayers().items():
    crs = layer.crs()
    ext = layer.extent()
    print(f"Layer: {layer.name()}")
    print(f"  CRS: {crs.authid()} ({crs.description()})")
    print(f"  Valid: {crs.isValid()}")
    print(f"  Geographic: {crs.isGeographic()}")
    print(f"  Units: {crs.mapUnits()}")
    print(f"  Extent: X({ext.xMinimum():.2f} to {ext.xMaximum():.2f}), Y({ext.yMinimum():.2f} to {ext.yMaximum():.2f})")
    print()

# Print project CRS
proj_crs = QgsProject.instance().crs()
print(f"Project CRS: {proj_crs.authid()} ({proj_crs.description()})")
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- CRS and transform API signatures
- [references/examples.md](references/examples.md) -- Working fix pattern examples
- [references/anti-patterns.md](references/anti-patterns.md) -- What NOT to do with CRS operations

### Official Sources

- https://qgis.org/pyqgis/master/core/QgsCoordinateReferenceSystem.html
- https://qgis.org/pyqgis/master/core/QgsCoordinateTransform.html
- https://qgis.org/pyqgis/master/core/QgsCoordinateTransformContext.html
- https://docs.qgis.org/3.34/en/docs/pyqgis_developer_cookbook/crs.html
