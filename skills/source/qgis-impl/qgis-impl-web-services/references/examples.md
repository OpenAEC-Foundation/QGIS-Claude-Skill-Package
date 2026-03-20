# qgis-impl-web-services — Examples

## Example 1: Load Multiple WMS Layers from a Single Server

```python
from qgis.core import QgsRasterLayer, QgsProject

base_url = "https://demo.mapserver.org/cgi-bin/wms"
layers_to_load = ["continents", "cities", "rivers"]

for layer_name in layers_to_load:
    uri = (
        f"crs=EPSG:4326"
        f"&format=image/png"
        f"&layers={layer_name}"
        f"&styles="
        f"&url={base_url}"
    )
    rlayer = QgsRasterLayer(uri, f"WMS - {layer_name}", "wms")
    if rlayer.isValid():
        QgsProject.instance().addMapLayer(rlayer)
    else:
        print(f"FAILED to load WMS layer: {layer_name}")
```

---

## Example 2: Load XYZ Basemap with Attribution

```python
from qgis.core import QgsRasterLayer, QgsProject

# OpenStreetMap
osm_url = "https://tile.openstreetmap.org/{z}/{x}/{y}.png"
osm_uri = f"type=xyz&url={osm_url}&zmax=19&zmin=0&crs=EPSG3857"
osm_layer = QgsRasterLayer(osm_uri, "OpenStreetMap", "wms")

if osm_layer.isValid():
    QgsProject.instance().addMapLayer(osm_layer)

# Stamen Terrain (via Stadia Maps)
terrain_url = "https://tiles.stadiamaps.com/tiles/stamen_terrain/{z}/{x}/{y}.png"
terrain_uri = f"type=xyz&url={terrain_url}&zmax=18&zmin=0&crs=EPSG3857"
terrain_layer = QgsRasterLayer(terrain_uri, "Stamen Terrain", "wms")

if terrain_layer.isValid():
    QgsProject.instance().addMapLayer(terrain_layer)
```

---

## Example 3: WFS with Paging (WFS 2.0.0)

```python
from qgis.core import QgsVectorLayer, QgsProject

# WFS 2.0.0 supports paging with startIndex and count
uri = (
    "https://demo.mapserver.org/cgi-bin/wfs"
    "?service=WFS"
    "&version=2.0.0"
    "&request=GetFeature"
    "&typename=ms:cities"
    "&count=100"
    "&startIndex=0"
)
vlayer = QgsVectorLayer(uri, "WFS Cities (Page 1)", "WFS")
if vlayer.isValid():
    QgsProject.instance().addMapLayer(vlayer)
    print(f"Loaded {vlayer.featureCount()} features")
```

---

## Example 4: Authenticated WMS Using QgsDataSourceUri

```python
from qgis.core import (
    QgsDataSourceUri, QgsRasterLayer, QgsProject,
    QgsAuthMethodConfig, QgsApplication,
)

# Step 1: Store credentials (run once)
config = QgsAuthMethodConfig()
config.setName("GeoServer Production")
config.setMethod("Basic")
config.setUri("https://geoserver.example.com/wms")
config.setConfig("username", "admin")
config.setConfig("password", "secret")

auth_mgr = QgsApplication.authManager()
auth_mgr.storeAuthenticationConfig(config)
auth_id = config.id()

# Step 2: Load layer using authcfg
quri = QgsDataSourceUri()
quri.setParam("layers", "workspace:buildings")
quri.setParam("format", "image/png")
quri.setParam("authcfg", auth_id)
quri.setParam("url", "https://geoserver.example.com/wms")
quri.setParam("crs", "EPSG:28992")
quri.setParam("styles", "")

rlayer = QgsRasterLayer(
    str(quri.encodedUri(), "utf-8"), "Buildings WMS", "wms"
)
if rlayer.isValid():
    QgsProject.instance().addMapLayer(rlayer)
```

---

## Example 5: QGIS Server — WMS GetMap Request

```python
from qgis.core import QgsApplication
from qgis.server import QgsServer, QgsBufferServerRequest, QgsBufferServerResponse

app = QgsApplication([], False)
server = QgsServer()

# GetMap request for a rendered image
request = QgsBufferServerRequest(
    "http://localhost:8081/?"
    "MAP=/data/projects/world.qgs"
    "&SERVICE=WMS"
    "&VERSION=1.3.0"
    "&REQUEST=GetMap"
    "&LAYERS=countries"
    "&CRS=EPSG:4326"
    "&BBOX=-90,-180,90,180"
    "&WIDTH=800"
    "&HEIGHT=400"
    "&FORMAT=image/png"
)
response = QgsBufferServerResponse()
server.handleRequest(request, response)

# Save the rendered image
if response.statusCode() == 200:
    with open("/tmp/map_output.png", "wb") as f:
        f.write(response.body())

app.exitQgis()
```

---

## Example 6: Server Filter — Watermark on WMS GetMap

