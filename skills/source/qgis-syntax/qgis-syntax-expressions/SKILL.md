---
name: qgis-syntax-expressions
description: >
  Use when writing QGIS expressions for filtering, labeling, symbology, or field calculations.
  Prevents expression syntax errors and context misconfiguration.
  Covers QgsExpression parsing, evaluation contexts, field calculator, data-defined properties, and custom functions.
  Keywords: QgsExpression, expression, field calculator, label expression, data-defined, @qgsfunction, filter, evaluate.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-syntax-expressions

## Quick Reference

### Expression Evaluation Pipeline

| Step | Class | Purpose |
|------|-------|---------|
| 1. Parse | `QgsExpression(string)` | Validate syntax, build AST |
| 2. Context | `QgsExpressionContext()` | Provide variables, fields, scopes |
| 3. Scopes | `QgsExpressionContextUtils` | Add global/project/layer/feature scopes |
| 4. Evaluate | `exp.evaluate(context)` | Execute and return result |
| 5. Validate | `exp.hasEvalError()` | Check for runtime errors |

### Expression Syntax Cheatsheet

| Element | Syntax | Example |
|---------|--------|---------|
| Field reference | `"field_name"` (double quotes) | `"population"` |
| String literal | `'text'` (single quotes) | `'hello world'` |
| Number literal | `123`, `3.14` | `42` |
| NULL check | `IS NULL`, `IS NOT NULL` | `"name" IS NOT NULL` |
| Comparison | `=`, `!=`, `>`, `>=`, `<`, `<=` | `"area" > 100` |
| Logical | `AND`, `OR`, `NOT` | `"pop" > 1000 AND "type" = 'city'` |
| Arithmetic | `+`, `-`, `*`, `/`, `^`, `%` | `"population" * 1.05` |
| Pattern match | `LIKE`, `ILIKE`, `~` (regex) | `"name" ILIKE '%lake%'` |
| Concatenation | `\|\|` | `"first" \|\| ' ' \|\| "last"` |
| Conditional | `CASE WHEN ... THEN ... END` | `CASE WHEN "pop" > 1e6 THEN 'large' ELSE 'small' END` |
| Geometry vars | `$area`, `$length`, `$x`, `$y` | `$area / 1e6` |
| Current geometry | `$geometry` | `num_points($geometry)` |

### Common Expression Functions

| Category | Functions |
|----------|-----------|
| Math | `sqrt()`, `abs()`, `round()`, `floor()`, `ceil()`, `sin()`, `cos()`, `pi()` |
| String | `upper()`, `lower()`, `length()`, `trim()`, `replace()`, `regexp_replace()`, `substr()`, `left()`, `right()` |
| Conversion | `to_int()`, `to_real()`, `to_string()`, `to_date()`, `to_datetime()` |
| Date/Time | `now()`, `day()`, `month()`, `year()`, `hour()`, `minute()`, `age()`, `day_of_week()` |
| Geometry | `$area`, `$length`, `$x`, `$y`, `$perimeter`, `centroid()`, `buffer()`, `area()`, `length()`, `num_geometries()`, `num_points()` |
| Aggregates | `aggregate()`, `sum()`, `count()`, `mean()`, `min()`, `max()`, `concatenate()`, `array_agg()` |
| Conditionals | `if()`, `coalesce()`, `nullif()`, `try()` |
| Arrays | `array()`, `array_length()`, `array_contains()`, `array_append()`, `array_to_string()` |
| Map/Record | `map()`, `map_get()`, `hstore_to_map()` |
| Color | `color_rgb()`, `color_hsv()`, `ramp_color()`, `darker()`, `lighter()` |

---

## Critical Warnings

**ALWAYS** check `exp.hasParserError()` after creating a `QgsExpression`. A parser error means the expression string is syntactically invalid and evaluation will fail silently or return NULL.

**ALWAYS** check `exp.hasEvalError()` after calling `exp.evaluate()`. Evaluation errors occur when the expression is syntactically valid but fails at runtime (e.g., field not found, type mismatch).

**ALWAYS** set up a proper `QgsExpressionContext` with appropriate scopes when evaluating expressions that reference fields, project variables, or layer properties. Evaluating without context returns NULL for all field references.

**NEVER** use double quotes for string literals in expressions. Double quotes reference field names: `"name"` reads the field, `'name'` is the string literal.

**NEVER** forget to call `context.setFeature(feature)` before evaluating feature-dependent expressions in a loop. Omitting this causes all features to evaluate against stale or empty data.

**NEVER** use `print()` inside custom expression functions (`@qgsfunction`). Use `QgsMessageLog` instead. Expression functions run during rendering and `print()` causes thread-safety issues.

**ALWAYS** specify `referenced_columns` in `@qgsfunction` for performance. Use `[QgsFeatureRequest.ALL_ATTRIBUTES]` if the function reads arbitrary fields; use `[]` if it reads no fields.

**ALWAYS** set `usesgeometry=True` in `@qgsfunction` if the function accesses `feature.geometry()`. Omitting this causes the geometry to be unavailable.

---

## Decision Tree: Choosing the Right Expression Approach

```
Need to use an expression?
|
+-- Filter features? --> QgsFeatureRequest.setFilterExpression()
|
+-- Calculate field values? --> Field Calculator pattern (edit session + evaluate per feature)
|
+-- Drive labeling? --> QgsPalLayerSettings.fieldName + isExpression = True
|
+-- Drive symbology? --> QgsProperty.fromExpression() on symbol/renderer properties
|
+-- Evaluate standalone? --> QgsExpression.evaluate(context)
|
+-- Need custom logic? --> @qgsfunction decorator + QgsExpression.registerFunction()
```

