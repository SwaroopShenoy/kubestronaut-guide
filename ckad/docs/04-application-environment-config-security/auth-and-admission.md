# Authentication, Authorization and Admission Control

Up: [CKAD README](../../README.md) · Domain 4 — Application Environment, Configuration and Security (25%) · Prev: [ServiceAccounts and RBAC](serviceaccounts-and-rbac.md) · Next: [Services, NetworkPolicy and Ingress](../05-services-networking/services-networking.md)

Every request an application or a developer makes passes three checks before the API server stores anything. Knowing which check rejected a request tells you where to look.

Official curriculum topic: "Understand authentication, authorization and admission control".

## The three stages

| Stage | Question | Typical failure |
|---|---|---|
| Authentication | Who is making the request? | `401 Unauthorized` |
| Authorization | Is that identity allowed to do this? | `403 Forbidden`, "cannot list resource" |
| Admission | Is the object acceptable, and should it be changed? | Request rejected with a policy message |

The order is fixed: authentication, then authorization, then admission, then the object is stored.

## Authentication

Methods the API server may accept include client certificates, bearer tokens (including ServiceAccount tokens), OpenID Connect, and webhook token authentication. Inside a pod, an application authenticates with its ServiceAccount token, which is mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token` by default.

```bash
kubectl auth whoami
kubectl create token <serviceaccount> -n <namespace>
```

## Authorization

RBAC is the usual mode. A subject needs a Role or ClusterRole that grants the verb on the resource, and a binding that attaches it. Details are in [ServiceAccounts and RBAC](serviceaccounts-and-rbac.md).

```bash
kubectl auth can-i list pods --as=system:serviceaccount:dev:app -n dev
kubectl auth can-i --list --as=system:serviceaccount:dev:app -n dev
```

The error message names the identity and the missing permission:

```
pods is forbidden: User "system:serviceaccount:dev:app" cannot list resource "pods" in API group "" in the namespace "dev"
```

Use it to write the exact Role rule that is missing.

## Admission control

Admission controllers run after authorization. Some change the request (mutating), some accept or reject it (validating).

Controllers a developer meets most often:

| Controller | Effect |
|---|---|
| ResourceQuota | Rejects objects that would exceed the namespace quota |
| LimitRanger | Applies default requests and limits and rejects values outside the allowed range |
| ServiceAccount | Assigns the default ServiceAccount and mounts its token |
| Pod Security Admission | Rejects pods that violate the namespace's Pod Security level |
| MutatingAdmissionWebhook and ValidatingAdmissionWebhook | Call external services that mutate or validate objects |
| ValidatingAdmissionPolicy | Applies validation written in CEL, with no external service |

Examples of what admission looks like from the outside:

```
Error from server (Forbidden): pods "web" is forbidden: exceeded quota: compute, requested: pods=1, used: pods=10, limited: pods=10
```

```
Error from server (Forbidden): pods "web" is forbidden: violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false
```

To see what a namespace enforces:

```bash
kubectl describe quota -n <namespace>
kubectl describe limitrange -n <namespace>
kubectl get namespace <namespace> --show-labels
```

Pod Security labels have the form `pod-security.kubernetes.io/enforce=restricted`.

## Telling the stages apart

| What you see | Stage | Fix |
|---|---|---|
| `Unauthorized`, certificate or token errors | Authentication | Use a valid token or kubeconfig |
| `forbidden: User ... cannot <verb> resource` | Authorization | Add the Role rule and binding |
| `forbidden: exceeded quota`, `violates PodSecurity` | Admission | Change the object or the namespace policy |

## Common mistakes

- Adding RBAC rules to fix a quota or Pod Security rejection; those come from admission, not authorization.
- Reading a 403 as an authentication problem. A 401 means the identity is unknown; a 403 means it is known but not allowed.
- Forgetting that a mutating webhook may have changed a field you did not set.

---

Prev: [ServiceAccounts and RBAC](serviceaccounts-and-rbac.md) · Next: [Services, NetworkPolicy and Ingress](../05-services-networking/services-networking.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
