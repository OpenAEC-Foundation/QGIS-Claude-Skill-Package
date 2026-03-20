---
name: qgis-impl-network-analysis
description: >
  Use when performing routing, shortest path, or service area analysis on road/transport networks.
  Prevents incorrect results from wrong edge direction assumptions and missing cost strategies.
  Covers QgsGraphBuilder, QgsGraphAnalyzer (Dijkstra), QgsVectorLayerDirector, cost strategies, and service areas.
  Keywords: network analysis, routing, shortest path, Dijkstra, service area, QgsGraphBuilder, QgsGraphAnalyzer, road network.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-impl-network-analysis

## Quick Reference

### Core Classes (`qgis.analysis`)

| Class | Purpose |
|-------|---------|
| `QgsVectorLayerDirector` | Controls edge direction rules for a line layer |
| `QgsNetworkDistanceStrategy` | Cost = geometric edge length |
| `QgsNetworkSpeedStrategy` | Cost = travel time (from speed field) |
| `QgsGraphBuilder` | Constructs a `QgsGraph` from a directed layer |
| `QgsGraphAnalyzer` | Static methods: `dijkstra()`, `shortestTree()` |
| `QgsGraph` | In-memory graph with vertices and edges |

### Workflow Steps

| Step | Class/Method | Output |
|------|-------------|--------|
| 1. Set direction rules | `QgsVectorLayerDirector(layer, ...)` | Director object |
| 2. Add cost strategy | `director.addStrategy(strategy)` | Strategy index (0, 1, ...) |
| 3. Build graph | `director.makeGraph(builder, points)` | Tied (snapped) points |
| 4. Get graph object | `builder.graph()` | `QgsGraph` |
| 5. Find vertex IDs | `graph.findVertex(tied_point)` | Integer vertex ID |
| 6. Run Dijkstra | `QgsGraphAnalyzer.dijkstra(graph, start, criterion)` | `(tree, cost)` tuple |
| 7. Reconstruct path | Walk `tree[]` backwards from destination | List of `QgsPointXY` |

### Processing Algorithm IDs

| Algorithm | ID |
|-----------|----|
| Shortest path (point to point) | `native:shortestpathpointtopoint` |
| Shortest path (point to layer) | `native:shortestpathpointtolayer` |
| Shortest path (layer to point) | `native:shortestpathlayertopoint` |
| Service area (from point) | `native:serviceareafrompoint` |
| Service area (from layer) | `native:serviceareafromlayer` |

---

## Critical Warnings

**NEVER** assume all edges are bidirectional -- road networks contain one-way streets. ALWAYS configure `QgsVectorLayerDirector` with the correct direction field and values.

**ALWAYS** check `tree[destination_id] == -1` before path reconstruction -- this value means the destination is unreachable from the source vertex.

**ALWAYS** use `graph.findVertex(tied_point)` with the tied (snapped) point returned by `makeGraph()`, NOT the original input point -- the original point does not exist in the graph.

**NEVER** skip adding a strategy before building the graph -- without at least one strategy, the graph has no edge costs and `dijkstra()` produces meaningless results.

**ALWAYS** use a projected CRS (meters) for distance-based analysis -- geographic CRS (degrees) produces incorrect distance calculations.

**NEVER** reconstruct a path using `graph.vertex(current).incomingEdges()` -- use `tree[current]` from the Dijkstra result instead, which gives the specific edge in the shortest path tree.

---

## Decision Tree

```
Need network analysis?
├── Simple point-to-point route?
│   ├── Need full control over graph → Use QgsGraphBuilder + QgsGraphAnalyzer
│   └── Need quick result → Use processing.run("native:shortestpathpointtopoint")
├── Route from one point to many destinations?
│   └── Use processing.run("native:shortestpathpointtolayer")
├── Route from many origins to one destination?
│   └── Use processing.run("native:shortestpathlayertopoint")
├── Service area (reachability)?
│   ├── Single origin → Use processing.run("native:serviceareafrompoint")
│   ├── Multiple origins → Use processing.run("native:serviceareafromlayer")
│   └── Need boundary interpolation → Use QgsGraphAnalyzer.dijkstra() manually
├── Cost criterion?
│   ├── Distance (length) → QgsNetworkDistanceStrategy()
│   ├── Travel time → QgsNetworkSpeedStrategy(field_index, default_speed, factor)
│   └── Multiple criteria → Add multiple strategies, use criterion index in dijkstra()
└── Direction handling?
    ├── All bidirectional → directionFieldId = -1
    ├── One-way from attribute → directionFieldId = field index
    └── All one-way (forward) → directionFieldId = -1, defaultDirection = DirectionForward
```

