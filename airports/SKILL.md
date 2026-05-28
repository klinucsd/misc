---
name: airports
description: Retrieve airports in the USA. Query by location, name, FAA code, or passenger volume. Automatically queries all size categories and combines results.
---

# Airports Skill

## ⚠️ IMPORTANT: Multi-Layer Service

**CRITICAL**: This dataset is split into **3 separate layers by annual passenger volume**. When searching for airports without specific conditions (e.g., "all airports in Colorado" or "find Denver International"), you MUST query **all three layers** and combine the results.

## Description

This skill retrieves locations and attributes of airports in the United States by accessing three ArcGIS Feature Service layers organized by annual passenger volume:

**Layer 1 - Major Airports (1,000,000+ passengers annually):**
```
https://services.arcgis.com/P3ePLMYs2RVChkJx/ArcGIS/rest/services/USA_Airports_by_scale/FeatureServer/1
```

**Layer 2 - Medium Airports (100,000 - 999,999 passengers annually):**
```
https://services.arcgis.com/P3ePLMYs2RVChkJx/ArcGIS/rest/services/USA_Airports_by_scale/FeatureServer/2
```

**Layer 3 - Small Airports (<100,000 passengers annually):**
```
https://services.arcgis.com/P3ePLMYs2RVChkJx/ArcGIS/rest/services/USA_Airports_by_scale/FeatureServer/3
```

**Service Details:**
- Maximum record count per layer: 1000
- Spatial reference: WGS84 (EPSG:4326) - no transformation needed
- Coverage: USA airports with passenger service

All layers share the same fields:

- **OBJECTID**: System-assigned unique identifier
- **FAA_ID**: FAA airport identifier code (e.g., "DEN", "LAX", "ORD")
- **NAME**: Airport name **⚠️ IMPORTANT**: Names include FAA code in format "Airport Name (FAA: XXX)"
- **FACILITY**: Facility type (e.g., "AIRPORT", "SEAPLANE BASE")
- **CITY**: City location
- **COUNTY**: County location
- **STATE**: State (2-letter abbreviation)
- **OWNER**: Ownership type (e.g., "PUBLIC", "PRIVATE")
- **ELEV_FEET**: Elevation in feet above sea level
- **INTL**: International airport (Y/N)
- **TOWER**: Control tower present (Y/N)
- **ARRIVALS**: Annual arrivals count
- **DEPARTURES**: Annual departures count
- **ENPLANEMEN**: Annual enplanements (passengers boarding)
- **PASSENGERS**: Annual total passengers

## ⚠️ CRITICAL: Airport Name Format

**Airport names in this dataset include the FAA code**. For example:
- ✅ Actual value: `"Denver International Airport (FAA: DEN)"`
- ❌ Won't work: `"Denver International Airport"`

**Always use partial matching with LIKE when searching by name:**
```python
# CORRECT - use LIKE with % wildcards
where = "NAME LIKE '%Denver%'"

# INCORRECT - exact match will fail
where = "NAME = 'Denver International Airport'"
```

## When to Use

Use this skill to:

- Find all airports in a city, county, state, or region
- Find airports by name or FAA code
- Find airports by passenger volume
- Find international airports
- Find airports with control towers
- Analyze airport distribution and traffic patterns

## How to Use

### Step 1: Import Required Libraries

```python
import geopandas as gpd
import requests
import pandas as pd
```

### Step 2: Define Query Functions

**⚠️ CRITICAL**: Always query all three layers to get complete results.

