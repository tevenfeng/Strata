# Strata runtime image (slim)

Built by [`.github/workflows/build.yml`](../.github/workflows/build.yml) when a release is
published, and pushed to GHCR. The tag names what is inside it — the Strata version and
the CUDA toolkit of the runtime base:

```
ghcr.io/tevenfeng/strata:v0.1.38-cuda13.0   # the newest release is also :latest
```

The CUDA part is read from `docker/Dockerfile.runtime`
(`nvidia/cuda:13.0.0-runtime-ubuntu24.04`), so the tag cannot drift from the base.
Manual runs push `sha-<commit>-cuda13.0` (or the `image_tag` input with `-cuda13.0`
appended unless it already names a CUDA version).

The image is assembled from a prebuilt engine (`strata`, `strata-vision`, `BUILD.json`)
and carries no compiler or CUDA toolkit — about 3.5-4 GiB on disk (~2 GB to pull),
instead of the ~8-9 GiB of a devel-stage build.

## Run

The host needs an NVIDIA driver >= 580 and the
[NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html).

```
docker run --rm --gpus all -p 8080:8080 \
  --ulimit memlock=-1 -v strata-data:/data \
  ghcr.io/tevenfeng/strata:latest
```

The first start downloads the model (~70 GB) into the `strata-data` volume; later
starts go straight to serving. The environment variables (`MODEL`, `FAMILY`, `CONTEXT`,
`VISION`, `KV`, `GPU`/`GPUS`, `LAYER_SPLIT`, `LOW_RAM`, `HOST`, `PORT`, `API_KEY`,
`REINSTALL`) work as in the upstream README's Docker section; `HF_ENDPOINT`
(e.g. `https://hf-mirror.com`) makes every Hugging Face download use a mirror.

## Architectures

The engine is a fat binary for `75;80;86;89;120` (RTX 20/30/40/50 and A-series) plus
PTX. A card outside that set needs a new image: run the `build` workflow with `archs`
set to your card (for example `89`) and `publish` enabled.

## Reaching the zip from a bare-metal Linux setup

`setup.sh` looks for the prebuilt engine at a base URL that upstream hardcodes to its
own repository (`Niko1221/Strata`), which ships no Linux zip. To use this fork's zip
in a bare-metal install:

```
STRATA_PREBUILT_URL=https://github.com/tevenfeng/Strata/releases/latest/download/ ./setup.sh
```

or pass `--prebuilt <url>` to `setup.py`. (Patching `setup.py`'s `PREBUILT_URL` makes
the fork the default.)

## Building it yourself

`docker/Dockerfile.runtime` expects two named build contexts with the prebuilt pieces
(the workflow supplies them):

```
docker buildx build \
  --build-context engine=<dir with strata, strata-vision, BUILD.json> \
  --build-context llama=<dir with the extracted llama.cpp source tree> \
  -f docker/Dockerfile.runtime -t strata-runtime .
```

For a from-source build without a prebuilt engine, use the upstream `Dockerfile`
instead.
