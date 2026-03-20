# QGIS Server Python API — Targeted Research

> Supplementary research for the QGIS Server skill. Covers server plugin/filter development,
> OGC service architecture, and deployment patterns in depth.
>
> Date: 2026-03-20

---

## 1. QgsServer: Initialization and Request Lifecycle

[CONFIRMED] Source: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/server.html

### Initialization

QgsServer is the main entry point for QGIS Server. It wraps a QgsApplication instance and provides OGC web services.

```python
from qgis.core import QgsApplication
from qgis.server import QgsServer, QgsBufferServerRequest, QgsBufferServerResponse

app = QgsApplication([], False)
server = QgsServer()
```

### handleRequest() — Complete Request Lifecycle

The `handleRequest(request, response)` method is the core entry point. It accepts a `QgsServerRequest` and a `QgsServerResponse`, processes the request through the filter chain and core services, and populates the response.

```python
request = QgsBufferServerRequest(
    'http://localhost:8081/?MAP=/qgis-server/projects/helloworld.qgs'
    '&SERVICE=WMS&REQUEST=GetCapabilities')
response = QgsBufferServerResponse()

server.handleRequest(request, response)

print(response.headers())
print(response.body().data().decode('utf8'))

app.exitQgis()
```

### Request/Response Classes

[CONFIRMED] Source: https://qgis.org/pyqgis/master/server/index.html

| Class | Purpose |
|-------|---------|
| `QgsServerRequest` | Abstract base for server requests |
| `QgsServerResponse` | Abstract base for server responses |
| `QgsBufferServerRequest` | Concrete request with in-memory data (for testing/standalone use) |
| `QgsBufferServerResponse` | Concrete response with in-memory buffer |
| `QgsFcgiServerRequest` | Request implementation for FCGI deployment |
| `QgsRequestHandler` | Interface hiding I/O details, provides `parameterMap()`, `setParameter()`, `clear()`, `setResponseHeader()`, `appendBody()`, `clearBody()` |

### Thread Safety

[CONFIRMED] QGIS Server classes are NOT thread safe. The documentation explicitly states: "Use a multiprocessing model or containers when building scalable applications." This means each worker process must have its own QgsServer instance.

---

## 2. QgsServerFilter: Complete Filter API

[CONFIRMED] Source: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/server.html

QgsServerFilter is the base class for I/O filters. Filters form a chain — if any callback returns `False`, the chain stops; returning `True` propagates to the next filter.

### Callback Methods

#### onRequestReady() -> bool

Called when the request URL and data have been parsed, BEFORE core services execute. Use cases:
- Authentication and authorization checks
- Parameter manipulation (inject, modify, remove parameters)
- Redirects and early exceptions
- Request logging

#### onResponseComplete() -> bool

Called ONCE after core services have finished processing. The entire response body is available. Use cases:
- Implementing new/custom services (WPS, custom endpoints)
- Direct manipulation of output (e.g., adding watermarks to WMS images)
- Response logging and auditing
- Response transformation

#### onSendResponse() -> bool

Called when output is being flushed in chunks (streaming). Returning `False` prevents the current chunk from being sent to the client. Use cases:
- Streaming response modification
- Bandwidth throttling
- Progressive response filtering

### Accessing Request Data Within Filters

```python
request = self.serverInterface().requestHandler()
params = request.parameterMap()           # Dict of all query parameters
request.setParameter('KEY', 'value')      # Inject/modify parameters
request.clear()                           # Clear response
request.setResponseHeader('Content-type', 'text/plain')
request.appendBody(b'Response data')      # Set response body
request.clearBody()                       # Clear response body
request.body()                            # Get current response body as bytes
request.exceptionRaised()                 # Check if an exception occurred
```

### Complete Filter Example: Custom "HELLO" Service

```python
from qgis.server import QgsServerFilter
from qgis.core import QgsMessageLog

class HelloFilter(QgsServerFilter):
    def __init__(self, serverIface):
        super().__init__(serverIface)

    def onRequestReady(self) -> bool:
        QgsMessageLog.logMessage("HelloFilter.onRequestReady")
        return True

    def onSendResponse(self) -> bool:
        QgsMessageLog.logMessage("HelloFilter.onSendResponse")
        return True

    def onResponseComplete(self) -> bool:
        QgsMessageLog.logMessage("HelloFilter.onResponseComplete")
        request = self.serverInterface().requestHandler()
        params = request.parameterMap()
        if params.get('SERVICE', '').upper() == 'HELLO':
            request.clear()
            request.setResponseHeader('Content-type', 'text/plain')
            request.appendBody(b'HelloServer!')
        return True
```

