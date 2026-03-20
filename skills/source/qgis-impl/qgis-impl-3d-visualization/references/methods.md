# qgis-impl-3d-visualization — Methods Reference

## Qgs3DMapSettings

Central configuration class for 3D scenes. Import from `qgis._3d`.

### Coordinate System and Extent

| Method | Signature | Notes |
|--------|-----------|-------|
| `setCrs` | `setCrs(crs: QgsCoordinateReferenceSystem)` | Scene CRS |
| `crs` | `crs() -> QgsCoordinateReferenceSystem` | |
| `setExtent` | `setExtent(extent: QgsRectangle)` | Auto-sets origin to center |
| `extent` | `extent() -> QgsRectangle` | |
| `setOrigin` | `setOrigin(origin: QgsVector3D)` | World origin in map coords |
| `origin` | `origin() -> QgsVector3D` | |
| `setTransformContext` | `setTransformContext(context: QgsCoordinateTransformContext)` | |
| `transformContext` | `transformContext() -> QgsCoordinateTransformContext` | |

### Coordinate Conversion

| Method | Signature | Notes |
|--------|-----------|-------|
| `mapToWorldCoordinates` | `mapToWorldCoordinates(mapCoords: QgsVector3D) -> QgsVector3D` | Applies x, -z, y swap |
| `worldToMapCoordinates` | `worldToMapCoordinates(worldCoords: QgsVector3D) -> QgsVector3D` | Inverse transformation |

### Layers

| Method | Signature | Notes |
|--------|-----------|-------|
| `setLayers` | `setLayers(layers: Iterable[QgsMapLayer])` | Layers to render |
| `layers` | `layers() -> List[QgsMapLayer]` | |

### Terrain

| Method | Signature | Notes |
|--------|-----------|-------|
| `setTerrainSettings` | `setTerrainSettings(settings: QgsAbstractTerrainSettings)` | 3.42+ |
| `terrainSettings` | `terrainSettings() -> QgsAbstractTerrainSettings` | 3.42+ |
| `setTerrainRenderingEnabled` | `setTerrainRenderingEnabled(enabled: bool)` | |
| `terrainRenderingEnabled` | `terrainRenderingEnabled() -> bool` | |
| `setTerrainShadingEnabled` | `setTerrainShadingEnabled(enabled: bool)` | |
| `isTerrainShadingEnabled` | `isTerrainShadingEnabled() -> bool` | |
| `setTerrainShadingMaterial` | `setTerrainShadingMaterial(material: QgsPhongMaterialSettings)` | |
| `terrainShadingMaterial` | `terrainShadingMaterial() -> QgsPhongMaterialSettings` | |
| `setTerrainMapTheme` | `setTerrainMapTheme(theme: str)` | |
| `terrainMapTheme` | `terrainMapTheme() -> str` | |
| `configureTerrainFromProject` | `configureTerrainFromProject(props, extent: QgsRectangle)` | |

### Light Sources (3.26+)

| Method | Signature | Notes |
|--------|-----------|-------|
| `setLightSources` | `setLightSources(lights: Iterable[QgsLightSource])` | |
| `lightSources` | `lightSources() -> List[QgsLightSource]` | |
| `setShowLightSourceOrigins` | `setShowLightSourceOrigins(show: bool)` | Debug visualization |

### Camera

| Method | Signature | Notes |
|--------|-----------|-------|
| `setFieldOfView` | `setFieldOfView(fov: float)` | 3.8+ |
| `fieldOfView` | `fieldOfView() -> float` | |
| `setCameraMovementSpeed` | `setCameraMovementSpeed(speed: float)` | 3.18+ |
| `cameraMovementSpeed` | `cameraMovementSpeed() -> float` | |

### Visual Settings

| Method | Signature | Notes |
|--------|-----------|-------|
| `setBackgroundColor` | `setBackgroundColor(color: QColor)` | |
| `backgroundColor` | `backgroundColor() -> QColor` | |
| `setSelectionColor` | `setSelectionColor(color: QColor)` | |
| `selectionColor` | `selectionColor() -> QColor` | |
| `setIsSkyboxEnabled` | `setIsSkyboxEnabled(enabled: bool)` | |
| `isSkyboxEnabled` | `isSkyboxEnabled() -> bool` | |
| `setShowLabels` | `setShowLabels(enabled: bool)` | |
| `showLabels` | `showLabels() -> bool` | |
| `setOutputDpi` | `setOutputDpi(dpi: int)` | |
| `outputDpi` | `outputDpi() -> int` | |

