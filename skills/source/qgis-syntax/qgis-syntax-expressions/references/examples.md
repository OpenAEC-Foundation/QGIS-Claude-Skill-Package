# Working Code Examples (QGIS Expression Engine)

## Example 1: Basic Expression Parsing and Evaluation

```python
from qgis.core import QgsExpression

# Simple arithmetic (no context needed)
exp = QgsExpression('2 + 3 * 4')
if exp.hasParserError():
    raise ValueError(exp.parserErrorString())
result = exp.evaluate()  # Returns 14

# Boolean expression
exp = QgsExpression('1 + 1 = 2')
result = exp.evaluate()  # Returns 1 (True)

# String expression
exp = QgsExpression("'Hello' || ' ' || 'World'")
result = exp.evaluate()  # Returns 'Hello World'
```

---

## Example 2: Evaluate Expression Against Features

```python
from qgis.core import (
    QgsExpression, QgsExpressionContext,
    QgsExpressionContextUtils, QgsProject
)

layer = QgsProject.instance().mapLayersByName('cities')[0]

exp = QgsExpression('"population" / "area_km2"')
if exp.hasParserError():
    raise ValueError(exp.parserErrorString())

context = QgsExpressionContext()
context.appendScopes(
    QgsExpressionContextUtils.globalProjectLayerScopes(layer)
)

# Prepare for repeated evaluation (performance optimization)
exp.prepare(context)

results = {}
for feature in layer.getFeatures():
    context.setFeature(feature)
    density = exp.evaluate(context)
    if exp.hasEvalError():
        raise ValueError(f"Feature {feature.id()}: {exp.evalErrorString()}")
    results[feature['name']] = density

print(results)
```

---

## Example 3: Filter Features with Expression

```python
from qgis.core import QgsFeatureRequest, QgsProject

layer = QgsProject.instance().mapLayersByName('buildings')[0]

# Filter by attribute
request = QgsFeatureRequest().setFilterExpression(
    '"building_type" = \'residential\' AND "floors" >= 3'
)
tall_residential = list(layer.getFeatures(request))
print(f"Found {len(tall_residential)} tall residential buildings")

# Filter by geometry expression
request = QgsFeatureRequest().setFilterExpression(
    '$area > 500'
)
large_buildings = list(layer.getFeatures(request))

# Combine with attribute subset for performance
request = QgsFeatureRequest().setFilterExpression(
    '"status" = \'active\''
).setSubsetOfAttributes(['name', 'status'], layer.fields())
active = list(layer.getFeatures(request))
```

---

## Example 4: Field Calculator — Compute New Values

```python
from qgis.core import (
    edit, QgsExpression, QgsExpressionContext,
    QgsExpressionContextUtils, QgsProject, QgsField
)
from qgis.PyQt.QtCore import QVariant

layer = QgsProject.instance().mapLayersByName('parcels')[0]

# Add new field if it does not exist
if layer.fields().indexOf('density') == -1:
    layer.dataProvider().addAttributes([
        QgsField('density', QVariant.Double)
    ])
    layer.updateFields()

exp = QgsExpression('"population" / ($area / 1000000)')
if exp.hasParserError():
    raise ValueError(exp.parserErrorString())

context = QgsExpressionContext()
context.appendScopes(
    QgsExpressionContextUtils.globalProjectLayerScopes(layer)
)
exp.prepare(context)

field_idx = layer.fields().indexOf('density')

with edit(layer):
    for f in layer.getFeatures():
        context.setFeature(f)
        value = exp.evaluate(context)
        layer.changeAttributeValue(f.id(), field_idx, value)
```

---

## Example 5: Expression-Based Labeling

```python
from qgis.core import (
    QgsPalLayerSettings, QgsVectorLayerSimpleLabeling,
    QgsTextFormat, QgsProject
)
from qgis.PyQt.QtGui import QFont, QColor

layer = QgsProject.instance().mapLayersByName('cities')[0]

# Configure label expression
settings = QgsPalLayerSettings()
settings.fieldName = (
    '"name" || \'\\n\' || '
    'format_number("population", 0) || \' inhabitants\''
)
settings.isExpression = True
settings.enabled = True

# Style the text
text_format = QgsTextFormat()
text_format.setFont(QFont('Arial', 10))
text_format.setColor(QColor('#333333'))
text_format.setSize(10)
settings.setFormat(text_format)

# Apply to layer
labeling = QgsVectorLayerSimpleLabeling(settings)
layer.setLabeling(labeling)
layer.setLabelsEnabled(True)
layer.triggerRepaint()
```

---

## Example 6: Expression-Based Labeling with Conditional Formatting

```python
from qgis.core import (
    QgsPalLayerSettings, QgsVectorLayerSimpleLabeling,
    QgsProperty, QgsProject
)

layer = QgsProject.instance().mapLayersByName('cities')[0]

settings = QgsPalLayerSettings()
settings.fieldName = '"name"'
settings.isExpression = True
settings.enabled = True

# Data-defined font size based on population
settings.dataDefinedProperties().setProperty(
    QgsPalLayerSettings.Property.Size,
    QgsProperty.fromExpression(
        'CASE '
        'WHEN "population" > 1000000 THEN 16 '
        'WHEN "population" > 100000 THEN 12 '
        'ELSE 8 END'
    )
)

# Data-defined color based on type
settings.dataDefinedProperties().setProperty(
    QgsPalLayerSettings.Property.Color,
    QgsProperty.fromExpression(
        "CASE WHEN \"capital\" = 1 THEN '#cc0000' ELSE '#333333' END"
    )
)

labeling = QgsVectorLayerSimpleLabeling(settings)
layer.setLabeling(labeling)
layer.setLabelsEnabled(True)
layer.triggerRepaint()
```