### Watermark Filter Example (Image Manipulation)

[CONFIRMED] Source: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/server.html

```python
from qgis.server import QgsServerFilter
from qgis.core import QgsMessageLog
from qgis.PyQt.QtCore import QByteArray, QBuffer, QIODevice
from qgis.PyQt.QtGui import QImage, QPainter, QRect
import os

class WatermarkFilter(QgsServerFilter):
    def __init__(self, serverIface):
        super().__init__(serverIface)

    def onResponseComplete(self) -> bool:
        request = self.serverInterface().requestHandler()
        params = request.parameterMap()
        if (params.get('SERVICE', '').upper() == 'WMS'
                and params.get('REQUEST', '').upper() == 'GETMAP'
                and not request.exceptionRaised()):
            img = QImage()
            img.loadFromData(request.body())
            watermark = QImage(os.path.join(
                os.path.dirname(__file__), 'media/watermark.png'))
            p = QPainter(img)
            p.drawImage(QRect(20, 20, 40, 40), watermark)
            p.end()
            ba = QByteArray()
            buffer = QBuffer(ba)
            buffer.open(QIODevice.WriteOnly)
            img.save(buffer, "PNG" if "png" in
                request.parameter("FORMAT") else "JPG")
            request.clearBody()
            request.appendBody(ba)
        return True
```

### Parameter Injection Example

```python
class ParamsFilter(QgsServerFilter):
    def __init__(self, serverIface):
        super().__init__(serverIface)

    def onRequestReady(self) -> bool:
        request = self.serverInterface().requestHandler()
        request.setParameter('TEST_NEW_PARAM', 'ParamsFilter')
        return True

    def onResponseComplete(self) -> bool:
        request = self.serverInterface().requestHandler()
        params = request.parameterMap()
        if params.get('TEST_NEW_PARAM') == 'ParamsFilter':
            QgsMessageLog.logMessage("Parameter injection SUCCESS")
        return True
```

---

## 3. QgsAccessControlFilter: Layer/Feature/Attribute Access Control

[CONFIRMED] Source: https://qgis.org/pyqgis/master/server/QgsAccessControlFilter.html

QgsAccessControlFilter provides fine-grained access control for QGIS Server. It allows plugins to restrict which layers, features, and attributes are visible or editable per request.

### Complete Method Reference

| Method | Signature | Return | Used In |
|--------|-----------|--------|---------|
| `layerFilterExpression` | `(layer: QgsVectorLayer) -> str` | QGIS expression string | WMS/GetMap, WMS/GetFeatureInfo, WFS/GetFeature |
| `layerFilterSubsetString` | `(layer: QgsVectorLayer) -> str` | SQL subset string | WMS/GetMap, WMS/GetFeatureInfo, WFS/GetFeature |
| `layerPermissions` | `(layer: QgsMapLayer) -> LayerPermissions` | Permission flags | All OGC services |
| `authorizedLayerAttributes` | `(layer: QgsVectorLayer, attributes: List[str]) -> List[str]` | Filtered attribute list | WMS/GetFeatureInfo, WFS/GetFeature |
| `allowToEdit` | `(layer: QgsVectorLayer, feature: QgsFeature) -> bool` | True/False | WFS-T |
| `cacheKey` | `() -> str` | Cache key string | Capabilities cache |

### LayerPermissions Inner Class

```python
rights = QgsAccessControlFilter.LayerPermissions()
rights.canRead = True
rights.canInsert = False
rights.canUpdate = False
rights.canDelete = False
```

### Complete Access Control Example

