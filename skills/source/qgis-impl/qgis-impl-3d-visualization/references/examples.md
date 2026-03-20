# qgis-impl-3d-visualization — Examples

> All examples use [CONFIRMED] API methods from PyQGIS 3.40+. No official 3D cookbook exists,
> so these are reconstructed from verified API signatures.

---

## Example 1: Complete 3D Scene with Extruded Buildings

```python
from qgis.core import (
    QgsProject,
    QgsCoordinateReferenceSystem,
    QgsRectangle,
    QgsVector3D,
)
from qgis._3d import (
    Qgs3DMapSettings,
    QgsFlatTerrainSettings,
    QgsVectorLayer3DRenderer,
    QgsPolygon3DSymbol,
    QgsPhongMaterialSettings,
    QgsPointLightSettings,
    QgsDirectionalLightSettings,
)
from qgis.PyQt.QtGui import QColor

project = QgsProject.instance()
buildings = project.mapLayersByName("buildings")[0]

# --- Scene setup ---
settings = Qgs3DMapSettings()
settings.setCrs(QgsCoordinateReferenceSystem("EPSG:28992"))
settings.setExtent(buildings.extent())
settings.setBackgroundColor(QColor(200, 220, 240))

# Flat terrain
terrain = QgsFlatTerrainSettings()
terrain.setElevation(0.0)
settings.setTerrainSettings(terrain)

# --- Building material ---
material = QgsPhongMaterialSettings()
material.setAmbient(QColor(80, 80, 80))
material.setDiffuse(QColor(200, 190, 170))
material.setSpecular(QColor(255, 255, 255))
material.setShininess(80.0)

# --- 3D building symbol ---
symbol = QgsPolygon3DSymbol.create()
symbol.setExtrusionHeight(12.0)
symbol.setMaterialSettings(material)
symbol.setEdgesEnabled(True)
symbol.setEdgeColor(QColor(60, 60, 60))
symbol.setEdgeWidth(1.0)

# --- Assign renderer ---
renderer = QgsVectorLayer3DRenderer(symbol)
buildings.setRenderer3D(renderer)

# --- Lighting ---
sun = QgsDirectionalLightSettings()
sun.setDirection(QgsVector3D(0.5, -1.0, 0.5))
sun.setColor(QColor(255, 255, 230))
sun.setIntensity(0.8)

fill = QgsPointLightSettings()
fill.setPosition(QgsVector3D(0, 500, 0))
fill.setColor(QColor(200, 200, 255))
fill.setIntensity(0.3)
fill.setConstantAttenuation(1.0)
fill.setLinearAttenuation(0.0)
fill.setQuadraticAttenuation(0.0)

settings.setLightSources([sun, fill])
settings.setLayers([buildings])
```

---

## Example 2: DEM Terrain with Draped Roads

```python
from qgis.core import QgsProject, QgsCoordinateReferenceSystem
from qgis._3d import (
    Qgs3DMapSettings,
    QgsDemTerrainSettings,
    QgsVectorLayer3DRenderer,
    QgsLine3DSymbol,
    QgsPhongMaterialSettings,
)
from qgis.PyQt.QtGui import QColor
from qgis.core import Qgis

project = QgsProject.instance()
dem = project.mapLayersByName("elevation")[0]
roads = project.mapLayersByName("roads")[0]

# --- Scene ---
settings = Qgs3DMapSettings()
settings.setCrs(dem.crs())
settings.setExtent(dem.extent())

# --- DEM terrain ---
terrain = QgsDemTerrainSettings()
terrain.setLayer(dem)
terrain.setResolution(16)
terrain.setSkirtHeight(10.0)
settings.setTerrainSettings(terrain)
settings.setTerrainRenderingEnabled(True)
settings.setTerrainShadingEnabled(True)

terrain_material = QgsPhongMaterialSettings()
terrain_material.setDiffuse(QColor(160, 140, 100))
settings.setTerrainShadingMaterial(terrain_material)

# --- Roads draped on terrain ---
road_material = QgsPhongMaterialSettings()
road_material.setDiffuse(QColor(80, 80, 80))

road_symbol = QgsLine3DSymbol.create()
road_symbol.setWidth(5.0)
road_symbol.setAltitudeClamping(Qgis.AltitudeClamping.Terrain)
road_symbol.setMaterialSettings(road_material)

road_renderer = QgsVectorLayer3DRenderer(road_symbol)
roads.setRenderer3D(road_renderer)

settings.setLayers([dem, roads])
```

