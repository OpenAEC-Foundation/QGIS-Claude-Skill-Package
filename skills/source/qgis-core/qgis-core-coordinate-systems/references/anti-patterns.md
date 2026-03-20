# Anti-Patterns (QGIS CRS)

## 1. Missing Transform Context

```python
# WRONG: No transform context — ignores datum transformation preferences
xform = QgsCoordinateTransform(crs_src, crs_dest)

# CORRECT: ALWAYS provide the project transform context
xform = QgsCoordinateTransform(
    crs_src, crs_dest,
    QgsProject.instance().transformContext()
)
```

**WHY**: Without a transform context, QGIS cannot apply user-configured datum transformation grids. The result may be off by meters or more, with no error or warning.

---

## 2. Confusing CRS Assignment with Reprojection

```python
# WRONG: This does NOT reproject — it reinterprets existing coordinates
layer.setCrs(QgsCoordinateReferenceSystem("EPSG:32632"))
# If the layer was in EPSG:4326, all coordinates are now silently corrupted

# CORRECT: Use QgsCoordinateTransform or processing to reproject
import processing
result = processing.run("native:reprojectlayer", {
    "INPUT": layer,
    "TARGET_CRS": QgsCoordinateReferenceSystem("EPSG:32632"),
    "OUTPUT": "memory:"
})
reprojected = result["OUTPUT"]
```

**WHY**: `setCrs()` only changes the metadata label — it does not transform any coordinate values. A layer with longitude/latitude values labeled as UTM meters produces silently wrong results in every spatial operation.

---

## 3. Wrong Coordinate Order for EPSG:4326

```python
# WRONG: Putting latitude in x and longitude in y
point = QgsPointXY(52.0, 5.0)  # This is lat=52, lon=5 — REVERSED

# CORRECT: QgsPointXY ALWAYS takes (x=longitude, y=latitude)
point = QgsPointXY(5.0, 52.0)  # lon=5, lat=52
```

**WHY**: The EPSG standard defines EPSG:4326 with axis order latitude, longitude. However, QGIS (like most GIS software) uses x=longitude, y=latitude internally. `QgsPointXY(x, y)` ALWAYS means `(longitude, latitude)` for geographic CRS.

---

## 4. Skipping CRS Validity Check

```python
# WRONG: Using a CRS without checking validity
crs = QgsCoordinateReferenceSystem("EPSG:99999")
xform = QgsCoordinateTransform(crs, other_crs, context)
# xform is invalid — transforms return garbage or original coordinates

# CORRECT: ALWAYS check isValid()
crs = QgsCoordinateReferenceSystem("EPSG:99999")
if not crs.isValid():
    raise RuntimeError("Invalid CRS: EPSG:99999")
```

**WHY**: An invalid CRS object does not raise an exception on creation. It silently propagates through transforms, measurements, and spatial operations, producing wrong results everywhere.

---

## 5. Using Deprecated Creation Methods

```python
# WRONG: Deprecated methods from older QGIS versions
crs = QgsCoordinateReferenceSystem()
crs.createFromSrid(4326)       # Deprecated
crs.createFromEpsg(4326)       # Deprecated

# CORRECT: Use string-prefix constructor
crs = QgsCoordinateReferenceSystem("EPSG:4326")

# Or use static factory methods
crs = QgsCoordinateReferenceSystem.fromEpsgId(4326)
```

**WHY**: Deprecated methods may be removed in future QGIS versions. The string-prefix constructor is the standard, forward-compatible approach.

---

## 6. Missing QgsApplication Init in Standalone Scripts

```python
# WRONG: Using CRS without initializing QgsApplication
from qgis.core import QgsCoordinateReferenceSystem
crs = QgsCoordinateReferenceSystem("EPSG:4326")
# crs.isValid() returns False — srs.db not found

# CORRECT: Initialize QgsApplication first
from qgis.core import QgsApplication, QgsCoordinateReferenceSystem
qgs = QgsApplication([], False)
qgs.setPrefixPath("/usr", True)  # Set correct path for your OS
qgs.initQgis()

crs = QgsCoordinateReferenceSystem("EPSG:4326")
assert crs.isValid()  # Now works
```

**WHY**: QGIS needs the `srs.db` SQLite database (bundled with QGIS) to resolve EPSG codes to CRS definitions. Without `setPrefixPath()`, QGIS cannot locate this database. Within the QGIS desktop application or QGIS Python console, this is handled automatically.

