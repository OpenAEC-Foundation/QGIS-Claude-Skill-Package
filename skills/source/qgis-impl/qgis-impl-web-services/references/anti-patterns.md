# qgis-impl-web-services — Anti-patterns

## AP-01: Not Checking Layer Validity After Construction

**WRONG:**
```python
rlayer = QgsRasterLayer(wms_uri, "My WMS", "wms")
QgsProject.instance().addMapLayer(rlayer)  # May add invalid layer
```

**RIGHT:**
```python
rlayer = QgsRasterLayer(wms_uri, "My WMS", "wms")
if rlayer.isValid():
    QgsProject.instance().addMapLayer(rlayer)
else:
    print(f"Layer failed to load: {rlayer.error().summary()}")
```

**WHY:** An invalid WMS/WFS layer silently fails with no exception. Adding an invalid layer to the project causes rendering errors and confusing blank map areas. ALWAYS check `isValid()` immediately after construction.

---

## AP-02: Hardcoding Credentials in URI Strings

**WRONG:**
```python
uri = (
    "crs=EPSG:4326&format=image/png&layers=buildings"
    "&url=https://server/wms"
    "&username=admin&password=secret123"
)
rlayer = QgsRasterLayer(uri, "Buildings", "wms")
```

**RIGHT:**
```python
from qgis.core import QgsDataSourceUri, QgsRasterLayer

quri = QgsDataSourceUri()
quri.setParam("layers", "buildings")
quri.setParam("format", "image/png")
quri.setParam("authcfg", "fm1s770")  # Stored auth config ID
quri.setParam("url", "https://server/wms")
quri.setParam("crs", "EPSG:4326")

rlayer = QgsRasterLayer(
    str(quri.encodedUri(), "utf-8"), "Buildings", "wms"
)
```

**WHY:** Hardcoded credentials appear in project files, log output, and layer properties dialogs. They are visible to anyone who opens the project. ALWAYS use `QgsAuthManager` with stored authentication configurations (`authcfg`).

---

## AP-03: Not URL-Encoding Curly Braces in XYZ URIs

**WRONG:**
```python
# Curly braces may be interpreted as parameter placeholders
uri = "type=xyz&url=https://tile.server.com/{z}/{x}/{y}.png&zmax=19&zmin=0"
```

**RIGHT:**
```python
# URL-encode curly braces in the URI parameter value
uri = "type=xyz&url=https://tile.server.com/%7Bz%7D/%7Bx%7D/%7By%7D.png&zmax=19&zmin=0"
```

**NOTE:** In practice, QGIS handles both encoded and unencoded forms for XYZ URLs passed to `QgsRasterLayer`. However, when constructing URIs programmatically via `QgsDataSourceUri.setParam()`, ALWAYS use the encoded form (`%7B` / `%7D`) to prevent parameter parsing issues.

---

## AP-04: Using Wrong Provider Key for WFS

**WRONG:**
```python
# Provider key is case-sensitive
vlayer = QgsVectorLayer(wfs_uri, "WFS Layer", "wfs")  # lowercase fails
```

**RIGHT:**
```python
vlayer = QgsVectorLayer(wfs_uri, "WFS Layer", "WFS")  # uppercase required
```

**WHY:** The WFS provider key is `"WFS"` (uppercase). Using lowercase `"wfs"` causes QGIS to fail to find the provider, resulting in an invalid layer with no clear error message.

---

## AP-05: Using classFactory Instead of serverClassFactory for Server Plugins

**WRONG:**
```python
# __init__.py for a server plugin
def classFactory(iface):  # This is for DESKTOP plugins
    from .MyPlugin import MyPlugin
    return MyPlugin(iface)
```

**RIGHT:**
```python
# __init__.py for a server plugin
def serverClassFactory(serverIface):  # This is for SERVER plugins
    from .MyPlugin import MyServerPlugin
    return MyServerPlugin(serverIface)
```

**WHY:** QGIS Server looks for `serverClassFactory()`, not `classFactory()`. Using the desktop entry point means the plugin is silently ignored by the server. The `serverIface` parameter is a `QgsServerInterface`, not `QgisInterface`.

---

## AP-06: Missing server=True in metadata.txt

**WRONG:**
```ini
[general]
name=MyServerPlugin
description=A server plugin
version=1.0.0
qgisMinimumVersion=3.0
# No server=True line
```

**RIGHT:**
```ini
[general]
name=MyServerPlugin
description=A server plugin
version=1.0.0
qgisMinimumVersion=3.0
server=True
```

**WHY:** Without `server=True` in `metadata.txt`, QGIS Server skips the plugin directory entirely during plugin discovery. There is no warning or error message -- the plugin is silently not loaded.

---

## AP-07: Using QGIS Server Classes from Multiple Threads

**WRONG:**
```python
import threading
from qgis.server import QgsServer

server = QgsServer()

def handle(url):
    # NEVER share QgsServer across threads
    request = QgsBufferServerRequest(url)
    response = QgsBufferServerResponse()
    server.handleRequest(request, response)

threads = [threading.Thread(target=handle, args=(url,)) for url in urls]
for t in threads:
    t.start()
```