```python
from qgis.server import QgsAccessControlFilter

class RoleBasedAccessControl(QgsAccessControlFilter):
    def __init__(self, server_iface):
        super().__init__(server_iface)

    def layerFilterExpression(self, layer):
        """Restrict features visible to users based on a 'role' attribute."""
        return "\"role\" = 'public'"

    def layerFilterSubsetString(self, layer):
        """SQL-level filter for data providers that support it."""
        return "status = 'published'"

    def layerPermissions(self, layer):
        """Control CRUD permissions per layer."""
        rights = QgsAccessControlFilter.LayerPermissions()
        rights.canRead = True
        rights.canInsert = False
        rights.canUpdate = False
        rights.canDelete = False
        return rights

    def authorizedLayerAttributes(self, layer, attributes):
        """Hide sensitive attributes from responses."""
        hidden = {'password', 'internal_id', 'role'}
        return [a for a in attributes if a not in hidden]

    def allowToEdit(self, layer, feature):
        """Only allow editing features owned by the current user."""
        return feature.attribute('owner') == 'current_user'

    def cacheKey(self):
        """Return role-specific cache key. Empty string disables caching."""
        return "role_public"
```

### Registration

```python
class AccessControlPlugin:
    def __init__(self, serverIface):
        serverIface.registerAccessControl(
            RoleBasedAccessControl(serverIface), 100)
```

---

## 4. QgsServerCacheFilter: Caching Implementation

[CONFIRMED] Source: https://qgis.org/pyqgis/master/server/QgsServerCacheFilter.html

Added in QGIS 3.4. Provides a cache interface for server plugins, supporting both document caching (capabilities XML, etc.) and image caching (tiles, GetMap responses).

### Complete Method Reference

| Method | Signature | Return | Purpose |
|--------|-----------|--------|---------|
| `getCachedDocument` | `(project: QgsProject, request: QgsServerRequest, key: str) -> QByteArray` | Cached data or empty | Retrieve cached document |
| `setCachedDocument` | `(doc: QByteArray, project: QgsProject, request: QgsServerRequest, key: str) -> bool` | Success flag | Store document in cache |
| `deleteCachedDocument` | `(project: QgsProject, request: QgsServerRequest, key: str) -> bool` | Success flag | Remove specific cached document |
| `deleteCachedDocuments` | `(project: QgsProject) -> bool` | Success flag | Remove all cached documents for project |
| `getCachedImage` | `(project: QgsProject, request: QgsServerRequest, key: str) -> QByteArray` | Cached data or empty | Retrieve cached image/tile |
| `setCachedImage` | `(img: QByteArray, project: QgsProject, request: QgsServerRequest, key: str) -> bool` | Success flag | Store image in cache |
| `deleteCachedImage` | `(project: QgsProject, request: QgsServerRequest, key: str) -> bool` | Success flag | Remove specific cached image |
| `deleteCachedImages` | `(project: QgsProject) -> bool` | Success flag | Remove all cached images for project |

### Cache Filter Example

[UNCONFIRMED] No official example exists in the cookbook. The following is derived from the API signature patterns:

```python
from qgis.server import QgsServerCacheFilter
from qgis.core import QgsMessageLog
from qgis.PyQt.QtCore import QByteArray
import hashlib
import os
import json

class FilesystemCacheFilter(QgsServerCacheFilter):
    def __init__(self, server_iface, cache_dir='/tmp/qgis_cache'):
        super().__init__(server_iface)
        self.cache_dir = cache_dir
        os.makedirs(cache_dir, exist_ok=True)

    def _cache_path(self, key):
        safe_key = hashlib.md5(key.encode()).hexdigest()
        return os.path.join(self.cache_dir, safe_key)

    def getCachedDocument(self, project, request, key):
        path = self._cache_path(key)
        if os.path.exists(path):
            with open(path, 'rb') as f:
                return QByteArray(f.read())
        return QByteArray()

    def setCachedDocument(self, doc, project, request, key):
        path = self._cache_path(key)
        with open(path, 'wb') as f:
            f.write(bytes(doc))
        return True

    def deleteCachedDocument(self, project, request, key):
        path = self._cache_path(key)
        if os.path.exists(path):
            os.remove(path)
            return True
        return False

    def deleteCachedDocuments(self, project):
        # Remove all files in cache directory
        for f in os.listdir(self.cache_dir):
            os.remove(os.path.join(self.cache_dir, f))
        return True

    # Image caching follows the same pattern
    getCachedImage = getCachedDocument
    setCachedImage = setCachedDocument
    deleteCachedImage = deleteCachedDocument
    deleteCachedImages = deleteCachedDocuments
```

