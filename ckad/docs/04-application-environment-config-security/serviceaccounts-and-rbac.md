# ServiceAccounts and RBAC (Application View)

Up: [CKAD hub](../../README.md) · Domain 4 — Application Environment, Configuration and Security (25%) · Prev: [SecurityContext, quotas and limits](security-context-quotas-limits.md) · Next: [Services, NetworkPolicy and Ingress](../05-services-networking/services-networking.md)

An application that talks to the Kubernetes API needs an identity. This topic explains ServiceAccounts, the permissions they receive, and how to fix the 403 errors that appear when those permissions are missing.

The full RBAC model is in [CKA RBAC](../../../cka/docs/02-cluster-architecture-installation-config/rbac.md). For CKAD, the usual task is "the app gets a 403 from the API; fix its ServiceAccount."

## Create and use a ServiceAccount

```bash
kubectl create serviceaccount app-reader -n dev
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev
kubectl create rolebinding app-reader-binding --role=pod-reader --serviceaccount=dev:app-reader -n dev
```

Use it in a pod:

```yaml
spec:
  serviceAccountName: app-reader
  containers:
  - name: app
    image: myapp:1.0
```

Or set it on an existing Deployment:

```bash
kubectl set serviceaccount deploy/web app-reader -n dev
```

## Verify

```bash
kubectl auth can-i list pods --as=system:serviceaccount:dev:app-reader -n dev
kubectl auth can-i delete pods --as=system:serviceaccount:dev:app-reader -n dev
```

## Token handling

Pods get a projected token at `/var/run/secrets/kubernetes.io/serviceaccount/token` by default. If the app does not talk to the API, turn that off:

```yaml
spec:
  automountServiceAccountToken: false
```

To give the token a shorter lifetime, mount it yourself with `expirationSeconds`:

```yaml
volumes:
- name: token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
```

## Common mistakes

- Creating the RoleBinding with `--user` for a ServiceAccount (use `--serviceaccount=<ns>:<name>`).
- Using the wrong namespace in the subject. The subject namespace is the ServiceAccount's namespace.
- Binding a Role that lacks the verb the app needs (`list` and `watch` are separate verbs).

## Quick reference

```bash
kubectl create serviceaccount <n>
kubectl create rolebinding <n> --role=<r> --serviceaccount=<ns>:<sa>
kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa>
```

---

Prev: [SecurityContext, quotas and limits](security-context-quotas-limits.md) · Next: [Services, NetworkPolicy and Ingress](../05-services-networking/services-networking.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
