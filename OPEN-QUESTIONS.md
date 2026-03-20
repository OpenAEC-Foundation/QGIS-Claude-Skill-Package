# QGIS Skill Package — OPEN QUESTIONS

## Active Questions

### Q-001: QGIS MCP Server Availability
**Status:** OPEN
**Question:** Are there existing QGIS MCP servers in the community? If so, which ones are mature enough to configure as external MCP servers (similar to Draw.io's dual MCP approach)?
**Action:** Research in Phase B0.5.

### Q-002: QGIS Server vs Desktop Scope
**Status:** OPEN
**Question:** How deep should QGIS Server coverage go? Should it be limited to the web-services impl skill, or warrant its own dedicated skills?
**Action:** Decide in Phase B1 (Raw Masterplan).

### Q-003: Processing Algorithm Coverage
**Status:** OPEN
**Question:** QGIS ships with 400+ processing algorithms. Which algorithm categories deserve dedicated coverage vs. being grouped under vector/raster analysis skills?
**Action:** Research in Phase B2 to determine optimal grouping.

---

### Q-004: Expressions Engine — Standalone Skill or Part of Syntax?
**Status:** OPEN
**Question:** The QGIS expressions engine is substantial (custom functions, field calculator, expression-based labeling/symbology). It's currently planned as `qgis-syntax-expressions`. Is the scope sufficient or should it be split?
**Action:** Evaluate during Phase B2 deep research.

### Q-005: 3D/Mesh/Point Clouds — Combined or Separate?
**Status:** OPEN
**Question:** Currently `qgis-impl-3d-visualization` covers all 3D. Mesh layers and point clouds are growing areas. Should they be combined or get separate treatment?
**Action:** Evaluate during Phase B2 deep research.

---

## Answered Questions

### Q-001: QGIS MCP Server Availability
**Status:** ANSWERED (2026-03-20)
**Answer:** Yes, multiple exist. Primary: `jjsantos01/qgis_mcp` (855★, 15 tools, Beta) and `nkarasiak/qgis-mcp` (51 tools, Alpha, most comprehensive). Also: `gdal-mcp` (MIT, production-ready), `gis-mcp` (95+ tools). Decision: reference these in skills, don't build custom (D-005).
