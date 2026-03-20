---
name: qgis-impl-3d-visualization
description: >
  Use when creating 3D map views, configuring terrain, or rendering vector/point cloud data in 3D.
  Prevents performance issues from unoptimized point cloud rendering and incorrect terrain configuration.
  Covers Qgs3DMapSettings, terrain providers, 3D symbols, materials, camera/lights, and layout integration.
  Keywords: 3D visualization, Qgs3DMapSettings, terrain, point cloud 3D, 3D symbols, Phong material, 3D map, 3D renderer.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-impl-3d-visualization

## Quick Reference

### Import Pattern

3D classes live in `qgis._3d` (underscore prefix because Python modules cannot start with a digit). Base classes live in `qgis.core`.

```python
# 3D-specific classes — ALWAYS import from qgis._3d
from qgis._3d import (
    Qgs3DMapSettings,
    QgsVectorLayer3DRenderer,
    QgsPolygon3DSymbol,
    QgsLine3DSymbol,
    QgsPoint3DSymbol,
    QgsPhongMaterialSettings,
    QgsGoochMaterialSettings,
    QgsMetalRoughMaterialSettings,
    QgsPointLightSettings,
    QgsDirectionalLightSettings,
    QgsCameraPose,
    QgsPointCloudLayer3DRenderer,
    QgsLayoutItem3DMap,
    QgsFlatTerrainSettings,
    QgsDemTerrainSettings,
)

# Base classes — ALWAYS import from qgis.core
from qgis.core import QgsAbstract3DSymbol, QgsAbstract3DRenderer
```

### Symbol Construction Pattern

| Symbol Class | Construction | Notes |
|-------------|-------------|-------|
| `QgsPolygon3DSymbol` | `QgsPolygon3DSymbol.create()` | Factory method returns `QgsAbstract3DSymbol` |
| `QgsLine3DSymbol` | `QgsLine3DSymbol.create()` | Factory method returns `QgsAbstract3DSymbol` |
| `QgsPoint3DSymbol` | `QgsPoint3DSymbol()` | Regular constructor |

### Material Types

| Material | Use Case | Since |
|----------|----------|-------|
| `QgsPhongMaterialSettings` | Standard Phong shading (default choice) | 3.0+ |
| `QgsGoochMaterialSettings` | Non-photorealistic warm/cool rendering | 3.16+ |
| `QgsMetalRoughMaterialSettings` | PBR metallic-roughness | 3.36+ |

### Terrain Providers

| Provider | Class | Use Case |
|----------|-------|----------|
| Flat | `QgsFlatTerrainSettings` | Flat plane at a fixed elevation |
| DEM | `QgsDemTerrainSettings` | Raster elevation model terrain |
| Mesh | `QgsMeshTerrainSettings` | Mesh layer as terrain |

---

## Critical Warnings

**NEVER** use `setMaterial()` on 3D symbols — it is DEPRECATED and removed. ALWAYS use `setMaterialSettings()` instead. Old code examples referencing `setMaterial()` will fail.

**NEVER** import 3D classes from `qgis.core` — they live in `qgis._3d`. The ONLY exceptions are `QgsAbstract3DSymbol` and `QgsAbstract3DRenderer`, which live in `qgis.core`.

**NEVER** construct `QgsPolygon3DSymbol` or `QgsLine3DSymbol` with a regular constructor — ALWAYS use the `create()` factory method. `QgsPoint3DSymbol` is the exception and uses a regular constructor.

**ALWAYS** treat all `qgis._3d` classes as tech preview / unstable API. Method signatures MAY change between QGIS minor versions. ALWAYS verify against the API docs for your specific QGIS version.

**ALWAYS** use `setExtent()` on `Qgs3DMapSettings` before other configuration — it auto-sets the origin to the extent center, which other coordinate-dependent settings rely on.

**NEVER** call deprecated terrain methods on `Qgs3DMapSettings` (`setTerrainVerticalScale()`, `setMaxTerrainScreenError()`, `setMapTileResolution()`) — ALWAYS configure these through the terrain settings object instead.

