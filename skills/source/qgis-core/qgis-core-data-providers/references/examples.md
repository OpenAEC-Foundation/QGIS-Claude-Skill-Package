# qgis-core-data-providers — URI Format Examples

## Vector Providers

### OGR Provider (`"ogr"`)

#### Shapefile
```python
vlayer = QgsVectorLayer("/data/airports.shp", "Airports", "ogr")
```

#### GeoPackage (by layer name — ALWAYS preferred)
```python
vlayer = QgsVectorLayer("/data/project.gpkg|layername=buildings", "Buildings", "ogr")
```

#### GeoPackage (by layer index — AVOID, index can change)
```python
vlayer = QgsVectorLayer("/data/project.gpkg|layerid=0", "First Layer", "ogr")
```

#### GeoJSON
```python
vlayer = QgsVectorLayer("/data/boundaries.geojson", "Boundaries", "ogr")
```

#### FlatGeoBuf
```python
vlayer = QgsVectorLayer("/data/parcels.fgb", "Parcels", "ogr")
```

#### KML
```python
vlayer = QgsVectorLayer("/data/places.kml", "Places", "ogr")
```

#### DXF (with geometry type filter)
```python
vlayer = QgsVectorLayer(
    "/data/drawing.dxf|layername=entities|geometrytype=Polygon",
    "DXF Polygons",
    "ogr"
)
```
DXF `geometrytype` values: `Point`, `LineString`, `Polygon`.

#### GPX
```python
vlayer = QgsVectorLayer("/data/track.gpx?type=track", "GPS Track", "gpx")
```
GPX `type` values: `track`, `route`, `waypoint`.

#### MySQL via OGR
```python
vlayer = QgsVectorLayer(
    "MySQL:mydb,host=localhost,port=3306,user=root,password=xxx|layername=my_table",
    "MySQL Table",
    "ogr"
)
```

#### Enumerate All GeoPackage Sublayers
```python
from qgis.core import QgsDataProvider, QgsVectorLayer, QgsProject

gpkg_path = "/data/multi_layer.gpkg"
probe = QgsVectorLayer(gpkg_path, "probe", "ogr")
for sub in probe.dataProvider().subLayers():
    name = sub.split(QgsDataProvider.SUBLAYER_SEPARATOR)[1]
    uri = f"{gpkg_path}|layername={name}"
    layer = QgsVectorLayer(uri, name, "ogr")
    if layer.isValid():
        QgsProject.instance().addMapLayer(layer)
```

---

### PostGIS Provider (`"postgres"`)

#### Basic Connection
```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer

uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "user", "password")
uri.setDataSource("public", "roads", "geom")
vlayer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
```

#### With SQL Filter and Primary Key
```python
uri = QgsDataSourceUri()
uri.setConnection("db.example.com", "5432", "gisdb", "analyst", "secret")
uri.setDataSource("public", "parcels", "the_geom", "area_sqm > 1000", "gid")
vlayer = QgsVectorLayer(uri.uri(False), "Large Parcels", "postgres")
```

#### With SSL
```python
uri = QgsDataSourceUri()
uri.setConnection("db.example.com", "5432", "gisdb", "user", "pass",
                   QgsDataSourceUri.SslRequire)
uri.setDataSource("public", "buildings", "geom")
vlayer = QgsVectorLayer(uri.uri(False), "Buildings", "postgres")
```

#### With QGIS Authentication Manager
```python
uri = QgsDataSourceUri()
uri.setConnection("db.example.com", "5432", "gisdb", "", "")
uri.setAuthConfigId("my_auth_config_id")
uri.setDataSource("public", "roads", "geom")
vlayer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
```

---

### SpatiaLite Provider (`"spatialite"`)

```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer

uri = QgsDataSourceUri()
uri.setDatabase("/data/regions.sqlite")
uri.setDataSource("", "towns", "geometry")
vlayer = QgsVectorLayer(uri.uri(), "Towns", "spatialite")
```

---

### Memory Provider (`"memory"`)

#### Geometry Types
```
"Point?crs=EPSG:4326"
"LineString?crs=EPSG:28992"
"Polygon?crs=EPSG:3857"
"MultiPoint?crs=EPSG:4326"
"MultiLineString?crs=EPSG:4326"
"MultiPolygon?crs=EPSG:4326"
"None"                          # Attribute-only table (no geometry)
```

