# qgis-impl-vector-analysis — Examples

## Example 1: Complete Buffer and Clip Workflow

Buffer a point layer by 500m and clip to a study area polygon.

```python
import processing
from qgis.core import QgsProject

# Load layers
points = QgsProject.instance().mapLayersByName('sample_points')[0]
study_area = QgsProject.instance().mapLayersByName('study_boundary')[0]

# Step 1: Fix geometries (ALWAYS do this for external data)
fixed = processing.run("native:fixgeometries", {
    'INPUT': points,
    'OUTPUT': 'memory:'
})['OUTPUT']

# Step 2: Buffer points by 500 meters
buffered = processing.run("native:buffer", {
    'INPUT': fixed,
    'DISTANCE': 500,
    'SEGMENTS': 5,
    'END_CAP_STYLE': 0,
    'JOIN_STYLE': 0,
    'MITER_LIMIT': 2,
    'DISSOLVE': False,
    'OUTPUT': 'memory:'
})['OUTPUT']

# Step 3: Clip buffers to study area
clipped = processing.run("native:clip", {
    'INPUT': buffered,
    'OVERLAY': study_area,
    'OUTPUT': '/path/to/output/buffered_clipped.gpkg'
})['OUTPUT']

# Step 4: Add to map
QgsProject.instance().addMapLayer(clipped)
```

---

## Example 2: Spatial Join — Transfer Zone Data to Points

Transfer land use zone attributes to point features based on spatial location.

```python
import processing

points = QgsProject.instance().mapLayersByName('buildings')[0]
zones = QgsProject.instance().mapLayersByName('land_use_zones')[0]

result = processing.run("native:joinattributesbylocation", {
    'INPUT': points,
    'PREDICATE': [5],              # 5 = within
    'JOIN': zones,
    'JOIN_FIELDS': ['zone_type', 'max_height', 'fsi'],
    'METHOD': 1,                   # First match only
    'DISCARD_NONMATCHING': False,
    'PREFIX': 'zone_',
    'OUTPUT': 'memory:'
})

joined_layer = result['OUTPUT']
joined_layer.setName('buildings_with_zones')
QgsProject.instance().addMapLayer(joined_layer)

# Verify join results
total = joined_layer.featureCount()
matched = 0
for f in joined_layer.getFeatures():
    if f['zone_zone_type'] is not None:
        matched += 1
print(f"Matched {matched} of {total} features")
```

---

## Example 3: Dissolve and Aggregate Statistics

Dissolve parcels by municipality and calculate total area and count.

```python
import processing

parcels = QgsProject.instance().mapLayersByName('parcels')[0]

# Simple dissolve by field
dissolved = processing.run("native:dissolve", {
    'INPUT': parcels,
    'FIELD': ['municipality'],
    'SEPARATE_DISJOINT': False,
    'OUTPUT': 'memory:'
})['OUTPUT']

# Aggregate with statistics
aggregated = processing.run("native:aggregate", {
    'INPUT': parcels,
    'GROUP_BY': '"municipality"',
    'AGGREGATES': [
        {'aggregate': 'first_value', 'delimiter': ',', 'input': '"municipality"',
         'length': 100, 'name': 'municipality', 'precision': 0, 'type': 10},
        {'aggregate': 'sum', 'delimiter': ',', 'input': '"area_ha"',
         'length': 10, 'name': 'total_area_ha', 'precision': 2, 'type': 6},
        {'aggregate': 'count', 'delimiter': ',', 'input': '"id"',
         'length': 10, 'name': 'parcel_count', 'precision': 0, 'type': 2}
    ],
    'OUTPUT': 'memory:'
})['OUTPUT']

aggregated.setName('municipality_summary')
QgsProject.instance().addMapLayer(aggregated)
```

---

## Example 4: Multi-Layer Intersection Analysis

Find areas where flood zones overlap with residential land use.

