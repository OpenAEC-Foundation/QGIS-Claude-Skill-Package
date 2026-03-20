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

# qgis-agents-analysis-orchestrator

## Quick Reference

This skill is an **orchestrator** — it guides analysis method selection and workflow ordering. It does NOT contain implementation details. For implementation, refer to the specific skill indicated by the decision trees.

### When to Use This Skill

| Trigger | Action |
|---------|--------|
| "Analyze spatial data" | Start at Decision Tree 1: Analysis Type |
| "Which algorithm should I use?" | Go to Algorithm Selection Guide |
| "Chain multiple operations" | Go to Workflow Patterns |
| "What format should I use?" | Go to Format Selection |
| "Which CRS for my analysis?" | Go to CRS Selection |
| "Connect Claude to QGIS" | Go to MCP Integration |

---

## Critical Warnings

**NEVER** start coding before completing the workflow plan. ALWAYS determine: (1) analysis type, (2) CRS, (3) input format, (4) algorithm chain, (5) output format.

**NEVER** mix vector and raster operations without explicit conversion steps. ALWAYS include rasterize/vectorize as a named step.

**NEVER** chain algorithms with mismatched CRS. ALWAYS reproject to a common CRS as the FIRST step.

**NEVER** use geographic CRS (EPSG:4326) for distance or area calculations. ALWAYS reproject to a projected CRS first.

**NEVER** assume third-party provider algorithms (GRASS, SAGA) are available. ALWAYS check availability with `QgsApplication.processingRegistry().algorithmById()` and fall back to `native:` algorithms.

**NEVER** hardcode file paths. ALWAYS use `QgsProcessing.TEMPORARY_OUTPUT` or `'memory:'` for intermediate results.

**ALWAYS** wrap `processing.run()` in try/except for `QgsProcessingException`.

**ALWAYS** check `layer.isValid()` after loading any layer.

**ALWAYS** create spatial indexes (`native:createspatialindex`) before spatial queries on large datasets.

---

## Decision Tree 1: Analysis Type Selection

```
What is your primary question?
│
├─ "How are features distributed spatially?"
│  └─ VECTOR ANALYSIS → Decision Tree 2
│
├─ "What is the value at this location?" / "How does a surface vary?"
│  └─ RASTER ANALYSIS → Decision Tree 3
│
├─ "What is the shortest/fastest route?" / "What areas are reachable?"
│  └─ NETWORK ANALYSIS → Decision Tree 4
│
├─ "How do two datasets relate spatially?"
│  ├─ Both datasets are vector → VECTOR OVERLAY → Decision Tree 2
│  ├─ Both datasets are raster → RASTER CELL STATISTICS → Decision Tree 3
│  └─ One vector + one raster → HYBRID ANALYSIS:
│     ├─ Extract raster values at vector locations → `native:rastersampling`
│     ├─ Summarize raster within vector zones → `native:zonalstatisticsfb`
│     └─ Convert vector to raster first → `gdal:rasterize` then Decision Tree 3
│
├─ "Create a continuous surface from point samples?"
│  └─ INTERPOLATION:
│     ├─ Sparse, irregular points → `native:tininterpolation`
│     ├─ Dense, regular points → `native:idwinterpolation`
│     └─ Event density visualization → `native:heatmapkerneldensityestimation`
│
└─ "Classify or cluster features?"
   └─ CLUSTERING:
      ├─ Known number of groups → `native:kmeansclustering`
      ├─ Unknown groups, density-based → `native:dbscanclustering`
      └─ Spatiotemporal data → `native:stdbscanclustering`
```

---

## Decision Tree 2: Vector Analysis Method

