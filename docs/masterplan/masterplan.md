# QGIS Skill Package — Definitive Masterplan

## Status

Phase 3 complete. Finalized from raw masterplan after vooronderzoek review.
Date: 2026-03-20

---

## Decisions Made During Refinement

| # | Decision | Rationale |
|---|----------|-----------|
| D-01 | **Kept** all 19 skills unchanged | Research confirms the scope of each skill is well-sized. No splits, merges, or removals needed. The PyQGIS API surface maps cleanly to the 5 categories. |
| D-02 | **Confirmed** expressions as single skill | Q-004 resolved: the expression engine is substantial but fits in one skill with references/. Custom functions, field calculator, expression-based labeling/symbology all belong together. |
| D-03 | **Confirmed** 3D as single skill | Q-005 resolved: mesh and point cloud 3D rendering are thin enough to include in `impl-3d-visualization`. Standalone mesh/point cloud skills would be too small. |
| D-04 | **Added** symbology coverage to `syntax-pyqgis-api` references | Symbology/rendering is documented in vooronderzoek-pyqgis §10. Core patterns go in `syntax-pyqgis-api/references/`, advanced patterns in relevant impl skills. |
| D-05 | **Set** QGIS 3.44 LTR as primary target | D-006 from DECISIONS.md. QGIS 4.0 compatibility notes included per skill where relevant. |
| D-06 | **Reordered** batch 3 to include `syntax-plugins` first | Plugin development is a common entry point for QGIS developers and does not depend on other syntax skills. Moving it earlier improves the batch dependency chain. |

**Result**: 19 raw skills → **19 definitive skills** (0 merges, 0 additions, 0 removals).

---

## Definitive Skill Inventory (19 skills)

### qgis-core/ (3 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `qgis-core-architecture` | QGIS architecture; Qt-based design; 5 core modules; project lifecycle (`QgsProject`); layer model; provider architecture; plugin system; main thread constraints; QGIS 4.0 Qt6 notes | `QgsProject`, `QgsApplication`, `QgsProviderRegistry`, `QgsMapLayer` | architecture §1-3, §6-7 | M | None |
| `qgis-core-data-providers` | Data provider system; 12+ providers; vector formats (OGR, PostGIS, memory, delimited text, WFS); raster formats (GDAL, WMS, XYZ); URI construction patterns; `QgsDataSourceUri`; provider capabilities; GeoPackage as preferred format | `QgsDataSourceUri`, `QgsVectorLayer`, `QgsRasterLayer`, `QgsProviderRegistry` | architecture §4 | M | None |
| `qgis-core-coordinate-systems` | CRS handling; creation from EPSG/WKT/Proj; `QgsCoordinateTransform`; transform context; datum transformations; `QgsDistanceArea`; common CRS pitfalls; lat/lon vs lon/lat; EPSG database | `QgsCoordinateReferenceSystem`, `QgsCoordinateTransform`, `QgsCoordinateTransformContext`, `QgsDistanceArea` | architecture §5 | M | None |

### qgis-syntax/ (4 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `qgis-syntax-pyqgis-api` | PyQGIS scripting patterns; 4 execution contexts; layer CRUD; feature iteration with `QgsFeatureRequest`; editing with `with edit(layer):`; geometry operations; spatial index; memory layers; background tasks (`QgsTask`); threading constraints; basic symbology | `QgsFeature`, `QgsFeatureRequest`, `QgsGeometry`, `QgsSpatialIndex`, `QgsTask`, `QgsSymbol*`, `QgsRenderer*` | pyqgis §1-5, §7-8, §10 | L | core-architecture |
| `qgis-syntax-expressions` | Expression engine; `QgsExpression` parsing/evaluation; context and scopes; field calculator patterns; expression-based filtering/labeling/symbology; `@qgsfunction` custom functions; common functions reference | `QgsExpression`, `QgsExpressionContext`, `QgsExpressionContextScope`, `QgsExpressionContextUtils` | pyqgis §6 | M | core-architecture |
| `qgis-syntax-processing-scripts` | Processing framework; `processing.run()` entry point; algorithm IDs and parameters; custom `QgsProcessingAlgorithm` subclass; `@alg` decorator; parameter types; feedback/progress; chaining; batch processing; background task execution | `QgsProcessingAlgorithm`, `QgsProcessingProvider`, `QgsProcessingContext`, `QgsProcessingFeedback`, `QgsProcessingParameter*` | processing §1-6 | L | core-architecture |
| `qgis-syntax-plugins` | QGIS plugin development; structure (metadata.txt, `__init__.py`); `classFactory`/`initGui`/`unload` lifecycle; UI integration (menus, toolbars, dock widgets); Qt Designer; resource compilation; Plugin Builder; QGIS Plugin Repository publishing | Plugin lifecycle, `iface`, Qt widgets, metadata.txt | pyqgis §9 | L | core-architecture |

