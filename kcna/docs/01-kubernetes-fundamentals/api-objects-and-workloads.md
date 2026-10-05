# API Objects and Workloads

Up: [KCNA hub](../../KCNA_Crash_Course.md) · Domain 1 — Kubernetes Fundamentals (44%) · Prev: [Kubernetes architecture](kubernetes-architecture.md) · Next: [Services, storage and configuration](services-storage-config.md)

## Every object has the same shape

```yaml
apiVersion: apps/v1     # group and version
kind: Deployment        # type
metadata:               # name, namespace, labels, annotations
  name: web
spec:                   # desired state
  replicas: 3
status:                 # actual state, written by the system
```

## Pod

- Smallest deployable unit. Usually one container; sometimes a helper alongside it.
- Containers in a pod share a network namespace (one IP, `localhost` between containers) and can share volumes.
- Pods are ephemeral. A replacement gets a new name and IP.

Pod phases: Pending, Running, Succeeded, Failed, Unknown. A container state (Waiting, Running, Terminated) is a separate concept from the pod phase.

## Workload resources

| Resource | Use when | Key behavior |
|---|---|---|
| ReplicaSet | You need N identical pods | Keeps N running. Usually created by a Deployment, not directly |
| Deployment | Stateless apps | Manages ReplicaSets; rolling updates and rollbacks |
| StatefulSet | Stateful apps needing stable identity | Ordered creation; stable names (`web-0`, `web-1`) and per-pod storage |
| DaemonSet | One pod per node | Logs, monitoring and storage agents |
| Job | Run to completion | Retries until a set number of successful completions |
| CronJob | Run on a schedule | Creates Jobs from a cron expression |

## Labels, selectors and annotations

- **Labels** identify objects and let selectors match them. Keep them short and meaningful.
- **Selectors** match labels. Deployments, Services and NetworkPolicies all use them.
- **Annotations** hold non-identifying metadata (tool config, descriptions). Selectors cannot match on them.

## Namespaces

Virtual partitions inside a cluster. Default namespaces: `default`, `kube-system`, `kube-public`, `kube-node-lease`. Names are unique within a namespace; many resources are namespace-scoped, while nodes and PersistentVolumes are cluster-scoped.

## Common exam distinctions

- Deployment vs StatefulSet: stable identity and storage vs interchangeable replicas.
- Job vs CronJob: one-off vs scheduled.
- DaemonSet vs Deployment: one per node vs a chosen replica count.
