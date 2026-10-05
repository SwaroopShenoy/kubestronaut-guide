# Authentication, Isolation and Segmentation

Up: [KCSA README](../../README.md) · Domain 3 — Kubernetes Security Fundamentals (22%) · Prev: [Secrets](secrets.md) · Next: [Threat model and attack paths](../04-kubernetes-threat-model/threat-model-and-attack-paths.md)

Official curriculum topics covered here: authentication, isolation and segmentation. Other fundamentals topics (Pod Security Standards and Admission, secrets, audit logging, NetworkPolicy) are in their own docs in this folder.

## Authentication

The API server accepts several identity methods:

| Method | Typical use | Security note |
|---|---|---|
| X.509 client certificates | Admins, components | Protect private keys; rotate; short lifetimes |
| ServiceAccount tokens | Pods | Projected, time-limited tokens are the default; avoid long-lived token Secrets |
| OpenID Connect (OIDC) | Human users through an identity provider | Recommended for people; central offboarding |
| Webhook token authentication | Custom identity systems | The webhook must be trusted and available |
| Authenticating proxy | Front-end SSO proxy | The proxy must be the only path to the API |
| Static token and password files | Legacy | Avoid; deprecated |

Authentication answers "who is this?". Authorization (RBAC) answers "what may they do?". A strong login does not help if RBAC grants `*`.

Anonymous authentication should be disabled on the API server and kubelet unless there is a specific, reviewed need.

## Isolation and segmentation

- **Namespace isolation:** separate teams and environments by namespace, with RBAC, quotas, and NetworkPolicy. Namespaces alone are not hard isolation.
- **Network segmentation:** default-deny NetworkPolicy per namespace; allow only the flows the application needs.
- **Node segmentation:** taints and dedicated node pools for sensitive or untrusted workloads.
- **Control plane segmentation:** the API server, etcd and nodes on separate network paths where possible.
