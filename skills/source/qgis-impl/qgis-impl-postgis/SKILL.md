---
name: qgis-impl-postgis
description: >
  Use when connecting to PostGIS databases, loading spatial tables, or executing SQL from PyQGIS.
  Prevents connection string errors and plain-text password exposure in production.
  Covers QgsDataSourceUri, PostGIS layer loading (table/view/SQL), SQL execution, schema discovery, and authentication.
  Keywords: PostGIS, QgsDataSourceUri, PostgreSQL, spatial database, SQL, database connection, schema, PostGIS raster.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-impl-postgis

## Quick Reference

### Core Classes

| Class | Purpose | Module |
|-------|---------|--------|
| `QgsDataSourceUri` | Build PostGIS connection URIs | `qgis.core` |
| `QgsVectorLayer` | Load vector layers from PostGIS | `qgis.core` |
| `QgsRasterLayer` | Load raster layers from PostGIS | `qgis.core` |
| `QgsProviderRegistry` | Access provider metadata and connections | `qgis.core` |
| `QgsAbstractDatabaseProviderConnection` | Execute SQL, discover schemas/tables | `qgis.core` |
| `QgsAuthManager` | Secure credential storage and retrieval | `qgis.core` |
| `QgsVectorFileWriter` | Export layers to PostGIS | `qgis.core` |

### QgsDataSourceUri Key Methods

| Method | Parameters | Purpose |
|--------|-----------|---------|
| `setConnection()` | host, port, dbname, user, password | Set connection parameters |
| `setDataSource()` | schema, table, geomColumn, sql, keyColumn | Set table and geometry |
| `setAuthConfigId()` | configId | Attach auth config (replaces user/password) |
| `setSrid()` | srid | Set spatial reference ID |
| `setWkbType()` | wkbType | Set geometry type |
| `setUseEstimatedMetadata()` | flag | Enable estimated metadata for performance |
| `setKeyColumn()` | column | Set primary key column |
| `uri()` | expandAuthConfig (bool) | Return URI string; use `False` to hide credentials |

### Provider Names

| Provider | String | Use Case |
|----------|--------|----------|
| PostGIS Vector | `"postgres"` | Vector tables, views, SQL queries |
| PostGIS Raster | `"postgresraster"` | Raster tables stored in PostGIS |

---

## Critical Warnings

**NEVER** store passwords in plain-text URIs in production code. ALWAYS use `QgsAuthManager` with stored authentication configurations (`authcfg`). Plain-text passwords leak into logs, project files, and connection strings.

**NEVER** load PostGIS views without specifying a unique key column -- this causes undefined behavior, duplicate features, and severe performance degradation. ALWAYS pass the key column as the 5th argument to `setDataSource()`.

**ALWAYS** use `uri.uri(False)` when passing URIs to `QgsVectorLayer` or `QgsRasterLayer` constructors. Passing `True` (or omitting the argument) expands the `authcfg` reference and exposes credentials.

**ALWAYS** use `QgsDataSourceUri` to construct connection strings. NEVER manually concatenate URI strings -- this causes escaping errors with special characters in passwords, table names, or schema names.

**ALWAYS** check `layer.isValid()` immediately after constructing a PostGIS layer. Invalid layers fail silently and produce empty results.

**NEVER** use `estimatedmetadata=true` for layers where row count accuracy matters (e.g., feature counting for reports). Estimated metadata skips expensive table scans but returns approximate counts.

---

## Decision Tree

```
Need to work with PostGIS?
├── Loading a layer?
│   ├── From a table → setDataSource(schema, table, geomCol)
│   ├── From a view → setDataSource(schema, view, geomCol, "", keyCol)  # key required
│   ├── From SQL query → setDataSource("", "(SELECT ...)", geomCol, "", keyCol)
│   └── Raster data → Use "postgresraster" provider with encodeUri()
├── Executing SQL?
│   ├── Non-spatial query → conn.executeSql("SELECT ...")
│   └── Spatial result needed → Load as SQL query layer (above)
├── Discovering schema?
│   ├── List schemas → conn.schemas()
│   └── List tables → conn.tables(schemaName)
├── Exporting to PostGIS?
│   ├── Processing toolbox → processing.run("native:importintopostgis", {...})
│   └── Direct export → QgsVectorFileWriter.writeAsVectorFormatV3()
└── Authentication?
    ├── Development/testing → setConnection() with user/password (temporary only)
    └── Production → setAuthConfigId() with QgsAuthManager config
```

