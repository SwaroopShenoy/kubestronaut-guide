# Service Mesh, PKI, Connectivity and Observability

Up: [KCSA README](../../README.md) · Domain 5 — Platform Security (16%) · Prev: [Runtime security](runtime-security.md) · Next: [CIS benchmark and audit logging](../06-compliance-and-frameworks/cis-benchmark-and-audit.md)

Trust in a cluster is built from certificates, and traffic between services can be encrypted and authorised. This topic covers service meshes, the certificate hierarchy behind the cluster, and observability as a security tool.

Official curriculum topics covered here: service mesh, PKI, connectivity and observability. Supply chain, image repository and admission control are in their own docs in this folder.

## Service mesh

- A mesh (Istio, Linkerd, and others) adds sidecar or node proxies that handle service-to-service traffic.
- Security value: mutual TLS between workloads, identity-based authorization policies, and traffic telemetry.
- Cost: more moving sections, extra latency and resource use. A mesh is not a substitute for NetworkPolicy or RBAC.

## PKI

- Kubernetes depends on a certificate authority (CA) hierarchy: a cluster CA signs API server, kubelet and client certificates; etcd has its own CA.
- Protect CA private keys most strictly. Anyone holding the cluster CA key can mint any identity.
- Rotate certificates before they expire. Expired certificates break the control plane.
- Check expiry with `kubeadm certs check-expiration` or `openssl x509 -noout -enddate`.

## Connectivity

- Use TLS for every hop that carries credentials or data: client to API server, control plane to etcd, ingress to services, and workload to workload where practical.
- Ingress terminates TLS; traffic behind it still needs protection from the cluster network.

## Observability

- Audit logs record API activity. Runtime alerts record suspicious behavior on nodes and in containers.
- Metrics and logs help detect abnormal behavior (for example, a spike in 403 responses or unexpected outbound connections).
- Logs must be shipped off the node. A node compromise can otherwise erase its own evidence.

---

Prev: [Runtime security](runtime-security.md) · Next: [CIS benchmark and audit logging](../06-compliance-and-frameworks/cis-benchmark-and-audit.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
