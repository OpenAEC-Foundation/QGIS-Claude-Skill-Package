---
name: qgis-syntax-plugins
description: >
  Use when creating QGIS plugins, adding menu items, or integrating custom UI into QGIS.
  Prevents plugin lifecycle violations and resource cleanup failures.
  Covers plugin structure, metadata.txt, classFactory/initGui/unload, Qt Designer, and Plugin Repository publishing.
  Keywords: QGIS plugin, classFactory, initGui, unload, metadata.txt, iface, Plugin Builder, plugin repository.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-syntax-plugins

## Quick Reference

### Plugin File Structure

| File | Required | Purpose |
|------|----------|---------|
| `__init__.py` | YES | Contains `classFactory()` entry point |
| `metadata.txt` | YES | Plugin metadata (name, version, description) |
| `mainPlugin.py` | YES | Main plugin class with `initGui()` and `unload()` |
| `resources.qrc` | NO | Qt resource definitions (icons, assets) |
| `resources.py` | NO | Compiled resources via `pyrcc5` |
| `form.ui` | NO | Qt Designer form file |
| `icon.png` | NO | Plugin icon (recommended) |
| `LICENSE` | YES* | Required for QGIS Plugin Repository submission |

### Plugin Lifecycle

```
QGIS Startup
    |
    v
classFactory(iface) ---------> returns plugin instance
    |
    v
initGui() -------------------> create menus, toolbars, actions, dock widgets
    |
    v
[plugin active -- user interacts]
    |
    v
unload() --------------------> remove ALL GUI elements, disconnect ALL signals
    |
    v
Plugin deactivated
```

### Plugin Installation Paths

| Platform | Path |
|----------|------|
| User plugins | `~/AppData/Roaming/QGIS/QGIS3/profiles/default/python/plugins` (Windows) |
| User plugins | `~/.local/share/QGIS/QGIS3/profiles/default/python/plugins` (Linux) |
| System plugins | `<qgis_prefix>/python/plugins` |
| Custom path | Set `QGIS_PLUGINPATH` environment variable |

### Key iface Methods

| Method | Purpose |
|--------|---------|
| `iface.addToolBarIcon(action)` | Add icon to plugin toolbar |
| `iface.removeToolBarIcon(action)` | Remove icon from plugin toolbar |
| `iface.addPluginToMenu(menu_name, action)` | Add to Plugins menu |
| `iface.removePluginMenu(menu_name, action)` | Remove from Plugins menu |
| `iface.addPluginToVectorMenu(name, action)` | Add to Vector menu |
| `iface.addPluginToRasterMenu(name, action)` | Add to Raster menu |
| `iface.addPluginToDatabaseMenu(name, action)` | Add to Database menu |
| `iface.addPluginToWebMenu(name, action)` | Add to Web menu |
| `iface.addDockWidget(area, widget)` | Add dock widget to main window |
| `iface.removeDockWidget(widget)` | Remove dock widget |
| `iface.mainWindow()` | Get main window (use as parent for dialogs) |
| `iface.mapCanvas()` | Get the map canvas |
| `iface.activeLayer()` | Get currently selected layer |
| `iface.messageBar()` | Get the message bar for notifications |

---

## Critical Warnings

**ALWAYS** implement `unload()` to remove ALL GUI elements, menu entries, toolbar icons, and dock widgets added in `initGui()`. Failure causes ghost UI elements that persist after plugin deactivation.

**ALWAYS** use `self.iface.mainWindow()` as the parent for QActions and dialogs. This ensures proper window management and garbage collection.

**ALWAYS** disconnect ALL signal connections in `unload()`. Dangling connections cause crashes when signals fire after plugin objects are destroyed.

**NEVER** access `iface`, `QgsProject.instance()`, or any GUI object from a background thread. This causes segfaults and silent crashes.

**NEVER** raise exceptions in `QgsTask.run()`. Return `False` to indicate failure instead.

**NEVER** leave compiled files (`resources_rc.py`, `ui_*.py`) in the repository. Generate them during build with `pyrcc5` and `pyuic5`.

**ALWAYS** set `objectName` on QActions via `action.setObjectName("uniqueName")`. This prevents conflicts with other plugins and enables QGIS to save toolbar customizations.

**ALWAYS** track every GUI element you create in `initGui()` as an instance attribute so `unload()` can remove it.

---

## Decision Tree: Plugin Type Selection

```
What does the plugin need to do?
|
+-- Add a toolbar button / menu item that runs a function?
|   --> Minimal plugin (QAction + run method)
|
+-- Show a dialog with input fields?
|   --> Dialog plugin (QAction + QDialog from .ui file)
|
+-- Show a persistent panel?
|   --> Dock widget plugin (QDockWidget added via iface.addDockWidget)
|
+-- Add a Processing algorithm?
|   --> Processing provider plugin (see qgis-syntax-processing-scripts)
|
+-- Interact with the map canvas?
|   --> Map tool plugin (QgsMapTool subclass)
```

---

## Essential Patterns

### Pattern 1: classFactory Entry Point

The `__init__.py` file MUST contain `classFactory()`:

```python
def classFactory(iface):
    """Load the plugin class. Called by QGIS on plugin startup."""
    from .mainPlugin import MyPlugin
    return MyPlugin(iface)
```

QGIS calls this function with the `QgisInterface` object. It MUST return an instance of the plugin class.

### Pattern 2: Plugin Class Skeleton

```python
from qgis.PyQt.QtWidgets import QAction
from qgis.PyQt.QtGui import QIcon
import os

class MyPlugin:
    def __init__(self, iface):
        self.iface = iface
        self.plugin_dir = os.path.dirname(__file__)
        self.actions = []

    def initGui(self):
        icon_path = os.path.join(self.plugin_dir, "icon.png")
        action = QAction(
            QIcon(icon_path),
            "My Plugin",
            self.iface.mainWindow()
        )
        action.setObjectName("myPluginAction")
        action.triggered.connect(self.run)
        self.iface.addToolBarIcon(action)
        self.iface.addPluginToMenu("&My Plugin", action)
        self.actions.append(action)

    def unload(self):
        for action in self.actions:
            self.iface.removePluginMenu("&My Plugin", action)
            self.iface.removeToolBarIcon(action)

    def run(self):
        pass
```

### Pattern 3: metadata.txt Required Fields

```ini
[general]
name=My Plugin Name
qgisMinimumVersion=3.0
description=Short one-line description
about=Longer multi-line description
version=1.0.0
author=Author Name
email=author@example.com
repository=https://github.com/author/my-plugin
```

**Required fields:** name, qgisMinimumVersion, description, about, version, author, email, repository.

If `qgisMaximumVersion` is omitted, it defaults to `major.99` (e.g., `3.99` for `qgisMinimumVersion=3.0`).

### Pattern 4: Plugin Settings Storage

```python
from qgis.core import QgsSettings

class MyPlugin:
    def __init__(self, iface):
        self.iface = iface
        self.settings = QgsSettings()

    def save_setting(self, key, value):
        self.settings.setValue(f"MyPlugin/{key}", value)

    def load_setting(self, key, default=None):
        return self.settings.value(f"MyPlugin/{key}", default)
```

ALWAYS prefix settings keys with the plugin name to avoid collisions.

### Pattern 5: Qt Designer Dialog Integration

```python
from qgis.PyQt import uic
import os

FORM_CLASS, _ = uic.loadUiType(
    os.path.join(os.path.dirname(__file__), "dialog.ui")
)

class MyDialog(QDialog, FORM_CLASS):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.setupUi(self)
```

### Pattern 6: Resource Compilation

Define resources in `resources.qrc`:
```xml
<RCC>
  <qresource prefix="/plugins/myplugin">
    <file>icon.png</file>
  </qresource>
</RCC>
```

Compile: `pyrcc5 -o resources.py resources.qrc`

Import in plugin: `from . import resources`

---

## Common Operations

### Add to Specific Menu Category

```python
# Vector menu
self.iface.addPluginToVectorMenu("&My Plugin", self.action)
self.iface.removePluginVectorMenu("&My Plugin", self.action)

# Raster menu
self.iface.addPluginToRasterMenu("&My Plugin", self.action)
self.iface.removePluginRasterMenu("&My Plugin", self.action)

# Database menu
self.iface.addPluginToDatabaseMenu("&My Plugin", self.action)
self.iface.removePluginDatabaseMenu("&My Plugin", self.action)
```

### Add a Dock Widget

```python
from qgis.PyQt.QtCore import Qt
from qgis.PyQt.QtWidgets import QDockWidget, QWidget

def initGui(self):
    self.dock = QDockWidget("My Panel", self.iface.mainWindow())
    self.dock.setObjectName("myPluginDock")
    self.dock.setWidget(QWidget())
    self.iface.addDockWidget(Qt.RightDockWidgetArea, self.dock)

def unload(self):
    self.iface.removeDockWidget(self.dock)
    del self.dock
```

### Show Messages to Users

```python
# Message bar (non-blocking)
from qgis.core import Qgis
self.iface.messageBar().pushMessage(
    "My Plugin", "Operation complete", level=Qgis.Success, duration=3
)

# Log messages (for debugging)
from qgis.core import QgsMessageLog
QgsMessageLog.logMessage("Debug info", "My Plugin", Qgis.Info)
```

### Publishing Checklist

1. Plugin folder name: ASCII letters, digits, underscore, minus ONLY. NEVER start with a digit.
2. Package as ZIP: `plugin.zip` containing `pluginfolder/` with all files.
3. Submit to https://plugins.qgis.org/ (requires OSGeo ID).
4. Staff approval required before publication.
5. Version MUST be unique across submissions.

---

## Reference Links

- [references/methods.md](references/methods.md) -- Plugin lifecycle methods, metadata.txt fields, QgisInterface methods
- [references/examples.md](references/examples.md) -- Complete plugin skeletons, dialog plugins, dock widget plugins
- [references/anti-patterns.md](references/anti-patterns.md) -- Plugin development pitfalls and fixes

### Official Sources

- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/plugins/index.html
- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/plugins/plugins.html
- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/plugins/releasing.html
- https://qgis.org/pyqgis/master/gui/QgisInterface.html
