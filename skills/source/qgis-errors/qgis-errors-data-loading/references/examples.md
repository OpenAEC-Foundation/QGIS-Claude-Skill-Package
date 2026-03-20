# qgis-errors-data-loading — Examples

## Example 1: Complete Layer Loading with Full Error Handling

```python
import os
from qgis.core import QgsVectorLayer, QgsProject

def load_vector_layer(path, name, provider="ogr"):
    """Load a vector layer with comprehensive error checking."""
    # Step 1: Verify file exists (for file-based providers)
    if provider in ("ogr", "gdal", "spatialite", "gpx"):
        # Strip URI parameters for file existence check
        file_path = path.split("|")[0].split("?")[0]
        if not os.path.exists(file_path):
            raise FileNotFoundError(f"Data file not found: {file_path}")

    # Step 2: Create layer
    layer = QgsVectorLayer(path, name, provider)

    # Step 3: ALWAYS check validity
    if not layer.isValid():
        dp = layer.dataProvider()
        if dp:
            error_msg = dp.error().message()
        else:
            error_msg = "Provider could not be instantiated"
        raise RuntimeError(f"Failed to load '{name}': {error_msg}")

    # Step 4: Verify CRS
    if not layer.crs().isValid():
        print(f"WARNING: Layer '{name}' has no valid CRS — assign one before analysis")

    # Step 5: Verify features exist
    if layer.featureCount() == 0:
        print(f"WARNING: Layer '{name}' loaded successfully but contains 0 features")

    return layer

# Usage
layer = load_vector_layer("/data/roads.gpkg|layername=roads", "Roads")
QgsProject.instance().addMapLayer(layer)
```

## Example 2: Diagnosing GeoPackage Sublayer Issues

```python
from qgis.core import QgsVectorLayer, QgsDataProvider, QgsProject

def load_all_gpkg_layers(gpkg_path):
    """Load all layers from a GeoPackage with error handling."""
    # First, open the file to inspect sublayers
    temp_layer = QgsVectorLayer(gpkg_path, "temp", "ogr")
    if not temp_layer.isValid():
        raise RuntimeError(f"Cannot open GeoPackage: {gpkg_path}")

    sublayers = temp_layer.dataProvider().subLayers()
    if not sublayers:
        print("GeoPackage contains no vector layers")
        return []

    loaded = []
    for sublayer_info in sublayers:
        parts = sublayer_info.split(QgsDataProvider.SUBLAYER_SEPARATOR)
        layer_name = parts[1]
        uri = f"{gpkg_path}|layername={layer_name}"

        layer = QgsVectorLayer(uri, layer_name, "ogr")
        if layer.isValid():
            QgsProject.instance().addMapLayer(layer)
            loaded.append(layer)
            print(f"Loaded: {layer_name} ({layer.featureCount()} features)")
        else:
            print(f"FAILED: {layer_name}")

    return loaded

layers = load_all_gpkg_layers("/data/project.gpkg")
```

## Example 3: PostGIS Connection with Fallback Diagnostics

```python
from qgis.core import QgsDataSourceUri, QgsVectorLayer

def connect_postgis(host, port, dbname, user, password, schema, table, geom_col, key_col="gid"):
    """Connect to PostGIS with detailed error reporting."""
    uri = QgsDataSourceUri()
    uri.setConnection(host, port, dbname, user, password)
    uri.setDataSource(schema, table, geom_col, "", key_col)

    # ALWAYS use uri.uri(False) to prevent credential leaking
    layer = QgsVectorLayer(uri.uri(False), f"{schema}.{table}", "postgres")

    if not layer.isValid():
        # Provide specific diagnostic guidance
        dp = layer.dataProvider()
        if dp is None:
            print("DIAGNOSIS: PostgreSQL provider not available — check QGIS installation")
        else:
            error = dp.error().message()
            if "could not connect" in error.lower():
                print(f"DIAGNOSIS: Network issue — verify {host}:{port} is reachable")
            elif "authentication failed" in error.lower():
                print(f"DIAGNOSIS: Wrong credentials for user '{user}'")
            elif "does not exist" in error.lower():
                print(f"DIAGNOSIS: Table '{schema}.{table}' not found in database '{dbname}'")
            elif "permission denied" in error.lower():
                print(f"DIAGNOSIS: User '{user}' lacks SELECT on '{schema}.{table}'")
            else:
                print(f"DIAGNOSIS: {error}")
        return None

    return layer

layer = connect_postgis("localhost", "5432", "gisdb", "gisuser", "pass", "public", "parcels", "geom")
```

## Example 4: Fixing Shapefile Encoding