---

## Decision Tree

### Which symbol class do I need?

```
Geometry type?
├── Point → QgsPoint3DSymbol()
├── Line → QgsLine3DSymbol.create()
├── Polygon → QgsPolygon3DSymbol.create()
└── Point Cloud → QgsPointCloudLayer3DRenderer + QgsPointCloud3DSymbol subclass
```

### Which material do I need?

```
Visual effect needed?
├── Standard shading (most cases) → QgsPhongMaterialSettings
├── Non-photorealistic / technical drawing → QgsGoochMaterialSettings (3.16+)
└── Physically-based rendering → QgsMetalRoughMaterialSettings (3.36+)
```

### Which terrain provider do I need?

```
Terrain data available?
├── No elevation data → QgsFlatTerrainSettings
├── Raster DEM layer → QgsDemTerrainSettings
├── Mesh layer → QgsMeshTerrainSettings
└── Project elevation properties → settings.configureTerrainFromProject()
```

---

## Essential Patterns

### Pattern 1: Configure a 3D Scene

```python
from qgis._3d import Qgs3DMapSettings, QgsFlatTerrainSettings
from qgis.core import QgsCoordinateReferenceSystem, QgsRectangle, QgsProject
from qgis.PyQt.QtGui import QColor

settings = Qgs3DMapSettings()
settings.setCrs(QgsCoordinateReferenceSystem("EPSG:3857"))
settings.setExtent(QgsRectangle(1000000, 6000000, 2000000, 7000000))
settings.setBackgroundColor(QColor(135, 206, 235))
settings.setSelectionColor(QColor(255, 255, 0))

# Flat terrain
terrain = QgsFlatTerrainSettings()
terrain.setElevation(0.0)
settings.setTerrainSettings(terrain)

# Add layers
settings.setLayers(QgsProject.instance().mapLayers().values())
```

### Pattern 2: DEM Terrain

```python
from qgis._3d import QgsDemTerrainSettings

terrain = QgsDemTerrainSettings()
terrain.setLayer(dem_raster_layer)
terrain.setResolution(16)
terrain.setSkirtHeight(10.0)
settings.setTerrainSettings(terrain)

# Terrain rendering options
settings.setTerrainRenderingEnabled(True)
settings.setTerrainShadingEnabled(True)

material = QgsPhongMaterialSettings()
material.setDiffuse(QColor(180, 160, 120))
settings.setTerrainShadingMaterial(material)
```

### Pattern 3: Extruded Building Polygons

```python
from qgis._3d import (
    QgsVectorLayer3DRenderer,
    QgsPolygon3DSymbol,
    QgsPhongMaterialSettings,
)
from qgis.PyQt.QtGui import QColor

material = QgsPhongMaterialSettings()
material.setAmbient(QColor(100, 100, 100))
material.setDiffuse(QColor(180, 180, 160))
material.setSpecular(QColor(255, 255, 255))
material.setShininess(50.0)

symbol = QgsPolygon3DSymbol.create()
symbol.setExtrusionHeight(15.0)
symbol.setMaterialSettings(material)
symbol.setEdgesEnabled(True)
symbol.setEdgeColor(QColor(0, 0, 0))
symbol.setEdgeWidth(1.0)

renderer = QgsVectorLayer3DRenderer(symbol)
building_layer.setRenderer3D(renderer)
```

### Pattern 4: 3D Point Symbols

```python
from qgis._3d import QgsVectorLayer3DRenderer, QgsPoint3DSymbol, QgsPhongMaterialSettings
from qgis.PyQt.QtGui import QColor
from qgis.core import Qgis

material = QgsPhongMaterialSettings()
material.setDiffuse(QColor(255, 0, 0))
material.setShininess(100.0)

symbol = QgsPoint3DSymbol()
symbol.setShape(Qgis.Point3DShape.Sphere)
symbol.setShapeProperties({"radius": 5.0})
symbol.setMaterialSettings(material)

renderer = QgsVectorLayer3DRenderer(symbol)
point_layer.setRenderer3D(renderer)
```