### Registration

```python
class CachePlugin:
    def __init__(self, serverIface):
        serverIface.registerServerCache(
            FilesystemCacheFilter(serverIface), 100)
```

---

## 5. QgsServerOgcApi and QgsServerOgcApiHandler: OGC API Features

[CONFIRMED] Source: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/server.html

These classes allow creating custom OGC-compliant API endpoints that serve JSON and HTML responses.

### QgsServerOgcApi

Constructor: `QgsServerOgcApi(serverIface, rootPath, name, description, version)`

Key methods:
- `registerHandler(handler)` — Register a handler for a path pattern
- Registration with service registry: `serverIface.serviceRegistry().registerApi(api)`

### QgsServerOgcApiHandler

Abstract base class for handling OGC API requests. Subclasses MUST implement:

| Method | Return | Purpose |
|--------|--------|---------|
| `path()` | `QRegularExpression` | URL path pattern to match |
| `operationId()` | `str` | Unique operation identifier |
| `summary()` | `str` | Short description |
| `description()` | `str` | Full description |
| `linkTitle()` | `str` | Title for links |
| `linkType()` | `QgsServerOgcApi.Rel` | Link relation type (`data`, `self`, etc.) |
| `handleRequest(context)` | `None` | Process the request |
| `parameters(context)` | `List[QgsServerQueryStringParameter]` | Define accepted parameters |
| `templatePath(context)` | `str` | Path to HTML template (for HTML content type) |

### Content Types

Set via `setContentTypes()` in the constructor:
- `QgsServerOgcApi.HTML`
- `QgsServerOgcApi.JSON`

### QgsServerQueryStringParameter

Defines a query parameter: `QgsServerQueryStringParameter(name, required, type, description)`

Types: `QgsServerQueryStringParameter.Type.Double`, `QgsServerQueryStringParameter.Type.Integer`, `QgsServerQueryStringParameter.Type.String`

### Complete OGC API Example

```python
import json
import os
from qgis.PyQt.QtCore import QRegularExpression
from qgis.server import (
    QgsServerOgcApi,
    QgsServerQueryStringParameter,
    QgsServerOgcApiHandler,
)
from qgis.core import (
    QgsJsonExporter, QgsCircle, QgsFeature,
    QgsPoint, QgsGeometry,
)

class CircleApiHandler(QgsServerOgcApiHandler):
    def __init__(self):
        super().__init__()
        self.setContentTypes([QgsServerOgcApi.HTML, QgsServerOgcApi.JSON])

    def path(self):
        return QRegularExpression("/customapi")

    def operationId(self):
        return "CustomApiXYCircle"

    def summary(self):
        return "Creates a circle around a point"

    def description(self):
        return "Creates a circle around a point"

    def linkTitle(self):
        return "Custom Api XY Circle"

    def linkType(self):
        return QgsServerOgcApi.data

    def handleRequest(self, context):
        values = self.values(context)
        x = values['x']
        y = values['y']
        r = values['r']
        f = QgsFeature()
        f.setAttributes([x, y, r])
        f.setGeometry(QgsCircle(QgsPoint(x, y), r).toCircularString())
        exporter = QgsJsonExporter()
        self.write(json.loads(exporter.exportFeature(f)), context)

    def templatePath(self, context):
        return os.path.join(os.path.dirname(__file__), 'circle.html')

    def parameters(self, context):
        return [
            QgsServerQueryStringParameter(
                'x', True,
                QgsServerQueryStringParameter.Type.Double,
                'X coordinate'),
            QgsServerQueryStringParameter(
                'y', True,
                QgsServerQueryStringParameter.Type.Double,
                'Y coordinate'),
            QgsServerQueryStringParameter(
                'r', True,
                QgsServerQueryStringParameter.Type.Double,
                'Radius'),
        ]


class CircleApiPlugin:
    def __init__(self, serverIface):
        api = QgsServerOgcApi(
            serverIface, '/customapi',
            'Circle API', 'Creates circles around points', '1.0')
        handler = CircleApiHandler()
        api.registerHandler(handler)
        serverIface.serviceRegistry().registerApi(api)
```

---

## 6. QgsService: Custom OGC Services

[CONFIRMED] Source: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/server.html

For implementing full custom OGC-style services (as opposed to OGC API endpoints).

