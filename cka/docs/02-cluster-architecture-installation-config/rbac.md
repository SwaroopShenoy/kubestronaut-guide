# RBAC

Up: [CKA hub](../../CKA_2026_Complete_Crash_Course.md) · Domain 2 — Cluster Architecture (25%) · Next: [kubeadm install and upgrade](kubeadm-install-and-upgrade.md)

For the security view of RBAC (auditing wildcards and dangerous verbs), see [CKS RBAC](../../../cks/docs/02-cluster-hardening/rbac.md).

## Model

- **Role / ClusterRole**: rules (apiGroups, resources, verbs).
- **RoleBinding / ClusterRoleBinding**: grants a role to subjects (User, Group, ServiceAccount).
- A RoleBinding can point at a ClusterRole; access is then limited to its namespace.

Check whether a resource is namespaced:

```bash
kubectl api-resources --namespaced=true | grep -i pods
kubectl api-resources --namespaced=false | grep -i node
```

## Create a Role and RoleBinding

```bash
kubectl create role developer --verb=get,list,watch,create,delete --resource=pods -n dev
kubectl create rolebinding developer-binding --role=developer --user=jane -n dev
kubectl create rolebinding sa-reader --role=developer --serviceaccount=dev:builder -n dev
```

`--serviceaccount` takes `<namespace>:<name>`.

## Cluster-wide access

```bash
kubectl create clusterrole pod-reader --verb=get,list,watch --resource=pods
kubectl create clusterrolebinding read-pods --clusterrole=pod-reader --user=jane
```

Use a ClusterRole with a RoleBinding when you want to reuse one set of rules in several namespaces.

## Verify

```bash
kubectl auth can-i create deployments --as=jane -n dev       # yes/no
kubectl auth can-i delete pods --as=jane -n dev
kubectl auth can-i --list --as=system:serviceaccount:dev:builder -n dev
kubectl auth whoami                                          # which identity kubectl is using
```

`--as` impersonation requires your own identity to have impersonate rights; on the exam it works for admin users.

## Testing as a real user

Create a client certificate signed by the cluster CA, then add it to a kubeconfig. See [certificates and kubeconfig](certificates-and-kubeconfig.md).

## Rules that trip people up

- A Role only grants verbs on the listed resources. `get` does not imply `list` or `watch`.
- Subresources need their own resource entry: `pods/log`, `pods/exec`, `deployments/scale`.
- RBAC is additive. There are no deny rules.
- The ServiceAccount binding needs the namespace of the ServiceAccount, not the namespace of the Role.

## Common mistakes

- Creating the RoleBinding in the wrong namespace.
- Forgetting `--serviceaccount=<ns>:<name>` and using `--user` for a ServiceAccount.
- Testing with a user that already has cluster-admin, so every check passes.

## Practice

1. Create a Role that can read pods and logs in `dev`; bind it to a ServiceAccount; prove it with `auth can-i`.
2. Create a ClusterRole and bind it cluster-wide to a user; prove it works in two namespaces.
3. Find which ClusterRole grants the `edit` permission in your cluster with `kubectl describe clusterrole edit`.

## Quick reference

```bash
kubectl create role <name> --verb=<v1,v2> --resource=<r> -n <ns>
kubectl create rolebinding <name> --role=<role> --user=<u> -n <ns>
kubectl auth can-i <verb> <resource> --as=<subject> -n <ns>
```
