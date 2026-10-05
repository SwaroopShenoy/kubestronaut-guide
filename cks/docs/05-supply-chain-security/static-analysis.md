# Static Analysis of Workloads and Images

Up: [CKS README](../../README.md) · Domain 4 — Supply Chain Security (20%) · Prev: [Admission control](admission-control.md) · Next: [SBOM and the software supply chain](sbom-and-supply-chain.md)

Some problems are visible in a manifest before anything runs. This topic covers static analysis of workloads and images with Kubesec and KubeLinter.

Official curriculum topic: "Perform static analysis of user workloads and container images (e.g. Kubesec, KubeLinter)". Static analysis reads manifests and images without running them.

## What static analysis checks

- **Manifests:** missing security settings (non-root, read-only filesystem, dropped capabilities), privileged containers, host namespaces, missing resource limits, hostPath mounts, and wildcard RBAC.
- **Images:** embedded secrets, unneeded packages, known vulnerabilities, and whether the image runs as root.

## Tools named in the curriculum

- **Kubesec**: scores a manifest's security settings and lists what to fix.
- **KubeLinter**: lints Kubernetes YAML against configurable checks (for example, required probes or non-root users).

Example usage (check current tool documentation for exact flags):

```bash
kubesec scan deployment.yaml
kube-linter lint deployment.yaml
```

## Where to run it

- Locally before commit, as a pre-commit hook.
- In CI, failing the pipeline on high-severity findings.
- Before admission, as a gate for manifests from outside sources.

Static analysis complements admission control. Admission enforces rules at deploy time; static analysis catches problems earlier and with more context.

## Other static analysis for images

Image scanners such as Trivy read the image's packages (see [Trivy](../05-supply-chain-security/image-scanning-trivy.md)). Dockerfile linters such as hadolint check build instructions.

## Limits

- Static analysis finds known patterns. It does not prove a workload is secure.
- Findings need triage: some warnings are acceptable for specific workloads, and should be documented as exceptions.
- Tool rule sets change; check the current documentation.

## Common mistakes

- Running the tool but ignoring its output.
- Suppressing findings without a recorded reason.
- Treating a clean scan as proof that a workload is safe.

---

Prev: [Admission control](admission-control.md) · Next: [SBOM and the software supply chain](sbom-and-supply-chain.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
