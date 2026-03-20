# Anti-Patterns (QGIS Network Analysis)

## 1. Using Original Points Instead of Tied Points

```python
# WRONG: Original point does not exist as a vertex in the graph
start_point = QgsPointXY(5.1214, 52.0907)
builder = QgsGraphBuilder(roads.crs())
tied_points = director.makeGraph(builder, [start_point])
graph = builder.graph()
start_id = graph.findVertex(start_point)  # Returns wrong vertex or -1!

# CORRECT: ALWAYS use the tied (snapped) point returned by makeGraph()
start_id = graph.findVertex(tied_points[0])
```

**WHY**: `makeGraph()` snaps input points to the nearest edge on the network. The original point coordinates do not match any vertex in the graph. Using the original point with `findVertex()` returns an incorrect or nonexistent vertex ID.

---

## 2. Skipping Reachability Check Before Path Reconstruction

```python
# WRONG: Crashes if destination is unreachable
(tree, cost) = QgsGraphAnalyzer.dijkstra(graph, start_id, 0)
route = [graph.vertex(end_id).point()]
current = end_id
while current != start_id:
    edge = graph.edge(tree[current])  # tree[end_id] is -1 → invalid edge!
    current = edge.fromVertex()
    route.insert(0, graph.vertex(current).point())

# CORRECT: ALWAYS check tree[end_id] first
(tree, cost) = QgsGraphAnalyzer.dijkstra(graph, start_id, 0)
if tree[end_id] == -1:
    print("Destination unreachable")
else:
    route = [graph.vertex(end_id).point()]
    current = end_id
    while current != start_id:
        edge = graph.edge(tree[current])
        current = edge.fromVertex()
        route.insert(0, graph.vertex(current).point())
```

**WHY**: When `tree[vertex_id] == -1`, the vertex is unreachable from the start. Passing `-1` to `graph.edge()` causes an index error or undefined behavior.

---

## 3. Assuming All Edges Are Bidirectional

```python
# WRONG: Ignores one-way streets in road networks
director = QgsVectorLayerDirector(
    roads, -1, '', '', '',
    QgsVectorLayerDirector.DirectionBoth
)

# CORRECT: Use the direction field when the data has one-way information
oneway_idx = roads.fields().indexOf('oneway')
if oneway_idx >= 0:
    director = QgsVectorLayerDirector(
        roads, oneway_idx,
        'yes', 'reverse', 'no',
        QgsVectorLayerDirector.DirectionBoth
    )
else:
    director = QgsVectorLayerDirector(
        roads, -1, '', '', '',
        QgsVectorLayerDirector.DirectionBoth
    )
```

**WHY**: Road networks contain one-way streets. Treating all edges as bidirectional produces routes that traverse one-way streets in the wrong direction, giving physically impossible results.

---

## 4. Forgetting to Add a Strategy

```python
# WRONG: No strategy added — graph edges have no cost values
director = QgsVectorLayerDirector(roads, -1, '', '', '',
    QgsVectorLayerDirector.DirectionBoth)
# director.addStrategy() is missing!
builder = QgsGraphBuilder(roads.crs())
tied = director.makeGraph(builder, points)
graph = builder.graph()
(tree, cost) = QgsGraphAnalyzer.dijkstra(graph, sid, 0)  # Criterion 0 has no data

# CORRECT: ALWAYS add at least one strategy before building the graph
director.addStrategy(QgsNetworkDistanceStrategy())
```

**WHY**: Without a strategy, there are no cost values for edges. Dijkstra cannot compute meaningful shortest paths when edge costs are undefined.

---

## 5. Using Geographic CRS for Distance Analysis

```python
# WRONG: CRS is EPSG:4326 (degrees) — distances are in degrees, not meters
roads_4326 = QgsProject.instance().mapLayersByName("roads_wgs84")[0]
builder = QgsGraphBuilder(roads_4326.crs())  # Costs will be in degrees

# CORRECT: Reproject to a projected CRS first, or use a projected layer
roads_projected = QgsProject.instance().mapLayersByName("roads_utm")[0]
builder = QgsGraphBuilder(roads_projected.crs())  # Costs in meters
```