**RIGHT:**
```python
# Use multiprocessing or container-based scaling
# Each process gets its own QgsServer instance
from multiprocessing import Process
from qgis.core import QgsApplication
from qgis.server import QgsServer, QgsBufferServerRequest, QgsBufferServerResponse

def worker(url):
    app = QgsApplication([], False)
    server = QgsServer()
    request = QgsBufferServerRequest(url)
    response = QgsBufferServerResponse()
    server.handleRequest(request, response)
    app.exitQgis()

processes = [Process(target=worker, args=(url,)) for url in urls]
for p in processes:
    p.start()
```

**WHY:** QGIS Server classes are explicitly NOT thread-safe. Sharing a `QgsServer` instance across threads causes race conditions, data corruption, and crashes. ALWAYS use multiprocessing or deploy multiple container instances.

---

## AP-08: Ignoring Filter Return Values

**WRONG:**
```python
class MyFilter(QgsServerFilter):
    def onRequestReady(self):
        # Forgot to return True/False
        request = self.serverInterface().requestHandler()
        params = request.parameterMap()
        # No return statement — defaults to None (falsy)
```

**RIGHT:**
```python
class MyFilter(QgsServerFilter):
    def onRequestReady(self) -> bool:
        request = self.serverInterface().requestHandler()
        params = request.parameterMap()
        return True  # ALWAYS explicitly return True to continue chain
```

**WHY:** Filter callbacks form a chain. If a callback returns `False` (or `None`, which is falsy), the chain stops and subsequent filters are not executed. ALWAYS explicitly return `True` to propagate to the next filter, or `False` only when intentionally stopping the chain.

---

## AP-09: Assuming WMS Supports All CRS

**WRONG:**
```python
# Assuming the server supports EPSG:28992 without checking
uri = "crs=EPSG:28992&format=image/png&layers=data&url=https://server/wms"
rlayer = QgsRasterLayer(uri, "My WMS", "wms")
```

**RIGHT:**
```python
# First check capabilities, then request with a supported CRS
# The GetCapabilities response lists supported CRS per layer
uri = "crs=EPSG:4326&format=image/png&layers=data&url=https://server/wms"
rlayer = QgsRasterLayer(uri, "My WMS", "wms")
if not rlayer.isValid():
    # Try a different CRS or check GetCapabilities
    uri = "crs=EPSG:3857&format=image/png&layers=data&url=https://server/wms"
    rlayer = QgsRasterLayer(uri, "My WMS", "wms")
```

**WHY:** WMS servers only support specific CRS per layer, as declared in their GetCapabilities response. Requesting an unsupported CRS results in an invalid layer or a server error. ALWAYS verify CRS support before loading.

---

## AP-10: Mixing WFS Version Parameter Names

**WRONG:**
```python
# WFS 2.0.0 uses typeNames (plural), not typeName (singular)
uri = (
    "https://server/wfs?service=WFS&version=2.0.0"
    "&request=GetFeature&typeName=ns:layer"  # Wrong for 2.0.0
)
```

**RIGHT:**
```python
# WFS 2.0.0: typeNames (plural)
uri = (
    "https://server/wfs?service=WFS&version=2.0.0"
    "&request=GetFeature&typeNames=ns:layer"
)

# WFS 1.0.0: typeName (singular)
uri = (
    "https://server/wfs?service=WFS&version=1.0.0"
    "&request=GetFeature&typeName=ns:layer"
)
```

**WHY:** WFS 2.0.0 renamed `typeName` to `typeNames` (plural). Using the wrong parameter name for the version causes the server to ignore the layer selection, returning either an error or unexpected results.

---

## AP-11: Manual String Concatenation for Service URIs

**WRONG:**
```python
uri = (
    "crs=" + crs + "&format=" + fmt + "&layers=" + layer
    + "&url=" + url + "&username=" + user + "&password=" + pwd
)
```

**RIGHT:**
```python
from qgis.core import QgsDataSourceUri

quri = QgsDataSourceUri()
quri.setParam("crs", crs)
quri.setParam("format", fmt)
quri.setParam("layers", layer)
quri.setParam("url", url)
quri.setParam("authcfg", auth_id)  # Use auth manager instead of plain creds
uri_string = str(quri.encodedUri(), "utf-8")
```

**WHY:** Manual string concatenation is error-prone: special characters in values break the URI, ampersands in URLs cause parameter splitting, and credentials end up in plain text. ALWAYS use `QgsDataSourceUri` for programmatic URI construction.

---

## AP-12: Not Setting QGIS_PLUGINPATH for Server Plugins

**WRONG:**
```bash
# Plugin placed in default desktop plugin path
~/.local/share/QGIS/QGIS3/profiles/default/python/plugins/MyServerPlugin/
```

**RIGHT:**
```bash
# Set QGIS_PLUGINPATH environment variable
export QGIS_PLUGINPATH=/opt/qgis-server-plugins

# Place plugin in that path
/opt/qgis-server-plugins/MyServerPlugin/
    __init__.py
    MyServerPlugin.py
    metadata.txt
```

**WHY:** QGIS Server does NOT use the desktop plugin paths. Server plugins MUST be placed in the directory specified by the `QGIS_PLUGINPATH` environment variable. Without this variable set, no server plugins are loaded.
