---
name: qgis-core-coordinate-systems
description: >
  Use when handling coordinate reference systems, transforming coordinates, or measuring distances/areas.
  Prevents silent data corruption from wrong CRS assignment and lat/lon coordinate order confusion.
  Covers CRS creation, coordinate transforms, datum transformations, distance measurement, and EPSG database.
  Keywords: CRS, EPSG, coordinate transform, projection, QgsCoordinateReferenceSystem, datum, QgsDistanceArea, reprojection, wrong location, map shifted, coordinates don't match, set projection.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-core-coordinate-systems

## Quick Reference

### Core CRS Classes

| Class | Purpose | Key Methods |
|-------|---------|-------------|
| `QgsCoordinateReferenceSystem` | Represents a spatial reference system | `isValid()`, `authid()`, `isGeographic()`, `mapUnits()` |
| `QgsCoordinateTransform` | Transforms coordinates between two CRS | `transform()`, `transformBoundingBox()` |
| `QgsCoordinateTransformContext` | Project-level datum transformation settings | Used as parameter for transforms |
| `QgsDistanceArea` | Measures distances and areas in any CRS | `measureLine()`, `measureArea()`, `setEllipsoid()` |

### CRS Creation Methods

| Input Format | Constructor Syntax | Example |
|-------------|-------------------|---------|
| EPSG code | `QgsCoordinateReferenceSystem("EPSG:<code>")` | `"EPSG:4326"` |
| PostGIS SRID | `QgsCoordinateReferenceSystem("POSTGIS:<srid>")` | `"POSTGIS:4326"` |
| Internal ID | `QgsCoordinateReferenceSystem("INTERNAL:<srsid>")` | `"INTERNAL:3452"` |
| Proj string | `QgsCoordinateReferenceSystem("PROJ:<proj>")` | `"PROJ:+proj=longlat +datum=WGS84"` |
| WKT string | `QgsCoordinateReferenceSystem("WKT:<wkt>")` | `"WKT:GEOGCS[...]"` |

### CRS Property Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `authid()` | `str` | Authority identifier, e.g., `"EPSG:4326"` |
| `description()` | `str` | Human-readable name, e.g., `"WGS 84"` |
| `isValid()` | `bool` | Whether the CRS was successfully created |
| `isGeographic()` | `bool` | `True` for geographic CRS (degrees), `False` for projected (meters) |
| `mapUnits()` | `Qgis.DistanceUnit` | Unit enumeration for the CRS |
| `toProj()` | `str` | Full Proj string representation |
| `toWkt()` | `str` | Full WKT string representation |
| `postgisSrid()` | `int` | PostGIS SRID value |
| `srsid()` | `int` | QGIS internal ID |
| `projectionAcronym()` | `str` | Projection type, e.g., `"longlat"` |
| `ellipsoidAcronym()` | `str` | Ellipsoid identifier, e.g., `"EPSG:7030"` |

### Common EPSG Codes

| Code | Name | Type | Units | Use Case |
|------|------|------|-------|----------|
| 4326 | WGS 84 | Geographic | Degrees | GPS coordinates, global data exchange |
| 3857 | Web Mercator | Projected | Meters | Web maps (OpenStreetMap, Google Maps) |
| 32631-32660 | UTM Zones 31N-60N | Projected | Meters | Accurate local measurements (Northern Hemisphere) |
| 32701-32760 | UTM Zones 1S-60S | Projected | Meters | Accurate local measurements (Southern Hemisphere) |
| 28992 | Amersfoort / RD New | Projected | Meters | Netherlands national grid |
| 27700 | OSGB 1936 | Projected | Meters | UK Ordnance Survey |
| 2154 | RGF93 / Lambert-93 | Projected | Meters | France national grid |

---

## Critical Warnings

**NEVER** create a `QgsCoordinateTransform` without a `QgsCoordinateTransformContext`. The bare constructor ignores datum transformation preferences and produces silently inaccurate results.

