# Anti-Patterns (QGIS Print Layouts)

## 1. Exporting Without Setting Map Extent

```python
# WRONG: Map item has no extent set -- produces blank or incorrect output
map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(200, 150, QgsUnitTypes.LayoutMillimeters))
layout.addLayoutItem(map_item)
exporter.exportToPdf("/tmp/output.pdf", settings)  # blank map!

# CORRECT: ALWAYS set the extent before export
map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(200, 150, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(layer.extent())  # set extent BEFORE export
layout.addLayoutItem(map_item)
exporter.exportToPdf("/tmp/output.pdf", settings)
```

**WHY**: A `QgsLayoutItemMap` without an extent defaults to an empty or arbitrary view. The export succeeds (returns `Success`) but the map area is blank or shows the wrong area.

---

## 2. Skipping initializeDefaults()

```python
# WRONG: Layout has no pages -- export produces nothing
layout = QgsPrintLayout(project)
layout.setName("No Pages")
# Missing: layout.initializeDefaults()
exporter = QgsLayoutExporter(layout)
result = exporter.exportToPdf("/tmp/output.pdf", settings)  # empty or error

# CORRECT: ALWAYS call initializeDefaults() after creating a new layout
layout = QgsPrintLayout(project)
layout.initializeDefaults()  # creates default A4 page
layout.setName("With Pages")
```

**WHY**: `QgsPrintLayout()` creates an empty layout with zero pages. Without `initializeDefaults()`, there is no page to render, and exports fail silently or produce empty files.

---

## 3. Forgetting to Add Items to Layout

```python
# WRONG: Item created but never added to layout
map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(200, 150, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(layer.extent())
# Missing: layout.addLayoutItem(map_item)
exporter.exportToPdf("/tmp/output.pdf", settings)  # map not visible!

# CORRECT: ALWAYS call addLayoutItem() after configuring the item
map_item = QgsLayoutItemMap(layout)
map_item.attemptResize(QgsLayoutSize(200, 150, QgsUnitTypes.LayoutMillimeters))
map_item.zoomToExtent(layer.extent())
layout.addLayoutItem(map_item)  # register with layout
```

**WHY**: Layout items exist as standalone objects until explicitly added. Without `addLayoutItem()`, the item is not part of the layout and does not appear in exports.

---

## 4. Atlas Iteration Without beginRender()

```python
# WRONG: Calling next() without beginRender() -- iterator not initialized
atlas = layout.atlas()
atlas.setCoverageLayer(coverage_layer)
atlas.setEnabled(True)
while atlas.next():  # undefined behavior!
    exporter.exportToPdf(f"/tmp/page_{atlas.currentFeatureNumber()}.pdf", settings)

# CORRECT: ALWAYS bracket iteration with beginRender() / endRender()
atlas.beginRender()
while atlas.next():
    exporter.exportToPdf(f"/tmp/page_{atlas.currentFeatureNumber()}.pdf", settings)
atlas.endRender()
```

**WHY**: The atlas iterator is not initialized until `beginRender()` is called. Calling `next()` on an uninitialized iterator produces undefined behavior or an infinite loop.

---

## 5. Not Checking Export Result

```python
# WRONG: Ignoring the export result -- silent failure
exporter.exportToPdf("/tmp/output.pdf", settings)
print("Export done!")  # might not have succeeded

# CORRECT: ALWAYS check the result code
result = exporter.exportToPdf("/tmp/output.pdf", settings)
if result != QgsLayoutExporter.Success:
    raise RuntimeError(f"Export failed with code: {result}")
```

**WHY**: Export can fail for many reasons (file permissions, memory, invalid path). The method returns an error code instead of raising an exception. Ignoring it masks failures.

---

## 6. Legend Not Linked to Map

```python
# WRONG: Legend shows all project layers, not map-specific layers
legend = QgsLayoutItemLegend(layout)
layout.addLayoutItem(legend)
# Missing: legend.setLinkedMap(map_item)

# CORRECT: ALWAYS link the legend to the map item
legend = QgsLayoutItemLegend(layout)
legend.setLinkedMap(map_item)
layout.addLayoutItem(legend)
```

