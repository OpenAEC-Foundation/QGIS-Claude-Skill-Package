# qgis-errors-projections — Examples

## Example 1: Diagnose CRS Issues for All Project Layers

```python
from qgis.core import QgsProject

def diagnose_crs_issues():
    """Print CRS diagnostic info for every layer in the project."""
    project = QgsProject.instance()
    project_crs = project.crs()
    print(f"Project CRS: {project_crs.authid()} ({project_crs.description()})")
    print(f"Project CRS valid: {project_crs.isValid()}")
    print("-" * 60)

    for layer_id, layer in project.mapLayers().items():
        crs = layer.crs()
        ext = layer.extent()
        print(f"Layer: {layer.name()}")
        print(f"  CRS: {crs.authid()} ({crs.description()})")
        print(f"  CRS valid: {crs.isValid()}")
        print(f"  Geographic: {crs.isGeographic()}")
        print(f"  Map units: {crs.mapUnits()}")
        print(f"  Extent X: {ext.xMinimum():.4f} to {ext.xMaximum():.4f}")
        print(f"  Extent Y: {ext.yMinimum():.4f} to {ext.yMaximum():.4f}")

        # Flag potential issues
        if not crs.isValid():
            print("  ** WARNING: Invalid CRS **")
        if crs.authid() != project_crs.authid():
            print(f"  ** NOTE: Differs from project CRS ({project_crs.authid()}) **")
        print()

diagnose_crs_issues()
```

---

## Example 2: Safe Coordinate Transformation with Validation

```python
from qgis.core import (
    QgsCoordinateReferenceSystem,
    QgsCoordinateTransform,
    QgsProject,
    QgsPointXY
)

def safe_transform(point, source_epsg, dest_epsg):
    """Transform a point between CRS with full error checking."""
    source_crs = QgsCoordinateReferenceSystem(f"EPSG:{source_epsg}")
    dest_crs = QgsCoordinateReferenceSystem(f"EPSG:{dest_epsg}")

    # Validate both CRS
    if not source_crs.isValid():
        raise ValueError(f"Invalid source CRS: EPSG:{source_epsg}")
    if not dest_crs.isValid():
        raise ValueError(f"Invalid destination CRS: EPSG:{dest_epsg}")

    # Create transform with project context
    xform = QgsCoordinateTransform(
        source_crs, dest_crs,
        QgsProject.instance().transformContext()
    )

    if not xform.isValid():
        raise RuntimeError(f"Cannot create transform from EPSG:{source_epsg} to EPSG:{dest_epsg}")

    # Perform transformation
    result = xform.transform(point)
    return result

# Usage: Transform Amsterdam from WGS 84 to Dutch RD New
amsterdam_wgs84 = QgsPointXY(4.8952, 52.3702)  # lon, lat
amsterdam_rd = safe_transform(amsterdam_wgs84, 4326, 28992)
print(f"Amsterdam in RD New: X={amsterdam_rd.x():.2f}, Y={amsterdam_rd.y():.2f}")
# Expected: approximately X=121000, Y=487000
```

---

## Example 3: Reproject Layer Before Geoprocessing

```python
from qgis.core import QgsCoordinateReferenceSystem, QgsProject
import processing

def reproject_and_intersect(layer_a, layer_b, target_epsg):
    """Reproject both layers to a common CRS, then intersect."""
    target_crs = QgsCoordinateReferenceSystem(f"EPSG:{target_epsg}")
    if not target_crs.isValid():
        raise ValueError(f"Invalid target CRS: EPSG:{target_epsg}")

    # Reproject layer A if needed
    if layer_a.crs().authid() != target_crs.authid():
        result_a = processing.run("native:reprojectlayer", {
            'INPUT': layer_a,
            'TARGET_CRS': target_crs,
            'OUTPUT': 'memory:'
        })
        layer_a_proj = result_a['OUTPUT']
    else:
        layer_a_proj = layer_a

    # Reproject layer B if needed
    if layer_b.crs().authid() != target_crs.authid():
        result_b = processing.run("native:reprojectlayer", {
            'INPUT': layer_b,
            'TARGET_CRS': target_crs,
            'OUTPUT': 'memory:'
        })
        layer_b_proj = result_b['OUTPUT']
    else:
        layer_b_proj = layer_b

    # Now intersect with matching CRS
    result = processing.run("native:intersection", {
        'INPUT': layer_a_proj,
        'OVERLAY': layer_b_proj,
        'OUTPUT': 'memory:'
    })
    return result['OUTPUT']
```

---

## Example 4: Detect and Fix Swapped Lat/Lon

```python
from qgis.core import QgsVectorLayer, QgsFeature, QgsGeometry, QgsPointXY

def detect_swapped_coordinates(layer):
    """Detect if a WGS 84 layer has lat/lon swapped (lat in X, lon in Y)."""
    if layer.crs().authid() != "EPSG:4326":
        print("This check is only relevant for EPSG:4326 layers")
        return False

    ext = layer.extent()
    x_range = (ext.xMinimum(), ext.xMaximum())
    y_range = (ext.yMinimum(), ext.yMaximum())

    # For correct lon/lat: X (lon) in -180..180, Y (lat) in -90..90
    # For swapped lat/lon: X would be in -90..90, Y could exceed 90
    x_looks_like_lat = -90 <= x_range[0] and x_range[1] <= 90
    y_looks_like_lon = abs(y_range[0]) > 90 or abs(y_range[1]) > 90

    if x_looks_like_lat and y_looks_like_lon:
        print(f"LIKELY SWAPPED: X range {x_range} looks like latitude, "
              f"Y range {y_range} looks like longitude")
        return True

    print(f"Coordinates appear correct: X (lon) {x_range}, Y (lat) {y_range}")
    return False

def fix_swapped_coordinates(layer):
    """Create a new memory layer with X/Y coordinates swapped."""
    fields = layer.fields()
    mem_layer = QgsVectorLayer(
        f"Point?crs=EPSG:4326",
        f"{layer.name()}_fixed",
        "memory"
    )
    provider = mem_layer.dataProvider()
    provider.addAttributes(fields.toList())
    mem_layer.updateFields()

    features = []
    for feat in layer.getFeatures():
        new_feat = QgsFeature(fields)
        new_feat.setAttributes(feat.attributes())
        geom = feat.geometry()
        if not geom.isNull():
            point = geom.asPoint()
            # Swap X and Y
            new_geom = QgsGeometry.fromPointXY(QgsPointXY(point.y(), point.x()))
            new_feat.setGeometry(new_geom)
        features.append(new_feat)

    provider.addFeatures(features)
    mem_layer.updateExtents()
    return mem_layer
```