**WHY**: `QgsNetworkDistanceStrategy` calculates edge length in CRS units. For EPSG:4326, units are degrees, making distance values meaningless for routing. ALWAYS use a projected CRS (meters) for distance-based analysis.

---

## 6. Using incomingEdges() for Path Reconstruction

```python
# WRONG: incomingEdges() returns ALL incoming edges, not just the shortest path edge
current = end_id
while current != start_id:
    incoming = graph.vertex(current).incomingEdges()
    edge = graph.edge(incoming[0])  # Picks arbitrary edge, not shortest path edge!
    current = edge.fromVertex()

# CORRECT: Use tree[] from Dijkstra result — it contains the specific shortest path edges
current = end_id
while current != start_id:
    edge = graph.edge(tree[current])  # tree[current] = edge ID in shortest path
    current = edge.fromVertex()
```

**WHY**: A vertex can have multiple incoming edges. `incomingEdges()` returns all of them. Only `tree[vertex_id]` from the Dijkstra result identifies which specific edge belongs to the shortest path.

---

## 7. Wrong Criterion Index in Multi-Strategy Graphs

```python
# WRONG: Strategies added in wrong order, then criterion index confused
director.addStrategy(QgsNetworkSpeedStrategy(speed_idx, 50.0, 1000.0 / 3600.0))  # index 0
director.addStrategy(QgsNetworkDistanceStrategy())                                 # index 1

# Developer thinks criterion 0 is distance, but it is actually time
(tree, cost) = QgsGraphAnalyzer.dijkstra(graph, sid, 0)  # This is TIME, not distance

# CORRECT: Document strategy order and use matching criterion indices
director.addStrategy(QgsNetworkDistanceStrategy())                                 # index 0 = distance
director.addStrategy(QgsNetworkSpeedStrategy(speed_idx, 50.0, 1000.0 / 3600.0))  # index 1 = time

(tree_dist, cost_dist) = QgsGraphAnalyzer.dijkstra(graph, sid, 0)  # Distance
(tree_time, cost_time) = QgsGraphAnalyzer.dijkstra(graph, sid, 1)  # Time
```

**WHY**: The criterion index in `dijkstra()` corresponds to the order strategies were added via `addStrategy()`. Mixing up the order produces results optimized for the wrong metric.

---

## 8. Not Handling Disconnected Networks

```python
# WRONG: Assumes all points are on the same connected component
points = [pointA, pointB, pointC, pointD]
tied = director.makeGraph(builder, points)
graph = builder.graph()

# Batch routing without checking connectivity
for i in range(len(points) - 1):
    sid = graph.findVertex(tied[i])
    eid = graph.findVertex(tied[i + 1])
    (tree, cost) = QgsGraphAnalyzer.dijkstra(graph, sid, 0)
    edge = graph.edge(tree[eid])  # Crashes if eid is on disconnected component!

# CORRECT: ALWAYS check tree[eid] != -1 for every pair
for i in range(len(points) - 1):
    sid = graph.findVertex(tied[i])
    eid = graph.findVertex(tied[i + 1])
    (tree, cost) = QgsGraphAnalyzer.dijkstra(graph, sid, 0)
    if tree[eid] == -1:
        print(f"No path from point {i} to point {i+1}")
        continue
    # ... reconstruct path
```

**WHY**: Real-world road networks are not guaranteed to be fully connected. Islands, disconnected road segments, and one-way restrictions can make certain vertex pairs unreachable.

---

## 9. Incorrect Speed Strategy Conversion Factor

```python
# WRONG: Speed field is in mph but using km/h conversion factor
director.addStrategy(QgsNetworkSpeedStrategy(
    speed_idx, 50.0,
    1000.0 / 3600.0     # This is km/h to m/s, wrong for mph!
))

# CORRECT: Use the right conversion factor for the speed unit
# For mph: 1609.344 / 3600.0
director.addStrategy(QgsNetworkSpeedStrategy(
    speed_idx, 50.0,
    1609.344 / 3600.0   # mph to m/s
))
```

**WHY**: The `toMetricFactor` converts the speed field value to meters per second. Using the wrong factor produces incorrect travel time calculations, often off by a factor of ~1.6.