#### With Inline Field Definitions
```python
uri = "Point?crs=EPSG:4326&field=name:string(100)&field=value:double&field=count:integer"
vlayer = QgsVectorLayer(uri, "Points", "memory")
```

Field type keywords: `string`, `integer`, `double`, `date`, `datetime`, `boolean`.

#### Materialize a Selection
```python
from qgis.core import QgsFeatureRequest

memory_layer = source_layer.materialize(
    QgsFeatureRequest().setFilterFids(source_layer.selectedFeatureIds())
)
```

---

### Delimited Text Provider (`"delimitedtext"`)

#### CSV with X/Y Coordinate Columns
```python
import os
from qgis.core import QgsVectorLayer

path = os.path.abspath("data/stations.csv").replace("\\", "/")
uri = f"file:///{path}?delimiter=,&xField=longitude&yField=latitude&crs=EPSG:4326"
vlayer = QgsVectorLayer(uri, "Stations", "delimitedtext")
```

#### TSV (Tab-Separated)
```python
uri = f"file:///{path}?delimiter=\\t&xField=x&yField=y&crs=EPSG:28992"
vlayer = QgsVectorLayer(uri, "Measurements", "delimitedtext")
```

#### CSV with WKT Geometry Column
```python
uri = f"file:///{path}?delimiter=,&wktField=geom&crs=EPSG:4326&geomType=polygon"
vlayer = QgsVectorLayer(uri, "Polygons", "delimitedtext")
```

#### CSV without Geometry (Attribute Table Only)
```python
uri = f"file:///{path}?delimiter=,&geomType=none"
vlayer = QgsVectorLayer(uri, "Lookup Table", "delimitedtext")
```

URI parameters reference:
- `file:///` — prefix + absolute path (three slashes for absolute paths)
- `delimiter` — field separator: `,`, `;`, `\t`
- `xField` / `yField` — column names for coordinates
- `wktField` — alternative: column name containing WKT geometry
- `crs` — CRS identifier (e.g., `EPSG:4326`)
- `geomType` — geometry type hint: `point`, `line`, `polygon`, `none`

---

### WFS Provider (`"WFS"`)

**Note**: The provider key is `"WFS"` (uppercase). This differs from most other provider keys.

```python
uri = "https://example.com/wfs?service=WFS&version=2.0.0&request=GetFeature&typename=ns:cities"
vlayer = QgsVectorLayer(uri, "Cities", "WFS")
```

#### With Bounding Box Filter
```python
uri = (
    "https://example.com/wfs?service=WFS&version=2.0.0"
    "&request=GetFeature&typename=ns:buildings"
    "&bbox=5.0,52.0,6.0,53.0,EPSG:4326"
)
vlayer = QgsVectorLayer(uri, "Buildings", "WFS")
```

---

### Virtual Layer Provider (`"virtual"`)

```python
# Query across layers loaded in the current project
uri = "?query=SELECT * FROM airports WHERE elevation > 500"
vlayer = QgsVectorLayer(uri, "High Airports", "virtual")

# Join two layers
uri = "?query=SELECT a.*, b.population FROM cities a JOIN stats b ON a.id = b.city_id"
vlayer = QgsVectorLayer(uri, "Cities with Stats", "virtual")
```

---

## Raster Providers

### GDAL Provider (`"gdal"`)

#### GeoTIFF
```python
rlayer = QgsRasterLayer("/data/srtm.tif", "Elevation", "gdal")
```

#### GeoPackage Raster
```python
rlayer = QgsRasterLayer("GPKG:/data/rasters.gpkg:elevation", "Elevation", "gdal")
```

#### Cloud Optimized GeoTIFF (COG) via /vsicurl/
```python
rlayer = QgsRasterLayer(
    "/vsicurl/https://example.com/data/cog.tif",
    "Remote COG",
    "gdal"
)
```

#### JPEG2000
```python
rlayer = QgsRasterLayer("/data/ortho.jp2", "Orthophoto", "gdal")
```

#### Virtual Raster (VRT)
```python
rlayer = QgsRasterLayer("/data/mosaic.vrt", "Mosaic", "gdal")
```