---

## Essential Patterns

### Pattern 1: Parse and Evaluate an Expression

```python
from qgis.core import (
    QgsExpression, QgsExpressionContext, QgsExpressionContextUtils
)

exp = QgsExpression('"population" * 1.05')
if exp.hasParserError():
    raise ValueError(f"Parse error: {exp.parserErrorString()}")

context = QgsExpressionContext()
context.appendScopes(
    QgsExpressionContextUtils.globalProjectLayerScopes(layer)
)

for feature in layer.getFeatures():
    context.setFeature(feature)
    value = exp.evaluate(context)
    if exp.hasEvalError():
        raise ValueError(f"Eval error: {exp.evalErrorString()}")
    print(f"Feature {feature.id()}: {value}")
```

### Pattern 2: Expression-Based Feature Filtering

```python
from qgis.core import QgsFeatureRequest

# Method A: expression string directly
request = QgsFeatureRequest().setFilterExpression('"population" >= 50000')
features = list(layer.getFeatures(request))

# Method B: QgsExpression object
exp = QgsExpression('"name" ILIKE \'%lake%\'')
request = QgsFeatureRequest(exp)
features = list(layer.getFeatures(request))
```

### Pattern 3: Field Calculator (Update Attributes)

```python
from qgis.core import (
    edit, QgsExpression, QgsExpressionContext,
    QgsExpressionContextUtils
)

exp = QgsExpression('"population" * 2')
context = QgsExpressionContext()
context.appendScopes(
    QgsExpressionContextUtils.globalProjectLayerScopes(layer)
)

with edit(layer):
    for f in layer.getFeatures():
        context.setFeature(f)
        f['doubled_pop'] = exp.evaluate(context)
        layer.updateFeature(f)
```

### Pattern 4: Expression-Based Labeling

```python
from qgis.core import QgsPalLayerSettings, QgsVectorLayerSimpleLabeling

settings = QgsPalLayerSettings()
settings.fieldName = '"name" || \' (\' || "population" || \')\''
settings.isExpression = True
settings.enabled = True

text_format = settings.format()
text_format.setSize(12)
settings.setFormat(text_format)

labeling = QgsVectorLayerSimpleLabeling(settings)
layer.setLabeling(labeling)
layer.setLabelsEnabled(True)
layer.triggerRepaint()
```

### Pattern 5: Data-Defined Symbology Properties

```python
from qgis.core import QgsProperty

symbol = layer.renderer().symbol()

# Size driven by expression
symbol.setDataDefinedSize(
    QgsProperty.fromExpression('"population" / 1000')
)

# Color driven by expression
symbol.setDataDefinedColor(
    QgsProperty.fromExpression(
        "CASE WHEN \"type\" = 'city' THEN '#ff0000' ELSE '#0000ff' END"
    )
)

layer.triggerRepaint()
```

### Pattern 6: Custom Expression Function

```python
from qgis.core import qgsfunction, QgsExpression

@qgsfunction(args='auto', group='Custom', referenced_columns=[])
def population_density(population, area_km2, feature, parent):
    """
    Calculate population density per square kilometer.
    <p>Usage: population_density("pop_field", "area_field")</p>
    """
    if area_km2 and area_km2 > 0:
        return population / area_km2
    return None

# Register (call once, e.g., in plugin initGui)
QgsExpression.registerFunction(population_density)

# Now usable: population_density("population", "area")

# Unregister (call in plugin unload)
QgsExpression.unregisterFunction('population_density')
```

---

## Context Scope Hierarchy

Scopes are stacked from generic to specific. When variable names collide, the most specific scope wins:

```
Global scope        → @qgis_version, @user_full_name, custom global vars
  Project scope     → @project_title, @project_crs, custom project vars
    Layer scope     → @layer_name, @layer_id, layer fields
      Feature scope → Feature attributes, $geometry, $id, $currentfeature
```

### Adding Custom Variables

```python
from qgis.core import QgsExpressionContextUtils

# Set global variable (persists across sessions)
QgsExpressionContextUtils.setGlobalVariable('my_threshold', 42)

# Set project variable (saved with project)
QgsExpressionContextUtils.setProjectVariable(
    QgsProject.instance(), 'analysis_year', 2024
)

# Set layer variable (saved with layer in project)
QgsExpressionContextUtils.setLayerVariable(layer, 'source_date', '2024-01-15')

# Access in expressions: @my_threshold, @analysis_year, @source_date
```

---

## Aggregate Expressions

Aggregates compute values across features within a layer:

```python
# In expression strings:
# aggregate('layer_name', 'sum', "population")
# aggregate('layer_name', 'mean', "area", filter:="type" = 'residential')

# Common shorthand (current layer):
# sum("population")
# count("id", filter:="active" = 1)
# mean("temperature", group_by:="region")
```

**ALWAYS** use the `filter` parameter in aggregates to limit the scope. Aggregating an entire large layer without a filter is a performance bottleneck.

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for QgsExpression, QgsExpressionContext, QgsExpressionContextUtils, @qgsfunction
- [references/examples.md](references/examples.md) -- Complete expression examples for filtering, labeling, symbology, field calculator
- [references/anti-patterns.md](references/anti-patterns.md) -- Expression pitfalls and incorrect patterns

### Official Sources

- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/expressions.html
- https://docs.qgis.org/latest/en/docs/user_manual/expressions/expression.html
- https://qgis.org/pyqgis/master/core/QgsExpression.html
- https://qgis.org/pyqgis/master/core/QgsExpressionContext.html
