# Packaging with Helm and Kustomize

Up: [KCNA hub](../../KCNA_Crash_Course.md) · Domain 3 — Cloud Native Application Delivery (16%) · Prev: [CI/CD and GitOps](cicd-and-gitops.md) · Next: [Principles and patterns](../04-cloud-native-architecture/principles-and-patterns.md)

## Helm

Helm is the package manager for Kubernetes. A chart is a package of templated manifests; a release is one installed instance of a chart.

Key terms:

- **Chart**: the package.
- **Values**: parameters that fill the templates.
- **Release**: an installed instance, with its own history.
- **Repository**: where charts are stored.

Helm 3 removed Tiller, the in-cluster component from Helm 2. Releases are stored in the cluster as Secrets.

Commands by concept: install, upgrade, rollback, list, uninstall.

## Kustomize

Kustomize customizes plain YAML without templates. You define a base and overlays that patch it for each environment. It is built into kubectl (`kubectl apply -k`).

Use Helm when you need a packaged, parameterized product. Use Kustomize when you own the manifests and need per-environment variations.

## Choosing

| Situation | Tool |
|---|---|
| Install a third-party application | Helm |
| Version and share your own app with parameters | Helm |
| Per-environment differences in your own manifests | Kustomize |
| GitOps with either | Both work with Argo CD and Flux |

## Practice questions

- What is a Helm release? (An installed instance of a chart)
- Does Kustomize use templates? (No; it uses base and overlays with patches)
- Did Helm 3 keep Tiller? (No; it was removed)