### Eye Dome Lighting (3.18+)

| Method | Signature | Notes |
|--------|-----------|-------|
| `setEyeDomeLightingEnabled` | `setEyeDomeLightingEnabled(enabled: bool)` | |
| `eyeDomeLightingEnabled` | `eyeDomeLightingEnabled() -> bool` | |
| `setEyeDomeLightingStrength` | `setEyeDomeLightingStrength(strength: float)` | |
| `eyeDomeLightingStrength` | `eyeDomeLightingStrength() -> float` | |
| `setEyeDomeLightingDistance` | `setEyeDomeLightingDistance(distance: int)` | |
| `eyeDomeLightingDistance` | `eyeDomeLightingDistance() -> int` | |

### Serialization

| Method | Signature | Notes |
|--------|-----------|-------|
| `writeXml` | `writeXml(doc: QDomDocument, context: QgsReadWriteContext) -> QDomElement` | |
| `readXml` | `readXml(elem: QDomElement, context: QgsReadWriteContext)` | |
| `resolveReferences` | `resolveReferences(project: QgsProject)` | Call after readXml |

---

## QgsPolygon3DSymbol

Factory: `QgsPolygon3DSymbol.create() -> QgsAbstract3DSymbol`. Import from `qgis._3d`.

| Method | Signature | Notes |
|--------|-----------|-------|
| `setAltitudeClamping` | `setAltitudeClamping(mode: Qgis.AltitudeClamping)` | |
| `altitudeClamping` | `altitudeClamping() -> Qgis.AltitudeClamping` | |
| `setAltitudeBinding` | `setAltitudeBinding(mode: Qgis.AltitudeBinding)` | |
| `altitudeBinding` | `altitudeBinding() -> Qgis.AltitudeBinding` | |
| `setOffset` | `setOffset(offset: float)` | Vertical offset |
| `offset` | `offset() -> float` | |
| `setExtrusionHeight` | `setExtrusionHeight(height: float)` | Extrude upward |
| `extrusionHeight` | `extrusionHeight() -> float` | |
| `setExtrusionFaces` | `setExtrusionFaces(faces: Qgis.ExtrusionFaces)` | Which faces to render |
| `setMaterialSettings` | `setMaterialSettings(material: QgsAbstractMaterialSettings)` | |
| `materialSettings` | `materialSettings() -> QgsAbstractMaterialSettings` | |
| `setCullingMode` | `setCullingMode(mode: Qgs3DTypes.CullingMode)` | |
| `cullingMode` | `cullingMode() -> Qgs3DTypes.CullingMode` | |
| `setEdgesEnabled` | `setEdgesEnabled(enabled: bool)` | |
| `edgesEnabled` | `edgesEnabled() -> bool` | |
| `setEdgeWidth` | `setEdgeWidth(width: float)` | |
| `edgeWidth` | `edgeWidth() -> float` | |
| `setEdgeColor` | `setEdgeColor(color: QColor)` | |
| `edgeColor` | `edgeColor() -> QColor` | |

---

## QgsLine3DSymbol

Factory: `QgsLine3DSymbol.create() -> QgsAbstract3DSymbol`. Import from `qgis._3d`.

| Method | Signature | Notes |
|--------|-----------|-------|
| `setWidth` | `setWidth(width: float)` | Width in map units |
| `width` | `width() -> float` | |
| `setOffset` | `setOffset(offset: float)` | Vertical offset |
| `offset` | `offset() -> float` | |
| `setExtrusionHeight` | `setExtrusionHeight(height: float)` | |
| `extrusionHeight` | `extrusionHeight() -> float` | |
| `setAltitudeClamping` | `setAltitudeClamping(mode: Qgis.AltitudeClamping)` | |
| `setAltitudeBinding` | `setAltitudeBinding(mode: Qgis.AltitudeBinding)` | |
| `setRenderAsSimpleLines` | `setRenderAsSimpleLines(enabled: bool)` | |
| `renderAsSimpleLines` | `renderAsSimpleLines() -> bool` | |
| `setMaterialSettings` | `setMaterialSettings(material: QgsAbstractMaterialSettings)` | |
| `materialSettings` | `materialSettings() -> QgsAbstractMaterialSettings` | |