```
What vector operation do you need?
│
├─ PROXIMITY ANALYSIS
│  ├─ Buffer around features → `native:buffer`
│  ├─ Distance between all pairs → `native:distancematrix`
│  ├─ Nearest feature connection → `native:distancetonearesthublines`
│  ├─ Shortest line between features → `native:shortestline`
│  └─ Select within distance → `native:extractwithindistance`
│
├─ OVERLAY ANALYSIS
│  ├─ Keep area in BOTH layers → `native:intersection`
│  ├─ Keep area in EITHER layer → `native:union`
│  ├─ Keep area in A but NOT B → `native:difference`
│  ├─ Cut A to shape of B → `native:clip`
│  └─ Keep area in A OR B but NOT both → `native:symmetricaldifference`
│
├─ SPATIAL JOIN
│  ├─ Join by shared location → `native:joinattributesbylocation`
│  ├─ Join by location with stats → `native:joinattributesbylocationsummary`
│  ├─ Join by nearest feature → `native:joinattributesbynearest`
│  └─ Join by matching field value → `native:joinattributesbyfieldvalue`
│
├─ AGGREGATION
│  ├─ Merge geometries by attribute → `native:dissolve`
│  ├─ Aggregate with expressions → `native:aggregate`
│  ├─ Count points in polygons → `native:countpointsinpolygon`
│  └─ Sum line lengths in polygons → `native:sumlinelengths`
│
├─ GEOMETRY TRANSFORMATION
│  ├─ Simplify (reduce vertices) → `native:simplifygeometries`
│  ├─ Convert polygon to line → `native:polygonstolines`
│  ├─ Convert line to polygon → `native:linestopolygons`
│  ├─ Multi-part to single → `native:multiparttosingleparts`
│  ├─ Fix invalid geometries → `native:fixgeometries`
│  ├─ Calculate centroids → `native:centroids`
│  └─ Convex hull → `native:convexhull`
│
└─ SELECTION / EXTRACTION
   ├─ By attribute value → `native:extractbyattribute`
   ├─ By expression → `native:extractbyexpression`
   ├─ By spatial relationship → `native:extractbylocation`
   └─ Random sample → `native:randomextract`
```

---

## Decision Tree 3: Raster Analysis Method

```
What raster operation do you need?
│
├─ TERRAIN ANALYSIS (from DEM)
│  ├─ Slope angle → `native:slope`
│  ├─ Aspect direction → `native:aspect`
│  ├─ Hillshade visualization → `native:hillshade`
│  ├─ Terrain ruggedness → `native:ruggednessindex`
│  ├─ Fill sinks (hydrology) → `native:fillsinks`
│  └─ Relief rendering → `native:relief`
│
├─ RASTER CALCULATION
│  ├─ Band math / map algebra → `native:rastercalculator`
│  ├─ Reclassify values → `native:reclassifybytable`
│  ├─ Rescale values → `native:rescaleraster`
│  └─ Fill NoData cells → `native:fillnodata`
│
├─ RASTER STATISTICS
│  ├─ Global statistics → `native:rasterlayerstatistics`
│  ├─ Zonal stats per polygon → `native:zonalstatisticsfb`
│  ├─ Zonal histogram → `native:zonalhistogram`
│  ├─ Sample at point locations → `native:rastersampling`
│  └─ Cell statistics across stack → `native:cellstatistics`
│
└─ RASTER COMPARISON
   ├─ Boolean AND across layers → `native:rasterbooleanand`
   ├─ Boolean OR across layers → `native:rasterbooleanor`
   └─ Frequency analysis → `native:equaltofrequency`
```

---

## Decision Tree 4: Network Analysis Method

```
What network question do you have?
│
├─ "Shortest path between two points"
│  └─ `native:shortestpathpointtopoint`
│
├─ "Shortest path from point to multiple destinations"
│  └─ `native:shortestpathpointtolayer`
│
├─ "Shortest path from multiple origins to one point"
│  └─ `native:shortestpathlayertopoint`
│
├─ "What area is reachable within X minutes/meters?"
│  ├─ From a single point → `native:serviceareafrompoint`
│  └─ From multiple points → `native:serviceareafromlayer`
│
└─ PREREQUISITES (ALWAYS complete these first):
   ├─ Network layer MUST be a line layer
   ├─ Configure direction field for one-way streets
   ├─ Configure speed/cost field for weighted analysis
   └─ NEVER assume bidirectional — check direction attributes
```

---

## CRS Selection Guide

| Analysis Type | CRS Requirement | Recommended |
|---------------|----------------|-------------|
| Distance measurement | Projected CRS with meter units | UTM zone for study area |
| Area calculation | Equal-area projection | Country-specific (e.g., EPSG:28992 for NL) |
| Angle/direction | Conformal projection | UTM or local state plane |
| Global analysis | Geographic CRS acceptable | EPSG:4326 (display only) |
| Web map output | Web Mercator | EPSG:3857 |
| Overlay (multi-layer) | ALL layers in SAME projected CRS | Reproject all to target first |

