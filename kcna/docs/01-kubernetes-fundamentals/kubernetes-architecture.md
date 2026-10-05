# Kubernetes Architecture

Up: [KCNA hub](../../KCNA_Crash_Course.md) · Domain 1 — Kubernetes Fundamentals (44%) · Next: [API objects and workloads](api-objects-and-workloads.md)

## Control plane

| Component | Role |
|---|---|
| kube-apiserver | The front end for the API. Every component talks to it. Authenticates, authorizes, admits and persists requests |
| etcd | Consistent key-value store holding all cluster state. The only stateful component of the control plane |
| kube-scheduler | Chooses a node for each unscheduled pod. It does not start pods |
| kube-controller-manager | Runs control loops (Deployment, ReplicaSet, Node, and others) that move actual state toward desired state |
| cloud-controller-manager | Connects the cluster to cloud APIs (load balancers, nodes, routes) |

## Nodes

| Component | Role |
|---|---|
| kubelet | Agent on each node. Makes sure the containers in its pods are running |
| kube-proxy | Maintains network rules so Service IPs reach pods |
| Container runtime | Runs containers (containerd, CRI-O) |

## The request path

1. A user runs `kubectl apply`.
2. The API server authenticates and authorizes the request, runs admission, and writes the object to etcd.
3. A controller sees the new object and creates related objects (for example, a Deployment creates a ReplicaSet, which creates Pods).
4. The scheduler assigns each unscheduled Pod to a node.
5. The kubelet on that node sees the assignment and asks the runtime to start the containers.
6. The kubelet reports status back to the API server.

Scheduler and kubelet each have one job. Neither does the other's.

## Desired state and reconciliation

You declare desired state in the API. Controllers compare it with actual state and act on the difference, continuously. This is the core model behind almost every Kubernetes behavior exam questions describe.

## Things that are easy to confuse

- kube-proxy does not forward every packet in the default mode. It programs rules (iptables or IPVS) that the kernel applies.
- The scheduler decides placement; it does not run containers.
- etcd is the only component that stores state. Other components are stateless and can be restarted.
- Docker Engine is not a runtime Kubernetes uses directly. Its dockershim was removed in Kubernetes 1.24; runtimes now speak CRI.
