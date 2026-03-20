# API Signatures Reference (QGIS Expression Engine)

## QgsExpression

The core class for parsing and evaluating QGIS expressions.

```python
class QgsExpression:
    def __init__(self, expr: str)
        # Parse the expression string. Check hasParserError() after construction.

    # --- Parsing ---
    def hasParserError(self) -> bool
        # Returns True if the expression string has syntax errors.

    def parserErrorString(self) -> str
        # Returns the parser error message. Empty string if no error.

    # --- Evaluation ---
    def evaluate(self, context: QgsExpressionContext = None) -> Any
        # Evaluate the expression. Returns the result value.
        # Pass a context for expressions referencing fields/variables.
        # Returns None/NULL on error.

    def hasEvalError(self) -> bool
        # Returns True if the last evaluate() call had an error.

    def evalErrorString(self) -> str
        # Returns the evaluation error message.

    def isValid(self) -> bool
        # Returns True if expression was parsed successfully.

    # --- Prepare (optimization for repeated evaluation) ---
    def prepare(self, context: QgsExpressionContext) -> bool
        # Prepare expression for repeated evaluation against a context.
        # Call once before looping over features for better performance.
        # Returns True on success.

    # --- Inspection ---
    def expression(self) -> str
        # Returns the original expression string.

    def dump(self) -> str
        # Returns a human-readable representation of the parsed expression.

    def referencedColumns(self) -> set[str]
        # Returns field names referenced by the expression.

    def referencedVariables(self) -> set[str]
        # Returns variable names referenced by the expression.

    def referencedFunctions(self) -> set[str]
        # Returns function names used in the expression.

    def needsGeometry(self) -> bool
        # Returns True if the expression uses geometry ($geometry, $area, etc.).

    def isField(self) -> bool
        # Returns True if the expression is a simple field reference.

    # --- Static methods ---
    @staticmethod
    def registerFunction(function) -> bool
        # Register a custom @qgsfunction with the expression engine.
        # Returns True on success.

    @staticmethod
    def unregisterFunction(name: str) -> bool
        # Unregister a custom function by name.
        # Returns True on success.

    @staticmethod
    def isFunctionName(name: str) -> bool
        # Returns True if name is a registered function.

    @staticmethod
    def functionIndex(name: str) -> int
        # Returns the index of a function in the functions list, or -1.

    @staticmethod
    def quotedColumnRef(name: str) -> str
        # Returns a properly quoted column reference: "field_name"

    @staticmethod
    def quotedString(text: str) -> str
        # Returns a properly quoted string literal: 'text'

    @staticmethod
    def createFieldEqualityExpression(fieldName: str, value) -> str
        # Creates "fieldName" = value with proper quoting.

    @staticmethod
    def replaceExpressionText(
        action: str,
        context: QgsExpressionContext,
        distanceArea: QgsDistanceArea = None
    ) -> str
        # Evaluates [% expression %] tags within a text template.
        # Used for expression-based text templates in layouts and labels.
```

---

## QgsExpressionContext

Provides the evaluation context with variables, scopes, and the current feature.

```python
class QgsExpressionContext:
    def __init__(self, scopes: list[QgsExpressionContextScope] = None)
        # Create a context, optionally with pre-built scopes.

    # --- Scope management ---
    def appendScope(self, scope: QgsExpressionContextScope)
        # Add a scope to the end (highest priority).

    def appendScopes(self, scopes: list[QgsExpressionContextScope])
        # Add multiple scopes at once.

    def removeLastScope(self) -> QgsExpressionContextScope
        # Remove and return the last (highest priority) scope.

    def lastScope(self) -> QgsExpressionContextScope
        # Return the last scope without removing it.

    def scopeCount(self) -> int
        # Returns the number of scopes.

    # --- Feature ---
    def setFeature(self, feature: QgsFeature)
        # Set the current feature for evaluation.
        # ALWAYS call this before evaluating feature-dependent expressions.

    def feature(self) -> QgsFeature
        # Returns the current feature.

    def hasFeature(self) -> bool
        # Returns True if a feature is set.

    # --- Variables ---
    def variable(self, name: str) -> Any
        # Returns the value of a variable (e.g., 'layer_name').

    def hasVariable(self, name: str) -> bool
        # Returns True if the variable exists in any scope.

    def variableNames(self) -> list[str]
        # Returns all variable names across all scopes.

    def setFields(self, fields: QgsFields)
        # Set available fields for field reference resolution.

    def fields(self) -> QgsFields
        # Returns the fields available in this context.
```

---

## QgsExpressionContextUtils

Static utility class for creating standard scopes.

