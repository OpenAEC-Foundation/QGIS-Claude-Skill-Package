# API Signatures Reference (QGIS CRS)

## QgsCoordinateReferenceSystem

Represents a coordinate reference system (CRS). Immutable after creation.

### Constructors

```python
# From string identifier (preferred method)
QgsCoordinateReferenceSystem(definition: str) -> QgsCoordinateReferenceSystem
# Supported prefixes: "EPSG:", "POSTGIS:", "INTERNAL:", "PROJ:", "WKT:"

# Empty (invalid) CRS
QgsCoordinateReferenceSystem() -> QgsCoordinateReferenceSystem

# Static factory methods
QgsCoordinateReferenceSystem.fromEpsgId(epsg: int) -> QgsCoordinateReferenceSystem
QgsCoordinateReferenceSystem.fromWkt(wkt: str) -> QgsCoordinateReferenceSystem
QgsCoordinateReferenceSystem.fromProj(proj: str) -> QgsCoordinateReferenceSystem
QgsCoordinateReferenceSystem.fromOgcWmsCrs(ogc: str) -> QgsCoordinateReferenceSystem
```

### Validation

```python
crs.isValid() -> bool
# Returns True if the CRS was successfully resolved. ALWAYS check after creation.
```

### Identity and Description

```python
crs.authid() -> str
# Authority identifier, e.g., "EPSG:4326". Returns empty string if no authority match.

crs.description() -> str
# Human-readable name, e.g., "WGS 84"

crs.srsid() -> int
# QGIS internal SRS ID (different from EPSG code)

crs.postgisSrid() -> int
# PostGIS SRID value, e.g., 4326
```

### Properties

```python
crs.isGeographic() -> bool
# True for geographic CRS (units are degrees), False for projected CRS (units are meters/feet)

crs.mapUnits() -> Qgis.DistanceUnit
# Returns the native unit: Qgis.DistanceUnit.Degrees, .Meters, .Feet, etc.

crs.projectionAcronym() -> str
# Projection type acronym, e.g., "longlat", "utm", "tmerc"

crs.ellipsoidAcronym() -> str
# Ellipsoid identifier, e.g., "EPSG:7030"
```

### Export

```python
crs.toProj() -> str
# Full Proj string representation

crs.toWkt(variant: Qgis.CrsWktVariant = Qgis.CrsWktVariant.Wkt2_2019) -> str
# Full WKT string. Default is WKT2:2019 format.
```

### Comparison

```python
crs1 == crs2  # Compares by authority ID
crs1 != crs2
```

---

## QgsCoordinateTransform

Transforms coordinates between two CRS. Requires a transform context.

### Constructor

```python
QgsCoordinateTransform(
    source: QgsCoordinateReferenceSystem,
    destination: QgsCoordinateReferenceSystem,
    context: QgsCoordinateTransformContext
) -> QgsCoordinateTransform

QgsCoordinateTransform(
    source: QgsCoordinateReferenceSystem,
    destination: QgsCoordinateReferenceSystem,
    project: QgsProject
) -> QgsCoordinateTransform

# NEVER use the bare constructor without context:
QgsCoordinateTransform() -> QgsCoordinateTransform  # Creates invalid transform
```

### Transform Methods

```python
xform.transform(point: QgsPointXY) -> QgsPointXY
# Transforms a single point. Default direction: ForwardTransform.

xform.transform(
    point: QgsPointXY,
    direction: QgsCoordinateTransform.TransformDirection
) -> QgsPointXY
# Direction: QgsCoordinateTransform.ForwardTransform or .ReverseTransform

xform.transform(x: float, y: float) -> QgsPointXY
# Transforms raw x, y coordinates.

xform.transformBoundingBox(
    rect: QgsRectangle,
    direction: QgsCoordinateTransform.TransformDirection = ForwardTransform,
    handle180Crossover: bool = False
) -> QgsRectangle
# Transforms a bounding box. Set handle180Crossover=True for boxes crossing the antimeridian.
```

### Geometry Transformation

```python
# QgsGeometry has its own transform method that takes a QgsCoordinateTransform
geometry.transform(xform: QgsCoordinateTransform) -> Qgis.GeometryOperationResult
# Transforms the geometry IN-PLACE. Returns 0 (Success) on success.
```

### Properties

```python
xform.sourceCrs() -> QgsCoordinateReferenceSystem
xform.destinationCrs() -> QgsCoordinateReferenceSystem
xform.isValid() -> bool
xform.isShortCircuited() -> bool
# Returns True if source and destination CRS are identical (no transformation needed)
```

### Transform Direction Enum

```python
QgsCoordinateTransform.ForwardTransform   # source -> destination
QgsCoordinateTransform.ReverseTransform   # destination -> source
```

---

## QgsCoordinateTransformContext

Holds project-level datum transformation settings. Determines which datum transformation pipeline to use for CRS pairs.