```python
import processing
from qgis.core import QgsProject, QgsVectorFileWriter, QgsCoordinateTransformContext

flood_zones = QgsProject.instance().mapLayersByName('flood_zones')[0]
land_use = QgsProject.instance().mapLayersByName('land_use')[0]

# Step 1: Extract residential areas
residential = processing.run("native:extractbyexpression", {
    'INPUT': land_use,
    'EXPRESSION': '"category" = \'residential\'',
    'OUTPUT': 'memory:'
})['OUTPUT']

# Step 2: Intersect flood zones with residential areas
at_risk = processing.run("native:intersection", {
    'INPUT': flood_zones,
    'OVERLAY': residential,
    'INPUT_FIELDS': ['flood_level', 'return_period'],
    'OVERLAY_FIELDS': ['neighborhood', 'dwelling_count'],
    'OVERLAY_FIELDS_PREFIX': 'res_',
    'OUTPUT': 'memory:'
})['OUTPUT']

# Step 3: Calculate affected area
with_area = processing.run("native:fieldcalculator", {
    'INPUT': at_risk,
    'FIELD_NAME': 'affected_area_m2',
    'FIELD_TYPE': 0,
    'FIELD_LENGTH': 15,
    'FIELD_PRECISION': 2,
    'FORMULA': '$area',
    'OUTPUT': 'memory:'
})['OUTPUT']

# Step 4: Write to GeoPackage
save_options = QgsVectorFileWriter.SaveVectorOptions()
save_options.driverName = "GPKG"
save_options.fileEncoding = "UTF-8"

QgsVectorFileWriter.writeAsVectorFormatV3(
    with_area,
    '/path/to/flood_risk_residential.gpkg',
    QgsCoordinateTransformContext(),
    save_options
)
```

---

## Example 5: Attribute Join from CSV

Join census data from a CSV table to a polygon layer.

```python
import processing
from qgis.core import QgsProject, QgsVectorLayer

# Load CSV as table (no geometry)
csv_uri = "file:///path/to/census_data.csv?delimiter=,&encoding=UTF-8"
census_table = QgsVectorLayer(csv_uri, "census_data", "delimitedtext")

neighborhoods = QgsProject.instance().mapLayersByName('neighborhoods')[0]

# Join by matching field
result = processing.run("native:joinattributestable", {
    'INPUT': neighborhoods,
    'FIELD': 'neighborhood_code',
    'INPUT_2': census_table,
    'FIELD_2': 'code',
    'FIELDS_TO_COPY': ['population', 'median_income', 'avg_household_size'],
    'METHOD': 1,                   # First match only
    'DISCARD_NONMATCHING': False,
    'PREFIX': 'census_',
    'OUTPUT': 'memory:'
})

joined = result['OUTPUT']
joined.setName('neighborhoods_with_census')
QgsProject.instance().addMapLayer(joined)
```

---

## Example 6: Spatial Query with Index for Performance

Find all buildings within 200m of a river using spatial index for fast lookups.

```python
from qgis.core import (
    QgsSpatialIndex, QgsFeatureRequest, QgsGeometry, QgsProject
)

buildings = QgsProject.instance().mapLayersByName('buildings')[0]
rivers = QgsProject.instance().mapLayersByName('rivers')[0]

# Build spatial index on buildings
building_index = QgsSpatialIndex(buildings.getFeatures())

# Buffer each river segment and find nearby buildings
nearby_ids = set()
for river_feat in rivers.getFeatures():
    if not river_feat.hasGeometry():
        continue
    river_geom = river_feat.geometry()
    buffer_geom = river_geom.buffer(200.0, 5)
    bbox = buffer_geom.boundingBox()

    # Fast bounding box check via index
    candidate_ids = building_index.intersects(bbox)

    # Exact geometry check
    for fid in candidate_ids:
        building = buildings.getFeature(fid)
        if building.hasGeometry() and buffer_geom.intersects(building.geometry()):
            nearby_ids.add(fid)

# Select the results
buildings.selectByIds(list(nearby_ids))
print(f"Found {len(nearby_ids)} buildings within 200m of rivers")
```

---

## Example 7: Field Calculation via Edit Buffer

Calculate population density on an existing field using the edit buffer.

