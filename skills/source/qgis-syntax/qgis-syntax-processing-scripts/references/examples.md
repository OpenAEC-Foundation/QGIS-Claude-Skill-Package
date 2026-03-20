# examples.md — qgis-syntax-processing-scripts

## Example 1: Running a Built-in Algorithm

```python
import processing
from qgis.core import QgsProcessingException, QgsVectorLayer

# Load input layer
layer = QgsVectorLayer('/data/buildings.gpkg|layername=buildings', 'buildings', 'ogr')

try:
    result = processing.run("native:buffer", {
        'INPUT': layer,
        'DISTANCE': 50,
        'SEGMENTS': 5,
        'END_CAP_STYLE': 0,   # 0=Round, 1=Flat, 2=Square
        'JOIN_STYLE': 0,       # 0=Round, 1=Miter, 2=Bevel
        'MITER_LIMIT': 2,
        'DISSOLVE': False,
        'OUTPUT': 'memory:'
    })
    buffered_layer = result['OUTPUT']
    print(f"Buffered layer has {buffered_layer.featureCount()} features")
except QgsProcessingException as e:
    print(f"Buffer failed: {e}")
```

## Example 2: Running and Loading Results into Project

```python
import processing

# Automatically adds the result to the QGIS project
result = processing.runAndLoadResults("native:buffer", {
    'INPUT': 'path/to/input.gpkg',
    'DISTANCE': 100,
    'OUTPUT': 'memory:'
})
```

## Example 3: Algorithm with Feedback and Progress

```python
import processing
from qgis.core import QgsProcessingFeedback, QgsProcessingContext, QgsProject

context = QgsProcessingContext()
context.setProject(QgsProject.instance())
feedback = QgsProcessingFeedback()

# Monitor progress
feedback.progressChanged.connect(lambda p: print(f"Progress: {p:.0f}%"))

try:
    result = processing.run("native:dissolve", {
        'INPUT': layer,
        'FIELD': ['category'],
        'OUTPUT': '/output/dissolved.gpkg'
    }, context=context, feedback=feedback)
except QgsProcessingException as e:
    print(f"Dissolve failed: {e}")
```

## Example 4: Background Task Execution

```python
from qgis.core import (
    QgsApplication, QgsProcessingAlgRunnerTask,
    QgsProcessingContext, QgsProcessingFeedback, QgsProject
)

context = QgsProcessingContext()
context.setProject(QgsProject.instance())
feedback = QgsProcessingFeedback()

alg = QgsApplication.processingRegistry().algorithmById('native:buffer')
params = {
    'INPUT': layer,
    'DISTANCE': 100,
    'OUTPUT': 'memory:'
}

task = QgsProcessingAlgRunnerTask(alg, params, context, feedback)

def on_complete(successful, results):
    if successful:
        output_layer = context.getMapLayer(results['OUTPUT'])
        QgsProject.instance().addMapLayer(output_layer)
        print("Background buffer complete")
    else:
        print("Background buffer failed")

task.executed.connect(on_complete)
QgsApplication.taskManager().addTask(task)
```

## Example 5: Algorithm Chaining

```python
import processing

# Step 1: Reproject to projected CRS for accurate distance calculations
reprojected = processing.run("native:reprojectlayer", {
    'INPUT': input_layer,
    'TARGET_CRS': 'EPSG:28992',
    'OUTPUT': 'memory:'
})

# Step 2: Buffer using projected coordinates
buffered = processing.run("native:buffer", {
    'INPUT': reprojected['OUTPUT'],
    'DISTANCE': 500,
    'SEGMENTS': 8,
    'OUTPUT': 'memory:'
})

# Step 3: Dissolve all buffers into one polygon
dissolved = processing.run("native:dissolve", {
    'INPUT': buffered['OUTPUT'],
    'OUTPUT': 'memory:'
})

# Step 4: Save final result to file
final = processing.run("native:fixgeometries", {
    'INPUT': dissolved['OUTPUT'],
    'OUTPUT': '/output/service_area.gpkg'
})
```

