# methods.md — Analysis Orchestrator

## Workflow Construction Method

### Step 1: Classify the Analysis Task

Every spatial analysis task falls into one of these categories. ALWAYS classify FIRST before selecting algorithms.

| Category | Input Data | Question Type | Primary Tools |
|----------|-----------|---------------|---------------|
| Vector proximity | Points, lines, polygons | "How close?" / "What's nearby?" | buffer, distance matrix, nearest hub |
| Vector overlay | Two+ polygon layers | "Where do they overlap?" | intersection, union, difference, clip |
| Vector aggregation | Features with shared attributes | "Summarize by group" | dissolve, aggregate, statistics |
| Raster terrain | DEM / elevation model | "What is the slope/aspect?" | slope, aspect, hillshade, ruggedness |
| Raster algebra | Multi-band / multi-layer raster | "Calculate between cells" | raster calculator, cell statistics |
| Raster-vector hybrid | Raster + vector | "Extract raster values for features" | zonal stats, raster sampling |
| Network analysis | Line network | "What is the route/reachability?" | shortest path, service area |
| Interpolation | Point samples | "Create surface from points" | TIN, IDW, heatmap KDE |
| Clustering | Point features | "Find spatial groups" | K-means, DBSCAN, ST-DBSCAN |

### Step 2: Determine CRS Requirements

```python
def select_crs(study_area_country, analysis_type):
    """
    ALWAYS call this before starting analysis.
    Returns the appropriate EPSG code.
    """
    # National projected CRS lookup (common examples)
    national_crs = {
        'NL': 'EPSG:28992',   # Amersfoort / RD New
        'BE': 'EPSG:31370',   # Belgian Lambert 72
        'DE': 'EPSG:25832',   # ETRS89 / UTM 32N
        'UK': 'EPSG:27700',   # British National Grid
        'FR': 'EPSG:2154',    # RGF93 / Lambert-93
        'US': 'EPSG:5070',    # NAD83 / Conus Albers
        'AU': 'EPSG:3577',    # GDA94 / Australian Albers
    }

    if analysis_type == 'web_display':
        return 'EPSG:3857'
    elif analysis_type == 'global_equal_area':
        return 'EPSG:6933'
    elif study_area_country in national_crs:
        return national_crs[study_area_country]
    else:
        # Fall back to UTM zone
        return determine_utm_zone(study_area_centroid)
```

### Step 3: Build the Algorithm Chain

ALWAYS follow this ordering:

1. **Load** — Load all input data
2. **Validate** — Fix geometries, check CRS validity
3. **Reproject** — Bring all layers to common CRS
4. **Index** — Create spatial indexes for large datasets
5. **Pre-process** — Any data cleaning (dissolve, simplify, extract subset)
6. **Analyze** — Core analysis algorithm(s)
7. **Post-process** — Join results, calculate additional fields
8. **Export** — Save to target format

### Step 4: Select Algorithms by Task

#### Proximity Tasks

| Task | Algorithm | Key Parameters |
|------|-----------|---------------|
| Buffer features | `native:buffer` | DISTANCE, SEGMENTS, END_CAP_STYLE, DISSOLVE |
| Multi-ring buffer | `native:multiringconstantbuffer` | DISTANCE, RINGS |
| One-side buffer | `native:singlesidedbuffer` | DISTANCE, SIDE |
| Distance matrix | `native:distancematrix` | INPUT, INPUT_FIELD, TARGET, TARGET_FIELD, MATRIX_TYPE |
| Nearest hub | `native:distancetonearesthublines` | INPUT, HUBS, HUB_FIELD |

#### Overlay Tasks

| Task | Algorithm | Key Parameters |
|------|-----------|---------------|
| Intersection | `native:intersection` | INPUT, OVERLAY, INPUT_FIELDS, OVERLAY_FIELDS |
| Union | `native:union` | INPUT, OVERLAY |
| Difference | `native:difference` | INPUT, OVERLAY |
| Clip | `native:clip` | INPUT, OVERLAY |
| Multi-layer intersection | `native:multiintersection` | INPUT, OVERLAYS |

#### Aggregation Tasks

| Task | Algorithm | Key Parameters |
|------|-----------|---------------|
| Dissolve by field | `native:dissolve` | INPUT, FIELD |
| Aggregate with expressions | `native:aggregate` | INPUT, GROUP_BY, AGGREGATES |
| Count points in polygons | `native:countpointsinpolygon` | POLYGONS, POINTS, FIELD |
| Statistics by category | `native:statisticsbycategories` | INPUT, VALUES_FIELD_NAME, CATEGORIES_FIELD_NAME |

#### Terrain Tasks

| Task | Algorithm | Key Parameters |
|------|-----------|---------------|
| Slope | `native:slope` | INPUT, Z_FACTOR |
| Aspect | `native:aspect` | INPUT |
| Hillshade | `native:hillshade` | INPUT, Z_FACTOR, AZIMUTH, V_ANGLE |
| Fill sinks | `native:fillsinks` | INPUT |

## Workflow Validation Method

After constructing an algorithm chain, ALWAYS validate:

1. **CRS chain**: Verify every algorithm receives layers in compatible CRS
2. **Field propagation**: Verify join/overlay operations propagate needed fields
3. **Geometry type chain**: Verify output geometry type of step N matches expected input of step N+1
4. **Memory management**: Verify intermediate results use `'memory:'`, only final output writes to disk
5. **Error propagation**: Verify every `processing.run()` is wrapped in try/except