---

## Example 3: Point Cloud with Eye Dome Lighting

```python
from qgis.core import QgsProject
from qgis._3d import (
    Qgs3DMapSettings,
    QgsFlatTerrainSettings,
    QgsPointCloudLayer3DRenderer,
    QgsRgbPointCloud3DSymbol,
)
from qgis.PyQt.QtGui import QColor

project = QgsProject.instance()
pc_layer = project.mapLayersByName("point_cloud")[0]

# --- Scene ---
settings = Qgs3DMapSettings()
settings.setCrs(pc_layer.crs())
settings.setExtent(pc_layer.extent())
settings.setBackgroundColor(QColor(30, 30, 30))

terrain = QgsFlatTerrainSettings()
settings.setTerrainSettings(terrain)

# --- Point cloud renderer ---
symbol = QgsRgbPointCloud3DSymbol()
renderer = QgsPointCloudLayer3DRenderer()
renderer.setSymbol(symbol)
pc_layer.setRenderer3D(renderer)

# --- Eye Dome Lighting for depth perception ---
settings.setEyeDomeLightingEnabled(True)
settings.setEyeDomeLightingStrength(1000.0)
settings.setEyeDomeLightingDistance(1)

settings.setLayers([pc_layer])
```

---

## Example 4: 3D Points as Colored Spheres

```python
from qgis.core import QgsProject, Qgis
from qgis._3d import (
    QgsVectorLayer3DRenderer,
    QgsPoint3DSymbol,
    QgsPhongMaterialSettings,
)
from qgis.PyQt.QtGui import QColor

layer = QgsProject.instance().mapLayersByName("sensors")[0]

material = QgsPhongMaterialSettings()
material.setDiffuse(QColor(255, 50, 50))
material.setSpecular(QColor(255, 255, 255))
material.setShininess(120.0)

symbol = QgsPoint3DSymbol()
symbol.setShape(Qgis.Point3DShape.Sphere)
symbol.setShapeProperties({"radius": 3.0})
symbol.setMaterialSettings(material)
symbol.setAltitudeClamping(Qgis.AltitudeClamping.Relative)

renderer = QgsVectorLayer3DRenderer(symbol)
layer.setRenderer3D(renderer)
```

---

## Example 5: 3D Map in Print Layout

```python
from qgis.core import (
    QgsProject,
    QgsLayout,
    QgsLayoutSize,
    QgsLayoutPoint,
    QgsUnitTypes,
    QgsVector3D,
    QgsCoordinateReferenceSystem,
    QgsRectangle,
)
from qgis._3d import (
    Qgs3DMapSettings,
    QgsFlatTerrainSettings,
    QgsLayoutItem3DMap,
    QgsCameraPose,
)
from qgis.PyQt.QtGui import QColor

project = QgsProject.instance()

# --- Configure 3D settings ---
settings = Qgs3DMapSettings()
settings.setCrs(QgsCoordinateReferenceSystem("EPSG:28992"))
settings.setExtent(QgsRectangle(100000, 400000, 200000, 500000))
settings.setBackgroundColor(QColor(200, 220, 240))

terrain = QgsFlatTerrainSettings()
settings.setTerrainSettings(terrain)
settings.setLayers(list(project.mapLayers().values()))

# --- Create layout ---
layout = QgsLayout(project)

# --- Add 3D map item ---
map_3d = QgsLayoutItem3DMap(layout)

camera = QgsCameraPose()
camera.setCenterPoint(QgsVector3D(150000, 450000, 0))
camera.setDistanceFromCenterPoint(20000)
camera.setPitchAngle(45.0)
camera.setHeadingAngle(315.0)
map_3d.setCameraPose(camera)

map_3d.setMapSettings(settings)
map_3d.attemptResize(QgsLayoutSize(200, 150, QgsUnitTypes.LayoutMillimeters))
map_3d.attemptMove(QgsLayoutPoint(10, 10, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(map_3d)
```