---

## Essential Patterns

### Pattern 1: Complete Shortest Path Workflow

```python
from qgis.analysis import (
    QgsGraphBuilder, QgsVectorLayerDirector,
    QgsNetworkDistanceStrategy, QgsGraphAnalyzer
)
from qgis.core import QgsProject, QgsPointXY, QgsGeometry, QgsPoint

# 1. Load road network layer
roads = QgsProject.instance().mapLayersByName("roads")[0]

# 2. Configure director (bidirectional, no direction field)
director = QgsVectorLayerDirector(
    roads,
    -1,                                    # No direction field
    '', '', '',                            # Direction values (unused)
    QgsVectorLayerDirector.DirectionBoth   # Default direction
)

# 3. Add distance-based cost strategy
director.addStrategy(QgsNetworkDistanceStrategy())

# 4. Define origin and destination
start_point = QgsPointXY(5.1214, 52.0907)   # Utrecht
end_point = QgsPointXY(4.8952, 52.3702)     # Amsterdam

# 5. Build graph (points get snapped to nearest network edge)
builder = QgsGraphBuilder(roads.crs())
tied_points = director.makeGraph(builder, [start_point, end_point])
graph = builder.graph()

# 6. Get vertex IDs from tied (snapped) points
start_id = graph.findVertex(tied_points[0])
end_id = graph.findVertex(tied_points[1])

# 7. Run Dijkstra from start vertex (criterion 0 = distance)
(tree, cost) = QgsGraphAnalyzer.dijkstra(graph, start_id, 0)

# 8. Check reachability
if tree[end_id] == -1:
    raise ValueError("Destination is unreachable from origin")

# 9. Reconstruct path (walk backwards from destination)
route_points = [graph.vertex(end_id).point()]
current = end_id
while current != start_id:
    edge = graph.edge(tree[current])
    current = edge.fromVertex()
    route_points.insert(0, graph.vertex(current).point())

# 10. Build route geometry
route_geom = QgsGeometry.fromPolyline(
    [QgsPoint(p.x(), p.y()) for p in route_points]
)
print(f"Route distance: {cost[end_id]:.1f} map units")
```

### Pattern 2: One-Way Street Handling

```python
# Find the direction field index
oneway_idx = roads.fields().indexOf('oneway')

director = QgsVectorLayerDirector(
    roads,
    oneway_idx,                            # Field containing direction info
    'F',                                   # Value meaning "forward only"
    'T',                                   # Value meaning "reverse only"
    'B',                                   # Value meaning "both directions"
    QgsVectorLayerDirector.DirectionBoth   # Default for unmatched values
)
```

**Direction constants:**

| Constant | Meaning |
|----------|---------|
| `QgsVectorLayerDirector.DirectionForward` | Digitized direction only |
| `QgsVectorLayerDirector.DirectionBackward` | Reverse of digitized direction only |
| `QgsVectorLayerDirector.DirectionBoth` | Both directions (bidirectional) |

### Pattern 3: Travel Time Cost Strategy

```python
speed_field_index = roads.fields().indexOf('speed_kmh')

strategy = QgsNetworkSpeedStrategy(
    speed_field_index,
    50.0,              # Default speed when field value is NULL
    1000.0 / 3600.0    # Conversion factor: km/h to m/s
)
director.addStrategy(strategy)
```

### Pattern 4: Multiple Cost Criteria

```python
# Criterion 0: distance
director.addStrategy(QgsNetworkDistanceStrategy())

# Criterion 1: travel time
speed_idx = roads.fields().indexOf('speed_kmh')
director.addStrategy(QgsNetworkSpeedStrategy(speed_idx, 50.0, 1000.0 / 3600.0))

# Build graph once
builder = QgsGraphBuilder(roads.crs())
tied = director.makeGraph(builder, [start_point, end_point])
graph = builder.graph()
sid = graph.findVertex(tied[0])
eid = graph.findVertex(tied[1])

# Shortest by distance (criterion 0)
(tree_dist, cost_dist) = QgsGraphAnalyzer.dijkstra(graph, sid, 0)

# Fastest by travel time (criterion 1)
(tree_time, cost_time) = QgsGraphAnalyzer.dijkstra(graph, sid, 1)
```

### Pattern 5: Service Area Analysis