```python
class QgsExpressionContextUtils:

    @staticmethod
    def globalScope() -> QgsExpressionContextScope
        # Scope with global variables: @qgis_version, @user_full_name, etc.

    @staticmethod
    def projectScope(project: QgsProject) -> QgsExpressionContextScope
        # Scope with project variables: @project_title, @project_crs, etc.

    @staticmethod
    def layerScope(layer: QgsMapLayer) -> QgsExpressionContextScope
        # Scope with layer variables: @layer_name, @layer_id, fields, etc.

    @staticmethod
    def globalProjectLayerScopes(layer: QgsMapLayer) -> list[QgsExpressionContextScope]
        # Convenience: returns [globalScope, projectScope, layerScope] in one call.
        # ALWAYS use this as the starting point for feature-based evaluation.

    @staticmethod
    def setGlobalVariable(name: str, value)
        # Set a global variable (persists across sessions in QGIS settings).

    @staticmethod
    def setGlobalVariables(variables: dict)
        # Set multiple global variables at once.

    @staticmethod
    def removeGlobalVariable(name: str)
        # Remove a global variable.

    @staticmethod
    def setProjectVariable(project: QgsProject, name: str, value)
        # Set a project variable (saved with the .qgz/.qgs file).

    @staticmethod
    def setProjectVariables(project: QgsProject, variables: dict)
        # Set multiple project variables at once.

    @staticmethod
    def removeProjectVariable(project: QgsProject, name: str)
        # Remove a project variable.

    @staticmethod
    def setLayerVariable(layer: QgsMapLayer, name: str, value)
        # Set a layer variable (saved with the layer in the project).

    @staticmethod
    def setLayerVariables(layer: QgsMapLayer, variables: dict)
        # Set multiple layer variables at once.

    @staticmethod
    def removeLayerVariable(layer: QgsMapLayer, name: str)
        # Remove a layer variable.
```

---

## QgsExpressionContextScope

A single scope within an expression context, holding variables and functions.

```python
class QgsExpressionContextScope:
    def __init__(self, name: str = '')
        # Create a named scope.

    def setVariable(self, name: str, value, isStatic: bool = False)
        # Set a variable in this scope.
        # isStatic=True means value does not change per feature (optimization).

    def variable(self, name: str) -> Any
        # Returns the variable value.

    def hasVariable(self, name: str) -> bool
        # Returns True if the variable exists in this scope.

    def variableNames(self) -> list[str]
        # Returns all variable names in this scope.

    def removeVariable(self, name: str) -> bool
        # Remove a variable. Returns True if it existed.

    def setFeature(self, feature: QgsFeature)
        # Set the feature for this scope.

    def name(self) -> str
        # Returns the scope name.
```

---

## @qgsfunction Decorator

Decorator for creating custom expression functions callable from QGIS expressions.

```python
from qgis.core import qgsfunction

@qgsfunction(
    args='auto',              # int or 'auto' — number of arguments
    group='Custom',           # str — category in expression builder UI
    referenced_columns=[],    # list[str] — field names accessed, or
                              #   [QgsFeatureRequest.ALL_ATTRIBUTES]
    usesgeometry=False,       # bool — True if function accesses geometry
    handlesnull=False,        # bool — True if function handles NULL inputs
    register=True             # bool — auto-register on import (default True)
)
def my_function(value1, value2, feature, parent):
    """
    Function description shown in expression builder.
    <p>HTML markup is supported for detailed help.</p>
    """
    return value1 + value2
```

### Parameter Rules

- When `args='auto'`: the decorator counts parameters excluding `feature` and `parent`
- `feature` parameter: the current QgsFeature (ALWAYS include as second-to-last param)
- `parent` parameter: the parent QgsExpression node (ALWAYS include as last param)
- Both `feature` and `parent` are injected automatically; do NOT pass them when calling

### Registration

```python
# Manual registration (if register=False in decorator)
QgsExpression.registerFunction(my_function)

# ALWAYS unregister in plugin unload() to prevent stale references
QgsExpression.unregisterFunction('my_function')
```

---

## QgsProperty

Binds an expression (or field) to a symbology/rendering property.

```python
class QgsProperty:
    @staticmethod
    def fromExpression(expression: str, isActive: bool = True) -> QgsProperty
        # Create a property driven by an expression string.

    @staticmethod
    def fromField(fieldName: str, isActive: bool = True) -> QgsProperty
        # Create a property driven by a field value.

    @staticmethod
    def fromValue(value, isActive: bool = True) -> QgsProperty
        # Create a property with a static value.

    def expressionString(self) -> str
        # Returns the expression string (if expression-based).

    def field(self) -> str
        # Returns the field name (if field-based).

    def isActive(self) -> bool
        # Returns True if this property is active.

    def setActive(self, active: bool)
        # Enable or disable this property.

    def valueAsString(self, context: QgsExpressionContext, defaultString: str = '') -> tuple[str, bool]
        # Evaluate and return (value, isValid).

    def valueAsDouble(self, context: QgsExpressionContext, defaultValue: float = 0.0) -> tuple[float, bool]
        # Evaluate and return (value, isValid).

    def valueAsInt(self, context: QgsExpressionContext, defaultValue: int = 0) -> tuple[int, bool]
        # Evaluate and return (value, isValid).

    def valueAsColor(self, context: QgsExpressionContext, defaultColor: QColor = QColor()) -> tuple[QColor, bool]
        # Evaluate and return (color, isValid).
```

---

## QgsFeatureRequest (Expression Filtering)

```python
class QgsFeatureRequest:
    def setFilterExpression(self, expression: str) -> QgsFeatureRequest
        # Filter features using an expression string.
        # Returns self for method chaining.

    def filterExpression(self) -> QgsExpression
        # Returns the current filter expression (or None).

    def setExpressionContext(self, context: QgsExpressionContext) -> QgsFeatureRequest
        # Set the expression context for filter evaluation.
        # Required when expressions reference project/global variables.
```
