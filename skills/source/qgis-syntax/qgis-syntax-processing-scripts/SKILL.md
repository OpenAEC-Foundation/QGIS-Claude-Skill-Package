---
name: qgis-syntax-processing-scripts
description: >
  Use when running QGIS processing algorithms, creating custom algorithms, or chaining geoprocessing operations.
  Prevents hardcoded algorithm IDs, missing feedback handling, and incorrect parameter types.
  Covers processing.run(), custom QgsProcessingAlgorithm, @alg decorator, parameter types, and batch processing.
  Keywords: processing.run, QgsProcessingAlgorithm, algorithm, geoprocessing, batch processing, processing script, native:buffer.
license: MIT
compatibility: "Designed for Claude Code. Requires QGIS 3.44+ / PyQGIS 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# qgis-syntax-processing-scripts

## Quick Reference

### Processing Framework Overview

| Component | Purpose |
|-----------|---------|
| `processing.run()` | Execute any registered algorithm from Python |
| `processing.runAndLoadResults()` | Execute and add results to the current project |
| `QgsProcessingAlgorithm` | Base class for custom algorithms |
| `QgsProcessingProvider` | Groups custom algorithms under a provider ID |
| `QgsProcessingContext` | Execution environment (project, CRS, transform context) |
| `QgsProcessingFeedback` | Progress reporting, logging, and cancellation |
| `QgsProcessingAlgRunnerTask` | Background (non-blocking) algorithm execution |
| `QgsApplication.processingRegistry()` | Central registry for all providers and algorithms |

### Algorithm ID Format

Algorithm IDs follow the pattern `provider:algorithm_name`:

| Provider | Prefix | Example |
|----------|--------|---------|
| Native QGIS (C++) | `native:` | `native:buffer` |
| QGIS (legacy Python) | `qgis:` | `qgis:regularpoints` |
| GDAL/OGR | `gdal:` | `gdal:warpreproject` |
| GRASS GIS | `grass:` | `grass:v.buffer` |
| PDAL (point clouds) | `pdal:` | `pdal:info` |
| Processing Models | `model:` | `model:my_workflow` |
| Custom plugin | `{provider_id}:` | `myplugin:myalgorithm` |

### Key Parameter Types

| Parameter Class | Purpose | Read Method |
|----------------|---------|-------------|
| `QgsProcessingParameterFeatureSource` | Vector input | `parameterAsSource()` |
| `QgsProcessingParameterRasterLayer` | Raster input | `parameterAsRasterLayer()` |
| `QgsProcessingParameterNumber` | Numeric value | `parameterAsDouble()` / `parameterAsInt()` |
| `QgsProcessingParameterEnum` | Dropdown choice | `parameterAsEnum()` |
| `QgsProcessingParameterField` | Attribute field | `parameterAsString()` |
| `QgsProcessingParameterExpression` | QGIS expression | `parameterAsExpression()` |
| `QgsProcessingParameterCrs` | CRS selection | `parameterAsCrs()` |
| `QgsProcessingParameterExtent` | Bounding box | `parameterAsExtent()` |
| `QgsProcessingParameterBoolean` | Toggle | `parameterAsBool()` |
| `QgsProcessingParameterFeatureSink` | Vector output | `parameterAsSink()` |
| `QgsProcessingParameterRasterDestination` | Raster output | (returned as path) |

---

## Critical Warnings

**NEVER** call `processing.run()` without wrapping it in `try/except QgsProcessingException`. Algorithm failures raise exceptions that MUST be caught.

**NEVER** use hardcoded algorithm IDs from external providers (GRASS, SAGA, OTB) without first verifying availability via `QgsApplication.processingRegistry().algorithmById()`. These providers may not be installed.

**NEVER** show GUI elements (message boxes, dialogs) from within `processAlgorithm()`. Algorithms run in background threads by default. ALWAYS use the `feedback` object for all user communication.

**NEVER** manually load output layers inside `processAlgorithm()` using `QgsProject.instance().addMapLayer()`. ALWAYS return the output ID and let the Processing framework manage results.

