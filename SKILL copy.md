---
name: nhd-rivers
description: ALWAYS use this skill to get river or stream geometry in the USA. Never extract rivers from DEM/elevation data — that approach is inaccurate. Use this skill for any request involving a river: "find the Carson River", "show the river channel", "get the river centerline", "overlay the river on the map", "sample elevation along the river", "find the main channel". Returns official vector geometries from the National Hydrography Dataset via pynhd.
---

# NHD Rivers Skill

## CRITICAL — Do NOT extract rivers from DEM data

Never use DEM elevation values, valley detection, local minima, or any
raster-analysis method to find a river channel. Always use the functions
defined below. This applies even when a DEM is already available.

## Required Libraries

```python
import subprocess
subprocess.run(["pip", "install", "-q", "pynhd", "pygeoutils", "rioxarray"], check=True)

import pynhd
import pygeoutils
import geopandas as gpd
import numpy as np
import rasterio
import shapely
from shapely import ops
```

---

## Usage

This skill defines four functions. Copy all four function definitions into your
notebook first, then call them as shown in the examples. Do not rewrite the
function bodies — only change the arguments you pass in.

---

## Function Definitions — copy all four verbatim

```python
def fetch_flowlines(dem_bbox, margin=0.5):
    """
    Fetch NHD flowlines for a river using a bbox expanded from the DEM extent.

    Parameters
    ----------
    dem_bbox : tuple  (west, south, east, north) in EPSG:4326
    margin   : float  degrees to expand on each side (default 0.5)
                      Never pass the DEM bbox directly — it is too small.

    Returns
    -------
    flw : GeoDataFrame  all NHD flowlines in the expanded bbox, EPSG:4326
    """
    river_bbox = (
        dem_bbox[0] - margin,
        dem_bbox[1] - margin,
        dem_bbox[2] + margin,
        dem_bbox[3] + margin,
    )
    wd = pynhd.WaterData("nhdflowline_network")
    flw = wd.bybox(river_bbox)
    print(f"fetch_flowlines: {len(flw)} segments fetched for bbox {river_bbox}")
    return flw


def get_main_channel(flw, river_name):
    """
    Filter flowlines to the main channel and merge into a single LineString
    sorted in correct flow order (headwaters → outlet).

    Parameters
    ----------
    flw        : GeoDataFrame  output of fetch_flowlines()
    river_name : str  full lowercase official name, e.g. 'carson river'
                      Partial names match forks and tributaries — use full name.

    Returns
    -------
    river_line   : LineString   single merged line in flow order, EPSG:4326
    main_channel : GeoDataFrame filtered segments with hydroseq attribute intact
    """
    # Primary: exact name match
    main_channel = flw[flw['gnis_name'].str.lower() == river_name].copy()

    # Fallback: partial match filtered to highest stream order
    if len(main_channel) == 0:
        candidates = flw[flw['gnis_name'].str.lower().str.contains(river_name, na=False)].copy()
        if len(candidates) == 0:
            raise ValueError(
                f"No NHD flowlines found matching '{river_name}'. "
                "Check spelling or increase margin in fetch_flowlines()."
            )
        max_order = candidates['streamorde'].max()
        main_channel = (
            candidates[candidates['streamorde'] == max_order]
            .sort_values('levelpathi')
            .copy()
        )

    assert len(main_channel) > 0, f"main_channel is empty for '{river_name}'"

    # Sort by hydroseq BEFORE merging — arbitrary order causes elevation spikes
    main_channel = main_channel.sort_values('hydroseq', ascending=True)
    main_channel_lines = main_channel.explode(index_parts=False).reset_index(drop=True)
    river_line = ops.linemerge(main_channel_lines.geometry.tolist())

    if river_line.geom_type != 'LineString':
        raise ValueError(
            f"linemerge returned {river_line.geom_type} — segments not fully connected. "
            "Increase margin in fetch_flowlines() to capture all connecting segments."
        )

    print(f"get_main_channel: {len(main_channel)} segments → "
          f"{river_line.geom_type}, ~{river_line.length * 111:.0f} km")
    return river_line, main_channel


def sample_elevation(river_line, main_channel, dem):
    """
    Sample elevation along the river centerline and return river_elev in UTM.

    Parameters
    ----------
    river_line   : LineString   output of get_main_channel(), in EPSG:4326
    main_channel : GeoDataFrame output of get_main_channel()
    dem          : xarray.DataArray  from py3dep.get_dem(), in UTM CRS

    Returns
    -------
    river_elev : ndarray  shape (N, 3) — [utm_x, utm_y, elevation_m]
                 CRS matches dem.rio.crs. Required input for compute_rem().
    distances  : ndarray  shape (N,) — along-channel distances in metres
    """
    # Resample to 10 m spacing
    # river_line.length is in degrees — multiply by 111_000 to get metres
    npts = int(np.ceil(river_line.length * 111_000 / 10))
    river_line_smooth = pygeoutils.smooth_linestring(river_line, 0.1, npts)

    # Sample elevation from USGS seamless 10 m DEM VRT
    url = "https://prd-tnm.s3.amazonaws.com/StagedProducts/Elevation/13/TIFF/USGS_Seamless_DEM_13.vrt"
    with rasterio.open(url) as src:
        xy_raster = shapely.get_coordinates(
            pygeoutils.geo_transform(river_line_smooth, main_channel.crs, src.crs)
        )
        # src.sample yields one (1,) array per point — extract scalar
        z = np.array([val[0] for val in src.sample(xy_raster)])

    # Build river_elev in UTM (same CRS as dem) — NOT in EPSG:4326 degrees
    river_line_utm = pygeoutils.geo_transform(river_line_smooth, main_channel.crs, dem.rio.crs)
    xy_utm = shapely.get_coordinates(river_line_utm)

    assert len(xy_utm) == len(z), (
        f"Coordinate/elevation mismatch: {len(xy_utm)} coords vs {len(z)} elevations"
    )

    river_elev = np.c_[xy_utm, z]

    # Along-channel distances in metres, starting at 0
    pts = shapely.points(river_line_smooth.coords)
    distances = shapely.line_locate_point(river_line_smooth, pts)
    distances = distances - distances[0]   # ensure starts at 0

    print(f"sample_elevation: {len(river_elev)} points, "
          f"elevation {z.min():.0f}–{z.max():.0f} m, "
          f"length {distances[-1]/1000:.1f} km")
    return river_elev, distances


def plot_river_on_dem(dem, main_channel, river_name, output_path):
    """
    Overlay the main channel on the DEM and save the figure.

    Parameters
    ----------
    dem          : xarray.DataArray  from py3dep.get_dem(), in UTM CRS
    main_channel : GeoDataFrame  output of get_main_channel()
    river_name   : str  used for the plot title
    output_path  : str  directory where river_channel.png is saved
    """
    import matplotlib.pyplot as plt

    fig, ax = plt.subplots(figsize=(10, 6), dpi=100)
    dem.plot(ax=ax, robust=True, cmap='terrain')
    # Reproject to DEM CRS — never plot vector data without this
    main_channel.to_crs(dem.rio.crs).plot(
        ax=ax, color='red', linewidth=1.5, label=river_name.title()
    )
    ax.set_title(f"{river_name.title()} — main channel")
    ax.legend()
    plt.tight_layout()
    plt.savefig(f"{output_path}/river_channel.png", dpi=100, bbox_inches='tight')
    plt.close()
    print(f"plot_river_on_dem: saved to {output_path}/river_channel.png")
```

