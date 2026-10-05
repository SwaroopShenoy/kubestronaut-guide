# Runtimes, Networking and Interfaces

Up: [KCNA hub](../../README.md) · Domain 2 — Container Orchestration (28%) · Prev: [Services, storage and configuration](../01-kubernetes-fundamentals/services-storage-config.md) · Next: [Scheduling and scaling](scheduling-and-scaling.md)

Kubernetes does not run containers itself, and it does not build the network. It relies on pluggable interfaces for each job. This topic explains the runtime, network and storage interfaces.

## Container runtimes

A container runtime starts and manages containers. Kubernetes talks to it through the **Container Runtime Interface (CRI)**.

- **containerd**: widely used general runtime; Docker uses it underneath.
- **CRI-O**: built specifically for Kubernetes; lightweight.

Docker Engine is not used directly by Kubernetes. Its shim was removed in Kubernetes 1.24.

## Three plug-in interfaces

| Interface | Plugs in | Examples |
|---|---|---|
| CRI (Container Runtime Interface) | Container runtime | containerd, CRI-O |
| CNI (Container Network Interface) | Pod networking | Calico, Cilium, Flannel, Weave |
| CSI (Container Storage Interface) | Storage backends | Cloud disk drivers, NFS, Ceph drivers |

Each is a standard so components can be swapped without changing Kubernetes itself.

## Pod networking model

- Every pod gets its own IP.
- Pods can reach other pods by IP without NAT.
- Containers in one pod share that IP.
- The CNI plugin provides this. Without a CNI, nodes stay NotReady.

A CNI plugin that supports NetworkPolicy (Calico, Cilium) enforces policy rules. Plugins without enforcement accept NetworkPolicy objects but do nothing with them.

## Service discovery

- DNS (CoreDNS) resolves Service names.
- Environment variables are also injected for Services that exist when a pod starts.

## Container images

- An image is a layered, read-only template. A tag names a version; a digest identifies exact content.
- Prefer pinned tags or digests. `latest` changes without warning.

---

Prev: [Services, storage and configuration](../01-kubernetes-fundamentals/services-storage-config.md) · Next: [Scheduling and scaling](scheduling-and-scaling.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
