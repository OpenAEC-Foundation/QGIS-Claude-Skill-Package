# QGIS Skill Package — DECISIONS

## Architectural Decisions

### D-001: Research-First Methodology
**Date:** 2026-03-20
**Decision:** Use the proven 7-phase research-first methodology from Blender, ERPNext, and Draw.io packages.
**Rationale:** This approach produces higher-quality, deterministic skills because research precedes creation. Proven across 4 prior packages with 150+ skills total.

### D-002: MCP Server Evaluation
**Date:** 2026-03-20
**Decision:** Evaluate existing QGIS MCP servers before deciding build vs. configure.
**Rationale:** The QGIS ecosystem may have existing MCP integrations. Evaluate before committing to a custom build. Decision will be revisited in Phase B0.5.

### D-003: ~19 Skills Preliminary Count
**Date:** 2026-03-20
**Decision:** Target ~19 skills across 5 categories (core: 3, syntax: 4, impl: 8, errors: 2, agents: 2).
**Rationale:** Based on initial QGIS technology landscape analysis. Covers the primary GIS workflows: vector/raster analysis, map production, data integration, and spatial data management. Will be refined in Phase B3 after deep research.

### D-004: Claude-First, No agentskills.io Compliance
**Date:** 2026-03-20
**Decision:** Focus on the Claude ecosystem. No explicit agentskills.io open standard compliance.
**Rationale:** Claude-specific features (allowed-tools, plugin marketplace, progressive disclosure) can be used freely without cross-platform constraints. Same decision as Draw.io package (D-017).

### D-005: No Custom MCP Server for v1.0
**Date:** 2026-03-20
**Decision:** Do NOT build a custom QGIS MCP server. Reference existing servers in skills instead.
**Rationale:** The ecosystem already has strong options: `jjsantos01/qgis_mcp` (855★, 15 tools, Beta, listed in official QGIS Plugin Repository) and `nkarasiak/qgis-mcp` (51 tools, actively developed, supports QGIS 3.28-4.x). Our skill package is complementary (knowledge vs runtime). For v2.0, consider contributing PRs to existing servers rather than building from scratch.

### D-006: Target QGIS 3.44 LTR, Note 4.0 Compatibility
**Date:** 2026-03-20
**Decision:** Target QGIS 3.44 LTR (current stable) as primary version. Include QGIS 4.0 (Qt6) compatibility notes where relevant.
**Rationale:** QGIS 4.0 released 2026-03-06 but is brand new. The LTR (3.44) is what production users run. PyQGIS API is largely preserved in 4.0 but Qt6 changes affect some patterns (e.g., `QMetaType.Type` instead of `QVariant.Type`).

### D-007: Clear Field — No Competing QGIS Skills Exist
**Date:** 2026-03-20
**Decision:** Proceed with full 19-skill package. No existing QGIS/PyQGIS skills found in the Claude ecosystem.
**Rationale:** Searched Claude Skills Hub, anthropics/skills repo, GitHub, OpenAEC Foundation, and community repos. Zero QGIS skills found. Closest related: K-Dense scientific skills (generic geospatial, no PyQGIS), gis-claude-skills (Leaflet only). We have a completely clear field.