---

## 7. Measuring with Geographic CRS Without Ellipsoid

```python
# WRONG: Measuring distances on geographic CRS without setting ellipsoid
d = QgsDistanceArea()
d.setSourceCrs(QgsCoordinateReferenceSystem("EPSG:4326"), context)
# Missing: d.setEllipsoid("WGS84")
distance = d.measureLine(p1, p2)
# Returns distance in DEGREES, not meters

# CORRECT: ALWAYS set ellipsoid for geographic CRS
d = QgsDistanceArea()
d.setSourceCrs(QgsCoordinateReferenceSystem("EPSG:4326"), context)
d.setEllipsoid("WGS84")
distance = d.measureLine(p1, p2)
# Returns distance in meters (ellipsoidal calculation)
```

**WHY**: Without an ellipsoid, `QgsDistanceArea` performs planimetric measurement in the CRS native units. For geographic CRS, native units are degrees — so the "distance" is in degrees, which is meaningless for real-world measurement.

---

## 8. Ignoring Datum Transformation Grid Availability

```python
# WRONG: Assuming all datum transformations are available
xform = QgsCoordinateTransform(
    QgsCoordinateReferenceSystem("EPSG:27700"),  # OSGB 1936
    QgsCoordinateReferenceSystem("EPSG:4326"),   # WGS 84
    context
)
# If OSTN15 grid is not installed, transform uses a less accurate fallback

# CORRECT: Check and warn about grid availability
xform = QgsCoordinateTransform(
    QgsCoordinateReferenceSystem("EPSG:27700"),
    QgsCoordinateReferenceSystem("EPSG:4326"),
    context
)
# Log or warn if high-precision grids are required for your use case
# Install transformation grids via QGIS Settings > Transformations
```

**WHY**: Datum transformations between different datums (e.g., OSGB 1936 to WGS 84) can require grid files for centimeter-level accuracy. Without the grid, QGIS falls back to a less accurate Helmert transformation, which can be off by several meters. There is no runtime error — only silent precision loss.

---

## 9. Hardcoding CRS Without Context

```python
# WRONG: Hardcoding a projected CRS for data that spans multiple zones
for layer in layers:
    xform = QgsCoordinateTransform(
        layer.crs(),
        QgsCoordinateReferenceSystem("EPSG:32632"),  # UTM 32N
        context
    )
    # Fails silently for data outside UTM zone 32N — extreme distortion

# CORRECT: Choose the CRS based on the data extent
extent = layer.extent()
centroid_lon = (extent.xMinimum() + extent.xMaximum()) / 2
if layer.crs().isGeographic():
    utm_zone = int((centroid_lon + 180) / 6) + 1
    epsg = 32600 + utm_zone  # Northern hemisphere
    target_crs = QgsCoordinateReferenceSystem(f"EPSG:{epsg}")
```

**WHY**: UTM zones cover 6 degrees of longitude. Using a UTM zone far from the data produces extreme distortion — distances and areas can be off by orders of magnitude.

---

## 10. Using setCrs() on Layers to "Fix" Coordinates

```python
# WRONG: Trying to fix misaligned data by changing the CRS label
layer = QgsVectorLayer("data.shp", "my_layer", "ogr")
# Layer appears in wrong location on the map
layer.setCrs(QgsCoordinateReferenceSystem("EPSG:4326"))  # "Fix" attempt
# This only changes the label — coordinates remain wrong

# CORRECT: Identify the actual CRS of the data and set it correctly
# If the data was created in RD New but has no .prj file:
layer.setCrs(QgsCoordinateReferenceSystem("EPSG:28992"))
# Now coordinates are interpreted correctly

# If you need the data in a different CRS, THEN reproject:
import processing
result = processing.run("native:reprojectlayer", {
    "INPUT": layer,
    "TARGET_CRS": QgsCoordinateReferenceSystem("EPSG:4326"),
    "OUTPUT": "memory:"
})
```

**WHY**: `setCrs()` is appropriate ONLY when you know the true CRS of the data and the file metadata is wrong or missing. It is NEVER appropriate as a way to reproject data. The distinction: `setCrs()` says "these coordinates ARE in this CRS"; reprojection says "convert these coordinates TO this CRS".
