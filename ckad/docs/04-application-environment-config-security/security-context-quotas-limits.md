# SecurityContext, Quotas and Limits

Up: [CKAD hub](../../README.md) · Domain 4 — Application Environment, Configuration and Security (25%) · Prev: [ConfigMaps and Secrets](configmaps-and-secrets.md) · Next: [ServiceAccounts and RBAC](serviceaccounts-and-rbac.md)

Security starts with what a container is allowed to do, and resources decide what it can consume. This chapter covers the securityContext settings, ResourceQuota and LimitRange, and resource requests and limits.

The security depth is in [CKS security context and PSS](../../../cks/docs/04-microservice-vulnerabilities/security-context-and-pss.md). This page covers what CKAD asks you to set.

## SecurityContext

```yaml
spec:
  securityContext:              # pod level
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    runAsNonRoot: true
  containers:
  - name: app
    image: myapp:1.0
    securityContext:            # container level, overrides pod
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]   # only if binding a port below 1024
```

If the app writes to disk with a read-only root filesystem, mount an `emptyDir` on the path it writes to.

Verify:

```bash
kubectl exec <pod> -- id
kubectl exec <pod> -- touch /usr/x         # expect read-only failure
```

## ResourceQuota

Limits totals per namespace.

```bash
kubectl create quota compute --hard=requests.cpu=4,requests.memory=8Gi,limits.cpu=8,limits.memory=16Gi,pods=10 -n dev
kubectl describe quota compute -n dev
```

Equivalent YAML:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "10"
```

When a quota is exceeded, creating the pod fails with `exceeded quota`. Pods in a namespace with a compute quota must set requests and limits, or use a LimitRange to supply defaults.

## LimitRange

Sets defaults and bounds per container.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: dev
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 200m
      memory: 256Mi
    max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: 100m
      memory: 128Mi
```

## Resources in a pod

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Requests are used for scheduling. Exceeding a memory limit kills the container (OOMKilled). Exceeding a CPU limit throttles it.

## Common mistakes

- Setting `readOnlyRootFilesystem: true` without a writable mount for the app's temp or cache directory.
- Setting `runAsNonRoot: true` with an image whose user is root and no `runAsUser`.
- Creating a quota and then wondering why pods without requests fail.
- Setting `drop: ["ALL"]` and forgetting that the app needs `NET_BIND_SERVICE` to bind port 80.

## Quick reference

```bash
kubectl create quota <n> --hard=pods=<n>,requests.cpu=<c>
kubectl describe quota <n>
kubectl exec <pod> -- id
```
