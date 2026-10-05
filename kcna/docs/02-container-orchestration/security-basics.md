# Security Basics

Up: [KCNA hub](../../README.md) · Domain 2 — Container Orchestration (28%) · Prev: [Scheduling and scaling](scheduling-and-scaling.md) · Next: [CI/CD and GitOps](../03-cloud-native-application-delivery/cicd-and-gitops.md)

KCNA covers security at the concept level. The deeper treatment is in the [KCSA hub](../../../kcsa/README.md) and the [CKS hub](../../../cks/README.md).

## Authentication and authorization

- **Authentication** answers "who are you?" Methods include client certificates, bearer tokens, ServiceAccount tokens, and OIDC.
- **Authorization** answers "what can you do?" RBAC is the standard mode.
- **Admission control** runs after both and can validate or change requests.

## RBAC

| Object | Scope |
|---|---|
| Role | Permissions in one namespace |
| ClusterRole | Permissions cluster-wide, or reusable across namespaces |
| RoleBinding | Grants a Role or ClusterRole to subjects in one namespace |
| ClusterRoleBinding | Grants a ClusterRole to subjects cluster-wide |

Subjects: User, Group, ServiceAccount. RBAC is additive; there are no deny rules.

ServiceAccounts identify pods, not people.

## Pod Security Standards

Three levels, applied per namespace by labels:

- **Privileged**: no restrictions
- **Baseline**: blocks known privilege escalations
- **Restricted**: hardened; requires non-root, no privilege escalation, dropped capabilities, and a seccomp profile

PodSecurityPolicy was removed in Kubernetes 1.25 and replaced by Pod Security Admission.

## SecurityContext

Pod- and container-level settings: `runAsUser`, `runAsNonRoot`, `fsGroup`, `allowPrivilegeEscalation`, `readOnlyRootFilesystem`, and capabilities.

## NetworkPolicy

Controls pod traffic by label. No policy means all traffic allowed; once a policy selects a pod, traffic is limited to what the policies allow. Enforcement requires a CNI that supports it.

## Secrets

Base64 is encoding, not protection. Enable encryption at rest and limit who can read Secrets with RBAC.
