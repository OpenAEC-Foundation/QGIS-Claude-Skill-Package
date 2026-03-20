---
name: qgis-core-data-providers
description: >
  Use when loading data into QGIS, constructing layer URIs, or choosing between data formats.
  Prevents URI format errors that cause silent layer loading failures.
  Covers 12+ data providers, URI construction patterns, vector/raster/mesh formats, and GeoPackage best practices.
  Keywords: data provider, QgsDataSourceUri, load layer, Shapefile, GeoPackage, GeoJSON, PostGIS, WMS, WFS, memory layer, CSV.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-core-data-providers

## Quick Reference

### Provider System Overview

QGIS uses a plugin-based provider architecture. Providers are registered in `QgsProviderRegistry` and loaded during `QgsApplication.initQgis()`. Each provider handles one or more data formats.

| Provider Key | Type | Formats |
|-------------|------|---------|
| `"ogr"` | Vector | Shapefile, GeoPackage, GeoJSON, FlatGeoBuf, KML, DXF, GPX |
| `"postgres"` | Vector | PostGIS tables and views |
| `"spatialite"` | Vector | SpatiaLite databases |
| `"memory"` | Vector | In-memory temporary layers |
| `"delimitedtext"` | Vector | CSV, TSV, custom-delimited text files |
| `"WFS"` | Vector | OGC Web Feature Service |
| `"virtual"` | Vector | SQL queries across loaded layers |
| `"gdal"` | Raster | GeoTIFF, JPEG2000, COG, VRT, GeoPackage raster |
| `"wms"` | Raster | WMS, WMTS, XYZ tiles |
| `"wcs"` | Raster | OGC Web Coverage Service |
| `"postgresraster"` | Raster | PostGIS raster tables |
| `"pdal"` | Point Cloud | LAS, LAZ (QGIS 3.18+) |
| `"copc"` | Point Cloud | Cloud Optimized Point Cloud (QGIS 3.26+) |
| `"ept"` | Point Cloud | Entwine Point Tile (QGIS 3.18+) |
| `"mdal"` | Mesh | NetCDF, GRIB, XMDF, DAT |
| `"vectortile"` | Vector Tile | Mapbox Vector Tiles (MVT) |
| `"cesiumtiles"` | Tiled Scene | Cesium 3D Tiles (QGIS 3.34+) |

### Layer Construction Pattern

```python
# Vector layer
vlayer = QgsVectorLayer(data_source_uri, display_name, provider_key)

# Raster layer
rlayer = QgsRasterLayer(data_source_uri, display_name, provider_key)

# Point cloud layer (QGIS 3.18+)
pclayer = QgsPointCloudLayer(data_source_uri, display_name, provider_key)

# Mesh layer
mlayer = QgsMeshLayer(data_source_uri, display_name, provider_key)

# Vector tile layer
vtlayer = QgsVectorTileLayer(data_source_uri, display_name)
```

**ALWAYS** check validity immediately after creation:

```python
layer = QgsVectorLayer(uri, name, provider)
if not layer.isValid():
    raise RuntimeError(f"Failed to load layer '{name}' from: {uri}")
```

---

## Critical Warnings

**NEVER** skip `isValid()` after layer creation. A layer object is ALWAYS returned even when loading fails -- the constructor NEVER raises exceptions.

**NEVER** use backslashes in URIs, even on Windows. QGIS/Qt normalizes to forward slashes internally. Backslashes in URIs cause silent provider failures.

**NEVER** pass `True` to `uri.uri(expandAuthConfig)` when logging or displaying URIs. This exposes authentication credentials in plain text. ALWAYS use `uri.uri(False)`.

**NEVER** assume a GeoPackage contains a single layer. ALWAYS use explicit `|layername=` in the URI or enumerate sublayers first.

**ALWAYS** call `layer.updateExtents()` after adding features to a memory layer. Without this, zoom-to-layer returns a wrong extent.