```python
from qgis.server import QgsService
from qgis.core import QgsMessageLog

class CustomOgcService(QgsService):
    def __init__(self):
        QgsService.__init__(self)

    def name(self):
        return "CUSTOM"

    def version(self):
        return "1.0.0"

    def executeRequest(self, request, response, project):
        response.setStatusCode(200)
        response.write("Custom service response")

class CustomServicePlugin:
    def __init__(self, serverIface):
        serverIface.serviceRegistry().registerService(CustomOgcService())
```

---

## 7. Server Plugin Structure

[CONFIRMED] Source: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/server.html

### Directory Layout

```
$QGIS_PLUGINPATH/
  MyServerPlugin/
    __init__.py        # MUST contain serverClassFactory()
    MyServerPlugin.py  # Plugin implementation
    metadata.txt       # MUST contain server=True
```

### metadata.txt — Required Fields

```ini
[general]
name=MyServerPlugin
description=A QGIS Server plugin
version=1.0.0
qgisMinimumVersion=3.0
server=True
```

The `server=True` line is CRITICAL — without it, QGIS Server will not load the plugin.

### __init__.py — Entry Point

```python
def serverClassFactory(serverIface):
    """Called by QGIS Server at startup. MUST return plugin instance."""
    from .MyServerPlugin import MyServerPluginServer
    return MyServerPluginServer(serverIface)
```

Note: Server plugins use `serverClassFactory()`, NOT `classFactory()` (which is for desktop plugins).

### Plugin Class — Filter Registration

```python
class MyServerPluginServer:
    def __init__(self, serverIface):
        # Register filters with priority (lower = invoked first)
        serverIface.registerFilter(MyFilter(serverIface), 100)
        serverIface.registerAccessControl(MyACL(serverIface), 100)
        serverIface.registerServerCache(MyCache(serverIface), 100)
```

### Filter Registration Summary

| Filter Type | Base Class | Registration Method | Priority |
|-------------|------------|---------------------|----------|
| I/O Filter | `QgsServerFilter` | `registerFilter(filter, priority)` | Lower = first |
| Access Control | `QgsAccessControlFilter` | `registerAccessControl(filter, priority)` | Lower = first |
| Cache | `QgsServerCacheFilter` | `registerServerCache(filter, priority)` | Lower = first |

### Plugin Loading

QGIS Server discovers plugins from the path set by the `QGIS_PLUGINPATH` environment variable. All Python directories in that path with a valid `metadata.txt` containing `server=True` and a `serverClassFactory()` function are loaded at server startup.

---

## 8. FCGI Deployment Patterns

[CONFIRMED] Source: https://docs.qgis.org/latest/en/docs/server_manual/containerized_deployment.html

### Architecture

QGIS Server runs as a FastCGI application (`qgis_mapserv.fcgi`) behind a reverse-proxy HTTP server (Nginx, Apache). A separate development server binary (`qgis_mapserver`) is available for testing.

### Standalone FCGI with spawn-fcgi

```bash
/usr/bin/xvfb-run --auto-servernum --server-num=1 \
    /usr/bin/spawn-fcgi -p 5555 -n -d /home/qgis \
    -- /usr/lib/cgi-bin/qgis_mapserv.fcgi
```

Components:
- `xvfb-run` — Virtual framebuffer (required for rendering without display)
- `spawn-fcgi` — Spawns FCGI processes, binds to port 5555
- `qgis_mapserv.fcgi` — The actual QGIS Server FCGI binary

### Nginx Reverse Proxy Configuration

[CONFIRMED] Source: https://docs.qgis.org/latest/en/docs/server_manual/containerized_deployment.html

```nginx
server {
    listen 80;
    server_name qgis-server;

    location /ogc/ {
        fastcgi_pass  qgis-server:5555;
        fastcgi_param QUERY_STRING       $query_string;
        fastcgi_param REQUEST_METHOD     $request_method;
        fastcgi_param CONTENT_TYPE       $content_type;
        fastcgi_param CONTENT_LENGTH     $content_length;
        fastcgi_param SERVER_NAME        $server_name;
        fastcgi_param SERVER_PORT        $server_port;
        fastcgi_param REQUEST_URI        $request_uri;
        fastcgi_param SCRIPT_NAME       /ogc/;
    }
}
```