**NEVER** assign a CRS to a layer as a substitute for reprojection. `layer.setCrs()` changes the interpretation of existing coordinates -- it does NOT transform the data. Use `QgsCoordinateTransform` to actually reproject.

**NEVER** assume `QgsPointXY` takes latitude first for EPSG:4326. `QgsPointXY` ALWAYS takes `(x, y)` = `(longitude, latitude)`, regardless of the EPSG axis order specification.

**NEVER** use deprecated creation methods (`createFromSrid()`, `createFromEpsg()`). ALWAYS use the string-prefix constructor: `QgsCoordinateReferenceSystem("EPSG:4326")`.

**NEVER** skip `isValid()` checks after CRS creation. An invalid CRS silently produces wrong results in ALL downstream operations (transforms, measurements, spatial queries).

**NEVER** run standalone PyQGIS scripts without `QgsApplication.setPrefixPath()` set correctly. Without it, QGIS cannot find the `srs.db` database and ALL CRS operations fail silently.

**ALWAYS** use `QgsProject.instance().transformContext()` instead of a bare `QgsCoordinateTransformContext()`. The project context includes user-configured datum transformation preferences.

**ALWAYS** check datum transformation grid availability before transforming between datums that require grid files. Missing grids cause silent loss of precision.

---

## Decision Tree

### Which CRS to Use?

```
Need to handle coordinates?
├── Receiving GPS/WGS84 data?
│   └── Use EPSG:4326 (geographic, degrees)
├── Displaying on a web map?
│   └── Use EPSG:3857 (Web Mercator, meters)
├── Measuring distances/areas accurately?
│   ├── Local area (< 6 degrees longitude)?
│   │   └── Use UTM zone for the area (EPSG:326xx for N, EPSG:327xx for S)
│   └── Large area or global?
│       └── Use EPSG:4326 + QgsDistanceArea with ellipsoidal measurement
├── Working with national data?
│   └── Use the country's national CRS (e.g., EPSG:28992 for NL)
└── Storing data for exchange?
    └── Use EPSG:4326 (universal standard)
```

### How to Transform Coordinates?

```
Need to transform coordinates?
├── Single point?
│   └── QgsCoordinateTransform.transform(QgsPointXY)
├── Bounding box?
│   └── QgsCoordinateTransform.transformBoundingBox(QgsRectangle)
├── Full geometry?
│   └── geometry.transform(QgsCoordinateTransform)
├── Entire layer?
│   └── Use processing: native:reprojectlayer
└── Just for display?
    └── Set project CRS — on-the-fly reprojection handles rendering
```

### Ellipsoidal vs. Planimetric Measurement?

```
Need to measure distances or areas?
├── Source CRS is geographic (degrees)?
│   └── ALWAYS use QgsDistanceArea with ellipsoid set
├── Source CRS is projected (meters)?
│   ├── Need high precision over large areas?
│   │   └── Use QgsDistanceArea with ellipsoid set
│   └── Local area, moderate precision OK?
│       └── Planimetric (Cartesian) measurement is acceptable
└── Unsure?
    └── ALWAYS use QgsDistanceArea with ellipsoid — it handles both cases
```

---

## Essential Patterns

### Pattern 1: Create and Validate a CRS

```python
from qgis.core import QgsCoordinateReferenceSystem

crs = QgsCoordinateReferenceSystem("EPSG:4326")
if not crs.isValid():
    raise RuntimeError("Failed to create CRS: EPSG:4326")

print(crs.authid())        # "EPSG:4326"
print(crs.description())   # "WGS 84"
print(crs.isGeographic())  # True
```

### Pattern 2: Transform Coordinates Between CRS

```python
from qgis.core import (
    QgsCoordinateReferenceSystem,
    QgsCoordinateTransform,
    QgsProject,
    QgsPointXY
)

crs_src = QgsCoordinateReferenceSystem("EPSG:4326")
crs_dest = QgsCoordinateReferenceSystem("EPSG:32633")
context = QgsProject.instance().transformContext()

xform = QgsCoordinateTransform(crs_src, crs_dest, context)

# Forward transform (source -> destination)
pt_utm = xform.transform(QgsPointXY(18.0, 5.0))

# Reverse transform (destination -> source)
pt_wgs = xform.transform(pt_utm, QgsCoordinateTransform.ReverseTransform)
```

