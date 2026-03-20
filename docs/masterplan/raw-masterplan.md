# QGIS Skill Package — Raw Masterplan

## Status

Phase 1 complete. Raw technology landscape mapped, preliminary skill inventory defined.
Date: 2026-03-20

---

## Technology Landscape Summary

### Version Targets
- **Primary target:** QGIS 3.44 LTR (current stable)
- **Secondary:** QGIS 4.0 "Norrkoping" (released 2026-03-06, Qt6 migration)
- **PyQGIS:** Python bindings for QGIS 3.x/4.x via SIP

### PyQGIS API Surface

| Module | Class Count | Key Areas |
|--------|-------------|-----------|
| `qgis.core` | ~200+ | Layers, features, geometry, CRS, symbology, expressions, layouts, processing, providers |
| `qgis.gui` | ~50+ | Map canvas, map tools, attribute table, layer tree |
| `qgis.analysis` | ~30+ | Raster calculator, network analysis, interpolation, georeferencing |
| `qgis._3d` | ~30+ | 3D scene, renderers, symbols, terrain, materials |
| `qgis.server` | ~15+ | QGIS Server, OGC services, filters, access control |
| `qgis.processing` | framework | 300+ built-in algorithms, custom algorithms, providers |

### Data Provider Landscape

| Type | Formats | Provider Keys |
|------|---------|---------------|
| Vector files | Shapefile, GeoPackage, GeoJSON, FlatGeoBuf, KML, DXF, GPX | `ogr` |
| Vector databases | PostGIS, SpatiaLite, MS SQL Server, Oracle Spatial | `postgres`, `spatialite`, `mssql`, `oracle` |
| Vector services | WFS | `WFS` |
| Vector special | Memory layers, delimited text (CSV), virtual layers | `memory`, `delimitedtext`, `virtual` |
| Raster files | GeoTIFF, JPEG2000, COG | `gdal` |
| Raster services | WMS, WMTS, WCS, XYZ tiles | `wms`, `wcs` |
| Raster databases | PostGIS Raster | `postgresraster` |
| Point clouds | LAS, LAZ, COPC, EPT | `pdal`, `copc`, `ept` |
| Mesh | NetCDF, GRIB | `mdal` |
| Vector tiles | MVT | `vectortile` |
| 3D tiles | Cesium 3D Tiles | `cesiumtiles` |

### Processing Framework

- **300+ built-in algorithms** across providers: Native, GDAL, GRASS, SAGA (deprecated), OTB, PDAL
- **Algorithm categories**: vector general, vector geometry, vector overlay, vector selection, vector creation, raster analysis, raster terrain, raster conversion, mesh, interpolation, point cloud, cartography, database, file tools, GPS, network analysis
- **Entry point**: `processing.run("provider:algorithm", {params})`
- **Custom algorithms**: Subclass `QgsProcessingAlgorithm`

### MCP Server Ecosystem (Reference Only — D-005)

| Server | Stars | Tools | Status | Notes |
|--------|-------|-------|--------|-------|
| jjsantos01/qgis_mcp | 855 | 15 | Beta | Dominant, in QGIS Plugin Repository |
| nkarasiak/qgis-mcp | 12 | 51 | Alpha | Most comprehensive, actively developed |
| Wayfinder-Foundry/gdal-mcp | 58 | 11 | Production | MIT, well-engineered |
| mahdin75/gis-mcp | 123 | 95+ | Beta | GIS libraries (Shapely, GeoPandas, Rasterio) |

---

## Developer Workflow Complexity Map

| Tier | Workflows | Complexity |
|------|-----------|------------|
| **Tier 1 (Low)** | Project loading, layer management, basic feature access, settings, memory layers | Simple API, few gotchas |
| **Tier 2 (Medium)** | Geometry operations, CRS transforms, spatial indexing, expressions, raster queries, editing features | More API surface, common mistakes |
| **Tier 3 (High)** | Symbology/rendering, print layouts, plugin development, processing algorithms, network analysis, server development, 3D visualization | Complex class hierarchies, many pitfalls |

---

## Preliminary Skill Inventory (~19 skills)

### Category: core/ (3 skills) — Foundation, no dependencies

