# Working Code Examples (QGIS Network Analysis)

## Example 1: Shortest Path Between Two Points (Full Workflow)

```python
from qgis.analysis import (
    QgsGraphBuilder, QgsVectorLayerDirector,
    QgsNetworkDistanceStrategy, QgsGraphAnalyzer
)
from qgis.core import (
    QgsProject, QgsPointXY, QgsGeometry, QgsPoint,
    QgsVectorLayer, QgsFeature, QgsField
)
from qgis.PyQt.QtCore import QVariant

# Load road network
roads = QgsProject.instance().mapLayersByName("roads")[0]

# Configure director: all edges bidirectional
director = QgsVectorLayerDirector(
    roads, -1, '', '', '',
    QgsVectorLayerDirector.DirectionBoth
)

# Cost = geometric distance
director.addStrategy(QgsNetworkDistanceStrategy())

# Define start and end points
start_point = QgsPointXY(5.1214, 52.0907)
end_point = QgsPointXY(4.8952, 52.3702)

# Build graph
builder = QgsGraphBuilder(roads.crs())
tied_points = director.makeGraph(builder, [start_point, end_point])
graph = builder.graph()

# Find vertex IDs (ALWAYS use tied points, not originals)
start_id = graph.findVertex(tied_points[0])
end_id = graph.findVertex(tied_points[1])

# Run Dijkstra (criterion 0 = distance)
(tree, cost) = QgsGraphAnalyzer.dijkstra(graph, start_id, 0)

# ALWAYS check reachability before reconstruction
if tree[end_id] == -1:
    print("ERROR: Destination is unreachable")
else:
    # Reconstruct route (backwards from destination)
    route_points = [graph.vertex(end_id).point()]
    current = end_id
    while current != start_id:
        edge = graph.edge(tree[current])
        current = edge.fromVertex()
        route_points.insert(0, graph.vertex(current).point())

    # Create route geometry
    route_geom = QgsGeometry.fromPolyline(
        [QgsPoint(p.x(), p.y()) for p in route_points]
    )

    # Create output layer
    route_layer = QgsVectorLayer(
        f"LineString?crs={roads.crs().authid()}", "Route", "memory"
    )
    prov = route_layer.dataProvider()
    prov.addAttributes([QgsField("cost", QVariant.Double)])
    route_layer.updateFields()

    feat = QgsFeature()
    feat.setGeometry(route_geom)
    feat.setAttributes([cost[end_id]])
    prov.addFeature(feat)
    route_layer.updateExtents()

    QgsProject.instance().addMapLayer(route_layer)
    print(f"Route found: cost = {cost[end_id]:.1f}")
```

---

## Example 2: One-Way Streets with Speed-Based Routing

```python
from qgis.analysis import (
    QgsGraphBuilder, QgsVectorLayerDirector,
    QgsNetworkSpeedStrategy, QgsGraphAnalyzer
)
from qgis.core import QgsProject, QgsPointXY

roads = QgsProject.instance().mapLayersByName("roads")[0]

# Direction from 'oneway' field
oneway_idx = roads.fields().indexOf('oneway')
director = QgsVectorLayerDirector(
    roads,
    oneway_idx,
    'yes',                                  # Forward only
    'reverse',                              # Reverse only
    'no',                                   # Both directions
    QgsVectorLayerDirector.DirectionBoth    # Default for unmatched
)

# Cost = travel time (speed in km/h)
speed_idx = roads.fields().indexOf('maxspeed')
director.addStrategy(QgsNetworkSpeedStrategy(
    speed_idx,
    50.0,               # Default 50 km/h when field is NULL
    1000.0 / 3600.0     # km/h to m/s
))

# Build and solve
start = QgsPointXY(5.1214, 52.0907)
end = QgsPointXY(4.8952, 52.3702)

builder = QgsGraphBuilder(roads.crs())
tied = director.makeGraph(builder, [start, end])
graph = builder.graph()

sid = graph.findVertex(tied[0])
eid = graph.findVertex(tied[1])

(tree, cost) = QgsGraphAnalyzer.dijkstra(graph, sid, 0)

if tree[eid] == -1:
    print("No route found (check one-way restrictions)")
else:
    print(f"Travel time: {cost[eid]:.1f} seconds")
```

