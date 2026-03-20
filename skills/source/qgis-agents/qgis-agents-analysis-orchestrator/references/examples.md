# examples.md — Analysis Orchestrator

## Example 1: Site Suitability Analysis (Vector Overlay Chain)

**Question**: Find areas within 500m of a train station, within a residential zone, and NOT in a flood risk area.

**Workflow Plan**:
1. Load: stations (point), zoning (polygon), flood_risk (polygon)
2. Validate + Reproject to EPSG:28992
3. Buffer stations by 500m
4. Intersect buffer with residential zones
5. Difference with flood risk areas
6. Export to GeoPackage

```python
import processing
from qgis.core import (
    QgsVectorLayer, QgsProject, QgsProcessingException, QgsMessageLog
)

# Step 1: Load
stations = QgsVectorLayer("/data/stations.gpkg", "stations", "ogr")
zoning = QgsVectorLayer("/data/zoning.gpkg|layername=residential", "zoning", "ogr")
flood = QgsVectorLayer("/data/flood_risk.gpkg", "flood", "ogr")

for lyr in [stations, zoning, flood]:
    assert lyr.isValid(), f"Layer {lyr.name()} failed to load"

# Step 2: Reproject all to EPSG:28992
target_crs = "EPSG:28992"
reprojected_layers = {}
for name, layer in [("stations", stations), ("zoning", zoning), ("flood", flood)]:
    try:
        result = processing.run("native:reprojectlayer", {
            'INPUT': layer,
            'TARGET_CRS': target_crs,
            'OUTPUT': 'memory:'
        })
        reprojected_layers[name] = result['OUTPUT']
    except QgsProcessingException as e:
        QgsMessageLog.logMessage(f"Reproject failed for {name}: {e}", "Analysis")
        raise

# Step 3: Buffer stations
try:
    buffered = processing.run("native:buffer", {
        'INPUT': reprojected_layers['stations'],
        'DISTANCE': 500,
        'SEGMENTS': 16,
        'DISSOLVE': True,
        'OUTPUT': 'memory:'
    })['OUTPUT']
except QgsProcessingException as e:
    raise RuntimeError(f"Buffer failed: {e}")

# Step 4: Intersect with residential zones
try:
    suitable = processing.run("native:intersection", {
        'INPUT': buffered,
        'OVERLAY': reprojected_layers['zoning'],
        'OUTPUT': 'memory:'
    })['OUTPUT']
except QgsProcessingException as e:
    raise RuntimeError(f"Intersection failed: {e}")

# Step 5: Remove flood risk areas
try:
    final = processing.run("native:difference", {
        'INPUT': suitable,
        'OVERLAY': reprojected_layers['flood'],
        'OUTPUT': '/data/output/suitable_sites.gpkg'
    })['OUTPUT']
except QgsProcessingException as e:
    raise RuntimeError(f"Difference failed: {e}")

QgsMessageLog.logMessage("Site suitability analysis complete", "Analysis")
```

---

## Example 2: Raster-Vector Hybrid (Zonal Statistics)

**Question**: Calculate average elevation per municipality from a DEM.

**Workflow Plan**:
1. Load: DEM (raster), municipalities (polygon)
2. Reproject municipalities to match DEM CRS
3. Run zonal statistics
4. Export enriched vector layer

```python
import processing
from qgis.core import QgsVectorLayer, QgsRasterLayer, QgsProcessingException

# Step 1: Load
dem = QgsRasterLayer("/data/dem_25m.tif", "dem")
municipalities = QgsVectorLayer("/data/gemeenten.gpkg", "gemeenten", "ogr")

assert dem.isValid(), "DEM failed to load"
assert municipalities.isValid(), "Municipalities failed to load"

# Step 2: Reproject vector to match raster CRS
try:
    reproj_muni = processing.run("native:reprojectlayer", {
        'INPUT': municipalities,
        'TARGET_CRS': dem.crs(),
        'OUTPUT': 'memory:'
    })['OUTPUT']
except QgsProcessingException as e:
    raise RuntimeError(f"Reproject failed: {e}")

# Step 3: Zonal statistics
try:
    result = processing.run("native:zonalstatisticsfb", {
        'INPUT': reproj_muni,
        'INPUT_RASTER': dem,
        'RASTER_BAND': 1,
        'COLUMN_PREFIX': 'elev_',
        'STATISTICS': [0, 1, 2],  # Count, Sum, Mean
        'OUTPUT': '/data/output/municipalities_elevation.gpkg'
    })
except QgsProcessingException as e:
    raise RuntimeError(f"Zonal statistics failed: {e}")
```

