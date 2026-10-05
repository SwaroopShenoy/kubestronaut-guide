# RBAC and ServiceAccounts

Up: [KCSA hub](../../README.md) · Domain 3 — Kubernetes Security Fundamentals (22%) · Prev: [etcd and node security](../02-kubernetes-cluster-component-security/etcd-and-node-security.md) · Next: [Pod security and NetworkPolicy](pod-security-and-networkpolicy.md)

Permissions are how Kubernetes decides what an identity may do. This topic covers roles, bindings, built-in roles and the identities that pods carry.

For hands-on RBAC work see the [CKA RBAC doc](../../../cka/docs/02-cluster-architecture-installation-config/rbac.md); for auditing see the [CKS RBAC doc](../../../cks/docs/02-cluster-hardening/rbac.md).

## RBAC objects

| Object | Scope | Purpose |
|---|---|---|
| Role | Namespace | Permissions in one namespace |
| ClusterRole | Cluster | Permissions cluster-wide, or reusable |
| RoleBinding | Namespace | Grants a role to subjects in one namespace |
| ClusterRoleBinding | Cluster | Grants a ClusterRole to subjects cluster-wide |

Subjects: User, Group, ServiceAccount. Verbs: get, list, watch, create, update, patch, delete, deletecollection.

RBAC is additive. There are no deny rules, so the sum of all bindings is the effective permission.

## Least privilege

- Prefer namespaced Roles.
- Avoid wildcards (`*`) in verbs, resources and apiGroups.
- Bind groups or ServiceAccounts rather than many individual users.
- Review bindings to `cluster-admin`; few identities need it.

## Built-in ClusterRoles

- `view`: read-only in a namespace (no secrets).
- `edit`: read and write most objects in a namespace, but cannot change RBAC.
- `admin`: full control of a namespace, including RBAC within it.
- `cluster-admin`: full control of the cluster.

## ServiceAccounts

- Identity for pods. Each namespace has a `default` ServiceAccount.
- Pods receive a projected token by default. Turn this off when the app does not call the API (`automountServiceAccountToken: false`).
- Tokens are time-limited and bound to the pod, which is safer than the long-lived token Secrets of older versions.
