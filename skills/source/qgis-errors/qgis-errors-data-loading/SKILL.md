---
name: qgis-errors-data-loading
description: >
  Use when diagnosing data loading failures, invalid layers, or provider connection errors.
  Prevents silent failures from unchecked layer validity and incorrect URI formats.
  Covers invalid layer detection, URI format errors, encoding issues, GeoPackage locking, connection failures, and performance.
  Keywords: invalid layer, isValid, data loading error, URI format, encoding, GeoPackage lock, connection failed, provider error, layer won't load, can't open shapefile.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-errors-data-loading

## Quick Reference

### The #1 Rule

**ALWAYS** call `layer.isValid()` immediately after creating ANY layer. An invalid layer does NOT raise an exception — it silently returns a broken object that crashes or produces undefined behavior on access.

```python
layer = QgsVectorLayer(uri, name, provider)
if not layer.isValid():
    raise RuntimeError(f"Failed to load layer '{name}': check URI, provider, and data source")
```

### Error Categories at a Glance

| Error Category | Symptoms | Typical Root Cause |
|----------------|----------|--------------------|
| Invalid layer | `isValid()` returns `False` | Wrong path, bad URI, missing provider |
| Wrong URI format | Layer loads with 0 features or fails silently | Provider-specific URI syntax violated |
| Encoding issues | Garbled attribute text, mojibake | Shapefile .dbf encoding mismatch |
| GeoPackage locking | "database is locked" error | Concurrent write access to .gpkg |
| PostGIS connection failure | Timeout, auth error, empty layer | Network, credentials, permissions |
| WMS/WFS errors | Blank tiles, timeout, parse failure | Service URL, capabilities, CRS mismatch |
| Large dataset performance | Memory exhaustion, UI freeze | Loading all features into memory at once |
| Missing CRS | Layer places at wrong location | No .prj file or CRS metadata absent |
| Shapefile limitations | Truncated field names, 2GB cap | Format constraints hit |

### Critical Warnings

**NEVER** access features, attributes, or data provider methods on an invalid layer — this leads to crashes or undefined behavior.

**NEVER** use backslashes in file paths or URIs — QGIS/Qt normalizes to forward slashes internally. Backslashes in URIs cause provider failures.

**NEVER** pass `True` to `uri.uri(expandAuthConfig)` when logging or displaying URIs — this exposes credentials in plain text. ALWAYS use `uri.uri(False)`.

**NEVER** hardcode absolute file paths in cross-platform code. ALWAYS use `os.path.join()` or `pathlib.Path` for path construction.

---

## Error Catalog

### E-001: layer.isValid() Returns False

**Symptoms**: Layer object exists but `isValid()` returns `False`. No exception raised. Adding to project shows a broken layer icon.

**Root causes (check in order)**:
1. File does not exist at the specified path
2. URI format is wrong for the chosen provider
3. Provider name is misspelled or missing
4. Authentication credentials are missing or incorrect
5. CRS database is unavailable (standalone scripts without `setPrefixPath`)
6. Data file is corrupted or truncated

**Fix pattern**:

```python
import os
from qgis.core import QgsVectorLayer

path = "/data/airports.shp"

# Step 1: Verify file exists
if not os.path.exists(path):
    raise FileNotFoundError(f"Data file not found: {path}")

# Step 2: Load with explicit provider
layer = QgsVectorLayer(path, "Airports", "ogr")

# Step 3: ALWAYS check validity
if not layer.isValid():
    # Step 4: Inspect error string for details
    error = layer.dataProvider().error().message() if layer.dataProvider() else "No provider"
    raise RuntimeError(f"Layer invalid: {error}")
```

### E-002: Wrong URI Format for Provider

**Symptoms**: Layer is invalid or loads with 0 features. No error message.

**Correct URI formats by provider**:

| Provider | Correct URI | Common Mistake |
|----------|-------------|----------------|
| `ogr` (Shapefile) | `"/path/to/file.shp"` | Missing file extension |
| `ogr` (GeoPackage) | `"/path/to/file.gpkg\|layername=roads"` | Missing `\|layername=` for multi-layer GPKG |
| `ogr` (GeoJSON) | `"/path/to/file.geojson"` | Using `"geojson"` as provider name |
| `postgres` | Use `QgsDataSourceUri` object | Manual string concatenation |
| `wms` | `"crs=EPSG:4326&format=image/png&layers=name&styles&url=https://..."` | Missing `url=` parameter |
| `WFS` | `"https://server/wfs?service=WFS&version=2.0.0&request=GetFeature&typename=ns:layer"` | Wrong provider name (must be uppercase `"WFS"`) |
| `delimitedtext` | `"file:///path/to/file.csv?delimiter=,&xField=lon&yField=lat"` | Missing `file://` prefix |
| `memory` | `"Point?crs=EPSG:4326&field=name:string(50)"` | Missing geometry type |
| `spatialite` | Use `QgsDataSourceUri` object | Raw path without URI builder |
| `gpx` | `"/path/to/file.gpx?type=track"` | Missing `?type=` parameter |

