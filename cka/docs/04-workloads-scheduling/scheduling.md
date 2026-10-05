# Scheduling

Up: [CKA hub](../../README.md) · Domain 4 — Workloads and Scheduling (15%) · Prev: [Workload types](workload-types.md) · Next: [Helm and Kustomize](helm-and-kustomize.md)

Each pod must land on a node that has room for it and is allowed to host it. This topic explains how the scheduler decides, and how labels, taints, affinity and resource requests shape those decisions.

## How scheduling works

1. A pod is created without `nodeName`.
2. The scheduler filters nodes (resources, taints, selectors, affinity).
3. It scores the remaining nodes and binds the pod.
4. The kubelet on that node starts the containers.

A pod stuck `Pending` has not passed step 2 for any node. `kubectl describe pod` says why.

## Node selector

```bash
kubectl label node worker-1 disktype=ssd
```

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

## Node affinity

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: [ssd]
```

`requiredDuringScheduling` is a hard rule. `preferredDuringScheduling` is a weight.

## Pod affinity and anti-affinity

Spread replicas across nodes:

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web
        topologyKey: kubernetes.io/hostname
```

## Taints and tolerations

```bash
kubectl taint nodes worker-1 dedicated=gpu:NoSchedule
kubectl taint nodes worker-1 dedicated=gpu:NoSchedule-     # remove
```

Effects: `NoSchedule` (new pods rejected), `PreferNoSchedule` (soft), `NoExecute` (evicts running pods that do not tolerate it).

```yaml
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule
```

A toleration allows scheduling onto a tainted node. It does not attract the pod there; combine with nodeSelector or affinity for that.

## Resources

Requests drive scheduling. Limits cap usage.

```yaml
resources:
  requests:
    cpu: 250m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 128Mi
```

A pod whose requests exceed every node's allocatable capacity stays Pending.

## Check why a pod is not scheduled

```bash
kubectl describe pod <pod> | grep -A8 Events
kubectl describe node <node> | grep -A8 -E 'Taints|Allocated resources'
kubectl get nodes --show-labels
```

## Common mistakes

- Using `nodeName` directly. It bypasses the scheduler and is rarely what a task wants.
- Applying a toleration and expecting the pod to land on the tainted node.
- Forgetting to label the node before using a nodeSelector.

## Quick reference

```bash
kubectl label node <n> <k>=<v>
kubectl taint node <n> <k>=<v>:NoSchedule
kubectl describe node <n> | grep -A6 Taints
```

---

Prev: [Workload types](workload-types.md) · Next: [Helm and Kustomize](helm-and-kustomize.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