### qgis-impl/ (8 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `qgis-impl-vector-analysis` | Spatial queries (by location/expression); buffer/dissolve/clip/union/difference; attribute joins; spatial joins; field calculations; feature selection/extraction; `QgsVectorFileWriter`; processing algorithms for vector ops | `QgsGeometry`, `QgsSpatialIndex`, `QgsFeatureRequest`, `QgsVectorFileWriter`, native:buffer, native:clip, etc. | integration §1, processing §3 (vector) | M | syntax-pyqgis-api |
| `qgis-impl-raster-analysis` | `QgsRasterCalculator` (map algebra); raster statistics; DEM terrain analysis (hillshade, slope, aspect, relief); raster sampling/identify; raster renderers; raster reprojection; GDAL processing algorithms | `QgsRasterCalculator`, `QgsRasterCalculatorEntry`, `QgsHillshadeFilter`, `QgsSlopeFilter`, `QgsAspectFilter`, `QgsRelief` | integration §2, processing §7 | M | syntax-pyqgis-api |
| `qgis-impl-print-layouts` | `QgsPrintLayout` creation; layout items (map, label, legend, scale bar, picture, table); atlas generation with `QgsLayoutAtlas`; export to PDF/SVG/image; page setup; template save/load | `QgsPrintLayout`, `QgsLayoutItemMap`, `QgsLayoutItemLabel`, `QgsLayoutItemLegend`, `QgsLayoutItemScaleBar`, `QgsLayoutExporter`, `QgsLayoutAtlas` | integration §3 | L | syntax-pyqgis-api |
| `qgis-impl-postgis` | PostGIS connection via `QgsDataSourceUri`; loading table/view/SQL layers; executing SQL; `QgsAbstractDatabaseProviderConnection`; schema/table discovery; creating tables from QGIS; authentication; PostGIS raster | `QgsDataSourceUri`, `QgsAbstractDatabaseProviderConnection`, `QgsProviderRegistry` | integration §4 | M | core-data-providers |
| `qgis-impl-web-services` | WMS/WMTS client (URI format); WFS client (URI format, versions); WCS; XYZ tiles; QGIS Server setup; server plugins/filters; `QgsServerOgcApi`; authentication for services | `wms`/`WFS` providers, `QgsServer`, `QgsServerFilter`, `QgsServerOgcApi`, `QgsAuthManager` | integration §5 | M | core-data-providers |
| `qgis-impl-3d-visualization` | `Qgs3DMapSettings` configuration; terrain providers (flat, DEM, mesh, online); vector/point cloud 3D renderers; 3D symbols; materials (Phong, Gooch); camera/lights; layout integration; mesh and point cloud 3D | `Qgs3DMapSettings`, `QgsVectorLayer3DRenderer`, `QgsPoint3DSymbol`, `QgsLine3DSymbol`, `QgsPolygon3DSymbol`, `QgsPhongMaterialSettings` | integration §6 | M | syntax-pyqgis-api |
| `qgis-impl-georeferencing` | GCP management (`QgsGcpPoint`); 7 transformation types (linear, Helmert, polynomial 1-3, TPS); `QgsGcpTransformerInterface`; `QgsVectorWarper`; raster georeferencing; residual computation; accuracy assessment; world files | `QgsGcpPoint`, `QgsGcpTransformerInterface`, `QgsVectorWarper` | integration §7 | M | syntax-pyqgis-api |
| `qgis-impl-network-analysis` | `QgsGraphBuilder` for building road network graphs; `QgsVectorLayerDirector`; distance/speed strategies; `QgsGraphAnalyzer.dijkstra()` shortest path; shortest tree; service area analysis; cost functions | `QgsGraph`, `QgsGraphAnalyzer`, `QgsGraphBuilder`, `QgsVectorLayerDirector`, `QgsNetworkDistanceStrategy`, `QgsNetworkSpeedStrategy` | integration §8 | M | syntax-pyqgis-api |

### qgis-errors/ (2 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `qgis-errors-projections` | CRS/projection errors; wrong EPSG selection; datum transformation warnings; lat/lon vs lon/lat confusion; silent data corruption; on-the-fly reprojection pitfalls; debugging projection problems; fix patterns | `QgsCoordinateReferenceSystem`, `QgsCoordinateTransform` | architecture §5 (pitfalls), §7, integration §9 | M | core-coordinate-systems |
| `qgis-errors-data-loading` | Invalid layer detection (`isValid()`); provider URI format errors; encoding issues; Shapefile limitations; GeoPackage locking; PostGIS connection failures; WMS/WFS timeouts; large dataset performance; missing CRS; fix patterns | `QgsVectorLayer.isValid()`, provider error messages | architecture §3 (validity), §4 (URIs), §7, integration §9 | M | core-data-providers |

### qgis-agents/ (2 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `qgis-agents-analysis-orchestrator` | Decision tree for analysis approach; vector vs raster decision; processing algorithm selection; workflow chaining guidance; data format recommendations; CRS selection; output format selection; MCP server reference | All analysis classes | All vooronderzoek anti-pattern sections | M | ALL other skills |
| `qgis-agents-map-generator` | Map generation pipeline; data loading → styling → layout → export; symbology selection guidance; label placement rules; print layout best practices; atlas patterns; cartographic conventions; MCP server reference | All layout and rendering classes | pyqgis §10, integration §3 | M | ALL other skills |

---

## Batch Execution Plan (DEFINITIVE)

| Batch | Skills | Count | Dependencies | Notes |
|-------|--------|-------|-------------|-------|
| 1 | `core-architecture`, `core-data-providers`, `core-coordinate-systems` | 3 | None | Foundation skills, no dependencies |
| 2 | `syntax-pyqgis-api`, `syntax-expressions`, `syntax-plugins` | 3 | Batch 1 | Primary syntax patterns |
| 3 | `syntax-processing-scripts`, `impl-vector-analysis`, `impl-raster-analysis` | 3 | Batch 1-2 | Processing + first impl skills |
| 4 | `impl-print-layouts`, `impl-postgis`, `impl-web-services` | 3 | Batch 1-2 | Output and integration skills |
| 5 | `impl-3d-visualization`, `impl-georeferencing`, `impl-network-analysis` | 3 | Batch 1-2 | Specialized impl skills |
| 6 | `errors-projections`, `errors-data-loading` | 2 | Batch 1-5 | Error diagnosis skills |
| 7 | `agents-analysis-orchestrator`, `agents-map-generator` | 2 | ALL above | Agent skills last |

**Total**: 19 skills across 7 batches.

---

## Per-Skill Agent Prompts

### Constants

```
PROJECT_ROOT = C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package
ARCH_RESEARCH = C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\docs\research\vooronderzoek-qgis-architecture.md
PYQGIS_RESEARCH = C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\docs\research\vooronderzoek-qgis-pyqgis.md
PROC_RESEARCH = C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\docs\research\vooronderzoek-qgis-processing.md
INTEG_RESEARCH = C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\docs\research\vooronderzoek-qgis-integration.md
REQUIREMENTS_FILE = C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\REQUIREMENTS.md
REFERENCE_SKILL = C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md
```

