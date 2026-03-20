# qgis-errors-projections — Anti-patterns

## AP-001: Using setCrs() to Reproject Data

**WRONG:**
```python
# Trying to "convert" a UTM layer to WGS 84 by changing its CRS label
layer = QgsVectorLayer("buildings_utm32.gpkg", "buildings", "ogr")
layer.setCrs(QgsCoordinateReferenceSystem("EPSG:4326"))
# Coordinates are still UTM meter values but now interpreted as degrees
# A coordinate like (500000, 5500000) is now treated as lon=500000, lat=5500000
```

**WHY:** `setCrs()` changes the CRS metadata only. It does NOT modify any coordinate values. The data is now silently corrupted -- every spatial operation will produce wrong results.

**CORRECT:**
```python
import processing
reprojected = processing.run("native:reprojectlayer", {
    'INPUT': layer,
    'TARGET_CRS': QgsCoordinateReferenceSystem("EPSG:4326"),
    'OUTPUT': 'memory:'
})['OUTPUT']
```

---

## AP-002: Creating QgsCoordinateTransform Without Context

**WRONG:**
```python
xform = QgsCoordinateTransform(crs_source, crs_dest)
# Missing QgsCoordinateTransformContext -- ignores datum transformation preferences
```

**WHY:** Without a transform context, QGIS uses a default transformation that may lack the correct datum shift parameters. This silently degrades accuracy by 50-200+ meters for cross-datum transforms.

**CORRECT:**
```python
xform = QgsCoordinateTransform(
    crs_source, crs_dest,
    QgsProject.instance().transformContext()
)
```

---

## AP-003: Putting Latitude in the X Parameter

**WRONG:**
```python
# Amsterdam: lat=52.37, lon=4.90
point = QgsPointXY(52.37, 4.90)  # X=52.37, Y=4.90 -- WRONG
# This places the point near Somalia, not Amsterdam
```

**WHY:** `QgsPointXY(x, y)` ALWAYS uses mathematical convention where X=easting/longitude and Y=northing/latitude. The EPSG:4326 standard axis order (lat, lon) does NOT apply to QgsPointXY construction.

**CORRECT:**
```python
point = QgsPointXY(4.90, 52.37)  # X=lon, Y=lat
```

---

## AP-004: Ignoring CRS Validity

**WRONG:**
```python
crs = QgsCoordinateReferenceSystem("EPSG:99999")
# No validity check -- crs.isValid() returns False
xform = QgsCoordinateTransform(crs, other_crs, context)
result = xform.transform(point)
# result contains garbage coordinates -- no error raised
```

**WHY:** An invalid CRS object does not raise exceptions when used. It silently produces wrong or undefined results in all downstream operations.

**CORRECT:**
```python
crs = QgsCoordinateReferenceSystem("EPSG:99999")
if not crs.isValid():
    raise ValueError("Invalid CRS: EPSG:99999")
```

---

## AP-005: Assuming OTF Reprojection Applies to Geoprocessing

**WRONG:**
```python
# Layer A is EPSG:4326, Layer B is EPSG:28992
# They look aligned on the canvas due to OTF reprojection
result = processing.run("native:intersection", {
    'INPUT': layer_a,    # EPSG:4326 (degrees)
    'OVERLAY': layer_b,  # EPSG:28992 (meters)
    'OUTPUT': 'memory:'
})
# Result is empty or wrong -- raw coordinates do not overlap
```

**WHY:** On-the-fly reprojection is a DISPLAY feature only. It reprojects for rendering on the map canvas. Geoprocessing algorithms operate on the actual stored coordinates. Mixing CRS produces meaningless results.

**CORRECT:**
```python
# Reproject to common CRS first
reprojected_a = processing.run("native:reprojectlayer", {
    'INPUT': layer_a,
    'TARGET_CRS': QgsCoordinateReferenceSystem("EPSG:28992"),
    'OUTPUT': 'memory:'
})['OUTPUT']

result = processing.run("native:intersection", {
    'INPUT': reprojected_a,
    'OVERLAY': layer_b,
    'OUTPUT': 'memory:'
})
```

---

## AP-006: Using Geographic CRS for Distance/Area Calculations

**WRONG:**
```python
from qgis.core import QgsDistanceArea, QgsCoordinateReferenceSystem, QgsPointXY

d = QgsDistanceArea()
# No CRS or ellipsoid set -- uses planar calculation on degree values
distance = d.measureLine(QgsPointXY(4.9, 52.3), QgsPointXY(5.1, 52.5))
# Returns a value in degrees, not meters -- meaningless for real-world distances
```

