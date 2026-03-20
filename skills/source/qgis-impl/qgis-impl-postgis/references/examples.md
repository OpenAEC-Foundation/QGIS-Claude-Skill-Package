# Working Code Examples (PostGIS / PyQGIS)

## Example 1: Basic Table Connection with Auth Config

```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer, QgsProject

uri = QgsDataSourceUri()
uri.setConnection("db.example.com", "5432", "city_gis", "", "")
uri.setAuthConfigId("pg_prod_auth")
uri.setDataSource("public", "buildings", "geom")

layer = QgsVectorLayer(uri.uri(False), "Buildings", "postgres")
assert layer.isValid(), f"Failed to load: {layer.error().message()}"

QgsProject.instance().addMapLayer(layer)
print(f"Loaded {layer.featureCount()} buildings")
```

---

## Example 2: Load a Database View

```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer

uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "analytics_db", "", "")
uri.setAuthConfigId("pg_local_auth")

# Views REQUIRE a unique key column -- without it, behavior is undefined
uri.setDataSource("reporting", "monthly_sales_view", "location", "", "report_id")

layer = QgsVectorLayer(uri.uri(False), "Monthly Sales", "postgres")
assert layer.isValid()
```

---

## Example 3: SQL Subquery as Layer

```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer

uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "transport_db", "", "")
uri.setAuthConfigId("pg_transport")

# Wrap SQL in parentheses, use empty string for schema
sql_query = """(
    SELECT r.gid, r.road_name, r.speed_limit, r.geom
    FROM roads r
    JOIN road_conditions rc ON r.gid = rc.road_id
    WHERE rc.condition = 'poor'
      AND r.speed_limit > 80
)"""

uri.setDataSource("", sql_query, "geom", "", "gid")

layer = QgsVectorLayer(uri.uri(False), "Poor Condition Highways", "postgres")
assert layer.isValid()
print(f"Found {layer.featureCount()} roads needing repair")
```

---

## Example 4: Schema Discovery and Table Listing

```python
from qgis.core import QgsProviderRegistry

md = QgsProviderRegistry.instance().providerMetadata("postgres")
conn = md.createConnection("Production PostGIS")

# List all schemas
for schema in conn.schemas():
    print(f"\nSchema: {schema}")

    # List tables with geometry in each schema
    for table in conn.tables(schema):
        geom_col = table.geometryColumn()
        if geom_col:
            pk_cols = ", ".join(table.primaryKeyColumns())
            print(f"  {table.tableName()} "
                  f"(geom: {geom_col}, pk: {pk_cols})")
```

---

## Example 5: Execute SQL and Process Results

```python
from qgis.core import QgsProviderRegistry

md = QgsProviderRegistry.instance().providerMetadata("postgres")
conn = md.createConnection("My PostGIS Server")

# Simple aggregation query
results = conn.executeSql("""
    SELECT road_type, COUNT(*), ROUND(SUM(ST_Length(geom))::numeric, 2)
    FROM public.roads
    GROUP BY road_type
    ORDER BY COUNT(*) DESC
""")

for row in results:
    road_type, count, total_length = row
    print(f"{road_type}: {count} segments, {total_length}m total")
```

---

## Example 6: Create Auth Config Programmatically

```python
from qgis.core import QgsApplication, QgsAuthMethodConfig

auth_mgr = QgsApplication.authManager()

# Build the auth config
config = QgsAuthMethodConfig()
config.setName("Production PostGIS")
config.setMethod("Basic")
config.setConfig("username", "gis_reader")
config.setConfig("password", "secure_password_here")

# Store -- the system assigns a unique ID
success, config = auth_mgr.storeAuthenticationConfig(config)
assert success, "Failed to store authentication config"

auth_id = config.id()
print(f"Auth config stored with ID: {auth_id}")

# Now use it in connections
from qgis.core import QgsDataSourceUri, QgsVectorLayer

uri = QgsDataSourceUri()
uri.setConnection("db.example.com", "5432", "production_gis", "", "")
uri.setAuthConfigId(auth_id)
uri.setDataSource("public", "parcels", "geom")

layer = QgsVectorLayer(uri.uri(False), "Parcels", "postgres")
assert layer.isValid()
```

---

## Example 7: Export Layer to PostGIS via Processing

```python
import processing
from qgis.core import QgsProject

# Get source layer from current project
source_layer = QgsProject.instance().mapLayersByName("Survey Points")[0]

result = processing.run("native:importintopostgis", {
    'INPUT': source_layer,
    'DATABASE': 'My PostGIS Server',
    'SCHEMA': 'survey_data',
    'TABLENAME': 'points_2024',
    'PRIMARY_KEY': 'id',
    'GEOMETRY_COLUMN': 'geom',
    'ENCODING': 'UTF-8',
    'OVERWRITE': True,
    'CREATEINDEX': True,
    'LOWERCASE_NAMES': True,
    'DROP_STRING_LENGTH': False,
    'FORCE_SINGLEPART': False
})

print("Export completed successfully")
```

