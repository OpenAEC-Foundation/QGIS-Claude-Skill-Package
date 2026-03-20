# qgis-impl-3d-visualization — Anti-Patterns

---

## AP-1: Using setMaterial() Instead of setMaterialSettings()

**Wrong:**
```python
symbol = QgsPolygon3DSymbol.create()
material = QgsPhongMaterialSettings()
material.setDiffuse(QColor(200, 100, 50))
symbol.setMaterial(material)  # DEPRECATED / REMOVED — will raise AttributeError
```

**Right:**
```python
symbol = QgsPolygon3DSymbol.create()
material = QgsPhongMaterialSettings()
material.setDiffuse(QColor(200, 100, 50))
symbol.setMaterialSettings(material)  # Correct method name
```

**Why:** `setMaterial()` was replaced by `setMaterialSettings()` in a breaking API change. Old tutorials and plugins (including some CityJSON examples) still reference the old method. ALWAYS use `setMaterialSettings()`.

---

## AP-2: Importing 3D Classes from qgis.core

**Wrong:**
```python
from qgis.core import QgsPolygon3DSymbol, QgsPhongMaterialSettings  # ImportError
```

**Right:**
```python
from qgis._3d import QgsPolygon3DSymbol, QgsPhongMaterialSettings
```

**Why:** All 3D-specific classes live in `qgis._3d`, not `qgis.core`. The underscore prefix exists because Python modules cannot start with a digit. The ONLY exceptions are the abstract base classes `QgsAbstract3DSymbol` and `QgsAbstract3DRenderer`, which live in `qgis.core`.

---

## AP-3: Constructing Polygon/Line Symbols with Regular Constructor

**Wrong:**
```python
symbol = QgsPolygon3DSymbol()  # May not work — factory pattern required
symbol = QgsLine3DSymbol()     # May not work — factory pattern required
```

**Right:**
```python
symbol = QgsPolygon3DSymbol.create()  # Factory method
symbol = QgsLine3DSymbol.create()     # Factory method
```

**Why:** `QgsPolygon3DSymbol` and `QgsLine3DSymbol` use a factory pattern via `create()`. `QgsPoint3DSymbol` is the exception — it uses a regular constructor `QgsPoint3DSymbol()`. Mixing up the construction patterns leads to errors or unexpected behavior.

---

## AP-4: Forgetting setExtent() Before Other Configuration

**Wrong:**
```python
settings = Qgs3DMapSettings()
settings.setCrs(QgsCoordinateReferenceSystem("EPSG:3857"))
settings.setLayers([layer])
# No setExtent() — origin defaults to (0,0), coordinates will be wrong
```

**Right:**
```python
settings = Qgs3DMapSettings()
settings.setCrs(QgsCoordinateReferenceSystem("EPSG:3857"))
settings.setExtent(layer.extent())  # Auto-sets origin to extent center
settings.setLayers([layer])
```

**Why:** `setExtent()` automatically sets the scene origin to the center of the extent. Without it, the origin defaults to (0,0) and all 3D world coordinates will be offset from the data, causing features to render far from the camera or not appear at all.

---

## AP-5: Using Deprecated Terrain Methods on Qgs3DMapSettings

**Wrong:**
```python
settings.setTerrainVerticalScale(2.0)      # Deprecated
settings.setMaxTerrainScreenError(3.0)     # Deprecated
settings.setMapTileResolution(512)         # Deprecated
```

**Right:**
```python
terrain = QgsDemTerrainSettings()
terrain.setLayer(dem_layer)
terrain.setResolution(16)
settings.setTerrainSettings(terrain)
```

**Why:** Since QGIS 3.42, terrain configuration moved to dedicated terrain settings objects. The old methods on `Qgs3DMapSettings` are deprecated and may be removed in future versions. ALWAYS use `setTerrainSettings()` with the appropriate terrain settings class.

---

## AP-6: Ignoring the Unstable API Warning

**Wrong:**
```python
# Assuming 3D API is stable across QGIS versions
# Hardcoding method names without version checks
symbol.setExtrusionFaces(Qgis.ExtrusionFaces.Top)  # Added in a specific version
```

