# QGIS Skill Package — Skill Index

> 19 deterministic skills for QGIS spatial data analysis, map creation, and geoprocessing with Claude.
> Each skill is self-contained, English-only, and follows ALWAYS/NEVER deterministic language.

---

## Skills by Category

### Core (3 skills) — Foundation knowledge

| # | Skill | Description | Lines |
|---|-------|-------------|-------|
| 1 | [qgis-core-architecture](skills/source/qgis-core/qgis-core-architecture/) | Qt-based architecture, 5 core modules, project lifecycle, layer model, provider system | 383 |
| 2 | [qgis-core-data-providers](skills/source/qgis-core/qgis-core-data-providers/) | 12+ data providers, URI construction patterns, GeoPackage best practices | 332 |
| 3 | [qgis-core-coordinate-systems](skills/source/qgis-core/qgis-core-coordinate-systems/) | CRS creation, coordinate transforms, datum transformations, distance measurement | 302 |

---

### Syntax (4 skills) — How to write PyQGIS code

| # | Skill | Description | Lines |
|---|-------|-------------|-------|
| 4 | [qgis-syntax-pyqgis-api](skills/source/qgis-syntax/qgis-syntax-pyqgis-api/) | Feature CRUD, geometry ops, spatial indexing, background tasks, symbology | 361 |
| 5 | [qgis-syntax-expressions](skills/source/qgis-syntax/qgis-syntax-expressions/) | Expression engine, field calculator, data-defined properties, custom functions | 295 |
| 6 | [qgis-syntax-processing-scripts](skills/source/qgis-syntax/qgis-syntax-processing-scripts/) | processing.run(), custom algorithms, parameter types, batch processing | 330 |
| 7 | [qgis-syntax-plugins](skills/source/qgis-syntax/qgis-syntax-plugins/) | Plugin structure, lifecycle, UI integration, Qt Designer, publishing | 316 |

---

### Implementation (8 skills) — GIS workflow recipes

| # | Skill | Description | Lines |
|---|-------|-------------|-------|
| 8 | [qgis-impl-vector-analysis](skills/source/qgis-impl/qgis-impl-vector-analysis/) | Spatial queries, overlay operations, joins, field calculations | 365 |
| 9 | [qgis-impl-raster-analysis](skills/source/qgis-impl/qgis-impl-raster-analysis/) | Map algebra, DEM terrain analysis, raster renderers, GDAL algorithms | 404 |
| 10 | [qgis-impl-print-layouts](skills/source/qgis-impl/qgis-impl-print-layouts/) | Layout composition, atlas generation, PDF/SVG/image export | 396 |
| 11 | [qgis-impl-postgis](skills/source/qgis-impl/qgis-impl-postgis/) | PostGIS connection, layer loading, SQL execution, authentication | 396 |
| 12 | [qgis-impl-web-services](skills/source/qgis-impl/qgis-impl-web-services/) | WMS/WFS/WMTS clients, XYZ tiles, QGIS Server, OGC API | 462 |
| 13 | [qgis-impl-3d-visualization](skills/source/qgis-impl/qgis-impl-3d-visualization/) | 3D scenes, terrain, 3D symbols, materials, point cloud rendering | 440 |
| 14 | [qgis-impl-georeferencing](skills/source/qgis-impl/qgis-impl-georeferencing/) | GCP management, 7 transformation types, vector/raster georeferencing | 388 |
| 15 | [qgis-impl-network-analysis](skills/source/qgis-impl/qgis-impl-network-analysis/) | Graph building, Dijkstra routing, service area analysis | 356 |

---

### Errors (2 skills) — Diagnosis and anti-patterns

| # | Skill | Description | Lines |
|---|-------|-------------|-------|
| 16 | [qgis-errors-projections](skills/source/qgis-errors/qgis-errors-projections/) | CRS errors, coordinate mismatches, datum issues, diagnostic flowchart | 491 |
| 17 | [qgis-errors-data-loading](skills/source/qgis-errors/qgis-errors-data-loading/) | Invalid layers, URI errors, encoding, GeoPackage locking, performance | 438 |

---

### Agents (2 skills) — Intelligent orchestration

| # | Skill | Description | Lines |
|---|-------|-------------|-------|
| 18 | [qgis-agents-analysis-orchestrator](skills/source/qgis-agents/qgis-agents-analysis-orchestrator/) | Analysis workflow planning, algorithm selection, code validation | 413 |
| 19 | [qgis-agents-map-generator](skills/source/qgis-agents/qgis-agents-map-generator/) | Full map pipeline: data → styling → layout → export | 456 |

---

## Statistics

| Metric | Value |
|--------|-------|
| Total skills | 19 |
| Total SKILL.md lines | 7,324 |
| Average lines per skill | 386 |
| Reference files per skill | 3 (methods.md, examples.md, anti-patterns.md) |
| Total reference files | 57 |
| Target QGIS version | 3.44 LTR (with 4.0 compatibility notes) |

---

## Skill Dependencies

```
agents/analysis-orchestrator ──> ALL impl/* skills (routes by analysis type)
agents/map-generator         ──> impl/print-layouts (map production)

impl/* skills                ──> core/architecture (QGIS data model)
impl/* skills                ──> core/data-providers (data loading)
impl/* skills                ──> core/coordinate-systems (CRS handling)
impl/* skills                ──> syntax/pyqgis-api (API patterns)

syntax/pyqgis-api            ──> core/architecture (class hierarchy)
syntax/expressions           ──> core/architecture (expression context)
syntax/processing-scripts    ──> core/architecture (processing framework)
syntax/plugins               ──> core/architecture (plugin interface)

errors/projections           ──> core/coordinate-systems (CRS diagnostics)
errors/data-loading          ──> core/data-providers (provider diagnostics)
```

---

## How to Use This Package

1. **Running a spatial analysis:** Start with `qgis-agents-analysis-orchestrator` (#18). It routes to the correct implementation skill automatically.

2. **Performing a specific GIS workflow:** Go directly to the relevant `impl/` skill (#8-15). Each contains complete PyQGIS code, processing algorithms, and working examples.

3. **Looking up PyQGIS API patterns:** Use `qgis-syntax-pyqgis-api` (#4) for the class/method reference, or `qgis-syntax-expressions` (#5) for the expression engine.

4. **Debugging CRS/projection issues:** Start with `qgis-errors-projections` (#16) for systematic diagnosis.

5. **Generating publication-quality maps:** Use `qgis-agents-map-generator` (#19) for the full map production pipeline.

---

**Version:** 1.0 — All 19 skills complete and validated.