**NEVER** use hardcoded temp paths like `/tmp/result.gpkg`. ALWAYS use `'memory:'` or `QgsProcessing.TEMPORARY_OUTPUT` for intermediate results.

**ALWAYS** check `feedback.isCanceled()` at the top of every loop iteration in custom algorithms. Failure to check causes unresponsive cancellation.

**ALWAYS** report progress in custom algorithms using `feedback.setProgress()` with a percentage (0-100).

**ALWAYS** declare all outputs in `initAlgorithm()`. Undeclared outputs are invisible to the Processing framework and cannot be used in models or chains.

---

## Decision Tree

### Which approach to use?

```
Need to run an existing algorithm?
├── Yes → processing.run("provider:algorithm", params)
│   ├── Need result in project? → processing.runAndLoadResults()
│   └── Need non-blocking? → QgsProcessingAlgRunnerTask
│
Need to create a custom algorithm?
├── For a QGIS plugin? → QgsProcessingAlgorithm subclass + QgsProcessingProvider
├── Standalone script? → @alg decorator (saved to Processing Scripts folder)
└── Quick prototype? → @alg decorator

Need to run same algorithm on many inputs?
└── Batch processing loop with processing.run() per iteration
```

### Output type selection

```
Where should the output go?
├── Intermediate result (not saved) → 'memory:' or QgsProcessing.TEMPORARY_OUTPUT
├── Persistent file → '/path/to/output.gpkg' (vectors) or '/path/to/output.tif' (rasters)
└── Add to project automatically → processing.runAndLoadResults()
```

---

## Essential Patterns

### Pattern 1: Running an Algorithm

```python
import processing
from qgis.core import QgsProcessingException

try:
    result = processing.run("native:buffer", {
        'INPUT': layer,           # QgsVectorLayer, file path, or URI
        'DISTANCE': 100,
        'SEGMENTS': 5,
        'END_CAP_STYLE': 0,      # 0=Round, 1=Flat, 2=Square
        'JOIN_STYLE': 0,          # 0=Round, 1=Miter, 2=Bevel
        'MITER_LIMIT': 2,
        'DISSOLVE': False,
        'OUTPUT': 'memory:'
    })
    buffered_layer = result['OUTPUT']
except QgsProcessingException as e:
    print(f"Buffer failed: {e}")
```

### Pattern 2: Running with Feedback

```python
from qgis.core import QgsProcessingFeedback

feedback = QgsProcessingFeedback()
feedback.progressChanged.connect(lambda p: print(f"Progress: {p}%"))

result = processing.run("native:buffer", {
    'INPUT': layer,
    'DISTANCE': 100,
    'OUTPUT': 'memory:'
}, feedback=feedback)
```

### Pattern 3: Background Execution

```python
from qgis.core import (
    QgsApplication, QgsProcessingAlgRunnerTask,
    QgsProcessingContext, QgsProcessingFeedback, QgsProject
)

context = QgsProcessingContext()
context.setProject(QgsProject.instance())
feedback = QgsProcessingFeedback()

alg = QgsApplication.processingRegistry().algorithmById('native:buffer')
params = {'INPUT': layer, 'DISTANCE': 100, 'OUTPUT': 'memory:'}

task = QgsProcessingAlgRunnerTask(alg, params, context, feedback)

def on_complete(successful, results):
    if successful:
        output_layer = context.getMapLayer(results['OUTPUT'])
        QgsProject.instance().addMapLayer(output_layer)

task.executed.connect(on_complete)
QgsApplication.taskManager().addTask(task)
```

### Pattern 4: Algorithm Chaining

```python
import processing

# Step 1: Reproject to a projected CRS
reprojected = processing.run("native:reprojectlayer", {
    'INPUT': input_layer,
    'TARGET_CRS': 'EPSG:28992',
    'OUTPUT': 'memory:'
})

# Step 2: Buffer (pass previous output directly)
buffered = processing.run("native:buffer", {
    'INPUT': reprojected['OUTPUT'],
    'DISTANCE': 100,
    'OUTPUT': 'memory:'
})

# Step 3: Dissolve into final output
dissolved = processing.run("native:dissolve", {
    'INPUT': buffered['OUTPUT'],
    'OUTPUT': '/output/final_result.gpkg'
})
```

