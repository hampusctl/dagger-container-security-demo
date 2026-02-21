# Dagger Container Security Demo

This repository demonstrates a minimal container security flow using Dagger modules from Daggerverse.

Current focus:
- Build a minimal Python container with Apko
- Generate an SBOM/report with Syft in GitHub Actions

## What This Repo Contains

- `apko/container.yaml`: Apko config for a minimal Wolfi-based Python image
- `.github/workflows/minimal-python-app.yaml`: CI workflow that builds and scans the image through Dagger modules

## How The Workflow Works

The workflow `apko-minimal-python-scan` runs on push, pull request, and manual dispatch.

### 1) Build and publish image (Apko module)

Job: `build-apko-image`

- Uses `dagger/dagger-for-github`
- Calls module: `github.com/hampusctl/daggerverse/apko@main`
- Runs:
  - `build --config-file=apko/container.yaml`
  - `publish --address=${IMAGE_REF}`

`IMAGE_REF` is set to a temporary image on `ttl.sh`:
- `ttl.sh/dagger-apko-${{ github.run_id }}:12h`

### 2) Scan image and export output (Syft module)

Job: `scan-apko-image` (depends on build job)

- Uses `dagger/dagger-for-github`
- Calls module: `github.com/hampusctl/daggerverse/syft@main`
- Runs:
  - `scan --image=${IMAGE_REF}`
  - `export --path=output`

Syft returns a directory. The workflow:
- reads `output/report.md` into `GITHUB_STEP_SUMMARY`
- uploads `output/` as artifact (`syft-report`)

## Apko Image Definition

`apko/container.yaml` currently builds a minimal image with:
- `wolfi-base`
- `python-3.12-base`

Entrypoint:
- `/usr/bin/python3`

Architecture:
- `amd64`

## How To Verify In GitHub Actions

After a workflow run:
- Open job summary and confirm SBOM report content is visible
- Download artifact `syft-report` and inspect files in `output/`

## Notes

- The image is published to `ttl.sh` and expires automatically.
- The commented Grype section in the workflow can be enabled later to add vulnerability scanning after Syft.
