# Pod Security and NetworkPolicy

Up: [KCSA hub](../../KCSA_Crash_Course.md) · Domain 3 — Kubernetes Security Fundamentals (22%) · Prev: [RBAC and ServiceAccounts](rbac-and-serviceaccounts.md) · Next: [Secrets](secrets.md)

## Pod Security Standards

Three levels, applied per namespace with labels (`pod-security.kubernetes.io/enforce`, `audit`, `warn`):

| Level | Purpose |
|---|---|
| Privileged | No restrictions; for trusted system workloads |
| Baseline | Blocks known privilege escalations (privileged containers, host namespaces, hostPath, risky capabilities) |
| Restricted | Hardened: non-root, no privilege escalation, drop ALL capabilities, seccomp profile set, restricted volume types |

PodSecurityPolicy was removed in Kubernetes 1.25. Pod Security Admission replaces it.

## SecurityContext

Pod and container settings that control the process:

- `runAsNonRoot`, `runAsUser`, `runAsGroup`, `fsGroup`
- `allowPrivilegeEscalation: false`
- `readOnlyRootFilesystem: true`
- `capabilities.drop: ["ALL"]`
- `seccompProfile.type: RuntimeDefault`

Capabilities to avoid granting: `CAP_SYS_ADMIN` (close to root), `CAP_NET_ADMIN`, `CAP_SYS_PTRACE`.

## Host namespaces and volumes

Dangerous settings that Baseline blocks: `hostNetwork`, `hostPID`, `hostIPC`, `hostPath` volumes, privileged containers. Each lets a pod see or change the node.

## NetworkPolicy

- With no policy selecting a pod, all traffic is allowed.
- Once a policy selects a pod, traffic is limited to allowed rules for that direction.
- Enforcement depends on the CNI plugin (Calico and Cilium enforce; some plugins do not).
- Default-deny first, then allow specific paths. Remember DNS if egress is restricted.

Selectors: `podSelector`, `namespaceSelector`, `ipBlock`.

## Practice questions

- Which Pod Security level is the most restrictive? (Restricted)
- What happens to a pod with no NetworkPolicy selecting it? (All traffic is allowed)
- Name one setting that Baseline blocks and why it matters. (Privileged containers: they can access host resources)
- Which replaced PodSecurityPolicy? (Pod Security Admission)