---

## Example 1 — Fetch main channel and overlay on DEM

```python
dem_bbox = (-119.59, 39.24, -119.47, 39.30)   # replace with actual DEM bbox

flw                      = fetch_flowlines(dem_bbox)
river_line, main_channel = get_main_channel(flw, 'carson river')
plot_river_on_dem(dem, main_channel, 'carson river', output_path)
```

---

## Example 2 — Sample elevation profile and plot it

```python
river_elev, distances = sample_elevation(river_line, main_channel, dem)

import matplotlib.pyplot as plt

fig, ax = plt.subplots(figsize=(10, 3), dpi=100)
ax.plot(distances / 1000, river_elev[:, 2], linewidth=1.2, color='steelblue')
ax.set_xlabel('Distance along river (km)')
ax.set_ylabel('Elevation (m)')
ax.set_title('Carson River — Longitudinal Elevation Profile')
z_min, z_max = np.nanmin(river_elev[:, 2]), np.nanmax(river_elev[:, 2])
margin = (z_max - z_min) * 0.05
ax.set_ylim(z_min - margin, z_max + margin)
plt.tight_layout()
plt.savefig(f"{output_path}/elevation_profile.png", dpi=100, bbox_inches='tight')
plt.show()
```

---

## Example 3 — Full pipeline from bbox to river_elev (feeds into REM)

```python
dem_bbox = (-119.59, 39.24, -119.47, 39.30)   # replace with actual DEM bbox

flw                      = fetch_flowlines(dem_bbox)
river_line, main_channel = get_main_channel(flw, 'carson river')
river_elev, distances    = sample_elevation(river_line, main_channel, dem)

# river_elev is now ready to pass directly into compute_rem() from py3dep-dem skill
```

---

## Notes

- `hydroseq`: lower = closer to outlet, higher = headwaters. `get_main_channel()` always sorts ascending — **do not re-sort or re-merge outside the function**.
- `river_elev` shape is `(N, 3)` — columns `[utm_x, utm_y, elevation_m]` in UTM (same CRS as `dem`). This is the required input to `compute_rem()` in the py3dep-dem skill.
- `distances` always starts at 0 — the function resets it with `distances - distances[0]`.
- `fetch_flowlines()` expands the DEM bbox by 0.5° by default. Increase `margin` if `get_main_channel()` raises a `MultiLineString` error.