---

## Example 3: Network Analysis (Service Area)

**Question**: Find all areas reachable within 10 minutes driving from a fire station.

**Workflow Plan**:
1. Load: road network (line), fire station (point)
2. Validate road network has speed attribute
3. Run service area analysis
4. Export result

```python
import processing
from qgis.core import QgsVectorLayer, QgsProcessingException

# Step 1: Load
roads = QgsVectorLayer("/data/roads.gpkg", "roads", "ogr")
station = QgsVectorLayer("/data/fire_station.gpkg", "station", "ogr")

assert roads.isValid(), "Roads failed to load"
assert station.isValid(), "Station failed to load"

# Step 2: Verify speed field exists
field_names = [f.name() for f in roads.fields()]
assert 'speed_kmh' in field_names, "Road layer MUST have 'speed_kmh' field"

# Step 3: Service area (10 minutes = 600 seconds)
try:
    result = processing.run("native:serviceareafrompoint", {
        'INPUT': roads,
        'START_POINT': f"{station.getFeature(1).geometry().asPoint().x()},"
                       f"{station.getFeature(1).geometry().asPoint().y()}"
                       f" [{roads.crs().authid()}]",
        'TRAVEL_COST': 600,
        'STRATEGY': 1,  # 0=Shortest, 1=Fastest
        'DEFAULT_SPEED': 50,
        'SPEED_FIELD': 'speed_kmh',
        'DEFAULT_DIRECTION': 2,  # Both directions
        'OUTPUT': '/data/output/fire_service_area.gpkg'
    })
except QgsProcessingException as e:
    raise RuntimeError(f"Service area analysis failed: {e}")
```

---

## Example 4: Clustering Analysis

**Question**: Find clusters of crime incidents using DBSCAN.

**Workflow Plan**:
1. Load incident points
2. Reproject to projected CRS for accurate distance
3. Create spatial index
4. Run DBSCAN clustering
5. Export with cluster IDs

```python
import processing
from qgis.core import QgsVectorLayer, QgsProcessingException

# Step 1: Load
incidents = QgsVectorLayer("/data/crime_incidents.gpkg", "incidents", "ogr")
assert incidents.isValid(), "Incidents layer failed to load"

# Step 2: Reproject to projected CRS
try:
    reproj = processing.run("native:reprojectlayer", {
        'INPUT': incidents,
        'TARGET_CRS': 'EPSG:28992',
        'OUTPUT': 'memory:'
    })['OUTPUT']
except QgsProcessingException as e:
    raise RuntimeError(f"Reproject failed: {e}")

# Step 3: Spatial index
processing.run("native:createspatialindex", {'INPUT': reproj})

# Step 4: DBSCAN clustering
try:
    result = processing.run("native:dbscanclustering", {
        'INPUT': reproj,
        'MIN_SIZE': 5,       # Minimum cluster size
        'EPS': 200,          # 200 meter radius (projected CRS = meters)
        'OUTPUT': '/data/output/crime_clusters.gpkg'
    })
except QgsProcessingException as e:
    raise RuntimeError(f"DBSCAN failed: {e}")
```

---

## Example 5: Batch Processing Multiple Layers

**Question**: Buffer and dissolve all layers in a GeoPackage.

```python
import processing
from osgeo import ogr
from qgis.core import QgsVectorLayer, QgsProcessingException, QgsMessageLog

gpkg_path = "/data/input_layers.gpkg"

# Discover all layers in GeoPackage
ds = ogr.Open(gpkg_path)
layer_names = [ds.GetLayerByIndex(i).GetName() for i in range(ds.GetLayerCount())]
ds = None

for name in layer_names:
    uri = f"{gpkg_path}|layername={name}"
    layer = QgsVectorLayer(uri, name, "ogr")

    if not layer.isValid():
        QgsMessageLog.logMessage(f"Skipping invalid layer: {name}", "Batch")
        continue

    try:
        buffered = processing.run("native:buffer", {
            'INPUT': layer,
            'DISTANCE': 100,
            'DISSOLVE': True,
            'OUTPUT': 'memory:'
        })['OUTPUT']

        processing.run("native:savefeatures", {
            'INPUT': buffered,
            'OUTPUT': f"/data/output/buffered_{name}.gpkg"
        })
    except QgsProcessingException as e:
        QgsMessageLog.logMessage(f"Failed for {name}: {e}", "Batch")
        continue

QgsMessageLog.logMessage("Batch processing complete", "Batch")
```
