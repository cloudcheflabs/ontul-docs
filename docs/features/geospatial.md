# Geospatial Support

Ontul provides PostGIS-compatible geospatial SQL functions, enabling location-based queries and spatial analysis directly within SQL.

## Implementation

Geospatial functions are powered by JTS (Java Topology Suite) and registered as Calcite UDFs. The default SRID is 4326 (WGS84).

## Spatial Functions

### Construction

| Function | Description |
|----------|-------------|
| `ST_GeomFromText(wkt)` | Create geometry from WKT |
| `ST_GeomFromText(wkt, srid)` | Create geometry from WKT with SRID |
| `ST_Point(x, y)` | Create a point |
| `ST_MakePoint(x, y)` | Create a point (alias) |
| `ST_GeomFromGeoJSON(json)` | Create geometry from GeoJSON |

### Output

| Function | Description |
|----------|-------------|
| `ST_AsText(geom)` | Export to WKT |
| `ST_AsGeoJSON(geom)` | Export to GeoJSON |

### Spatial Relationships

| Function | Description |
|----------|-------------|
| `ST_Distance(g1, g2)` | Euclidean distance between geometries |
| `ST_Contains(g1, g2)` | Check if g1 contains g2 |
| `ST_Within(g1, g2)` | Check if g1 is within g2 |
| `ST_Intersects(g1, g2)` | Check if geometries intersect |
| `ST_Overlaps(g1, g2)` | Check if geometries overlap |
| `ST_Touches(g1, g2)` | Check if geometries touch |

### Processing

| Function | Description |
|----------|-------------|
| `ST_Area(geom)` | Polygon area |
| `ST_Buffer(geom, dist)` | Create buffer zone |
| `ST_Centroid(geom)` | Compute centroid |
| `ST_Union(g1, g2)` | Union of geometries |
| `ST_Intersection(g1, g2)` | Intersection of geometries |
| `ST_Envelope(geom)` | Bounding box |
| `ST_Length(geom)` | LineString length |

### Accessors

| Function | Description |
|----------|-------------|
| `ST_X(geom)` / `ST_Y(geom)` | Coordinate extraction |
| `ST_SRID(geom)` | Get SRID |
| `ST_SetSRID(geom, srid)` | Set SRID |
| `ST_GeometryType(geom)` | Geometry type name |
| `ST_NumPoints(geom)` | Vertex count |
| `ST_IsValid(geom)` | Validity check |

## Example

```sql
SELECT name, ST_Distance(
  ST_Point(126.9780, 37.5665),
  ST_GeomFromText(location)
) AS distance
FROM iceberg_catalog.geo.stores
WHERE ST_Contains(
  ST_Buffer(ST_Point(126.9780, 37.5665), 0.01),
  ST_GeomFromText(location)
)
ORDER BY distance;
```
