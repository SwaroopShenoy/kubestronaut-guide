# Scheduling and Scaling

Up: [KCNA hub](../../README.md) · Domain 2 — Container Orchestration (28%) · Prev: [Runtimes, networking and interfaces](runtimes-networking-interfaces.md) · Next: [Security basics](security-basics.md)

## How a pod gets a node

1. The pod is created without a node.
2. The scheduler filters nodes that cannot run it (not enough resources, taints it does not tolerate, selectors that do not match).
3. It scores the remaining nodes.
4. It binds the pod to the best node. The kubelet there starts it.

## Placement controls

| Mechanism | Effect |
|---|---|
| nodeSelector | Pod runs only on nodes with matching labels |
| Node affinity | Richer label rules; required or preferred |
| Pod affinity / anti-affinity | Place pods near or away from other pods (for example, spread replicas) |
| Taints and tolerations | Nodes repel pods unless the pod tolerates the taint |

Taint effects: `NoSchedule` (new pods rejected), `PreferNoSchedule` (avoided if possible), `NoExecute` (existing pods evicted unless they tolerate it).

## Requests and limits

- **Request**: the amount the scheduler reserves. Drives placement.
- **Limit**: the most the container may use. Memory over the limit kills the container; CPU over the limit is throttled.
- CPU units: 1000m = 1 core. Memory: `Mi`, `Gi`.

## Scaling

| Mechanism | Scales | Trigger |
|---|---|---|
| Manual (`kubectl scale`) | Replicas | Human decision |
| Horizontal Pod Autoscaler (HPA) | Replicas | CPU, memory or custom metrics; needs metrics-server |
| Vertical Pod Autoscaler (VPA) | Requests and limits | Observed usage; usually restarts pods |
| Cluster Autoscaler | Nodes | Pending pods that cannot fit |
| KEDA | Replicas, including to zero | External events such as queue length |

Do not run HPA and VPA on the same resource metric at once.