---

## QgsPoint3DSymbol

Constructor: `QgsPoint3DSymbol()`. Import from `qgis._3d`.

| Method | Signature | Notes |
|--------|-----------|-------|
| `setShape` | `setShape(shape: Qgis.Point3DShape)` | Sphere, Cylinder, Cone, Cube, Torus, Plane, Billboard, Model |
| `shape` | `shape() -> Qgis.Point3DShape` | |
| `setShapeProperties` | `setShapeProperties(props: Dict[str, Any])` | e.g. `{"radius": 5.0}` |
| `shapeProperties` | `shapeProperties() -> Dict[str, Any]` | |
| `setMaterialSettings` | `setMaterialSettings(material: QgsAbstractMaterialSettings)` | |
| `materialSettings` | `materialSettings() -> QgsAbstractMaterialSettings` | |
| `setAltitudeClamping` | `setAltitudeClamping(mode: Qgis.AltitudeClamping)` | |
| `setTransform` | `setTransform(matrix: QMatrix4x4)` | Scale/rotation/translation |
| `transform` | `transform() -> QMatrix4x4` | |
| `setBillboardSymbol` | `setBillboardSymbol(symbol: QgsMarkerSymbol)` | For billboard mode |
| `billboardSymbol` | `billboardSymbol() -> QgsMarkerSymbol` | |

Static helpers:
- `shapeFromString(str) -> Qgis.Point3DShape`
- `shapeToString(Qgis.Point3DShape) -> str`

---

## QgsVectorLayer3DRenderer

Constructor: `QgsVectorLayer3DRenderer(symbol: QgsAbstract3DSymbol | None = None)`. Import from `qgis._3d`.

| Method | Signature | Notes |
|--------|-----------|-------|
| `setSymbol` | `setSymbol(symbol: QgsAbstract3DSymbol)` | Takes ownership |
| `symbol` | `symbol() -> QgsAbstract3DSymbol` | |

Apply to a layer: `layer.setRenderer3D(renderer)`

---

## QgsPointCloudLayer3DRenderer

Import from `qgis._3d`.

| Method | Signature | Notes |
|--------|-----------|-------|
| `setSymbol` | `setSymbol(symbol: QgsPointCloud3DSymbol)` | |
| `symbol` | `symbol() -> QgsPointCloud3DSymbol` | |

---

## QgsPhongMaterialSettings

Import from `qgis._3d`.

| Method | Signature | Notes |
|--------|-----------|-------|
| `setAmbient` | `setAmbient(color: QColor)` | |
| `ambient` | `ambient() -> QColor` | |
| `setDiffuse` | `setDiffuse(color: QColor)` | |
| `diffuse` | `diffuse() -> QColor` | |
| `setSpecular` | `setSpecular(color: QColor)` | |
| `specular` | `specular() -> QColor` | |
| `setShininess` | `setShininess(shininess: float)` | |
| `shininess` | `shininess() -> float` | |
| `setOpacity` | `setOpacity(opacity: float)` | 3.26+ |
| `opacity` | `opacity() -> float` | |
| `setAmbientCoefficient` | `setAmbientCoefficient(coeff: float)` | 3.36+ |
| `setDiffuseCoefficient` | `setDiffuseCoefficient(coeff: float)` | 3.36+ |
| `setSpecularCoefficient` | `setSpecularCoefficient(coeff: float)` | 3.36+ |

---

## QgsGoochMaterialSettings (3.16+)

Import from `qgis._3d`.

| Method | Signature | Notes |
|--------|-----------|-------|
| `setWarm` | `setWarm(color: QColor)` | Warm tone |
| `warm` | `warm() -> QColor` | |
| `setCool` | `setCool(color: QColor)` | Cool tone |
| `cool` | `cool() -> QColor` | |
| `setDiffuse` | `setDiffuse(color: QColor)` | |
| `setSpecular` | `setSpecular(color: QColor)` | |
| `setShininess` | `setShininess(shininess: float)` | |
| `setAlpha` | `setAlpha(alpha: float)` | Warm/cool blend |
| `setBeta` | `setBeta(beta: float)` | Diffuse blend |

