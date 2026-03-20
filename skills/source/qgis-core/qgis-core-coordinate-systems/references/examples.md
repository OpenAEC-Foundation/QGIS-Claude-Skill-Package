# Working Code Examples (QGIS CRS)

## Example 1: Create CRS from Various Sources

```python
from qgis.core import QgsCoordinateReferenceSystem

# From EPSG code (most common)
crs_wgs84 = QgsCoordinateReferenceSystem("EPSG:4326")
assert crs_wgs84.isValid()

# From static factory
crs_utm = QgsCoordinateReferenceSystem.fromEpsgId(32632)
assert crs_utm.isValid()

# From WKT string
wkt = 'GEOGCS["WGS84", DATUM["WGS84", SPHEROID["WGS84", 6378137.0, 298.257223563]], PRIMEM["Greenwich", 0], UNIT["degree", 0.0174532925199433]]'
crs_from_wkt = QgsCoordinateReferenceSystem(f"WKT:{wkt}")

# From Proj string
crs_from_proj = QgsCoordinateReferenceSystem("PROJ:+proj=longlat +datum=WGS84 +no_defs")

# ALWAYS validate after creation
for crs in [crs_wgs84, crs_utm, crs_from_wkt, crs_from_proj]:
    if not crs.isValid():
        raise RuntimeError(f"Invalid CRS: {crs.authid()}")
```

---

## Example 2: Inspect CRS Properties

```python
from qgis.core import QgsCoordinateReferenceSystem

crs = QgsCoordinateReferenceSystem("EPSG:32633")

print(f"Auth ID:      {crs.authid()}")          # "EPSG:32633"
print(f"Description:  {crs.description()}")      # "WGS 84 / UTM zone 33N"
print(f"Geographic:   {crs.isGeographic()}")      # False
print(f"Map units:    {crs.mapUnits()}")          # Qgis.DistanceUnit.Meters
print(f"Projection:   {crs.projectionAcronym()}")  # "utm"
print(f"Ellipsoid:    {crs.ellipsoidAcronym()}")   # "EPSG:7030"
print(f"PostGIS SRID: {crs.postgisSrid()}")        # 32633
```

---

## Example 3: Transform a Point Between CRS

```python
from qgis.core import (
    QgsCoordinateReferenceSystem,
    QgsCoordinateTransform,
    QgsProject,
    QgsPointXY
)

crs_wgs84 = QgsCoordinateReferenceSystem("EPSG:4326")
crs_utm33 = QgsCoordinateReferenceSystem("EPSG:32633")
context = QgsProject.instance().transformContext()

xform = QgsCoordinateTransform(crs_wgs84, crs_utm33, context)

# Forward: WGS84 -> UTM 33N
# QgsPointXY takes (x=longitude, y=latitude) for geographic CRS
pt_wgs = QgsPointXY(18.0, 5.0)
pt_utm = xform.transform(pt_wgs)
print(f"UTM: {pt_utm.x():.2f}, {pt_utm.y():.2f}")

# Reverse: UTM 33N -> WGS84
pt_back = xform.transform(pt_utm, QgsCoordinateTransform.ReverseTransform)
print(f"WGS84: {pt_back.x():.6f}, {pt_back.y():.6f}")
```

---

## Example 4: Transform a Geometry In-Place

```python
from qgis.core import (
    QgsCoordinateReferenceSystem,
    QgsCoordinateTransform,
    QgsProject,
    QgsGeometry,
    QgsPointXY
)

# Create a polygon in WGS84
polygon = QgsGeometry.fromPolygonXY([[
    QgsPointXY(5.0, 52.0),
    QgsPointXY(5.1, 52.0),
    QgsPointXY(5.1, 52.1),
    QgsPointXY(5.0, 52.1),
    QgsPointXY(5.0, 52.0)  # Close the ring
]])

# Transform to Dutch national CRS (RD New)
xform = QgsCoordinateTransform(
    QgsCoordinateReferenceSystem("EPSG:4326"),
    QgsCoordinateReferenceSystem("EPSG:28992"),
    QgsProject.instance().transformContext()
)

# transform() modifies the geometry in-place
result = polygon.transform(xform)
# result == 0 means success (Qgis.GeometryOperationResult.Success)
print(f"Transform result: {result}")
print(f"Transformed extent: {polygon.boundingBox()}")
```

---

## Example 5: Transform a Bounding Box

```python
from qgis.core import (
    QgsCoordinateReferenceSystem,
    QgsCoordinateTransform,
    QgsProject,
    QgsRectangle
)

# Bounding box for the Netherlands in WGS84
bbox = QgsRectangle(3.37, 50.75, 7.21, 53.47)

xform = QgsCoordinateTransform(
    QgsCoordinateReferenceSystem("EPSG:4326"),
    QgsCoordinateReferenceSystem("EPSG:28992"),
    QgsProject.instance().transformContext()
)

bbox_rd = xform.transformBoundingBox(bbox)
print(f"RD New extent: {bbox_rd}")
```