---

### WMS / WMTS / XYZ Provider (`"wms"`)

**Note**: The `"wms"` provider key handles WMS, WMTS, AND XYZ tile sources.

#### WMS
```python
uri = (
    "crs=EPSG:4326"
    "&format=image/png"
    "&layers=boundaries"
    "&styles"
    "&url=https://example.com/wms"
)
rlayer = QgsRasterLayer(uri, "WMS Boundaries", "wms")
```

WMS URI parameters:
- `url` — WMS service endpoint URL
- `layers` — comma-separated layer names
- `styles` — comma-separated style names (can be empty)
- `crs` — CRS identifier
- `format` — image format (`image/png`, `image/jpeg`)
- `username` / `password` — optional authentication

#### WMTS
```python
uri = (
    "crs=EPSG:3857"
    "&format=image/png"
    "&layers=topographic"
    "&styles=default"
    "&tileMatrixSet=GoogleMapsCompatible"
    "&url=https://example.com/wmts?service=WMTS&request=GetCapabilities"
)
rlayer = QgsRasterLayer(uri, "WMTS Topo", "wms")
```

#### XYZ Tiles
```python
# {z}, {x}, {y} MUST be URL-encoded as %7Bz%7D, %7Bx%7D, %7By%7D
uri = (
    "type=xyz"
    "&url=https://tile.openstreetmap.org/%7Bz%7D/%7Bx%7D/%7By%7D.png"
    "&zmin=0&zmax=19"
    "&crs=EPSG3857"
)
rlayer = QgsRasterLayer(uri, "OpenStreetMap", "wms")
```

Common XYZ tile sources:
```python
# OpenStreetMap
"type=xyz&url=https://tile.openstreetmap.org/%7Bz%7D/%7Bx%7D/%7By%7D.png&zmin=0&zmax=19&crs=EPSG3857"

# Esri World Imagery
"type=xyz&url=https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/%7Bz%7D/%7By%7D/%7Bx%7D&zmin=0&zmax=18&crs=EPSG3857"
```

---

### WCS Provider (`"wcs"`)

```python
uri = "https://example.com/wcs?identifier=dem_layer"
rlayer = QgsRasterLayer(uri, "DEM Coverage", "wcs")
```

WCS URI parameters: `url`, `identifier`, `time`, `format`, `crs`, `username`, `password`, `cache`.

---

### PostGIS Raster Provider (`"postgresraster"`)

```python
from qgis.core import QgsDataSourceUri, QgsRasterLayer

uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "user", "password")
uri.setDataSource("public", "elevation_raster", "rast")
rlayer = QgsRasterLayer(uri.uri(False), "Elevation", "postgresraster")
```

---

## Point Cloud Providers (QGIS 3.18+)

### PDAL Provider (`"pdal"`)
```python
from qgis.core import QgsPointCloudLayer

pclayer = QgsPointCloudLayer("/data/lidar.las", "LiDAR", "pdal")
# Also supports: .laz (compressed)
```

### COPC Provider (`"copc"`) — QGIS 3.26+
```python
pclayer = QgsPointCloudLayer("/data/cloud.copc.laz", "COPC Cloud", "copc")
```

### EPT Provider (`"ept"`)
```python
pclayer = QgsPointCloudLayer("/data/ept/ept.json", "EPT Cloud", "ept")
```

---

## Mesh Provider

### MDAL Provider (`"mdal"`)
```python
from qgis.core import QgsMeshLayer

mlayer = QgsMeshLayer("/data/flow.nc", "Flow Model", "mdal")
# Supports: NetCDF, GRIB, XMDF, DAT, and other mesh formats
```

---

## Vector Tile Provider

### Mapbox Vector Tiles
```python
from qgis.core import QgsVectorTileLayer

uri = "type=xyz&url=https://example.com/tiles/{z}/{x}/{y}.pbf&zmin=0&zmax=14"
vtlayer = QgsVectorTileLayer(uri, "Vector Tiles")
```

---

## 3D Tiled Scene Provider (QGIS 3.34+)

### Cesium 3D Tiles
```python
from qgis.core import QgsTiledSceneLayer

layer = QgsTiledSceneLayer("url=https://example.com/tileset.json", "3D Buildings", "cesiumtiles")
```