### Pattern 3: Transform a Geometry

```python
from qgis.core import (
    QgsCoordinateReferenceSystem,
    QgsCoordinateTransform,
    QgsProject,
    QgsGeometry,
    QgsPointXY
)

geom = QgsGeometry.fromPointXY(QgsPointXY(5.0, 52.0))
xform = QgsCoordinateTransform(
    QgsCoordinateReferenceSystem("EPSG:4326"),
    QgsCoordinateReferenceSystem("EPSG:28992"),
    QgsProject.instance().transformContext()
)
geom.transform(xform)  # In-place transformation
```

### Pattern 4: Measure Distance with QgsDistanceArea

```python
from qgis.core import (
    QgsDistanceArea,
    QgsCoordinateReferenceSystem,
    QgsPointXY,
    QgsProject,
    Qgis
)

d = QgsDistanceArea()
d.setSourceCrs(
    QgsCoordinateReferenceSystem("EPSG:4326"),
    QgsProject.instance().transformContext()
)
d.setEllipsoid("WGS84")

point1 = QgsPointXY(5.0, 52.0)
point2 = QgsPointXY(5.1, 52.1)
distance_m = d.measureLine(point1, point2)

# Convert to kilometers
distance_km = d.convertLengthMeasurement(distance_m, Qgis.DistanceUnit.Kilometers)
```

### Pattern 5: Set Project CRS

```python
from qgis.core import QgsProject, QgsCoordinateReferenceSystem

# Set project CRS: all layers render in this CRS via on-the-fly reprojection
QgsProject.instance().setCrs(QgsCoordinateReferenceSystem("EPSG:3857"))
```

---

## Common Operations

### Get CRS from an Existing Layer

```python
layer = QgsProject.instance().mapLayersByName("my_layer")[0]
crs = layer.crs()
print(crs.authid())  # e.g., "EPSG:4326"
```

### Transform a Bounding Box

```python
from qgis.core import QgsRectangle

bbox = QgsRectangle(4.0, 51.0, 6.0, 53.0)  # xmin, ymin, xmax, ymax
xform = QgsCoordinateTransform(
    QgsCoordinateReferenceSystem("EPSG:4326"),
    QgsCoordinateReferenceSystem("EPSG:3857"),
    QgsProject.instance().transformContext()
)
bbox_transformed = xform.transformBoundingBox(bbox)
```

### Measure Area of a Polygon

```python
from qgis.core import QgsDistanceArea, QgsCoordinateReferenceSystem, QgsProject, Qgis

d = QgsDistanceArea()
d.setSourceCrs(layer.crs(), QgsProject.instance().transformContext())
d.setEllipsoid("WGS84")

for feature in layer.getFeatures():
    area_sqm = d.measureArea(feature.geometry())
    area_ha = d.convertAreaMeasurement(area_sqm, Qgis.AreaUnit.Hectares)
```

### Reproject a Layer via Processing

```python
import processing

result = processing.run("native:reprojectlayer", {
    "INPUT": layer,
    "TARGET_CRS": QgsCoordinateReferenceSystem("EPSG:32632"),
    "OUTPUT": "memory:"
})
reprojected_layer = result["OUTPUT"]
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for QgsCoordinateReferenceSystem, QgsCoordinateTransform, QgsDistanceArea
- [references/examples.md](references/examples.md) -- Working code examples for CRS operations
- [references/anti-patterns.md](references/anti-patterns.md) -- CRS pitfalls and what NOT to do

### Official Sources

- https://qgis.org/pyqgis/master/core/QgsCoordinateReferenceSystem.html
- https://qgis.org/pyqgis/master/core/QgsCoordinateTransform.html
- https://qgis.org/pyqgis/master/core/QgsDistanceArea.html
- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/crs.html
