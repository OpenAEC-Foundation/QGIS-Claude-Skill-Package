---
name: qgis-impl-web-services
description: >
  Use when loading WMS/WFS/WMTS layers, configuring XYZ tile sources, or setting up QGIS Server.
  Prevents URI format errors for OGC web services and server misconfiguration.
  Covers WMS/WMTS/WFS/WCS clients, XYZ tiles, QGIS Server, server filters, and OGC API features.
  Keywords: WMS, WFS, WMTS, WCS, XYZ tiles, OGC, QGIS Server, web service, tile layer, OGC API.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-impl-web-services

## Quick Reference

### OGC Client Layer Types

| Service | Provider | Layer Class | URI Style |
|---------|----------|-------------|-----------|
| WMS | `"wms"` | `QgsRasterLayer` | Key-value params |
| WMTS | `"wms"` | `QgsRasterLayer` | Key-value params + tilematrixset |
| WFS | `"WFS"` | `QgsVectorLayer` | Full URL with query params |
| WCS | `"wcs"` | `QgsRasterLayer` | Key-value params |
| XYZ Tiles | `"wms"` | `QgsRasterLayer` | `type=xyz&url=...` |

### Exact URI Format Strings

**WMS:**
```
crs=EPSG:4326&format=image/png&layers=layername&styles=&url=https://server/wms
```

**WMTS:**
```
crs=EPSG:3857&format=image/png&layers=layername&styles=default&tilematrixset=GoogleMapsCompatible&url=https://server/wmts
```

**WFS:**
```
https://server/wfs?service=WFS&version=2.0.0&request=GetFeature&typename=namespace:layername
```

**WCS:**
```
url=https://server/wcs&identifier=coveragename&crs=EPSG:4326
```

**XYZ Tiles:**
```
type=xyz&url=https://tile.server.com/%7Bz%7D/%7Bx%7D/%7By%7D.png&zmax=19&zmin=0&crs=EPSG3857
```

### Server Filter Registration

| Filter Type | Base Class | Registration Method |
|-------------|------------|---------------------|
| I/O Filter | `QgsServerFilter` | `serverIface.registerFilter(filter, priority)` |
| Access Control | `QgsAccessControlFilter` | `serverIface.registerAccessControl(filter, priority)` |
| Cache | `QgsServerCacheFilter` | `serverIface.registerServerCache(filter, priority)` |

Priority: lower number = invoked first.

---

## Critical Warnings

**NEVER** assume a WMS layer supports all CRS -- ALWAYS check the GetCapabilities response before requesting a specific CRS.

**ALWAYS** verify layer validity immediately after construction with `layer.isValid()`. An invalid WMS/WFS layer silently fails with no exception.

**ALWAYS** URL-encode curly braces in XYZ tile URLs when building the URI parameter string: `{z}` becomes `%7Bz%7D`, `{x}` becomes `%7Bx%7D`, `{y}` becomes `%7By%7D`.

**NEVER** hardcode credentials in URI strings. ALWAYS use `QgsAuthManager` with stored authentication configurations (`authcfg` parameter).

**NEVER** use QGIS Server classes from multiple threads -- they are NOT thread-safe. ALWAYS use multiprocessing or container-based scaling.

**ALWAYS** set `server=True` in `metadata.txt` for server plugins -- without it, QGIS Server will NOT load the plugin.

**ALWAYS** use `serverClassFactory()` (not `classFactory()`) as the entry point for server plugins.

**ALWAYS** check filter callback return values -- returning `False` from `onRequestReady()`, `onSendResponse()`, or `onResponseComplete()` stops filter chain propagation.

---

## Decision Tree

### Which OGC client to use?

```
Need map imagery (rendered tiles/images)?
├── YES: Need pre-rendered tiles?
│   ├── YES: Is the server OGC WMTS?
│   │   ├── YES → Use WMTS (provider "wms" + tilematrixset param)
│   │   └── NO → Use XYZ Tiles (provider "wms" + type=xyz)
│   └── NO → Use WMS (provider "wms")
└── NO: Need vector features (geometry + attributes)?
    ├── YES → Use WFS (provider "WFS")
    └── NO: Need raster coverage data (raw values)?
        └── YES → Use WCS (provider "wcs")
```

### Which WFS version?

```
Need paging or temporal filters?
├── YES → WFS 2.0.0 (uses typeNames, plural)
└── NO: Need stored queries?
    ├── YES → WFS 1.1.0
    └── NO → WFS 1.0.0 (uses typeName, singular)
```

### Server plugin filter type?

```
Need to control layer/feature/attribute visibility?
├── YES → QgsAccessControlFilter
└── NO: Need to modify request/response content?
    ├── YES → QgsServerFilter
    └── NO: Need to cache responses?
        └── YES → QgsServerCacheFilter
```

---

## Essential Patterns

