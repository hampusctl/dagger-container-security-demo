# Dagger Container Security Demo 📦

A minimal **container security pipeline** that builds an image, generates an SBOM, then runs CVE and license scans—all via [Dagger](https://dagger.io) and [Daggerverse](https://github.com/hampusctl/daggerverse) modules.

## What happens in the pipeline 🔄

1. **Build** – Apko builds a minimal Wolfi-based Python image and publishes it to `ttl.sh`.
2. **SBOM** – Syft scans that image and produces a Software Bill of Materials (e.g. `sbom.json` + `report.md`).
3. **CVE scan** – Grype scans the SBOM and reports known vulnerabilities (using `.grype.yaml`).
4. **License scan** – Grant scans the SBOM and reports license compliance (using `.grant.yaml`).

Jobs 3 and 4 run in parallel after the SBOM job; each gets the same SBOM artifact.

## Modules used

| Module | Purpose |
|--------|--------|
| `apko` | Build and publish minimal container image (Wolfi + Python) |
| `syft` | Generate SBOM from the built image |
| `grype` | Scan SBOM for CVEs |
| `grant` | Scan SBOM for license compliance |

## Repository structure

| Path | Description |
|------|-------------|
| `apko/container.yaml` | Apko image definition (Wolfi, Python 3.12) |
| `.github/workflows/minimal-python-app.yaml` | CI/CD workflow |
| `.grype.yaml` | Grype config (e.g. ignore list) |
| `.grant.yaml` | Grant config (license policy) |

## Apko configuration

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

## CI/CD flow

**Workflow name:** `apko-minimal-python-scan`

**Triggers:** push/PR to `main`, or manual `workflow_dispatch`.

**Shared env:** `IMAGE_REF: ttl.sh/dagger-apko-${{ github.run_id }}:12h` (image is reused by Syft).

---

### 1) Build image with Apko 🐳

**Job:** `build-apko-image`

Builds from `apko/container.yaml` and publishes to `IMAGE_REF`.

```yaml
- name: Build with apko image and publish
  uses: dagger/dagger-for-github@v8.2.0
  with:
    version: "latest"
    module: github.com/hampusctl/daggerverse/apko@main
    call: |-
      build \
      --config-file=apko/container.yaml \
      publish \
      --address=${{ env.IMAGE_REF }}
```

---

### 2) Generate SBOM with Syft 📋

**Job:** `generate-sbom-apko-image` (needs: build)

Scans the published image and writes `syft-output/` (e.g. `sbom.json`, `report.md`).

```yaml
- name: scan apko image
  uses: dagger/dagger-for-github@v8.2.0
  with:
    version: "latest"
    module: github.com/hampusctl/daggerverse/syft@main
    call: |-
      scan --image=${{ env.IMAGE_REF }} \
      export \
      --path=syft-output
```

**After the step:**  
- `syft-output/report.md` is appended to the job’s **GITHUB_STEP_SUMMARY**.  
- `syft-output/` is uploaded as artifact **`syft-report`**.

---

### 3) Scan SBOM with Grype (CVE) 🔒

**Job:** `cve-scan-apko-image` (needs: SBOM job)

Downloads `syft-report`, runs Grype on `syft-output/sbom.json` with `.grype.yaml`, writes `grype-output/`.

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
    call: |-
      scan \
      --sbom=syft-output/sbom.json \
      --config=".grype.yaml" \
      export \
      --path=grype-output
```

**After the step:**  
- `grype-output/report.md` is appended to the job’s **GITHUB_STEP_SUMMARY**.  
- `grype-output/` is uploaded as artifact **`grype-report`**.

---

### 4) License scan SBOM with Grant 📄

**Job:** `license-scan-apko-image` (needs: SBOM job)

Downloads `syft-report`, runs Grant on `syft-output/sbom.json` with `.grant.yaml`, writes `grant-output/`.

```yaml
- name: Download syft output directory
  uses: actions/download-artifact@v4
  with:
    name: syft-report
    path: syft-output/

- name: License scan SBOM with grant module
  uses: dagger/dagger-for-github@v8.2.0
  with:
    version: "latest"
    module: github.com/hampusctl/daggerverse/grant@main
    call: |-
      scan \
      --sbom=syft-output/sbom.json \
      --config=".grant.yaml" \
      export \
      --path=grant-output
```

**After the step:**  
- `grant-output/report.md` is appended to the job’s **GITHUB_STEP_SUMMARY**.  
- `grant-output/` is uploaded as artifact **`grant-report`**.

---

## Verify a run ✅

After a workflow run:

1. Open each job’s **Summary** and check that Syft, Grype, and Grant reports are visible.
2. Download artifacts:
   - **`syft-report`** – SBOM and Syft report
   - **`grype-report`** – CVE scan report
   - **`grant-report`** – License scan report
3. Confirm:
   - `syft-output/sbom.json` exists in the Syft artifact.
   - `report.md` exists in Syft, Grype, and Grant outputs.

## Notes 📌

- Images are published to **ttl.sh** and expire after 12 hours.
- All Dagger steps use **`call`** with **`|-`** for multiline readability (no trailing newline).
- Grype and Grant use repo config: **`.grype.yaml`** (e.g. CVE ignore list), **`.grant.yaml`** (license policy).
