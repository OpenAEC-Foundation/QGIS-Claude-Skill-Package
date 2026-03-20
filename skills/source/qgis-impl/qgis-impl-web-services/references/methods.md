# qgis-impl-web-services — Methods Reference

## OGC Client Classes

### QgsRasterLayer (WMS/WMTS/WCS/XYZ)

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| Constructor | `QgsRasterLayer(uri: str, baseName: str, providerKey: str)` | `QgsRasterLayer` | Create raster layer from URI |
| `isValid()` | `() -> bool` | `bool` | Check if layer loaded successfully |
| `renderer()` | `() -> QgsRasterRenderer` | Renderer | Get the current renderer |
| `dataProvider()` | `() -> QgsRasterDataProvider` | Provider | Access data provider |
| `crs()` | `() -> QgsCoordinateReferenceSystem` | CRS | Get layer CRS |
| `extent()` | `() -> QgsRectangle` | Extent | Get layer extent |
| `bandCount()` | `() -> int` | Band count | Number of bands |
| `width()` | `() -> int` | Width | Raster width in pixels |
| `height()` | `() -> int` | Height | Raster height in pixels |

### QgsVectorLayer (WFS)

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| Constructor | `QgsVectorLayer(uri: str, baseName: str, providerKey: str)` | `QgsVectorLayer` | Create vector layer from URI |
| `isValid()` | `() -> bool` | `bool` | Check if layer loaded successfully |
| `featureCount()` | `() -> int` | Count | Number of features |
| `fields()` | `() -> QgsFields` | Fields | Get field definitions |
| `getFeatures()` | `(request?: QgsFeatureRequest) -> QgsFeatureIterator` | Iterator | Iterate features |
| `geometryType()` | `() -> Qgis.GeometryType` | Geometry type | Point/Line/Polygon |
| `crs()` | `() -> QgsCoordinateReferenceSystem` | CRS | Get layer CRS |

### QgsDataSourceUri

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| `setParam()` | `(key: str, value: str)` | `None` | Set a URI parameter |
| `param()` | `(key: str) -> str` | `str` | Get a URI parameter value |
| `removeParam()` | `(key: str)` | `None` | Remove a URI parameter |
| `encodedUri()` | `() -> QByteArray` | `QByteArray` | Get encoded URI bytes |
| `uri()` | `(expandAuthConfig?: bool) -> str` | `str` | Get full URI string |
| `setUsername()` | `(username: str)` | `None` | Set username |
| `setPassword()` | `(password: str)` | `None` | Set password |

---

## Authentication Classes

### QgsAuthMethodConfig

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| `setName()` | `(name: str)` | `None` | Set config display name |
| `setMethod()` | `(method: str)` | `None` | Set auth method (Basic, OAuth2, etc.) |
| `setUri()` | `(uri: str)` | `None` | Set associated URI |
| `setConfig()` | `(key: str, value: str)` | `None` | Set config key-value pair |
| `id()` | `() -> str` | `str` | Get auto-generated auth config ID |

### QgsApplication.authManager() -> QgsAuthManager

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| `storeAuthenticationConfig()` | `(config: QgsAuthMethodConfig) -> bool` | `bool` | Store a new auth config |
| `loadAuthenticationConfig()` | `(authcfg: str, config: QgsAuthMethodConfig) -> bool` | `bool` | Load auth config by ID |
| `removeAuthenticationConfig()` | `(authcfg: str) -> bool` | `bool` | Remove auth config |
| `availableAuthMethodConfigs()` | `() -> dict` | `dict` | Get all stored configs |

---

## QGIS Server Core Classes

### QgsServer

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| Constructor | `QgsServer()` | `QgsServer` | Create server instance |
| `handleRequest()` | `(request: QgsServerRequest, response: QgsServerResponse)` | `None` | Process OGC request |

### QgsBufferServerRequest

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| Constructor | `QgsBufferServerRequest(url: str, method?: QgsServerRequest.Method)` | Request | Create request from URL |
| `url()` | `() -> QUrl` | URL | Get request URL |
| `method()` | `() -> QgsServerRequest.Method` | Method | GET/POST/PUT/DELETE |

### QgsBufferServerResponse

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| `headers()` | `() -> dict` | Headers | Get response headers |
| `body()` | `() -> QByteArray` | Body | Get response body |
| `statusCode()` | `() -> int` | Status | Get HTTP status code |

### QgsRequestHandler

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| `parameterMap()` | `() -> dict` | `dict` | Get all query parameters |
| `setParameter()` | `(key: str, value: str)` | `None` | Inject/modify parameter |
| `parameter()` | `(key: str) -> str` | `str` | Get single parameter |
| `clear()` | `()` | `None` | Clear response |
| `setResponseHeader()` | `(name: str, value: str)` | `None` | Set response header |
| `appendBody()` | `(body: bytes)` | `None` | Append to response body |
| `clearBody()` | `()` | `None` | Clear response body |
| `body()` | `() -> bytes` | `bytes` | Get current response body |
| `exceptionRaised()` | `() -> bool` | `bool` | Check for exceptions |

