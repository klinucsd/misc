---
name: national-parks
description: Use this skill for requests to retrieve National Park Service (NPS) unit boundaries in the USA as a GeoDataFrame in GeoPandas. This skill can retrieve parks by name, unit code, state, region, or unit type.
---

# National Parks Skill

## Description

This skill retrieves geometries and attributes of National Park Service (NPS) units by accessing the ArcGIS Feature Service at the following URL:

```
https://services1.arcgis.com/fBc8EJBxQRMcHlei/ArcGIS/rest/services/NPS_Land_Resources_Division_Boundary_and_Tract_Data_Service/FeatureServer/2
```

The service returns the following columns:

- **OBJECTID**: System-assigned unique identifier for each park feature in the service
- **UNIT_CODE**: Official NPS unit code (e.g., "YELL" for Yellowstone, "GRCA" for Grand Canyon)
- **UNIT_NAME**: Official name of the NPS unit
- **PARKNAME**: Park name (may differ slightly from UNIT_NAME)
- **STATE**: U.S. state where the park is located 
- **REGION**: NPS administrative region code (two-letter abbreviation)
- **UNIT_TYPE**: Type of NPS unit (e.g., "National Park", "National Monument", "National Historic Site")
- **GNIS_ID**: Geographic Names Information System identifier
- **GIS_Notes**: Additional notes about the geographic data
- **DATE_EDIT**: Last edit date for the feature
- **CREATED_BY**: User who created the feature
- **METADATA**: Link or reference to metadata
- **CreationDate**: Date the feature was created in the database
- **Creator**: User who created the record
- **EditDate**: Date of last edit
- **Editor**: User who last edited the record
- **GlobalID**: Globally unique identifier used to track the feature across databases and services
- **AreaID**: Area identifier
- **Shape__Area**: System-calculated area of the park geometry in the feature service's coordinate system
- **Shape__Length**: System-calculated perimeter length of the park geometry in the feature service's coordinate system

## When to Use

Use this skill to:

- Find a national park by name or unit code
- Find objects within a national park boundary
- Find national parks in specific states or regions
- Query national parks by their attributes (unit type, region, etc.)
- Analyze park boundaries and spatial relationships

## How to Use

### Step 1: Import Required Libraries

```python
import geopandas as gpd
import requests
```

### Step 2: Define the Query Function

Use the `get_features` function to query the feature service. The **bbox parameter** is crucial for spatial filtering - it defines a bounding box to limit results to a specific geographic area.

```python
def get_features(service_url, where, bbox=None):
    """
    Query the ArcGIS Feature Service for national parks.
    
    Parameters:
    -----------
    service_url : str
        The URL of the ArcGIS Feature Service
    where : str
        SQL-like WHERE clause to filter features
    bbox : list or tuple, optional
        Bounding box as [minx, miny, maxx, maxy] in WGS84 (EPSG:4326)
        Default is the continental USA bounds
    
    Returns:
    --------
    geopandas.GeoDataFrame
        GeoDataFrame containing the queried national park features
    """
    
    # Default to continental USA bounds if no bbox provided
    if bbox is None:
        bbox = [-125.0, 24.396308, -66.93457, 49.384358]
    
    minx, miny, maxx, maxy = bbox
    
    params = {
        "where": where,
        "geometry": f"{minx},{miny},{maxx},{maxy}",
        "geometryType": "esriGeometryEnvelope",
        "spatialRel": "esriSpatialRelIntersects",
        "outFields": "*",
        "returnGeometry": "true",
        "f": "geojson",
        "outSR": "4326",  # Ensure output is in WGS84
        "resultOffset": 0,
        "resultRecordCount": 1000  # Increase if expecting more results
    }
    
    response = requests.get(service_url + "/query", params=params)
    data = response.json()
    
    if data.get('features'):
        return gpd.GeoDataFrame.from_features(data['features'], crs="EPSG:4326")
    else:
        return gpd.GeoDataFrame(columns=['geometry'], crs="EPSG:4326")
```

### Step 3: Query Examples

#### Example 1: Find a National Park by Name

```python
url = "https://services1.arcgis.com/fBc8EJBxQRMcHlei/ArcGIS/rest/services/NPS_Land_Resources_Division_Boundary_and_Tract_Data_Service/FeatureServer/2"

# Query for "Yellowstone National Park"
where = "LOWER(UNIT_NAME) LIKE '%yellowstone%'"
parks_gdf = get_features(url, where)
```

#### Example 2: Find All National Parks in California

This example demonstrates how to find parks in California. The `STATE` column contains two-letter state abbreviations.

```python
# Query for parks in California
where = "STATE = '<state>'"
ca_parks = get_features(url, where)
```

#### Example 3: Find Parks by Unit Code

```python
# Find Yosemite by its unit code
where = "UNIT_CODE = 'YOSE'"
yosemite = get_features(url, where)
```

#### Example 4: Find All National Parks (excluding monuments, historic sites, etc.)

```python
# Find only "National Park" unit types
where = "UNIT_TYPE = 'National Park'"
national_parks = get_features(url, where)
```

#### Example 5: Find Parks by Partial Name Match

```python
# Find all parks with "Canyon" in the name
where = "UNIT_NAME LIKE '%Canyon%'"
canyon_parks = get_features(url, where)
```

## Notes

Fetching all parks in a state may take about one minute depending on the number of units.

### Understanding the bbox Parameter

The **bbox (bounding box) parameter** is essential for efficient spatial queries:

- **Format**: `[minx, miny, maxx, maxy]` in WGS84 coordinates (longitude, latitude)
- **Purpose**: Limits the geographic extent of the query, reducing response time and data volume
- **Default**: If not provided, defaults to continental USA bounds
- **Use case**: Always provide a bbox when querying for features in a specific region (like a state or area)

**Why bbox matters**: Without a bbox, the service may return all features matching the WHERE clause across the entire USA, which is inefficient. The bbox acts as a spatial pre-filter, dramatically improving query performance.

### Using the UNIT_TYPE Column

Filter by different types of NPS units:

- **Common unit types**:
  - `"National Park"`
  - `"National Monument"`
  - `"National Historic Site"`
  - `"National Memorial"`
  - `"National Preserve"`
- **Query pattern**: Use `UNIT_TYPE = 'type'` for exact matches
  - Example: `"UNIT_TYPE = 'National Park'"` for only national parks
  - Example: `"UNIT_TYPE LIKE '%Historic%'"` for all historic sites

### WHERE Clause Tips

- Use `LOWER(field)` for case-insensitive string matching on text fields
- **For the STATE column**: Use exact match `STATE = 'XX'` where XX is the state abbreviation
  - Example: `"STATE = 'CA'"` for California
  - For multiple states: `"STATE = 'CA' OR STATE = 'OR'"` finds parks in California or Oregon
- **For the UNIT_CODE column**: Use exact match `UNIT_CODE = 'XXXX'`
  - Example: `"UNIT_CODE = 'GRCA'"` for Grand Canyon
- Use `LIKE` with `%` wildcards for partial matches
  - Example: `"UNIT_NAME LIKE '%Yellowstone%'"` finds Yellowstone-related units
- Use `1=1` to match all features when relying solely on spatial filtering (bbox)
- Combine multiple conditions with `AND` / `OR`
- Always enclose string values in single quotes