---

## Example 6: Measure Distance Between Two Points

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

# Amsterdam to Rotterdam (approximate)
amsterdam = QgsPointXY(4.9, 52.37)
rotterdam = QgsPointXY(4.48, 51.92)

distance_m = d.measureLine(amsterdam, rotterdam)
distance_km = d.convertLengthMeasurement(distance_m, Qgis.DistanceUnit.Kilometers)

print(f"Distance: {distance_m:.0f} m = {distance_km:.1f} km")
```

---

## Example 7: Measure Area of Features

```python
from qgis.core import (
    QgsDistanceArea,
    QgsCoordinateReferenceSystem,
    QgsProject,
    Qgis
)

layer = QgsProject.instance().mapLayersByName("parcels")[0]

d = QgsDistanceArea()
d.setSourceCrs(layer.crs(), QgsProject.instance().transformContext())
d.setEllipsoid("WGS84")

for feature in layer.getFeatures():
    area_sqm = d.measureArea(feature.geometry())
    area_ha = d.convertAreaMeasurement(area_sqm, Qgis.AreaUnit.Hectares)
    perimeter_m = d.measurePerimeter(feature.geometry())
    print(f"Feature {feature.id()}: {area_ha:.2f} ha, perimeter {perimeter_m:.0f} m")
```

---

## Example 8: Measure Distance Along a Polyline

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

# Route: Amsterdam -> Utrecht -> Eindhoven
route = [
    QgsPointXY(4.9, 52.37),    # Amsterdam
    QgsPointXY(5.12, 52.09),   # Utrecht
    QgsPointXY(5.47, 51.44),   # Eindhoven
]

total_m = d.measureLine(route)
total_km = d.convertLengthMeasurement(total_m, Qgis.DistanceUnit.Kilometers)
print(f"Route distance: {total_km:.1f} km")
```

---

## Example 9: Set and Read Project CRS

```python
from qgis.core import QgsProject, QgsCoordinateReferenceSystem

project = QgsProject.instance()

# Read current project CRS
current_crs = project.crs()
print(f"Current project CRS: {current_crs.authid()}")

# Set project CRS (enables on-the-fly reprojection for all layers)
project.setCrs(QgsCoordinateReferenceSystem("EPSG:3857"))
print(f"New project CRS: {project.crs().authid()}")
```

---

## Example 10: Display Layer Names with CRS

```python
from qgis.core import QgsProject

for layer in QgsProject.instance().mapLayers().values():
    crs = layer.crs()
    print(f"{layer.name()} -> {crs.authid()} ({crs.description()})")
    if crs.isGeographic():
        print(f"  Warning: geographic CRS (degrees) — measurements need ellipsoidal calculation")
```

---

## Example 11: Reproject Layer Using Processing

```python
import processing
from qgis.core import QgsCoordinateReferenceSystem, QgsProject

layer = QgsProject.instance().mapLayersByName("input_layer")[0]

result = processing.run("native:reprojectlayer", {
    "INPUT": layer,
    "TARGET_CRS": QgsCoordinateReferenceSystem("EPSG:32632"),
    "OUTPUT": "memory:"
})

reprojected = result["OUTPUT"]
QgsProject.instance().addMapLayer(reprojected)
print(f"Reprojected layer CRS: {reprojected.crs().authid()}")
```

---

## Example 12: Standalone Script CRS Setup

```python
import sys
from qgis.core import (
    QgsApplication,
    QgsCoordinateReferenceSystem,
    QgsCoordinateTransform,
    QgsCoordinateTransformContext,
    QgsPointXY
)

# Initialize QGIS application (REQUIRED for standalone scripts)
qgs = QgsApplication([], False)
qgs.setPrefixPath("/usr", True)  # Adjust path for your OS
qgs.initQgis()

# Now CRS operations work
crs = QgsCoordinateReferenceSystem("EPSG:4326")
if not crs.isValid():
    raise RuntimeError("CRS creation failed — check QgsApplication prefix path")

# In standalone scripts without a project, use a bare context
context = QgsCoordinateTransformContext()
xform = QgsCoordinateTransform(
    QgsCoordinateReferenceSystem("EPSG:4326"),
    QgsCoordinateReferenceSystem("EPSG:3857"),
    context
)

pt = xform.transform(QgsPointXY(5.0, 52.0))
print(f"Web Mercator: {pt.x():.2f}, {pt.y():.2f}")

# Cleanup
qgs.exitQgis()
```
