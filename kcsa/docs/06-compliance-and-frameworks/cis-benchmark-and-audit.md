# CIS Benchmark and Audit Logging

Up: [KCSA hub](../../README.md) · Domain 6 — Compliance and Security Frameworks (10%) · Prev: [Runtime security](../05-platform-security/runtime-security.md) · Next: [Frameworks and regulations](frameworks-and-regulations.md)

## CIS Kubernetes Benchmark

A consensus set of hardening recommendations for Kubernetes, organized by component:

1. Control plane components
2. etcd
3. Control plane configuration
4. Worker nodes
5. Policies (RBAC, Pod Security, NetworkPolicy)

Checks are scored (should be met) or not scored (advisory). kube-bench is an open-source tool that runs the checks against a node and reports PASS, FAIL, WARN or INFO with remediation text.

Check IDs differ between benchmark versions; read the remediation printed for your version. The CKS hub has the hands-on fixes: [CIS benchmark and kube-bench](../../../cks/docs/01-cluster-setup/cis-benchmark-kube-bench.md).

## Audit logging

The API server records requests against an audit policy. Each rule sets a level:

| Level | Records |
|---|---|
| None | Nothing |
| Metadata | Who, verb, resource, time, result |
| Request | Metadata plus request body |
| RequestResponse | Metadata, request and response bodies |

Rules are checked top to bottom; the first match wins.

Worth logging: Secret access (including reads), RBAC changes, exec and attach into pods, creation of privileged workloads, namespace deletion.

Backends: a log file, or a webhook that forwards events to a SIEM. The old dynamic audit sink (AuditSink) was removed; use the webhook backend.

Caution: request bodies can contain secrets. Use `RequestResponse` for a narrow set of resources only.