### CRS Selection Decision

```
Where is your study area?
│
├─ Single country/region
│  └─ Use the national projected CRS (e.g., NL=EPSG:28992, UK=EPSG:27700)
│
├─ Crosses UTM zones (narrow east-west)
│  └─ Use the UTM zone covering the majority of the area
│
├─ Continental or global
│  ├─ Area calculations → Equal-area (e.g., EPSG:6933 World Equal Area)
│  ├─ Distance calculations → Equidistant projection
│  └─ Display only → EPSG:4326 or EPSG:3857
│
└─ ALWAYS verify with:
   crs = QgsCoordinateReferenceSystem("EPSG:28992")
   assert crs.isValid(), "CRS is invalid"
```

---

## Output Format Selection Guide

| Criterion | GeoPackage (.gpkg) | Shapefile (.shp) | GeoJSON (.geojson) | PostGIS |
|-----------|-------------------|-------------------|---------------------|---------|
| Multi-layer support | YES | NO | NO | YES |
| Field name length | Unlimited | 10 chars max | Unlimited | Unlimited |
| File size limit | None practical | 2 GB | Memory-bound | None |
| CRS storage | Full WKT | .prj (limited) | EPSG:4326 only | Full |
| NULL geometry | YES | NO | YES | YES |
| Concurrent access | Single-writer | Single-writer | N/A | Multi-user |
| Web transfer | No | No | YES | No |
| **Recommended for** | Default choice | Legacy compat | Web APIs | Enterprise |

### Format Decision

```
What is the output purpose?
│
├─ Intermediate / temporary result
│  └─ Use 'memory:' or QgsProcessing.TEMPORARY_OUTPUT
│
├─ Final file output
│  ├─ Single dataset → GeoPackage (ALWAYS default choice)
│  ├─ Multiple related layers → GeoPackage with layername parameter
│  ├─ Web API / JavaScript → GeoJSON
│  └─ Legacy system requirement → Shapefile (truncate field names to 10 chars)
│
├─ Multi-user / enterprise
│  └─ PostGIS (see qgis-impl-postgis skill)
│
└─ NEVER use Shapefile as default — ALWAYS prefer GeoPackage
```

---

## Workflow Patterns

### Pattern 1: Standard Analysis Chain

```
1. LOAD      → Load input layers, check isValid()
2. VALIDATE  → Fix geometries (native:fixgeometries)
3. REPROJECT → Reproject all to common CRS (native:reprojectlayer)
4. INDEX     → Create spatial index (native:createspatialindex)
5. ANALYZE   → Run analysis algorithm(s)
6. EXPORT    → Save to target format
```

### Pattern 2: Multi-Layer Overlay

```
1. LOAD      → Load all input layers
2. VALIDATE  → Fix geometries on ALL layers
3. REPROJECT → Reproject ALL to common projected CRS
4. INDEX     → Create spatial index on ALL layers
5. OVERLAY   → Run overlay operation (intersection/union/difference)
6. CLEAN     → Remove slivers, fix topology
7. EXPORT    → Save result
```

### Pattern 3: Raster-Vector Hybrid

```
1. LOAD      → Load raster + vector layers
2. REPROJECT → Reproject vector to match raster CRS
3. EXTRACT   → Sample raster at vector locations (native:rastersampling)
              OR Zonal statistics (native:zonalstatisticsfb)
4. ANALYZE   → Further vector analysis on enriched data
5. EXPORT    → Save result
```

### Pattern 4: Iterative Processing (Batch)

```python
# ALWAYS use processing.runAndLoadResults() for final output only
# ALWAYS use 'memory:' for intermediate results
layers = QgsProject.instance().mapLayersByName("input")
for layer in layers:
    if not layer.isValid():
        continue
    try:
        buffered = processing.run("native:buffer", {
            'INPUT': layer,
            'DISTANCE': 100,
            'OUTPUT': 'memory:'
        })['OUTPUT']
        # Chain next operation...
    except QgsProcessingException as e:
        QgsMessageLog.logMessage(f"Failed: {e}", "Analysis")
```

---

## Performance Optimization Checklist

