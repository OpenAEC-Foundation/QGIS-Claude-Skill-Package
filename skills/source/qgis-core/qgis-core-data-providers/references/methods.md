# qgis-core-data-providers — Method Reference

## QgsVectorLayer Constructor

```python
QgsVectorLayer(
    path: str = "",           # Data source URI (format depends on provider)
    baseName: str = "",       # Display name in layer tree
    providerLib: str = "",    # Provider key: "ogr", "postgres", "memory", etc.
    options: QgsVectorLayer.LayerOptions = QgsVectorLayer.LayerOptions()
)
```

Returns a `QgsVectorLayer` instance. ALWAYS check `isValid()` after construction.

### LayerOptions

```python
options = QgsVectorLayer.LayerOptions()
options.loadDefaultStyle = False    # Skip loading default .qml style
options.readExtentFromXml = True    # Read extent from project XML instead of scanning data
```

---

## QgsRasterLayer Constructor

```python
QgsRasterLayer(
    uri: str = "",            # Data source URI
    baseName: str = "",       # Display name in layer tree
    providerType: str = "gdal"  # Provider key: "gdal", "wms", "wcs", "postgresraster"
)
```

Returns a `QgsRasterLayer` instance. ALWAYS check `isValid()` after construction.

**Note**: When `providerType` is omitted, it defaults to `"gdal"`. For WMS, WMTS, XYZ, or WCS layers, you MUST specify the provider explicitly.

---

## QgsPointCloudLayer Constructor (QGIS 3.18+)

```python
QgsPointCloudLayer(
    uri: str,                 # Path to LAS/LAZ/COPC/EPT file
    baseName: str = "",       # Display name
    providerLib: str = "pdal" # Provider key: "pdal", "copc", "ept"
)
```

---

## QgsMeshLayer Constructor

```python
QgsMeshLayer(
    path: str,                # Path to mesh file (NetCDF, GRIB, etc.)
    baseName: str = "",       # Display name
    providerLib: str = "mdal" # Provider key: "mdal"
)
```

---

## QgsVectorTileLayer Constructor

```python
QgsVectorTileLayer(
    uri: str,                 # URI string with type and url parameters
    baseName: str = ""        # Display name
)
```

URI format: `type=xyz&url=https://example.com/{z}/{x}/{y}.pbf&zmin=0&zmax=14`

---

## QgsDataSourceUri

### Constructor

```python
uri = QgsDataSourceUri()
# OR from existing URI string:
uri = QgsDataSourceUri(existing_uri_string)
```

### Connection Methods

```python
uri.setConnection(
    aHost: str,       # Hostname or IP
    aPort: str,       # Port number as string
    aDatabase: str,   # Database name
    aUsername: str,    # Username
    aPassword: str    # Password
)

# With SSL mode
uri.setConnection(
    aHost: str,
    aPort: str,
    aDatabase: str,
    aUsername: str,
    aPassword: str,
    sslmode: QgsDataSourceUri.SslMode  # SslDisable, SslAllow, SslPrefer, SslRequire, SslVerifyCa, SslVerifyFull
)
```

### Data Source Methods

```python
uri.setDataSource(
    aSchema: str,          # Schema name (e.g., "public")
    aTable: str,           # Table or view name
    aGeometryColumn: str,  # Geometry column name
    aSql: str = "",        # Optional SQL WHERE filter
    aKeyColumn: str = ""   # Optional primary key column
)
```

### URI Generation

```python
uri_string = uri.uri(expandAuthConfig: bool)
# expandAuthConfig=False → ALWAYS use this for logging/display (hides credentials)
# expandAuthConfig=True  → NEVER use for logging (exposes credentials)
```

### Authentication Manager Integration

```python
uri.setAuthConfigId(authcfg: str)  # Use QGIS auth manager configuration ID
```

### Getter Methods

```python
uri.host()           # str
uri.port()           # str
uri.database()       # str
uri.username()       # str
uri.password()       # str
uri.schema()         # str
uri.table()          # str
uri.geometryColumn() # str
uri.keyColumn()      # str
uri.sql()            # str
uri.sslMode()        # QgsDataSourceUri.SslMode
```

### SSL Mode Enum

```python
QgsDataSourceUri.SslPrefer       # Default
QgsDataSourceUri.SslDisable
QgsDataSourceUri.SslAllow
QgsDataSourceUri.SslRequire
QgsDataSourceUri.SslVerifyCa
QgsDataSourceUri.SslVerifyFull
```

---

## QgsProviderRegistry

### Key Methods

```python
registry = QgsProviderRegistry.instance()

registry.providerList()                    # list[str] — all registered provider keys
registry.providerMetadata(providerKey)     # QgsProviderMetadata or None
registry.library(providerKey)              # str — path to provider library
registry.pluginList()                      # str — formatted list of all providers
```

### QgsProviderMetadata

```python
meta = registry.providerMetadata("ogr")
meta.key()                  # str — provider key
meta.description()          # str — human-readable description
meta.supportedLayerTypes()  # list — supported QgsMapLayerType values
```

---

## QgsDataProvider

### Sublayer Discovery

```python
provider = layer.dataProvider()

# Get sublayer list (for multi-layer sources like GeoPackage)
sublayers = provider.subLayers()  # list[str]

# Sublayer string format: "{index}:{name}:{feature_count}:{geometry_type}:{srid}"
# Split with QgsDataProvider.SUBLAYER_SEPARATOR
```

### Constants

```python
QgsDataProvider.SUBLAYER_SEPARATOR  # str — separator character for sublayer strings
```

---

## Layer Validity

### isValid()

```python
layer.isValid()  # bool — True if layer loaded successfully
```

A layer is invalid when:
- File path does not exist or is inaccessible
- URI format is malformed for the provider
- Provider cannot parse the data format
- Authentication credentials are missing or incorrect
- CRS database is unavailable (standalone scripts without `setPrefixPath`)
- Required driver is not installed (e.g., ECW, MrSID)

### Error Reporting

```python
layer.error().summary()    # str — human-readable error summary (QGIS 3.x)
layer.dataProvider()       # None if provider failed to initialize
```
