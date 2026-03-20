# Handoff: QGIS-Claude-Skill-Package

> Gegenereerd: 2026-03-20 vanuit Skill-Package-Workflow-Template

## Status
- **Fase:** B0 Bootstrap DONE — docs compleet, skills/ structuur aangemaakt
- **Skills:** 0/19 geschreven
- **GitHub remote:** Nog niet aangemaakt
- **README:** Ontbreekt

## Wat er is
Alle governance docs zijn klaar: CLAUDE.md, ROADMAP.md, INDEX.md, WAY_OF_WORK.md, DECISIONS.md, LESSONS.md, SOURCES.md, REQUIREMENTS.md, CHANGELOG.md, START-PROMPT.md.
De `skills/source/` directory structuur met 19 lege skill mappen staat klaar.

## Wat er moet gebeuren
1. **Fase B1: Raw Masterplan** — Map het complete QGIS/PyQGIS landschap
2. **Fase B2: Deep Research** — 4 research documenten (min. 2000 woorden elk)
3. **Fase B3: Masterplan Refinement** — Skill inventory definitief maken
4. **Fase B4+B5: Skill Creation** — Batches van 3 skills, dependency-aware
5. **Fase B6: Validation** — Quality gates
6. **Fase B7: Publication** — README, LICENSE, GitHub remote, compliance audit

## Batch volgorde
1. core (architecture, data-providers, coordinate-systems) — 3 skills
2. syntax (pyqgis-api, expressions, processing-scripts, plugins) — 4 skills
3. impl deel 1 (vector-analysis, raster-analysis, print-layouts, postgis) — 4 skills
4. impl deel 2 (web-services, 3d-visualization, georeferencing, network-analysis) — 4 skills
5. errors (projections, data-loading) — 2 skills
6. agents (analysis-orchestrator, map-generator) — 2 skills

## Hoe te starten
Open deze map in VS Code, Claude Code, typ:
```
Lees START-PROMPT.md en begin fase B1 (Raw Masterplan).
Focus op PyQGIS API en QGIS Processing framework.
```

## Bijzonderheden
- QGIS 3.34+ LTR als target versie
- PyQGIS API verandert regelmatig — pin versie in SOURCES.md
- Georeferencing skill is ook relevant voor Cross-Tech-AEC package
- From scratch — geen bestaande research basis
