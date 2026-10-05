# The 4Cs and Shared Responsibility

Up: [KCSA hub](../../README.md) · Domain 1 — Overview of Cloud Native Security (14%) · Next: [Isolation and workload security](isolation-and-workload-security.md)

Security is not one wall but a set of layers. This topic introduces the four layers of cloud native security and the line between what the provider and what you are responsible for.

## The 4Cs

Security is layered. Each layer relies on the layers beneath it.

| Layer | Examples of controls |
|---|---|
| Cloud (or datacenter) | Network isolation, IAM, physical security, provider compliance |
| Cluster | API server authentication and authorization, RBAC, admission, etcd encryption, audit logs, network policies, node hardening |
| Container | Minimal and scanned images, non-root user, read-only filesystem, dropped capabilities, runtime detection |
| Code | Secure coding, dependency scanning, secret management, input validation |

A weakness in a lower layer undermines every layer above it. A perfect RBAC setup does not help if a container runs as root with a privileged flag.

Exam tip: the order matters when a question asks "from the foundation up". The diagram usually runs Cloud at the base and Code at the top, but check how the question names them.

## Shared responsibility

| Provider usually owns | You usually own |
|---|---|
| Physical data centers, hardware, hypervisor | Operating system configuration and patching |
| Managed control-plane availability (on managed Kubernetes) | Workloads, images, RBAC, network policies |
| Underlying network fabric | Secrets, data classification, application code |

On managed Kubernetes services the split moves; the control plane is partly theirs. You still own what runs in your namespaces.

## Cloud provider and infrastructure security

The curriculum lists this as its own topic. The cloud layer underpins everything above it:

- **Identity and access:** least-privilege roles for people and machines; no long-lived administrator keys; multi-factor authentication for human access.
- **Network:** private subnets for nodes, restricted security groups, private or allow-listed API server endpoints, and no public access to etcd or the kubelet.
- **Instance metadata:** restrict access from pods to the metadata endpoint, and prefer the hardened metadata version where the provider offers one.
- **Data:** encrypted disks and snapshots, with keys managed by the provider's key service.
- **Logging:** provider audit logs enabled and stored where an attacker on a node cannot erase them.
- **Images and nodes:** hardened, minimal node images that are patched or replaced regularly.
- **Managed services:** know which parts the provider operates (often the control plane) and which remain yours (workloads, RBAC, network policy).

## Defense in depth

Assume one control will fail. Stack several:

- Prevention: image scanning, admission control, RBAC.
- Detection: audit logs, runtime alerts (Falco), monitoring.
- Response: isolate the pod, revoke credentials, roll the image back.
- Recovery: backups tested by restoring them.

## Zero trust in Kubernetes

Verify every request (authenticate every identity), grant least privilege, and do not assume the network is safe. In practice: strong identities for workloads, RBAC, NetworkPolicy default-deny, and mutual TLS where a mesh provides it.

---

Next: [Isolation and workload security](isolation-and-workload-security.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
