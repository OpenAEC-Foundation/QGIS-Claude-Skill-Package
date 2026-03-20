# qgis-errors-data-loading — Methods Reference

## Layer Construction

### QgsVectorLayer

```python
QgsVectorLayer(path: str, baseName: str = "", providerLib: str = "", options: QgsVectorLayer.LayerOptions = QgsVectorLayer.LayerOptions()) -> QgsVectorLayer
```

- `path` — Data source URI (format depends on provider)
- `baseName` — Display name in the layer tree
- `providerLib` — Provider identifier: `"ogr"`, `"postgres"`, `"memory"`, `"WFS"`, `"delimitedtext"`, `"spatialite"`, `"gpx"`, `"virtual"`
- Returns a layer object that MUST be checked with `isValid()`

### QgsRasterLayer

```python
QgsRasterLayer(path: str = "", baseName: str = "", providerType: str = "", options: QgsRasterLayer.LayerOptions = QgsRasterLayer.LayerOptions()) -> QgsRasterLayer
```

- `providerType` — Provider identifier: `"gdal"`, `"wms"`, `"wcs"`, `"postgresraster"`
- Returns a layer object that MUST be checked with `isValid()`

---

## Layer Validity

### QgsMapLayer.isValid()

```python
layer.isValid() -> bool
```

Returns `True` if the layer was loaded successfully. ALWAYS call immediately after construction.

### QgsMapLayer.error()

```python
layer.error() -> QgsError
```

Returns the error object for the layer. Use `error().message()` to get a human-readable string.

---

## Data Provider Error Inspection

### QgsDataProvider.error()

```python
layer.dataProvider().error() -> QgsError
```

Returns detailed error information from the data provider. More specific than `layer.error()`.

### QgsError.message()

```python
error.message() -> str
```

Returns the error message as a formatted string including all error components.

### QgsError.isEmpty()

```python
error.isEmpty() -> bool
```

Returns `True` if no error was recorded.

---

## QgsDataSourceUri (Database URI Builder)

### Connection Methods

```python
uri = QgsDataSourceUri()

uri.setConnection(host: str, port: str, database: str, username: str, password: str) -> None
uri.setConnection(host: str, port: str, database: str, username: str, password: str, sslmode: QgsDataSourceUri.SslMode) -> None
```

### Data Source Methods

```python
uri.setDataSource(schema: str, table: str, geometryColumn: str, sql: str = "", keyColumn: str = "") -> None
```

- `schema` — Database schema (e.g., `"public"`)
- `table` — Table or view name
- `geometryColumn` — Name of the geometry column
- `sql` — Optional WHERE clause filter
- `keyColumn` — Primary key column (ALWAYS specify for views)

### Authentication

```python
uri.setAuthConfigId(authcfg: str) -> None
```

Uses a stored authentication configuration from `QgsAuthManager`. ALWAYS prefer this over plain passwords.

### URI Generation

```python
uri.uri(expandAuthConfig: bool = True) -> str
```

- Pass `False` to prevent credential expansion in the output string
- ALWAYS use `uri.uri(False)` when logging, displaying, or passing to layer constructors

---

## Feature Request (Performance Optimization)

### QgsFeatureRequest

```python
request = QgsFeatureRequest()

# Spatial filter
request.setFilterRect(rect: QgsRectangle) -> QgsFeatureRequest

# Attribute filter
request.setFilterExpression(expression: str) -> QgsFeatureRequest

# Feature ID filter
request.setFilterFids(fids: set) -> QgsFeatureRequest

# Limit returned attributes
request.setSubsetOfAttributes(attrs: list, fields: QgsFields) -> QgsFeatureRequest

# Skip geometry loading
request.setFlags(QgsFeatureRequest.NoGeometry) -> QgsFeatureRequest

# Limit result count
request.setLimit(limit: int) -> QgsFeatureRequest
```

---

## CRS Methods

### QgsMapLayer.crs()

```python
layer.crs() -> QgsCoordinateReferenceSystem
```

Returns the layer's coordinate reference system.

### QgsMapLayer.setCrs()

```python
layer.setCrs(crs: QgsCoordinateReferenceSystem) -> None
```

Assigns CRS metadata to the layer. Does NOT reproject coordinates.

### QgsCoordinateReferenceSystem.isValid()

```python
crs.isValid() -> bool
```

Returns `True` if the CRS is a recognized, valid coordinate reference system.

### QgsCoordinateReferenceSystem.authid()

```python
crs.authid() -> str
```

Returns the authority identifier (e.g., `"EPSG:4326"`).

---

## Encoding Methods

### QgsVectorDataProvider.setEncoding()

```python
layer.dataProvider().setEncoding(encoding: str) -> None
```

Sets the character encoding for reading attribute data. Common values: `"UTF-8"`, `"ISO-8859-1"`, `"Windows-1252"`.

---

## Sublayer Inspection

### QgsDataProvider.subLayers()

```python
layer.dataProvider().subLayers() -> list[str]
```

Returns a list of sublayer descriptions for multi-layer data sources (e.g., GeoPackage, GDB). Each string contains fields separated by `QgsDataProvider.SUBLAYER_SEPARATOR`.

### QgsDataProvider.SUBLAYER_SEPARATOR

```python
QgsDataProvider.SUBLAYER_SEPARATOR  # Constant string used to split sublayer info
```
