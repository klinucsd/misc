---
name: py3dep-dem
description: "Use this skill for digital elevation model (DEM) requests: check available 3DEP resolutions for an area, download DEM rasters, compute terrain derivatives (slope, aspect, hillshade), or compute a Relative Elevation Model (REM) for a river floodplain visualization."
---

# py3dep DEM Skill

## Required Libraries

```python
import subprocess
subprocess.run(["pip", "install", "-q", "py3dep", "rioxarray", "xarray"], check=True)

import py3dep
import rioxarray
import xarray as xr
```

---

## When to Use

* When need to fetch DEM for a bbox, use the Example 1
* When need to check if DEM is availavle for a bbox, use the Example 2

---

## Usage

This skill defines two functions. Copy both function definitions into your notebook
first, then call them as shown in the examples. Do not rewrite the function bodies.

---

## Function Definitions — copy both verbatim

```python
def get_dem(bbox, res=10, output_path=None):
    """
    Download a DEM for a bounding box and optionally save it.

    Parameters
    ----------
    bbox        : tuple  (west, south, east, north) in EPSG:4326
    res         : int    resolution in metres — 1, 10 (default), or 30
    output_path : str    if provided, saves dem_10m.tif here

    Returns
    -------
    dem : xarray.DataArray  in UTM CRS (e.g. EPSG:32611). NOT in EPSG:4326.
          All x/y coordinates are in metres.

    CRITICAL: dem is in UTM, not EPSG:4326. Any vector data (rivers, polygons)
    plotted on top of dem MUST be reprojected with .to_crs(dem.rio.crs) first,
    or it will be invisible (coordinates in degrees vs metres).
    """
    import py3dep

    dem = py3dep.get_dem(bbox, res)

    if output_path:
        dem.rio.to_raster(f"{output_path}/dem_{res}m.tif")
        print(f"get_dem: saved to {output_path}/dem_{res}m.tif")

    print(f"get_dem: shape={dem.shape}, CRS={dem.rio.crs}, "
          f"elevation {float(dem.min()):.0f}–{float(dem.max()):.0f} m")
    return dem


def compute_rem(dem, river_elev, output_path):
    """
    Compute a Relative Elevation Model (REM) using IDW interpolation and
    visualize it with datashader (hillshade base + inferno REM overlay).

    Parameters
    ----------
    dem         : xarray.DataArray  from get_dem(), in UTM CRS
    river_elev  : ndarray  shape (N, 3) — [utm_x, utm_y, elevation_m]
                  output of sample_elevation() from nhd-rivers skill,
                  in the SAME UTM CRS as dem
    output_path : str  directory where rem.png is saved

    Returns
    -------
    rem : xarray.DataArray  REM values in metres, same grid as dem

    DO NOT substitute a different interpolation method, colormap, or
    visualization library. The output will look wrong — a generic elevation
    map instead of a floodplain visualization.
    """
    import numpy as np
    import xarray as xr
    from scipy.spatial import KDTree
    import opt_einsum as oe
    from pathlib import Path

    # xarray-spatial installs as 'xarray-spatial' but imports as 'xrspatial'
    import xrspatial as xs
    import datashader.transfer_functions as tf
    from datashader.colors import Greys9, inferno
    from datashader import utils as ds_utils

    import subprocess
    subprocess.run(
        ["pip", "install", "-q", "scipy", "opt_einsum", "xarray-spatial", "datashader"],
        check=True
    )

    assert river_elev.ndim == 2 and river_elev.shape[1] == 3, \
        "river_elev must be shape (N, 3): [utm_x, utm_y, elevation_m]"

    # k must not exceed the number of river points
    k = min(200, len(river_elev))

    # IDW trend surface — KDTree.query returns (distances, indices)
    grid_coords = np.dstack(np.meshgrid(dem.x, dem.y)).reshape(-1, 2)
    distances, idxs = KDTree(river_elev[:, :2]).query(grid_coords, k=k, workers=-1)

    w = np.reciprocal(np.power(distances, 2) + np.isclose(distances, 0))
    w_sum = np.sum(w, axis=1)
    w_norm = oe.contract(
        "ij,i->ij", w,
        np.reciprocal(w_sum + np.isclose(w_sum, 0)),
        optimize="optimal"
    )
    elevation = oe.contract("ij,ij->i", w_norm, river_elev[idxs, 2], optimize="optimal")
    elevation = elevation.reshape((dem.sizes["y"], dem.sizes["x"]))
    elevation = xr.DataArray(elevation, dims=("y", "x"), coords={"x": dem.x, "y": dem.y})

    rem = (dem - elevation).clip(min=0)

    # Datashader visualization: greyscale DEM + hillshade + inferno REM
    illuminated = xs.hillshade(dem, angle_altitude=10, azimuth=90)
    tf.Image.border = 0
    img = tf.stack(
        tf.shade(dem,         cmap=Greys9,            how="linear"),
        tf.shade(illuminated, cmap=["black", "white"], how="linear", alpha=180),
        tf.shade(rem,         cmap=inferno[::-1],      span=[0, 7],  how="log", alpha=200),
    )
    ds_utils.export_image(img[::-1], Path(output_path, "rem").as_posix())
    print(f"compute_rem: saved to {Path(output_path, 'rem.png')}")
    return rem
```

