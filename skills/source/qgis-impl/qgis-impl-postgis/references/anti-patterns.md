# Anti-Patterns (PostGIS / PyQGIS)

## 1. Plain-Text Passwords in URIs

```python
# WRONG: Credentials stored in plain text -- leaks to logs, project files, history
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "mydb", "admin", "s3cret_pw!")
uri.setDataSource("public", "roads", "geom")
layer = QgsVectorLayer(uri.uri(), "Roads", "postgres")  # uri() expands credentials

# CORRECT: Use QgsAuthManager with stored auth config
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "mydb", "", "")
uri.setAuthConfigId("my_auth_config")
uri.setDataSource("public", "roads", "geom")
layer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
```

**WHY**: Plain-text passwords appear in QGIS project files (.qgz/.qgs), server logs, layer metadata, and `uri.uri()` output. `QgsAuthManager` stores credentials in an encrypted database (`qgis-auth.db`) and `uri(False)` prevents credential expansion.

---

## 2. Calling uri() Without False

```python
# WRONG: uri() without argument defaults to True, expanding authcfg to plain credentials
layer = QgsVectorLayer(uri.uri(), "Roads", "postgres")

# ALSO WRONG: Explicitly passing True
layer = QgsVectorLayer(uri.uri(True), "Roads", "postgres")

# CORRECT: ALWAYS pass False to prevent credential expansion
layer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
```

**WHY**: `uri(True)` resolves the `authcfg` reference and embeds the actual username/password into the URI string. This string then appears in layer metadata, debug output, and logs.

---

## 3. Loading Views Without a Key Column

```python
# WRONG: No key column specified for a view
uri.setDataSource("public", "my_view", "geom")
layer = QgsVectorLayer(uri.uri(False), "My View", "postgres")
# Results: duplicate features, missing features, crashes on edit, wrong feature count

# CORRECT: ALWAYS specify a unique key column for views
uri.setDataSource("public", "my_view", "geom", "", "view_id")
layer = QgsVectorLayer(uri.uri(False), "My View", "postgres")
```

**WHY**: QGIS uses the primary key to uniquely identify features. Tables have primary keys in their schema, but views do not. Without an explicit key column, QGIS uses `ctid` (a physical row identifier) which is unstable for views and causes duplicate or missing features.

---

## 4. Manual URI String Concatenation

```python
# WRONG: Manual string building -- vulnerable to injection and escaping errors
uri_str = f"dbname='my db' host=localhost port=5432 user='admin' password='p@ss\"word' table=\"public\".\"roads\" (geom)"
layer = QgsVectorLayer(uri_str, "Roads", "postgres")

# CORRECT: Use QgsDataSourceUri which handles escaping
uri = QgsDataSourceUri()
uri.setConnection("localhost", "5432", "my db", "", "")
uri.setAuthConfigId("my_auth")
uri.setDataSource("public", "roads", "geom")
layer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
```

**WHY**: `QgsDataSourceUri` properly escapes special characters in database names, schema names, table names, and passwords. Manual concatenation breaks with spaces, quotes, backslashes, and other special characters.

---

## 5. Not Checking Layer Validity

```python
# WRONG: Using a layer without checking validity
uri.setDataSource("public", "nonexistent_table", "geom")
layer = QgsVectorLayer(uri.uri(False), "Data", "postgres")
QgsProject.instance().addMapLayer(layer)  # adds an invalid layer silently
for feature in layer.getFeatures():       # iterates over nothing, no error
    process(feature)

# CORRECT: ALWAYS check validity immediately after construction
layer = QgsVectorLayer(uri.uri(False), "Data", "postgres")
if not layer.isValid():
    error_msg = layer.error().message()
    raise RuntimeError(f"Failed to load PostGIS layer: {error_msg}")
```

**WHY**: PostGIS layer construction never raises exceptions. A wrong host, missing table, bad credentials, or network timeout all produce an invalid layer that silently returns zero features. Code continues executing with empty data.

---

## 6. Using estimatedmetadata for Accurate Counts

```python
# WRONG: Using estimated metadata when exact counts are required
uri.setUseEstimatedMetadata(True)
layer = QgsVectorLayer(uri.uri(False), "Data", "postgres")
total = layer.featureCount()  # returns approximate count from pg_class.reltuples
report.set_total_features(total)  # report shows wrong number

# CORRECT: Disable estimated metadata when accuracy matters
uri.setUseEstimatedMetadata(False)  # default
layer = QgsVectorLayer(uri.uri(False), "Data", "postgres")
total = layer.featureCount()  # runs SELECT COUNT(*) -- accurate but slower
```

