# Containerization and Administration Basics

Up: [KCNA README](../../README.md) · Domain 1 — Kubernetes Fundamentals (44%) · Prev: [Services, storage and configuration](services-storage-config.md) · Next: [Runtimes, networking and interfaces](../02-container-orchestration/runtimes-networking-interfaces.md)

Two topics in the official curriculum sit under Kubernetes Fundamentals and are easy to skip: containerization and administration. This topic gives the concepts for both.

Official curriculum topics: "Containerization" and "Administration".

## What a container is

A container is an ordinary process on the host that the kernel isolates:

- **Namespaces** limit what the process can see: its own process tree, network, mounts and hostname.
- **Control groups (cgroups)** limit what it can use: CPU, memory and I/O.
- The process shares the host kernel with every other container on the node.

This is why containers start faster and are lighter than virtual machines, and also why a kernel flaw can affect every container on the node.

## Images

- An image is a read-only, layered package: a filesystem plus metadata about how to run it.
- Each layer is the difference from the layer below it, so layers can be shared and cached.
- The Open Container Initiative (OCI) defines the image and runtime formats, which is why images built with one tool run under another.
- A tag (`nginx:1.27`) is a movable label. A digest (`nginx@sha256:...`) identifies exact content.
- A registry stores and serves images. Pulling needs credentials for private registries.

A Dockerfile is a recipe that builds an image: a base image, then steps that add files and set the command to run. Multi-stage builds keep build tools out of the final image.

## Container lifecycle

1. The image is pulled to the node (if not already there).
2. The runtime creates the container with its namespaces and limits.
3. The main process starts and runs.
4. It exits, or is stopped; the kubelet restarts it according to the pod's restart policy.

A container runs one main process. Logs are the process's standard output and standard error.

## Administering a cluster

Administration is the everyday work of operating Kubernetes.

| Task | Concept |
|---|---|
| Talk to the cluster | `kubectl` uses a kubeconfig file holding clusters, users and contexts |
| Separate workloads | Namespaces divide a cluster into named partitions |
| Organise objects | Labels identify and select; annotations store extra metadata |
| Control access | RBAC grants permissions to users, groups and ServiceAccounts |
| Build a cluster | Tools such as kubeadm install one; managed services run the control plane for you |
| Update a cluster | Upgrade one minor version at a time, control plane before nodes |
| Maintain nodes | Cordon stops new pods landing on a node; drain evicts the existing ones |
| Keep state safe | Back up etcd, which holds all cluster state |

Common `kubectl` verbs, by purpose:

| Purpose | Verbs |
|---|---|
| Create | `create`, `apply`, `run`, `expose` |
| Read | `get`, `describe`, `logs` |
| Change | `edit`, `scale`, `set image`, `rollout` |
| Remove | `delete` |
| Context | `config use-context`, `config set-context` |

`apply` is declarative and can update an existing object; `create` fails if the object already exists.

## Versions

Kubernetes releases a new minor version about three times a year. Clusters should be kept within the supported versions, and node components must not be newer than the control plane.

## Common distinctions

- A container is a process with isolation; a virtual machine is a full operating system on virtualised hardware.
- A tag can move; a digest cannot.
- `cordon` stops scheduling; `drain` also evicts running pods.

---

Prev: [Services, storage and configuration](services-storage-config.md) · Next: [Runtimes, networking and interfaces](../02-container-orchestration/runtimes-networking-interfaces.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