### Pattern 5: Safe Algorithm Execution

```python
from qgis.core import QgsApplication, QgsProcessingException
import processing

def safe_run(algorithm_id, params, context=None, feedback=None):
    """Run an algorithm after verifying it exists."""
    registry = QgsApplication.processingRegistry()
    if registry.algorithmById(algorithm_id) is None:
        raise QgsProcessingException(
            f'Algorithm "{algorithm_id}" not found. '
            f'Check that the required provider is installed and enabled.'
        )
    return processing.run(algorithm_id, params,
                          context=context, feedback=feedback)
```

### Pattern 6: Batch Processing

```python
import os
import processing

input_dir = '/data/input/'
output_dir = '/data/output/'
input_files = [f for f in os.listdir(input_dir) if f.endswith('.gpkg')]

for input_file in input_files:
    input_path = os.path.join(input_dir, input_file)
    output_path = os.path.join(output_dir, f'buffered_{input_file}')

    try:
        processing.run("native:buffer", {
            'INPUT': input_path,
            'DISTANCE': 50,
            'OUTPUT': output_path
        })
    except QgsProcessingException as e:
        print(f"Failed for {input_file}: {e}")
```

---

## Common Operations

### Discover Available Algorithms

```python
from qgis.core import QgsApplication

registry = QgsApplication.processingRegistry()

# List all providers
for provider in registry.providers():
    print(provider.id(), provider.name())

# List algorithms from a provider
for alg in registry.algorithms():
    if alg.provider().id() == 'native':
        print(alg.id(), alg.displayName())

# Look up a specific algorithm
alg = registry.algorithmById('native:buffer')
if alg:
    print(alg.shortHelpString())
```

### Custom Algorithm Required Methods

| Method | Required | Purpose |
|--------|----------|---------|
| `name()` | YES | Unique ID (lowercase, no spaces) |
| `displayName()` | YES | User-visible name |
| `initAlgorithm(config)` | YES | Define parameters |
| `processAlgorithm(parameters, context, feedback)` | YES | Core logic |
| `createInstance()` | YES | Return new instance of the algorithm |
| `group()` / `groupId()` | Recommended | Category in toolbox |
| `shortHelpString()` | Recommended | Help text in dialog |
| `tags()` | Optional | Search keywords |
| `flags()` | Optional | e.g., `FlagNoThreading` |

### Register Custom Provider in Plugin

```python
# In plugin __init__.py or main module
from qgis.core import QgsApplication
from .provider import MyPluginProvider

class MyPlugin:
    def __init__(self, iface):
        self.provider = None

    def initGui(self):
        self.provider = MyPluginProvider()
        QgsApplication.processingRegistry().addProvider(self.provider)

    def unload(self):
        QgsApplication.processingRegistry().removeProvider(self.provider)
```

Plugin `metadata.txt` MUST include: `hasProcessingProvider=yes`

### Threading Flags

If your algorithm uses GUI elements or non-thread-safe APIs, set `FlagNoThreading`:

```python
def flags(self):
    return super().flags() | QgsProcessingAlgorithm.FlagNoThreading
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for QgsProcessingAlgorithm, processing.run, parameter types, and providers
- [references/examples.md](references/examples.md) -- Complete custom algorithm, @alg decorator script, batch processing, and chaining examples
- [references/anti-patterns.md](references/anti-patterns.md) -- Processing pitfalls with WRONG/CORRECT comparisons

### Official Sources

- https://docs.qgis.org/latest/en/docs/pyqgis_developer_cookbook/processing.html
- https://docs.qgis.org/latest/en/docs/user_manual/processing/index.html
- https://docs.qgis.org/latest/en/docs/user_manual/processing/toolbox.html
- https://qgis.org/pyqgis/master/core/QgsProcessingAlgorithm.html
