# QGIS Skill Package — REQUIREMENTS

## Quality Guarantees

Every skill in this package MUST meet the following criteria:

### 1. Format-correct
- SKILL.md < 500 lines
- Valid YAML frontmatter with `name` and `description`
- `name`: kebab-case, max 64 characters, prefix `qgis-`
- `description`: max 1024 characters, includes trigger keywords
- `references/` directory with: methods.md, examples.md, anti-patterns.md

### 2. Code-accurate
- All PyQGIS code examples MUST be syntactically correct Python
- Processing algorithm calls MUST use correct algorithm IDs
- Data provider URIs MUST follow correct format
- CRS definitions MUST use valid EPSG codes or WKT

### 3. Version-explicit
- Target: QGIS 3.x (latest LTR and current release)
- PyQGIS: QGIS 3.x Python API
- Note breaking changes between QGIS versions where applicable
- Specify minimum QGIS version for features introduced after 3.0

### 4. Anti-pattern-free
- Every skill MUST document known mistakes in references/anti-patterns.md
- Common CRS transformation errors documented
- Data loading pitfalls documented
- Processing algorithm misuse documented

### 5. Deterministic
- Use ALWAYS/NEVER language
- No hedging words: "might", "consider", "often", "usually"
- Decision trees for conditional logic

### 6. Self-contained
- Each skill works independently without requiring other skills
- All necessary context included within the skill
- Cross-references to related skills are informational only

### 7. English-only
- All skill content in English
- Claude reads English, responds in any language
- No Dutch, German, or other languages in skill files

## Per-Category Requirements

### Core Skills
- MUST cover the complete QGIS data model (layers, features, geometries)
- MUST include the provider architecture (OGR, GDAL, PostgreSQL, WMS/WFS)
- MUST document CRS handling and transformations

### Syntax Skills
- MUST include complete PyQGIS class/method references
- MUST show both minimal and full examples
- MUST document expression syntax and functions
- MUST cover Processing script structure and registration

### Implementation Skills
- MUST produce working PyQGIS code
- MUST include realistic analysis examples (not just toy examples)
- MUST document required data formats and expected outputs

### Error Skills
- MUST include real error scenarios with reproduction steps
- MUST provide fix patterns for every documented error

### Agent Skills
- MUST include decision trees for analysis type selection
- MUST validate generated code against PyQGIS API rules