---

### Batch 1

#### Prompt: qgis-core-architecture

```
## Task: Create the qgis-core-architecture skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-core\qgis-core-architecture\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for QgsProject, QgsApplication, QgsMapLayer, QgsProviderRegistry)
3. references/examples.md (working code examples)
4. references/anti-patterns.md (what NOT to do)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-core-architecture
description: >
  Use when creating QGIS projects, understanding PyQGIS architecture, or reasoning about the QGIS data model.
  Prevents mixing standalone script initialization with plugin context, and threading violations.
  Covers Qt-based architecture, 5 core modules, project lifecycle, layer model, provider system, and QGIS 4.0 Qt6 notes.
  Keywords: QGIS architecture, QgsProject, QgsApplication, PyQGIS modules, layer model, provider system, plugin architecture.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- QGIS Qt-based architecture: core modules (qgis.core, qgis.gui, qgis.analysis, qgis.server, qgis._3d)
- QgsApplication initialization: standalone scripts vs plugin context vs processing scripts
- QgsProject singleton: loading, saving, project formats (.qgs, .qgz), path resolution
- Layer model: QgsMapLayer hierarchy (vector, raster, mesh, point cloud, annotation)
- Provider registry: QgsProviderRegistry and provider plugin system
- Main thread constraints and signal/slot patterns
- QGIS 4.0 Qt6 migration notes (QMetaType.Type vs QVariant.Type)

### Research Sections to Read
From vooronderzoek-qgis-architecture.md:
- Section 1: QGIS Architecture Overview (lines 9-77)
- Section 2: QgsProject and Application Lifecycle (lines 78-235)
- Section 3: Layer Architecture (lines 236-376)
- Section 6: QGIS 4.0 Changes (lines 887-930)
- Section 7: Anti-patterns (lines 931-1001)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language, not "you should" or "consider"
- All code examples must be verified against vooronderzoek research
- Include version annotations (QGIS 3.44 LTR, 4.0 where relevant)
- Include Critical Warnings section with NEVER rules
- Delete the .gitkeep file in the output directory before writing
```

#### Prompt: qgis-core-data-providers

```
## Task: Create the qgis-core-data-providers skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-core\qgis-core-data-providers\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for QgsDataSourceUri, QgsVectorLayer, QgsRasterLayer constructors)
3. references/examples.md (URI format examples for every provider)
4. references/anti-patterns.md (what NOT to do)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-core-data-providers
description: >
  Use when loading data into QGIS, constructing layer URIs, or choosing between data formats.
  Prevents URI format errors that cause silent layer loading failures.
  Covers 12+ data providers, URI construction patterns, vector/raster/mesh formats, and GeoPackage best practices.
  Keywords: data provider, QgsDataSourceUri, load layer, Shapefile, GeoPackage, GeoJSON, PostGIS, WMS, WFS, memory layer, CSV.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Provider system overview: QgsProviderRegistry, provider keys
- Vector providers: ogr (Shapefile, GeoPackage, GeoJSON, FlatGeoBuf, KML, DXF), postgres, spatialite, memory, delimitedtext, WFS, virtual
- Raster providers: gdal (GeoTIFF, JPEG2000, COG), wms (WMS, WMTS, XYZ tiles), wcs, postgresraster
- Other providers: pdal/copc/ept (point clouds), mdal (mesh), vectortile, cesiumtiles
- URI construction patterns with exact format strings for each provider
- QgsDataSourceUri for database connections
- Layer validity checking after loading
- GeoPackage as recommended default format

### Research Sections to Read
From vooronderzoek-qgis-architecture.md:
- Section 4: Data Provider Architecture (lines 377-670) — ALL provider URIs
- Section 3: Layer Architecture (lines 236-376) — layer creation and validity
- Section 7: Anti-patterns (lines 931-1001) — provider-related anti-patterns

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include EXACT URI format strings for every provider — this is the most important content
- Include version annotations
- Include Critical Warnings section
- Delete the .gitkeep file in the output directory before writing
```

#### Prompt: qgis-core-coordinate-systems

```
## Task: Create the qgis-core-coordinate-systems skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-core\qgis-core-coordinate-systems\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for QgsCoordinateReferenceSystem, QgsCoordinateTransform, QgsDistanceArea)
3. references/examples.md (working code examples for CRS operations)
4. references/anti-patterns.md (CRS pitfalls and what NOT to do)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-core-coordinate-systems
description: >
  Use when handling coordinate reference systems, transforming coordinates, or measuring distances/areas.
  Prevents silent data corruption from wrong CRS assignment and lat/lon coordinate order confusion.
  Covers CRS creation, coordinate transforms, datum transformations, distance measurement, and EPSG database.
  Keywords: CRS, EPSG, coordinate transform, projection, QgsCoordinateReferenceSystem, datum, QgsDistanceArea, reprojection.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- QgsCoordinateReferenceSystem creation: fromEpsgId(), fromWkt(), fromProj(), fromOgcWmsCrs()
- CRS properties: authid(), description(), isGeographic(), mapUnits()
- QgsCoordinateTransform: construction, transform(), transformBoundingBox()
- QgsCoordinateTransformContext: project-level transform settings
- Datum transformations: availability, selection
- QgsDistanceArea: measuring in any CRS, ellipsoidal vs planimetric
- Common CRS pitfalls: lat/lon order, wrong EPSG, silent corruption, on-the-fly reprojection
- EPSG database and proj.db

### Research Sections to Read
From vooronderzoek-qgis-architecture.md:
- Section 5: Coordinate Reference System Handling (lines 671-886) — ALL CRS content
- Section 7: Anti-patterns (lines 931-1001) — CRS-related warnings

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples must be verified against vooronderzoek research
- Include a decision tree for "which CRS to use" scenarios
- Include Critical Warnings section with CRS-specific NEVER rules
- Delete the .gitkeep file in the output directory before writing
```

