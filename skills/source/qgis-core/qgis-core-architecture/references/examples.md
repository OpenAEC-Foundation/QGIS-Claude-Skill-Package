# Working Code Examples (QGIS 3.44+ / PyQGIS 3.x)

## Example 1: Standalone Script -- Load and Query a Vector Layer

```python
from qgis.core import (
    QgsApplication,
    QgsVectorLayer,
    QgsProject,
    QgsFeatureRequest,
)

# Initialize QGIS (headless)
QgsApplication.setPrefixPath("/usr/share/qgis", True)
qgs = QgsApplication([], False)
qgs.initQgis()

# Load a GeoPackage layer
layer = QgsVectorLayer(
    "data/buildings.gpkg|layername=buildings",
    "Buildings",
    "ogr"
)
if not layer.isValid():
    raise RuntimeError("Layer failed to load")

QgsProject.instance().addMapLayer(layer)

# Query features with an expression filter
request = QgsFeatureRequest().setFilterExpression('"area_sqm" > 500')
for feature in layer.getFeatures(request):
    print(f"Building {feature['name']}: {feature['area_sqm']} sqm")

print(f"Total features: {layer.featureCount()}")
print(f"CRS: {layer.crs().authid()}")

# ALWAYS clean up
qgs.exitQgis()
```

---

## Example 2: Plugin Entry Point with Full Lifecycle

```python
from qgis.PyQt.QtWidgets import QAction, QMessageBox
from qgis.PyQt.QtGui import QIcon
from qgis.core import QgsProject


class AreaCalculatorPlugin:
    def __init__(self, iface):
        self.iface = iface

    def initGui(self):
        self.action = QAction(
            QIcon(":/plugins/area_calc/icon.png"),
            "Calculate Areas",
            self.iface.mainWindow()
        )
        self.action.triggered.connect(self.run)
        self.iface.addToolBarIcon(self.action)
        self.iface.addPluginToMenu("Area Calculator", self.action)

    def unload(self):
        # ALWAYS clean up all registered GUI elements
        self.iface.removeToolBarIcon(self.action)
        self.iface.removePluginFromMenu("Area Calculator", self.action)

    def run(self):
        layer = self.iface.activeLayer()
        if layer is None:
            QMessageBox.warning(
                self.iface.mainWindow(),
                "No Layer",
                "Select a vector layer first."
            )
            return

        if not layer.isValid():
            QMessageBox.critical(
                self.iface.mainWindow(),
                "Invalid Layer",
                "The selected layer is not valid."
            )
            return

        total_area = 0.0
        for feature in layer.getFeatures():
            geom = feature.geometry()
            if geom and not geom.isNull():
                total_area += geom.area()

        QMessageBox.information(
            self.iface.mainWindow(),
            "Result",
            f"Total area: {total_area:.2f} map units squared"
        )


def classFactory(iface):
    return AreaCalculatorPlugin(iface)
```

---

## Example 3: Project Operations -- Load, Modify, Save

```python
from qgis.core import (
    QgsApplication,
    QgsProject,
    QgsVectorLayer,
    Qgis,
)

QgsApplication.setPrefixPath("/usr/share/qgis", True)
qgs = QgsApplication([], False)
qgs.initQgis()

project = QgsProject.instance()

# Load project with read flags (skip layer resolution for fast access)
readflags = Qgis.ProjectReadFlags()
readflags |= Qgis.ProjectReadFlag.DontResolveLayers
project.read("/projects/analysis.qgz", readflags)

print(f"Project CRS: {project.crs().authid()}")
print(f"Project home: {project.homePath()}")
print(f"Layers (unresolved): {len(project.mapLayers())}")

# Clear and load fresh
project.clear()

# Full load with layer resolution
project.read("/projects/analysis.qgz")
print(f"Layers (resolved): {len(project.mapLayers())}")

# Add a new layer
new_layer = QgsVectorLayer(
    "Point?crs=EPSG:4326&field=name:string(50)&field=value:double",
    "Analysis Points",
    "memory"
)
if new_layer.isValid():
    project.addMapLayer(new_layer)

# Save as new project
project.write("/projects/analysis_updated.qgz")

qgs.exitQgis()
```

---

## Example 4: Layer Tree Organization

```python
from qgis.core import (
    QgsProject,
    QgsVectorLayer,
    QgsRasterLayer,
    QgsLayerTreeGroup,
    QgsLayerTreeLayer,
)

project = QgsProject.instance()
root = project.layerTreeRoot()

# Create groups
basemap_group = root.addGroup("Basemaps")
analysis_group = root.addGroup("Analysis")

# Add raster to basemap group
raster = QgsRasterLayer("data/ortho.tif", "Orthophoto", "gdal")
if raster.isValid():
    project.addMapLayer(raster, False)  # False = don't add to root
    basemap_group.addLayer(raster)

# Add vector to analysis group
vector = QgsVectorLayer("data/parcels.gpkg|layername=parcels", "Parcels", "ogr")
if vector.isValid():
    project.addMapLayer(vector, False)
    analysis_group.addLayer(vector)

# Traverse the layer tree
def print_tree(group, indent=0):
    for child in group.children():
        prefix = "  " * indent
        if isinstance(child, QgsLayerTreeGroup):
            print(f"{prefix}Group: {child.name()}")
            print_tree(child, indent + 1)
        elif isinstance(child, QgsLayerTreeLayer):
            layer = child.layer()
            valid = "valid" if layer and layer.isValid() else "INVALID"
            print(f"{prefix}Layer: {child.name()} ({valid})")

print_tree(root)
```