**ALWAYS** call `layer.updateFields()` after calling `dataProvider().addAttributes()`. Without this, the layer schema is stale.

**ALWAYS** use `file:///` prefix (three slashes) for delimited text URIs with absolute paths.

**NEVER** access features or data provider methods on an invalid layer -- this causes crashes or undefined behavior.

---

## Decision Tree: Which Format to Use

```
Need to store spatial data?
├── Temporary / in-memory only?
│   └── USE: memory provider
├── Exchange with non-GIS tools?
│   ├── JSON-based → USE: GeoJSON
│   └── Tabular → USE: CSV with delimitedtext provider
├── Single-layer vector file?
│   ├── Small dataset → USE: GeoPackage (single layer)
│   └── Streaming / append-heavy → USE: FlatGeoBuf
├── Multi-layer project database?
│   └── USE: GeoPackage (recommended default)
├── Enterprise / multi-user database?
│   └── USE: PostGIS with postgres provider
├── Web service?
│   ├── Vector features → USE: WFS
│   ├── Map images → USE: WMS
│   ├── Tile basemaps → USE: XYZ tiles via wms provider
│   └── Raw raster coverage → USE: WCS
├── Raster data?
│   ├── Local file → USE: GeoTIFF via gdal provider
│   ├── Cloud storage → USE: COG via /vsicurl/
│   └── Database → USE: postgresraster
├── Point cloud / LiDAR?
│   ├── Local file → USE: pdal (LAS/LAZ)
│   └── Cloud optimized → USE: copc
└── Legacy requirement?
    └── Shapefile ONLY if mandated by external system
```

