# Runtime Sandboxes (RuntimeClass)

Up: [CKS hub](../../README.md) · Domain 4 — Minimize Microservice Vulnerabilities (20%) · Prev: [Secrets and encryption at rest](secrets-and-encryption-at-rest.md) · Next: [Pod-to-pod encryption (Cilium, Istio)](pod-to-pod-encryption.md)

Containers share the host kernel, which is convenient and risky. This topic covers sandboxed runtimes such as gVisor and Kata, how to select them with RuntimeClass, and how to isolate tenants on a shared cluster.

## Why sandboxes exist

Containers share the host kernel. A kernel exploit from a container affects the node. Sandboxed runtimes add a boundary:

| Runtime | Boundary | Trade-off |
|---|---|---|
| `runc` (default) | Shared kernel, namespaces and cgroups | Fastest, least isolation |
| gVisor (`runsc`) | User-space kernel intercepts syscalls | Strong syscall isolation; some syscalls and performance costs |
| Kata Containers | Lightweight VM per pod | Strongest isolation; highest overhead; needs virtualization |

## Setup is on the node

RuntimeClass only selects a handler. The handler must be configured in the container runtime on each node that should run that class. For containerd, that means a runtime entry in `/etc/containerd/config.toml` pointing at the installed binary (for gVisor, `runsc`):

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"
```

Restart containerd after editing. The exact plugin path varies by containerd version; check the runtime's install docs.

## The RuntimeClass object

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc          # must match the handler name in the runtime config
```

Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: untrusted
  namespace: sandbox
spec:
  runtimeClassName: gvisor
  containers:
  - name: app
    image: untrusted-image:1.0
```

Verify the pod uses the sandbox:

```bash
kubectl get pod untrusted -n sandbox -o jsonpath='{.spec.runtimeClassName}{"\n"}'
kubectl exec untrusted -n sandbox -- dmesg 2>&1 | head -3
# gVisor reports its own kernel banner; runc shows the host kernel
```

## Scheduling

If only some nodes have the handler, a pod with `runtimeClassName` must land on those nodes. Add `scheduling` to the RuntimeClass:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
scheduling:
  nodeSelector:
    sandbox.example.com/gvisor: "true"
```

## Pitfalls

- `handler` name does not match the runtime configuration. Pods stay pending or fail with a "no runtime handler" style error.
- Assuming RuntimeClass alone isolates the cluster. It isolates the pods that opt in; you still need PSA, NetworkPolicy, and RBAC.
- Using `runtimeClassName` on a node that lacks the handler, and not adding scheduling constraints.

## Quick reference

```bash
kubectl get runtimeclass
kubectl get pod <pod> -o jsonpath='{.spec.runtimeClassName}'
```

## Isolation for multi-tenancy

The curriculum asks for isolation techniques such as multi-tenancy and sandboxed containers. For shared clusters, layer the controls: a namespace per tenant, RBAC scoped to that namespace, ResourceQuota and LimitRange, default-deny NetworkPolicy, a Pod Security level on each namespace, and dedicated node pools or sandboxed runtimes for untrusted workloads. Namespaces alone do not stop a determined attacker from reaching the node kernel.

---

Prev: [Secrets and encryption at rest](secrets-and-encryption-at-rest.md) · Next: [Pod-to-pod encryption (Cilium, Istio)](pod-to-pod-encryption.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