### Pattern 5: 3D Line Symbols

```python
from qgis._3d import QgsVectorLayer3DRenderer, QgsLine3DSymbol, QgsPhongMaterialSettings
from qgis.PyQt.QtGui import QColor

material = QgsPhongMaterialSettings()
material.setDiffuse(QColor(0, 100, 200))

symbol = QgsLine3DSymbol.create()
symbol.setWidth(3.0)
symbol.setExtrusionHeight(10.0)
symbol.setMaterialSettings(material)

renderer = QgsVectorLayer3DRenderer(symbol)
line_layer.setRenderer3D(renderer)
```

### Pattern 6: Lighting Configuration

```python
from qgis._3d import QgsPointLightSettings, QgsDirectionalLightSettings
from qgis.core import QgsVector3D
from qgis.PyQt.QtGui import QColor

# Point light — illuminates from a specific position
point_light = QgsPointLightSettings()
point_light.setPosition(QgsVector3D(0, 1000, 0))
point_light.setColor(QColor(255, 255, 255))
point_light.setIntensity(1.0)
point_light.setConstantAttenuation(1.0)
point_light.setLinearAttenuation(0.0)
point_light.setQuadraticAttenuation(0.0)

# Directional light — simulates sunlight
dir_light = QgsDirectionalLightSettings()
dir_light.setDirection(QgsVector3D(0.5, -1.0, 0.5))
dir_light.setColor(QColor(255, 255, 230))
dir_light.setIntensity(0.8)

settings.setLightSources([point_light, dir_light])
```

### Pattern 7: Camera Pose

```python
from qgis._3d import QgsCameraPose
from qgis.core import QgsVector3D

camera = QgsCameraPose()
camera.setCenterPoint(QgsVector3D(1500000, 6500000, 0))
camera.setDistanceFromCenterPoint(500)
camera.setPitchAngle(45.0)   # 0 = top-down, 90 = horizontal
camera.setHeadingAngle(0.0)  # North-facing
```

### Pattern 8: Eye Dome Lighting (Point Clouds)

```python
# Enhances depth perception — especially useful for point clouds
settings.setEyeDomeLightingEnabled(True)
settings.setEyeDomeLightingStrength(1000.0)
settings.setEyeDomeLightingDistance(1)
```

### Pattern 9: 3D Map in Print Layout

```python
from qgis.core import QgsProject, QgsLayout, QgsLayoutSize, QgsLayoutPoint, QgsUnitTypes
from qgis._3d import QgsLayoutItem3DMap, QgsCameraPose
from qgis.core import QgsVector3D

project = QgsProject.instance()
layout = QgsLayout(project)

map_3d = QgsLayoutItem3DMap(layout)

camera = QgsCameraPose()
camera.setCenterPoint(QgsVector3D(0, 0, 0))
camera.setDistanceFromCenterPoint(500)
camera.setPitchAngle(45.0)
camera.setHeadingAngle(0.0)
map_3d.setCameraPose(camera)

# map_3d.setMapSettings(configured_3d_settings)
map_3d.attemptResize(QgsLayoutSize(200, 150, QgsUnitTypes.LayoutMillimeters))
map_3d.attemptMove(QgsLayoutPoint(10, 10, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(map_3d)
```

---

## Common Operations

### Altitude Clamping Modes

Control how features relate to terrain elevation:

```python
from qgis.core import Qgis

# Absolute — Z values are absolute heights (ignore terrain)
symbol.setAltitudeClamping(Qgis.AltitudeClamping.Absolute)

# Relative — Z values are added to terrain elevation
symbol.setAltitudeClamping(Qgis.AltitudeClamping.Relative)

# Terrain — features are draped onto terrain (Z values ignored)
symbol.setAltitudeClamping(Qgis.AltitudeClamping.Terrain)
```

### Altitude Binding (Polygons/Lines)