```python
def get_features_single_layer(service_url, where, bbox=None):
    """
    Query a single airport layer.
    
    Parameters:
    -----------
    service_url : str
        The URL of the ArcGIS Feature Service layer
    where : str
        SQL-like WHERE clause to filter features
    bbox : list or tuple, optional
        Bounding box as [minx, miny, maxx, maxy] in WGS84 (EPSG:4326)
    
    Returns:
    --------
    geopandas.GeoDataFrame
        GeoDataFrame containing queried airport features in EPSG:4326
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
        "outSR": "4326"
    }
    
    response = requests.get(service_url + "/query", params=params)
    data = response.json()
    
    if data.get('features'):
        return gpd.GeoDataFrame.from_features(data['features'], crs="EPSG:4326")
    else:
        return gpd.GeoDataFrame(columns=['geometry'], crs="EPSG:4326")

def get_all_airports(where, bbox=None):
    """
    Query ALL three airport layers and combine results.
    
    USE THIS FUNCTION for most queries to ensure complete results.
    
    Parameters:
    -----------
    where : str
        SQL-like WHERE clause to filter features
    bbox : list or tuple, optional
        Bounding box as [minx, miny, maxx, maxy] in WGS84 (EPSG:4326)
    
    Returns:
    --------
    geopandas.GeoDataFrame
        Combined GeoDataFrame from all three layers
    """
    base_url = "https://services.arcgis.com/P3ePLMYs2RVChkJx/ArcGIS/rest/services/USA_Airports_by_scale/FeatureServer"
    
    # Query all three layers
    layer1 = get_features_single_layer(f"{base_url}/1", where, bbox)  # Major (1M+)
    layer2 = get_features_single_layer(f"{base_url}/2", where, bbox)  # Medium (100K-999K)
    layer3 = get_features_single_layer(f"{base_url}/3", where, bbox)  # Small (<100K)
    
    # Combine all results
    all_airports = pd.concat([layer1, layer2, layer3], ignore_index=True)
    
    # Convert back to GeoDataFrame if we have results
    if len(all_airports) > 0:
        return gpd.GeoDataFrame(all_airports, crs="EPSG:4326")
    else:
        return gpd.GeoDataFrame(columns=layer1.columns, crs="EPSG:4326")
```

### Step 3: Query Examples

#### Example 1: Find ALL Airports in a State (Most Common Use Case)

```python
# Find all airports in Colorado
# MUST query all three layers to get complete results
colorado_bbox = [-109.06, 36.99, -102.04, 41.00]
where = "STATE = 'CO'"

co_airports = get_all_airports(where, bbox=colorado_bbox)

print(f"Found {len(co_airports)} airports in Colorado")
print(f"\nBreakdown by size:")
major = len(co_airports[co_airports['PASSENGERS'] >= 1000000])
medium = len(co_airports[(co_airports['PASSENGERS'] >= 100000) & (co_airports['PASSENGERS'] < 1000000)])
small = len(co_airports[co_airports['PASSENGERS'] < 100000])
print(f"  Major (1M+ passengers): {major}")
print(f"  Medium (100K-999K): {medium}")
print(f"  Small (<100K): {small}")
```

#### Example 2: Find Airport by Name

```python
# Find Denver International Airport
# IMPORTANT: Use LIKE with % because name includes "(FAA: DEN)" at the end
# Actual name: "Denver International Airport (FAA: DEN)"
where = "NAME LIKE '%Denver International%'"
denver_airport = get_all_airports(where)

if len(denver_airport) > 0:
    airport = denver_airport.iloc[0]
    print(f"Airport: {airport['NAME']}")  # Will show: "Denver International Airport (FAA: DEN)"
    print(f"FAA Code: {airport['FAA_ID']}")
    print(f"City: {airport['CITY']}, {airport['STATE']}")
    print(f"Annual Passengers: {airport['PASSENGERS']:,.0f}")
    print(f"International: {airport['INTL']}")

# Alternative: Search by keyword only
where = "NAME LIKE '%Denver%'"  # Will find any airport with "Denver" in the name
denver_airports = get_all_airports(where)
print(f"Found {len(denver_airports)} airports with 'Denver' in the name")
```
```

#### Example 3: Find Airport by FAA Code

```python
# Find LAX
where = "FAA_ID = 'LAX'"
lax = get_all_airports(where)

if len(lax) > 0:
    print(f"Found: {lax.iloc[0]['NAME']}")
    print(f"Passengers: {lax.iloc[0]['PASSENGERS']:,.0f}")
```

#### Example 4: Find International Airports

```python
# Find all international airports
where = "INTL = 'Y'"
intl_airports = get_all_airports(where)

