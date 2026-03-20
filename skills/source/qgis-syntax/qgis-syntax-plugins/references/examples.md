# qgis-syntax-plugins — Examples

## Example 1: Minimal Plugin Skeleton

Complete, working minimal plugin with toolbar button and menu entry.

### File: `__init__.py`

```python
def classFactory(iface):
    """Load the plugin class."""
    from .minimal_plugin import MinimalPlugin
    return MinimalPlugin(iface)
```

### File: `metadata.txt`

```ini
[general]
name=Minimal Plugin
qgisMinimumVersion=3.0
description=A minimal QGIS plugin skeleton
about=Demonstrates the minimum required plugin structure with toolbar icon and menu entry.
version=0.1.0
author=Developer Name
email=dev@example.com
repository=https://github.com/developer/minimal-plugin
icon=icon.png
tags=example,skeleton,minimal
category=Vector
```

### File: `minimal_plugin.py`

```python
from qgis.PyQt.QtWidgets import QAction, QMessageBox
from qgis.PyQt.QtGui import QIcon
from qgis.core import Qgis
import os


class MinimalPlugin:
    """Minimal QGIS plugin demonstrating required lifecycle methods."""

    def __init__(self, iface):
        self.iface = iface
        self.plugin_dir = os.path.dirname(__file__)
        self.actions = []

    def initGui(self):
        """Create the menu entries and toolbar icons."""
        icon_path = os.path.join(self.plugin_dir, "icon.png")
        action = QAction(
            QIcon(icon_path),
            "Minimal Plugin",
            self.iface.mainWindow()
        )
        action.setObjectName("minimalPluginAction")
        action.triggered.connect(self.run)

        self.iface.addToolBarIcon(action)
        self.iface.addPluginToMenu("&Minimal Plugin", action)
        self.actions.append(action)

    def unload(self):
        """Remove the plugin menu item and icon."""
        for action in self.actions:
            self.iface.removePluginMenu("&Minimal Plugin", action)
            self.iface.removeToolBarIcon(action)

    def run(self):
        """Run the plugin logic."""
        layer = self.iface.activeLayer()
        if layer is None:
            self.iface.messageBar().pushMessage(
                "Minimal Plugin",
                "No active layer selected",
                level=Qgis.Warning,
                duration=3
            )
            return

        self.iface.messageBar().pushMessage(
            "Minimal Plugin",
            f"Active layer: {layer.name()} ({layer.featureCount()} features)",
            level=Qgis.Info,
            duration=5
        )
```

---

## Example 2: Dialog Plugin with Qt Designer

Plugin that shows a dialog built with Qt Designer.

### File: `dialog.ui` (Qt Designer)

Create this file in Qt Designer. It defines a dialog with an input field and OK/Cancel buttons.

### File: `dialog.py`

```python
from qgis.PyQt.QtWidgets import QDialog
from qgis.PyQt import uic
import os

FORM_CLASS, _ = uic.loadUiType(
    os.path.join(os.path.dirname(__file__), "dialog.ui")
)


class BufferDialog(QDialog, FORM_CLASS):
    """Dialog for buffer distance input."""

    def __init__(self, parent=None):
        super().__init__(parent)
        self.setupUi(self)
```

### File: `buffer_plugin.py`

```python
from qgis.PyQt.QtWidgets import QAction
from qgis.PyQt.QtGui import QIcon
from qgis.core import (
    Qgis, QgsProject, QgsVectorLayer, QgsFeature, QgsGeometry
)
import os
import processing

from .dialog import BufferDialog


class BufferPlugin:
    def __init__(self, iface):
        self.iface = iface
        self.plugin_dir = os.path.dirname(__file__)
        self.actions = []
        self.dlg = None

    def initGui(self):
        icon_path = os.path.join(self.plugin_dir, "icon.png")
        action = QAction(
            QIcon(icon_path),
            "Buffer Tool",
            self.iface.mainWindow()
        )
        action.setObjectName("bufferPluginAction")
        action.triggered.connect(self.run)

        self.iface.addToolBarIcon(action)
        self.iface.addPluginToVectorMenu("&Buffer Tool", action)
        self.actions.append(action)

    def unload(self):
        for action in self.actions:
            self.iface.removePluginVectorMenu("&Buffer Tool", action)
            self.iface.removeToolBarIcon(action)

    def run(self):
        if self.dlg is None:
            self.dlg = BufferDialog(self.iface.mainWindow())

        result = self.dlg.exec_()
        if result:
            distance = self.dlg.distanceSpinBox.value()
            layer = self.iface.activeLayer()
            if layer is None:
                return

            params = {
                "INPUT": layer,
                "DISTANCE": distance,
                "SEGMENTS": 5,
                "OUTPUT": "memory:"
            }
            result = processing.run("native:buffer", params)
            QgsProject.instance().addMapLayer(result["OUTPUT"])
```

---

## Example 3: Dock Widget Plugin

Plugin with a persistent side panel.

