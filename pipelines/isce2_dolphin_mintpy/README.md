# isce2-dolphin-mintpy pipeline (Argo)

Download → ISCE2 coregistration with ESD (radar coordinates) → dolphin phase linking
→ MintPy corrections → point file. The code of each step lives in its own repository
and runs from its published image:

| Step | Repository | Image |
| --- | --- | --- |
| Download SLCs and orbits | `SLC_downloader` | `ghcr.io/lukas-scharf-plus/slc_downloader` |
| ISCE2, crop, dolphin, MintPy | `isce2-dolphin-minpty` (GitHub: `isce2-dolphin-mintpy`) | `ghcr.io/lukas-scharf-plus/isce2-dolphin-mintpy` |
| Point file with EGMS-style parameters | `mintpy_postprocessing` | `ghcr.io/lukas-scharf-plus/isce2-dolphin-minpty_postprocessing` |

## Files

```
pipelines/isce2_dolphin_mintpy/
├── workflow.yaml             download -> isce -> dolphin -> mintpy -> post-processing
├── configs/
│   └── ASC_073/
│       └── pipeline.yaml     pipeline config with paths inside the containers
└── README.md
```

## How the pipeline works

`workflow.yaml` runs six steps one after another:

```
download-slc -> download-orbits -> isce -> dolphin -> mintpy -> postprocess
```

| Step | Command in the pipeline image | What it does |
| --- | --- | --- |
| `isce` | `run pipeline.yaml --steps isce crop` | DEM conversion and cut, swath selection, ISCE2 topsStack (NESD) of all bursts covering the AOI. Then the AOI crop in radar coordinates and the water fraction on dolphin's grid |
| `dolphin` | `run pipeline.yaml --steps dolphin` | phase linking, interferograms, unwrapping (whirlwind) with the water mask |
| `mintpy` | `run pipeline.yaml --steps mintpy` | MintPy inputs from dolphin and ISCE2, network inversion, corrections, geocoding |
| `postprocess` | `run_postprocessing.py --run-dir /work/processed` | one GeoParquet point file |

All steps share one volume, mounted at `/work`:

```
/work/
├── input/
│   ├── slc/S1_073_ASC/   .SAFE folders            (download-slc)
│   ├── orbits/           .EOF files               (download-orbits)
│   ├── dem/              cop_dem.tif              (input artifact, isce step)
│   ├── watermask/        watermask.tif            (input artifact, isce step)
│   └── config/           pipeline.yaml            (input artifact, isce step)
├── processed/            the pipeline's work_dir
│   ├── isce/             ISCE2 topsStack           (not uploaded, ~5 GB per date)
│   ├── crop/             AOI windows, water fraction -> output artifact
│   ├── dolphin/          dolphin's work directory  -> output artifact
│   ├── mintpy/           MintPy's work directory, incl. pic/ (figures of every step) -> output artifact
│   ├── logs/             all pipeline logs, versions.json -> output artifact
│   └── pipeline_config.yaml                       -> output artifact
├── final/
│   ├── S1_ASC_073_<swaths>_VV_<first>_<last>_<area_name>.parquet  -> output artifact
│   └── S1_ASC_073_<swaths>_VV_<first>_<last>_<area_name>.metadata.json
└── logs/                 argo_<step>.log, peaks, storage_report.txt/.json
```

**Inputs** are files in the artifact bucket. Their keys are the workflow parameters
`dem_key`, `watermask_key` and `pipeline_config_key`. The DEM and the water mask are
the same files as in the compass-dolphin pipeline.

**Outputs** are uploaded unpacked to `<s3_output_prefix>/<workflow name>/processed/...`
and `.../final/`. The workflow name, for example `isce2-dolphin-mintpy-x7k2p`, is unique
per run. The SLCs, orbits and the ISCE2 stack are not uploaded. `processed/logs` is
uploaded after each of the three processing steps, so the logs of a failed step end up
in the bucket too.

**Logs.** Each processing step tees its console output to `/work/logs/argo_<step>.log`
and uploads it to `<s3_output_prefix>/<workflow name>/logs/` (with the storage report).
The pipeline's own detailed logs are in `processed/logs/`: one log per ISCE2 command
in `isce/`, `dolphin.log`, `mintpy.log` and `pipeline.log`. dolphin's own JSON log
is `processed/dolphin/dolphin.log`.

**Storage report.** The exit handler writes the disk use of every input and processed
folder, including the subfolders of `processed/isce`, which show where the space goes,
and the peak volume usage of the `isce`, `dolphin` and `mintpy` steps. It is written to
`<s3_output_prefix>/<workflow name>/logs/storage_report.txt` (and `.json`).

## How to run it

### One-time setup

1. The three images must be published on ghcr.io. Each repository's GitHub workflow
   does this on a push to `main`. Make the packages public, or give the cluster pull access.