| # | Skill Name | Scope | Key Classes | Complexity |
|---|-----------|-------|-------------|------------|
| 1 | `qgis-core-architecture` | QGIS architecture overview; Qt-based design; project lifecycle (`QgsProject`); layer model; provider architecture; plugin system; main thread constraints; QGIS 4.0 Qt6 notes | `QgsProject`, `QgsApplication`, `QgsProviderRegistry` | M |
| 2 | `qgis-core-data-providers` | Data provider system; vector formats (OGR, PostGIS, memory, delimited text); raster formats (GDAL, WMS); URI construction patterns; `QgsDataSourceUri`; provider capabilities; GeoPackage as preferred format | `QgsDataSourceUri`, `QgsVectorLayer`, `QgsRasterLayer`, `QgsProviderRegistry` | M |
| 3 | `qgis-core-coordinate-systems` | CRS handling; `QgsCoordinateReferenceSystem` creation (EPSG, WKT, Proj); `QgsCoordinateTransform`; transform context; datum transformations; `QgsDistanceArea`; common projection errors; EPSG database | `QgsCoordinateReferenceSystem`, `QgsCoordinateTransform`, `QgsCoordinateTransformContext`, `QgsDistanceArea` | M |

### Category: syntax/ (4 skills) — How to write PyQGIS code

| # | Skill Name | Scope | Key Classes | Complexity |
|---|-----------|-------|-------------|------------|
| 4 | `qgis-syntax-pyqgis-api` | PyQGIS scripting patterns; Python console vs standalone scripts; `QgsApplication` initialization; layer CRUD; feature iteration with `QgsFeatureRequest`; editing with `with edit(layer):`; spatial index; memory layers; background tasks (`QgsTask`); threading constraints | `QgsFeature`, `QgsFeatureRequest`, `QgsGeometry`, `QgsSpatialIndex`, `QgsTask`, `QgsVectorLayerUtils` | L |
| 5 | `qgis-syntax-expressions` | QGIS expression engine; `QgsExpression` parsing and evaluation; expression context and scopes; field calculator patterns; expression-based labeling; expression-based symbology; custom expression functions; common expression functions reference | `QgsExpression`, `QgsExpressionContext`, `QgsExpressionContextScope`, `QgsExpressionContextUtils` | M |
| 6 | `qgis-syntax-processing-scripts` | Processing framework; `processing.run()` entry point; algorithm IDs and parameters; custom `QgsProcessingAlgorithm` subclass; parameter types; feedback and progress; chaining algorithms; batch processing; running as background task | `QgsProcessingAlgorithm`, `QgsProcessingProvider`, `QgsProcessingContext`, `QgsProcessingFeedback`, `QgsProcessingParameter*` | L |
| 7 | `qgis-syntax-plugins` | QGIS plugin development; plugin structure (metadata.txt, `__init__.py`, main class); `classFactory`/`initGui`/`unload` lifecycle; UI integration (menus, toolbars, dock widgets); Qt Designer for `.ui` files; plugin resources; Plugin Builder tool; QGIS Plugin Repository publishing | Plugin lifecycle methods, `iface` object, Qt widgets | L |

### Category: impl/ (8 skills) — Implementation patterns

