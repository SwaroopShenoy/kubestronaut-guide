# SBOM and the Software Supply Chain

Up: [CKS README](../../README.md) · Domain 5 — Supply Chain Security (20%) · Prev: [Static analysis (Kubesec, KubeLinter)](static-analysis.md) · Next: [Audit logging](../06-monitoring-logging-runtime/audit-logging.md)

An image reaches a cluster through a chain of steps, and an attacker can enter at any of them. This topic covers the curriculum's supply chain items: understanding the chain (SBOM, CI/CD, artifact repositories) and securing it (permitted registries, signed and validated artifacts).

Official curriculum topics: "Understand your supply chain (e.g. SBOM, CI/CD, artifact repositories)" and "Secure your supply chain (permitted registries, sign and validate artifacts, etc.)". The documentation for `bom` (kubernetes-sigs.github.io/bom) is on the exam's allowed-resources list.

## The chain

| Stage | Risk | Control |
|---|---|---|
| Source code | Malicious or vulnerable changes | Code review, protected branches, signed commits |
| Dependencies | Vulnerable or typosquatted packages | Pinned versions, dependency scanning |
| Build (CI/CD) | A compromised pipeline signs and ships bad artifacts | Least-privilege pipeline credentials, isolated runners, pinned actions |
| Artifact repository | Tampering or unauthorised pushes | Access control, immutable tags, scan on push |
| Deployment | Untrusted images run in the cluster | Permitted registries, signature verification, admission policy |

## SBOM

A Software Bill of Materials lists the packages inside an artifact, with versions and licenses. It lets you answer "does this new CVE affect anything we run?" without pulling and unpacking every image.

Common formats: SPDX and CycloneDX.

### Generate with bom

`bom` is the Kubernetes SIGs tool for producing SPDX documents.

```bash
bom generate --image registry.example.com/team/app:1.0 --output app.spdx
bom document outline app.spdx
```

The exact flags depend on the installed version; run `bom generate --help` and check the documentation on the allowed-resources list.

### Generate with other tools

```bash
trivy image --format spdx-json --output app.spdx.json registry.example.com/team/app:1.0
trivy image --format cyclonedx --output app.cdx.json registry.example.com/team/app:1.0
syft registry.example.com/team/app:1.0 -o spdx-json > app.spdx.json
```

### Use an SBOM

- Scan the SBOM instead of the image: `trivy sbom app.cdx.json`.
- Store it next to the image as an attestation (see [Image signing with Cosign](image-signing-cosign.md)).
- Keep it with release records so it can be searched when a new CVE is published.

## CI/CD security

- Give pipeline jobs the minimum credentials, and short-lived ones where possible.
- Do not print secrets in logs; use the platform's secret store.
- Pin third-party actions and base images by version or digest.
- Run builds on isolated, ephemeral runners.
- Require review before pipeline definitions change.
- Sign artifacts in the pipeline and record provenance.

## Artifact repositories

- Private registry with authentication and role-based access.
- Push access limited to the pipeline identity.
- Immutable tags, or deploy by digest.
- Vulnerability scanning on push and on a schedule.
- Retention rules that keep signed, scanned releases.

Private registry credentials are supplied to pods as an image pull secret:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=<user> \
  --docker-password=<password> \
  -n production
```

```yaml
spec:
  imagePullSecrets:
  - name: regcred
```

A ServiceAccount can carry the pull secret so every pod that uses it inherits it:

```bash
kubectl patch serviceaccount default -n production \
  -p '{"imagePullSecrets":[{"name":"regcred"}]}'
```

## Permitted registries

Allowing images only from approved registries stops a pod from pulling arbitrary images. Enforce it at admission; see [Admission control](admission-control.md) for a ValidatingAdmissionPolicy that rejects images from other registries.

## Sign and validate

Signing and verification are covered in [Image signing with Cosign](image-signing-cosign.md). Combine them: build, scan, generate an SBOM, sign the digest, and verify the signature at admission.

## Common mistakes

- Treating the SBOM as a one-off file rather than keeping it searchable.
- Verifying a tag instead of a digest.
- Allowing the pipeline to push to production registries with a long-lived admin credential.
- Enforcing permitted registries only in documentation, not at admission.

---

Prev: [Static analysis (Kubesec, KubeLinter)](static-analysis.md) · Next: [Audit logging](../06-monitoring-logging-runtime/audit-logging.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