### Docker Deployment

Standard Dockerfile pattern:

```dockerfile
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
    qgis-server \
    spawn-fcgi \
    xvfb \
    xauth

ENV QGIS_SERVER_LOG_STDERR=1
ENV QGIS_SERVER_LOG_LEVEL=0
ENV QGIS_PREFIX_PATH=/usr

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["/usr/bin/xvfb-run", "--auto-servernum", "--server-num=1", \
     "/usr/bin/spawn-fcgi", "-p", "5555", "-n", "-d", "/home/qgis", \
     "--", "/usr/lib/cgi-bin/qgis_mapserv.fcgi"]
```

### Scaling

Since QGIS Server is NOT thread-safe, scaling MUST use multiprocessing:
- Docker Compose/Swarm: multiple replicas of the server container
- Kubernetes: Deployment with replica count > 1
- Each replica runs its own `qgis_mapserv.fcgi` process

---

## 9. Available OGC Services

[CONFIRMED] Source: https://docs.qgis.org/latest/en/docs/server_manual/index.html, https://docs.qgis.org/latest/en/docs/server_manual/services/wms.html

### WMS (Web Map Service)

Versions: 1.1.1, 1.3.0

Standard operations:
- `GetCapabilities` — XML metadata about the server
- `GetMap` — Rendered map image
- `GetFeatureInfo` — Feature data at a pixel location
- `GetLegendGraphic` — Legend symbols
- `GetStyle(s)` — SLD style descriptions
- `DescribeLayer` — WFS/WCS availability info

Vendor-specific operations:
- `GetPrint` — QGIS print layout rendering
- `GetProjectSettings` — QGIS-specific project info
- `GetSchemaExtension` — Extended capabilities XML

### WFS (Web Feature Service)

[CONFIRMED] Supported. Serves vector features as GML/GeoJSON. Supports WFS-T (transactional) for editing when access control allows it.

### WCS (Web Coverage Service)

[CONFIRMED] Supported. Serves raster coverages.

### WMTS (Web Map Tile Service)

[CONFIRMED] Supported. Serves pre-rendered or on-the-fly map tiles.

### OGC API Features (WFS3)

[CONFIRMED] Supported. Modern REST/JSON-based feature service. Configuration:
- `QGIS_SERVER_API_RESOURCES_DIRECTORY` — Static resources for OGC API
- `QGIS_SERVER_API_WFS3_MAX_LIMIT` — Max features per request (default: 10000)

---

## 10. QgsServerInterface: Complete Reference

[CONFIRMED] Source: https://qgis.org/pyqgis/master/server/QgsServerInterface.html

The central interface passed to all server plugins. Provides access to all server subsystems.

| Method | Return Type | Purpose |
|--------|-------------|---------|
| `registerFilter(filter, priority)` | None | Register I/O filter |
| `registerAccessControl(filter, priority)` | None | Register access control filter |
| `registerServerCache(filter, priority)` | None | Register cache filter (v3.4+) |
| `filters()` | QgsServerFiltersMap | Get registered filters |
| `accessControls()` | QgsAccessControl | Get access control manager |
| `cacheManager()` | QgsServerCacheManager | Get cache manager (v3.4+) |
| `requestHandler()` | QgsRequestHandler | Get current request handler |
| `capabilitiesCache()` | QgsCapabilitiesCache | Get capabilities cache |
| `configFilePath()` | str | Get config file path |
| `setConfigFilePath(path)` | None | Set config file path |
| `serviceRegistry()` | QgsServiceRegistry | Get service registry |
| `getEnv(name)` | str | Get environment variable |
| `removeConfigCacheEntry(path)` | None | Remove config cache entry |
| `reloadSettings()` | None | Reload server settings (v3.28+) |

---

## 11. Server Environment Variables Reference

[CONFIRMED] Source: https://docs.qgis.org/latest/en/docs/server_manual/config.html