---

## Example 1 — Download DEM and display it

```python
bbox = (-119.59, 39.24, -119.47, 39.30)   # replace with actual bbox

dem = get_dem(bbox, res=10, output_path=output_path)

import matplotlib.pyplot as plt
fig, ax = plt.subplots(figsize=(8, 6))
dem.plot(ax=ax, robust=True, cmap='terrain')
plt.tight_layout()
plt.savefig(f"{output_path}/dem.png", dpi=100, bbox_inches='tight')
plt.close()
```

---

## Example 2 — Check available DEM resolutions

```python
import py3dep
bbox = (-119.59, 39.24, -119.47, 39.30)
availability = py3dep.check_3dep_availability(bbox)
# {'1m': True, '10m': True, '30m': True, ...}
# 1 m (Lidar) = best detail; 10 m = good default; 30 m = large regions
```

---

## Example 3 — Overlay vector data on DEM

Always reproject vector data to `dem.rio.crs` before plotting.
Skipping this makes the layer invisible — no error, no warning, just missing data.

```python
# WRONG — river in EPSG:4326 degrees, DEM in UTM metres → invisible
river_gdf.plot(ax=ax, color='red')

# CORRECT — reproject first
river_gdf.to_crs(dem.rio.crs).plot(ax=ax, color='red', linewidth=1.5)
```

---

## Example 4 — Compute REM (requires river_elev from nhd-rivers skill)

```python
# river_elev comes from sample_elevation() in the nhd-rivers skill
rem = compute_rem(dem, river_elev, output_path)
```

---

## Example 5 — Full pipeline: DEM + river + REM

```python
output_path = "/home/jovyan/work/rem_output"   # set explicitly
bbox        = (-119.59, 39.24, -119.47, 39.30)

# DEM
dem = get_dem(bbox, res=10, output_path=output_path)

# River (nhd-rivers skill)
flw                      = fetch_flowlines(bbox)
river_line, main_channel = get_main_channel(flw, 'carson river')
river_elev, distances    = sample_elevation(river_line, main_channel, dem)

# REM
rem = compute_rem(dem, river_elev, output_path)
```

---

## Notes

- `get_dem()` returns UTM, not EPSG:4326. Always use `.to_crs(dem.rio.crs)` before plotting any vector layer on top.
- `compute_rem()` uses KDTree IDW + datashader. Do not substitute `scipy.ndimage`, `matplotlib`, `viridis`, or any other method — the result will be a generic elevation map, not a floodplain visualization.
- `xarray-spatial` installs as `xarray-spatial` but **imports as `xrspatial`** — not `xarray_spatial`.
- `output_path` must be defined as a string before calling either function.
