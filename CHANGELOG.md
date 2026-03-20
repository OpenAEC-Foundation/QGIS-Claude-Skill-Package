# Changelog

All notable changes to the QGIS Skill Package will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.0.0] - 2026-03-20

### Added
- 19 deterministic skills across 5 categories (core, syntax, impl, errors, agents)
- 57 reference files (methods.md, examples.md, anti-patterns.md per skill)
- 4 deep research documents (19,000+ words total)
- 3 targeted topic-specific research documents (3D, georeferencing, QGIS Server)
- Definitive masterplan with complete agent prompts for all 19 skills
- Raw masterplan with technology landscape mapping
- MCP server ecosystem research (jjsantos01/qgis_mcp, nkarasiak/qgis-mcp, gdal-mcp, gis-mcp)
- Existing skills ecosystem scan (clear field confirmed)
- INDEX.md with complete skill catalog and dependency graph
- README.md with installation instructions
- GitHub Actions quality workflow
- LICENSE (MIT)

### Core Skills (3)
- `qgis-core-architecture` — QGIS architecture, modules, project lifecycle, layer model
- `qgis-core-data-providers` — 12+ providers, URI formats, GeoPackage best practices
- `qgis-core-coordinate-systems` — CRS handling, transforms, distance measurement

### Syntax Skills (4)
- `qgis-syntax-pyqgis-api` — Feature CRUD, geometry, spatial index, tasks, symbology
- `qgis-syntax-expressions` — Expression engine, field calculator, custom functions
- `qgis-syntax-processing-scripts` — processing.run(), custom algorithms, batch processing
- `qgis-syntax-plugins` — Plugin structure, lifecycle, UI integration, publishing

### Implementation Skills (8)
- `qgis-impl-vector-analysis` — Spatial queries, overlays, joins, calculations
- `qgis-impl-raster-analysis` — Map algebra, terrain analysis, GDAL algorithms
- `qgis-impl-print-layouts` — Layout composition, atlas, PDF/SVG/image export
- `qgis-impl-postgis` — PostGIS connection, SQL execution, authentication
- `qgis-impl-web-services` — WMS/WFS/WMTS, XYZ tiles, QGIS Server
- `qgis-impl-3d-visualization` — 3D scenes, terrain, symbols, materials
- `qgis-impl-georeferencing` — GCP management, 7 transformation types
- `qgis-impl-network-analysis` — Dijkstra routing, service area analysis

### Error Skills (2)
- `qgis-errors-projections` — CRS errors, coordinate mismatches, diagnostic flowchart
- `qgis-errors-data-loading` — Invalid layers, URI errors, encoding, performance

### Agent Skills (2)
- `qgis-agents-analysis-orchestrator` — Analysis planning, algorithm selection, validation
- `qgis-agents-map-generator` — Full map pipeline, symbology guide, cartographic conventions

## [0.1.0] - 2026-03-20

### Added
- Bootstrap workspace with 7-phase methodology
- CLAUDE.md with 8 protocols and QGIS technology scope
- ROADMAP.md with ~19 skill inventory
- WAY_OF_WORK.md with proven research-first methodology
- REQUIREMENTS.md with quality guarantees
- DECISIONS.md with D-001 to D-007
- LESSONS.md with 15 inherited lessons
- SOURCES.md with QGIS primary/secondary/standard sources
- OPEN-QUESTIONS.md tracker
- START-PROMPT.md for session initialization