**WHY:** Without setting the source CRS and ellipsoid, `QgsDistanceArea` performs a simple Euclidean calculation on raw coordinate values. For geographic CRS, these values are in degrees, producing meaningless distance/area values.

**CORRECT:**
```python
from qgis.core import QgsDistanceArea, QgsCoordinateReferenceSystem, QgsPointXY, QgsProject

d = QgsDistanceArea()
d.setSourceCrs(
    QgsCoordinateReferenceSystem("EPSG:4326"),
    QgsProject.instance().transformContext()
)
d.setEllipsoid('WGS84')
distance = d.measureLine(QgsPointXY(4.9, 52.3), QgsPointXY(5.1, 52.5))
# Returns distance in meters using geodesic calculation
```

---

## AP-007: Hardcoding CRS Without Checking the Source Data

**WRONG:**
```python
# Assuming all shapefiles are WGS 84
layer = QgsVectorLayer("unknown_data.shp", "data", "ogr")
layer.setCrs(QgsCoordinateReferenceSystem("EPSG:4326"))
```

**WHY:** Shapefiles may use any CRS. The .prj file (if present) defines the CRS. Blindly assigning EPSG:4326 corrupts data that is in a different CRS. If the .prj is missing, examine the coordinate ranges to determine the correct CRS.

**CORRECT:**
```python
layer = QgsVectorLayer("unknown_data.shp", "data", "ogr")
crs = layer.crs()

if not crs.isValid():
    # CRS unknown -- examine coordinates to determine CRS
    ext = layer.extent()
    print(f"X: {ext.xMinimum()} to {ext.xMaximum()}")
    print(f"Y: {ext.yMinimum()} to {ext.yMaximum()}")
    # Use the diagnostic table from SKILL.md ERR-001 to identify the CRS
```

---

## AP-008: Not Initializing QgsApplication in Standalone Scripts

**WRONG:**
```python
from qgis.core import QgsCoordinateReferenceSystem

# No QgsApplication initialization
crs = QgsCoordinateReferenceSystem("EPSG:4326")
print(crs.isValid())  # False -- srs.db not found
```

**WHY:** Without `QgsApplication.initQgis()`, the CRS database (`srs.db`) is not loaded. ALL CRS objects will be invalid, and all coordinate operations will fail silently.

**CORRECT:**
```python
from qgis.core import QgsApplication, QgsCoordinateReferenceSystem

qgs = QgsApplication([], False)
qgs.setPrefixPath("/path/to/qgis", True)
qgs.initQgis()

crs = QgsCoordinateReferenceSystem("EPSG:4326")
print(crs.isValid())  # True

qgs.exitQgis()
```

---

## AP-009: Using Deprecated CRS Creation Methods

**WRONG:**
```python
crs = QgsCoordinateReferenceSystem()
crs.createFromSrid(4326)   # Deprecated
crs.createFromEpsg(4326)   # Deprecated
```

**WHY:** These methods are deprecated since QGIS 3.x and may be removed in QGIS 4.x. They also require two steps (construct + initialize) instead of one.

**CORRECT:**
```python
crs = QgsCoordinateReferenceSystem("EPSG:4326")
# Or use the static factory:
crs = QgsCoordinateReferenceSystem.fromEpsgId(4326)
```

---

## AP-010: Mixing CRS in Geometry Construction

**WRONG:**
```python
# Building a polygon with vertices from different CRS
from qgis.core import QgsGeometry, QgsPointXY

# Point A from WGS 84 (degrees)
p1 = QgsPointXY(4.9, 52.3)
# Point B from UTM (meters) -- different CRS!
p2 = QgsPointXY(500000, 5800000)

polygon = QgsGeometry.fromPolygonXY([[p1, p2, QgsPointXY(4.95, 52.35), p1]])
# Geometry is nonsensical -- mixing degrees and meters
```

**WHY:** `QgsPointXY` has no CRS awareness. Combining points from different coordinate reference systems produces geometrically meaningless results. ALWAYS transform all points to the same CRS before constructing geometries.

**CORRECT:**
```python
from qgis.core import (
    QgsCoordinateReferenceSystem, QgsCoordinateTransform,
    QgsProject, QgsPointXY, QgsGeometry
)

# Transform p2 from UTM to WGS 84 first
xform = QgsCoordinateTransform(
    QgsCoordinateReferenceSystem("EPSG:32632"),
    QgsCoordinateReferenceSystem("EPSG:4326"),
    QgsProject.instance().transformContext()
)
p2_wgs84 = xform.transform(QgsPointXY(500000, 5800000))

# Now all points are in the same CRS
p1 = QgsPointXY(4.9, 52.3)
polygon = QgsGeometry.fromPolygonXY([[p1, p2_wgs84, QgsPointXY(4.95, 52.35), p1]])
```
