# Research Database Queries

This workflow queries the DesignSafe research databases using dapi, covering database exploration, geographic filtering, multi-table joins, aggregate statistics, and data export. The databases contain geotechnical field data, liquefaction observations, earthquake event records, and CPT soundings collected from research sites worldwide.

## Initialize the Client

```python
from dapi import DSClient

ds = DSClient()
```

No special credentials are needed for the public databases. The default read-only credentials are included in dapi and work out of the box.

## Available Databases

| Database | Code | Domain |
|---|---|---|
| NGL | `ngl` | Geotechnical and liquefaction |
| Earthquake Recovery | `eq` | Social and economic impacts |
| VP | `vp` | Model validation |

All examples below use the NGL (Next Generation Liquefaction) database. The same `read_sql()` method applies to the other databases through `ds.db.eq` and `ds.db.vp`.

## Explore the Database Structure

Start by discovering what tables exist and what columns they contain.

```python
# List all tables in the NGL database
tables = ds.db.ngl.read_sql("SHOW TABLES")
print(tables)
```

Inspect the column definitions for a specific table.

```python
structure = ds.db.ngl.read_sql("DESCRIBE SITE")
print(structure)
```

This returns the column name, type, nullability, and default values for each field. Knowing the column names and types helps you write accurate queries.

## Basic Queries

Count the total number of sites in the database.

```python
count = ds.db.ngl.read_sql("SELECT COUNT(*) as total FROM SITE")
print(f"Total sites: {count['total'].iloc[0]}")
```

List sites with their coordinates and geological classification.

```python
sites = ds.db.ngl.read_sql("""
    SELECT SITE_ID, SITE_NAME, SITE_LAT, SITE_LON, SITE_GEOL
    FROM SITE
    WHERE SITE_STAT = 1
    ORDER BY SITE_NAME
    LIMIT 20
""")
print(sites)
```

Every `read_sql()` call returns a pandas DataFrame. You can use all standard pandas operations on the results (filtering, sorting, grouping, plotting).

## Geographic Filtering

Filter sites by latitude and longitude to focus on a specific region. This example selects sites within an approximate bounding box for California.

```python
california = ds.db.ngl.read_sql("""
    SELECT SITE_NAME, SITE_LAT, SITE_LON
    FROM SITE
    WHERE SITE_LAT BETWEEN 32.0 AND 42.0
      AND SITE_LON BETWEEN -125.0 AND -114.0
    ORDER BY SITE_NAME
""")
print(f"California sites: {len(california)}")
print(california.head(10))
```

For Japan, adjust the bounding box accordingly.

```python
japan = ds.db.ngl.read_sql("""
    SELECT SITE_NAME, SITE_LAT, SITE_LON
    FROM SITE
    WHERE SITE_LAT BETWEEN 30.0 AND 46.0
      AND SITE_LON BETWEEN 128.0 AND 146.0
    ORDER BY SITE_NAME
""")
print(f"Japan sites: {len(japan)}")
```

## Liquefaction Data and Table Joins

The NGL database stores site metadata, field records, and liquefaction observations in separate tables. Join them to get a combined view.

```python
liq_sites = ds.db.ngl.read_sql("""
    SELECT DISTINCT
        s.SITE_NAME,
        s.SITE_LAT,
        s.SITE_LON,
        f.FLDN_LIQ
    FROM SITE s
    JOIN FLDN f ON s.SITE_ID = f.SITE_ID
    WHERE f.FLDN_LIQ = 'Yes'
    ORDER BY s.SITE_NAME
""")
print(f"Sites with observed liquefaction: {len(liq_sites)}")
print(liq_sites.head(10))
```

A three-table join pulls in event information alongside site and record data.

```python
combined = ds.db.ngl.read_sql("""
    SELECT
        s.SITE_NAME,
        s.SITE_LAT,
        s.SITE_LON,
        e.EVENT_NAME,
        e.EVENT_MAG
    FROM SITE s
    JOIN RECORD r ON s.SITE_ID = r.SITE_ID
    JOIN EVENT e ON r.EVENT_ID = e.EVENT_ID
    WHERE s.SITE_STAT = 1
      AND r.RECORD_STAT = 1
    ORDER BY e.EVENT_MAG DESC
    LIMIT 50
""")
print(combined)
```

## Earthquake Events

Query the EVENT table for recent earthquakes, sorted by date.

```python
quakes = ds.db.ngl.read_sql("""
    SELECT DISTINCT EVENT_NAME, EVENT_DATE, EVENT_MAG
    FROM EVENT
    WHERE EVENT_STAT = 1
    ORDER BY EVENT_DATE DESC
    LIMIT 20
""")
print(quakes)
```

Filter by magnitude threshold.

```python
large_quakes = ds.db.ngl.read_sql("""
    SELECT EVENT_NAME, EVENT_DATE, EVENT_MAG, EVENT_LAT, EVENT_LON
    FROM EVENT
    WHERE EVENT_MAG >= 7.0
      AND EVENT_STAT = 1
    ORDER BY EVENT_MAG DESC
""")
print(f"Events with magnitude >= 7.0: {len(large_quakes)}")
print(large_quakes)
```