**Fix pattern for GeoPackage**:

```python
# WRONG: loads first layer or fails silently
layer = QgsVectorLayer("/data/data.gpkg", "My Layer", "ogr")

# CORRECT: explicit layer name
layer = QgsVectorLayer("/data/data.gpkg|layername=roads", "Roads", "ogr")
```

**Fix pattern for PostGIS**:

```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer

# ALWAYS use QgsDataSourceUri: NEVER concatenate strings
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "mydb", "user", "pass")
uri.setDataSource("public", "roads", "geom", "", "gid")

layer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
```

### E-003: Character Encoding Issues in Shapefile .dbf Files

**Symptoms**: Attribute values contain garbled characters (mojibake). Accented characters display incorrectly. Field values show `Ã©` instead of `é`.

**Root cause**: Shapefile .dbf files use a code page byte in the header, but many tools write it incorrectly or omit it. QGIS/OGR falls back to system encoding, which may not match the actual encoding.

**Fix pattern**:

```python
from qgis.core import QgsVectorLayer

# Method 1: Set encoding via open options in the URI
layer = QgsVectorLayer(
    "/data/old_data.shp|option:ENCODING=UTF-8",
    "Data", "ogr"
)

# Method 2: Create a .cpg file next to the .shp with the encoding name
# Write "UTF-8" or "ISO-8859-1" to /data/old_data.cpg

# Method 3: Set encoding after loading (for display only)
layer = QgsVectorLayer("/data/old_data.shp", "Data", "ogr")
if layer.isValid():
    layer.dataProvider().setEncoding("UTF-8")
```

**Common encodings to try**: `UTF-8`, `ISO-8859-1` (Latin-1), `Windows-1252`, `ISO-8859-15` (Latin-9 with Euro sign).

### E-004: GeoPackage File Locking (Concurrent Access)

**Symptoms**: `"database is locked"` error. Write operations fail. Layer becomes read-only unexpectedly.

**Root cause**: GeoPackage uses SQLite, which has limited concurrent write support. Multiple processes or QGIS instances writing to the same .gpkg file cause locking conflicts. WAL (Write-Ahead Logging) mode can help but does not eliminate all issues.

**Fix pattern**:

```python
# Prevention: Use journal_mode=WAL for better concurrent read support
# Set GDAL config option BEFORE loading
from osgeo import gdal
gdal.SetConfigOption("OGR_SQLITE_JOURNAL", "WAL")

# Detection: Check for lock files
import os
gpkg_path = "/data/project.gpkg"
lock_files = [
    gpkg_path + "-wal",
    gpkg_path + "-shm",
    gpkg_path + "-journal"
]
for lock_file in lock_files:
    if os.path.exists(lock_file):
        print(f"Lock file present: {lock_file}")

# Resolution strategies:
# 1. Close all other QGIS instances accessing the file
# 2. Delete stale lock files (-wal, -shm, -journal) ONLY if no process is using the file
# 3. Copy the file, work on the copy, then replace the original
# 4. For multi-user workflows: use PostGIS instead of GeoPackage
```

### E-005: PostGIS Connection Failures

**Symptoms**: Layer is invalid. Error messages include "could not connect to server", "authentication failed", "permission denied", or "relation does not exist".

**Diagnostic checklist**:
1. **Network**: Can the machine reach the database host and port?
2. **Authentication**: Are username/password correct? Is the auth method configured in `pg_hba.conf`?
3. **Database**: Does the database exist? Does the user have CONNECT privilege?
4. **Schema/Table**: Does the table exist in the specified schema? Does the user have SELECT privilege?
5. **Geometry column**: Does the specified geometry column exist in the table?
6. **Primary key**: Is the specified key column a valid unique column?

**Fix pattern**:

```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer

uri = QgsDataSourceUri()
uri.setConnection("dbhost", "5432", "gisdb", "gisuser", "password")
uri.setDataSource("public", "parcels", "geom", "", "gid")

layer = QgsVectorLayer(uri.uri(False), "Parcels", "postgres")
if not layer.isValid():
    # Check provider error for specific failure reason
    provider = layer.dataProvider()
    if provider:
        print(f"Provider error: {provider.error().message()}")
    else:
        print("Provider could not be instantiated — check connection parameters")

# For production: ALWAYS use QgsAuthManager instead of plain passwords
uri_secure = QgsDataSourceUri()
uri_secure.setConnection("dbhost", "5432", "gisdb", "", "")
uri_secure.setAuthConfigId("my_authcfg_id")  # Stored in encrypted qgis-auth.db
uri_secure.setDataSource("public", "parcels", "geom", "", "gid")
```

**NEVER** store passwords in plain text URIs in production. ALWAYS use `QgsAuthManager` with stored authentication configurations.

**NEVER** load PostGIS views without specifying a unique key column — this causes undefined behavior and performance issues.

### E-006: WMS/WFS Timeout or Capability Parsing Errors

**Symptoms**: WMS layer shows blank tiles or fails to load. WFS layer times out. GetCapabilities returns an XML parsing error.

**Root causes**:
1. Service URL is incorrect or unreachable
2. Layer name does not match the service capabilities
3. CRS is not supported by the service
4. Network timeout is too short for slow services
5. XYZ tile URL placeholders are not URL-encoded

**Fix pattern for WMS**:

```python
from qgis.core import QgsRasterLayer

# CORRECT WMS URI format: note all required parameters
uri = (
    "crs=EPSG:4326"
    "&format=image/png"
    "&layers=my_layer_name"
    "&styles"
    "&url=https://example.com/wms"
)
layer = QgsRasterLayer(uri, "WMS Layer", "wms")
if not layer.isValid():
    print("WMS layer invalid — check URL, layer name, and CRS support")
```

**Fix pattern for XYZ tiles**:

```python
# ALWAYS URL-encode {z}, {x}, {y} placeholders
url = "type=xyz&url=https://tiles.example.com/%7Bz%7D/%7Bx%7D/%7By%7D.png&zmax=19&zmin=0"
layer = QgsRasterLayer(url, "Tiles", "wms")
```

**NEVER** assume a WMS layer supports all CRS — check the GetCapabilities response first.

### E-007: Large Dataset Performance Issues

**Symptoms**: QGIS freezes or runs out of memory. Feature iteration takes excessively long. Script hangs on `getFeatures()`.

**Root cause**: Loading all features into memory at once for large datasets (>1M features) or iterating without spatial/attribute filters.

**Fix pattern**:

```python
from qgis.core import QgsVectorLayer, QgsFeatureRequest, QgsRectangle

layer = QgsVectorLayer("/data/huge_dataset.gpkg|layername=parcels", "Parcels", "ogr")

# WRONG: loads ALL features into memory
all_features = list(layer.getFeatures())  # Memory explosion for large datasets

# CORRECT: use spatial filter to limit features
bbox = QgsRectangle(100000, 400000, 110000, 410000)
request = QgsFeatureRequest().setFilterRect(bbox)
for feature in layer.getFeatures(request):
    # Process feature
    pass

# CORRECT: use attribute filter
request = QgsFeatureRequest().setFilterExpression('"status" = \'active\'')
for feature in layer.getFeatures(request):
    pass

# CORRECT: limit returned attributes for performance
request = QgsFeatureRequest()
request.setSubsetOfAttributes(["name", "area"], layer.fields())
request.setFlags(QgsFeatureRequest.NoGeometry)  # Skip geometry if not needed
for feature in layer.getFeatures(request):
    pass

# CORRECT: use setLimit() to cap result count
request = QgsFeatureRequest().setLimit(1000)
for feature in layer.getFeatures(request):
    pass
```

### E-008: Missing CRS in Loaded Data

**Symptoms**: Layer loads but displays at wrong location. Features cluster at origin (0,0). Layer does not align with other layers.

**Root cause**: Data file has no CRS metadata (missing .prj file for Shapefile, no CRS in GeoJSON, etc.). QGIS assigns a default or no CRS.

**Fix pattern**:

```python
from qgis.core import QgsVectorLayer, QgsCoordinateReferenceSystem

layer = QgsVectorLayer("/data/no_crs.shp", "Data", "ogr")
if layer.isValid():
    # Check if CRS is valid
    if not layer.crs().isValid():
        print(f"WARNING: Layer has no valid CRS")
        # Assign the correct CRS (does NOT reproject — just sets metadata)
        layer.setCrs(QgsCoordinateReferenceSystem("EPSG:28992"))

    # Verify CRS is what you expect
    print(f"Layer CRS: {layer.crs().authid()}")
```

