# aducat_argo

Argo Workflows for the ADUCAT InSAR processing chains, and the config files they
use. Each pipeline has its own folder with its workflow, its configs and a README.
The code of every step lives in its own repository and runs from its published
Docker image. This repository contains no Python code, only YAML.

| Pipeline | Steps | Folder |
| --- | --- | --- |
| compass-dolphin | download → COMPASS (geocoded GSLCs) → dolphin → point file | [`pipelines/compass_dolphin`](pipelines/compass_dolphin/README.md) |
| isce2-dolphin-mintpy | download → ISCE2 (radar coordinates, ESD) → dolphin → MintPy → point file | [`pipelines/isce2_dolphin_mintpy`](pipelines/isce2_dolphin_mintpy/README.md) |

Both pipelines share the downloader, the DEM (`cop_dem.tif`, WGS84 ellipsoid heights)
and the water mask, and write point files with the same name pattern and metadata
layout, so their results can be compared directly.

## Project structure

```
aducat_argo/
├── pipelines/
│   ├── compass_dolphin/
│   │   ├── workflow.yaml          the Argo workflow
│   │   ├── configs/ASC_073/       compass.yaml, dolphin.yaml
│   │   └── README.md
│   └── isce2_dolphin_mintpy/
│       ├── workflow.yaml          the Argo workflow
│       ├── configs/ASC_073/       pipeline.yaml
│       └── README.md
└── README.md
```

Add a new pipeline as another folder under `pipelines/`, and a new track or area as
another folder under its `configs/`, for example `configs/DES_124/`.

## Configs in the bucket

The workflows read their configs from the artifact bucket, not from this repository.
The bucket mirrors this layout:

```
WP5_Infrastructure_and_Underground_Safety/argo_configs/<pipeline>/<track>/<file>
```

For example `argo_configs/isce2_dolphin_mintpy/ASC_073/pipeline.yaml`. Upload a
config again after every change, as described in each pipeline's README.

## Common setup

- The images must be published on ghcr.io. Each repository's GitHub workflow does
  this on a push to `main`.
- The cluster needs the Kubernetes secret `cdse` with the keys `S3_KEY` and
  `S3_SECRET`, the CDSE S3 credentials used by the download steps.
- All images run as the non-root user `mambauser` (UID/GID 57439). The workflows'
  `securityContext` with `fsGroup: 57439` makes the shared volume writable.
- Successful workflows are deleted after 5 minutes and failed ones after 2 hours
  (`ttlStrategy`). The uploaded outputs stay in the bucket.