```python
from qgis.core import QgsVectorLayer

def load_shapefile_with_encoding(shp_path, name, encoding="UTF-8"):
    """Load a Shapefile with explicit encoding for .dbf attributes."""
    layer = QgsVectorLayer(shp_path, name, "ogr")
    if not layer.isValid():
        raise RuntimeError(f"Cannot load Shapefile: {shp_path}")

    # Set encoding on the data provider
    layer.dataProvider().setEncoding(encoding)

    # Verify: read first feature's attributes
    feature = next(layer.getFeatures(), None)
    if feature:
        for field in layer.fields():
            value = feature[field.name()]
            if isinstance(value, str):
                print(f"  {field.name()}: {value}")

    return layer

# Try different encodings if text is garbled
for enc in ["UTF-8", "ISO-8859-1", "Windows-1252"]:
    try:
        layer = load_shapefile_with_encoding("/data/old_dutch_parcels.shp", "Parcels", enc)
        print(f"Encoding {enc} appears correct")
        break
    except Exception as e:
        print(f"Encoding {enc} failed: {e}")
```

## Example 5: Large Dataset with Spatial Filter

```python
from qgis.core import QgsVectorLayer, QgsFeatureRequest, QgsRectangle

def count_features_in_bbox(layer_path, bbox, provider="ogr"):
    """Efficiently count features within a bounding box without loading all data."""
    layer = QgsVectorLayer(layer_path, "temp", provider)
    if not layer.isValid():
        raise RuntimeError(f"Cannot load: {layer_path}")

    total = layer.featureCount()
    print(f"Total features in dataset: {total}")

    # Use QgsFeatureRequest to filter server-side
    request = QgsFeatureRequest()
    request.setFilterRect(bbox)
    request.setFlags(QgsFeatureRequest.NoGeometry)  # Skip geometry for counting
    request.setSubsetOfAttributes([])  # No attributes needed

    count = 0
    for _ in layer.getFeatures(request):
        count += 1

    print(f"Features in bounding box: {count}")
    return count

bbox = QgsRectangle(100000, 400000, 110000, 410000)
count_features_in_bbox("/data/huge_parcels.gpkg|layername=parcels", bbox)
```

## Example 6: WMS Layer Loading with Validation

```python
from qgis.core import QgsRasterLayer, QgsProject

def load_wms_layer(service_url, layer_name, crs="EPSG:4326", img_format="image/png"):
    """Load a WMS layer with full error handling."""
    uri = (
        f"crs={crs}"
        f"&format={img_format}"
        f"&layers={layer_name}"
        f"&styles"
        f"&url={service_url}"
    )

    layer = QgsRasterLayer(uri, f"WMS: {layer_name}", "wms")

    if not layer.isValid():
        print(f"WMS FAILED — Checklist:")
        print(f"  1. Is the URL reachable? {service_url}")
        print(f"  2. Does layer '{layer_name}' exist in GetCapabilities?")
        print(f"  3. Does the service support CRS {crs}?")
        print(f"  4. Is the format {img_format} supported?")
        return None

    QgsProject.instance().addMapLayer(layer)
    return layer

layer = load_wms_layer(
    "https://geodata.nationaalgeoregister.nl/top10nlv2/wms",
    "top10nlv2",
    crs="EPSG:28992"
)
```

## Example 7: Assigning Missing CRS

```python
from qgis.core import QgsVectorLayer, QgsCoordinateReferenceSystem, QgsProject

def load_with_crs_check(path, name, expected_crs="EPSG:4326"):
    """Load a layer and assign CRS if missing."""
    layer = QgsVectorLayer(path, name, "ogr")
    if not layer.isValid():
        raise RuntimeError(f"Cannot load: {path}")

    if not layer.crs().isValid():
        print(f"Layer '{name}' has no CRS — assigning {expected_crs}")
        crs = QgsCoordinateReferenceSystem(expected_crs)
        if not crs.isValid():
            raise RuntimeError(f"Invalid CRS: {expected_crs}")
        layer.setCrs(crs)
    elif layer.crs().authid() != expected_crs:
        print(f"WARNING: Layer CRS is {layer.crs().authid()}, expected {expected_crs}")

    QgsProject.instance().addMapLayer(layer)
    return layer

layer = load_with_crs_check("/data/old_survey.shp", "Survey", "EPSG:28992")
```

## Example 8: GeoPackage Lock Detection and Recovery

```python
import os
from qgis.core import QgsVectorLayer

def load_gpkg_safe(gpkg_path, layer_name):
    """Load a GeoPackage layer with lock file detection."""
    # Check for stale lock files
    lock_extensions = ["-wal", "-shm", "-journal"]
    locks_found = []
    for ext in lock_extensions:
        lock_path = gpkg_path + ext
        if os.path.exists(lock_path):
            locks_found.append(lock_path)

    if locks_found:
        print(f"WARNING: Lock files detected for {gpkg_path}:")
        for lf in locks_found:
            size = os.path.getsize(lf)
            print(f"  {lf} ({size} bytes)")
        print("Another process may be writing to this file.")

    uri = f"{gpkg_path}|layername={layer_name}"
    layer = QgsVectorLayer(uri, layer_name, "ogr")

    if not layer.isValid():
        if locks_found:
            print("DIAGNOSIS: Lock files present — close other QGIS instances or database tools")
        raise RuntimeError(f"Cannot load {layer_name} from {gpkg_path}")

    return layer

layer = load_gpkg_safe("/data/project.gpkg", "buildings")
```
