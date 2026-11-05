# Cloud-Native Geospatial Data Converter

A notebook pipeline that takes geospatial files in many formats, checks and repairs them, tags each one with a UUID, and converts it to the matching **cloud-native format**. It then builds a **STAC catalog** of the outputs.

| Input family | Examples | Output |
|---|---|---|
| Vector | GeoJSON, Shapefile, GeoPackage, GML, KML, FlatGeobuf, FileGDB, CSV with lon/lat, GeoParquet | **GeoParquet** (2.0 by default, or 1.1), Hilbert-sorted, zstd |
| Raster | GeoTIFF, JPEG2000, IMG, ASCII grid, VRT, gridded XYZ, Sentinel-1/2 `.SAFE`, ArcInfo grid | **COG** (Cloud-Optimized GeoTIFF) |
| Datacube | netCDF, GRIB/GRIB2, Zarr | **Zarr v3** (512 × 512 spatial chunks) |
| Point cloud | LAS/LAZ, E57, EPT, TileDB, ASCII X Y Z text | **COPC** (`.copc.laz`) |

Archives (`.zip`, `.7z`, `.tar*`, `.gz`, `.bz2`, `.xz`) are extracted and their contents processed. Each output still points back to the archive it came from.

---

## Repository contents

| File | Purpose |
|---|---|
| `notebooks/data_converter.ipynb` | The whole pipeline: discovery → check & fix → UUID → convert → manifest → validation → STAC |
| `environment_data-converter.yml` | Conda environment (PDAL, GDAL, cfgrib, zarr, geoparquet-io, pystac, …) |

The notebook does not import any local modules, so it only needs the environment and your input data.

---

## Setup

```bash
conda env create -f environment_data-converter.yml
conda activate data-converter
jupyter lab notebooks/data_converter.ipynb    # or open it in VS Code with the data-converter kernel
```

Notes:
- **Windows:** the first cell adds the env's `Library/bin` to `PATH` so cfgrib can find the ecCodes DLL even when the kernel was started without activating the env. It also removes `PROJ_LIB` if it points at a PostGIS install, because PostGIS's older `proj.db` breaks CRS lookups.
- `py7zr` is optional. Without it, `.7z` archives are reported as skipped.

---

## Usage

1. **Configure** (cell 2):

   ```python
   DATA_DIR = Path("../data").resolve()   # where inputs live (relative to notebooks/)
   OUT_DIR  = DATA_DIR / "cng"            # all outputs go here
   WORK_DIR = OUT_DIR / "_work"           # extracted archives + temporary VRTs

   ASSUME_CRS = None          # e.g. "EPSG:6668": stamped on inputs with no CRS
   UUID_COL = "uuid"          # per-feature UUID column for vectors
   GEOPARQUET_VERSION = "2.0" # "2.0" (native GEOMETRY type) or "1.1" (WKB + bbox column)
   ZARR_FORMAT = 3            # 2 for older zarr-python 2 era tools
   OVERWRITE = True           # False: keep existing outputs (status "exists")
   ```

2. **List your inputs** and run:

   ```python
   inputs = [DATA_DIR / "my_layer.geojson", DATA_DIR / "dem_tiles.zip", DATA_DIR / "survey.las"]
   results = run_pipeline(inputs)

   # or a whole folder, walked recursively:
   results = run_pipeline([DATA_DIR])
   ```

   `run_pipeline` returns a DataFrame with one row per output: status, kind, input, output, CRS, count, the fixes it applied, warnings, any error, and the time taken.

3. **Validate** the outputs (cell 15). This re-opens every `ok` output and checks that it really is a GeoParquet, COG, Zarr or COPC with the right UUID.

4. **Build the STAC catalog** (last cell) with `build_stac_catalog(results)`.

---

## Pipeline steps

### 1. Discovery

`discover()` walks each input and assigns every dataset one of the kinds `vector`, `raster`, `datacube`, `pointcloud` or `unsupported`:

- **Known extensions** are classified directly. `.gpkg` / `.mbtiles` can hold both vector and raster data and produce one item per kind.
- **Unknown extensions** (GML as `.xml`, GRIB as `.bin`, extension-less files) are checked for magic bytes first (`GRIB`, `CDF`) and then probed with GDAL (pyogrio / rasterio) and then PDAL.
- **Folder datasets** are treated as one dataset and not walked: `.SAFE`, `.zarr`, `.gdb`, ArcInfo grids/coverages, EPT (`ept.json`), TileDB arrays.
- **`.xyz` text** becomes a raster if GDAL reads it as a regular grid, or a point cloud if it is scattered X Y Z rows. Headerless rows get their column names from the column count.
- **Duplicates:** if one folder holds the same file name in several vector formats (e.g. N03 ships `.shp` + `.geojson` + `.xml`), only the best one is converted. The order of preference is: Parquet > GPKG > FGB > SHP > GeoJSON > GML > KML.
- **Sidecars** (`.shx`, `.dbf`, `.prj`, `.aux.xml`, `.tfw`, …) and hidden files are ignored silently. Everything else that can't be read is recorded as `unsupported` with the reason.

### 2. Check & fix