---

### Batch 2

#### Prompt: qgis-syntax-pyqgis-api

```
## Task: Create the qgis-syntax-pyqgis-api skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-syntax\qgis-syntax-pyqgis-api\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for QgsFeature, QgsFeatureRequest, QgsGeometry, QgsSpatialIndex, QgsTask)
3. references/examples.md (working code examples for all major patterns)
4. references/anti-patterns.md (editing, threading, iteration anti-patterns)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-syntax-pyqgis-api
description: >
  Use when writing PyQGIS scripts, accessing features, editing layers, or performing geometry operations.
  Prevents edit session violations, threading crashes, and inefficient feature iteration.
  Covers 4 scripting contexts, feature CRUD, geometry operations, spatial indexing, background tasks, and basic symbology.
  Keywords: PyQGIS, QgsFeature, QgsFeatureRequest, QgsGeometry, edit layer, spatial index, QgsTask, background thread, symbology.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- 4 scripting contexts: Python console, standalone script, processing script, plugin
- Feature access: QgsFeature (attributes, geometry, id), QgsFeatureRequest (filters, flags, limit)
- Feature editing: edit sessions, `with edit(layer):`, add/modify/delete features, field management
- Geometry operations: construction, predicates, GEOS operations, measurements, validation
- QgsPoint vs QgsPointXY distinction
- Spatial indexing: QgsSpatialIndex (R-tree), nearest neighbor, intersection
- Background tasks: QgsTask subclass, QgsTaskManager, threading constraints
- User communication: QgsMessageLog, QgsMessageBar
- Basic symbology: symbol hierarchy, renderer types, labeling overview

### Research Sections to Read
From vooronderzoek-qgis-pyqgis.md:
- Section 1: PyQGIS Scripting Context (lines 9-74)
- Section 2: Feature Access and Iteration (lines 75-174)
- Section 3: Feature Editing (lines 175-319)
- Section 4: Geometry Operations (lines 320-515)
- Section 5: Spatial Indexing (lines 516-555)
- Section 7: Background Tasks (lines 731-875)
- Section 8: Communicating with User (lines 876-994)
- Section 10: Symbology and Rendering (lines 1152-1366) — overview only, details in references/
- Section 11: Anti-patterns (lines 1367-1413)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- This is the LARGEST syntax skill — be ruthless about what goes in SKILL.md vs references/
- Include Critical Warnings section (threading, editing, iteration)
- Delete the .gitkeep file in the output directory before writing
```

#### Prompt: qgis-syntax-expressions

```
## Task: Create the qgis-syntax-expressions skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-syntax\qgis-syntax-expressions\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for QgsExpression, QgsExpressionContext, @qgsfunction)
3. references/examples.md (expression examples for filtering, labeling, symbology, field calculator)
4. references/anti-patterns.md (expression pitfalls)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-syntax-expressions
description: >
  Use when writing QGIS expressions for filtering, labeling, symbology, or field calculations.
  Prevents expression syntax errors and context misconfiguration.
  Covers QgsExpression parsing, evaluation contexts, field calculator, data-defined properties, and custom functions.
  Keywords: QgsExpression, expression, field calculator, label expression, data-defined, @qgsfunction, filter, evaluate.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- QgsExpression: parsing, evaluation, error checking
- QgsExpressionContext: project/layer/feature scopes, custom scopes
- Expression syntax: operators, field references, string/math/date/geometry functions
- Field calculator patterns via PyQGIS
- Expression-based filtering in QgsFeatureRequest
- Expression-based labeling (data-defined label properties)
- Expression-based symbology (data-defined symbol properties)
- Custom expression functions: @qgsfunction decorator
- Common expression functions reference (geometry accessors, string ops, aggregates)

### Research Sections to Read
From vooronderzoek-qgis-pyqgis.md:
- Section 6: Expression Engine (lines 556-730) — ALL expression content

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include a quick-reference table of common expression functions
- Include Critical Warnings section
- Delete the .gitkeep file in the output directory before writing
```

#### Prompt: qgis-syntax-plugins

```
## Task: Create the qgis-syntax-plugins skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-syntax\qgis-syntax-plugins\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (plugin lifecycle methods, metadata.txt format, iface methods)
3. references/examples.md (minimal plugin skeleton, UI integration examples)
4. references/anti-patterns.md (plugin development pitfalls)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
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

### Scope (EXACT — do not exceed)
- Plugin file structure: __init__.py, metadata.txt, main class, resources_rc.py
- classFactory() entry point
- initGui(): adding menus, toolbars, actions, dock widgets
- unload(): cleanup (remove actions, disconnect signals)
- metadata.txt format: required and optional fields
- iface object: QgisInterface methods for UI integration
- Qt Designer integration for .ui files
- Resource compilation (pyrcc5)
- Plugin Builder tool
- Publishing to QGIS Plugin Repository
- Plugin settings storage

### Research Sections to Read
From vooronderzoek-qgis-pyqgis.md:
- Section 9: Plugin Development (lines 995-1151) — ALL plugin content
- Section 11: Anti-patterns (lines 1367-1413) — plugin-related warnings

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include a complete minimal plugin skeleton in references/examples.md
- Include Critical Warnings section (cleanup, threading in plugins)
- Delete the .gitkeep file in the output directory before writing
```

---

### Batch 3

#### Prompt: qgis-syntax-processing-scripts