### Critical Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `QGIS_PLUGINPATH` | — | Python plugin directory |
| `QGIS_PROJECT_FILE` | — | Default project file |
| `QGIS_OPTIONS_PATH` | — | Path to QGIS3.ini |
| `QGIS_SERVER_LOG_STDERR` | false | Enable stderr logging |
| `QGIS_SERVER_LOG_LEVEL` | 0 | 0=INFO, 1=WARNING, 2=CRITICAL |
| `QGIS_SERVER_LOG_PROFILE` | false | Detailed profiling |
| `QGIS_SERVER_PARALLEL_RENDERING` | false | Parallel WMS GetMap |
| `QGIS_SERVER_MAX_THREADS` | -1 | Thread count for parallel rendering |
| `QGIS_SERVER_PROJECT_CACHE_SIZE` | 100 | Max cached projects |
| `QGIS_SERVER_PROJECT_CACHE_STRATEGY` | filesystem | `filesystem`, `periodic`, `off` |
| `QGIS_SERVER_CACHE_SIZE` | 50 | Network cache in MB |
| `QGIS_SERVER_WMS_MAX_HEIGHT` | -1 | Max WMS image height |
| `QGIS_SERVER_WMS_MAX_WIDTH` | -1 | Max WMS image width |
| `QGIS_SERVER_FORCE_READONLY_LAYERS` | false | Force read-only mode |
| `QGIS_SERVER_IGNORE_BAD_LAYERS` | false | Allow broken layers |
| `QGIS_SERVER_TRUST_LAYER_METADATA` | false | Trust project metadata |
| `QGIS_SERVER_DISABLE_GETPRINT` | false | Disable GetPrint |
| `QGIS_SERVER_API_WFS3_MAX_LIMIT` | 10000 | Max OGC API features |
| `QGIS_SERVER_LANDING_PAGE_PREFIX` | — | Landing page URL prefix |
| `QGIS_SERVER_ALLOWED_EXTRA_SQL_TOKENS` | — | Extra SQL tokens in filters |
| `QGIS_SERVER_APPLICATION_NAME` | "QGIS3 server" | DB connection identifier |

---

## 12. Complete Server Module Class Inventory

[CONFIRMED] Source: https://qgis.org/pyqgis/master/server/index.html

Total: 40 classes in the `qgis.server` module.

### By Category

**Core**: QgsServer, QgsServerInterface, QgsServerRequest, QgsServerResponse, QgsBufferServerRequest, QgsBufferServerResponse, QgsFcgiServerRequest, QgsRequestHandler

**Services**: QgsService, QgsServiceModule, QgsServiceRegistry

**OGC API**: QgsServerApi, QgsServerOgcApi, QgsServerOgcApiHandler, QgsServerStaticHandler, QgsServerApiContext, QgsServerApiUtils

**Filters**: QgsServerFilter, QgsAccessControlFilter, QgsAccessControl, QgsServerCacheFilter, QgsServerCacheManager, QgsFeatureFilter, QgsFeatureFilterProviderGroup

**Configuration**: QgsConfigCache, QgsCapabilitiesCache, QgsServerSettings, QgsServerSettingsEnv, QgsServerLogger

**Parameters**: QgsServerParameter, QgsServerParameters, QgsServerParameterDefinition, QgsServerQueryStringParameter

**Exceptions**: QgsServerException, QgsOgcServiceException, QgsServerApiBadRequestException, QgsServerApiInternalServerError

**Utilities**: QgsServerProjectUtils, QgsServerFeatureId, QgsStoreBadLayerInfo

---

## Sources

- PyQGIS Developer Cookbook — Server chapter: https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/server.html
- QGIS Server Manual: https://docs.qgis.org/latest/en/docs/server_manual/index.html
- QGIS Server Manual — Configuration: https://docs.qgis.org/latest/en/docs/server_manual/config.html
- QGIS Server Manual — Containerized Deployment: https://docs.qgis.org/latest/en/docs/server_manual/containerized_deployment.html
- PyQGIS Server API Reference: https://qgis.org/pyqgis/master/server/index.html
- QgsServerInterface API: https://qgis.org/pyqgis/master/server/QgsServerInterface.html
- QgsAccessControlFilter API: https://qgis.org/pyqgis/master/server/QgsAccessControlFilter.html
- QgsServerCacheFilter API: https://qgis.org/pyqgis/master/server/QgsServerCacheFilter.html
- QGIS Source — Test Server: https://github.com/qgis/QGIS/blob/master/tests/src/python/qgis_wrapped_server.py
- 3Liz WPS Server Plugin: https://github.com/3liz/qgis-wps4server
