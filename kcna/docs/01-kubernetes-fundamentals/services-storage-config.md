# Services, Storage and Configuration

Up: [KCNA hub](../../README.md) · Domain 1 — Kubernetes Fundamentals (44%) · Prev: [API objects and workloads](api-objects-and-workloads.md) · Next: [Containerization and administration](containerization-and-administration.md)

Applications need to be reached, need somewhere to keep data, and need configuration. This topic covers Services, volumes and storage objects, and ConfigMaps and Secrets.

## Services

A Service gives a stable address to a set of pods chosen by label.

| Type | Exposure |
|---|---|
| ClusterIP (default) | Inside the cluster only |
| NodePort | Each node's IP on a port in 30000–32767 |
| LoadBalancer | An external load balancer from the cloud provider (builds on NodePort) |
| ExternalName | DNS alias to an external name; no proxying |

Cluster DNS resolves Service names: `<service>.<namespace>.svc.cluster.local`.

Ingress is a separate object for HTTP routing from outside the cluster; it needs an ingress controller. Gateway API is the newer, more expressive alternative.

## Volumes and storage

- **Volume**: storage attached to a pod's containers. `emptyDir` lives as long as the pod; `hostPath` mounts a node path (avoid in production).
- **PersistentVolume (PV)**: a piece of storage provisioned by an administrator or dynamically.
- **PersistentVolumeClaim (PVC)**: a user's request for storage.
- **StorageClass**: a template for dynamic provisioning through a CSI driver.

Access modes: ReadWriteOnce (one node, read-write), ReadOnlyMany (many nodes, read-only), ReadWriteMany (many nodes, read-write, needs a backend that supports it).

## ConfigMaps and Secrets

- **ConfigMap**: non-sensitive configuration as key-value pairs, consumed as environment variables or files.
- **Secret**: sensitive data. Stored base64-encoded, which is **encoding, not encryption**. Encryption at rest must be configured separately.

Secret types include Opaque (generic), `kubernetes.io/tls`, and `kubernetes.io/dockerconfigjson` for registry credentials.

## Common distinctions

- Service (stable network address) vs Pod IP (changes when the pod is replaced).
- ConfigMap (plain) vs Secret (sensitive, but not encrypted by default).
- PV (the storage) vs PVC (the request for it).

---

Prev: [API objects and workloads](api-objects-and-workloads.md) · Next: [Containerization and administration](containerization-and-administration.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