**Vector**
- missing CRS → `ASSUME_CRS` if set, otherwise a warning (CSV lon/lat columns → EPSG:4326)
- null / empty geometries → dropped
- invalid geometries → `make_valid(method="structure")`, with geometry collections reduced back to the layer's dimension
- mixed single/multi (e.g. Polygon + MultiPolygon) → promoted to Multi
- polygon rings → counter-clockwise (GeoParquet recommendation)
- geographic CRS with coordinates outside ±180/±90 → warning (wrong CRS or swapped axes)
- mixed-type / non-string column names → cast so Parquet can store them
- Shift-JIS (cp932) fallback for Japanese CSVs / shapefiles without a `.cpg`

**Raster**
- geotransform but no CRS (e.g. GSI DEM1A tiles, world file without `.prj`) → `ASSUME_CRS` if set, otherwise a warning
- GCP-only georeferencing (Sentinel-1 GRD) → warped to the GCP CRS while writing the COG
- no georeferencing at all → skipped (not geospatial)
- float raster without nodata → nodata = NaN

**Datacube**
- no spatial grid → skipped; curvilinear (2-D lat/lon) grids → written as-is with a warning
- no CRS → EPSG:4326 for plain lon/lat axes, otherwise `ASSUME_CRS`, stored as a CF `spatial_ref` grid mapping
- longitudes 0..360 → −180..180
- netCDF/GRIB storage encodings dropped (only dtype / fill / scale / offset / time units are kept)
- GRIB files that mix level types are split into one cube per level type (cfgrib)

**Point cloud**
- missing SRS → `ASSUME_CRS` if set, otherwise a warning; empty clouds are skipped

### 3. UUIDs

- **Vector:** every feature gets a UUID in `UUID_COL`. If the column already exists, valid unique values are kept and only missing, invalid or duplicate ones are regenerated, so IDs stay the same across re-runs.
- **Raster / datacube / point cloud:** the dataset UUID is embedded in the file: a `CNG_UUID` GDAL tag on the COG, a `CNG_UUID` root attribute on the Zarr, and a `CNG` VLR in the COPC.
- Every output also has a `dataset_uuid` in the manifest.

### 4. Convert

| Kind | How |
|---|---|
| Vector | `geoparquet_io`: Hilbert sort, zstd, 256 MB row groups. In 2.0 the geometry uses the native Parquet `GEOMETRY` type, and row-group bbox statistics let DuckDB etc. prune spatially. In 1.1 it is WKB plus a `bbox` covering column. Multi-layer sources produce one file per layer (`<name>__<layer>.parquet`). |
| Raster | source → VRT (CRS/nodata/tag edits, no pixel copy) → COG with DEFLATE + predictor, 512 px blocks and BIGTIFF if needed. Overviews use NEAREST for categorical rasters (paletted or single-band integer) and AVERAGE for the rest. Subdatasets (HDF, Sentinel-2) produce one COG each. |
| Datacube | opened lazily with xarray, chunked 512 × 512 on the grid and dask "auto" on other dimensions, then written to a temporary store and swapped in, so a failed run never leaves a half-written Zarr behind |
| Point cloud | PDAL `readers.*` → `writers.copc`, keeping all dimensions |

Errors are caught for each dataset. A failure is recorded with the last line of its traceback, and the run carries on.

---

## Outputs

```
data/cng/
├── vector/       *.parquet
├── raster/       *.tif         (COG)
├── datacube/     *.zarr
├── pointcloud/   *.copc.laz
├── stac/<run or folder name>/  catalog.json / collection.json + items
├── manifest.json               every dataset ever processed (merged across runs, keyed by output path)
├── file_index.csv              source → input → output → dataset_uuid (successful outputs only, utf-8-sig for Excel)
└── _work/                      extracted archives + temp files (safe to delete)
```

**`manifest.json`** has one record for each dataset: `kind, source, input, output, format, dataset_uuid, status (ok | exists | skipped | failed), crs, bounds, count, fixes, warnings, error, seconds, created`.

### STAC catalog

`build_stac_catalog(results)` catalogs **the current run only** and follows the input folder structure:
- one input folder that holds only files → a **Collection** with one Item per output
- one input folder that holds subfolders → a **Catalog** with one **Collection** per subfolder (deeper levels are flattened)
- several inputs, or loose files → a `run_<timestamp>` root
- an archive counts as a folder unless it holds a single file

Each Item has a WGS84 bbox/geometry, the projection extension (`proj:code` or `proj:wkt2`), `dataset_uuid`, the source name, and one `data` asset with the correct media type. Asset hrefs are relative, so **serve `OUT_DIR` itself** (not just `stac/`). For example, to browse it with [STAC Browser](https://github.com/radiantearth/stac-browser):

```bash
npx http-server data/cng -p 8000 --cors -c-1
# STAC Browser: SB_catalogUrl=http://localhost:8000/stac/<run>/collection.json
```

Items whose CRS can't be reprojected to WGS84 (local engineering CRS) are written without a geometry, and the notebook prints a note about them.

---

## Known limitations

- **GSI 基盤地図情報 DEM GML (`FG-GML-*-DEM*.xml`) is not supported.** GDAL can't read it, so these files are reported as `unsupported`. Convert them to GeoTIFF first.
- Plain Parquet without GeoParquet `geo` metadata, spreadsheets, images without georeferencing, PDFs, etc. are skipped as non-geospatial.
- Curvilinear datacubes are written without a CRS grid mapping.
- rasterio's pip wheel bundles GDAL 3.10, which can't read Zarr v3. Use xarray/zarr-python (or conda GDAL ≥ 3.11) to open the datacube outputs, or set `ZARR_FORMAT = 2`.
- Only tested on Windows.
