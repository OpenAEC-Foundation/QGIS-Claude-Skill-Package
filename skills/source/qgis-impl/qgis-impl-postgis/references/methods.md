# API Signatures Reference (PostGIS / PyQGIS)

## QgsDataSourceUri

Constructs and parses data source URIs for database providers.

```python
class QgsDataSourceUri:
    def __init__(self, uri: str = "")

    # Connection parameters
    def setConnection(self, host: str, port: str, database: str,
                      username: str, password: str,
                      sslmode: SslMode = SslPrefer) -> None
    def host(self) -> str
    def port(self) -> str
    def database(self) -> str
    def username(self) -> str
    def password(self) -> str
    def sslMode(self) -> SslMode

    # Data source parameters
    def setDataSource(self, schema: str, table: str,
                      geometryColumn: str,
                      sql: str = "",
                      keyColumn: str = "") -> None
    def schema(self) -> str
    def table(self) -> str
    def geometryColumn(self) -> str
    def sql(self) -> str
    def keyColumn(self) -> str

    # Authentication
    def setAuthConfigId(self, authcfg: str) -> None
    def authConfigId(self) -> str

    # Metadata and performance
    def setUseEstimatedMetadata(self, flag: bool) -> None
    def useEstimatedMetadata(self) -> bool
    def setSrid(self, srid: str) -> None
    def srid(self) -> str
    def setWkbType(self, wkbType: QgsWkbTypes.Type) -> None
    def wkbType(self) -> QgsWkbTypes.Type
    def setKeyColumn(self, column: str) -> None

    # Generic parameters
    def setParam(self, key: str, value: str) -> None
    def param(self, key: str) -> str
    def params(self, key: str) -> list[str]
    def removeParam(self, key: str) -> int
    def hasParam(self, key: str) -> bool

    # URI output
    def uri(self, expandAuthConfig: bool = True) -> str
    def connectionInfo(self, expandAuthConfig: bool = True) -> str
    def quotedTablename(self) -> str

    # SSL modes
    class SslMode:
        SslPrefer = 0
        SslDisable = 1
        SslAllow = 2
        SslRequire = 3
        SslVerifyCa = 4
        SslVerifyFull = 5
```

**Key rule**: ALWAYS call `uri(False)` when passing to layer constructors to prevent credential exposure.

---

## QgsAbstractDatabaseProviderConnection

Base class for database provider connections. Access via `QgsProviderRegistry`.

```python
class QgsAbstractDatabaseProviderConnection:
    # Schema operations
    def schemas(self) -> list[str]
    def createSchema(self, name: str) -> None
    def dropSchema(self, name: str, force: bool = False) -> None

    # Table operations
    def tables(self, schema: str = "",
               flags: TableFlags = TableFlags()) -> list[QgsAbstractDatabaseProviderConnection.TableProperty]
    def table(self, schema: str, name: str) -> TableProperty
    def createVectorTable(self, schema: str, name: str,
                          fields: QgsFields, wkbType: QgsWkbTypes.Type,
                          srs: QgsCoordinateReferenceSystem,
                          overwrite: bool, options: dict = {}) -> None
    def dropVectorTable(self, schema: str, name: str) -> None
    def renameVectorTable(self, schema: str, name: str, newName: str) -> None

    # SQL execution
    def executeSql(self, sql: str, feedback: QgsFeedback = None) -> list[list]
    def execSql(self, sql: str, feedback: QgsFeedback = None) -> QueryResult

    # Vacuum
    def vacuum(self, schema: str, name: str) -> None

    # Spatial index
    def createSpatialIndex(self, schema: str, name: str,
                           options: dict = {}) -> None
    def spatialIndexExists(self, schema: str, name: str,
                           geometryColumn: str) -> bool
    def dropSpatialIndex(self, schema: str, name: str,
                         geometryColumn: str) -> None
```

---

## QgsAbstractDatabaseProviderConnection.TableProperty

Describes a database table.

```python
class TableProperty:
    def tableName(self) -> str
    def schema(self) -> str
    def geometryColumn(self) -> str
    def geometryColumnTypes(self) -> list[TableProperty.GeometryColumnType]
    def primaryKeyColumns(self) -> list[str]
    def geometryColumnCount(self) -> int
    def comment(self) -> str
    def flags(self) -> TableFlags
    def maxCoordinateDimensions(self) -> int
```