```python
from qgis.server import QgsServerFilter
from qgis.PyQt.QtCore import QByteArray, QBuffer, QIODevice
from qgis.PyQt.QtGui import QImage, QPainter, QRect
import os

class WatermarkFilter(QgsServerFilter):
    def __init__(self, serverIface):
        super().__init__(serverIface)

    def onResponseComplete(self) -> bool:
        request = self.serverInterface().requestHandler()
        params = request.parameterMap()
        if (params.get("SERVICE", "").upper() == "WMS"
                and params.get("REQUEST", "").upper() == "GETMAP"
                and not request.exceptionRaised()):
            img = QImage()
            img.loadFromData(request.body())
            watermark = QImage(os.path.join(
                os.path.dirname(__file__), "media/watermark.png"))
            p = QPainter(img)
            p.drawImage(QRect(20, 20, 40, 40), watermark)
            p.end()
            ba = QByteArray()
            buf = QBuffer(ba)
            buf.open(QIODevice.WriteOnly)
            img.save(buf, "PNG" if "png" in
                request.parameter("FORMAT") else "JPG")
            request.clearBody()
            request.appendBody(ba)
        return True

# Registration in plugin:
# serverIface.registerFilter(WatermarkFilter(serverIface), 200)
```

---

## Example 7: Access Control — Role-Based Layer Filtering

```python
from qgis.server import QgsAccessControlFilter

class RoleBasedAccessControl(QgsAccessControlFilter):
    def __init__(self, server_iface):
        super().__init__(server_iface)

    def layerFilterExpression(self, layer):
        """Only show features marked as public."""
        return "\"visibility\" = 'public'"

    def layerFilterSubsetString(self, layer):
        return "status = 'published'"

    def layerPermissions(self, layer):
        rights = QgsAccessControlFilter.LayerPermissions()
        rights.canRead = True
        rights.canInsert = False
        rights.canUpdate = False
        rights.canDelete = False
        return rights

    def authorizedLayerAttributes(self, layer, attributes):
        hidden = {"password", "internal_id", "api_key"}
        return [a for a in attributes if a not in hidden]

    def allowToEdit(self, layer, feature):
        return False

    def cacheKey(self):
        return "role_public"

# Registration:
# serverIface.registerAccessControl(RoleBasedAccessControl(serverIface), 100)
```

---

## Example 8: Custom OGC API Endpoint

```python
import json
from qgis.PyQt.QtCore import QRegularExpression
from qgis.server import (
    QgsServerOgcApi, QgsServerOgcApiHandler,
    QgsServerQueryStringParameter,
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
        return QRegularExpression("/circles")

    def operationId(self):
        return "CreateCircle"

    def summary(self):
        return "Creates a circle around a point"

    def description(self):
        return "Generates a circular geometry from x, y, and radius parameters"

    def linkTitle(self):
        return "Circle Generator"

    def linkType(self):
        return QgsServerOgcApi.data

    def handleRequest(self, context):
        values = self.values(context)
        x, y, r = values["x"], values["y"], values["r"]
        feature = QgsFeature()
        feature.setAttributes([x, y, r])
        feature.setGeometry(QgsCircle(QgsPoint(x, y), r).toCircularString())
        exporter = QgsJsonExporter()
        self.write(json.loads(exporter.exportFeature(feature)), context)

    def templatePath(self, context):
        return ""

    def parameters(self, context):
        return [
            QgsServerQueryStringParameter(
                "x", True,
                QgsServerQueryStringParameter.Type.Double, "X coordinate"),
            QgsServerQueryStringParameter(
                "y", True,
                QgsServerQueryStringParameter.Type.Double, "Y coordinate"),
            QgsServerQueryStringParameter(
                "r", True,
                QgsServerQueryStringParameter.Type.Double, "Radius"),
        ]

class CircleApiPlugin:
    def __init__(self, serverIface):
        api = QgsServerOgcApi(
            serverIface, "/circles",
            "Circle API", "Generate circles", "1.0")
        api.registerHandler(CircleApiHandler())
        serverIface.serviceRegistry().registerApi(api)
```

---

## Example 9: Custom Service (Full OGC-style)

```python
from qgis.server import QgsService

class HealthCheckService(QgsService):
    def name(self):
        return "HEALTH"

    def version(self):
        return "1.0.0"

    def executeRequest(self, request, response, project):
        response.setStatusCode(200)
        response.setHeader("Content-Type", "application/json")
        response.write('{"status": "ok", "service": "QGIS Server"}')

# Registration in plugin __init__:
# serverIface.serviceRegistry().registerService(HealthCheckService())
# Access via: ?SERVICE=HEALTH&REQUEST=GetCapabilities
```

---

## Example 10: Docker Deployment Configuration

**docker-compose.yml:**
```yaml
version: "3"
services:
  qgis-server:
    image: debian:bookworm-slim
    build:
      context: .
    environment:
      QGIS_SERVER_LOG_STDERR: "1"
      QGIS_SERVER_LOG_LEVEL: "0"
      QGIS_PREFIX_PATH: /usr
      QGIS_PLUGINPATH: /plugins
      QGIS_PROJECT_FILE: /data/project.qgs
    volumes:
      - ./data:/data
      - ./plugins:/plugins
    ports:
      - "5555:5555"

  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "80:80"
    depends_on:
      - qgis-server
```

**nginx.conf:**
```nginx
server {
    listen 80;
    location /ogc/ {
        fastcgi_pass qgis-server:5555;
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