## Example 6: Batch Processing Multiple Files

```python
import os
import processing
from qgis.core import QgsProcessingException

input_dir = '/data/input_layers/'
output_dir = '/data/buffered/'
os.makedirs(output_dir, exist_ok=True)

input_files = [f for f in os.listdir(input_dir) if f.endswith('.gpkg')]

results = []
for input_file in input_files:
    input_path = os.path.join(input_dir, input_file)
    output_path = os.path.join(output_dir, f'buffered_{input_file}')

    try:
        result = processing.run("native:buffer", {
            'INPUT': input_path,
            'DISTANCE': 50,
            'SEGMENTS': 5,
            'OUTPUT': output_path
        })
        results.append((input_file, 'success'))
    except QgsProcessingException as e:
        results.append((input_file, f'failed: {e}'))

# Report
for filename, status in results:
    print(f"{filename}: {status}")
```

## Example 7: Complete Custom QgsProcessingAlgorithm

```python
from qgis.PyQt.QtCore import QCoreApplication
from qgis.core import (
    QgsProcessing,
    QgsProcessingAlgorithm,
    QgsProcessingParameterFeatureSource,
    QgsProcessingParameterFeatureSink,
    QgsProcessingParameterNumber,
    QgsProcessingParameterField,
    QgsProcessingException,
    QgsFeatureSink,
    QgsWkbTypes,
)


class BufferByFieldAlgorithm(QgsProcessingAlgorithm):
    """Buffers features using a numeric field value as the distance."""

    INPUT = 'INPUT'
    DISTANCE_FIELD = 'DISTANCE_FIELD'
    SEGMENTS = 'SEGMENTS'
    OUTPUT = 'OUTPUT'

    def name(self):
        return 'bufferbyfieldvalue'

    def displayName(self):
        return self.tr('Buffer by field value')

    def group(self):
        return self.tr('Custom vector tools')

    def groupId(self):
        return 'customvectortools'

    def shortHelpString(self):
        return self.tr(
            'Creates buffers around features using a numeric field '
            'value as the buffer distance for each feature.'
        )

    def tags(self):
        return ['buffer', 'variable', 'field', 'distance']

    def initAlgorithm(self, config=None):
        self.addParameter(
            QgsProcessingParameterFeatureSource(
                self.INPUT,
                self.tr('Input layer'),
                [QgsProcessing.TypeVectorAnyGeometry]
            )
        )
        self.addParameter(
            QgsProcessingParameterField(
                self.DISTANCE_FIELD,
                self.tr('Distance field'),
                parentLayerParameterName=self.INPUT,
                type=QgsProcessingParameterField.Numeric
            )
        )
        self.addParameter(
            QgsProcessingParameterNumber(
                self.SEGMENTS,
                self.tr('Segments'),
                type=QgsProcessingParameterNumber.Integer,
                defaultValue=5,
                minValue=1
            )
        )
        self.addParameter(
            QgsProcessingParameterFeatureSink(
                self.OUTPUT,
                self.tr('Buffered')
            )
        )

    def processAlgorithm(self, parameters, context, feedback):
        source = self.parameterAsSource(parameters, self.INPUT, context)
        if source is None:
            raise QgsProcessingException(
                self.invalidSourceError(parameters, self.INPUT)
            )

        field_name = self.parameterAsString(
            parameters, self.DISTANCE_FIELD, context
        )
        segments = self.parameterAsInt(parameters, self.SEGMENTS, context)

        (sink, dest_id) = self.parameterAsSink(
            parameters, self.OUTPUT, context,
            source.fields(),
            QgsWkbTypes.Polygon,
            source.sourceCrs()
        )
        if sink is None:
            raise QgsProcessingException(
                self.invalidSinkError(parameters, self.OUTPUT)
            )

        total = 100.0 / source.featureCount() if source.featureCount() else 0

        for current, feature in enumerate(source.getFeatures()):
            if feedback.isCanceled():
                break

            distance = feature[field_name]
            if distance is None or distance == 0:
                feedback.pushInfo(
                    f'Skipping feature {feature.id()}: no valid distance'
                )
                continue

            buffered_geom = feature.geometry().buffer(distance, segments)
            feature.setGeometry(buffered_geom)
            sink.addFeature(feature, QgsFeatureSink.FastInsert)

            feedback.setProgress(int(current * total))

        return {self.OUTPUT: dest_id}

    def createInstance(self):
        return BufferByFieldAlgorithm()

    def tr(self, string):
        return QCoreApplication.translate('Processing', string)
```

