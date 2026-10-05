# Supply Chain and Images

Up: [KCSA hub](../../README.md) · Domain 5 — Platform Security (16%) · Prev: [Threat model and attack paths](../04-kubernetes-threat-model/threat-model-and-attack-paths.md) · Next: [Admission and policy](admission-and-policy.md)

Software reaches a cluster through a chain of builds, registries and deployments, and any link can be attacked. This chapter covers the supply chain and the images that travel along it.

## Software supply chain

The path from source to running workload: code, dependencies, build, package, registry, deploy. An attacker who compromises any step can ship code you trust.

## SLSA

Supply-chain Levels for Software Artifacts is a framework of increasing build-integrity requirements. Higher levels require tamper-resistant, auditable builds and provenance. Check the current SLSA specification for the exact level definitions; the wording changed between versions.

## SBOM

A Software Bill of Materials lists the components in an artifact, with versions and licenses. It lets you find out quickly whether a new CVE affects you. Common formats: SPDX and CycloneDX. Tools include Syft and Trivy.

## Signing and verification

- Sign images at build time (Cosign from the Sigstore project, or Notary).
- Verify at deploy time with an admission check.
- Verify by digest. A tag can be moved to a different image.
- Keyless signing uses the CI workload's OIDC identity instead of a long-lived key.

## Image scanning

- Scanners such as Trivy and Grype match installed packages against vulnerability databases.
- Scan in CI and on push to a registry; rescan as new CVEs appear.
- A scanner finds known CVEs. It does not find misconfigurations in your manifests or flaws in your own code.

## Image hygiene

- Minimal base images (distroless, Alpine, scratch) reduce packages and CVEs.
- Multi-stage builds keep compilers out of the final image.
- Run as a non-root user.
- Keep secrets out of image layers.
- Pin versions or digests; avoid `latest`.
- Use a trusted, private registry with access control.