### Loading a WMS Layer

```python
from qgis.core import QgsRasterLayer, QgsProject

url_with_params = (
    "crs=EPSG:4326"
    "&format=image/png"
    "&layers=continents"
    "&styles="
    "&url=https://demo.mapserver.org/cgi-bin/wms"
)
rlayer = QgsRasterLayer(url_with_params, "WMS Layer", "wms")
if rlayer.isValid():
    QgsProject.instance().addMapLayer(rlayer)
```

### Loading a WMTS Layer

```python
from qgis.core import QgsRasterLayer, QgsProject

url_with_params = (
    "crs=EPSG:3857"
    "&format=image/png"
    "&layers=layername"
    "&styles=default"
    "&tilematrixset=GoogleMapsCompatible"
    "&url=https://server/wmts"
)
rlayer = QgsRasterLayer(url_with_params, "WMTS Layer", "wms")
if rlayer.isValid():
    QgsProject.instance().addMapLayer(rlayer)
```

### Loading a WFS Layer

```python
from qgis.core import QgsVectorLayer, QgsProject

uri = (
    "https://demo.mapserver.org/cgi-bin/wfs"
    "?service=WFS"
    "&version=2.0.0"
    "&request=GetFeature"
    "&typename=ms:cities"
)
vlayer = QgsVectorLayer(uri, "WFS Cities", "WFS")
if vlayer.isValid():
    QgsProject.instance().addMapLayer(vlayer)
```

### Loading XYZ Tiles

```python
from qgis.core import QgsRasterLayer, QgsProject

xyz_url = "https://tile.openstreetmap.org/{z}/{x}/{y}.png"
uri = f"type=xyz&url={xyz_url}&zmax=19&zmin=0&crs=EPSG3857"
rlayer = QgsRasterLayer(uri, "OpenStreetMap", "wms")
if rlayer.isValid():
    QgsProject.instance().addMapLayer(rlayer)
```

### Loading a WCS Layer

```python
from qgis.core import QgsRasterLayer, QgsProject

layer_name = "modis"
url = f"https://demo.mapserver.org/cgi-bin/wcs?identifier={layer_name}"
rlayer = QgsRasterLayer(url, "WCS Coverage", "wcs")
if rlayer.isValid():
    QgsProject.instance().addMapLayer(rlayer)
```

### Authenticated WMS with QgsDataSourceUri

```python
from qgis.core import QgsDataSourceUri, QgsRasterLayer, QgsProject

auth_cfg = "fm1s770"  # Stored auth config ID from QgsAuthManager

quri = QgsDataSourceUri()
quri.setParam("layers", "usa:states")
quri.setParam("format", "image/png")
quri.setParam("authcfg", auth_cfg)
quri.setParam("url", "https://server/wms")
quri.setParam("crs", "EPSG:4326")
quri.setParam("styles", "")

rlayer = QgsRasterLayer(
    str(quri.encodedUri(), "utf-8"), "Authenticated WMS", "wms"
)
if rlayer.isValid():
    QgsProject.instance().addMapLayer(rlayer)
```

### Storing Authentication Config

```python
from qgis.core import QgsAuthMethodConfig, QgsApplication

config = QgsAuthMethodConfig()
config.setName("My WMS Auth")
config.setMethod("Basic")
config.setUri("https://server/wms")
config.setConfig("username", "myuser")
config.setConfig("password", "mypassword")

auth_mgr = QgsApplication.authManager()
auth_mgr.storeAuthenticationConfig(config)
auth_id = config.id()  # Auto-generated ID like "fm1s770"
```

Supported auth methods: Basic, PKI-Paths, PKI-PKCS#12, Identity-Cert, OAuth2, APIHeader, MapTilerHmacSha256.

---

## Common Operations

### QGIS Server: Handling a Request

```python
from qgis.core import QgsApplication
from qgis.server import QgsServer, QgsBufferServerRequest, QgsBufferServerResponse

app = QgsApplication([], False)
server = QgsServer()

request = QgsBufferServerRequest(
    "http://localhost:8081/?SERVICE=WMS&REQUEST=GetCapabilities"
    "&MAP=/path/to/project.qgs"
)
response = QgsBufferServerResponse()
server.handleRequest(request, response)

print(response.headers())
print(response.body().data().decode("utf8"))
app.exitQgis()
```

### Server Plugin Structure

```
$QGIS_PLUGINPATH/
  MyServerPlugin/
    __init__.py        # MUST contain serverClassFactory()
    MyServerPlugin.py  # Plugin implementation
    metadata.txt       # MUST contain server=True
```

**__init__.py:**
```python
def serverClassFactory(serverIface):
    from .MyServerPlugin import MyServerPluginServer
    return MyServerPluginServer(serverIface)
```

**Plugin class with filter registration:**
```python
class MyServerPluginServer:
    def __init__(self, serverIface):
        serverIface.registerFilter(MyFilter(serverIface), 100)
```