---

## QgsProviderRegistry

Singleton registry for data provider access.

```python
class QgsProviderRegistry:
    @staticmethod
    def instance() -> QgsProviderRegistry

    def providerMetadata(self, providerKey: str) -> QgsProviderMetadata
    def providerList(self) -> list[str]
```

---

## QgsProviderMetadata

Metadata and connection factory for a data provider.

```python
class QgsProviderMetadata:
    def createConnection(self, uri_or_name: str,
                         configuration: dict = {}) -> QgsAbstractDatabaseProviderConnection
    def connections(self, cached: bool = True) -> dict[str, QgsAbstractDatabaseProviderConnection]
    def deleteConnection(self, name: str) -> None
    def saveConnection(self, connection: QgsAbstractDatabaseProviderConnection,
                       name: str) -> None
    def encodeUri(self, parts: dict) -> str
    def decodeUri(self, uri: str) -> dict
```

---

## QgsAuthManager

Manages the encrypted authentication database (`qgis-auth.db`).

```python
class QgsAuthManager:
    # Access via QgsApplication.authManager()

    def storeAuthenticationConfig(self, config: QgsAuthMethodConfig) -> tuple[bool, QgsAuthMethodConfig]
    def loadAuthenticationConfig(self, authcfg: str, config: QgsAuthMethodConfig,
                                  full: bool = False) -> bool
    def removeAuthenticationConfig(self, authcfg: str) -> bool
    def configIds(self) -> list[str]
    def availableAuthMethodConfigs(self, dataprovider: str = "") -> dict[str, QgsAuthMethodConfig]
    def updateAuthenticationConfig(self, config: QgsAuthMethodConfig) -> bool
    def authenticationDbPath(self) -> str
    def masterPasswordIsSet(self) -> bool
    def setMasterPassword(self, password: str, verify: bool = True) -> bool
```

---

## QgsAuthMethodConfig

Configuration object for a single authentication entry.

```python
class QgsAuthMethodConfig:
    def __init__(self, method: str = "", version: int = 0)

    def id(self) -> str
    def setId(self, id: str) -> None
    def name(self) -> str
    def setName(self, name: str) -> None
    def method(self) -> str
    def setMethod(self, method: str) -> None
    def config(self, key: str, defaultValue: str = "") -> str
    def setConfig(self, key: str, value: str) -> None
    def configMap(self) -> dict[str, str]
    def isValid(self, validateId: bool = False) -> bool
```

---

## QgsVectorFileWriter (PostGIS Export)

```python
class QgsVectorFileWriter:
    class SaveVectorOptions:
        driverName: str           # "PostgreSQL" for PostGIS
        layerName: str            # target table name
        actionOnExistingFile: int # 0=CreateOrOverwrite, 1=CreateNewFile, 2=AppendToLayer

    @staticmethod
    def writeAsVectorFormatV3(
        layer: QgsVectorLayer,
        fileName: str,           # PG connection string for PostGIS
        transformContext: QgsCoordinateTransformContext,
        options: SaveVectorOptions
    ) -> tuple[WriterError, str, str, str]

    class WriterError:
        NoError = 0
        ErrDriverNotFound = 1
        ErrCreateDataSource = 2
        ErrCreateLayer = 3
        ErrAttributeTypeUnsupported = 4
        ErrAttributeCreationFailed = 5
        ErrProjection = 6
        ErrFeatureWriteFailed = 7
        ErrInvalidLayer = 8
        Canceled = 9
```

---

## Processing Algorithm: native:importintopostgis

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | QgsVectorLayer | Source layer |
| `DATABASE` | str | Stored connection name |
| `SCHEMA` | str | Target schema (default: "public") |
| `TABLENAME` | str | Target table name |
| `PRIMARY_KEY` | str | Primary key column name |
| `GEOMETRY_COLUMN` | str | Geometry column name (default: "geom") |
| `ENCODING` | str | Character encoding (default: "UTF-8") |
| `OVERWRITE` | bool | Overwrite existing table |
| `CREATEINDEX` | bool | Create spatial index |
| `LOWERCASE_NAMES` | bool | Convert column names to lowercase |
| `DROP_STRING_LENGTH` | bool | Remove string length constraints |
| `FORCE_SINGLEPART` | bool | Force single-part geometries |