---

## Example 8: Export Layer with QgsVectorFileWriter

```python
from qgis.core import (
    QgsVectorFileWriter, QgsCoordinateTransformContext, QgsProject
)

layer = QgsProject.instance().mapLayersByName("Roads")[0]

save_options = QgsVectorFileWriter.SaveVectorOptions()
save_options.driverName = "PostgreSQL"
save_options.layerName = "roads_backup"

# Use authcfg in the PG connection string
pg_uri = "PG:host=localhost port=5432 dbname=backup_db authcfg=pg_backup_auth"

error_code, error_msg, new_filename, new_layer = (
    QgsVectorFileWriter.writeAsVectorFormatV3(
        layer,
        pg_uri,
        QgsCoordinateTransformContext(),
        save_options
    )
)

if error_code != QgsVectorFileWriter.NoError:
    raise RuntimeError(f"Export failed ({error_code}): {error_msg}")

print(f"Exported to {new_filename}")
```

---

## Example 9: Load PostGIS Raster

```python
from qgis.core import (
    QgsProviderRegistry, QgsDataSourceUri, QgsRasterLayer, QgsProject
)

uri_config = {
    'dbname': 'elevation_db',
    'host': 'localhost',
    'port': '5432',
    'sslmode': QgsDataSourceUri.SslDisable,
    'authcfg': 'pg_raster_auth',
    'schema': 'terrain',
    'table': 'dem_tiles',
    'geometrycolumn': 'rast',
    'mode': '2'  # union all tiles into one layer
}

md = QgsProviderRegistry.instance().providerMetadata('postgresraster')
uri = QgsDataSourceUri(md.encodeUri(uri_config))

rlayer = QgsRasterLayer(uri.uri(False), "DEM", "postgresraster")
assert rlayer.isValid(), f"Raster load failed: {rlayer.error().message()}"

QgsProject.instance().addMapLayer(rlayer)
print(f"Raster size: {rlayer.width()}x{rlayer.height()}, bands: {rlayer.bandCount()}")
```

---

## Example 10: Full Workflow -- Connect, Query, Load, Analyze

```python
from qgis.core import (
    QgsProviderRegistry, QgsDataSourceUri, QgsVectorLayer,
    QgsProject, QgsFeatureRequest
)
import processing

# Step 1: Connect and discover
md = QgsProviderRegistry.instance().providerMetadata("postgres")
conn = md.createConnection("City GIS Database")

# Step 2: Check what tables exist
tables = conn.tables("infrastructure")
spatial_tables = [t for t in tables if t.geometryColumn()]
print(f"Found {len(spatial_tables)} spatial tables in 'infrastructure' schema")

# Step 3: Load a specific table
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "city_gis", "", "")
uri.setAuthConfigId("city_gis_auth")
uri.setDataSource("infrastructure", "water_pipes", "geom")
uri.setUseEstimatedMetadata(True)

pipes = QgsVectorLayer(uri.uri(False), "Water Pipes", "postgres")
assert pipes.isValid()
QgsProject.instance().addMapLayer(pipes)

# Step 4: Run a spatial analysis
result = processing.run("native:buffer", {
    'INPUT': pipes,
    'DISTANCE': 50,
    'SEGMENTS': 16,
    'DISSOLVE': True,
    'OUTPUT': 'memory:'
})

buffer_layer = result['OUTPUT']
QgsProject.instance().addMapLayer(buffer_layer)

# Step 5: Export result back to PostGIS
processing.run("native:importintopostgis", {
    'INPUT': buffer_layer,
    'DATABASE': 'City GIS Database',
    'SCHEMA': 'analysis_results',
    'TABLENAME': 'pipe_buffer_50m',
    'PRIMARY_KEY': 'id',
    'GEOMETRY_COLUMN': 'geom',
    'OVERWRITE': True,
    'CREATEINDEX': True,
    'LOWERCASE_NAMES': True
})

print("Analysis complete -- buffer zones exported to PostGIS")
```

---

## Example 11: List and Use Stored Connections

```python
from qgis.core import QgsProviderRegistry

md = QgsProviderRegistry.instance().providerMetadata("postgres")

# List all stored PostgreSQL connections
connections = md.connections()
for name, conn in connections.items():
    print(f"Connection: {name}")
    try:
        version = conn.executeSql("SELECT PostGIS_Version()")
        print(f"  PostGIS: {version[0][0]}")
    except Exception as e:
        print(f"  Error: {e}")
```

---

## Example 12: Create Spatial Index and Vacuum

```python
from qgis.core import QgsProviderRegistry

md = QgsProviderRegistry.instance().providerMetadata("postgres")
conn = md.createConnection("My PostGIS Server")

# Create spatial index if missing
if not conn.spatialIndexExists("public", "large_dataset", "geom"):
    conn.createSpatialIndex("public", "large_dataset")
    print("Spatial index created")

# Vacuum to reclaim space and update statistics
conn.vacuum("public", "large_dataset")
print("Vacuum complete")
```