---

## Example 5: Verify Datum Transformation Accuracy

```python
from qgis.core import (
    QgsCoordinateReferenceSystem,
    QgsCoordinateTransform,
    QgsProject,
    QgsPointXY
)

def verify_datum_transform(source_epsg, dest_epsg, test_point, expected_point, tolerance_m=1.0):
    """Verify a datum transformation against a known reference point."""
    source_crs = QgsCoordinateReferenceSystem(f"EPSG:{source_epsg}")
    dest_crs = QgsCoordinateReferenceSystem(f"EPSG:{dest_epsg}")

    xform = QgsCoordinateTransform(
        source_crs, dest_crs,
        QgsProject.instance().transformContext()
    )

    result = xform.transform(test_point)

    dx = result.x() - expected_point.x()
    dy = result.y() - expected_point.y()

    # For geographic CRS, rough conversion: 1 degree ~ 111320m at equator
    if dest_crs.isGeographic():
        error_m = ((dx * 111320) ** 2 + (dy * 111320) ** 2) ** 0.5
    else:
        error_m = (dx ** 2 + dy ** 2) ** 0.5

    print(f"Transform result: ({result.x():.8f}, {result.y():.8f})")
    print(f"Expected:         ({expected_point.x():.8f}, {expected_point.y():.8f})")
    print(f"Estimated error:  {error_m:.3f} meters")

    if error_m > tolerance_m:
        print(f"WARNING: Error exceeds tolerance of {tolerance_m}m")
        print("Possible cause: missing datum transformation grid")
        return False

    print("PASS: Within tolerance")
    return True

# Example: Verify ED50 to WGS 84 transformation
verify_datum_transform(
    source_epsg=4230,  # ED50
    dest_epsg=4326,    # WGS 84
    test_point=QgsPointXY(5.0, 52.0),
    expected_point=QgsPointXY(4.99867, 51.99895),  # Approximate known value
    tolerance_m=5.0
)
```

---

## Example 6: Standalone Script with Correct CRS Initialization

```python
import sys
from qgis.core import (
    QgsApplication,
    QgsCoordinateReferenceSystem,
    QgsCoordinateTransform,
    QgsPointXY
)

# Initialize QGIS application (REQUIRED for standalone scripts)
qgs = QgsApplication([], False)

# Platform-specific prefix paths:
# Windows: "C:/Program Files/QGIS 3.34/apps/qgis"
# Linux:   "/usr"
# macOS:   "/Applications/QGIS.app/Contents/MacOS"
qgs.setPrefixPath("C:/Program Files/QGIS 3.34/apps/qgis", True)
qgs.initQgis()

# Verify CRS system is working
test_crs = QgsCoordinateReferenceSystem("EPSG:4326")
if not test_crs.isValid():
    print("ERROR: CRS system not initialized. Check prefix path.")
    qgs.exitQgis()
    sys.exit(1)

# Perform CRS operations
source = QgsCoordinateReferenceSystem("EPSG:4326")
dest = QgsCoordinateReferenceSystem("EPSG:32632")

from qgis.core import QgsCoordinateTransformContext
context = QgsCoordinateTransformContext()
xform = QgsCoordinateTransform(source, dest, context)

point = QgsPointXY(9.0, 48.0)
result = xform.transform(point)
print(f"UTM 32N: {result.x():.2f}, {result.y():.2f}")

# Clean up
qgs.exitQgis()
```

---

## Example 7: Batch Validate CRS for All Layers

```python
from qgis.core import QgsProject

def validate_project_crs():
    """Validate CRS consistency across all project layers. Returns list of issues."""
    project = QgsProject.instance()
    project_crs = project.crs()
    issues = []

    for layer_id, layer in project.mapLayers().items():
        crs = layer.crs()
        name = layer.name()

        # Check 1: CRS validity
        if not crs.isValid():
            issues.append(f"CRITICAL: '{name}' has invalid CRS")
            continue

        # Check 2: CRS matches project
        if crs.authid() != project_crs.authid():
            issues.append(
                f"MISMATCH: '{name}' uses {crs.authid()}, "
                f"project uses {project_crs.authid()}"
            )

        # Check 3: Extent sanity for geographic CRS
        if crs.isGeographic():
            ext = layer.extent()
            if abs(ext.xMinimum()) > 180 or abs(ext.xMaximum()) > 180:
                issues.append(
                    f"SUSPECT: '{name}' has geographic CRS but "
                    f"X values exceed 180 degrees (possible wrong CRS)"
                )
            if abs(ext.yMinimum()) > 90 or abs(ext.yMaximum()) > 90:
                issues.append(
                    f"SUSPECT: '{name}' has geographic CRS but "
                    f"Y values exceed 90 degrees (possible wrong CRS)"
                )

    if not issues:
        print("All layers pass CRS validation")
    else:
        for issue in issues:
            print(issue)

    return issues

validate_project_crs()
```