**WHY**: An unlinked legend displays all layers in the project, not the layers visible in the map item. This creates misleading legends that do not match the map content.

---

## 7. Scale Bar Not Linked to Map

```python
# WRONG: Scale bar has no reference map -- shows incorrect scale
scalebar = QgsLayoutItemScaleBar(layout)
scalebar.setStyle("Single Box")
scalebar.applyDefaultSize()
layout.addLayoutItem(scalebar)
# Missing: scalebar.setLinkedMap(map_item)

# CORRECT: ALWAYS link the scale bar to the map item
scalebar = QgsLayoutItemScaleBar(layout)
scalebar.setLinkedMap(map_item)
scalebar.setStyle("Single Box")
scalebar.applyDefaultSize()
layout.addLayoutItem(scalebar)
```

**WHY**: A scale bar without a linked map cannot calculate the correct scale. It displays meaningless or zero values.

---

## 8. Attribute Table Without Frame

```python
# WRONG: Table created but no frame added -- invisible in export
table = QgsLayoutItemAttributeTable.create(layout)
table.setVectorLayer(layer)
layout.addMultiFrame(table)
# Missing: QgsLayoutFrame creation and table.addFrame()

# CORRECT: ALWAYS create a frame and add it to the table
table = QgsLayoutItemAttributeTable.create(layout)
table.setVectorLayer(layer)
frame = QgsLayoutFrame(layout, table)
frame.attemptResize(QgsLayoutSize(200, 100, QgsUnitTypes.LayoutMillimeters))
frame.attemptMove(QgsLayoutPoint(10, 10, QgsUnitTypes.LayoutMillimeters))
table.addFrame(frame)
layout.addMultiFrame(table)
```

**WHY**: `QgsLayoutItemAttributeTable` is a multi-frame object. Without at least one `QgsLayoutFrame`, the table has no visual representation and does not appear in the layout.

---

## 9. Using addLayout() Without Setting Name

```python
# WRONG: Layout without a name causes issues with layoutByName()
layout = QgsPrintLayout(project)
layout.initializeDefaults()
project.layoutManager().addLayout(layout)
# Later: manager.layoutByName("") returns None or wrong result

# CORRECT: ALWAYS set a unique name before registering
layout = QgsPrintLayout(project)
layout.initializeDefaults()
layout.setName("Unique Layout Name")
project.layoutManager().addLayout(layout)
```

**WHY**: The layout manager uses names for lookup. Layouts without names are inaccessible via `layoutByName()` and create confusion when multiple unnamed layouts exist.

---

## 10. Forgetting endRender() After Atlas Iteration

```python
# WRONG: beginRender() without endRender() -- resources not cleaned up
atlas.beginRender()
while atlas.next():
    exporter.exportToPdf(f"/tmp/page_{atlas.currentFeatureNumber()}.pdf", settings)
# Missing: atlas.endRender()

# CORRECT: ALWAYS call endRender() to release resources
atlas.beginRender()
while atlas.next():
    exporter.exportToPdf(f"/tmp/page_{atlas.currentFeatureNumber()}.pdf", settings)
atlas.endRender()
```

**WHY**: `beginRender()` acquires resources and locks the coverage layer for iteration. Without `endRender()`, those resources are not released, which can cause memory leaks and layer lock issues.

---

## 11. Writing to Non-Existent Output Directory

```python
# WRONG: Parent directory does not exist -- FileError
result = exporter.exportToPdf("/tmp/nonexistent_dir/output.pdf", settings)
# result == QgsLayoutExporter.FileError

# CORRECT: ALWAYS ensure the output directory exists
import os
output_dir = "/tmp/map_exports/"
os.makedirs(output_dir, exist_ok=True)
result = exporter.exportToPdf(os.path.join(output_dir, "output.pdf"), settings)
```

**WHY**: `QgsLayoutExporter` does not create parent directories. If the target directory does not exist, the export fails with `FileError`.