---

## Example 6: PBR Material (QGIS 3.36+)

```python
from qgis._3d import (
    QgsVectorLayer3DRenderer,
    QgsPolygon3DSymbol,
    QgsMetalRoughMaterialSettings,
)
from qgis.PyQt.QtGui import QColor

# Metallic building facade
material = QgsMetalRoughMaterialSettings()
material.setBaseColor(QColor(180, 180, 190))
material.setMetalness(0.9)
material.setRoughness(0.2)

symbol = QgsPolygon3DSymbol.create()
symbol.setExtrusionHeight(30.0)
symbol.setMaterialSettings(material)

renderer = QgsVectorLayer3DRenderer(symbol)
building_layer.setRenderer3D(renderer)
```

---

## Example 7: Serialize and Restore 3D Settings

```python
from qgis.PyQt.QtXml import QDomDocument
from qgis.core import QgsReadWriteContext, QgsProject
from qgis._3d import Qgs3DMapSettings

# --- Save settings to XML ---
doc = QDomDocument()
elem = settings.writeXml(doc, QgsReadWriteContext())
doc.appendChild(elem)
xml_string = doc.toString()

# --- Restore settings from XML ---
doc2 = QDomDocument()
doc2.setContent(xml_string)
root = doc2.documentElement()

restored = Qgs3DMapSettings()
restored.readXml(root, QgsReadWriteContext())
restored.resolveReferences(QgsProject.instance())
```

---

## Example 8: Gooch Non-Photorealistic Material (QGIS 3.16+)

```python
from qgis._3d import (
    QgsVectorLayer3DRenderer,
    QgsPolygon3DSymbol,
    QgsGoochMaterialSettings,
)
from qgis.PyQt.QtGui import QColor

material = QgsGoochMaterialSettings()
material.setWarm(QColor(255, 200, 100))
material.setCool(QColor(100, 150, 255))
material.setDiffuse(QColor(200, 200, 200))
material.setSpecular(QColor(255, 255, 255))
material.setShininess(50.0)
material.setAlpha(0.25)
material.setBeta(0.5)

symbol = QgsPolygon3DSymbol.create()
symbol.setExtrusionHeight(20.0)
symbol.setMaterialSettings(material)

renderer = QgsVectorLayer3DRenderer(symbol)
layer.setRenderer3D(renderer)
```

---

## Example 9: Multiple Light Sources

```python
from qgis._3d import QgsPointLightSettings, QgsDirectionalLightSettings
from qgis.core import QgsVector3D
from qgis.PyQt.QtGui import QColor

# Key light (directional — simulates sun)
key = QgsDirectionalLightSettings()
key.setDirection(QgsVector3D(0.3, -1.0, 0.4))
key.setColor(QColor(255, 250, 230))
key.setIntensity(1.0)

# Fill light (point — softens shadows)
fill = QgsPointLightSettings()
fill.setPosition(QgsVector3D(-500, 300, 200))
fill.setColor(QColor(180, 200, 255))
fill.setIntensity(0.4)
fill.setConstantAttenuation(1.0)
fill.setLinearAttenuation(0.001)
fill.setQuadraticAttenuation(0.0)

# Rim light (point — adds depth)
rim = QgsPointLightSettings()
rim.setPosition(QgsVector3D(0, 200, -500))
rim.setColor(QColor(255, 255, 255))
rim.setIntensity(0.3)
rim.setConstantAttenuation(1.0)
rim.setLinearAttenuation(0.001)
rim.setQuadraticAttenuation(0.0)

settings.setLightSources([key, fill, rim])
```