```
## Task: Create the qgis-syntax-processing-scripts skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-syntax\qgis-syntax-processing-scripts\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for QgsProcessingAlgorithm, processing.run, parameter types)
3. references/examples.md (running algorithms, custom algorithm, batch processing)
4. references/anti-patterns.md (processing pitfalls)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-syntax-processing-scripts
description: >
  Use when running QGIS processing algorithms, creating custom algorithms, or chaining geoprocessing operations.
  Prevents hardcoded algorithm IDs, missing feedback handling, and incorrect parameter types.
  Covers processing.run(), custom QgsProcessingAlgorithm, @alg decorator, parameter types, and batch processing.
  Keywords: processing.run, QgsProcessingAlgorithm, algorithm, geoprocessing, batch processing, processing script, native:buffer.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- processing.run() entry point: parameters dict, context, feedback
- Algorithm ID format: "provider:algorithm_id"
- Key parameter types (FeatureSource, FeatureSink, RasterLayer, Number, Enum, Field, Expression, Crs, etc.)
- Custom QgsProcessingAlgorithm subclass: name, displayName, initAlgorithm, processAlgorithm, createInstance
- @alg decorator alternative for script algorithms
- Custom QgsProcessingProvider: registration in plugin
- QgsProcessingFeedback: progress, cancellation, logging
- Algorithm chaining: passing outputs as inputs
- Batch processing patterns
- Running as QgsProcessingAlgRunnerTask (background)
- Error handling

### Research Sections to Read
From vooronderzoek-qgis-processing.md:
- Section 1: Processing Framework Architecture (lines 9-94)
- Section 2: Running Algorithms (lines 95-285)
- Section 4: Parameter Types (lines 638-778)
- Section 5: Custom Algorithm Development (lines 779-1054)
- Section 6: Algorithm Chaining (lines 1055-1168)
- Section 8: Anti-patterns (lines 1370-1564)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include a complete custom algorithm example in references/examples.md
- Move the algorithm ID reference table to references/methods.md
- Include Critical Warnings section
- Delete the .gitkeep file in the output directory before writing
```

#### Prompt: qgis-impl-vector-analysis

```
## Task: Create the qgis-impl-vector-analysis skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-impl\qgis-impl-vector-analysis\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for QgsGeometry operations, QgsVectorFileWriter, processing alg IDs)
3. references/examples.md (complete vector analysis workflows)
4. references/anti-patterns.md (vector analysis pitfalls)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-impl-vector-analysis
description: >
  Use when performing spatial analysis on vector data: buffering, clipping, intersecting, dissolving, or joining layers.
  Prevents geometry operation failures from NULL geometries and wrong CRS combinations.
  Covers spatial queries, overlay operations, attribute/spatial joins, field calculations, and vector output.
  Keywords: vector analysis, buffer, clip, intersect, dissolve, spatial join, overlay, QgsGeometry, QgsVectorFileWriter.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Spatial queries: by location (intersects, within, contains), by expression
- Buffer analysis: single/variable distance, dissolve option
- Overlay operations: intersection, union, difference, clip, symmetrical difference
- Attribute joins: joinByField
- Spatial joins: joinByLocation
- Field calculations via PyQGIS
- Feature selection and extraction
- QgsVectorFileWriter: writing output to files
- Key processing algorithms: native:buffer, native:clip, native:intersection, native:union, native:dissolve, native:joinattributestable, native:joinattributesbylocation

### Research Sections to Read
From vooronderzoek-qgis-integration.md:
- Section 1: Vector Analysis Workflows (lines 9-285) — ALL vector analysis
From vooronderzoek-qgis-processing.md:
- Section 3: Algorithm Categories (lines 286-637) — vector algorithm IDs

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include decision tree: "which overlay operation to use"
- Include both PyQGIS native and processing.run() approaches for each operation
- Delete the .gitkeep file in the output directory before writing
```

#### Prompt: qgis-impl-raster-analysis

```
## Task: Create the qgis-impl-raster-analysis skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-impl\qgis-impl-raster-analysis\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for QgsRasterCalculator, raster renderers, terrain classes)
3. references/examples.md (raster analysis workflows)
4. references/anti-patterns.md (raster analysis pitfalls)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-impl-raster-analysis
description: >
  Use when performing raster analysis: map algebra, terrain analysis, raster classification, or DEM processing.
  Prevents NoData handling errors and incorrect raster calculator expressions.
  Covers QgsRasterCalculator, DEM analysis (hillshade/slope/aspect), raster renderers, and GDAL processing algorithms.
  Keywords: raster analysis, QgsRasterCalculator, DEM, hillshade, slope, aspect, raster renderer, terrain, map algebra.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- QgsRasterLayer: loading, querying (sample, identify), statistics
- QgsRasterCalculator: map algebra expressions, entry setup
- Terrain analysis: QgsHillshadeFilter, QgsSlopeFilter, QgsAspectFilter, QgsRelief
- Raster renderers: single band gray, pseudocolor, multiband color, paletted
- Raster statistics and histograms
- Raster reprojection via processing
- Key processing algorithms: gdal:hillshade, gdal:slope, gdal:aspect, gdal:contour, gdal:warpreproject, gdal:translate, gdal:polygonize, gdal:rasterize

### Research Sections to Read
From vooronderzoek-qgis-integration.md:
- Section 2: Raster Analysis (lines 286-529) — ALL raster content
From vooronderzoek-qgis-processing.md:
- Section 7: GDAL Provider Algorithms (lines 1169-1369) — GDAL algorithm details

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- ALWAYS warn about NoData handling
- Include both PyQGIS native and processing.run() approaches
- Delete the .gitkeep file in the output directory before writing
```

---

### Batch 4

#### Prompt: qgis-impl-print-layouts