**WHY**: With `estimatedmetadata=true`, QGIS reads `pg_class.reltuples` instead of running `SELECT COUNT(*)`. After bulk inserts, deletes, or without recent `ANALYZE`, this value is stale. Use estimated metadata for UI responsiveness on large tables, but NEVER for reporting or logic that depends on exact counts.

---

## 7. SQL Subquery Without Parentheses

```python
# WRONG: SQL query not wrapped in parentheses
uri.setDataSource("", "SELECT * FROM roads WHERE type='highway'", "geom", "", "gid")

# CORRECT: Wrap SQL in parentheses
uri.setDataSource("", "(SELECT * FROM roads WHERE type='highway')", "geom", "", "gid")
```

**WHY**: The PostGIS provider interprets the table argument as a table name unless wrapped in parentheses. Without them, the SQL is treated as a literal table name, causing a "relation does not exist" error.

---

## 8. Missing Key Column in SQL Subqueries

```python
# WRONG: SQL subquery without a key column
uri.setDataSource("", "(SELECT name, geom FROM roads)", "geom")
layer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
# Results: unpredictable feature IDs, editing fails, possible duplicates

# CORRECT: Include a unique key column in SELECT and specify it
uri.setDataSource("", "(SELECT gid, name, geom FROM roads)", "geom", "", "gid")
layer = QgsVectorLayer(uri.uri(False), "Roads", "postgres")
```

**WHY**: Without a key column, QGIS auto-generates feature IDs which are unstable across sessions. Editing, selection, and joins fail or produce incorrect results.

---

## 9. Forgetting to Create Spatial Index After Import

```python
# WRONG: Import data and start querying without an index
processing.run("native:importintopostgis", {
    'INPUT': layer,
    'DATABASE': 'My DB',
    'SCHEMA': 'public',
    'TABLENAME': 'big_dataset',
    'CREATEINDEX': False,  # no spatial index
    'OVERWRITE': True
})
# Spatial queries on big_dataset are now extremely slow

# CORRECT: ALWAYS create a spatial index for spatial tables
processing.run("native:importintopostgis", {
    'INPUT': layer,
    'DATABASE': 'My DB',
    'SCHEMA': 'public',
    'TABLENAME': 'big_dataset',
    'CREATEINDEX': True,   # creates GIST index on geometry column
    'OVERWRITE': True
})
```

**WHY**: Without a spatial index (GiST), every spatial query (intersection, bounding box, nearest neighbor) performs a full table scan. On tables with more than a few thousand rows, this causes severe performance degradation.

---

## 10. Using Wrong Provider String

```python
# WRONG: Using "postgis" as provider name
layer = QgsVectorLayer(uri.uri(False), "Data", "postgis")  # no such provider

# WRONG: Using "postgresql" as provider name
layer = QgsVectorLayer(uri.uri(False), "Data", "postgresql")  # no such provider

# CORRECT: Provider name is "postgres" for vector
layer = QgsVectorLayer(uri.uri(False), "Data", "postgres")

# CORRECT: Provider name is "postgresraster" for raster
rlayer = QgsRasterLayer(uri.uri(False), "Raster", "postgresraster")
```

**WHY**: QGIS registers providers by exact string. The vector provider is `"postgres"` (not "postgis" or "postgresql"). The raster provider is `"postgresraster"`. Using the wrong string creates an invalid layer with no error message.

---

## 11. Connecting Without SSL in Production

```python
# WRONG: Disabling SSL for production database over network
uri.setConnection("remote-db.example.com", "5432", "prod_db", "", "",
                  QgsDataSourceUri.SslDisable)

# CORRECT: Use SSL for remote connections
uri.setConnection("remote-db.example.com", "5432", "prod_db", "", "",
                  QgsDataSourceUri.SslRequire)  # or SslVerifyFull for maximum security
```

**WHY**: Without SSL, database credentials and query data travel in plain text over the network. For localhost connections, `SslDisable` is acceptable. For any remote connection, ALWAYS use `SslRequire` or `SslVerifyFull`.