---

## Example 3: Service Area with Boundary Interpolation

```python
from qgis.analysis import (
    QgsGraphBuilder, QgsVectorLayerDirector,
    QgsNetworkDistanceStrategy, QgsGraphAnalyzer
)
from qgis.core import (
    QgsProject, QgsPointXY, QgsGeometry, QgsPoint,
    QgsVectorLayer, QgsFeature
)

roads = QgsProject.instance().mapLayersByName("roads")[0]

director = QgsVectorLayerDirector(
    roads, -1, '', '', '',
    QgsVectorLayerDirector.DirectionBoth
)
director.addStrategy(QgsNetworkDistanceStrategy())

origin = QgsPointXY(5.1214, 52.0907)
builder = QgsGraphBuilder(roads.crs())
tied = director.makeGraph(builder, [origin])
graph = builder.graph()
origin_id = graph.findVertex(tied[0])

(tree, cost) = QgsGraphAnalyzer.dijkstra(graph, origin_id, 0)

threshold = 2000.0  # 2 km service area

# Interior: fully reachable vertices
interior = []
for vid in range(graph.vertexCount()):
    if cost[vid] <= threshold and tree[vid] != -1:
        interior.append(graph.vertex(vid).point())

# Boundary: interpolated points where edges cross the threshold
boundary = []
for vid in range(graph.vertexCount()):
    if cost[vid] > threshold and tree[vid] != -1:
        edge = graph.edge(tree[vid])
        from_v = edge.fromVertex()
        if cost[from_v] < threshold:
            ratio = (threshold - cost[from_v]) / (cost[vid] - cost[from_v])
            p1 = graph.vertex(from_v).point()
            p2 = graph.vertex(vid).point()
            bp = QgsPointXY(
                p1.x() + ratio * (p2.x() - p1.x()),
                p1.y() + ratio * (p2.y() - p1.y())
            )
            boundary.append(bp)

# Create point layer from all reachable points
all_points = interior + boundary
point_layer = QgsVectorLayer(
    f"Point?crs={roads.crs().authid()}", "Service Area Points", "memory"
)
prov = point_layer.dataProvider()
for pt in all_points:
    feat = QgsFeature()
    feat.setGeometry(QgsGeometry.fromPointXY(pt))
    prov.addFeature(feat)

point_layer.updateExtents()
QgsProject.instance().addMapLayer(point_layer)
print(f"Service area: {len(interior)} interior + {len(boundary)} boundary points")
```

---

## Example 4: Multi-Criteria Analysis (Distance vs Time)

```python
from qgis.analysis import (
    QgsGraphBuilder, QgsVectorLayerDirector,
    QgsNetworkDistanceStrategy, QgsNetworkSpeedStrategy,
    QgsGraphAnalyzer
)
from qgis.core import QgsProject, QgsPointXY

roads = QgsProject.instance().mapLayersByName("roads")[0]

director = QgsVectorLayerDirector(
    roads, -1, '', '', '',
    QgsVectorLayerDirector.DirectionBoth
)

# Criterion 0: distance
director.addStrategy(QgsNetworkDistanceStrategy())

# Criterion 1: travel time
speed_idx = roads.fields().indexOf('speed_kmh')
director.addStrategy(QgsNetworkSpeedStrategy(speed_idx, 50.0, 1000.0 / 3600.0))

start = QgsPointXY(5.1214, 52.0907)
end = QgsPointXY(4.8952, 52.3702)

builder = QgsGraphBuilder(roads.crs())
tied = director.makeGraph(builder, [start, end])
graph = builder.graph()

sid = graph.findVertex(tied[0])
eid = graph.findVertex(tied[1])

# Shortest by distance
(tree_dist, cost_dist) = QgsGraphAnalyzer.dijkstra(graph, sid, 0)

# Fastest by travel time
(tree_time, cost_time) = QgsGraphAnalyzer.dijkstra(graph, sid, 1)

if tree_dist[eid] != -1:
    print(f"Shortest route: {cost_dist[eid]:.1f} meters")
if tree_time[eid] != -1:
    print(f"Fastest route: {cost_time[eid]:.1f} seconds")
```