```
## Task: Create the qgis-impl-print-layouts skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-impl\qgis-impl-print-layouts\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for QgsPrintLayout, all layout items, QgsLayoutExporter)
3. references/examples.md (complete layout creation and export workflows)
4. references/anti-patterns.md (layout pitfalls)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-impl-print-layouts
description: >
  Use when creating print layouts, exporting maps to PDF/SVG/image, or generating atlas series.
  Prevents empty map exports from missing extent configuration and atlas misconfiguration.
  Covers QgsPrintLayout, layout items (map, label, legend, scale bar), atlas generation, and PDF/SVG/image export.
  Keywords: print layout, QgsPrintLayout, map export, PDF, atlas, legend, scale bar, QgsLayoutExporter, map composition.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- QgsPrintLayout creation and QgsLayoutManager
- Layout items: QgsLayoutItemMap (extent, scale, CRS, grids), QgsLayoutItemLabel, QgsLayoutItemLegend, QgsLayoutItemScaleBar, QgsLayoutItemPicture, QgsLayoutItemAttributeTable
- Page setup: QgsLayoutSize, QgsLayoutPoint, page dimensions
- Atlas generation: QgsLayoutAtlas, coverage layer, expression-based filenames
- Export: QgsLayoutExporter (exportToPdf, exportToSvg, exportToImage)
- Template save/load
- Layout expressions

### Research Sections to Read
From vooronderzoek-qgis-integration.md:
- Section 3: Print Layout System (lines 530-761) — ALL layout content

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include a step-by-step "create layout → add items → export" workflow
- ALWAYS set map extent before export
- Delete the .gitkeep file in the output directory before writing
```

#### Prompt: qgis-impl-postgis

```
## Task: Create the qgis-impl-postgis skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-impl\qgis-impl-postgis\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for QgsDataSourceUri, QgsAbstractDatabaseProviderConnection)
3. references/examples.md (PostGIS connection, loading, SQL execution examples)
4. references/anti-patterns.md (PostGIS integration pitfalls)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-impl-postgis
description: >
  Use when connecting to PostGIS databases, loading spatial tables, or executing SQL from PyQGIS.
  Prevents connection string errors and plain-text password exposure in production.
  Covers QgsDataSourceUri, PostGIS layer loading (table/view/SQL), SQL execution, schema discovery, and authentication.
  Keywords: PostGIS, QgsDataSourceUri, PostgreSQL, spatial database, SQL, database connection, schema, PostGIS raster.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- QgsDataSourceUri construction: host, port, dbname, schema, table, geometryColumn, srid, key
- Loading PostGIS layers: table, view, SQL subquery (estimatedmetadata, checkPrimaryKeyUnicity)
- Executing SQL: QgsAbstractDatabaseProviderConnection.executeSql()
- Schema and table discovery
- Creating tables from QGIS layers (native:importintopostgis)
- PostGIS raster loading
- Authentication: QgsAuthManager integration, auth config IDs
- Connection management: stored connections vs URI

### Research Sections to Read
From vooronderzoek-qgis-integration.md:
- Section 4: PostGIS Integration (lines 762-919) — ALL PostGIS content
- Section 9: Anti-patterns (lines 1785-end) — PostGIS warnings

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- NEVER store passwords in plain text — always use QgsAuthManager
- Include complete connection examples
- Delete the .gitkeep file in the output directory before writing
```

#### Prompt: qgis-impl-web-services

```
## Task: Create the qgis-impl-web-services skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-impl\qgis-impl-web-services\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (URI formats for WMS/WMTS/WFS/WCS/XYZ, QgsServer API)
3. references/examples.md (loading web services, QGIS Server setup)
4. references/anti-patterns.md (web service pitfalls)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-impl-web-services
description: >
  Use when loading WMS/WFS/WMTS layers, configuring XYZ tile sources, or setting up QGIS Server.
  Prevents URI format errors for OGC web services and server misconfiguration.
  Covers WMS/WMTS/WFS/WCS clients, XYZ tiles, QGIS Server, server filters, and OGC API features.
  Keywords: WMS, WFS, WMTS, WCS, XYZ tiles, OGC, QGIS Server, web service, tile layer, OGC API.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- WMS/WMTS client: URI format, layer capabilities, styles, CRS
- WFS client: URI format, version differences (1.0/1.1/2.0), paging
- WCS client: URI format, coverage access
- XYZ tile layers: URI format with {z}/{x}/{y}, common tile sources
- QGIS Server: setup, configuration, QgsServer, QgsServerInterface
- Server plugins and filters: QgsServerFilter, QgsAccessControlFilter
- QgsServerOgcApi: OGC API features
- Authentication for web services

### Research Sections to Read
From vooronderzoek-qgis-integration.md:
- Section 5: Web Services (lines 920-1188) — ALL web services content
- Section 9: Anti-patterns (lines 1785-end) — web service warnings

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include EXACT URI format strings for WMS, WFS, XYZ
- ALWAYS verify layer capabilities before loading
- Delete the .gitkeep file in the output directory before writing
```

---

### Batch 5

#### Prompt: qgis-impl-3d-visualization

```
## Task: Create the qgis-impl-3d-visualization skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-impl\qgis-impl-3d-visualization\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for Qgs3DMapSettings, 3D renderers, symbols, materials)
3. references/examples.md (3D scene setup, vector/point cloud rendering)
4. references/anti-patterns.md (3D visualization pitfalls)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
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

### Scope (EXACT — do not exceed)
- Qgs3DMapSettings: origin, CRS, terrain settings, background color, skybox
- Terrain providers: QgsDemTerrainSettings, QgsFlatTerrainSettings, QgsMeshTerrainSettings
- Vector 3D renderers: QgsVectorLayer3DRenderer
- Point cloud 3D: QgsPointCloudLayer3DRenderer
- 3D symbols: QgsPoint3DSymbol, QgsLine3DSymbol, QgsPolygon3DSymbol
- Materials: QgsPhongMaterialSettings, QgsGoochMaterialSettings
- Camera and light settings
- Layout integration: QgsLayoutItem3DMap
- Mesh 3D rendering

### Research Sections to Read
From vooronderzoek-qgis-integration.md:
- Section 6: 3D Visualization (lines 1189-1371) — ALL 3D content
- Section 9: Anti-patterns (lines 1785-end) — 3D warnings

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Mark [VERIFY] items from research that need confirmation
- Delete the .gitkeep file in the output directory before writing
```

#### Prompt: qgis-impl-georeferencing

