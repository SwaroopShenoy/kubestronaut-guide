# Image Scanning with Trivy

Up: [CKS hub](../../README.md) · Domain 5 — Supply Chain Security (20%) · Prev: [Runtime sandboxes](../04-microservice-vulnerabilities/runtime-sandboxes.md) · Next: [Image signing with Cosign](image-signing-cosign.md)

## Exam scope

**In scope:** running Trivy against an image, reading severities and fixed versions, failing a build on findings, and scanning Kubernetes manifests for misconfigurations.

The earlier course stated that Trivy exits non-zero on any finding by default. That is wrong: it exits 0 unless you pass `--exit-code 1`.

## Install

Install from the project's official docs (aquasecurity.github.io/trivy) for your OS. Confirm the version:

```bash
trivy version
```

Pin the version you install in CI. Do not use `@master` for the GitHub Action in production; pin a release tag.

## Scan an image

```bash
trivy image nginx:1.25
trivy image --severity HIGH,CRITICAL nginx:1.25
trivy image --ignore-unfixed --severity CRITICAL nginx:1.25    # only findings with a fix
```

Reading the table:

- **Library / Vulnerability ID / Severity / Installed Version / Fixed Version**
- Fixed Version empty: no upstream fix yet. Mitigate (remove the package, change the base image, or accept with a documented exception).
- Fixed Version present: upgrade to at least that version by rebuilding the image.

The output of real scans will not match any example in this course; the CVE IDs and counts change daily. Read the columns, not memorized results.

## Gate a build on findings

Trivy's exit code is the gate:

```bash
trivy image --severity CRITICAL --exit-code 1 --ignore-unfixed myapp:1.0
echo "exit=$?"    # 0 = pass, 1 = findings
```

Exceptions go in `.trivyignore` (one CVE ID per line), reviewed and committed:

```bash
trivy image --ignorefile .trivyignore --severity CRITICAL --exit-code 1 myapp:1.0
```

## Output formats

```bash
trivy image -f json -o results.json myapp:1.0
jq -r '.Results[]?.Vulnerabilities[]? | select(.Severity=="CRITICAL") | "\(.PkgName) \(.VulnerabilityID) \(.FixedVersion // "no-fix")"' results.json
```

## Scan without pulling into the local daemon

```bash
trivy image --input myapp.tar          # image saved with docker save
trivy image registry.example.com/team/app@sha256:...   # scan by digest
```

Scan by digest in pipelines so the scanned image is the image you deploy.

## Scan Kubernetes manifests for misconfigurations

```bash
trivy config ./manifests/
trivy config --severity HIGH,CRITICAL ./manifests/
```

This flags things like privileged containers, missing resource limits, and hostPath mounts in YAML, before anything is applied. It complements PSA and admission policy; it does not enforce anything at runtime.

## CI example

```yaml
# .github/workflows/scan.yml (fragment)
- name: Build
  run: docker build -t app:${{ github.sha }} .
- name: Scan
  uses: aquasecurity/trivy-action@0.28.0      # pin a real release tag
  with:
    image-ref: app:${{ github.sha }}
    severity: CRITICAL,HIGH
    ignore-unfixed: true
    exit-code: "1"
```

Check the action's README for the current tag and inputs before copying.

## SBOM

```bash
trivy image --format cyclonedx --output sbom.cdx.json myapp:1.0
```

SBOM output is useful for audit trails and for re-scanning later without pulling the image.

## Common mistakes

- Assuming a clean scan means no risk. Scanners only know CVEs in their database; they miss misconfigurations and custom code.
- Gating on all severities and blocking every build on unfixable findings. Use `--ignore-unfixed` and a reviewed `.trivyignore`.
- Scanning a tag that was rebuilt since. Scan the digest you push.

## Quick reference

```bash
trivy image --severity CRITICAL --exit-code 1 --ignore-unfixed <image>
trivy image --ignorefile .trivyignore <image>
trivy config <dir>
trivy image --format cyclonedx -o sbom.json <image>
```