## Example 8: @alg Decorator Script Algorithm

```python
from qgis.processing import alg

@alg(name='my_buffer_script',
     label='My Buffer Script',
     group='My Scripts',
     group_label='My Script Group')
@alg.input(type=alg.SOURCE, name='INPUT', label='Input layer')
@alg.input(type=alg.DISTANCE, name='DISTANCE', label='Buffer distance',
           default=10.0)
@alg.input(type=alg.SINK, name='OUTPUT', label='Output layer')
def my_buffer(instance, parameters, context, feedback, inputs):
    """Buffer features by a fixed distance."""
    source = instance.parameterAsSource(parameters, 'INPUT', context)
    distance = instance.parameterAsDouble(parameters, 'DISTANCE', context)

    (sink, dest_id) = instance.parameterAsSink(
        parameters, 'OUTPUT', context,
        source.fields(), source.wkbType(), source.sourceCrs()
    )

    total = 100.0 / source.featureCount() if source.featureCount() else 0

    for current, feature in enumerate(source.getFeatures()):
        if feedback.isCanceled():
            break
        feature.setGeometry(feature.geometry().buffer(distance, 5))
        sink.addFeature(feature)
        feedback.setProgress(int(current * total))

    return {'OUTPUT': dest_id}
```

**Important**: `@alg` scripts are ALWAYS added to the Processing Scripts provider. They CANNOT be used in plugins. For plugin algorithms, ALWAYS use the full `QgsProcessingAlgorithm` subclass.

## Example 9: Custom Processing Provider

```python
from qgis.core import QgsProcessingProvider
from .buffer_by_field import BufferByFieldAlgorithm


class MyPluginProvider(QgsProcessingProvider):

    def loadAlgorithms(self):
        self.addAlgorithm(BufferByFieldAlgorithm())

    def id(self):
        return 'myplugin'

    def name(self):
        return 'My Plugin Tools'

    def longName(self):
        return 'My Plugin GIS Tools v1.0'
```

### Register in Plugin

```python
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

## Example 10: Safe Algorithm Runner with Provider Check

```python
from qgis.core import QgsApplication, QgsProcessingException
import processing


def safe_run(algorithm_id, params, context=None, feedback=None):
    """Run a processing algorithm after verifying it exists."""
    registry = QgsApplication.processingRegistry()
    alg = registry.algorithmById(algorithm_id)
    if alg is None:
        raise QgsProcessingException(
            f'Algorithm "{algorithm_id}" not found. '
            f'Verify that the provider is installed and enabled.'
        )
    return processing.run(algorithm_id, params,
                          context=context, feedback=feedback)


# Usage
try:
    result = safe_run("grass:v.buffer", {
        'input': layer,
        'distance': 100,
        'output': 'memory:'
    })
except QgsProcessingException as e:
    print(f"Algorithm error: {e}")
```

## Example 11: Discovering Algorithm Parameters

```python
from qgis.core import QgsApplication

registry = QgsApplication.processingRegistry()
alg = registry.algorithmById('native:buffer')

if alg:
    print(f"Algorithm: {alg.displayName()}")
    print(f"Help: {alg.shortHelpString()}")
    print("\nParameters:")
    for param in alg.parameterDefinitions():
        print(f"  {param.name()} ({param.type()}): {param.description()}")
        if hasattr(param, 'defaultValue'):
            print(f"    Default: {param.defaultValue()}")
    print("\nOutputs:")
    for output in alg.outputDefinitions():
        print(f"  {output.name()} ({output.type()}): {output.description()}")
```