**NEVER** confuse `setCrs()` (assigning metadata) with reprojection (coordinate transformation). `setCrs()` changes the label, not the coordinates. To reproject, use Processing `native:reprojectlayer` or `QgsCoordinateTransform`.

### E-009: Shapefile Limitations

**Symptoms**: Field names truncated to 10 characters. File size capped at 2GB. Date/time fields lose precision. No support for NULL vs empty string.

**Root cause**: Shapefile is a legacy format with hard limitations from dBASE III.

**Known limitations**:

| Limitation | Details |
|-----------|---------|
| Field name length | Maximum 10 characters — names are silently truncated |
| File size | Maximum 2GB per component file (.shp, .dbf) |
| Geometry types | Single geometry type per file — no mixed geometries |
| Character encoding | Relies on .cpg file or code page byte — unreliable |
| No NULL support | Cannot distinguish NULL from empty string or zero |
| Date precision | Date only, no time or datetime support in .dbf |
| No nested fields | No support for JSON, arrays, or nested structures |

**Fix pattern**: Migrate to GeoPackage for any new work:

```python
import processing

# Convert Shapefile to GeoPackage
processing.run("native:package", {
    'LAYERS': [shapefile_layer],
    'OUTPUT': '/data/output.gpkg',
    'OVERWRITE': True
})
```

---

## Diagnostic Flowchart

```
Layer fails to load or behaves unexpectedly
│
├─ Step 1: Does the file/service exist?
│  ├─ NO → Fix the file path or service URL
│  └─ YES ↓
│
├─ Step 2: Is layer.isValid() True?
│  ├─ NO → Check layer.dataProvider().error().message()
│  │  ├─ "Could not connect" → E-005 (PostGIS) or E-006 (WMS/WFS)
│  │  ├─ "not recognized as a supported file format" → E-002 (URI format)
│  │  ├─ "database is locked" → E-004 (GeoPackage locking)
│  │  └─ No provider at all → Wrong provider name in constructor
│  └─ YES ↓
│
├─ Step 3: Does the layer have features?
│  ├─ featureCount() == 0 → Check URI (E-002), filter expression, or empty source
│  └─ featureCount() > 0 ↓
│
├─ Step 4: Are features at the correct location?
│  ├─ NO → Check CRS (E-008) or coordinate order
│  └─ YES ↓
│
├─ Step 5: Are attribute values correct?
│  ├─ Garbled text → E-003 (encoding)
│  ├─ Truncated names → E-009 (Shapefile limits)
│  └─ YES ↓
│
├─ Step 6: Is performance acceptable?
│  ├─ NO → E-007 (large dataset optimization)
│  └─ YES → Layer is working correctly
```

---

## Fix Patterns Summary

| Error | First Action | Fallback Action |
|-------|-------------|-----------------|
| E-001: Invalid layer | Verify file path exists, check provider name | Inspect `dataProvider().error().message()` |
| E-002: Wrong URI | Compare URI against format table above | Use `QgsDataSourceUri` for database providers |
| E-003: Encoding | Create .cpg file with correct encoding | Set encoding via `dataProvider().setEncoding()` |
| E-004: GPKG lock | Close other processes accessing the file | Delete stale lock files, switch to PostGIS |
| E-005: PostGIS | Verify host/port/db/user/password | Check `pg_hba.conf`, schema permissions |
| E-006: WMS/WFS | Verify service URL in browser | Check GetCapabilities for layer names and CRS |
| E-007: Performance | Add spatial filter to `QgsFeatureRequest` | Use `NoGeometry` flag, limit attributes |
| E-008: Missing CRS | Assign CRS with `setCrs()` | Create .prj file alongside Shapefile |
| E-009: Shapefile limits | Migrate to GeoPackage | Accept limitations for legacy workflows |

---

## Reference Links

- [references/methods.md](references/methods.md) — Key API methods for layer loading, validity checking, and error inspection
- [references/examples.md](references/examples.md) — Working diagnostic code examples for each error type
- [references/anti-patterns.md](references/anti-patterns.md) — What NEVER to do when loading data

### Official Sources

- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/loadlayer.html
- https://qgis.org/pyqgis/3.44/core/QgsVectorLayer.html
- https://qgis.org/pyqgis/3.44/core/QgsRasterLayer.html
- https://qgis.org/pyqgis/3.44/core/QgsDataSourceUri.html
- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/cheat_sheet.html
