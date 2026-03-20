# Anti-Patterns (QGIS Expression Engine)

## 1. Missing Parser Error Check

```python
# WRONG: No error check after parsing
exp = QgsExpression('"nonexistent_field" + ')  # syntax error
result = exp.evaluate(context)  # Returns NULL silently

# CORRECT: ALWAYS check for parser errors
exp = QgsExpression('"nonexistent_field" + ')
if exp.hasParserError():
    raise ValueError(f"Expression parse error: {exp.parserErrorString()}")
```

**WHY**: `QgsExpression` does NOT raise exceptions on parse failure. It stores the error internally. Without checking, invalid expressions silently return NULL, making bugs invisible.

---

## 2. Missing Evaluation Error Check

```python
# WRONG: Evaluate without error check
exp = QgsExpression('"population" / "area"')
value = exp.evaluate(context)
# If area is 0, division error occurs silently

# CORRECT: ALWAYS check for evaluation errors
value = exp.evaluate(context)
if exp.hasEvalError():
    raise ValueError(f"Eval error: {exp.evalErrorString()}")
```

**WHY**: Division by zero, type mismatches, and missing fields cause evaluation errors that return NULL. Without checking, calculations silently produce incorrect results.

---

## 3. Evaluating Without Context

```python
# WRONG: No context when expression references fields
exp = QgsExpression('"population" * 2')
result = exp.evaluate()  # Returns NULL — no fields available

# CORRECT: ALWAYS provide context with appropriate scopes
context = QgsExpressionContext()
context.appendScopes(
    QgsExpressionContextUtils.globalProjectLayerScopes(layer)
)
context.setFeature(feature)
result = exp.evaluate(context)
```

**WHY**: Without a context, field references (`"population"`) resolve to NULL. The expression engine has no way to know which layer or feature to read from.

---

## 4. Forgetting to Set Feature in Loop

```python
# WRONG: Feature never set — all iterations use stale/empty data
exp = QgsExpression('"value" * 2')
context = QgsExpressionContext()
context.appendScopes(
    QgsExpressionContextUtils.globalProjectLayerScopes(layer)
)

for feature in layer.getFeatures():
    result = exp.evaluate(context)  # Same NULL/stale result every iteration

# CORRECT: ALWAYS set feature before evaluating
for feature in layer.getFeatures():
    context.setFeature(feature)
    result = exp.evaluate(context)
```

**WHY**: The context does not automatically track which feature is current. You MUST call `setFeature()` for each feature in the loop.

---

## 5. Confusing Single and Double Quotes

```python
# WRONG: Double quotes for string literal — reads field named "residential"
exp = QgsExpression('"type" = "residential"')
# This compares field "type" with field "residential" (probably NULL)

# CORRECT: Single quotes for string literals
exp = QgsExpression('"type" = \'residential\'')
# Or use Python's quoting to avoid escaping:
exp = QgsExpression("\"type\" = 'residential'")
```

**WHY**: In QGIS expressions, double quotes (`"..."`) ALWAYS reference field names. Single quotes (`'...'`) ALWAYS denote string literals. Mixing them up causes silent mismatches.

---

## 6. Modifying Features Without Edit Session

```python
# WRONG: Field calculator without edit session — changes are lost
exp = QgsExpression('"population" * 2')
context = QgsExpressionContext()
context.appendScopes(
    QgsExpressionContextUtils.globalProjectLayerScopes(layer)
)

for f in layer.getFeatures():
    context.setFeature(f)
    f['doubled'] = exp.evaluate(context)
    layer.updateFeature(f)  # Fails silently — no edit session

# CORRECT: ALWAYS wrap in edit session
with edit(layer):
    for f in layer.getFeatures():
        context.setFeature(f)
        f['doubled'] = exp.evaluate(context)
        layer.updateFeature(f)
```

**WHY**: `updateFeature()` requires an active edit session. Without `edit(layer)` or `startEditing()`, changes are silently discarded.

---

## 7. Custom Function Missing referenced_columns

```python
# WRONG: Empty referenced_columns but function reads fields
@qgsfunction(args='auto', group='Custom', referenced_columns=[])
def classify(value, feature, parent):
    # Reads 'category' field directly — but QGIS does not know this
    return feature['category'] + '_' + str(value)

# CORRECT: Declare all accessed fields
@qgsfunction(
    args='auto',
    group='Custom',
    referenced_columns=['category']
)
def classify(value, feature, parent):
    return feature['category'] + '_' + str(value)

# Or use ALL_ATTRIBUTES if fields are dynamic
@qgsfunction(
    args='auto',
    group='Custom',
    referenced_columns=[QgsFeatureRequest.ALL_ATTRIBUTES]
)
def classify(value, feature, parent):
    return feature['category'] + '_' + str(value)
```