2. Upload the config to the bucket, at the key of `pipeline_config_key`:

   ```bash
   mkdir -p ~/bucket/WP5_Infrastructure_and_Underground_Safety/argo_configs/isce2_dolphin_mintpy
   cp -r pipelines/isce2_dolphin_mintpy/configs/ASC_073 \
     ~/bucket/WP5_Infrastructure_and_Underground_Safety/argo_configs/isce2_dolphin_mintpy/
   ```

   Upload it again after every change. The workflow reads the bucket copy.
3. The cluster needs the Kubernetes secret `cdse`, as for the compass-dolphin pipeline.
4. For the ERA5 tropospheric correction (`mintpy.troposphericDelay.method: pyaps` in the
   config), the cluster needs the Kubernetes secret `cds` with the keys `CDS_URL` and
   `CDS_KEY`, your Copernicus Climate Data Store credentials. MintPy expects
   `CDS_URL=https://cds.climate.copernicus.eu/api`. MintPy and PyAPS read the
   credentials only from `~/.cdsapirc`, so the mintpy step writes that file from the
   two variables before MintPy starts. The secret is optional for the workflow. Without
   it, set `mintpy.troposphericDelay.method: "no"` in the config, otherwise the ERA5 step fails.

### Submit

Default test run: ascending track 73, January and February 2025, greater Vienna.

```bash
argo lint pipelines/isce2_dolphin_mintpy/workflow.yaml
argo submit pipelines/isce2_dolphin_mintpy/workflow.yaml --watch
```

Other dates or parameters:

```bash
argo submit pipelines/isce2_dolphin_mintpy/workflow.yaml -p start_date=2024-01-01 -p end_date=2024-12-31

# every pixel, not only MintPy's mask, and without the per-date standard deviations
argo submit pipelines/isce2_dolphin_mintpy/workflow.yaml -p postprocess_all_pixels=true \
    -p postprocess_exclude="displacement_std"
```

- `platforms`: satellites to download, `all` (default) or e.g. `S1A`, `"S1C S1D"`.
- `area_name`: name of the area in the point file name and metadata (default `vienna`).
- `postprocess_exclude`: attribute groups or columns to leave out, separated by spaces.
  The groups are listed in the `mintpy_postprocessing` README.
- `postprocess_all_pixels`: `true` writes every pixel, not only those in MintPy's final mask.

## Configuration notes

- **Config paths** must be paths inside the containers (`/work/...`). The download step
  writes to `/work/input/slc/S1_<track, 3 digits>_<ASC|DES>`, which must match `paths.slc_dir`.
- **DEM datum.** `cop_dem.tif` already holds WGS84 ellipsoid heights. It reads 221.5 m at
  Stephansdom, which is the Copernicus EGM2008 height plus the 44.7 m geoid undulation.
  The config therefore says `dem_vertical_datum: ellipsoid`. With a geoid DEM, for example
  the original Copernicus tiles, set `egm2008`, and the pipeline converts it.
- **AOI.** `aoi.bbox` is the same area as the dolphin bounds of the compass-dolphin
  pipeline. ISCE2 processes every burst overlapping it, in the swaths whose real
  footprints cover it. dolphin and MintPy only process the AOI itself.
- **Volume size.** Measured in the 3-burst test, ISCE2 needs about 2.5 GB per date for the
  coregistered and merged SLCs, plus about 10 GB of full-resolution geometry. Greater
  Vienna is about twice the bursts, so expect about 8 GB SAFE + 5 GB ISCE2 per date, plus
  about 20 GB fixed. 2 months fit in the default 300 GB. For a longer period, raise
  `storage` under `volumeClaimTemplates`. Check the storage report of a first run, because
  these are estimates.
- **MintPy options.** In the `mintpy` section of `pipeline.yaml`, every key that starts with
  `mintpy.` is an original `smallbaselineApp.cfg` option, passed to MintPy unchanged. Any
  MintPy option can be added there; see the isce2-dolphin-minpty README. An explicit
  `mintpy.reference.date` must be a date of the processed period. The isce step checks
  this before ISCE2 starts. Figures of every MintPy step are in `processed/mintpy/pic/`.
- **Resources.** isce and dolphin request 8 CPUs and 32 GiB (limit 15 CPUs, 56 GiB, the
  cluster maximum per container). `isce.num_proc × isce.omp_threads` and
  `dolphin.worker_settings.threads_per_worker` and
  `dolphin.unwrap_options.whirlwind_options.num_threads` in the config match the 8 CPUs.
  mintpy requests 4 CPUs and 16 GiB.
- **dolphin options.** The `dolphin` section of `pipeline.yaml` uses dolphin's own config
  structure, and every dolphin option can be set there. The isce2-dolphin-minpty file
  `config/dolphin_options_reference.yaml` lists them all. Unwrapping uses whirlwind
  (whirlwind-insar 0.10.0), like the compass-dolphin pipeline.
- **Images** default to `latest` with `imagePullPolicy: Always`. Pin `sha-<commit>`
  tags for reproducible runs. The pipeline image records its commit in
  `processed/logs/versions.json` and in the point file's metadata.
- **Clean-up and retries.** The volume is deleted with the workflow. A failed run is
  resubmitted from the start, including ISCE2. The uploaded outputs stay in the bucket.