print(f"Found {len(intl_airports)} international airports")
print(intl_airports[['NAME', 'CITY', 'STATE', 'PASSENGERS']].sort_values('PASSENGERS', ascending=False).head(10))
```

#### Example 5: Find Major Airports Only (1M+ passengers)

```python
# If you know you only want major airports, query layer 1 directly
base_url = "https://services.arcgis.com/P3ePLMYs2RVChkJx/ArcGIS/rest/services/USA_Airports_by_scale/FeatureServer"
where = "1=1"  # Get all from this layer

major_airports = get_features_single_layer(f"{base_url}/1", where)

print(f"Found {len(major_airports)} major airports (1M+ passengers)")
print(major_airports.nlargest(10, 'PASSENGERS')[['NAME', 'FAA_ID', 'CITY', 'PASSENGERS']])
```

#### Example 6: Find Airports in a City

```python
# Find all airports in Miami
where = "CITY = 'MIAMI' AND STATE = 'FL'"
miami_airports = get_all_airports(where)

print(f"Found {len(miami_airports)} airports in Miami")
print(miami_airports[['NAME', 'FAA_ID', 'FACILITY', 'PASSENGERS']])
```

#### Example 7: Find Airports with Control Towers

```python
# California airports with control towers
california_bbox = [-124.48, 32.53, -114.13, 42.01]
where = "TOWER = 'Y' AND STATE = 'CA'"

ca_tower_airports = get_all_airports(where, bbox=california_bbox)

print(f"Found {len(ca_tower_airports)} California airports with control towers")
```

#### Example 8: Find Busiest Airports in a Region

```python
# Find top 10 busiest airports in the Pacific Northwest
pnw_bbox = [-125.0, 42.0, -116.0, 49.0]
where = "STATE IN ('WA', 'OR', 'ID')"

pnw_airports = get_all_airports(where, bbox=pnw_bbox)

print(f"Top 10 busiest airports in Pacific Northwest:")
top10 = pnw_airports.nlargest(10, 'PASSENGERS')[['NAME', 'CITY', 'STATE', 'FAA_ID', 'PASSENGERS']]
for idx, row in top10.iterrows():
    print(f"  {row['NAME']} ({row['FAA_ID']}) - {row['CITY']}, {row['STATE']}: {row['PASSENGERS']:,.0f} passengers")
```

## Notes

This service is divided into three layers by passenger volume. For most queries, use the `get_all_airports()` function to search across all three layers.

### Understanding the Multi-Layer Structure

**Why three layers?**
- **Layer 1**: Major airports (60-80 airports nationwide)
- **Layer 2**: Medium airports (100-200 airports)
- **Layer 3**: Small airports (300-500 airports)

**When to query all layers vs. single layer:**
- **Query all layers** (use `get_all_airports()`):
  - Finding airports by name (don't know size)
  - Finding all airports in a region
  - Finding airports by FAA code
  - General searches
  
- **Query single layer**:
  - Specifically want only major airports
  - Analyzing busiest airports nationwide
  - Already know the airport size category

### Understanding the bbox Parameter

The **bbox (bounding box) parameter** limits the geographic extent:

- **Format**: `[minx, miny, maxx, maxy]` in WGS84 coordinates (longitude, latitude)
- **Purpose**: Limits spatial extent of the query
- **Use case**: Always provide a bbox when querying for airports in a specific region

**Common State/City Bounding Boxes**:
```python
# Colorado
colorado_bbox = [-109.06, 36.99, -102.04, 41.00]

# California
california_bbox = [-124.48, 32.53, -114.13, 42.01]

# New York City area
nyc_bbox = [-74.26, 40.49, -73.70, 40.92]

# Pacific Northwest (WA, OR, ID)
pnw_bbox = [-125.0, 42.0, -116.0, 49.0]

