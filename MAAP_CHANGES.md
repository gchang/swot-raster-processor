# Changes from upstream

Upstream: https://github.com/SWOTAlgorithms/Raster-Processor, tag `release_v1.5` at commit 4d9aed8bba62fddb2585a6c04b3f7efbdf83ddb4 (2026-06-25; also the head of `develop` and `delivery`), by Alexander Corben (JPL) and Shuai Zhang (UNC), maintained by the SWOT Algorithm Definition Team. Copyright California Institute of Technology, Apache License 2.0.

This copy starts from upstream's file tree at 4d9aed8 without its git history. It is unchanged except for the items below. The LICENSE, the processing algorithms and the version string (`VERSION_ID = 'V1.5'` in `src/SWOTRaster/products.py`, written to the product's `pge_version`) are untouched. Nothing was left out: upstream has no `.github` folder.

## 2026-10-25: height-constrained geolocation works with the public toolbox

`GeolocRaster.taylor_improved_geoloc` (`src/SWOTRaster/geoloc_raster.py`) called the hydrology toolbox's `pointcloud_height_geoloc_vect(..., swath=this_side)`. No published branch of https://github.com/CNES/swot-hydrology-toolbox accepts a `swath` argument, so since the per-side split was added (release 1.3) every run with the `smoothed_slant_plane` or `lowres_raster` geolocation stopped about 20 seconds in with `TypeError: ... unexpected keyword argument 'swath'`.

The argument is removed, the same change as upstream commit f5589e70 on its `hc_geoloc_swath_side_tmp_fix` branch, which never reached `develop`. The public function takes its starting angle from each pixel's own position, so it solves on the correct side without being told. Nothing else used the argument: the loop still processes the left and right swaths separately and merges them the same way. With the public toolbox this turns a crash into a product; the values match the official product (see Check).

## 2026-10-25: running without a PIXCVec file

`-pv` is optional and the processing handles a missing PIXCVec (the pixel cloud's own geolocation is used and the ice flags stay empty), but `swot_pixc2raster.py` then crashed after all the processing, before writing anything, because it took `os.path.basename(None)` for the `xref_l2_hr_pixcvec_files` attribute. The attribute is now set only when a PIXCVec file is given; otherwise it keeps the product's default `'None'`, like the other inputs that were not supplied. Runs with `-pv` are unchanged.

## 2026-10-25: README requirements

The README now lists what the code needs: Python 3.11 or newer (the product code uses `datetime.UTC`, while RiverObs' `environment.yml` pins Python 3.9, which fails when writing the product), rasterio (imported by `raster_proc.py` but not in RiverObs' list) and the other packages on the code path, the RiverObs and toolbox commits it was tested with (neither has tags), and the `PYTHONPATH` line.

## Notes for running it

- No configuration files ship with the code. The algorithmic and runtime parameters in the docstring of `src/bin/swot_pixc2raster.py` work as is; copy them into two RDF files.
- Input: one SWOT `L2_HR_PIXC` tile (0.3 to 1 GB) and, optionally, its `L2_HR_PIXCVec` tile (2 to 80 MB) from PO.DAAC, which needs an Earthdata login. The output covers the tile's bounding box, not a full raster scene.
- One tile takes about 8 minutes on 4 cores (`-mp 4`) with about 3.3 GB for the largest process; the worker processes add to that, so allow 6 to 8 GB.
- NumPy 2 prints a `DeprecationWarning` for `np.row_stack` in `raster_crs.py` many thousands of times (about 85 MB of log for one tile). `PYTHONWARNINGS=ignore` silences it.

## Check

On 2026-10-25, in a fresh conda env made with the README's command (Python 3.11.17, numpy 2.4.6, scipy 1.17.1, netCDF4 1.7.4, GDAL 3.12.3, shapely 2.2.0, pyproj 3.7.2, bottleneck 1.6.0, rasterio 1.4.4) and RiverObs `dc0c7fde` and swot-hydrology-toolbox `9c72c352` cloned as the README says, on tile cycle 57, pass 509, tile 208L (Mississippi River near Baton Rouge, 2026-10-19):

```sh
python src/bin/swot_pixc2raster.py \
  SWOT_L2_HR_PIXC_057_509_208L_20261019T194751_20261019T194802_PID0_01.nc \
  raster_alg_config.rdf raster_runtime_config.rdf raster_208L.nc \
  -pv SWOT_L2_HR_PIXCVec_057_509_208L_20261019T194751_20261019T194802_PID0_01.nc -id internal -mp 4
```

It finished in 7 min 48 s on 4 cores, 3.2 GB peak for the largest process. Against the official PO.DAAC product `SWOT_L2_HR_Raster_100m_UTM15R_N_x_x_x_057_509_104F_20261019T194741_20261019T194802_PID0_01.nc`, matched cell by cell on UTM coordinates: 97.4% of the 109,633 water surface elevation cells are identical, and 99.3% of the 84,770 cells more than 5 km from the tile edge (at most 1.6 cm apart); water area and sigma0 are identical in every cell more than 5 km from the edge. The differences sit along the tile edges, where the official scene also uses the neighboring tiles. The output is bit-identical in all 42 variables to a run of the unchanged release with only the `swath` argument removed. Without `-pv`, the run now writes its product. That product is identical except that the two ice flags are empty and `xref_l2_hr_pixcvec_files` is `'None'`. The product's own validator (`RasterUTM.test_product()`) passes on both.