---

## Essential Patterns

### Pattern 1: Connect and Load a Table

```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer

uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "", "")
uri.setAuthConfigId("my_postgis_auth")  # stored auth config
uri.setDataSource("public", "roads", "geom")

layer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
assert layer.isValid(), f"Layer failed to load: {layer.error().message()}"

QgsProject.instance().addMapLayer(layer)
```

### Pattern 2: Load a View (Key Column Required)

```python
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "", "")
uri.setAuthConfigId("my_postgis_auth")
# 5th argument = unique key column -- REQUIRED for views
uri.setDataSource("public", "roads_summary_view", "geom", "", "view_id")

layer = QgsVectorLayer(uri.uri(False), "Roads Summary", "postgres")
assert layer.isValid()
```

### Pattern 3: Load a SQL Query as Layer

```python
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "", "")
uri.setAuthConfigId("my_postgis_auth")

sql = "(SELECT r.gid, r.name, r.geom FROM roads r WHERE r.type = 'highway')"
uri.setDataSource("", sql, "geom", "", "gid")

layer = QgsVectorLayer(uri.uri(False), "Highways", "postgres")
assert layer.isValid()
```

### Pattern 4: Load with SQL Filter and Performance Options

```python
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "", "")
uri.setAuthConfigId("my_postgis_auth")

# SQL filter as 4th argument to setDataSource
uri.setDataSource("public", "parcels", "geom", "area_sqm > 1000", "gid")
uri.setUseEstimatedMetadata(True)  # faster for large tables
uri.setParam("checkPrimaryKeyUnicity", "0")  # skip uniqueness check

layer = QgsVectorLayer(uri.uri(False), "Large Parcels", "postgres")
assert layer.isValid()
```

### Pattern 5: Execute SQL Queries

```python
from qgis.core import QgsProviderRegistry, QgsDataSourceUri

# Option A: Using a stored connection name
md = QgsProviderRegistry.instance().providerMetadata("postgres")
conn = md.createConnection("My PostGIS Server")  # stored connection name

results = conn.executeSql("SELECT count(*) FROM public.roads")
print(f"Row count: {results[0][0]}")

# Option B: Using a URI
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "", "")
uri.setAuthConfigId("my_postgis_auth")

conn = md.createConnection(uri.uri(False), {})
results = conn.executeSql("SELECT DISTINCT road_type FROM public.roads ORDER BY road_type")
```

### Pattern 6: Schema and Table Discovery

```python
from qgis.core import QgsProviderRegistry

md = QgsProviderRegistry.instance().providerMetadata("postgres")
conn = md.createConnection("My PostGIS Server")

# List all schemas
schemas = conn.schemas()
for schema in schemas:
    print(f"Schema: {schema}")

# List tables in a schema
tables = conn.tables("public")
for table in tables:
    print(f"Table: {table.tableName()}, "
          f"Geometry: {table.geometryColumn()}, "
          f"Type: {table.geometryColumnTypes()}")
```

### Pattern 7: Export Layer to PostGIS

```python
import processing

# Using the Processing algorithm (recommended)
result = processing.run("native:importintopostgis", {
    'INPUT': source_layer,
    'DATABASE': 'My PostGIS Server',  # stored connection name
    'SCHEMA': 'public',
    'TABLENAME': 'exported_roads',
    'PRIMARY_KEY': 'id',
    'GEOMETRY_COLUMN': 'geom',
    'ENCODING': 'UTF-8',
    'OVERWRITE': True,
    'CREATEINDEX': True,
    'LOWERCASE_NAMES': True,
    'DROP_STRING_LENGTH': False,
    'FORCE_SINGLEPART': False
})
```

### Pattern 8: Export with QgsVectorFileWriter

```python
from qgis.core import QgsVectorFileWriter, QgsCoordinateTransformContext

save_options = QgsVectorFileWriter.SaveVectorOptions()
save_options.driverName = "PostgreSQL"
save_options.layerName = "new_table"

# ALWAYS use authcfg in production instead of plain password
uri_str = "PG:host=localhost port=5432 dbname=gisdb authcfg=my_postgis_auth"
error = QgsVectorFileWriter.writeAsVectorFormatV3(
    layer,
    uri_str,
    QgsCoordinateTransformContext(),
    save_options
)
if error[0] != QgsVectorFileWriter.NoError:
    raise RuntimeError(f"Export failed: {error[1]}")
```

---

## PostGIS Raster Loading

