# aducat_argo

Argo Workflows templates for the ADUCAT InSAR processing chain, and the config
files they use. The code of each step lives in its own repository and is run
from its published Docker image:

| Step | Repository | Image |
| --- | --- | --- |
| Download SLCs and orbits | `SLC_downloader` | `ghcr.io/lukas-scharf-plus/slc_downloader` |
| COMPASS (GSLCs) and dolphin (time series) | `compass_dolphin_claude` (GitHub: `compass-dolphin`) | `ghcr.io/lukas-scharf-plus/compass-dolphin` |
| Point file with EGMS-style parameters | `dolphin_postprocessing` | `ghcr.io/lukas-scharf-plus/dolphin_postprocessing` |

This repository contains no Python code, only YAML.

## Project structure

```
aducat_argo/
├── workflows/
│   └── insar_pipeline.yaml   download -> COMPASS -> dolphin -> post-processing
├── configs/
│   └── ASC_073/
│       ├── compass.yaml      COMPASS config with paths inside the containers
│       └── dolphin.yaml      dolphin config with paths inside the containers
└── README.md
```

## How the InSAR pipeline works

`workflows/insar_pipeline.yaml` runs five steps one after another:

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
    └── dolphin_points.parquet                     -> output artifact
```

**Inputs** are files in the artifact bucket. Their keys are workflow parameters
(`dem_key`, `watermask_key`, `burst_db_key`, `compass_config_key`,
`dolphin_config_key`). Argo downloads each one before the step that needs it.

**Outputs** of the compass, dolphin and postprocess steps are uploaded unpacked
(no `.tgz`) to

```
<s3_output_prefix>/<workflow name>/processed/compass
<s3_output_prefix>/<workflow name>/processed/dolphin
<s3_output_prefix>/<workflow name>/final/dolphin_points.parquet
```

The workflow name (e.g. `insar-pipeline-x7k2p`) is unique per run, so runs never
overwrite each other. The downloaded SLCs and orbits are not uploaded.

COMPASS and dolphin are separate steps of the same image. If dolphin fails, the
GSLCs have already been uploaded, and COMPASS does not have to be repeated.

## How to run it

### One-time setup

1. The three images must be published on ghcr.io (each repository's GitHub
   workflow does this on a push to `main`).
2. Upload the config files to the bucket, at the keys of `compass_config_key`
   and `dolphin_config_key`:

   ```bash
   cp -r configs/ASC_073 ~/bucket/WP5_Infrastructure_and_Underground_Safety/argo_configs/
   ```

   Upload them again after every change; the workflow reads the bucket copy, not
   this repository.
3. The cluster needs the Kubernetes secret `cdse` with the keys `S3_KEY` and
   `S3_SECRET` (CDSE S3 credentials, used by the download steps).

### Submit

Default test run: ascending track 73, January and February 2025.

```bash
argo lint workflows/insar_pipeline.yaml
argo submit workflows/insar_pipeline.yaml --watch
```

Other dates or parameters:

```bash
argo submit workflows/insar_pipeline.yaml -p start_date=2024-01-01 -p end_date=2024-12-31
```

For a longer period, also raise the volume size (`storage` under
`volumeClaimTemplates`, about 8 GB per SLC).

## Configuration notes

- **Config paths** must be paths inside the containers (`/work/...`), not paths
  on your computer. Track and bursts in `compass.yaml` must match the
  `relative_orbit` and `orbit_state` parameters; the download step writes to
  `/work/input/slc/S1_<track, 3 digits>_<ASC|DES>`.
- **Resources:** compass and dolphin request 8 CPUs and 32 GiB memory (limit
  16 CPUs, 64 GiB). `n_workers × threads_per_worker` in `compass.yaml` should
  match the CPU request. Lower the requests if pods stay `Pending` because no node
  is large enough.
- **Images:** `compass_dolphin_image` defaults to `latest`. After a successful
  test, pin it to a `sha-<commit>` tag so results stay reproducible.
- **Users:** all images run as the non-root user `mambauser` (UID/GID 57439); the
  workflow's `securityContext` with `fsGroup: 57439` makes the volume writable.
- **Clean-up:** a successful workflow is deleted after 5 minutes, a failed one
  after 2 hours (`ttlStrategy`). The uploaded artifacts stay in the bucket.
- Cluster-specific settings (`serviceAccountName`, node affinity, tolerations,
  storage class) are those of the ADUCAT cluster.
