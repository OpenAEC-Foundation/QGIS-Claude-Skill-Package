# anti-patterns.md — qgis-syntax-processing-scripts

## AP-01: Unverified Algorithm ID

**NEVER** call `processing.run()` with an algorithm from an external provider without checking availability first.

```python
# WRONG — crashes if GRASS is not installed
result = processing.run("grass:v.buffer", params)

# CORRECT — verify algorithm exists before running
from qgis.core import QgsApplication

registry = QgsApplication.processingRegistry()
alg = registry.algorithmById('grass:v.buffer')
if alg is None:
    feedback.reportError('GRASS provider not available. Install GRASS GIS.')
    return {}
result = processing.run('grass:v.buffer', params)
```

**Why**: External providers (GRASS, SAGA, OTB) require separate installations. Native (`native:`) and GDAL (`gdal:`) algorithms are ALWAYS available.

---

## AP-02: Missing Exception Handling

**NEVER** call `processing.run()` without a try/except block.

```python
# WRONG — unhandled exception crashes the script
result = processing.run("native:buffer", params)
layer = result['OUTPUT']

# CORRECT — catch processing exceptions
from qgis.core import QgsProcessingException

try:
    result = processing.run("native:buffer", params)
    layer = result['OUTPUT']
except QgsProcessingException as e:
    feedback.reportError(f'Buffer operation failed: {e}')
    return {}
```

**Why**: Invalid parameters, missing inputs, invalid geometries, and disk space issues all cause `QgsProcessingException`.

---

## AP-03: No Cancellation Check in Loops

**NEVER** iterate over features without checking for cancellation.

```python
# WRONG — user cannot cancel, no progress reporting
def processAlgorithm(self, parameters, context, feedback):
    for feature in source.getFeatures():
        # process feature...
        pass

# CORRECT — cancellation check and progress reporting
def processAlgorithm(self, parameters, context, feedback):
    total = 100.0 / source.featureCount() if source.featureCount() else 0
    for current, feature in enumerate(source.getFeatures()):
        if feedback.isCanceled():
            break
        # process feature...
        feedback.setProgress(int(current * total))
```

**Why**: Without cancellation checks, the user has no way to stop a long-running algorithm. Without progress, the UI appears frozen.

---

## AP-04: Hardcoded Temporary Paths

**NEVER** use hardcoded temporary file paths for intermediate results.

```python
# WRONG — /tmp may not exist on Windows, path conflicts possible
result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 10,
    'OUTPUT': '/tmp/buffer_result.gpkg'
})

# CORRECT — let QGIS manage temporary storage
result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 10,
    'OUTPUT': 'memory:'
})

# ALSO CORRECT — using the constant
from qgis.core import QgsProcessing

result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 10,
    'OUTPUT': QgsProcessing.TEMPORARY_OUTPUT
})
```

**Why**: Hardcoded paths break cross-platform compatibility and risk file conflicts. Memory layers are automatically cleaned up.

---

## AP-05: GUI Elements in processAlgorithm

**NEVER** use message boxes, dialogs, or any GUI widgets inside `processAlgorithm()`.

```python
# WRONG — will crash in background thread
def processAlgorithm(self, parameters, context, feedback):
    from qgis.PyQt.QtWidgets import QMessageBox
    QMessageBox.warning(None, 'Warning', 'Something happened')

# CORRECT — use the feedback object for all communication
def processAlgorithm(self, parameters, context, feedback):
    feedback.pushWarning('Something happened')
    feedback.pushInfo('Processing step completed')
    feedback.reportError('Critical error occurred')
```

**Why**: `processAlgorithm()` runs in a background thread by default. GUI operations from non-main threads cause crashes.

---

## AP-06: Manual Layer Loading in Algorithm

**NEVER** load output layers into the project from within `processAlgorithm()`.

```python
# WRONG — bypasses the Processing framework's result handling
def processAlgorithm(self, parameters, context, feedback):
    # ... processing ...
    result_layer = QgsVectorLayer(output_path, 'result', 'ogr')
    QgsProject.instance().addMapLayer(result_layer)

# CORRECT — return the output and let the framework handle loading
def processAlgorithm(self, parameters, context, feedback):
    # ... processing ...
    return {self.OUTPUT: dest_id}
```

**Why**: The Processing framework manages output loading, including adding to the project, storing in context, and passing to downstream algorithms in chains/models.

---

## AP-07: Missing createInstance()

**NEVER** forget to implement `createInstance()` in a custom algorithm.

```python
# WRONG — missing createInstance(), algorithm will fail
class MyAlgorithm(QgsProcessingAlgorithm):
    def name(self):
        return 'myalgorithm'
    # ... other methods but no createInstance()

# CORRECT — ALWAYS implement createInstance()
class MyAlgorithm(QgsProcessingAlgorithm):
    def name(self):
        return 'myalgorithm'

    def createInstance(self):
        return MyAlgorithm()
```