```python
(tree, cost) = QgsGraphAnalyzer.dijkstra(graph, start_id, 0)

threshold = 5000.0  # 5000 meters (or seconds, depending on strategy)

# Collect fully reachable vertices (interior)
reachable = []
for vid in range(graph.vertexCount()):
    if cost[vid] <= threshold and tree[vid] != -1:
        reachable.append(graph.vertex(vid).point())

# Interpolate boundary points (edges that cross the threshold)
boundary = []
for vid in range(graph.vertexCount()):
    if cost[vid] > threshold and tree[vid] != -1:
        edge = graph.edge(tree[vid])
        from_v = edge.fromVertex()
        if cost[from_v] < threshold:
            ratio = (threshold - cost[from_v]) / (cost[vid] - cost[from_v])
            p1 = graph.vertex(from_v).point()
            p2 = graph.vertex(vid).point()
            interpolated = QgsPointXY(
                p1.x() + ratio * (p2.x() - p1.x()),
                p1.y() + ratio * (p2.y() - p1.y())
            )
            boundary.append(interpolated)

# Combine all reachable points for convex hull / polygon
all_points = reachable + boundary
```

### Pattern 6: Shortest Path Tree

```python
# Get the full shortest path tree from a single origin
tree_graph = QgsGraphAnalyzer.shortestTree(graph, start_id, 0)

# tree_graph is a new QgsGraph containing only edges in the shortest path tree
# Extract all edges for visualization
for eid in range(tree_graph.edgeCount()):
    edge = tree_graph.edge(eid)
    from_pt = tree_graph.vertex(edge.fromVertex()).point()
    to_pt = tree_graph.vertex(edge.toVertex()).point()
    # Create line geometry for each edge
    line = QgsGeometry.fromPolylineXY([from_pt, to_pt])
```

---

## Common Operations

### Using Processing Algorithms for Routing

```python
import processing

# Shortest path: point to point
result = processing.run("native:shortestpathpointtopoint", {
    'INPUT': roads,
    'STRATEGY': 0,              # 0=Shortest, 1=Fastest
    'DIRECTION_FIELD': 'oneway',
    'VALUE_FORWARD': 'F',
    'VALUE_BACKWARD': 'T',
    'VALUE_BOTH': 'B',
    'DEFAULT_DIRECTION': 2,     # 0=Forward, 1=Backward, 2=Both
    'SPEED_FIELD': 'speed_kmh',
    'DEFAULT_SPEED': 50.0,
    'TOLERANCE': 0.0,
    'START_POINT': '5.1214,52.0907 [EPSG:4326]',
    'END_POINT': '4.8952,52.3702 [EPSG:4326]',
    'OUTPUT': 'TEMPORARY_OUTPUT'
})
route_layer = result['OUTPUT']
```

### Service Area via Processing

```python
result = processing.run("native:serviceareafrompoint", {
    'INPUT': roads,
    'STRATEGY': 0,              # 0=Shortest, 1=Fastest
    'DIRECTION_FIELD': '',
    'VALUE_FORWARD': '',
    'VALUE_BACKWARD': '',
    'VALUE_BOTH': '',
    'DEFAULT_DIRECTION': 2,
    'SPEED_FIELD': '',
    'DEFAULT_SPEED': 50.0,
    'TOLERANCE': 0.0,
    'START_POINT': '5.1214,52.0907 [EPSG:4326]',
    'TRAVEL_COST': 5000.0,      # Distance or time threshold
    'OUTPUT': 'TEMPORARY_OUTPUT'
})
```

### Creating a Route Layer from Graph Results

```python
from qgis.core import QgsVectorLayer, QgsFeature, QgsField
from qgis.PyQt.QtCore import QVariant

# Create memory layer for the route
route_layer = QgsVectorLayer(
    f"LineString?crs={roads.crs().authid()}",
    "Route",
    "memory"
)
provider = route_layer.dataProvider()
provider.addAttributes([
    QgsField("distance", QVariant.Double),
])
route_layer.updateFields()

# Add route feature
feat = QgsFeature()
feat.setGeometry(route_geom)
feat.setAttributes([cost[end_id]])
provider.addFeature(feat)
route_layer.updateExtents()

QgsProject.instance().addMapLayer(route_layer)
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for QgsGraphBuilder, QgsGraphAnalyzer, QgsVectorLayerDirector, strategy classes
- [references/examples.md](references/examples.md) -- Complete working examples for routing and service area workflows
- [references/anti-patterns.md](references/anti-patterns.md) -- Common mistakes in network analysis with corrections

### Official Sources

- https://docs.qgis.org/3.34/en/docs/pyqgis_developer_cookbook/network_analysis.html
- https://qgis.org/pyqgis/3.34/analysis/QgsGraphAnalyzer.html
- https://qgis.org/pyqgis/3.34/analysis/QgsGraphBuilder.html
- https://qgis.org/pyqgis/3.34/analysis/QgsVectorLayerDirector.html