| # | Skill Name | Scope | Key Classes | Complexity |
|---|-----------|-------|-------------|------------|
| 8 | `qgis-impl-vector-analysis` | Vector analysis workflows; spatial queries (intersects, within, contains); buffer/dissolve/clip/union/difference; attribute joins; spatial joins; field calculations; feature selection; vector creation; `QgsVectorFileWriter` | `QgsGeometry`, `QgsSpatialIndex`, `QgsFeatureRequest`, `QgsVectorFileWriter`, processing algorithms (native:buffer, native:clip, etc.) | M |
| 9 | `qgis-impl-raster-analysis` | Raster analysis; `QgsRasterCalculator` (map algebra); raster statistics; DEM analysis (hillshade, slope, aspect, relief); raster sampling; raster classification; raster renderers (single band, multiband, pseudocolor); terrain analysis | `QgsRasterCalculator`, `QgsRasterCalculatorEntry`, `QgsHillshadeFilter`, `QgsSlopeFilter`, `QgsAspectFilter`, `QgsRelief` | M |
| 10 | `qgis-impl-print-layouts` | Print layout composition; `QgsPrintLayout` creation; layout items (map, label, legend, scale bar, north arrow, picture, table); atlas generation; layout export (PDF, SVG, image); page setup; template save/load | `QgsPrintLayout`, `QgsLayoutItemMap`, `QgsLayoutItemLabel`, `QgsLayoutItemLegend`, `QgsLayoutItemScaleBar`, `QgsLayoutExporter` | L |
| 11 | `qgis-impl-postgis` | PostGIS integration; connection via `QgsDataSourceUri`; loading PostGIS layers; spatial queries in PostGIS; executing SQL; connection management; authentication; schema/table discovery; creating tables from QGIS; PostGIS raster | `QgsDataSourceUri`, `QgsAbstractDatabaseProviderConnection`, `QgsProviderRegistry` | M |
| 12 | `qgis-impl-web-services` | OGC web services; WMS/WMTS client (loading tile layers); WFS client (loading feature layers); WCS client; XYZ tile layers; authentication for services; QGIS Server setup and configuration; custom server filters; OGC API features | `wms`/`WFS` providers, `QgsServer`, `QgsServerFilter`, `QgsServerOgcApi` | M |
| 13 | `qgis-impl-3d-visualization` | 3D visualization; `Qgs3DMapSettings` configuration; terrain providers; vector layer 3D renderers; point cloud 3D rendering; 3D symbols (point, line, polygon); materials; camera and light settings; 3D export; layout integration (`QgsLayoutItem3DMap`) | `Qgs3DMapSettings`, `QgsVectorLayer3DRenderer`, `QgsPoint3DSymbol`, `QgsLine3DSymbol`, `QgsPolygon3DSymbol` | M |
| 14 | `qgis-impl-georeferencing` | Georeferencing workflows; GCP point management; transformation algorithms (linear, Helmert, polynomial, thin plate spline); `QgsGcpPoint`; `QgsVectorWarper`; raster georeferencing via Sketcher API; accuracy assessment | `QgsGcpPoint`, `QgsGcpTransformerInterface`, `QgsVectorWarper` | M |
| 15 | `qgis-impl-network-analysis` | Network/graph analysis; `QgsGraphBuilder` for building road networks; `QgsGraphAnalyzer` for Dijkstra shortest path; service area analysis; `QgsVectorLayerDirector`; speed/distance strategies; routing workflows | `QgsGraph`, `QgsGraphAnalyzer`, `QgsGraphBuilder`, `QgsVectorLayerDirector`, `QgsNetworkDistanceStrategy`, `QgsNetworkSpeedStrategy` | M |

### Category: errors/ (2 skills) — Error diagnosis and anti-patterns

| # | Skill Name | Scope | Key Classes | Complexity |
|---|-----------|-------|-------------|------------|
| 16 | `qgis-errors-projections` | CRS/projection errors; "on-the-fly" reprojection pitfalls; wrong EPSG selection; datum transformation warnings; coordinate order confusion (lat/lon vs lon/lat); transformation accuracy issues; silent data corruption from wrong CRS; debugging projection problems | `QgsCoordinateReferenceSystem`, `QgsCoordinateTransform` | M |
| 17 | `qgis-errors-data-loading` | Data loading failures; invalid layer detection (`layer.isValid()`); provider URI format errors; encoding issues; character encoding in Shapefiles; missing CRS; GeoPackage locking; PostGIS connection failures; WMS/WFS timeout handling; large dataset performance | `QgsVectorLayer.isValid()`, provider error messages | M |

### Category: agents/ (2 skills) — Intelligent orchestration

| # | Skill Name | Scope | Key Classes | Complexity |
|---|-----------|-------|-------------|------------|
| 18 | `qgis-agents-analysis-orchestrator` | Decision tree for selecting analysis approach; vector vs raster decision; processing algorithm selection; workflow chaining; data format recommendations; CRS selection guidance; output format selection; performance optimization hints | All analysis-related classes | M |
| 19 | `qgis-agents-map-generator` | Map generation workflow orchestrator; data loading → styling → layout composition → export pipeline; symbology selection guidance; label placement rules; print layout best practices; atlas generation patterns; cartographic conventions | All layout and rendering classes | M |