### Getting the Context

```python
# ALWAYS use project context (includes user preferences)
context = QgsProject.instance().transformContext()

# Bare constructor — ONLY for standalone scripts with no project
context = QgsCoordinateTransformContext()
```

### Methods

```python
context.addCoordinateOperation(
    source: QgsCoordinateReferenceSystem,
    destination: QgsCoordinateReferenceSystem,
    operation: str,
    allowFallback: bool = True
) -> bool
# Register a specific coordinate operation (Proj pipeline) for a CRS pair.

context.removeCoordinateOperation(
    source: QgsCoordinateReferenceSystem,
    destination: QgsCoordinateReferenceSystem
) -> None
# Remove a registered operation for a CRS pair.

context.hasTransform(
    source: QgsCoordinateReferenceSystem,
    destination: QgsCoordinateReferenceSystem
) -> bool
# Check if a specific operation is registered for this CRS pair.

context.calculateCoordinateOperation(
    source: QgsCoordinateReferenceSystem,
    destination: QgsCoordinateReferenceSystem
) -> str
# Get the Proj pipeline string for a CRS pair.
```

---

## QgsDistanceArea

Calculates distances and areas, supporting both ellipsoidal and planimetric measurement.

### Setup

```python
d = QgsDistanceArea()

d.setSourceCrs(
    crs: QgsCoordinateReferenceSystem,
    context: QgsCoordinateTransformContext
) -> None
# Set the CRS of the geometries being measured.

d.setEllipsoid(ellipsoid: str) -> bool
# Set the ellipsoid for calculations. Common values: "WGS84", "GRS80", "Sketchy".
# Returns True if ellipsoid was set successfully.
# ALWAYS set this for accurate measurements on geographic CRS.
```

### Distance Measurement

```python
d.measureLine(p1: QgsPointXY, p2: QgsPointXY) -> float
# Distance between two points in meters (ellipsoidal) or CRS units (planimetric).

d.measureLine(points: list[QgsPointXY]) -> float
# Total distance along a polyline.

d.measureLineProjected(p: QgsPointXY, distance: float, azimuth: float) -> float
# Measure distance from a point along an azimuth. Returns the geodesic distance.
```

### Area Measurement

```python
d.measureArea(geometry: QgsGeometry) -> float
# Area of a polygon geometry in square meters (ellipsoidal) or CRS square units (planimetric).

d.measurePerimeter(geometry: QgsGeometry) -> float
# Perimeter of a polygon geometry.
```

### Unit Conversion

```python
d.convertLengthMeasurement(length: float, toUnit: Qgis.DistanceUnit) -> float
# Convert a length measurement to the target unit.
# Units: Qgis.DistanceUnit.Meters, .Kilometers, .Feet, .NauticalMiles, .Yards, .Miles

d.convertAreaMeasurement(area: float, toUnit: Qgis.AreaUnit) -> float
# Convert an area measurement to the target unit.
# Units: Qgis.AreaUnit.SquareMeters, .SquareKilometers, .Hectares, .Acres, .SquareFeet

d.lengthUnits() -> Qgis.DistanceUnit
# Returns the unit of length measurements based on current CRS and ellipsoid settings.

d.areaUnits() -> Qgis.AreaUnit
# Returns the unit of area measurements based on current CRS and ellipsoid settings.
```

### Properties

```python
d.sourceCrs() -> QgsCoordinateReferenceSystem
d.ellipsoid() -> str
d.ellipsoidSemiMajor() -> float
d.ellipsoidSemiMinor() -> float
d.ellipsoidInverseFlattening() -> float
```

---

## Qgis.DistanceUnit Enum

```python
Qgis.DistanceUnit.Meters
Qgis.DistanceUnit.Kilometers
Qgis.DistanceUnit.Feet
Qgis.DistanceUnit.NauticalMiles
Qgis.DistanceUnit.Yards
Qgis.DistanceUnit.Miles
Qgis.DistanceUnit.Degrees
Qgis.DistanceUnit.Centimeters
Qgis.DistanceUnit.Millimeters
Qgis.DistanceUnit.Inches
Qgis.DistanceUnit.Unknown
```

## Qgis.AreaUnit Enum

```python
Qgis.AreaUnit.SquareMeters
Qgis.AreaUnit.SquareKilometers
Qgis.AreaUnit.SquareFeet
Qgis.AreaUnit.SquareYards
Qgis.AreaUnit.SquareMiles
Qgis.AreaUnit.Hectares
Qgis.AreaUnit.Acres
Qgis.AreaUnit.SquareNauticalMiles
Qgis.AreaUnit.SquareDegrees
Qgis.AreaUnit.SquareCentimeters
Qgis.AreaUnit.SquareMillimeters
Qgis.AreaUnit.SquareInches
Qgis.AreaUnit.Unknown
```