**Why**: The Processing framework calls `createInstance()` to create copies of the algorithm for execution. Without it, the algorithm cannot be run.

---

## AP-08: CRS Mismatch in Multi-Layer Operations

**NEVER** combine layers with different CRSs without reprojecting first.

```python
# WRONG — intersection with mismatched CRSs gives incorrect results
result = processing.run("native:intersection", {
    'INPUT': layer_epsg4326,
    'OVERLAY': layer_epsg28992,
    'OUTPUT': 'memory:'
})

# CORRECT — reproject to matching CRS first
reprojected = processing.run("native:reprojectlayer", {
    'INPUT': layer_epsg4326,
    'TARGET_CRS': layer_epsg28992.crs(),
    'OUTPUT': 'memory:'
})

result = processing.run("native:intersection", {
    'INPUT': reprojected['OUTPUT'],
    'OVERLAY': layer_epsg28992,
    'OUTPUT': 'memory:'
})
```

**Why**: Processing algorithms execute in the input layer's CRS. Mismatched CRSs cause silent geometry errors, wrong results, or exceptions.

---

## AP-09: Undeclared Algorithm Outputs

**NEVER** return output values from `processAlgorithm()` without declaring them in `initAlgorithm()`.

```python
# WRONG — COUNT is returned but not declared as an output
def initAlgorithm(self, config=None):
    self.addParameter(QgsProcessingParameterFeatureSink('OUTPUT', 'Output'))

def processAlgorithm(self, parameters, context, feedback):
    return {'OUTPUT': dest_id, 'COUNT': feature_count}

# CORRECT — declare all outputs
def initAlgorithm(self, config=None):
    self.addParameter(QgsProcessingParameterFeatureSink('OUTPUT', 'Output'))
    self.addOutput(QgsProcessingOutputNumber('COUNT', 'Feature count'))

def processAlgorithm(self, parameters, context, feedback):
    return {'OUTPUT': dest_id, 'COUNT': feature_count}
```

**Why**: Undeclared outputs are invisible to the Processing framework and cannot be used in models, chains, or the GUI.

---

## AP-10: Using @alg Decorator in Plugins

**NEVER** use the `@alg` decorator for algorithms that belong to a plugin.

```python
# WRONG — @alg scripts go to the Scripts provider, not your plugin provider
from qgis.processing import alg

@alg(name='my_plugin_tool', label='My Plugin Tool', group='My Plugin')
def my_tool(instance, parameters, context, feedback, inputs):
    pass

# CORRECT — use QgsProcessingAlgorithm subclass for plugin algorithms
class MyPluginTool(QgsProcessingAlgorithm):
    # Full class implementation registered via QgsProcessingProvider
    pass
```

**Why**: `@alg` scripts are ALWAYS added to the Processing Scripts provider. They cannot be added to a custom provider. Plugin algorithms MUST use the full subclass approach.

---

## AP-11: Ignoring Source Validation

**NEVER** skip validation of source and sink parameters in `processAlgorithm()`.

```python
# WRONG — source could be None
def processAlgorithm(self, parameters, context, feedback):
    source = self.parameterAsSource(parameters, 'INPUT', context)
    for feature in source.getFeatures():  # NoneType error if source is None
        pass

# CORRECT — validate source before use
def processAlgorithm(self, parameters, context, feedback):
    source = self.parameterAsSource(parameters, 'INPUT', context)
    if source is None:
        raise QgsProcessingException(
            self.invalidSourceError(parameters, 'INPUT')
        )

    (sink, dest_id) = self.parameterAsSink(
        parameters, 'OUTPUT', context,
        source.fields(), source.wkbType(), source.sourceCrs()
    )
    if sink is None:
        raise QgsProcessingException(
            self.invalidSinkError(parameters, 'OUTPUT')
        )
```

**Why**: `parameterAsSource()` and `parameterAsSink()` return `None` when the input is invalid. Using `invalidSourceError()` and `invalidSinkError()` provides clear error messages.

---

## AP-12: Thread-Unsafe Operations Without Flag

**NEVER** access `iface`, GUI elements, or non-thread-safe libraries in `processAlgorithm()` without setting `FlagNoThreading`.

```python
# WRONG — iface access from background thread causes crash
def processAlgorithm(self, parameters, context, feedback):
    canvas = iface.mapCanvas()  # Thread-unsafe!

# CORRECT — set FlagNoThreading if you must access GUI
def flags(self):
    return super().flags() | QgsProcessingAlgorithm.FlagNoThreading

def processAlgorithm(self, parameters, context, feedback):
    canvas = iface.mapCanvas()  # Safe with FlagNoThreading
```

**Why**: Algorithms run in background threads by default. `FlagNoThreading` forces execution on the main thread, making GUI access safe but blocking the UI.
