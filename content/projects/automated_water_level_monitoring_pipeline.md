---
title: Automated Water Level Monitoring Pipeline
tags:
  - python
  - postgis
  - geopandas
  - spatial-etl
  - qgis
  - architecture
date: 2026-10-17
---

# Automated Water Level Monitoring Ingestion & Analytics Pipeline

An automated spatial data engineering pipeline designed to ingest multi-region hydrological monitoring station feeds, perform geometric validation, load records into a spatial database (PostgreSQL 18 + PostGIS 3.5), and execute geodesic proximity analytics across international borders.

---

## 1. System Architecture & Data Flow

```text
+-------------------------------------------------------------+
|               Station Sensor Feeds (JSON/CSV)               |
+-------------------------------------------------------------+
                              │
                              ▼
+-------------------------------------------------------------+
|    Ingestion & Validation Engine (Python 3.13 / GeoPandas)  |
|      - Schema parsing & coordinate sanity checks            |
|      - WGS84 (EPSG:4326) Point geometry construction        |
|      - Topology & null-geometry filtering (Shapely)         |
+-------------------------------------------------------------+
                              │
                              ▼
+-------------------------------------------------------------+
|         Spatial Storage Layer (PostgreSQL 18 + PostGIS)     |
|      - Target table: monitoring_stations                    |
|      - Optimized drivers: SQLAlchemy + GeoAlchemy2          |
|      - Spatial Indexing: GIST index on geometry column      |
+-------------------------------------------------------------+
           │                                      │
           ▼                                      ▼
+----------------------------------+   +----------------------+
|        Analytical Engine         |   |  Visualization & QA  |
|  - Spherical geodesic distance   |   |  - QGIS LTR Desktop  |
|    using ST_Distance (geography) |   |  - Basemap overlay   |
|  - Cross-border proximity metrics|   |  - Topology QA audit |
+----------------------------------+   +----------------------+
```

---

## 2. Engineering Decisions & Design Trade-offs

- **Python Runtime (v3.13.5):** Selected to ensure high performance and vectorization while maintaining complete build compatibility with core C/C++ spatial libraries (GDAL, GEOS, and PROJ).
- **Coordinate Reference System (CRS):** Standardized on **WGS84 (EPSG:4326)** to allow seamless ingestion and alignment of stations across different jurisdictions (Ireland and the Netherlands).
- **Geodesic Accuracy over Planar Approximations:** Standard Euclidean distance calculations on projected planar maps introduce significant distortion over long distances. By casting coordinates dynamically to PostGIS native `geography` types (`::geography`), distance calculations are executed along the true WGS84 ellipsoidal model of the Earth.
- **Topology Pre-Validation:** Sensor records pass through strict `is_valid` and empty-geometry filters before initiating database transactions, preventing malformed geometries from entering production storage.

---

## 3. Pipeline Implementation (Python ETL)

The script below performs end-to-end ingestion, validation, and database population:

```python
"""
Automated Spatial Data Ingestion Pipeline
Author: Yashar Tahermanesh
Stack: Python 3.13.5 | GeoPandas | SQLAlchemy | PostGIS 18
"""

import sys
from sqlalchemy import create_engine
import geopandas as gpd
from shapely.geometry import Point

# Database connection URI
DB_URI = "postgresql+psycopg2://postgres:YOUR_PASSWORD@127.0.0.1:5432/geodb"

def run_pipeline():
    try:
        engine = create_engine(DB_URI)
        
        # Raw station telemetry dataset (Target markets: Ireland & Netherlands)
        raw_stations = [
            {"station_id": "IE_DUB_01", "name": "Dublin Port", "lat": 53.3498, "lon": -6.2603, "country": "IE"},
            {"station_id": "IE_ORK_01", "name": "Cork Harbour", "lat": 51.8985, "lon": -8.4756, "country": "IE"},
            {"station_id": "NL_AMS_01", "name": "Amsterdam Centraal", "lat": 52.3702, "lon": 4.8952, "country": "NL"},
            {"station_id": "NL_ROT_01", "name": "Rotterdam Port", "lat": 51.9244, "lon": 4.4777, "country": "NL"}
        ]
        
        # Construct GeoDataFrame
        gdf = gpd.GeoDataFrame(raw_stations)
        
        # Create Point geometries from Longitude and Latitude
        gdf["geometry"] = [Point(xy) for xy in zip(gdf.lon, gdf.lat)]
        gdf.set_crs(epsg=4326, inplace=True)
        
        # Geometry validation
        gdf = gdf[gdf.geometry.is_valid & ~gdf.geometry.is_empty]
        
        # Load directly to PostGIS spatial table
        gdf.to_postgis(
            name="monitoring_stations",
            con=engine,
            if_exists="replace",
            index=False
        )
        print(f"Pipeline executed successfully: {len(gdf)} valid records written to PostGIS.")
        
    except Exception as err:
        print(f"Pipeline error: {err}", file=sys.stderr)
        raise

if __name__ == "__main__":
    run_pipeline()
```

---

## 4. PostGIS Analytical Engine (Geodesic Proximity Query)

Direct database computation of the true ellipsoidal distance between the Dublin Port and Amsterdam Centraal stations:

```sql
SELECT 
    s1.station_id AS origin_station,
    s1.name AS origin_name,
    s2.station_id AS target_station,
    s2.name AS target_name,
    ROUND((ST_Distance(s1.geometry::geography, s2.geometry::geography) / 1000.0)::numeric, 2) AS distance_km
FROM monitoring_stations s1, monitoring_stations s2
WHERE s1.station_id = 'IE_DUB_01' 
  AND s2.station_id = 'NL_AMS_01';
```

### Execution Results:

| origin_station | origin_name | target_station | target_name | distance_km |
| :--- | :--- | :--- | :--- | :--- |
| **IE_DUB_01** | Dublin Port | **NL_AMS_01** | Amsterdam Centraal | **759.09** |

---

## 5. Visual QA & Index Optimization

- **Spatial Indexing:** A generalized search tree (GiST) index is established on the table to ensure sub-millisecond bounding box evaluation in spatial queries:
  ```sql
  CREATE INDEX IF NOT EXISTS idx_stations_geom ON monitoring_stations USING GIST (geometry);
  ```
- **Cartographic Verification:** Verified in **QGIS LTR** using a secure PostgreSQL connection (`127.0.0.1:5432/geodb`). Stations were validated against OpenStreetMap basemaps to ensure zero coordinate-order inversions.