# Helm and Kustomize

Up: [CKA hub](../../README.md) · Domain 4 — Workloads and Scheduling (15%) · Prev: [Scheduling](scheduling.md) · Next: [Configuration, autoscaling and self-healing](configuration-and-autoscaling.md)

Applications are rarely one file. Helm packages them as charts, and Kustomize layers changes over a shared base. This topic covers both tools and when each one fits.

Helm and Kustomize were added to the CKA curriculum in the 2025 update. Confirm exact scope against the current curriculum; the tasks are usually install, upgrade, roll back, or apply an overlay.

## Helm

Helm packages Kubernetes manifests as charts and tracks each install as a release.

```bash
helm repo add <name> <url>
helm repo update
helm search repo <keyword>

helm install <release> <repo>/<chart> -n <ns> --create-namespace
helm install <release> ./mychart -f values.yaml
helm upgrade <release> <repo>/<chart> --set replicaCount=3
helm list -A
helm get values <release>
helm history <release>
helm rollback <release> <revision>
helm uninstall <release>
```

Preview without installing:

```bash
helm template <release> ./mychart -f values.yaml
helm install <release> ./mychart --dry-run
```

Create and lint a chart:

```bash
helm create mychart
helm lint mychart
```

Helm 3 has no Tiller; releases are stored as Secrets in the release namespace.

## Kustomize

Kustomize builds manifests from a base plus overlays. No templates; plain YAML with patches.

Layout:

```
base/
  deployment.yaml
  service.yaml
  kustomization.yaml
overlays/
  prod/
    kustomization.yaml
    patch.yaml
```

`overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
namespace: prod
labels:
- pairs:
    env: prod
  includeSelectors: false
replicas:
- name: web
  count: 3
images:
- name: nginx
  newTag: "1.27"
patches:
- path: patch.yaml
```

Note: `commonLabels` is deprecated in favor of `labels`. Use the field form above.

Commands:

```bash
kubectl apply -k overlays/prod
kubectl kustomize overlays/prod           # render without applying
kubectl diff -k overlays/prod
```

## Choosing between them

If the task gives a chart, use Helm. If it gives a base directory and an overlay, use Kustomize. Read the task for the exact tool name.

## Common mistakes

- Running `helm upgrade` without `-n <ns>` and modifying the wrong release.
- Forgetting that `kubectl apply -k` takes a directory that contains `kustomization.yaml`, not the file itself.
- Using a deprecated kustomize field (`commonLabels`, `bases`).

## Quick reference

```bash
helm install <r> <chart> -n <ns> -f values.yaml
helm rollback <r> <rev>
kubectl apply -k <dir>
kubectl kustomize <dir>
```

---

Prev: [Scheduling](scheduling.md) · Next: [Configuration, autoscaling and self-healing](configuration-and-autoscaling.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