```
## Task: Create the qgis-impl-georeferencing skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-impl\qgis-impl-georeferencing\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for QgsGcpPoint, QgsGcpTransformerInterface, QgsVectorWarper)
3. references/examples.md (georeferencing workflows)
4. references/anti-patterns.md (georeferencing pitfalls)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-impl-georeferencing
description: >
  Use when georeferencing raster images or vector data using ground control points.
  Prevents accuracy loss from insufficient GCPs and wrong transformation type selection.
  Covers GCP management, 7 transformation types, QgsVectorWarper, residual computation, and accuracy assessment.
  Keywords: georeferencing, GCP, ground control point, QgsGcpPoint, transformation, Helmert, polynomial, thin plate spline, world file.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- QgsGcpPoint: source/destination coordinates, enabled state
- 7 transformation types: linear (min 2 GCPs), Helmert (min 2), polynomial 1 (min 3), polynomial 2 (min 6), polynomial 3 (min 10), projective (min 4), thin plate spline (min 1)
- QgsGcpTransformerInterface: creating transformers from GCP lists
- QgsVectorWarper: vector georeferencing
- Raster georeferencing workflow (via Sketcher/Sketcher API)
- Residual computation and accuracy assessment
- World file output (.tfw, .jgw, .pgw)

### Research Sections to Read
From vooronderzoek-qgis-integration.md:
- Section 7: Georeferencing (lines 1372-1549) — ALL georeferencing content
- Section 9: Anti-patterns (lines 1785-end) — georeferencing warnings

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include decision tree: "which transformation type based on GCP count and accuracy needs"
- Delete the .gitkeep file in the output directory before writing
```

#### Prompt: qgis-impl-network-analysis

```
## Task: Create the qgis-impl-network-analysis skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-impl\qgis-impl-network-analysis\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for QgsGraphBuilder, QgsGraphAnalyzer, directors, strategies)
3. references/examples.md (shortest path, service area workflows)
4. references/anti-patterns.md (network analysis pitfalls)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-impl-network-analysis
description: >
  Use when performing routing, shortest path, or service area analysis on road/transport networks.
  Prevents incorrect results from wrong edge direction assumptions and missing cost strategies.
  Covers QgsGraphBuilder, QgsGraphAnalyzer (Dijkstra), QgsVectorLayerDirector, cost strategies, and service areas.
  Keywords: network analysis, routing, shortest path, Dijkstra, service area, QgsGraphBuilder, QgsGraphAnalyzer, road network.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- QgsGraphBuilder: constructing graphs from vector line layers
- QgsVectorLayerDirector: handling edge direction (bidirectional, one-way from field)
- Strategy classes: QgsNetworkDistanceStrategy, QgsNetworkSpeedStrategy
- QgsGraphAnalyzer.dijkstra(): shortest path between two points
- QgsGraphAnalyzer.shortestTree(): shortest path tree from a point
- Service area analysis: finding all reachable vertices within cost threshold
- Building route geometry from graph results
- Processing algorithms: native:shortestpathpointtopoint, native:serviceareafromlayer

### Research Sections to Read
From vooronderzoek-qgis-integration.md:
- Section 8: Network Analysis (lines 1550-1784) — ALL network content
- Section 9: Anti-patterns (lines 1785-end) — network warnings

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include complete shortest path workflow example
- Include service area workflow example
- Delete the .gitkeep file in the output directory before writing
```

---

### Batch 6

#### Prompt: qgis-errors-projections

```
## Task: Create the qgis-errors-projections skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-errors\qgis-errors-projections\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (CRS-related API signatures, error messages)
3. references/examples.md (error reproduction and fix patterns)
4. references/anti-patterns.md (comprehensive CRS anti-pattern catalog)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-errors-projections
description: >
  Use when diagnosing CRS/projection errors, coordinate mismatches, or datum transformation issues.
  Prevents silent data corruption from wrong CRS assignment and coordinate order confusion.
  Covers wrong EPSG selection, lat/lon vs lon/lat, datum warnings, on-the-fly reprojection pitfalls, and fix patterns.
  Keywords: CRS error, projection error, wrong EPSG, coordinate mismatch, datum transformation, lat lon order, reprojection.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Error: Wrong EPSG code selected → data appears in wrong location
- Error: lat/lon vs lon/lat coordinate order confusion (EPSG:4326 axis order)
- Error: Missing datum transformation → shifted coordinates
- Error: On-the-fly reprojection hiding CRS mismatches
- Error: Silent data corruption from assigning wrong CRS to layer
- Error: QgsCoordinateTransform without proper context
- For each error: symptoms, root cause, reproduction, fix pattern
- Debugging workflow: how to diagnose CRS issues

### Research Sections to Read
From vooronderzoek-qgis-architecture.md:
- Section 5: CRS Handling (lines 671-886) — CRS pitfalls subsection
- Section 7: Anti-patterns (lines 931-1001) — CRS-related warnings
From vooronderzoek-qgis-integration.md:
- Section 9: Anti-patterns (lines 1785-end) — projection warnings

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Every error MUST have: symptoms, cause, reproduction steps, fix pattern
- Include a diagnostic flowchart
- Delete the .gitkeep file in the output directory before writing
```

#### Prompt: qgis-errors-data-loading