```python
from qgis.PyQt.QtWidgets import (
    QAction, QDockWidget, QVBoxLayout, QWidget,
    QLabel, QPushButton, QListWidget
)
from qgis.PyQt.QtCore import Qt
from qgis.PyQt.QtGui import QIcon
from qgis.core import Qgis, QgsProject
import os


class LayerInfoPlugin:
    def __init__(self, iface):
        self.iface = iface
        self.plugin_dir = os.path.dirname(__file__)
        self.actions = []
        self.dock = None

    def initGui(self):
        icon_path = os.path.join(self.plugin_dir, "icon.png")
        action = QAction(
            QIcon(icon_path),
            "Layer Info Panel",
            self.iface.mainWindow()
        )
        action.setObjectName("layerInfoAction")
        action.setCheckable(True)
        action.triggered.connect(self.toggle_dock)

        self.iface.addToolBarIcon(action)
        self.iface.addPluginToMenu("&Layer Info", action)
        self.actions.append(action)

        # Create dock widget
        self.dock = QDockWidget("Layer Info", self.iface.mainWindow())
        self.dock.setObjectName("layerInfoDock")

        # Build dock content
        container = QWidget()
        layout = QVBoxLayout(container)
        self.info_label = QLabel("Select a layer")
        self.refresh_btn = QPushButton("Refresh")
        self.refresh_btn.clicked.connect(self.refresh_info)
        self.field_list = QListWidget()

        layout.addWidget(self.info_label)
        layout.addWidget(self.field_list)
        layout.addWidget(self.refresh_btn)

        self.dock.setWidget(container)
        self.iface.addDockWidget(Qt.RightDockWidgetArea, self.dock)

        # Connect to layer change signal
        self.iface.currentLayerChanged.connect(self.on_layer_changed)

    def unload(self):
        # Disconnect signals
        self.iface.currentLayerChanged.disconnect(self.on_layer_changed)

        # Remove dock widget
        self.iface.removeDockWidget(self.dock)
        del self.dock
        self.dock = None

        # Remove menu and toolbar entries
        for action in self.actions:
            self.iface.removePluginMenu("&Layer Info", action)
            self.iface.removeToolBarIcon(action)

    def toggle_dock(self, checked):
        if self.dock is not None:
            self.dock.setVisible(checked)

    def on_layer_changed(self, layer):
        self.refresh_info()

    def refresh_info(self):
        layer = self.iface.activeLayer()
        self.field_list.clear()

        if layer is None:
            self.info_label.setText("No layer selected")
            return

        self.info_label.setText(
            f"{layer.name()} - {layer.featureCount()} features"
        )

        if hasattr(layer, "fields"):
            for field in layer.fields():
                self.field_list.addItem(
                    f"{field.name()} ({field.typeName()})"
                )
```

---

## Example 4: Plugin with Settings

```python
from qgis.core import QgsSettings


class SettingsPlugin:
    SETTING_PREFIX = "SettingsPlugin"

    def __init__(self, iface):
        self.iface = iface
        self.settings = QgsSettings()

    def save_last_directory(self, path):
        self.settings.setValue(
            f"{self.SETTING_PREFIX}/lastDirectory", path
        )

    def load_last_directory(self):
        return self.settings.value(
            f"{self.SETTING_PREFIX}/lastDirectory", ""
        )

    def save_buffer_distance(self, distance):
        self.settings.setValue(
            f"{self.SETTING_PREFIX}/bufferDistance", distance
        )

    def load_buffer_distance(self):
        return float(self.settings.value(
            f"{self.SETTING_PREFIX}/bufferDistance", 100.0
        ))
```

---

## Example 5: Plugin with Custom Toolbar

```python
from qgis.PyQt.QtWidgets import QAction, QToolBar
from qgis.PyQt.QtGui import QIcon
import os


class MultiToolPlugin:
    def __init__(self, iface):
        self.iface = iface
        self.plugin_dir = os.path.dirname(__file__)
        self.actions = []
        self.toolbar = None

    def initGui(self):
        # Create a dedicated toolbar
        self.toolbar = self.iface.addToolBar("My Tools")
        self.toolbar.setObjectName("myToolsToolbar")

        # Add multiple actions
        for name, callback in [
            ("Tool A", self.run_a),
            ("Tool B", self.run_b),
            ("Tool C", self.run_c),
        ]:
            action = QAction(name, self.iface.mainWindow())
            action.setObjectName(f"myTools_{name.replace(' ', '')}")
            action.triggered.connect(callback)
            self.toolbar.addAction(action)
            self.iface.addPluginToMenu("&My Tools", action)
            self.actions.append(action)

    def unload(self):
        for action in self.actions:
            self.iface.removePluginMenu("&My Tools", action)

        # Remove the custom toolbar
        if self.toolbar is not None:
            del self.toolbar
            self.toolbar = None

    def run_a(self):
        pass

    def run_b(self):
        pass

    def run_c(self):
        pass
```

---

## Example 6: Plugin Builder Workflow

Plugin Builder is a QGIS plugin that generates a complete plugin skeleton.

### Steps

1. Install Plugin Builder from QGIS Plugin Manager (Plugins > Manage and Install Plugins)
2. Run Plugin Builder (Plugins > Plugin Builder > Plugin Builder)
3. Fill in plugin details (name, module name, description, author)
4. Choose plugin type (Tool button, Dialog, Dock widget, Processing provider)
5. Plugin Builder generates all required files

### Generated Files

```
my_generated_plugin/
├── __init__.py              # classFactory entry point
├── metadata.txt             # Plugin metadata
├── my_generated_plugin.py   # Main plugin class
├── my_generated_plugin_dialog.py  # Dialog class
├── my_generated_plugin_dialog_base.ui  # Qt Designer form
├── resources.qrc            # Resource definitions
├── icon.png                 # Default icon
├── Makefile                 # Build automation
├── pb_tool.cfg              # Plugin Builder config
├── pylintrc                 # Linting config
├── README.txt               # Documentation
└── test/                    # Test directory
    └── __init__.py
```

### Build Commands (from Makefile)

```bash
# Compile resources
pyrcc5 -o resources.py resources.qrc

# Compile UI files
pyuic5 -o my_generated_plugin_dialog_base.py my_generated_plugin_dialog_base.ui

# Deploy to QGIS plugin directory
# Copy plugin folder to ~/.local/share/QGIS/QGIS3/profiles/default/python/plugins/
```