| Step | When | How |
|------|------|-----|
| Spatial index | Before ANY spatial query | `native:createspatialindex` |
| Limit attributes | When only some fields needed | `QgsFeatureRequest().setSubsetOfAttributes(['name'], layer.fields())` |
| Skip geometry | When only attributes needed | `QgsFeatureRequest().setFlags(QgsFeatureRequest.NoGeometry)` |
| Limit extent | When only part of layer needed | `QgsFeatureRequest().setFilterRect(extent)` |
| Memory layers | For intermediate results | `'OUTPUT': 'memory:'` |
| Feature batching | When editing many features | Use `layer.dataProvider().addFeatures()` not per-feature adds |

---

## MCP Server Integration

Two existing MCP servers enable Claude to control a running QGIS instance:

### jjsantos01/qgis_mcp (855+ stars)

- **Purpose**: Direct QGIS control from Claude Desktop
- **Capabilities**: Load layers, run processing algorithms, manage projects, create layouts
- **Setup**: Install via QGIS Plugin Manager + configure Claude Desktop MCP settings
- **URL**: https://github.com/jjsantos01/qgis_mcp

### nkarasiak/qgis-mcp (51+ tools)

- **Purpose**: Comprehensive QGIS MCP with 51 tools
- **Capabilities**: Layer management, styling, processing, spatial queries, project management
- **URL**: https://github.com/nkarasiak/qgis-mcp

### When to Use MCP vs PyQGIS Scripts

```
Is QGIS currently running and accessible?
│
├─ YES → Use MCP server for interactive control
│  ├─ Layer manipulation → MCP tools
│  ├─ Visual feedback needed → MCP (sees map canvas)
│  └─ Complex multi-step analysis → MCP + this orchestrator skill
│
└─ NO → Generate standalone PyQGIS scripts
   ├─ Headless processing → Use qgis.core initialization
   ├─ Batch processing → Standalone script with QgsApplication
   └─ Plugin development → Follow plugin skill patterns
```

---

## Code Validation Checklist

ALWAYS verify generated PyQGIS code against this checklist before presenting it:

- [ ] **Layer validity**: Every `QgsVectorLayer` / `QgsRasterLayer` creation is followed by `isValid()` check
- [ ] **CRS consistency**: All layers in the same operation share a CRS, or explicit reprojection is included
- [ ] **CRS for measurements**: Distance/area calculations use a projected CRS, NEVER geographic
- [ ] **Transform context**: Every `QgsCoordinateTransform` includes `QgsProject.instance().transformContext()`
- [ ] **Edit sessions**: Feature modifications use `with edit(layer):` context manager
- [ ] **Error handling**: `processing.run()` is wrapped in try/except
- [ ] **Algorithm existence**: Third-party algorithms are checked with `algorithmById()` before use
- [ ] **No GUI in threads**: `processAlgorithm()` uses `feedback` object, NEVER `QMessageBox` or `iface`
- [ ] **Temp outputs**: Intermediate results use `'memory:'` or `QgsProcessing.TEMPORARY_OUTPUT`
- [ ] **No hardcoded paths**: Paths use `os.path.join()` or `pathlib.Path`, NEVER backslashes
- [ ] **Spatial index**: Large datasets have `native:createspatialindex` before spatial operations
- [ ] **Feature request flags**: `NoGeometry` flag used when geometry is not needed
- [ ] **NULL geometry check**: `geom.isNull()` checked before geometry operations
- [ ] **Expression validation**: `hasParserError()` checked after `QgsExpression` creation
- [ ] **Memory layer updates**: `updateExtents()` called after adding features to memory layers
- [ ] **Field updates**: `updateFields()` called after modifying field structure

---

## Reference Links

- [references/methods.md](references/methods.md) — Algorithm selection methods and workflow construction patterns
- [references/examples.md](references/examples.md) — Complete workflow examples for common analysis scenarios
- [references/anti-patterns.md](references/anti-patterns.md) — Workflow-level anti-patterns and planning mistakes

### Official Sources

- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/
- https://docs.qgis.org/latest/en/docs/user_manual/processing/
- https://qgis.org/pyqgis/master/
- https://github.com/jjsantos01/qgis_mcp
- https://github.com/nkarasiak/qgis-mcp