---

## Example 5: Background Processing with QgsTask

```python
from qgis.core import QgsTask, QgsApplication, QgsProject, QgsVectorLayer


class BufferTask(QgsTask):
    """Creates buffers around features in a background thread."""

    def __init__(self, layer_id, buffer_distance):
        super().__init__("Buffer Analysis", QgsTask.CanCancel)
        self.layer_id = layer_id
        self.buffer_distance = buffer_distance
        self.result_features = []
        self.error_message = None

    def run(self):
        """Runs on BACKGROUND thread -- NO GUI access, NO project writes."""
        layer = QgsProject.instance().mapLayer(self.layer_id)
        if not layer:
            self.error_message = f"Layer {self.layer_id} not found"
            return False

        total = layer.featureCount()
        for i, feature in enumerate(layer.getFeatures()):
            if self.isCanceled():
                return False

            geom = feature.geometry()
            if geom and not geom.isNull():
                buffered = geom.buffer(self.buffer_distance, 8)
                self.result_features.append((feature.id(), buffered))

            self.setProgress((i + 1) / total * 100)

        return True

    def finished(self, result):
        """Runs on MAIN thread -- safe to update GUI and project."""
        if result and self.result_features:
            # Create result layer on main thread
            result_layer = QgsVectorLayer(
                "Polygon?crs=EPSG:28992",
                "Buffer Results",
                "memory"
            )
            # Add features to result layer...
            QgsProject.instance().addMapLayer(result_layer)
        elif self.error_message:
            print(f"Task failed: {self.error_message}")


# Usage (inside QGIS context)
layer = iface.activeLayer()
if layer and layer.isValid():
    task = BufferTask(layer.id(), 100.0)
    QgsApplication.taskManager().addTask(task)
```

---

## Example 6: Signal/Slot Pattern for Layer Monitoring

```python
from qgis.core import QgsProject, QgsMapLayer


def on_layers_added(layers):
    """Called when layers are added to the project."""
    for layer in layers:
        print(f"Layer added: {layer.name()} ({layer.providerType()})")
        if not layer.isValid():
            print(f"  WARNING: Layer {layer.name()} is invalid!")


def on_layers_removed(layer_ids):
    """Called when layers are removed from the project."""
    for layer_id in layer_ids:
        print(f"Layer removed: {layer_id}")


# Connect signals
project = QgsProject.instance()
project.layersAdded.connect(on_layers_added)
project.layersRemoved.connect(on_layers_removed)

# Later, disconnect when no longer needed
project.layersAdded.disconnect(on_layers_added)
project.layersRemoved.disconnect(on_layers_removed)
```

---

## Example 7: Provider Registry Inspection

```python
from qgis.core import QgsProviderRegistry

registry = QgsProviderRegistry.instance()

# List all available providers
print("Available providers:")
for provider_name in sorted(registry.providerList()):
    metadata = registry.providerMetadata(provider_name)
    if metadata:
        print(f"  {provider_name}: {metadata.description()}")
    else:
        print(f"  {provider_name}")
```

---

## Example 8: Path Resolution for Cross-Platform Projects

```python
import os
from qgis.core import QgsProject, QgsPathResolver


# Rewrite paths when loading a project from a different machine
def rewrite_paths(path):
    replacements = {
        "C:/Users/OldUser/Projects": "/home/newuser/projects",
        "\\\\server\\share": "/mnt/share",
    }
    for old, new in replacements.items():
        if path.startswith(old):
            return path.replace(old, new, 1)
    return path


preprocessor_id = QgsPathResolver.setPathPreprocessor(rewrite_paths)

# Load the project -- paths are rewritten during load
QgsProject.instance().read("/home/newuser/projects/analysis.qgz")

# Remove the preprocessor after loading
QgsPathResolver.removePathPreprocessor(preprocessor_id)

# Use os.path for constructing new paths
project_home = QgsProject.instance().homePath()
output_dir = os.path.join(project_home, "output")
os.makedirs(output_dir, exist_ok=True)
```

---

## Example 9: Forward-Compatible Field Creation (QGIS 3.x + 4.x)

```python
from qgis.core import QgsField, QgsVectorLayer

# QGIS 3.x approach (works now, breaks in 4.x)
from qgis.PyQt.QtCore import QVariant
field_3x = QgsField("name", QVariant.String, len=100)

# Forward-compatible approach for QGIS 4.x (Qt6)
# Use this when targeting QGIS 3.40+ for future-proofing
try:
    from qgis.PyQt.QtCore import QMetaType
    field_4x = QgsField("name", QMetaType.Type.QString)
except ImportError:
    # Fallback for older QGIS versions
    from qgis.PyQt.QtCore import QVariant
    field_4x = QgsField("name", QVariant.String)
```
