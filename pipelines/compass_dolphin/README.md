# compass-dolphin pipeline (Argo)

Download → COMPASS (geocoded GSLCs) → dolphin → point file. The code of each
step lives in its own repository and runs from its published image:

| Step | Repository | Image |
| --- | --- | --- |
| Download SLCs and orbits | `SLC_downloader` | `ghcr.io/lukas-scharf-plus/slc_downloader` |
| COMPASS (GSLCs) and dolphin (time series) | `compass_dolphin_claude` (GitHub: `compass-dolphin`) | `ghcr.io/lukas-scharf-plus/compass-dolphin` |
| Point file with EGMS-style parameters | `dolphin_postprocessing` | `ghcr.io/lukas-scharf-plus/dolphin_postprocessing` |

## Files

```
pipelines/compass_dolphin/
├── workflow.yaml             download -> COMPASS -> dolphin -> post-processing
├── configs/
│   └── ASC_073/
│       ├── compass.yaml      COMPASS config with paths inside the containers
│       └── dolphin.yaml      dolphin config with paths inside the containers
└── README.md
```

## How the InSAR pipeline works

`workflow.yaml` runs five steps one after another:

```
download-slc -> download-orbits -> compass -> dolphin -> postprocess
```

All steps share one volume, mounted at `/work`, so each step finds the files of
the previous ones:

```
/work/
├── input/
│   ├── slc/S1_073_ASC/   .SAFE folders            (download-slc)
│   ├── orbits/           .EOF files               (download-orbits)
│   ├── dem/              cop_dem.tif              (input artifact, compass step)
│   ├── burst_db/         OPERA burst database     (input artifact, compass step)
│   ├── watermask/        watermask.tif            (input artifact, dolphin step)
│   └── config/           compass.yaml, dolphin.yaml (input artifacts)
├── processed/
│   ├── compass/          gslcs/ and logs/         -> output artifact
│   └── dolphin/          dolphin's work directory -> output artifact
└── final/
    ├── S1_ASC_073_IW1-IW2_VV_<first>_<last>_<area_name>.parquet  -> output artifact
    └── S1_ASC_073_IW1-IW2_VV_<first>_<last>_<area_name>.metadata.json
```

The point file is named after its content (direction, track, subswaths,
polarization, first and last acquisition, the `area_name` parameter) and
contains the processing metadata: software versions, the SLC and orbit files
used, the full dolphin config, and the run details passed as `INSAR_RUN_*`
variables (workflow name, requested dates, platforms, image names). See the
dolphin_postprocessing README, "Output file name and metadata".

**Inputs** are files in the artifact bucket. Their keys are workflow parameters
(`dem_key`, `watermask_key`, `burst_db_key`, `compass_config_key`,
`dolphin_config_key`). Argo downloads each one before the step that needs it.

**Outputs** of the compass, dolphin and postprocess steps are uploaded unpacked
(no `.tgz`) to

```
<s3_output_prefix>/<workflow name>/processed/compass
<s3_output_prefix>/<workflow name>/processed/dolphin
<s3_output_prefix>/<workflow name>/reference_point.json   (copy of processed/dolphin/reference_point.json)
<s3_output_prefix>/<workflow name>/final/S1_..._<area_name>.parquet (and .metadata.json)
```

The workflow name (e.g. `insar-pipeline-x7k2p`) is unique per run, so runs never
overwrite each other. The downloaded SLCs and orbits are not uploaded.

**Logs.** Each step pipes its full console output (stdout and stderr) through
`tee` into a log file, which is uploaded with the step's outputs, also when the
step fails:

```
<s3_output_prefix>/<workflow name>/logs/download-slc.log
<s3_output_prefix>/<workflow name>/logs/download-orbits.log
<s3_output_prefix>/<workflow name>/processed/compass/logs/run_pipeline_compass.log
<s3_output_prefix>/<workflow name>/processed/dolphin/run_pipeline_dolphin.log
<s3_output_prefix>/<workflow name>/final/run_postprocessing.log
```

COMPASS's per-job logs (`processed/compass/logs/`) and dolphin's own `dolphin.log`
(`processed/dolphin/`) are uploaded as before.

**Storage report.** After every run, also a failed one, the exit handler
`storage-report` measures the disk use of each folder on the shared volume and
writes it to `<s3_output_prefix>/<workflow name>/logs/storage_report.txt` (and
`.json`, also printed in the pod log):