### I/O Filter (QgsServerFilter)

```python
from qgis.server import QgsServerFilter

class HelloFilter(QgsServerFilter):
    def __init__(self, serverIface):
        super().__init__(serverIface)

    def onRequestReady(self) -> bool:
        request = self.serverInterface().requestHandler()
        params = request.parameterMap()
        return True  # Return False to stop filter chain

    def onSendResponse(self) -> bool:
        return True

    def onResponseComplete(self) -> bool:
        request = self.serverInterface().requestHandler()
        params = request.parameterMap()
        if params.get("SERVICE", "").upper() == "HELLO":
            request.clear()
            request.setResponseHeader("Content-type", "text/plain")
            request.appendBody(b"HelloServer!")
        return True
```

### Access Control Filter

```python
from qgis.server import QgsAccessControlFilter

class MyAccessControl(QgsAccessControlFilter):
    def layerFilterExpression(self, layer):
        return '"public" = true'

    def layerFilterSubsetString(self, layer):
        return "public = true"

    def layerPermissions(self, layer):
        rights = QgsAccessControlFilter.LayerPermissions()
        rights.canRead = True
        rights.canInsert = False
        rights.canUpdate = False
        rights.canDelete = False
        return rights

    def authorizedLayerAttributes(self, layer, attributes):
        return [a for a in attributes if a != "secret_field"]

    def allowToEdit(self, layer, feature):
        return False

    def cacheKey(self):
        return "my_access_control"
```

### OGC API Handler

```python
from qgis.PyQt.QtCore import QRegularExpression
from qgis.server import (
    QgsServerOgcApi, QgsServerOgcApiHandler,
    QgsServerQueryStringParameter,
)

class MyApiHandler(QgsServerOgcApiHandler):
    def __init__(self):
        super().__init__()
        self.setContentTypes([QgsServerOgcApi.HTML, QgsServerOgcApi.JSON])

    def path(self):
        return QRegularExpression("/myapi")

    def operationId(self):
        return "MyApiEndpoint"

    def summary(self):
        return "My custom endpoint"

    def description(self):
        return "My custom endpoint description"

    def linkTitle(self):
        return "My API"

    def linkType(self):
        return QgsServerOgcApi.data

    def handleRequest(self, context):
        values = self.values(context)
        self.write({"result": "ok"}, context)

    def templatePath(self, context):
        return ""

    def parameters(self, context):
        return [
            QgsServerQueryStringParameter(
                "param1", True,
                QgsServerQueryStringParameter.Type.String,
                "A required parameter",
            ),
        ]
```

**Registration:**
```python
class MyApiPlugin:
    def __init__(self, serverIface):
        api = QgsServerOgcApi(
            serverIface, "/myapi", "My API",
            "Custom API endpoint", "1.0"
        )
        api.registerHandler(MyApiHandler())
        serverIface.serviceRegistry().registerApi(api)
```

### Custom OGC Service

```python
from qgis.server import QgsService

class CustomService(QgsService):
    def name(self):
        return "CUSTOM"

    def version(self):
        return "1.0.0"

    def executeRequest(self, request, response, project):
        response.setStatusCode(200)
        response.write("Custom service response")

# Register: serverIface.serviceRegistry().registerService(CustomService())
```

### Key Server Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `QGIS_PLUGINPATH` | -- | Python plugin directory |
| `QGIS_PROJECT_FILE` | -- | Default project file |
| `QGIS_SERVER_LOG_STDERR` | false | Enable stderr logging |
| `QGIS_SERVER_LOG_LEVEL` | 0 | 0=INFO, 1=WARNING, 2=CRITICAL |
| `QGIS_SERVER_PARALLEL_RENDERING` | false | Parallel WMS GetMap |
| `QGIS_SERVER_MAX_THREADS` | -1 | Thread count for parallel rendering |
| `QGIS_SERVER_PROJECT_CACHE_SIZE` | 100 | Max cached projects |
| `QGIS_SERVER_WMS_MAX_HEIGHT` | -1 | Max WMS image height |
| `QGIS_SERVER_WMS_MAX_WIDTH` | -1 | Max WMS image width |
| `QGIS_SERVER_API_WFS3_MAX_LIMIT` | 10000 | Max OGC API features per request |

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for web service classes, server filters, and OGC API handlers
- [references/examples.md](references/examples.md) -- Working code examples for all OGC client types and server patterns
- [references/anti-patterns.md](references/anti-patterns.md) -- Common mistakes with URI formats, authentication, and server configuration

### Official Sources

- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/loadlayer.html
- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/server.html
- https://docs.qgis.org/latest/en/docs/server_manual/index.html
- https://docs.qgis.org/latest/en/docs/server_manual/config.html
- https://qgis.org/pyqgis/master/server/index.html
