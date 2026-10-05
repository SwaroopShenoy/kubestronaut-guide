# Control Plane Security

Up: [KCSA hub](../../README.md) · Domain 2 — Kubernetes Cluster Component Security (22%) · Prev: [Isolation and workload security](../01-overview-cloud-native-security/isolation-and-workload-security.md) · Next: [etcd and node security](etcd-and-node-security.md)

The control plane decides what happens in a cluster, so it is the most valuable thing to protect. This topic follows a request through the API server and explains how to secure each step.

## API request path

Every request to the API server passes through:

1. **Authentication**: who is the caller? Methods include X.509 client certificates, ServiceAccount tokens, OpenID Connect (OIDC), and authenticating proxies. Static token files are deprecated.
2. **Authorization**: may this identity do this action? RBAC is the standard mode; Node and Webhook modes exist too. ABAC is deprecated.
3. **Admission control**: mutating admission can change the request; validating admission can accept or reject it. Examples: Pod Security Admission, ResourceQuota, NodeRestriction, ValidatingAdmissionPolicy.
4. The object is persisted to etcd.

The order matters: authentication, then authorization, then admission.

## Hardening settings the benchmark asks for

| Setting | Why |
|---|---|
| `--anonymous-auth=false` | Unauthenticated requests should not reach the API |
| `--authorization-mode` without `AlwaysAllow` | Every request must be authorized (use `Node,RBAC`) |
| `--enable-admission-plugins` including `NodeRestriction` | Kubelets may only modify their own node and pods |
| Audit logging enabled | Records who did what |
| `--profiling=false` on control-plane components | Profiling endpoints expose internals |
| TLS between components | Prevents interception and impersonation |

The old `--insecure-port` flag was removed from the API server; older guidance that sets it to 0 refers to a flag that no longer exists.

## Controller manager and scheduler

- Each uses its own ServiceAccount credentials (or a kubeconfig) to talk to the API server.
- They do not talk to etcd directly; only the API server does.
- Bind them to localhost or private interfaces where possible.

## Audit logging

The API server records requests according to an audit policy. Levels, from least to most detail: `None`, `Metadata`, `Request`, `RequestResponse`. Logs answer who, what, when and the result. Log secret reads and RBAC changes at a useful level; log full request bodies only where needed, since they may contain secrets.

---

Prev: [Isolation and workload security](../01-overview-cloud-native-security/isolation-and-workload-security.md) · Next: [etcd and node security](etcd-and-node-security.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