```python
from qgis.core import QgsProviderRegistry, QgsDataSourceUri, QgsRasterLayer

uri_config = {
    'dbname': 'gisdb',
    'host': 'localhost',
    'port': '5432',
    'sslmode': QgsDataSourceUri.SslDisable,
    'authcfg': 'my_postgis_auth',
    'schema': 'public',
    'table': 'elevation_tiles',
    'geometrycolumn': 'rast',
    'mode': '2'  # 0=one tile per row, 1=one layer per row, 2=union all tiles
}

md = QgsProviderRegistry.instance().providerMetadata('postgresraster')
uri = QgsDataSourceUri(md.encodeUri(uri_config))

rlayer = QgsRasterLayer(uri.uri(False), "Elevation", "postgresraster")
assert rlayer.isValid(), f"Raster layer failed: {rlayer.error().message()}"

QgsProject.instance().addMapLayer(rlayer)
```

### Raster Mode Values

| Mode | Behavior |
|------|----------|
| `0` | Load one tile per row as separate band |
| `1` | Load one raster layer per row |
| `2` | Union all raster tiles into a single layer (most common) |

---

## Authentication

### Using QgsAuthManager (Production)

```python
from qgis.core import QgsApplication, QgsAuthMethodConfig

# Create a new auth config
auth_mgr = QgsApplication.authManager()
config = QgsAuthMethodConfig()
config.setName("My PostGIS Server")
config.setMethod("Basic")
config.setConfig("username", "db_user")
config.setConfig("password", "db_password")

# Store it -- returns (success, config) with config.id() populated
success, config = auth_mgr.storeAuthenticationConfig(config)
assert success, "Failed to store auth config"
auth_config_id = config.id()  # e.g., "abc123"

# Use in URI
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "", "")
uri.setAuthConfigId(auth_config_id)
uri.setDataSource("public", "roads", "geom")

layer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
```

### Retrieving an Existing Auth Config

```python
auth_mgr = QgsApplication.authManager()
config = QgsAuthMethodConfig()
auth_mgr.loadAuthenticationConfig("abc123", config, True)
# config now contains stored credentials
```

---

## Connection Management

### Stored Connections vs URI

| Approach | When to Use |
|----------|------------|
| Stored connection name | QGIS Desktop with Data Source Manager configured; Processing algorithms |
| `QgsDataSourceUri` + `authcfg` | Scripting, plugins, automated workflows |
| `QgsDataSourceUri` + user/password | Development/testing ONLY -- NEVER in production |

### Using Stored Connections

```python
from qgis.core import QgsProviderRegistry

md = QgsProviderRegistry.instance().providerMetadata("postgres")

# List all stored connections
connections = md.connections()
for name, conn in connections.items():
    print(f"Connection: {name}")

# Use a stored connection
conn = md.createConnection("My PostGIS Server")
tables = conn.tables("public")
```

---

## Common Operations

### Check if PostGIS Extension is Installed

```python
conn = md.createConnection("My PostGIS Server")
result = conn.executeSql("SELECT PostGIS_Version()")
print(f"PostGIS version: {result[0][0]}")
```

### Create a Spatial Index

```python
conn.executeSql("CREATE INDEX IF NOT EXISTS idx_roads_geom ON public.roads USING GIST (geom)")
```

### Vacuum and Analyze

```python
conn.executeSql("VACUUM ANALYZE public.roads")
```

### Count Features with Filter

```python
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "gisdb", "", "")
uri.setAuthConfigId("my_postgis_auth")
uri.setDataSource("public", "roads", "geom", "road_type = 'highway'")

layer = QgsVectorLayer(uri.uri(False), "Highways", "postgres")
assert layer.isValid()
print(f"Feature count: {layer.featureCount()}")
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- Complete API signatures for QgsDataSourceUri and database connection classes
- [references/examples.md](references/examples.md) -- Working code examples for all PostGIS operations
- [references/anti-patterns.md](references/anti-patterns.md) -- What NOT to do with PostGIS connections

### Official Sources

- https://docs.qgis.org/3.34/en/docs/pyqgis_developer_cookbook/loadlayer.html
- https://qgis.org/pyqgis/3.34/core/QgsDataSourceUri.html
- https://qgis.org/pyqgis/3.34/core/QgsAbstractDatabaseProviderConnection.html
- https://qgis.org/pyqgis/3.34/core/QgsAuthManager.html
- https://qgis.org/pyqgis/3.34/core/QgsProviderRegistry.html
