# RBAC Hardening

Up: [CKS hub](../../README.md) · Domain 2 — Cluster Hardening (15%) · Prev: [Ingress, TLS and node metadata](../01-cluster-setup/ingress-tls-and-node-metadata.md) · Next: [Service accounts and API access](service-accounts-and-api-access.md)

Permissions accumulate quietly. This topic shows how to find the over-broad bindings, recognise the verbs that give away the cluster, and tighten access without breaking the workloads that depend on it.

## Exam scope

**In scope:** finding over-permissioned subjects, tightening Roles and RoleBindings, recognizing dangerous verbs and resources, and verifying with `kubectl auth can-i`.

## Model

- **Role / ClusterRole** — a set of rules (apiGroups, resources, verbs).
- **RoleBinding / ClusterRoleBinding** — grants a role to subjects (User, Group, ServiceAccount).
- A RoleBinding can reference a ClusterRole; it then applies only inside its own namespace.

## Facts that are easy to get wrong

- A new ServiceAccount has **no** meaningful permissions. Discovery endpoints are open to all authenticated users (`system:discovery`, `system:basic-user`), but they cannot read pods or secrets.
- Do not claim the `default` ServiceAccount can do everything. Test it:

```bash
kubectl auth can-i --list --as=system:serviceaccount:default:default -n default
```

- The API server prevents privilege escalation: you cannot create or update a role that grants permissions you do not hold, unless you have the `escalate` verb on roles. You also need the `bind` verb to create a binding to a role with more permissions than you have.

## Auditing

```bash
# Who holds cluster-admin?
kubectl get clusterrolebindings -o json \
  | jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name + " -> " + ([.subjects[]? | "\(.kind)/\(.name)"] | join(", "))'

# Every binding that references a given Role in a namespace
kubectl get rolebindings -n production -o wide

# Everything one subject can do, across namespaces
kubectl auth can-i --list --as=alice -n production
kubectl auth can-i create pods --as=system:serviceaccount:production:app -n production
```

Expected baseline: `cluster-admin` bound only to `system:masters` (group) and system component identities. Anything else is a finding.

## Dangerous permissions to flag

| Permission | Why it matters |
|---|---|
| `*` on any of apiGroups, resources, verbs | Effectively cluster-admin |
| `create` on `pods` | Can mount host paths, run privileged pods, reach node secrets |
| `create` on `pods/exec`, `pods/attach` | Shell into any pod in scope |
| `get`/`list`/`watch` on `secrets` | Reads every credential in scope |
| `create` on `rolebindings`, `clusterrolebindings` | Grants arbitrary permissions to self |
| `escalate`, `bind` on roles | Bypasses the no-escalation rule |
| `impersonate` on users/groups/serviceaccounts | Acts as any identity |
| `get` on `nodes/proxy` | Reaches kubelet APIs |

## Tightening: a worked example

Goal: the `app` ServiceAccount in `production` can read pods and configmaps only.

```bash
# 1. Find and remove the broad binding
kubectl get rolebindings -n production -o json \
  | jq -r '.items[] | select(.subjects[]?.name=="app") | .metadata.name'
kubectl delete rolebinding app-admin -n production
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-read
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-read
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-read
subjects:
- kind: ServiceAccount
  name: app
  namespace: production
```

Verify:

```bash
kubectl auth can-i get pods   --as=system:serviceaccount:production:app -n production   # yes
kubectl auth can-i delete pods --as=system:serviceaccount:production:app -n production  # no
kubectl auth can-i get secrets --as=system:serviceaccount:production:app -n production  # no
```

## Rules of thumb

- Prefer namespaced `Role` over `ClusterRole` when the scope is one namespace.
- Bind to ServiceAccounts for workloads; bind to Users or Groups only for humans.
- Never use wildcards in production roles.
- Do not grant `pods/exec` or `secrets` read without a specific reason.
- Use `resourceNames` to restrict to one named object where the API supports it (not for `create` or `list`).

## Restoring cluster-admin safely

The default `cluster-admin` ClusterRole stays. The thing to remove is the **binding** to a human or ServiceAccount. Delete the specific binding by name:

```bash
kubectl get clusterrolebindings -o wide | grep -v system:
kubectl delete clusterrolebinding <name>
```

Do not delete and recreate `cluster-admin` bindings to `system:masters`; you can lock yourself out of the cluster.

## Common mistakes

- Verifying with a user who is also cluster-admin, so every check says "yes".
- Creating the Role in the wrong namespace (the RoleBinding must be in the same namespace as the subject's access target).
- Forgetting the subject namespace on a ServiceAccount in a RoleBinding (`namespace: production` is required).
- Assuming `get` implies `list` or `watch`. Each verb is separate.

## Quick reference

```bash
kubectl auth can-i <verb> <resource> --as=<subject> -n <ns>
kubectl auth can-i --list --as=<subject> -n <ns>
kubectl get rolebindings,clusterrolebindings -A -o wide
```
