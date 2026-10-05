# Secrets

Up: [KCSA hub](../../KCSA_Crash_Course.md) · Domain 3 — Kubernetes Security Fundamentals (22%) · Prev: [Pod security and NetworkPolicy](pod-security-and-networkpolicy.md) · Next: [Threat model and attack paths](../04-kubernetes-threat-model/threat-model-and-attack-paths.md)

## What a Kubernetes Secret is

A Secret stores sensitive values in the cluster. Its data is base64-encoded, which is an encoding anyone with read access can reverse. Secrets are not encrypted by default.

## Protecting Secrets

| Control | What it protects against |
|---|---|
| Encryption at rest (EncryptionConfiguration) | Reading Secrets from etcd or a backup |
| RBAC (`get`, `list`, `watch` on secrets are sensitive) | Users and workloads reading Secrets they should not |
| Audit logging of Secret access | Undetected reads |
| Volume mounts over environment variables | Values leaking through `env` output, process listings and logs |
| Read-only mounts with restrictive file modes | Modification and broad file access |
| Immutable Secrets | Accidental or malicious edits |
| External secret managers (Vault, cloud secret stores, External Secrets Operator) | Storing the source of truth outside the cluster; rotation and central audit |

## Environment variables vs volumes

Environment variables are visible through `kubectl describe` output, to child processes, and sometimes in crash dumps. Volume mounts are files that can be permissioned and updated without a restart. Prefer volumes for sensitive values.

## Secret types

- `Opaque`: generic key-value data.
- `kubernetes.io/tls`: certificate and key for Ingress TLS.
- `kubernetes.io/dockerconfigjson`: registry credentials.
- `kubernetes.io/service-account-token`: legacy long-lived ServiceAccount tokens; avoid creating these.

## Practice questions

- Is a default Kubernetes Secret encrypted? (No; it is base64-encoded)
- Which control reduces what a user can read from etcd directly? (Encryption at rest, plus restricting etcd access)
- Why prefer a volume mount to an environment variable? (Less exposure through process and describe output)
- Which secret type should you avoid creating for new workloads? (Legacy long-lived ServiceAccount token Secrets)