# Texas
texas_bbox = [-106.65, 25.84, -93.51, 36.50]
```

### Using the NAME Field (CRITICAL)

**⚠️ AIRPORT NAMES INCLUDE FAA CODE**: The NAME field contains the FAA code suffix in the format `"Airport Name (FAA: XXX)"`.

**Examples of actual NAME values:**
- `"Denver International Airport (FAA: DEN)"`
- `"Los Angeles International Airport (FAA: LAX)"`
- `"Chicago O'Hare International Airport (FAA: ORD)"`

**ALWAYS use LIKE with % wildcards when searching by name:**

```python
# ✅ CORRECT - partial match
where = "NAME LIKE '%Denver%'"
where = "NAME LIKE '%International%'"
where = "NAME LIKE '%Los Angeles%'"

# ❌ INCORRECT - exact match will fail
where = "NAME = 'Denver International Airport'"  # Won't find anything!

# ✅ BEST PRACTICE - Search by FAA code instead when you know it
where = "FAA_ID = 'DEN'"  # More reliable
```

**Search patterns:**
```python
# Find by city name
where = "NAME LIKE '%Seattle%'"

# Find international airports by name
where = "NAME LIKE '%International%'"

# Find regional airports
where = "NAME LIKE '%Regional%'"

# Combine with other criteria
where = "NAME LIKE '%Los Angeles%' AND INTL = 'Y'"
```

**TIP**: If you know the FAA code, use `FAA_ID` instead of `NAME` for more reliable queries.

### Using the STATE Field

The `STATE` field contains two-letter state abbreviations:

- **Format**: Uppercase two-letter codes (e.g., "CO", "CA", "NY")
- **Query pattern**: Use exact match
  - Example: `"STATE = 'CA'"`
  - Multiple states: `"STATE IN ('CA', 'OR', 'WA')"`

### Using the INTL and TOWER Fields

Both are Y/N fields:

**International airports**:
```python
where = "INTL = 'Y'"  # International airports only
```

**Airports with control towers**:
```python
where = "TOWER = 'Y'"  # Towered airports only
```

**Combine conditions**:
```python
where = "INTL = 'Y' AND TOWER = 'Y'"  # International airports with towers
```

### Using Passenger Volume Fields

- **PASSENGERS**: Total annual passengers (arrivals + departures)
- **ENPLANEMEN**: Passengers boarding (departures)
- **ARRIVALS**: Annual arrival count
- **DEPARTURES**: Annual departure count

**Query patterns**:
```python
# Very busy airports (>10 million passengers)
where = "PASSENGERS >= 10000000"

# Medium traffic
where = "PASSENGERS BETWEEN 500000 AND 1000000"

# Sort by busiest
airports.sort_values('PASSENGERS', ascending=False)
```

### WHERE Clause Tips

- **For ALL airports in a region**: Use `"1=1"` or `"STATE = 'XX'"`
- **By FAA code** (RECOMMENDED): `"FAA_ID = 'DEN'"` (always uppercase, 3-4 letters)
- **By name** (use LIKE, not =): 
  - ✅ `"NAME LIKE '%Denver%'"` (CORRECT)
  - ❌ `"NAME = 'Denver International Airport'"` (WILL FAIL - names include FAA code)
  - **Best practice**: Use `FAA_ID` if you know it, as it's more reliable
- **International only**: `"INTL = 'Y'"`
- **With control tower**: `"TOWER = 'Y'"`
- **By passenger volume**: Use numeric comparisons
  - Example: `"PASSENGERS >= 1000000"`
- **Multiple states**: `"STATE IN ('CA', 'OR', 'WA')"`
- Always enclose string values in single quotes
- Use uppercase for STATE codes
- **Remember**: NAME field includes "(FAA: XXX)" suffix, so always use LIKE with %

### Complete Example: Comprehensive Airport Analysis

```python
import geopandas as gpd
import requests
import pandas as pd
import matplotlib.pyplot as plt

# Function definitions (as above)
def get_features_single_layer(service_url, where, bbox=None):
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
        "outSR": "4326"
    }
    
    response = requests.get(service_url + "/query", params=params)
    data = response.json()
    
    if data.get('features'):
        return gpd.GeoDataFrame.from_features(data['features'], crs="EPSG:4326")
    else:
        return gpd.GeoDataFrame(columns=['geometry'], crs="EPSG:4326")