## CPT Data Summary

Aggregate statistics give a quick overview of the cone penetration test data.

```python
cpt_summary = ds.db.ngl.read_sql("""
    SELECT
        COUNT(*) as total_soundings,
        AVG(CPT_DEPTH) as avg_depth,
        MIN(CPT_DEPTH) as min_depth,
        MAX(CPT_DEPTH) as max_depth
    FROM CPT
    WHERE CPT_STAT = 1
""")
print(cpt_summary)
```

Pull individual sounding data for a specific depth range.

```python
shallow_cpt = ds.db.ngl.read_sql("""
    SELECT *
    FROM SCPG
    WHERE SCPG_DPTH <= 10.0
    LIMIT 500
""")
print(f"Shallow CPT records: {len(shallow_cpt)}")
print(shallow_cpt.describe())
```

## Parameterized Queries

Always use parameterized queries when incorporating user input or variable values. The `%s` placeholder prevents SQL injection and handles escaping automatically.

```python
site_name = "Amagasaki"
site = ds.db.ngl.read_sql(
    "SELECT * FROM SITE WHERE SITE_NAME = %s",
    params=[site_name],
)
print(site)
```

Range queries work the same way.

```python
min_lat, max_lat = 35.0, 38.0
sites_in_range = ds.db.ngl.read_sql(
    "SELECT SITE_NAME, SITE_LAT, SITE_LON FROM SITE WHERE SITE_LAT BETWEEN %s AND %s",
    params=[min_lat, max_lat],
)
print(f"Sites between {min_lat} and {max_lat}: {len(sites_in_range)}")
```

Pattern matching with LIKE also works with parameterized queries.

```python
pattern = "%Sand%"
sandy = ds.db.ngl.read_sql(
    "SELECT * FROM SITE WHERE SITE_NAME LIKE %s",
    params=[pattern],
)
print(sandy)
```

## Export to CSV, Excel, and GeoJSON

All query results are DataFrames, so pandas export methods work directly.

### CSV

```python
combined.to_csv("ngl_data.csv", index=False)
```

### Excel

```python
combined.to_excel("ngl_data.xlsx", index=False)
```

### GeoJSON

For spatial data with latitude and longitude columns, convert to GeoJSON using geopandas.

```python
import geopandas as gpd
from shapely.geometry import Point

geometry = [Point(lon, lat) for lon, lat in zip(combined["SITE_LON"], combined["SITE_LAT"])]
gdf = gpd.GeoDataFrame(combined, geometry=geometry, crs="EPSG:4326")
gdf.to_file("ngl_sites.geojson", driver="GeoJSON")
```

The resulting GeoJSON file can be loaded into QGIS, Leaflet, or any GIS viewer for spatial visualization.

## Best Practices

- Use `LIMIT` on exploratory queries to avoid pulling entire tables into memory. Some tables contain millions of rows.
- Use parameterized queries (`%s` placeholders) instead of f-strings or string formatting for any user-provided values. This prevents SQL injection and handles special characters.
- For large result sets, paginate with `LIMIT` and `OFFSET` to process data in manageable chunks.

```python
page_size = 1000
offset = 0
all_chunks = []

while True:
    chunk = ds.db.ngl.read_sql(
        "SELECT * FROM SCPG LIMIT %s OFFSET %s",
        params=[page_size, offset],
    )
    if chunk.empty:
        break
    all_chunks.append(chunk)
    offset += page_size

import pandas as pd
full_dataset = pd.concat(all_chunks, ignore_index=True)
print(f"Total records retrieved: {len(full_dataset)}")
```

- Close long-running sessions and recreate `DSClient()` if you hit token expiration errors. JWT tokens expire after a few hours.
- The databases are read-only. INSERT, UPDATE, and DELETE statements will fail.

## Complete Workflow

```python
from dapi import DSClient
import pandas as pd

ds = DSClient()

# Explore
tables = ds.db.ngl.read_sql("SHOW TABLES")
print(tables)

# Query sites with liquefaction observations
liq = ds.db.ngl.read_sql("""
    SELECT
        s.SITE_NAME,
        s.SITE_LAT,
        s.SITE_LON,
        e.EVENT_NAME,
        e.EVENT_MAG,
        f.FLDN_LIQ
    FROM SITE s
    JOIN FLDN f ON s.SITE_ID = f.SITE_ID
    JOIN RECORD r ON s.SITE_ID = r.SITE_ID
    JOIN EVENT e ON r.EVENT_ID = e.EVENT_ID
    WHERE f.FLDN_LIQ = 'Yes'
      AND s.SITE_STAT = 1
    ORDER BY e.EVENT_MAG DESC
    LIMIT 100
""")
print(liq)

# Export
liq.to_csv("liquefaction_sites.csv", index=False)
```

## Related

- [dapi documentation](https://designsafe-ci.github.io/dapi/) for the full database API reference