- every input folder (`slc` with the number of SAFE folders, `orbits`, `dem`,
  `watermask`, `burst_db`, `config`),
- `processed/compass` and `processed/dolphin` with their subfolders,
- `final` with each file (the `.parquet`, its `.metadata.json` and the log),
- the **peak** volume usage while compass and dolphin ran, and the volume size.

The peak matters for sizing the volume: COMPASS deletes its scratch files at the
end, so the final folder sizes alone are too small. The compass and dolphin
wrappers sample `df /work` every 60 s in the background and keep the maximum in
`/work/logs/peak_<step>.bytes`.

COMPASS and dolphin are separate steps of the same image. If dolphin fails, the
GSLCs have already been uploaded, and COMPASS does not have to be repeated.

## How to run it

### One-time setup

1. The three images must be published on ghcr.io (each repository's GitHub
   workflow does this on a push to `main`).
2. Upload the config files to the bucket, at the keys of `compass_config_key`
   and `dolphin_config_key`:

   ```bash
   mkdir -p ~/bucket/WP5_Infrastructure_and_Underground_Safety/argo_configs/compass_dolphin
   cp -r pipelines/compass_dolphin/configs/ASC_073 \
     ~/bucket/WP5_Infrastructure_and_Underground_Safety/argo_configs/compass_dolphin/
   ```

   Upload them again after every change; the workflow reads the bucket copy, not
   this repository.
3. The cluster needs the Kubernetes secret `cdse` with the keys `S3_KEY` and
   `S3_SECRET` (CDSE S3 credentials, used by the download steps).

### Submit

Default test run: ascending track 73, January and February 2025.

```bash
argo lint pipelines/compass_dolphin/workflow.yaml
argo submit pipelines/compass_dolphin/workflow.yaml --watch
```

Other dates or parameters:

```bash
argo submit pipelines/compass_dolphin/workflow.yaml -p start_date=2024-01-01 -p end_date=2024-12-31

# only S1A scenes, and a point file without the per-date standard deviations
argo submit pipelines/compass_dolphin/workflow.yaml -p platforms=S1A \
    -p postprocess_exclude="displacement_std persistent_scatterers"
```

- `platforms`: satellites to download, `all` (default) or e.g. `S1A`, `"S1C S1D"`.
- `postprocess_exclude`: attribute groups or columns to leave out of the point
  file, separated by spaces. Empty (default) writes all attributes. The groups
  are listed in the `dolphin_postprocessing` README (`--exclude`).
- `area_name`: name of the area in the point file name and metadata (default
  `vienna`). Empty leaves it out of the name.
- `reference_point`: reference point of the dolphin time series as `"LON LAT"`
  (WGS84 degrees), e.g. `-p reference_point="16.3804 48.1484"`. Empty (default)
  lets dolphin pick the point. The point used is written to
  `processed/dolphin/reference_point.json` and uploaded as
  `<workflow name>/reference_point.json` (skipped, without failing the step, when
  the dolphin config turns off the time series).

For a longer period, also raise the volume size (`storage` under
`volumeClaimTemplates`, about 8 GB per SLC).

## Configuration notes

- **Config paths** must be paths inside the containers (`/work/...`), not paths
  on your computer. Track and bursts in `compass.yaml` must match the
  `relative_orbit` and `orbit_state` parameters; the download step writes to
  `/work/input/slc/S1_<track, 3 digits>_<ASC|DES>`.
- **Resources:** compass and dolphin request 8 CPUs and 32 GiB memory (limit
  15 CPUs, 56 GiB: the cluster allows at most this per container and refuses
  larger limits). `n_workers × threads_per_worker` in `compass.yaml` should
  match the CPU request. Lower the requests if pods stay `Pending` because no node
  is large enough.
- **Images:** all three images default to `latest`, and every step uses
  `imagePullPolicy: Always`, so each run uses the newest image from `main`.
  For reproducible results, pin them to `sha-<commit>` tags.
- **Users:** all images run as the non-root user `mambauser` (UID/GID 57439); the
  workflow's `securityContext` with `fsGroup: 57439` makes the volume writable.
- **Clean-up:** a successful workflow is deleted after 5 minutes, a failed one
  after 2 hours (`ttlStrategy`). The uploaded artifacts stay in the bucket.
- Cluster-specific settings (`serviceAccountName`, node affinity, tolerations,
  storage class) are those of the ADUCAT cluster.