def get_all_airports(where, bbox=None):
    base_url = "https://services.arcgis.com/P3ePLMYs2RVChkJx/ArcGIS/rest/services/USA_Airports_by_scale/FeatureServer"
    
    layer1 = get_features_single_layer(f"{base_url}/1", where, bbox)
    layer2 = get_features_single_layer(f"{base_url}/2", where, bbox)
    layer3 = get_features_single_layer(f"{base_url}/3", where, bbox)
    
    all_airports = pd.concat([layer1, layer2, layer3], ignore_index=True)
    
    if len(all_airports) > 0:
        return gpd.GeoDataFrame(all_airports, crs="EPSG:4326")
    else:
        return gpd.GeoDataFrame(columns=layer1.columns, crs="EPSG:4326")

# Query all airports in California
california_bbox = [-124.48, 32.53, -114.13, 42.01]
where = "STATE = 'CA'"

ca_airports = get_all_airports(where, bbox=california_bbox)

print(f"AIRPORTS IN CALIFORNIA")
print("="*60)
print(f"\nTotal airports: {len(ca_airports)}")

# Breakdown by size
print(f"\nBreakdown by passenger volume:")
major = ca_airports[ca_airports['PASSENGERS'] >= 1000000]
medium = ca_airports[(ca_airports['PASSENGERS'] >= 100000) & (ca_airports['PASSENGERS'] < 1000000)]
small = ca_airports[ca_airports['PASSENGERS'] < 100000]

print(f"  Major (1M+ passengers): {len(major)}")
print(f"  Medium (100K-999K): {len(medium)}")
print(f"  Small (<100K): {len(small)}")

# International airports
intl = ca_airports[ca_airports['INTL'] == 'Y']
print(f"\nInternational airports: {len(intl)}")

# Airports with control towers
towered = ca_airports[ca_airports['TOWER'] == 'Y']
print(f"Airports with control towers: {len(towered)}")

# Total passenger traffic
total_passengers = ca_airports['PASSENGERS'].sum()
print(f"\nTotal annual passengers: {total_passengers:,.0f}")

# Top 10 busiest
print(f"\nTop 10 busiest airports:")
top10 = ca_airports.nlargest(10, 'PASSENGERS')[['NAME', 'FAA_ID', 'CITY', 'PASSENGERS']]
for idx, row in top10.iterrows():
    print(f"  {row['FAA_ID']} - {row['NAME']} ({row['CITY']}): {row['PASSENGERS']:,.0f}")

# Ownership breakdown
print(f"\nOwnership breakdown:")
ownership_counts = ca_airports['OWNER'].value_counts()
for owner, count in ownership_counts.items():
    print(f"  {owner}: {count}")

# Plot airports by size
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(16, 8))

# Map colored by passenger volume
ca_airports['size_category'] = pd.cut(ca_airports['PASSENGERS'], 
                                      bins=[0, 100000, 1000000, float('inf')],
                                      labels=['Small', 'Medium', 'Major'])

ca_airports.plot(ax=ax1, column='size_category', legend=True,
                markersize=50, alpha=0.7, categorical=True,
                cmap='YlOrRd',
                legend_kwds={'title': 'Airport Size'})
ax1.set_title('California Airports by Passenger Volume')
ax1.set_xlabel('Longitude')
ax1.set_ylabel('Latitude')

# Bar chart of top airports
top10_chart = ca_airports.nlargest(10, 'PASSENGERS')
ax2.barh(range(len(top10_chart)), top10_chart['PASSENGERS'])
ax2.set_yticks(range(len(top10_chart)))
ax2.set_yticklabels(top10_chart['FAA_ID'])
ax2.set_xlabel('Annual Passengers')
ax2.set_title('Top 10 Busiest California Airports')
ax2.invert_yaxis()
ax2.grid(axis='x', alpha=0.3)

plt.tight_layout()
plt.show()

print(f"\n{'='*60}")
print(f"Analysis complete!")
```