```python
from qgis.core import QgsProject

layer = QgsProject.instance().mapLayersByName('districts')[0]
density_idx = layer.fields().indexOf('pop_density')
pop_idx = layer.fields().indexOf('population')

with edit(layer):
    for feature in layer.getFeatures():
        if not feature.hasGeometry():
            continue
        area_km2 = feature.geometry().area() / 1_000_000  # m2 to km2
        population = feature[pop_idx]
        if area_km2 > 0 and population is not None:
            density = population / area_km2
            layer.changeAttributeValue(feature.id(), density_idx, density)
```

---

## Example 8: Chain Multiple Overlay Operations

Identify parks within 1km of schools that are NOT in flood zones.

```python
import processing

schools = QgsProject.instance().mapLayersByName('schools')[0]
parks = QgsProject.instance().mapLayersByName('parks')[0]
flood_zones = QgsProject.instance().mapLayersByName('flood_zones')[0]

# Step 1: Buffer schools by 1km
school_buffers = processing.run("native:buffer", {
    'INPUT': schools,
    'DISTANCE': 1000,
    'SEGMENTS': 5,
    'END_CAP_STYLE': 0,
    'JOIN_STYLE': 0,
    'DISSOLVE': True,          # Merge overlapping buffers
    'OUTPUT': 'memory:'
})['OUTPUT']

# Step 2: Extract parks within school buffer zones
parks_near_schools = processing.run("native:extractbylocation", {
    'INPUT': parks,
    'PREDICATE': [0],          # intersects
    'INTERSECT': school_buffers,
    'OUTPUT': 'memory:'
})['OUTPUT']

# Step 3: Remove parks in flood zones (difference by location)
safe_parks = processing.run("native:extractbylocation", {
    'INPUT': parks_near_schools,
    'PREDICATE': [2],          # disjoint (no spatial relationship)
    'INTERSECT': flood_zones,
    'OUTPUT': '/path/to/safe_parks_near_schools.gpkg'
})['OUTPUT']

# Note: PREDICATE [2] = disjoint is NOT available in extractbylocation.
# Instead, use a two-step approach:
parks_in_flood = processing.run("native:extractbylocation", {
    'INPUT': parks_near_schools,
    'PREDICATE': [0],          # intersects
    'INTERSECT': flood_zones,
    'OUTPUT': 'memory:'
})['OUTPUT']

# Get IDs of parks in flood zones
flood_park_ids = {f.id() for f in parks_in_flood.getFeatures()}

# Select parks NOT in flood zones
safe_ids = []
for f in parks_near_schools.getFeatures():
    if f.id() not in flood_park_ids:
        safe_ids.append(f.id())

parks_near_schools.selectByIds(safe_ids)
```

---

## Example 9: Write Features to Multiple Formats

Export analysis results to GeoPackage, GeoJSON, and Shapefile.

```python
from qgis.core import (
    QgsVectorFileWriter, QgsCoordinateTransformContext, QgsProject
)

layer = QgsProject.instance().mapLayersByName('analysis_result')[0]
ctx = QgsCoordinateTransformContext()

# GeoPackage (ALWAYS preferred)
opts_gpkg = QgsVectorFileWriter.SaveVectorOptions()
opts_gpkg.driverName = "GPKG"
opts_gpkg.fileEncoding = "UTF-8"
QgsVectorFileWriter.writeAsVectorFormatV3(layer, '/output/result.gpkg', ctx, opts_gpkg)

# GeoJSON (for web applications)
opts_json = QgsVectorFileWriter.SaveVectorOptions()
opts_json.driverName = "GeoJSON"
opts_json.fileEncoding = "UTF-8"
QgsVectorFileWriter.writeAsVectorFormatV3(layer, '/output/result.geojson', ctx, opts_json)

# Shapefile (legacy compatibility only)
opts_shp = QgsVectorFileWriter.SaveVectorOptions()
opts_shp.driverName = "ESRI Shapefile"
opts_shp.fileEncoding = "UTF-8"
QgsVectorFileWriter.writeAsVectorFormatV3(layer, '/output/result.shp', ctx, opts_shp)
```