---

## QgsMetalRoughMaterialSettings (3.36+)

Import from `qgis._3d`.

| Method | Signature | Notes |
|--------|-----------|-------|
| `setBaseColor` | `setBaseColor(color: QColor)` | |
| `baseColor` | `baseColor() -> QColor` | |
| `setMetalness` | `setMetalness(metalness: float)` | 0.0-1.0 |
| `metalness` | `metalness() -> float` | |
| `setRoughness` | `setRoughness(roughness: float)` | 0.0-1.0 |
| `roughness` | `roughness() -> float` | |

---

## QgsPointLightSettings

Constructor: `QgsPointLightSettings()`. Import from `qgis._3d`.

| Method | Signature | Notes |
|--------|-----------|-------|
| `setPosition` | `setPosition(pos: QgsVector3D)` | World coordinates |
| `position` | `position() -> QgsVector3D` | |
| `setColor` | `setColor(color: QColor)` | |
| `color` | `color() -> QColor` | |
| `setIntensity` | `setIntensity(intensity: float)` | |
| `intensity` | `intensity() -> float` | |
| `setConstantAttenuation` | `setConstantAttenuation(a0: float)` | |
| `setLinearAttenuation` | `setLinearAttenuation(a1: float)` | |
| `setQuadraticAttenuation` | `setQuadraticAttenuation(a2: float)` | |

Attenuation formula: `Total = A0 + A1*D + A2*D^2`

---

## QgsDirectionalLightSettings (3.16+)

Constructor: `QgsDirectionalLightSettings()`. Import from `qgis._3d`.

| Method | Signature | Notes |
|--------|-----------|-------|
| `setDirection` | `setDirection(direction: QgsVector3D)` | Direction in degrees |
| `direction` | `direction() -> QgsVector3D` | |
| `setColor` | `setColor(color: QColor)` | |
| `color` | `color() -> QColor` | |
| `setIntensity` | `setIntensity(intensity: float)` | |
| `intensity` | `intensity() -> float` | |

---

## QgsCameraPose

Constructor: `QgsCameraPose()`. Import from `qgis._3d`.

| Method | Signature | Notes |
|--------|-----------|-------|
| `setCenterPoint` | `setCenterPoint(point: QgsVector3D)` | Focal point |
| `centerPoint` | `centerPoint() -> QgsVector3D` | |
| `setDistanceFromCenterPoint` | `setDistanceFromCenterPoint(distance: float)` | Camera distance |
| `distanceFromCenterPoint` | `distanceFromCenterPoint() -> float` | |
| `setPitchAngle` | `setPitchAngle(angle: float)` | 0=down, 90=horizontal |
| `pitchAngle` | `pitchAngle() -> float` | |
| `setHeadingAngle` | `setHeadingAngle(angle: float)` | Horizontal rotation |
| `headingAngle` | `headingAngle() -> float` | |

---

## Terrain Settings Classes

### QgsFlatTerrainSettings

| Method | Signature | Notes |
|--------|-----------|-------|
| `setElevation` | `setElevation(elevation: float)` | Fixed height |
| `elevation` | `elevation() -> float` | |

### QgsDemTerrainSettings

| Method | Signature | Notes |
|--------|-----------|-------|
| `setLayer` | `setLayer(layer: QgsRasterLayer)` | DEM raster |
| `layer` | `layer() -> QgsRasterLayer` | |
| `setResolution` | `setResolution(resolution: int)` | Tile resolution |
| `resolution` | `resolution() -> int` | |
| `setSkirtHeight` | `setSkirtHeight(height: float)` | Prevent gaps |
| `skirtHeight` | `skirtHeight() -> float` | |

---

## QgsLayoutItem3DMap (3.4+)

Import from `qgis._3d`.

| Method | Signature | Notes |
|--------|-----------|-------|
| `setMapSettings` | `setMapSettings(settings: Qgs3DMapSettings)` | |
| `setCameraPose` | `setCameraPose(pose: QgsCameraPose)` | |
| `cameraPose` | `cameraPose() -> QgsCameraPose` | |