---

## Example 5: Processing Algorithm — Shortest Path

```python
import processing

roads = QgsProject.instance().mapLayersByName("roads")[0]

result = processing.run("native:shortestpathpointtopoint", {
    'INPUT': roads,
    'STRATEGY': 0,                   # 0=Shortest, 1=Fastest
    'DIRECTION_FIELD': '',
    'VALUE_FORWARD': '',
    'VALUE_BACKWARD': '',
    'VALUE_BOTH': '',
    'DEFAULT_DIRECTION': 2,          # Both
    'SPEED_FIELD': '',
    'DEFAULT_SPEED': 50.0,
    'TOLERANCE': 0.0,
    'START_POINT': '5.1214,52.0907 [EPSG:4326]',
    'END_POINT': '4.8952,52.3702 [EPSG:4326]',
    'OUTPUT': 'TEMPORARY_OUTPUT'
})

route_layer = result['OUTPUT']
QgsProject.instance().addMapLayer(route_layer)
```

---

## Example 6: Processing Algorithm — Service Area from Layer

```python
import processing

roads = QgsProject.instance().mapLayersByName("roads")[0]
facilities = QgsProject.instance().mapLayersByName("hospitals")[0]

result = processing.run("native:serviceareafromlayer", {
    'INPUT': roads,
    'STRATEGY': 0,                   # Shortest
    'DIRECTION_FIELD': 'oneway',
    'VALUE_FORWARD': 'F',
    'VALUE_BACKWARD': 'T',
    'VALUE_BOTH': 'B',
    'DEFAULT_DIRECTION': 2,
    'SPEED_FIELD': '',
    'DEFAULT_SPEED': 50.0,
    'TOLERANCE': 0.0,
    'START_POINTS': facilities,
    'TRAVEL_COST': 5000.0,           # 5 km
    'OUTPUT': 'TEMPORARY_OUTPUT'
})

service_layer = result['OUTPUT']
QgsProject.instance().addMapLayer(service_layer)
```

---

## Example 7: Shortest Path Tree Visualization

```python
from qgis.analysis import (
    QgsGraphBuilder, QgsVectorLayerDirector,
    QgsNetworkDistanceStrategy, QgsGraphAnalyzer
)
from qgis.core import (
    QgsProject, QgsPointXY, QgsGeometry,
    QgsVectorLayer, QgsFeature, QgsField
)
from qgis.PyQt.QtCore import QVariant

roads = QgsProject.instance().mapLayersByName("roads")[0]

director = QgsVectorLayerDirector(
    roads, -1, '', '', '',
    QgsVectorLayerDirector.DirectionBoth
)
director.addStrategy(QgsNetworkDistanceStrategy())

origin = QgsPointXY(5.1214, 52.0907)
builder = QgsGraphBuilder(roads.crs())
tied = director.makeGraph(builder, [origin])
graph = builder.graph()
origin_id = graph.findVertex(tied[0])

# Get shortest path tree as a graph
tree_graph = QgsGraphAnalyzer.shortestTree(graph, origin_id, 0)

# Create line layer from tree edges
tree_layer = QgsVectorLayer(
    f"LineString?crs={roads.crs().authid()}", "Shortest Path Tree", "memory"
)
prov = tree_layer.dataProvider()
prov.addAttributes([QgsField("edge_id", QVariant.Int)])
tree_layer.updateFields()

for eid in range(tree_graph.edgeCount()):
    edge = tree_graph.edge(eid)
    from_pt = tree_graph.vertex(edge.fromVertex()).point()
    to_pt = tree_graph.vertex(edge.toVertex()).point()
    line = QgsGeometry.fromPolylineXY([from_pt, to_pt])

    feat = QgsFeature()
    feat.setGeometry(line)
    feat.setAttributes([eid])
    prov.addFeature(feat)

tree_layer.updateExtents()
QgsProject.instance().addMapLayer(tree_layer)
print(f"Tree has {tree_graph.edgeCount()} edges")
```
