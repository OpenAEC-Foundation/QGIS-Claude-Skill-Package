# API Signatures Reference (QGIS Network Analysis)

## QgsVectorLayerDirector

Determines edge direction rules for graph construction from a vector line layer.

```python
QgsVectorLayerDirector(
    source: QgsFeatureSource,      # Line layer to build graph from
    directionFieldId: int,         # Field index for direction (-1 = use default for all)
    directDirectionValue: str,     # Attribute value meaning "forward direction"
    reverseDirectionValue: str,    # Attribute value meaning "reverse direction"
    bothDirectionValue: str,       # Attribute value meaning "both directions"
    defaultDirection: int          # Default when field value does not match any above
)
```

### Direction Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `QgsVectorLayerDirector.DirectionForward` | 1 | Edge follows digitized direction |
| `QgsVectorLayerDirector.DirectionBackward` | 2 | Edge goes against digitized direction |
| `QgsVectorLayerDirector.DirectionBoth` | 3 | Edge is bidirectional |

### Methods

```python
director.addStrategy(strategy: QgsNetworkStrategy) -> None
    # Add a cost strategy. First added = criterion 0, second = criterion 1, etc.

director.makeGraph(
    builder: QgsGraphBuilder,
    additionalPoints: list[QgsPointXY]
) -> list[QgsPointXY]
    # Build graph. Returns tied (snapped) points on the network.
    # Snapped points correspond 1:1 with input additionalPoints.
```

---

## QgsNetworkDistanceStrategy

Cost strategy based on geometric edge length.

```python
QgsNetworkDistanceStrategy()
    # No parameters. Cost = edge length in CRS units.
```

---

## QgsNetworkSpeedStrategy

Cost strategy based on travel time derived from a speed attribute.

```python
QgsNetworkSpeedStrategy(
    fieldId: int,          # Field index containing speed values
    defaultValue: float,   # Default speed when field value is NULL or invalid
    toMetricFactor: float  # Conversion factor to metric units (e.g., 1000.0/3600.0 for km/h to m/s)
)
```

**Conversion factors:**

| Speed Unit | toMetricFactor | Explanation |
|------------|---------------|-------------|
| km/h | `1000.0 / 3600.0` | Converts km/h to m/s |
| mph | `1609.344 / 3600.0` | Converts mph to m/s |
| m/s | `1.0` | Already in metric units |

---

## QgsGraphBuilder

Constructs a `QgsGraph` from director-provided edges.

```python
QgsGraphBuilder(
    crs: QgsCoordinateReferenceSystem,   # CRS for the graph
    otfEnabled: bool = True,             # On-the-fly reprojection
    topologyTolerance: float = 0.0,      # Snapping tolerance
    ellipsoidID: str = "WGS84"           # Ellipsoid for distance calculations
)
```

### Methods

```python
builder.graph() -> QgsGraph
    # Returns the built graph. Call AFTER director.makeGraph().
```

---

## QgsGraph

In-memory graph structure with vertices and edges.

### Methods

```python
graph.vertexCount() -> int
    # Number of vertices in the graph.

graph.edgeCount() -> int
    # Number of edges in the graph.

graph.findVertex(point: QgsPointXY) -> int
    # Find vertex ID closest to point. Returns -1 if not found.

graph.vertex(id: int) -> QgsGraphVertex
    # Get vertex by ID.

graph.edge(id: int) -> QgsGraphEdge
    # Get edge by ID.
```

---

## QgsGraphVertex

A vertex in the graph.

### Methods

```python
vertex.point() -> QgsPointXY
    # Coordinates of this vertex.

vertex.incomingEdges() -> list[int]
    # List of edge IDs arriving at this vertex.

vertex.outgoingEdges() -> list[int]
    # List of edge IDs leaving this vertex.
```

---

## QgsGraphEdge

An edge in the graph connecting two vertices.

### Methods

```python
edge.fromVertex() -> int
    # ID of the source vertex.

edge.toVertex() -> int
    # ID of the destination vertex.

edge.cost(strategyIndex: int) -> float
    # Cost of traversing this edge for the given strategy criterion.
```

---

## QgsGraphAnalyzer

Static methods for graph analysis algorithms.

### dijkstra()

```python
QgsGraphAnalyzer.dijkstra(
    graph: QgsGraph,
    startVertexIdx: int,
    criterionNum: int       # Strategy index (0, 1, ...)
) -> tuple[list[int], list[float]]
    # Returns (tree, cost):
    #   tree[i]  = edge ID of incoming edge on shortest path to vertex i
    #              (-1 means vertex i is unreachable or is the start vertex)
    #   cost[i]  = total cost from start to vertex i
    #              (inf or max float for unreachable vertices)
```

### shortestTree()

```python
QgsGraphAnalyzer.shortestTree(
    graph: QgsGraph,
    startVertexIdx: int,
    criterionNum: int
) -> QgsGraph
    # Returns a NEW QgsGraph containing only the shortest path tree edges.
    # The returned graph has the same vertex positions but only tree edges.
```

---

## Processing Algorithm Parameters

### native:shortestpathpointtopoint

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | vector line layer | Road network |
| `STRATEGY` | enum | 0=Shortest, 1=Fastest |
| `DIRECTION_FIELD` | field name | Direction attribute (empty = all bidirectional) |
| `VALUE_FORWARD` | string | Forward direction value |
| `VALUE_BACKWARD` | string | Backward direction value |
| `VALUE_BOTH` | string | Both directions value |
| `DEFAULT_DIRECTION` | enum | 0=Forward, 1=Backward, 2=Both |
| `SPEED_FIELD` | field name | Speed attribute (for Fastest strategy) |
| `DEFAULT_SPEED` | float | Default speed in km/h |
| `TOLERANCE` | float | Topology tolerance |
| `START_POINT` | point | Origin as `"x,y [CRS]"` |
| `END_POINT` | point | Destination as `"x,y [CRS]"` |
| `OUTPUT` | sink | Output line layer |

### native:serviceareafrompoint

| Parameter | Type | Description |
|-----------|------|-------------|
| `INPUT` | vector line layer | Road network |
| `STRATEGY` | enum | 0=Shortest, 1=Fastest |
| `DIRECTION_FIELD` | field name | Direction attribute |
| `VALUE_FORWARD` | string | Forward direction value |
| `VALUE_BACKWARD` | string | Backward direction value |
| `VALUE_BOTH` | string | Both directions value |
| `DEFAULT_DIRECTION` | enum | 0=Forward, 1=Backward, 2=Both |
| `SPEED_FIELD` | field name | Speed attribute |
| `DEFAULT_SPEED` | float | Default speed in km/h |
| `TOLERANCE` | float | Topology tolerance |
| `START_POINT` | point | Origin as `"x,y [CRS]"` |
| `TRAVEL_COST` | float | Maximum cost (distance in meters or time in seconds) |
| `OUTPUT` | sink | Output line layer (reachable network segments) |

### native:serviceareafromlayer

Same as `native:serviceareafrompoint` but replaces `START_POINT` with:

| Parameter | Type | Description |
|-----------|------|-------------|
| `START_POINTS` | vector point layer | Multiple origin points |
