# Service Accounts and API Access

Up: [CKS hub](../../README.md) · Domain 2 — Cluster Hardening (15%) · Prev: [RBAC](rbac.md) · Next: [Cluster upgrades](cluster-upgrades.md)

Every pod can carry an identity, and an identity can call the API. This topic covers how to stop unneeded tokens from being mounted, keep the tokens short-lived, and restrict who can reach the API at all.

## 1. Stop auto-mounting tokens you do not need

By default every pod gets its ServiceAccount token mounted at `/var/run/secrets/kubernetes.io/serviceaccount`. A compromised app can use it to call the API.

Disable it at the ServiceAccount level and, where needed, per pod:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
  namespace: production
automountServiceAccountToken: false
---
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: production
spec:
  serviceAccountName: app
  automountServiceAccountToken: false   # pod setting wins over the SA setting
  containers:
  - name: app
    image: myapp:1.0
```

Verify:

```bash
kubectl exec -n production app -- ls /var/run/secrets/kubernetes.io/serviceaccount 2>&1
# ls: ... No such file or directory
```

If a pod needs the API, mount the token explicitly and bind it to a narrow Role.

## 2. Tokens are short-lived now

Since Kubernetes 1.24, ServiceAccounts do not get long-lived token Secrets automatically. Tokens come from the TokenRequest API and expire.

```bash
kubectl create token app -n production --duration=10m
```

Do not recreate legacy token Secrets for workloads. If you find an old `kubernetes.io/service-account-token` Secret, list what references it and delete it when unused:

```bash
kubectl get secrets -A --field-selector type=kubernetes.io/service-account-token
```

## 3. Default ServiceAccount hygiene

Namespaces get a `default` ServiceAccount that pods use unless told otherwise. Two good defaults:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: production
automountServiceAccountToken: false
```

Then give real workloads their own ServiceAccount with a minimal RoleBinding.

## 4. Restricting anonymous and unauthenticated access

The API server should not accept anonymous requests beyond what discovery needs.

```yaml
# kube-apiserver.yaml
- --anonymous-auth=false
```

Verify from outside the cluster:

```bash
curl -sk https://<api-server>:6443/api/v1/namespaces
# expect 401 or 403, not a JSON list
```

Also check that the API server is not exposed on a public address unless the environment requires it, and that `--authorization-mode` does not include `AlwaysAllow` (see [CIS benchmark](../01-cluster-setup/cis-benchmark-kube-bench.md)).

## 5. Limiting who can reach the API endpoint

Network-level restriction complements RBAC. Allow only control-plane administrators and worker nodes to reach port 6443:

```bash
sudo iptables -A INPUT -p tcp --dport 6443 -s 10.0.0.0/8 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6443 -j DROP
```

Be careful to add the accept rule before the drop rule, and to keep your own access path open before applying a drop.

## 6. Audit what the API is being asked

Pair these controls with the audit policy in [Audit logging](../06-monitoring-logging-runtime/audit-logging.md). Useful signals:

- `create` on `serviceaccounts/token` (token minting)
- requests with `user.username` starting with `system:serviceaccount:` touching secrets or RBAC
- `401` and `403` response codes (`responseStatus.code`)

## Quick reference

```bash
kubectl create token <sa> -n <ns> --duration=10m
kubectl get sa -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,AUTOMOUNT:.automountServiceAccountToken
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>
```