```
## Task: Create the qgis-errors-data-loading skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-errors\qgis-errors-data-loading\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (layer validity API, provider error methods)
3. references/examples.md (error reproduction and fix patterns)
4. references/anti-patterns.md (comprehensive data loading anti-pattern catalog)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-errors-data-loading
description: >
  Use when diagnosing data loading failures, invalid layers, or provider connection errors.
  Prevents silent failures from unchecked layer validity and incorrect URI formats.
  Covers invalid layer detection, URI format errors, encoding issues, GeoPackage locking, connection failures, and performance.
  Keywords: invalid layer, isValid, data loading error, URI format, encoding, GeoPackage lock, connection failed, provider error.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Error: layer.isValid() returns False → silent failure
- Error: Wrong URI format for provider → layer not loaded
- Error: Character encoding issues in Shapefile .dbf files
- Error: GeoPackage file locking (concurrent access)
- Error: PostGIS connection failures (auth, network, permissions)
- Error: WMS/WFS timeout or capability parsing errors
- Error: Large dataset performance (memory, slow iteration)
- Error: Missing CRS in loaded data
- Error: Shapefile limitations (field name length, file size)
- For each error: symptoms, root cause, reproduction, fix pattern
- Debugging workflow: how to diagnose data loading issues

### Research Sections to Read
From vooronderzoek-qgis-architecture.md:
- Section 3: Layer Architecture (lines 236-376) — validity checking
- Section 4: Data Provider Architecture (lines 377-670) — URI formats, errors
- Section 7: Anti-patterns (lines 931-1001) — data loading warnings
From vooronderzoek-qgis-integration.md:
- Section 9: Anti-patterns (lines 1785-end) — data loading warnings

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Every error MUST have: symptoms, cause, fix pattern
- ALWAYS check isValid() after loading — this is the #1 rule
- Include a diagnostic flowchart
- Delete the .gitkeep file in the output directory before writing
```

---

### Batch 7

#### Prompt: qgis-agents-analysis-orchestrator

```
## Task: Create the qgis-agents-analysis-orchestrator skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-agents\qgis-agents-analysis-orchestrator\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (decision trees, algorithm selection tables)
3. references/examples.md (complete orchestrated analysis workflows)
4. references/anti-patterns.md (orchestration pitfalls)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-agents-analysis-orchestrator
description: >
  Use when planning a spatial analysis workflow, choosing between analysis approaches, or chaining multiple operations.
  Prevents wrong analysis method selection and suboptimal workflow ordering.
  Covers vector vs raster decision trees, processing algorithm selection, workflow chaining, CRS/format guidance, and MCP server integration.
  Keywords: analysis workflow, spatial analysis, which algorithm, vector vs raster, workflow chain, analysis plan, orchestrate.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Decision tree: vector analysis vs raster analysis vs network analysis
- Processing algorithm selection guide by task type
- Workflow chaining patterns: which operations in which order
- Data format recommendations: input format → analysis type → output format
- CRS selection guidance: which CRS for which analysis
- Output format selection: GeoPackage vs Shapefile vs GeoJSON vs PostGIS
- Performance optimization hints (spatial index, feature request flags)
- Reference to existing MCP servers (jjsantos01/qgis_mcp, nkarasiak/qgis-mcp) for runtime integration
- Validation checklist for generated PyQGIS code

### Research Sections to Read
From ALL vooronderzoek documents — focus on anti-pattern sections:
- architecture §7 (lines 931-1001)
- pyqgis §11 (lines 1367-1413)
- processing §8 (lines 1370-1564)
- integration §9 (lines 1785-end)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Decision trees MUST be complete (no "it depends" without criteria)
- This skill is an ORCHESTRATOR — it guides choices, not implementation details
- Delete the .gitkeep file in the output directory before writing
```

#### Prompt: qgis-agents-map-generator

```
## Task: Create the qgis-agents-map-generator skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\QGIS-Claude-Skill-Package\skills\source\qgis-agents\qgis-agents-map-generator\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (symbology selection tables, layout configuration reference)
3. references/examples.md (complete map generation workflows)
4. references/anti-patterns.md (map generation pitfalls)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: qgis-agents-map-generator
description: >
  Use when generating complete maps from data: loading, styling, composing layouts, and exporting.
  Prevents poor cartographic choices and empty map exports.
  Covers the full map pipeline: data loading, symbology selection, labeling, layout composition, atlas generation, and export.
  Keywords: map generation, create map, export map, symbology, labeling, layout, atlas, cartography, map pipeline, PDF export.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Map generation pipeline: data loading → CRS setup → styling → labeling → layout → export
- Symbology selection guidance: which renderer for which data type
- Label placement rules and best practices
- Color scheme selection: sequential, diverging, qualitative
- Print layout best practices: page sizes, margins, item placement
- Atlas generation patterns: coverage layer, expression-based output
- Export format selection: PDF vs SVG vs PNG (when to use which)
- Cartographic conventions: north arrow, scale bar, legend, title, source attribution
- Reference to existing MCP servers for runtime map generation
- Validation checklist for generated maps

### Research Sections to Read
From vooronderzoek-qgis-pyqgis.md:
- Section 10: Symbology and Rendering (lines 1152-1366) — symbology guidance
From vooronderzoek-qgis-integration.md:
- Section 3: Print Layout System (lines 530-761) — layout patterns
- Section 9: Anti-patterns (lines 1785-end) — layout/rendering warnings

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include the complete pipeline as a step-by-step workflow
- Include symbology selection decision tree
- This skill is a GENERATOR — it guides the full pipeline, not just one step
- Delete the .gitkeep file in the output directory before writing
```

---

## Appendix: Skill Directory Structure

```
skills/source/
├── qgis-core/
│   ├── qgis-core-architecture/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── methods.md
│   │       ├── examples.md
│   │       └── anti-patterns.md
│   ├── qgis-core-data-providers/
│   │   ├── SKILL.md
│   │   └── references/
│   └── qgis-core-coordinate-systems/
│       ├── SKILL.md
│       └── references/
├── qgis-syntax/
│   ├── qgis-syntax-pyqgis-api/
│   ├── qgis-syntax-expressions/
│   ├── qgis-syntax-processing-scripts/
│   └── qgis-syntax-plugins/
├── qgis-impl/
│   ├── qgis-impl-vector-analysis/
│   ├── qgis-impl-raster-analysis/
│   ├── qgis-impl-print-layouts/
│   ├── qgis-impl-postgis/
│   ├── qgis-impl-web-services/
│   ├── qgis-impl-3d-visualization/
│   ├── qgis-impl-georeferencing/
│   └── qgis-impl-network-analysis/
├── qgis-errors/
│   ├── qgis-errors-projections/
│   └── qgis-errors-data-loading/
└── qgis-agents/
    ├── qgis-agents-analysis-orchestrator/
    └── qgis-agents-map-generator/
```
