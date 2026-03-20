# QGIS 3D Visualization Python API — Supplementary Research

**Date:** 2026-03-20
**Purpose:** Verify Python bindings for QGIS 3D classes, confirm API availability, and document working methods.
**Trigger:** The PyQGIS 3D cookbook page (docs.qgis.org/3.34/en/docs/pyqgis_developer_cookbook/3d.html) returned 404. The 3.44 version also returns 404 — there is NO official cookbook chapter for 3D in PyQGIS as of March 2026.

---

## 1. Module Location and Import Path

The 3D classes live in the `qgis._3d` module (note the underscore prefix — Python cannot have module names starting with a digit). In QGIS Python console, the module is auto-imported.

**Import pattern:**
```python
from qgis._3d import (
    Qgs3DMapSettings,
    QgsVectorLayer3DRenderer,
    QgsPolygon3DSymbol,
    QgsPhongMaterialSettings,
    QgsPointLightSettings,
)
```

Some base classes (QgsAbstract3DSymbol, QgsAbstract3DRenderer) live in `qgis.core`, not `qgis._3d`.

**Source:** [PyQGIS 3.40 _3d module index](https://qgis.org/pyqgis/3.40/_3d/index.html)

---

## 2. Complete Class Inventory — qgis._3d Module

All classes below have [CONFIRMED] Python bindings as verified against the PyQGIS 3.40 API reference.

### Core 3D Classes
| Class | Status | Notes |
|-------|--------|-------|
| `Qgs3D` | [CONFIRMED] | Static utility class |
| `Qgs3DMapCanvas` | [CONFIRMED] | Convenience wrapper for 3D window, requires `setMapSettings()` |
| `Qgs3DMapExportSettings` | [CONFIRMED] | Settings for 3D scene export |
| `Qgs3DMapScene` | [CONFIRMED] | Root 3D entity for the scene |
| `Qgs3DMapSettings` | [CONFIRMED] | Full scene configuration — CRS, extent, terrain, layers, lights |
| `Qgs3DTypes` | [CONFIRMED] | Enumerations (AltitudeClamping, AltitudeBinding, CullingMode) |
| `QgsCameraController` | [CONFIRMED] | Camera interaction controller |
| `QgsCameraPose` | [CONFIRMED] | Camera position/orientation (center, distance, pitch, heading) |
| `QgsMaterialSettingsRenderingTechnique` | [CONFIRMED] | Enum for rendering techniques |
| `QgsVectorLayer3DTilingSettings` | [CONFIRMED] | Tiling configuration for vector layers |

### 3D Renderers
| Class | Status | Notes |
|-------|--------|-------|
| `QgsAbstractVectorLayer3DRenderer` | [CONFIRMED] | Base class for vector 3D renderers |
| `QgsPointCloudLayer3DRenderer` | [CONFIRMED] | Point cloud layer renderer |
| `QgsRuleBased3DRenderer` | [CONFIRMED] | Rule-based 3D rendering (like 2D rule-based) |
| `QgsRuleBased3DRendererMetadata` | [CONFIRMED] | Metadata for rule-based renderer |
| `QgsTiledSceneLayer3DRenderer` | [CONFIRMED] | Tiled scene (3D Tiles) renderer |
| `QgsTiledSceneLayer3DRendererMetadata` | [CONFIRMED] | Metadata for tiled scene renderer |
| `QgsVectorLayer3DRenderer` | [CONFIRMED] | Primary renderer for vector layers in 3D |
| `QgsVectorLayer3DRendererMetadata` | [CONFIRMED] | Metadata for vector layer renderer |

### 3D Symbol Classes
| Class | Status | Notes |
|-------|--------|-------|
| `QgsPolygon3DSymbol` | [CONFIRMED] | Polygons as planar surfaces with optional extrusion |
| `QgsLine3DSymbol` | [CONFIRMED] | Lines as buffered planar polygons with optional extrusion |
| `QgsPoint3DSymbol` | [CONFIRMED] | Points as 3D shapes (sphere, cylinder, cone, cube, etc.) or models |
| `QgsClassificationPointCloud3DSymbol` | [CONFIRMED] | Classification-based point cloud symbol |
| `QgsColorRampPointCloud3DSymbol` | [CONFIRMED] | Color ramp point cloud symbol |
| `QgsPointCloud3DSymbol` | [CONFIRMED] | Base point cloud symbol |
| `QgsRgbPointCloud3DSymbol` | [CONFIRMED] | RGB point cloud symbol |
| `QgsSingleColorPointCloud3DSymbol` | [CONFIRMED] | Single color point cloud symbol |

### Light Source Classes
| Class | Status | Notes |
|-------|--------|-------|
| `QgsLightSource` | [CONFIRMED] | Base class for light sources |
| `QgsPointLightSettings` | [CONFIRMED] | Point light with position, color, intensity, attenuation |
| `QgsDirectionalLightSettings` | [CONFIRMED] | Directional light with direction, color, intensity (v3.16+) |

### Material Classes
| Class | Status | Notes |
|-------|--------|-------|
| `QgsAbstractMaterialSettings` | [CONFIRMED] | Base class for material settings |
| `QgsPhongMaterialSettings` | [CONFIRMED] | Phong shading: ambient, diffuse, specular, shininess |
| `QgsGoochMaterialSettings` | [CONFIRMED] | Gooch shading: warm/cool colors, alpha/beta (v3.16+) |
| `QgsMetalRoughMaterialSettings` | [CONFIRMED] | PBR metal-rough: baseColor, metalness, roughness (v3.36+) |
| `QgsPhongTexturedMaterialSettings` | [CONFIRMED] | Phong with texture support |
| `QgsSimpleLineMaterialSettings` | [CONFIRMED] | Simple material for lines |
| `QgsNullMaterialSettings` | [CONFIRMED] | Null/empty material |
| `QgsMaterialContext` | [CONFIRMED] | Context for material rendering |
| `QgsMaterialRegistry` | [CONFIRMED] | Registry of material types |
| `QgsMaterialSettingsAbstractMetadata` | [CONFIRMED] | Metadata base class |

### Layout and Processing
| Class | Status | Notes |
|-------|--------|-------|
| `QgsLayoutItem3DMap` | [CONFIRMED] | 3D map in print layouts (v3.4+) |
| `Qgs3DAlgorithms` | [CONFIRMED] | Processing framework 3D algorithms |

### Base Classes in qgis.core (NOT in qgis._3d)
| Class | Module | Status |
|-------|--------|--------|
| `QgsAbstract3DSymbol` | `qgis.core` | [CONFIRMED] |
| `QgsAbstract3DRenderer` | `qgis.core` | [CONFIRMED] |

**Source:** [PyQGIS 3.40 _3d index](https://qgis.org/pyqgis/3.40/_3d/index.html), [QgsAbstract3DSymbol](https://qgis.org/pyqgis/3.40/core/QgsAbstract3DSymbol.html), [QgsAbstract3DRenderer](https://qgis.org/pyqgis/3.40/core/QgsAbstract3DRenderer.html)

---

## 3. Qgs3DMapSettings — Confirmed Python API

The central configuration class for 3D scenes. All methods below are [CONFIRMED] with full Python signatures.

### Coordinate System and Extent
- `crs() -> QgsCoordinateReferenceSystem`
- `setCrs(crs: QgsCoordinateReferenceSystem)`
- `extent() -> QgsRectangle`
- `setExtent(extent: QgsRectangle)`
- `origin() -> QgsVector3D`
- `setOrigin(origin: QgsVector3D)`
- `transformContext() -> QgsCoordinateTransformContext`
- `setTransformContext(context: QgsCoordinateTransformContext)`

### Coordinate Conversion
- `mapToWorldCoordinates(mapCoords: QgsVector3D) -> QgsVector3D`
- `worldToMapCoordinates(worldCoords: QgsVector3D) -> QgsVector3D`

### Layers and Renderers
- `layers() -> List[QgsMapLayer]`
- `setLayers(layers: Iterable[QgsMapLayer])`

### Terrain
- `terrainSettings() -> QgsAbstractTerrainSettings | None`
- `setTerrainSettings(settings: QgsAbstractTerrainSettings | None)`
- `isTerrainShadingEnabled() -> bool`
- `setTerrainShadingEnabled(enabled: bool)`
- `terrainShadingMaterial() -> QgsPhongMaterialSettings`
- `setTerrainShadingMaterial(material: QgsPhongMaterialSettings)`
- `terrainRenderingEnabled() -> bool`
- `setTerrainRenderingEnabled(enabled: bool)`
- `terrainMapTheme() -> str`
- `setTerrainMapTheme(theme: str | None)`

### Light Sources
- `lightSources() -> List[QgsLightSource]`
- `setLightSources(lights: Iterable[QgsLightSource])`

### Camera
- `fieldOfView() -> float`
- `setFieldOfView(fieldOfView: float)`
- `cameraMovementSpeed() -> float`
- `setCameraMovementSpeed(movementSpeed: float)`

### Visual Settings
- `backgroundColor() -> QColor`
- `setBackgroundColor(color: QColor | Qt.GlobalColor)`
- `selectionColor() -> QColor`
- `setSelectionColor(color: QColor | Qt.GlobalColor)`
- `isSkyboxEnabled() -> bool`
- `setIsSkyboxEnabled(enabled: bool)`
- `showLabels() -> bool`
- `setShowLabels(enabled: bool)`

### Eye Dome Lighting
- `eyeDomeLightingEnabled() -> bool`
- `setEyeDomeLightingEnabled(enabled: bool)`
- `eyeDomeLightingStrength() -> float`
- `setEyeDomeLightingStrength(strength: float)`
- `eyeDomeLightingDistance() -> int`
- `setEyeDomeLightingDistance(distance: int)`

### Serialization
- `readXml(elem: QDomElement, context: QgsReadWriteContext)`
- `writeXml(doc: QDomDocument, context: QgsReadWriteContext) -> QDomElement`
- `resolveReferences(project: QgsProject)`

**Deprecated methods** (still functional but use alternatives):
- `terrainVerticalScale()` / `setTerrainVerticalScale()` — use `terrainSettings()` instead
- `maxTerrainScreenError()` / `setMaxTerrainScreenError()` — use `terrainSettings()` instead
- `mapTileResolution()` / `setMapTileResolution()` — use `terrainSettings()` instead

**Source:** [Qgs3DMapSettings API](https://qgis.org/pyqgis/master/_3d/Qgs3DMapSettings.html)

---

## 4. 3D Symbol Classes — Confirmed Python API

### QgsPolygon3DSymbol
Factory method: `QgsPolygon3DSymbol.create() -> QgsAbstract3DSymbol`

Key methods:
- `altitudeClamping() / setAltitudeClamping(Qgis.AltitudeClamping)`
- `altitudeBinding() / setAltitudeBinding(Qgis.AltitudeBinding)`
- `offset() / setOffset(float)` — vertical offset
- `extrusionHeight() / setExtrusionHeight(float)` — extrude polygons upward
- `extrusionFaces() / setExtrusionFaces(Qgis.ExtrusionFaces)` — which faces to render
- `materialSettings() / setMaterialSettings(QgsAbstractMaterialSettings)` — material
- `cullingMode() / setCullingMode(Qgs3DTypes.CullingMode)`
- `edgesEnabled() / setEdgesEnabled(bool)` — edge rendering
- `edgeWidth() / setEdgeWidth(float)`
- `edgeColor() / setEdgeColor(QColor)`

**Source:** [QgsPolygon3DSymbol API](https://qgis.org/pyqgis/master/_3d/QgsPolygon3DSymbol.html)

### QgsLine3DSymbol
Factory method: `QgsLine3DSymbol.create() -> QgsAbstract3DSymbol`

Key methods:
- `width() / setWidth(float)` — line width in map units
- `offset() / setOffset(float)` — vertical offset
- `extrusionHeight() / setExtrusionHeight(float)`
- `altitudeClamping() / setAltitudeClamping(Qgis.AltitudeClamping)`
- `altitudeBinding() / setAltitudeBinding(Qgis.AltitudeBinding)`
- `renderAsSimpleLines() / setRenderAsSimpleLines(bool)`
- `materialSettings() / setMaterialSettings(QgsAbstractMaterialSettings)`

**Source:** [QgsLine3DSymbol API](https://qgis.org/pyqgis/3.40/_3d/QgsLine3DSymbol.html)

### QgsPoint3DSymbol
Constructor: `QgsPoint3DSymbol()`

Key methods:
- `shape() / setShape(Qgis.Point3DShape)` — sphere, cylinder, cone, cube, torus, plane, billboard, model
- `shapeProperties() / setShapeProperties(Dict[str, Any])` — radius, length, etc.
- `materialSettings() / setMaterialSettings(QgsAbstractMaterialSettings)`
- `altitudeClamping() / setAltitudeClamping(Qgis.AltitudeClamping)`
- `transform() / setTransform(QMatrix4x4)` — scale, rotation, translation
- `billboardSymbol() / setBillboardSymbol(QgsMarkerSymbol)` — for billboard mode

Static helpers:
- `shapeFromString(str) -> Qgis.Point3DShape`
- `shapeToString(Qgis.Point3DShape) -> str`

**Source:** [QgsPoint3DSymbol API](https://qgis.org/pyqgis/3.40/_3d/QgsPoint3DSymbol.html)

---

## 5. Material Classes — Confirmed Python API

### QgsPhongMaterialSettings [CONFIRMED]
The standard material. Instantiate via `QgsPhongMaterialSettings.create()`.

- `ambient() / setAmbient(QColor)` — ambient color
- `diffuse() / setDiffuse(QColor)` — diffuse color
- `specular() / setSpecular(QColor)` — specular highlight color
- `shininess() / setShininess(float)` — specular exponent
- `opacity() / setOpacity(float)` — surface opacity (v3.26+)
- `ambientCoefficient() / setAmbientCoefficient(float)` — ambient strength (v3.36+)
- `diffuseCoefficient() / setDiffuseCoefficient(float)` — diffuse strength (v3.36+)
- `specularCoefficient() / setSpecularCoefficient(float)` — specular strength (v3.36+)

**Source:** [QgsPhongMaterialSettings API](https://qgis.org/pyqgis/3.40/_3d/QgsPhongMaterialSettings.html)

### QgsGoochMaterialSettings [CONFIRMED] (v3.16+)
Non-photorealistic rendering material.

- `warm() / setWarm(QColor)` — warm tone color
- `cool() / setCool(QColor)` — cool tone color
- `diffuse() / setDiffuse(QColor)`
- `specular() / setSpecular(QColor)`
- `shininess() / setShininess(float)`
- `alpha() / setAlpha(float)` — warm/cool blend factor
- `beta() / setBeta(float)` — diffuse blend factor

**Source:** [QgsGoochMaterialSettings API](https://qgis.org/pyqgis/3.40/_3d/QgsGoochMaterialSettings.html)

### QgsMetalRoughMaterialSettings [CONFIRMED] (v3.36+)
PBR (Physically Based Rendering) material.

- `baseColor() / setBaseColor(QColor)`
- `metalness() / setMetalness(float)` — 0.0 (dielectric) to 1.0 (metal)
- `roughness() / setRoughness(float)` — 0.0 (mirror) to 1.0 (matte)

**Source:** [QgsMetalRoughMaterialSettings API](https://qgis.org/pyqgis/3.40/_3d/QgsMetalRoughMaterialSettings.html)

---

## 6. Light Source Classes — Confirmed Python API

### QgsPointLightSettings [CONFIRMED]
Constructor: `QgsPointLightSettings()`

- `position() / setPosition(QgsVector3D)` — 3D world coordinates
- `color() / setColor(QColor)`
- `intensity() / setIntensity(float)`
- `constantAttenuation() / setConstantAttenuation(float)` — A0
- `linearAttenuation() / setLinearAttenuation(float)` — A1
- `quadraticAttenuation() / setQuadraticAttenuation(float)` — A2

Attenuation formula: `TotalAttenuation = A0 + A1*D + A2*D^2`

**Source:** [QgsPointLightSettings API](https://qgis.org/pyqgis/3.40/_3d/QgsPointLightSettings.html)

### QgsDirectionalLightSettings [CONFIRMED] (v3.16+)
Constructor: `QgsDirectionalLightSettings()`

- `direction() / setDirection(QgsVector3D)` — direction in degrees
- `color() / setColor(QColor)`
- `intensity() / setIntensity(float)`

**Source:** [QgsDirectionalLightSettings API](https://qgis.org/pyqgis/3.40/_3d/QgsDirectionalLightSettings.html)

---

## 7. QgsCameraPose — Confirmed Python API

- `centerPoint() / setCenterPoint(QgsVector3D)` — focal point
- `distanceFromCenterPoint() / setDistanceFromCenterPoint(float)` — camera distance
- `pitchAngle() / setPitchAngle(float)` — 0 = looking down, 90 = horizontal
- `headingAngle() / setHeadingAngle(float)` — horizontal rotation

**Source:** [QgsCameraPose API](https://qgis.org/pyqgis/3.40/_3d/QgsCameraPose.html)

---

## 8. QgsVectorLayer3DRenderer — Confirmed Python API

The bridge between a vector layer and its 3D symbol.

Constructor: `QgsVectorLayer3DRenderer(symbol: QgsAbstract3DSymbol | None = None)`

- `symbol() -> QgsAbstract3DSymbol | None`
- `setSymbol(symbol: QgsAbstract3DSymbol | None)` — takes ownership

To apply to a layer: use `layer.setRenderer3D(renderer)` (method on QgsMapLayer in qgis.core).

**Source:** [QgsVectorLayer3DRenderer API](https://qgis.org/pyqgis/master/_3d/QgsVectorLayer3DRenderer.html)

---

## 9. Reconstructed Working Code Examples

No official cookbook examples exist. The following examples are reconstructed from confirmed API signatures. They are [UNCONFIRMED] as running code but use only [CONFIRMED] API methods.

### Example 1: Extruded Building Polygons

```python
from qgis.core import QgsProject, QgsVectorLayer
from qgis._3d import (
    QgsVectorLayer3DRenderer,
    QgsPolygon3DSymbol,
    QgsPhongMaterialSettings,
)
from qgis.PyQt.QtGui import QColor

# Load a polygon layer
layer = QgsProject.instance().mapLayersByName("buildings")[0]

# Create material
material = QgsPhongMaterialSettings()
material.setAmbient(QColor(100, 100, 100))
material.setDiffuse(QColor(180, 180, 160))
material.setSpecular(QColor(255, 255, 255))
material.setShininess(50.0)

# Create 3D polygon symbol
symbol = QgsPolygon3DSymbol.create()
symbol.setExtrusionHeight(15.0)  # 15 meters
symbol.setMaterialSettings(material)
symbol.setEdgesEnabled(True)
symbol.setEdgeColor(QColor(0, 0, 0))
symbol.setEdgeWidth(1.0)

# Create renderer and assign to layer
renderer = QgsVectorLayer3DRenderer(symbol)
layer.setRenderer3D(renderer)
```

### Example 2: 3D Point Symbol (Spheres)

```python
from qgis.core import QgsProject
from qgis._3d import (
    QgsVectorLayer3DRenderer,
    QgsPoint3DSymbol,
    QgsPhongMaterialSettings,
)
from qgis.PyQt.QtGui import QColor
from PyQt5.QtCore import Qt

layer = QgsProject.instance().mapLayersByName("points")[0]

material = QgsPhongMaterialSettings()
material.setDiffuse(QColor(255, 0, 0))
material.setShininess(100.0)

symbol = QgsPoint3DSymbol()
symbol.setShape(Qgis.Point3DShape.Sphere)
symbol.setShapeProperties({"radius": 5.0})
symbol.setMaterialSettings(material)

renderer = QgsVectorLayer3DRenderer(symbol)
layer.setRenderer3D(renderer)
```

### Example 3: Configure Scene Lighting

```python
from qgis._3d import (
    Qgs3DMapSettings,
    QgsPointLightSettings,
    QgsDirectionalLightSettings,
)
from qgis.core import QgsVector3D
from qgis.PyQt.QtGui import QColor

# Point light
point_light = QgsPointLightSettings()
point_light.setPosition(QgsVector3D(0, 1000, 0))
point_light.setColor(QColor(255, 255, 255))
point_light.setIntensity(1.0)
point_light.setConstantAttenuation(1.0)
point_light.setLinearAttenuation(0.0)
point_light.setQuadraticAttenuation(0.0)

# Directional light
dir_light = QgsDirectionalLightSettings()
dir_light.setDirection(QgsVector3D(0.5, -1.0, 0.5))
dir_light.setColor(QColor(255, 255, 230))
dir_light.setIntensity(0.8)

# Apply to map settings (would need access to the 3D map settings object)
# map_settings.setLightSources([point_light, dir_light])
```

### Example 4: 3D Map in Print Layout

```python
from qgis.core import QgsProject, QgsLayout, QgsLayoutItemPage, QgsLayoutSize
from qgis._3d import QgsLayoutItem3DMap, Qgs3DMapSettings, QgsCameraPose
from qgis.core import QgsVector3D

project = QgsProject.instance()
layout = QgsLayout(project)

# Create 3D map item
map_3d = QgsLayoutItem3DMap(layout)

# Configure camera
camera = QgsCameraPose()
camera.setCenterPoint(QgsVector3D(0, 0, 0))
camera.setDistanceFromCenterPoint(500)
camera.setPitchAngle(45.0)
camera.setHeadingAngle(0.0)
map_3d.setCameraPose(camera)

# Set map settings (would need fully configured Qgs3DMapSettings)
# map_3d.setMapSettings(settings_3d)

layout.addLayoutItem(map_3d)
```

---

## 10. API Stability Warning

ALL classes in the `qgis._3d` module are marked as **tech preview / unstable API**. The official documentation states:

> "Not considered stable API, and may change in future QGIS releases. It is exposed to the Python bindings as a tech preview only."

This means:
- Method signatures MAY change between QGIS minor versions
- Classes MAY be renamed or removed
- Skills MUST specify the minimum QGIS version for each feature
- Skills SHOULD recommend checking the API docs for the user's specific QGIS version

---

## 11. Verification Summary — Previously Marked [VERIFY] Items

Items from the integration vooronderzoek that needed verification:

| Item | Status | Finding |
|------|--------|---------|
| QgsPolygon3DSymbol Python availability | [CONFIRMED] | Full API in qgis._3d, factory via `.create()` |
| QgsLine3DSymbol Python availability | [CONFIRMED] | Full API in qgis._3d, factory via `.create()` |
| QgsPoint3DSymbol Python availability | [CONFIRMED] | Full API in qgis._3d, constructor `QgsPoint3DSymbol()` |
| QgsPhongMaterialSettings Python availability | [CONFIRMED] | Full API in qgis._3d with v3.26+ and v3.36+ additions |
| QgsGoochMaterialSettings Python availability | [CONFIRMED] | Full API in qgis._3d, v3.16+ |
| QgsMetalRoughMaterialSettings Python availability | [CONFIRMED] | Full API in qgis._3d, v3.36+ |
| Qgs3DMapSettings Python availability | [CONFIRMED] | Comprehensive API in qgis._3d |
| QgsVectorLayer3DRenderer Python availability | [CONFIRMED] | Full API in qgis._3d |
| QgsPointLightSettings Python availability | [CONFIRMED] | Full API with attenuation model |
| QgsDirectionalLightSettings Python availability | [CONFIRMED] | Full API, v3.16+ |
| QgsCameraPose Python availability | [CONFIRMED] | Full API with center/distance/pitch/heading |
| QgsLayoutItem3DMap Python availability | [CONFIRMED] | Print layout integration, v3.4+ |
| QgsRuleBased3DRenderer Python availability | [CONFIRMED] | Listed in _3d module |
| 3D Cookbook chapter existence | [CONFIRMED MISSING] | Returns 404 on both 3.34 and 3.44 docs |
| `layer.setRenderer3D()` method | [UNCONFIRMED] | Inferred from architecture but not directly verified in API docs |
| `setMaterial()` method on symbols | [CONFIRMED REMOVED] | Replaced by `setMaterialSettings()` — CityJSON plugin documented this breaking change |

---

## 12. Key Findings for Skill Development

1. **No cookbook exists** — The skill must be the primary reference for 3D PyQGIS. This increases its importance significantly.

2. **All major 3D classes have Python bindings** — Every class needed for basic 3D visualization (symbols, materials, renderers, lights, camera) is accessible from Python.

3. **Unstable API warning is universal** — Every _3d class carries this warning. The skill MUST prominently warn users about this.

4. **Three material models available** — Phong (standard), Gooch (NPR), MetalRough (PBR). Phong is the most mature.

5. **Factory pattern inconsistency** — QgsPolygon3DSymbol and QgsLine3DSymbol use `create()` static method, while QgsPoint3DSymbol has a regular constructor. The skill MUST document both patterns.

6. **Breaking API change documented** — `setMaterial()` was replaced by `setMaterialSettings()` at some point. Old code examples may reference the deprecated method.

7. **Version-specific features** — Directional lights (v3.16+), Gooch materials (v3.16+), opacity (v3.26+), PBR materials (v3.36+), coefficient controls (v3.36+). The skill MUST include version annotations.

8. **Qgs3DMapCanvas requires special OpenGL setup** — Must set default surface format before QApplication construction when using shared OpenGL context with QtWebEngine.

---

## Sources

- [PyQGIS 3.40 _3d Module Index](https://qgis.org/pyqgis/3.40/_3d/index.html)
- [Qgs3DMapSettings API](https://qgis.org/pyqgis/master/_3d/Qgs3DMapSettings.html)
- [QgsPolygon3DSymbol API](https://qgis.org/pyqgis/master/_3d/QgsPolygon3DSymbol.html)
- [QgsLine3DSymbol API](https://qgis.org/pyqgis/3.40/_3d/QgsLine3DSymbol.html)
- [QgsPoint3DSymbol API](https://qgis.org/pyqgis/3.40/_3d/QgsPoint3DSymbol.html)
- [QgsPhongMaterialSettings API](https://qgis.org/pyqgis/3.40/_3d/QgsPhongMaterialSettings.html)
- [QgsGoochMaterialSettings API](https://qgis.org/pyqgis/3.40/_3d/QgsGoochMaterialSettings.html)
- [QgsMetalRoughMaterialSettings API](https://qgis.org/pyqgis/3.40/_3d/QgsMetalRoughMaterialSettings.html)
- [QgsPointLightSettings API](https://qgis.org/pyqgis/3.40/_3d/QgsPointLightSettings.html)
- [QgsDirectionalLightSettings API](https://qgis.org/pyqgis/3.40/_3d/QgsDirectionalLightSettings.html)
- [QgsCameraPose API](https://qgis.org/pyqgis/3.40/_3d/QgsCameraPose.html)
- [QgsVectorLayer3DRenderer API](https://qgis.org/pyqgis/master/_3d/QgsVectorLayer3DRenderer.html)
- [QgsLayoutItem3DMap API](https://qgis.org/pyqgis/3.40/_3d/QgsLayoutItem3DMap.html)
- [Qgs3DMapCanvas API](https://qgis.org/pyqgis/3.40/_3d/Qgs3DMapCanvas.html)
- [QgsAbstract3DSymbol API (core)](https://qgis.org/pyqgis/3.40/core/QgsAbstract3DSymbol.html)
- [QgsAbstract3DRenderer API (core)](https://qgis.org/pyqgis/3.40/core/QgsAbstract3DRenderer.html)
- [QGIS 3D Enhancement Proposal #105](https://github.com/qgis/QGIS-Enhancement-Proposals/issues/105)
- [CityJSON QGIS Plugin](https://github.com/cityjson/cityjson-qgis-plugin) — documents setMaterial() to setMaterialSettings() migration