---

## Server Filter Classes

### QgsServerFilter

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| Constructor | `QgsServerFilter(serverIface: QgsServerInterface)` | Filter | Create filter |
| `serverInterface()` | `() -> QgsServerInterface` | Interface | Get server interface |
| `onRequestReady()` | `() -> bool` | `bool` | Called before core processing |
| `onSendResponse()` | `() -> bool` | `bool` | Called on partial output flush |
| `onResponseComplete()` | `() -> bool` | `bool` | Called after processing complete |

### QgsAccessControlFilter

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| `layerFilterExpression()` | `(layer: QgsVectorLayer) -> str` | Expression | QGIS expression to filter features |
| `layerFilterSubsetString()` | `(layer: QgsVectorLayer) -> str` | SQL | SQL subset string filter |
| `layerPermissions()` | `(layer: QgsMapLayer) -> LayerPermissions` | Permissions | CRUD permission flags |
| `authorizedLayerAttributes()` | `(layer: QgsVectorLayer, attributes: list) -> list` | Filtered list | Remove hidden attributes |
| `allowToEdit()` | `(layer: QgsVectorLayer, feature: QgsFeature) -> bool` | `bool` | Per-feature edit permission |
| `cacheKey()` | `() -> str` | `str` | Cache key for this ACL context |

### QgsAccessControlFilter.LayerPermissions

| Attribute | Type | Default | Purpose |
|-----------|------|---------|---------|
| `canRead` | `bool` | `True` | Layer is visible |
| `canInsert` | `bool` | `True` | Features can be added (WFS-T) |
| `canUpdate` | `bool` | `True` | Features can be modified (WFS-T) |
| `canDelete` | `bool` | `True` | Features can be deleted (WFS-T) |

### QgsServerCacheFilter (QGIS 3.4+)

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| `getCachedDocument()` | `(project, request, key) -> QByteArray` | Cached data | Retrieve cached document |
| `setCachedDocument()` | `(doc, project, request, key) -> bool` | Success | Store document |
| `deleteCachedDocument()` | `(project, request, key) -> bool` | Success | Remove cached document |
| `deleteCachedDocuments()` | `(project) -> bool` | Success | Remove all cached documents |
| `getCachedImage()` | `(project, request, key) -> QByteArray` | Cached data | Retrieve cached image |
| `setCachedImage()` | `(img, project, request, key) -> bool` | Success | Store image |
| `deleteCachedImage()` | `(project, request, key) -> bool` | Success | Remove cached image |
| `deleteCachedImages()` | `(project) -> bool` | Success | Remove all cached images |

---

## OGC API Classes

### QgsServerOgcApi

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| Constructor | `QgsServerOgcApi(serverIface, rootPath, name, description, version)` | API | Create OGC API |
| `registerHandler()` | `(handler: QgsServerOgcApiHandler)` | `None` | Register path handler |

### QgsServerOgcApiHandler (Abstract)

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| `path()` | `() -> QRegularExpression` | Pattern | URL path pattern |
| `operationId()` | `() -> str` | ID | Unique operation ID |
| `summary()` | `() -> str` | Text | Short description |
| `description()` | `() -> str` | Text | Full description |
| `linkTitle()` | `() -> str` | Title | Title for links |
| `linkType()` | `() -> QgsServerOgcApi.Rel` | Rel | Link relation type |
| `handleRequest()` | `(context: QgsServerApiContext)` | `None` | Process request |
| `parameters()` | `(context) -> list` | Param list | Accepted parameters |
| `templatePath()` | `(context) -> str` | Path | HTML template path |
| `setContentTypes()` | `(types: list)` | `None` | Set supported content types |
| `values()` | `(context) -> dict` | Values | Get parsed parameter values |
| `write()` | `(data, context)` | `None` | Write response data |

### QgsServerQueryStringParameter

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| Constructor | `(name, required, type, description)` | Param | Define query parameter |

Types: `QgsServerQueryStringParameter.Type.Double`, `Type.Integer`, `Type.String`

---

## QgsServerInterface

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| `registerFilter()` | `(filter, priority: int)` | `None` | Register I/O filter |
| `registerAccessControl()` | `(filter, priority: int)` | `None` | Register access control |
| `registerServerCache()` | `(filter, priority: int)` | `None` | Register cache filter (3.4+) |
| `requestHandler()` | `() -> QgsRequestHandler` | Handler | Get current request handler |
| `serviceRegistry()` | `() -> QgsServiceRegistry` | Registry | Get service registry |
| `configFilePath()` | `() -> str` | Path | Get config file path |
| `reloadSettings()` | `()` | `None` | Reload server settings (3.28+) |

### QgsService (Abstract)

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| `name()` | `() -> str` | Name | Service name (e.g., "CUSTOM") |
| `version()` | `() -> str` | Version | Service version |
| `executeRequest()` | `(request, response, project)` | `None` | Process service request |