**WHY**: QGIS uses `referenced_columns` to optimize which fields are loaded. If a field is accessed but not declared, it may be NULL because it was not fetched from the data provider.

---

## 8. Custom Function Missing usesgeometry Flag

```python
# WRONG: Accesses geometry but does not declare it
@qgsfunction(args=0, group='Custom', referenced_columns=[])
def my_area(feature, parent):
    return feature.geometry().area()  # geometry may be NULL

# CORRECT: ALWAYS set usesgeometry=True when accessing geometry
@qgsfunction(
    args=0,
    group='Custom',
    referenced_columns=[],
    usesgeometry=True
)
def my_area(feature, parent):
    geom = feature.geometry()
    if geom.isNull():
        return None
    return geom.area()
```

**WHY**: When `usesgeometry=False` (default), QGIS may skip loading geometries for performance. The function then receives a NULL geometry.

---

## 9. Using print() in Custom Expression Functions

```python
# WRONG: print() in expression function causes thread-safety issues
@qgsfunction(args='auto', group='Debug', referenced_columns=[])
def debug_value(val, feature, parent):
    print(f"Debug: {val}")  # thread-unsafe during rendering
    return val

# CORRECT: Use QgsMessageLog
from qgis.core import QgsMessageLog, Qgis

@qgsfunction(args='auto', group='Debug', referenced_columns=[])
def debug_value(val, feature, parent):
    QgsMessageLog.logMessage(f"Debug: {val}", 'Custom', Qgis.Info)
    return val
```

**WHY**: Expression functions execute during rendering, which runs on multiple threads. `print()` is not thread-safe and causes crashes or garbled output.

---

## 10. Not Unregistering Custom Functions on Plugin Unload

```python
# WRONG: Register in initGui but forget to unregister
class MyPlugin:
    def initGui(self):
        QgsExpression.registerFunction(my_custom_func)

    def unload(self):
        pass  # Stale function reference remains

# CORRECT: ALWAYS unregister in unload()
class MyPlugin:
    def initGui(self):
        QgsExpression.registerFunction(my_custom_func)

    def unload(self):
        QgsExpression.unregisterFunction('my_custom_func')
```

**WHY**: Registered functions persist globally. If the plugin is reloaded without unregistering, the old function reference becomes stale and causes errors or crashes.

---

## 11. Inefficient Expression Evaluation Without prepare()

```python
# INEFFICIENT: Expression re-parsed for every feature
exp = QgsExpression('sqrt("area") * 100 + length("name")')
for feature in layer.getFeatures():
    context.setFeature(feature)
    result = exp.evaluate(context)

# BETTER: Prepare once, evaluate many
exp = QgsExpression('sqrt("area") * 100 + length("name")')
exp.prepare(context)  # Resolves field indices once
for feature in layer.getFeatures():
    context.setFeature(feature)
    result = exp.evaluate(context)
```

**WHY**: `prepare()` resolves field name lookups and function bindings once. Without it, these lookups happen on every `evaluate()` call, which is measurably slower on large datasets.

---

## 12. Aggregate Expression Without Filter on Large Datasets

```python
# WRONG: Aggregates entire layer on every feature evaluation
exp = QgsExpression(
    '"population" / aggregate(\'cities\', \'sum\', "population")'
)
# This recomputes sum(population) for every single feature

# CORRECT: Pre-compute the aggregate or use a filter
# Option A: Pre-compute
total_exp = QgsExpression('aggregate(\'cities\', \'sum\', "population")')
total_exp.prepare(context)
total = total_exp.evaluate(context)

# Then use the value directly
for feature in layer.getFeatures():
    context.setFeature(feature)
    ratio = feature['population'] / total

# Option B: Use with filter to limit scope
exp = QgsExpression(
    'aggregate(\'cities\', \'sum\', "population", '
    'filter:="region" = attribute(@parent, \'region\'))'
)
```

**WHY**: Unfiltered aggregates on large layers are recomputed for each feature evaluation. This turns O(n) operations into O(n^2).

---

## 13. Labeling Expression Without isExpression Flag

```python
# WRONG: Expression string but isExpression not set
settings = QgsPalLayerSettings()
settings.fieldName = '"name" || \' (\' || "population" || \')\''
# settings.isExpression not set — defaults to False
# QGIS treats this as a literal field name, not an expression

# CORRECT: ALWAYS set isExpression = True for expression-based labels
settings = QgsPalLayerSettings()
settings.fieldName = '"name" || \' (\' || "population" || \')\''
settings.isExpression = True
```

**WHY**: `QgsPalLayerSettings` defaults to treating `fieldName` as a simple field reference. Without `isExpression = True`, the expression is interpreted as a literal field name that does not exist, producing empty labels.