```python
# Vertex — each vertex clamped individually (follows terrain closely)
symbol.setAltitudeBinding(Qgis.AltitudeBinding.Vertex)

# Centroid — entire feature clamped by centroid height (flat placement)
symbol.setAltitudeBinding(Qgis.AltitudeBinding.Centroid)
```

### Coordinate Conversion

```python
from qgis.core import QgsVector3D

# Map CRS coordinates to 3D world coordinates (applies x, -z, y swap)
world = settings.mapToWorldCoordinates(QgsVector3D(x_map, y_map, z_map))

# 3D world coordinates back to map CRS
map_coords = settings.worldToMapCoordinates(QgsVector3D(x_world, y_world, z_world))
```

### Serialize / Restore 3D Settings

```python
from qgis.PyQt.QtXml import QDomDocument
from qgis.core import QgsReadWriteContext, QgsProject

# Save
doc = QDomDocument()
elem = settings.writeXml(doc, QgsReadWriteContext())
doc.appendChild(elem)

# Restore
settings2 = Qgs3DMapSettings()
settings2.readXml(elem, QgsReadWriteContext())
settings2.resolveReferences(QgsProject.instance())
```

### Point Cloud 3D Rendering

```python
from qgis._3d import (
    QgsPointCloudLayer3DRenderer,
    QgsRgbPointCloud3DSymbol,
    QgsSingleColorPointCloud3DSymbol,
    QgsClassificationPointCloud3DSymbol,
    QgsColorRampPointCloud3DSymbol,
)

# RGB point cloud
symbol = QgsRgbPointCloud3DSymbol()
renderer = QgsPointCloudLayer3DRenderer()
renderer.setSymbol(symbol)
point_cloud_layer.setRenderer3D(renderer)
```

### PBR Material (QGIS 3.36+)

```python
from qgis._3d import QgsMetalRoughMaterialSettings
from qgis.PyQt.QtGui import QColor

material = QgsMetalRoughMaterialSettings()
material.setBaseColor(QColor(200, 200, 200))
material.setMetalness(0.8)   # 0.0 = dielectric, 1.0 = metal
material.setRoughness(0.3)   # 0.0 = mirror, 1.0 = matte
```

### Gooch Material (QGIS 3.16+)

```python
from qgis._3d import QgsGoochMaterialSettings
from qgis.PyQt.QtGui import QColor

material = QgsGoochMaterialSettings()
material.setWarm(QColor(255, 200, 100))
material.setCool(QColor(100, 150, 255))
material.setDiffuse(QColor(200, 200, 200))
material.setSpecular(QColor(255, 255, 255))
material.setShininess(50.0)
material.setAlpha(0.25)
material.setBeta(0.5)
```

---

## Version Compatibility

| Feature | Minimum QGIS Version |
|---------|---------------------|
| Basic 3D (symbols, Phong material) | 3.0 |
| QgsLayoutItem3DMap | 3.4 |
| Camera field of view | 3.8 |
| Directional lights, Gooch material | 3.16 |
| Shadows (QgsShadowSettings) | 3.16 |
| Eye Dome Lighting, camera navigation mode | 3.18 |
| Light sources list API | 3.26 |
| Material opacity | 3.26 |
| PBR material (MetalRough) | 3.36 |
| Material coefficients (ambient/diffuse/specular) | 3.36 |
| Terrain settings API (setTerrainSettings) | 3.42 |

---

## Reference Links

- [references/methods.md](references/methods.md) -- Complete API signatures for all 3D classes
- [references/examples.md](references/examples.md) -- Full working code examples
- [references/anti-patterns.md](references/anti-patterns.md) -- Common mistakes and how to avoid them

### Official Sources

- https://qgis.org/pyqgis/3.40/_3d/index.html
- https://qgis.org/pyqgis/master/_3d/Qgs3DMapSettings.html
- https://qgis.org/pyqgis/master/_3d/QgsPolygon3DSymbol.html
- https://qgis.org/pyqgis/master/_3d/QgsVectorLayer3DRenderer.html