---

## Dependency Graph

```
Layer 0 (no deps):     core-architecture, core-data-providers, core-coordinate-systems
Layer 1 (core deps):   syntax-pyqgis-api, syntax-expressions, syntax-processing-scripts, syntax-plugins
Layer 2 (syntax deps): impl-vector-analysis, impl-raster-analysis, impl-print-layouts,
                        impl-postgis, impl-web-services, impl-3d-visualization,
                        impl-georeferencing, impl-network-analysis
Layer 3 (impl deps):   errors-projections, errors-data-loading
Layer 4 (all deps):    agents-analysis-orchestrator, agents-map-generator
```

---

## Preliminary Batch Plan (6 batches)

| Batch | Skills | Count | Dependencies |
|-------|--------|-------|-------------|
| 1 | `core-architecture`, `core-data-providers`, `core-coordinate-systems` | 3 | None |
| 2 | `syntax-pyqgis-api`, `syntax-expressions`, `syntax-processing-scripts` | 3 | Batch 1 |
| 3 | `syntax-plugins`, `impl-vector-analysis`, `impl-raster-analysis` | 3 | Batch 1-2 |
| 4 | `impl-print-layouts`, `impl-postgis`, `impl-web-services` | 3 | Batch 1-2 |
| 5 | `impl-3d-visualization`, `impl-georeferencing`, `impl-network-analysis` | 3 | Batch 1-2 |
| 6 | `errors-projections`, `errors-data-loading` | 2 | Batch 1-5 |
| 7 | `agents-analysis-orchestrator`, `agents-map-generator` | 2 | ALL above |

**Total**: 19 skills across 7 batches.

---

## Open Questions for Phase B2 Research

1. **Q-003**: Which processing algorithm categories deserve dedicated coverage vs grouping?
2. **Q-004**: Is the expressions engine scope sufficient for one skill or should it split?
3. **Q-005**: Should 3D/mesh/point clouds be combined or separated?
4. **Symbology/rendering**: Currently spread across impl skills. Should there be a dedicated `syntax-symbology` skill?
5. **Background tasks**: The `QgsTask` API has critical threading constraints. Covered in `syntax-pyqgis-api` or separate?

---

## Research Needed for Phase B2

### Document 1: vooronderzoek-qgis-architecture.md
- QGIS core architecture and data model
- Qt-based design, signal/slot patterns
- Provider architecture
- Main thread constraints
- QGIS 4.0 Qt6 migration impact

### Document 2: vooronderzoek-qgis-pyqgis.md
- PyQGIS API patterns
- Feature access and editing
- Geometry operations
- Expression engine
- Background tasks and threading

### Document 3: vooronderzoek-qgis-processing.md
- Processing framework architecture
- Algorithm categories and IDs
- Custom algorithm development
- Batch processing
- Algorithm chaining

### Document 4: vooronderzoek-qgis-integration.md
- PostGIS integration patterns
- WMS/WFS/WCS client and server
- Data format specifics
- Print layout system
- 3D visualization
- Network analysis
- Georeferencing

---

## Appendix: PyQGIS Developer Cookbook Chapter Map

1. Introduction — Python console, startup, PyQt/SIP
2. Loading Projects — paths, optimization
3. Loading Layers — vector, raster, all providers
4. Table of Contents — layer tree, groups
5. Raster Layers — renderers, querying, editing
6. Vector Layers — attributes, iteration, selection, modification, indexing, symbology
7. Geometry Handling — construction, access, predicates, operations
8. Projections — CRS, transforms
9. Map Canvas — embedding, rubber bands, map tools
10. Map Rendering and Printing — simple render, layouts, export
11. Expressions — parsing, evaluating, filtering, calculating
12. Settings — reading/storing configuration
13. Communicating with User — messages, progress, logging
14. Authentication — auth manager, plugin adaptation
15. Tasks — background processing, QgsTask
16. Plugin Development — structure, snippets, IDE, releasing
17. Processing Plugins — custom algorithms
18. Plugin Layers — QgsPluginLayer
19. Network Analysis — graph building, Dijkstra, service areas
20. QGIS Server — API, deployment, plugins, OGC services
21. Cheat Sheet — quick reference
