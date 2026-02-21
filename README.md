# Dagger Container Security Demo

This repository demonstrates a minimal container security pipeline using your Daggerverse modules:
- `apko` to build/publish a minimal Python image
- `syft` to generate SBOM output
- `grype` to scan the SBOM for vulnerabilities

## Repository Structure

- `apko/container.yaml` - Apko image definition
- `.github/workflows/minimal-python-app.yaml` - CI/CD pipeline in GitHub Actions

## Apko Configuration

Current image definition (`apko/container.yaml`):

```yaml
contents:
  keyring:
    - https://packages.wolfi.dev/os/wolfi-signing.rsa.pub
  repositories:
    - https://packages.wolfi.dev/os
  packages:
    - wolfi-base
    - python-3.12-base

entrypoint:
  command: /usr/bin/python3

work-dir: /app

archs:
  - amd64
```

## CI/CD Flow

Workflow name: `apko-minimal-python-scan`

Triggers:

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
```

Shared image reference:

```yaml
env:
  IMAGE_REF: ttl.sh/dagger-apko-${{ github.run_id }}:12h
```

### 1) Build image with Apko

Job: `build-apko-image`

```yaml
- name: Build with apko image and publish
  uses: dagger/dagger-for-github@v8.2.0
  with:
    version: "latest"
    module: github.com/hampusctl/daggerverse/apko@main
    args: "build --config-file=apko/container.yaml publish --address=${{ env.IMAGE_REF }}"
```

### 2) Generate SBOM with Syft

Job: `generate-sbom-apko-image` (depends on build)

```yaml
- name: scan apko image
  uses: dagger/dagger-for-github@v8.2.0
  with:
    version: "latest"
    module: github.com/hampusctl/daggerverse/syft@main
    args: "scan --image=${{ env.IMAGE_REF }} export --path=syft-output"
```

Then CI:
- appends `syft-output/report.md` to `GITHUB_STEP_SUMMARY`
- uploads `syft-output/` as artifact `syft-report`

### 3) Scan SBOM with Grype

Job: `cve-scan-apko-image` (depends on SBOM job)

```yaml
- name: Download syft output directory
  uses: actions/download-artifact@v4
  with:
    name: syft-report
    path: syft-output/

- name: Scan SBOM with grype module
  uses: dagger/dagger-for-github@v8.2.0
  with:
    version: "latest"
    module: github.com/hampusctl/daggerverse/grype@main
    args: "scan --sbom=syft-output/sbom.json export --path=grype-output"
```

Then CI:
- appends `grype-output/report.md` to `GITHUB_STEP_SUMMARY`
- uploads `grype-output/` as artifact `grype-report`

## Verify a Run

After a workflow run:
- Open each job summary and verify Syft/Grype report content is rendered
- Download artifacts:
  - `syft-report`
  - `grype-report`
- Confirm `sbom.json` exists in Syft output and `report.md` exists in both outputs

## Notes

- Images are published to `ttl.sh` and expire automatically.
- This repo is a baseline for adding more demos (for example Melange and Grant) using the same Dagger-driven workflow pattern.