**GeoPackage is the recommended default format.** It supports vector, raster, and attribute tables in a single SQLite-based file with no file count limitations (unlike Shapefile's multi-file structure).

---

## Essential Patterns

### Pattern 1: Load a GeoPackage Layer

```python
from qgis.core import QgsVectorLayer, QgsProject

# Single known layer
vlayer = QgsVectorLayer("data/project.gpkg|layername=buildings", "Buildings", "ogr")
if not vlayer.isValid():
    raise RuntimeError("Layer failed to load")
QgsProject.instance().addMapLayer(vlayer)
```

### Pattern 2: Enumerate All Sublayers

```python
from qgis.core import QgsDataProvider, QgsVectorLayer, QgsProject

gpkg_path = "data/project.gpkg"
layer = QgsVectorLayer(gpkg_path, "probe", "ogr")
for sub in layer.dataProvider().subLayers():
    name = sub.split(QgsDataProvider.SUBLAYER_SEPARATOR)[1]
    uri = f"{gpkg_path}|layername={name}"
    sub_layer = QgsVectorLayer(uri, name, "ogr")
    if sub_layer.isValid():
        QgsProject.instance().addMapLayer(sub_layer)
```

### Pattern 3: PostGIS Connection with QgsDataSourceUri

```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer

uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "mydb", "user", "password")
uri.setDataSource("public", "roads", "geom", "status = 'active'", "gid")

vlayer = QgsVectorLayer(uri.uri(False), "Active Roads", "postgres")
if not vlayer.isValid():
    raise RuntimeError("PostGIS connection failed")
```

### Pattern 4: Create a Memory Layer with Fields

```python
from qgis.core import QgsVectorLayer, QgsField, QgsFeature, QgsGeometry, QgsPointXY
from qgis.PyQt.QtCore import QVariant

# URI with inline field definitions
layer = QgsVectorLayer(
    "Point?crs=EPSG:4326&field=name:string(100)&field=value:double",
    "Results",
    "memory"
)

# OR add fields programmatically
layer = QgsVectorLayer("Point?crs=EPSG:4326", "Results", "memory")
pr = layer.dataProvider()
pr.addAttributes([
    QgsField("name", QVariant.String),
    QgsField("value", QVariant.Double),
])
layer.updateFields()

# Add features
feat = QgsFeature()
feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(5.0, 52.0)))
feat.setAttributes(["Sample", 42.0])
pr.addFeatures([feat])
layer.updateExtents()
```

### Pattern 5: Load WMS / XYZ Tiles

```python
from qgis.core import QgsRasterLayer, QgsProject

# WMS
wms_uri = (
    "crs=EPSG:4326"
    "&format=image/png"
    "&layers=my_layer"
    "&styles"
    "&url=https://example.com/wms"
)
wms_layer = QgsRasterLayer(wms_uri, "WMS Layer", "wms")

# XYZ tiles -- {z}/{x}/{y} MUST be URL-encoded
xyz_uri = (
    "type=xyz"
    "&url=https://tile.openstreetmap.org/%7Bz%7D/%7Bx%7D/%7By%7D.png"
    "&zmin=0&zmax=19"
    "&crs=EPSG3857"
)
xyz_layer = QgsRasterLayer(xyz_uri, "OpenStreetMap", "wms")

for lyr in [wms_layer, xyz_layer]:
    if not lyr.isValid():
        raise RuntimeError(f"Layer '{lyr.name()}' failed to load")
    QgsProject.instance().addMapLayer(lyr)
```

### Pattern 6: Load CSV with Coordinates

```python
import os
from qgis.core import QgsVectorLayer

csv_path = os.path.abspath("data/stations.csv").replace("\\", "/")
uri = (
    f"file:///{csv_path}"
    "?delimiter=,"
    "&xField=longitude"
    "&yField=latitude"
    "&crs=EPSG:4326"
)
vlayer = QgsVectorLayer(uri, "Stations", "delimitedtext")
```

---

## Common Operations

### List Available Providers

```python
from qgis.core import QgsProviderRegistry

registry = QgsProviderRegistry.instance()
for key in registry.providerList():
    print(key)
```

### Load Raster from GeoPackage

```python
from qgis.core import QgsRasterLayer

rlayer = QgsRasterLayer("GPKG:/data/rasters.gpkg:elevation", "Elevation", "gdal")
```

### Load Cloud Optimized GeoTIFF (COG)

```python
from qgis.core import QgsRasterLayer

rlayer = QgsRasterLayer("/vsicurl/https://example.com/data.tif", "Remote COG", "gdal")
```

### Load WFS Layer

```python
from qgis.core import QgsVectorLayer

uri = "https://example.com/wfs?service=WFS&version=2.0.0&request=GetFeature&typename=ns:layer"
vlayer = QgsVectorLayer(uri, "WFS Layer", "WFS")
```

### Load SpatiaLite Layer

```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer

uri = QgsDataSourceUri()
uri.setDatabase("/data/regions.sqlite")
uri.setDataSource("", "regions", "geometry")
vlayer = QgsVectorLayer(uri.uri(), "Regions", "spatialite")
```

### Virtual Layer (SQL Across Loaded Layers)

```python
from qgis.core import QgsVectorLayer

uri = "?query=SELECT * FROM airports WHERE elevation > 500"
vlayer = QgsVectorLayer(uri, "High Airports", "virtual")
```

### Materialize Selection as Memory Layer

```python
from qgis.core import QgsFeatureRequest, QgsProject

memory_layer = source_layer.materialize(
    QgsFeatureRequest().setFilterFids(source_layer.selectedFeatureIds())
)
QgsProject.instance().addMapLayer(memory_layer)
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for QgsDataSourceUri, QgsVectorLayer, QgsRasterLayer, QgsProviderRegistry
- [references/examples.md](references/examples.md) -- URI format strings for every supported provider
- [references/anti-patterns.md](references/anti-patterns.md) -- What NOT to do when loading data

### Official Sources

- https://qgis.org/pyqgis/master/core/QgsVectorLayer.html
- https://qgis.org/pyqgis/master/core/QgsRasterLayer.html
- https://qgis.org/pyqgis/master/core/QgsDataSourceUri.html
- https://qgis.org/pyqgis/master/core/QgsProviderRegistry.html
- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/loadlayer.html
