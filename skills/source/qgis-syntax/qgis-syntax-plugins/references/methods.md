# qgis-syntax-plugins — Methods Reference

## Plugin Lifecycle Methods

### classFactory(iface)

| Aspect | Detail |
|--------|--------|
| Location | `__init__.py` |
| Parameter | `iface` — `QgisInterface` instance |
| Returns | Plugin class instance |
| Called by | QGIS plugin loader on startup |

```python
def classFactory(iface):
    from .mainPlugin import MyPlugin
    return MyPlugin(iface)
```

### __init__(self, iface)

| Aspect | Detail |
|--------|--------|
| Purpose | Store iface reference, initialize variables |
| Rules | NEVER create GUI elements here — use `initGui()` instead |
| Rules | ALWAYS store `self.iface = iface` |

### initGui(self)

| Aspect | Detail |
|--------|--------|
| Purpose | Create and register ALL GUI elements |
| Called when | Plugin is activated by user or at QGIS startup if enabled |
| Rules | ALWAYS track created elements as instance attributes |
| Rules | ALWAYS set `objectName` on QActions |
| Rules | ALWAYS use `self.iface.mainWindow()` as parent |

### unload(self)

| Aspect | Detail |
|--------|--------|
| Purpose | Remove ALL GUI elements and disconnect ALL signals |
| Called when | Plugin is deactivated or QGIS shuts down |
| Rules | MUST remove every menu item added in `initGui()` |
| Rules | MUST remove every toolbar icon added in `initGui()` |
| Rules | MUST remove every dock widget added in `initGui()` |
| Rules | MUST disconnect every signal connected in `initGui()` |

---

## metadata.txt Fields

### Required Fields

| Field | Description | Example |
|-------|-------------|---------|
| `name` | Display name of the plugin | `My Awesome Plugin` |
| `qgisMinimumVersion` | Minimum QGIS version | `3.0` |
| `description` | One-line summary | `Performs spatial analysis on vector layers` |
| `about` | Multi-line detailed description | `This plugin provides...` |
| `version` | Semantic version | `1.0.0` |
| `author` | Author name | `John Doe` |
| `email` | Author email | `john@example.com` |
| `repository` | Source code URL | `https://github.com/author/plugin` |

### Optional Fields

| Field | Description | Default |
|-------|-------------|---------|
| `qgisMaximumVersion` | Maximum QGIS version | `major.99` |
| `category` | Plugin category | None |
| `icon` | Icon filename (relative to plugin dir) | None |
| `experimental` | Mark as experimental | `False` |
| `deprecated` | Mark as deprecated | `False` |
| `tags` | Comma-separated search tags | None |
| `homepage` | Plugin homepage URL | None |
| `tracker` | Issue tracker URL | None |
| `changelog` | Version changelog text | None |
| `hasProcessingProvider` | Plugin provides Processing algorithms | `no` |
| `server` | Plugin is for QGIS Server | `False` |
| `plugin_dependencies` | Comma-separated plugin names | None |

### Category Values

| Value | Use for |
|-------|---------|
| `Raster` | Raster data analysis plugins |
| `Vector` | Vector data analysis plugins |
| `Database` | Database interaction plugins |
| `Mesh` | Mesh data plugins |
| `Web` | Web service plugins |

---

## QgisInterface (iface) Methods

### Menu Integration

| Method | Purpose |
|--------|---------|
| `addPluginToMenu(name, action)` | Add action to Plugins > name submenu |
| `removePluginMenu(name, action)` | Remove action from Plugins > name submenu |
| `addPluginToVectorMenu(name, action)` | Add to Vector menu |
| `removePluginVectorMenu(name, action)` | Remove from Vector menu |
| `addPluginToRasterMenu(name, action)` | Add to Raster menu |
| `removePluginRasterMenu(name, action)` | Remove from Raster menu |
| `addPluginToDatabaseMenu(name, action)` | Add to Database menu |
| `removePluginDatabaseMenu(name, action)` | Remove from Database menu |
| `addPluginToWebMenu(name, action)` | Add to Web menu |
| `removePluginWebMenu(name, action)` | Remove from Web menu |

### Toolbar Integration

| Method | Purpose |
|--------|---------|
| `addToolBarIcon(action)` | Add icon to the Plugins toolbar |
| `removeToolBarIcon(action)` | Remove icon from the Plugins toolbar |
| `addToolBar(name)` | Create a new named toolbar |
| `addToolBarWidget(widget)` | Add custom widget to Plugins toolbar |

### Dock Widget Integration

| Method | Purpose |
|--------|---------|
| `addDockWidget(area, widget)` | Add a dock widget to the main window |
| `removeDockWidget(widget)` | Remove a dock widget |

### UI Access

| Method | Returns | Purpose |
|--------|---------|---------|
| `mainWindow()` | `QMainWindow` | Main QGIS window (use as parent for dialogs) |
| `mapCanvas()` | `QgsMapCanvas` | The map canvas widget |
| `layerTreeView()` | `QgsLayerTreeView` | Layer panel tree view |
| `activeLayer()` | `QgsMapLayer` or `None` | Currently selected layer |
| `messageBar()` | `QgsMessageBar` | Message bar for notifications |
| `statusBarIface()` | `QgsStatusBar` | Access to status bar |

### Layer Operations

| Method | Purpose |
|--------|---------|
| `addVectorLayer(path, name, provider)` | Add vector layer to project |
| `addRasterLayer(path, name, provider)` | Add raster layer to project |
| `setActiveLayer(layer)` | Set the active layer |
| `zoomToActiveLayer()` | Zoom to active layer extent |

---

## QgsSettings Methods

| Method | Purpose |
|--------|---------|
| `setValue(key, value)` | Store a setting |
| `value(key, defaultValue=None)` | Retrieve a setting |
| `remove(key)` | Remove a setting |
| `contains(key)` | Check if setting exists |
| `allKeys()` | Get all setting keys |
| `childGroups()` | Get child group names |
| `beginGroup(prefix)` | Set group prefix |
| `endGroup()` | End group prefix |

ALWAYS prefix plugin settings with the plugin name: `"MyPlugin/settingName"`.

---

## Resource Compilation Commands

| Command | Purpose |
|---------|---------|
| `pyrcc5 -o resources.py resources.qrc` | Compile Qt resources to Python |
| `pyuic5 -o ui_dialog.py dialog.ui` | Compile Qt Designer .ui to Python |

NEVER commit compiled files to version control. ALWAYS regenerate during build.