---

## Example 7: Data-Defined Symbology

```python
from qgis.core import (
    QgsProperty, QgsSingleSymbolRenderer,
    QgsMarkerSymbol, QgsProject
)

layer = QgsProject.instance().mapLayersByName('stations')[0]

# Create symbol with data-defined properties
symbol = QgsMarkerSymbol.createSimple({
    'name': 'circle',
    'color': '0,0,255',
    'size': '4'
})

# Size proportional to value
symbol.setDataDefinedSize(
    QgsProperty.fromExpression(
        'scale_linear("passenger_count", 0, 100000, 2, 20)'
    )
)

# Color based on expression
symbol.setDataDefinedColor(
    QgsProperty.fromExpression(
        'ramp_color(\'RdYlGn\', scale_linear("score", 0, 100, 0, 1))'
    )
)

renderer = QgsSingleSymbolRenderer(symbol)
layer.setRenderer(renderer)
layer.triggerRepaint()
```

---

## Example 8: Custom Expression Function

```python
from qgis.core import (
    qgsfunction, QgsExpression, QgsFeatureRequest, QgsMessageLog, Qgis
)

@qgsfunction(args='auto', group='Analysis', referenced_columns=['population', 'area_km2'])
def pop_density_class(population, area_km2, feature, parent):
    """
    Classify population density into categories.
    <h3>Usage</h3>
    <p>pop_density_class("population", "area_km2")</p>
    <h3>Returns</h3>
    <p>'high', 'medium', or 'low'</p>
    """
    if area_km2 is None or area_km2 <= 0:
        return None
    density = population / area_km2
    if density > 1000:
        return 'high'
    elif density > 200:
        return 'medium'
    else:
        return 'low'

# Register
QgsExpression.registerFunction(pop_density_class)

# Now usable in any expression:
# pop_density_class("population", "area_km2")
```

---

## Example 9: Custom Function Using Geometry

```python
from qgis.core import qgsfunction, QgsExpression

@qgsfunction(
    args=0,
    group='Geometry',
    referenced_columns=[],
    usesgeometry=True
)
def vertex_count(feature, parent):
    """
    Returns the total number of vertices in the feature geometry.
    <p>Usage: vertex_count()</p>
    """
    geom = feature.geometry()
    if geom.isNull():
        return 0
    return len(geom.asGeometryCollection()) if geom.isMultipart() else geom.constGet().nCoordinates()

QgsExpression.registerFunction(vertex_count)
```

---

## Example 10: Expression Templates with [% %] Tags

```python
from qgis.core import (
    QgsExpression, QgsExpressionContext,
    QgsExpressionContextUtils, QgsProject
)

layer = QgsProject.instance().mapLayersByName('parcels')[0]

template = 'Parcel [% "parcel_id" %] has area [% round($area, 2) %] m2'

context = QgsExpressionContext()
context.appendScopes(
    QgsExpressionContextUtils.globalProjectLayerScopes(layer)
)

for feature in layer.getFeatures():
    context.setFeature(feature)
    text = QgsExpression.replaceExpressionText(template, context)
    print(text)
    # Output: "Parcel A-123 has area 542.31 m2"
```

---

## Example 11: Aggregate Expressions

```python
from qgis.core import (
    QgsExpression, QgsExpressionContext,
    QgsExpressionContextUtils, QgsProject
)

layer = QgsProject.instance().mapLayersByName('sales')[0]

# Total sales per region (using aggregate function)
exp = QgsExpression(
    'aggregate(\'sales\', \'sum\', "amount", '
    'filter:="region" = attribute(@parent, \'region\'))'
)

context = QgsExpressionContext()
context.appendScopes(
    QgsExpressionContextUtils.globalProjectLayerScopes(layer)
)

# Standalone aggregate without feature context
exp_total = QgsExpression('aggregate(\'sales\', \'sum\', "amount")')
exp_total.prepare(context)
total = exp_total.evaluate(context)
print(f"Total sales: {total}")
```

---

## Example 12: Using Expression Context with Custom Variables

```python
from qgis.core import (
    QgsExpression, QgsExpressionContext,
    QgsExpressionContextScope, QgsExpressionContextUtils,
    QgsProject
)

layer = QgsProject.instance().mapLayersByName('measurements')[0]

context = QgsExpressionContext()
context.appendScopes(
    QgsExpressionContextUtils.globalProjectLayerScopes(layer)
)

# Add custom scope with analysis parameters
custom_scope = QgsExpressionContextScope('analysis')
custom_scope.setVariable('threshold', 50.0, isStatic=True)
custom_scope.setVariable('multiplier', 1.15, isStatic=True)
context.appendScope(custom_scope)

# Expression using custom variables
exp = QgsExpression('"value" * @multiplier > @threshold')
exp.prepare(context)

for feature in layer.getFeatures():
    context.setFeature(feature)
    if exp.evaluate(context):
        print(f"Feature {feature.id()} exceeds threshold")
```