**Right:**
```python
# Document minimum version in comments
# Qgis.ExtrusionFaces requires QGIS 3.x+
symbol.setExtrusionFaces(Qgis.ExtrusionFaces.Top)

# For maximum compatibility, check QGIS version
from qgis.core import Qgis
if Qgis.versionInt() >= 34200:
    settings.setTerrainSettings(terrain)
```

**Why:** ALL classes in `qgis._3d` carry a "tech preview / unstable API" warning. Method signatures, class names, and enum values can change between QGIS minor versions. ALWAYS document the minimum QGIS version for each feature used, and add version checks when targeting multiple QGIS releases.

---

## AP-7: Not Resolving References After readXml()

**Wrong:**
```python
settings2 = Qgs3DMapSettings()
settings2.readXml(elem, QgsReadWriteContext())
# Immediately using settings2 — layer references are unresolved
```

**Right:**
```python
settings2 = Qgs3DMapSettings()
settings2.readXml(elem, QgsReadWriteContext())
settings2.resolveReferences(QgsProject.instance())  # Resolve layer references
```

**Why:** After `readXml()`, layer references stored in the XML are just IDs. Without calling `resolveReferences()`, the settings object will have null layer references and terrain layers will not render.

---

## AP-8: Point Light with Zero Attenuation

**Wrong:**
```python
light = QgsPointLightSettings()
light.setPosition(QgsVector3D(0, 1000, 0))
light.setIntensity(1.0)
# Default attenuation may be 0,0,0 — light has infinite range, washes out scene
```

**Right:**
```python
light = QgsPointLightSettings()
light.setPosition(QgsVector3D(0, 1000, 0))
light.setIntensity(1.0)
light.setConstantAttenuation(1.0)
light.setLinearAttenuation(0.0)
light.setQuadraticAttenuation(0.0)
```

**Why:** ALWAYS explicitly set attenuation values for point lights. The attenuation formula is `Total = A0 + A1*D + A2*D^2`. With all values at zero, light intensity is undefined. Setting `constantAttenuation` to 1.0 with zero linear and quadratic values gives uniform light with no distance falloff.

---

## AP-9: Missing Terrain Settings When Using DEM

**Wrong:**
```python
terrain = QgsDemTerrainSettings()
terrain.setLayer(dem_layer)
settings.setTerrainSettings(terrain)
# Terrain renders with gaps between tiles
```

**Right:**
```python
terrain = QgsDemTerrainSettings()
terrain.setLayer(dem_layer)
terrain.setResolution(16)
terrain.setSkirtHeight(10.0)  # Prevents gaps between terrain tiles
settings.setTerrainSettings(terrain)
```

**Why:** Without `setSkirtHeight()`, terrain tiles can have visible gaps at their edges due to level-of-detail transitions. ALWAYS set a skirt height for DEM terrain to prevent visual artifacts.

---

## AP-10: Using Absolute Altitude Clamping Without Z Values

**Wrong:**
```python
# Layer has no Z coordinates
symbol.setAltitudeClamping(Qgis.AltitudeClamping.Absolute)
# Features render at Z=0, flat on ground — no visible 3D effect
```

**Right:**
```python
# For 2D data, use Terrain clamping to drape on terrain
symbol.setAltitudeClamping(Qgis.AltitudeClamping.Terrain)

# Or use Relative with an offset
symbol.setAltitudeClamping(Qgis.AltitudeClamping.Relative)
symbol.setOffset(5.0)  # 5 meters above terrain
```

**Why:** `Absolute` clamping uses the feature's Z values directly. If the data has no Z coordinates, all features render at Z=0 (ground level), making them invisible if terrain is also at ground level. For 2D data, ALWAYS use `Terrain` clamping (drape on surface) or `Relative` with an offset.

---

## AP-11: Forgetting to Add Layer to 3D Scene

**Wrong:**
```python
renderer = QgsVectorLayer3DRenderer(symbol)
layer.setRenderer3D(renderer)
# Layer has a 3D renderer but is not in settings.setLayers() — nothing renders
```

**Right:**
```python
renderer = QgsVectorLayer3DRenderer(symbol)
layer.setRenderer3D(renderer)
settings.setLayers([layer])  # ALWAYS include the layer in the scene
```

**Why:** Setting a 3D renderer on a layer configures HOW it renders in 3D, but `Qgs3DMapSettings.setLayers()` controls WHICH layers are included in the 3D scene. Both are required for a layer to appear in the 3D view.
