# Network Policy

Up: [CKS hub](../../CKS_2026_Complete_Crash_Course.md) · Domain 1 — Cluster Setup (10%) · Next: [CIS benchmark](cis-benchmark-kube-bench.md)

## Exam scope

**In scope:** writing ingress and egress policies, default-deny patterns, namespace and pod selectors, DNS egress, debugging a policy that blocks traffic.

**Prerequisite knowledge:** NetworkPolicy is only enforced by a CNI that supports it (Calico, Cilium, Weave, and similar). Flannel alone and kindnet do not enforce it, so a policy can "work" in YAML and do nothing. Check the lab's CNI before trusting results.

## Mental model

- No policy selects a pod → all traffic to and from it is allowed.
- Once any policy selects a pod for a direction (Ingress or Egress), that direction is default-deny for that pod, plus whatever the union of matching policies allows.
- Policies are additive. There is no "deny" rule. Allow rules are unioned.
- `policyTypes` controls which directions the policy governs. If `Egress` is listed and there are no egress rules, all egress is denied for the selected pods.

| `policyTypes` | Effect on selected pods |
|---|---|
| `[Ingress]` | Inbound denied except allowed rules; outbound unaffected |
| `[Egress]` | Outbound denied except allowed rules; inbound unaffected |
| `[Ingress, Egress]` | Both directions denied except allowed rules |
| omitted | Inferred: `Ingress`, plus `Egress` only if an egress block exists |

## Selector semantics (the most common exam trap)

```yaml
# OR: two list items. frontend pods from ANY namespace, OR any pod in prod namespaces
ingress:
- from:
  - podSelector:
      matchLabels: {app: frontend}
  - namespaceSelector:
      matchLabels: {environment: prod}

# AND: one list item with both selectors. frontend pods that live in a prod namespace
ingress:
- from:
  - podSelector:
      matchLabels: {app: frontend}
    namespaceSelector:
      matchLabels: {environment: prod}
```

The difference is a single dash. Get this wrong and you silently widen access.

### Namespace labels

Since Kubernetes 1.21 every namespace carries `kubernetes.io/metadata.name=<name>` automatically. Prefer it over custom labels, which you would otherwise have to add yourself:

```yaml
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: production
```

## Ports

- Omitting `ports` allows traffic on **any** port. Always specify ports.
- `endPort` (GA in 1.25) defines a range: `port: 32000` + `endPort: 32768`.
- Named ports (`port: http`) resolve against the container port name.

## Default-deny baseline

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

## Allowing DNS (always required when egress is restricted)

Restrict DNS to the cluster DNS pods rather than all of kube-system or all namespaces:

```yaml
egress:
- to:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
  ports:
  - protocol: UDP
    port: 53
  - protocol: TCP
    port: 53
```

Without this, `curl https://api.example.com` hangs at name resolution while `curl https://<ip>` works. That symptom is the tell.

## Worked example: zero-trust for a three-tier namespace

Requirements: frontend → backend on 8080; backend → database on 5432; backend → external HTTPS (IP range 203.0.113.0/24); everything else denied.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress
  namespace: production
spec:
  podSelector:
    matchLabels: {app: backend}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels: {app: frontend}
    ports:
    - protocol: TCP
      port: 8080
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress
  namespace: production
spec:
  podSelector:
    matchLabels: {app: backend}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels: {k8s-app: kube-dns}
    ports:
    - {protocol: UDP, port: 53}
    - {protocol: TCP, port: 53}
  - to:
    - podSelector:
        matchLabels: {app: database}
    ports:
    - {protocol: TCP, port: 5432}
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24
    ports:
    - {protocol: TCP, port: 443}
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-ingress
  namespace: production
spec:
  podSelector:
    matchLabels: {app: database}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels: {app: backend}
    ports:
    - {protocol: TCP, port: 5432}
```

Note that the database also needs its own ingress policy. A policy on backend egress does not open the database's ingress.

## Cross-namespace access

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-namespace
  namespace: backend
spec:
  podSelector:
    matchLabels: {app: api}
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
    ports:
    - {protocol: TCP, port: 8080}
```

## Blocking cloud metadata from pods

`ipBlock` with `except` lets you allow the internet and exclude the metadata endpoint:

```yaml
egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0
      except:
      - 169.254.169.254/32
```

Combine with DNS and the specific allows above. See [Ingress, TLS and node metadata](ingress-tls-and-node-metadata.md).

## Debugging workflow

```bash
# 1. Which policies select this pod?
kubectl get netpol -n production
kubectl get netpol -n production -o yaml | less

# 2. Do the labels actually match?
kubectl get pods -n production --show-labels

# 3. Test from a throwaway pod (pod-level connectivity)
kubectl run tester -n production --rm -it --image=busybox:1.36 -- sh
#   wget -qO- -T 3 http://backend:8080
#   nslookup api.example.com

# 4. Namespace label present?
kubectl get ns --show-labels
```

Diagnosis table:

| Symptom | Likely cause |
|---|---|
| Service hostname does not resolve | Missing UDP/TCP 53 egress |
| Resolves, connection times out | Missing ingress on target or egress on source |
| Works from one pod, not another | Selector typo or wrong label |
| Namespace selector matches nothing | Namespace label missing |
| Policy has no effect at all | CNI does not enforce NetworkPolicy |

## Common mistakes

- Adding `Egress` to `policyTypes` with no egress rules → pod cannot reach anything.
- Forgetting DNS after adding default-deny egress.
- Separating `podSelector` and `namespaceSelector` into two list items when you meant AND.
- Writing `ports` at the wrong level (it belongs inside the same list item as `from`/`to`).

## Practice

1. Create default-deny ingress and egress in a namespace; confirm a busybox pod cannot reach a web pod.
2. Add the DNS rule only; confirm name resolution works but HTTP does not.
3. Add the minimal allows for the three-tier example; test every allowed and denied path.
4. Rewrite a policy from OR semantics to AND semantics and observe the difference.

## Quick reference

```bash
kubectl get netpol -A
kubectl describe netpol <name> -n <ns>
kubectl label ns <ns> <key>=<value>        # only if you cannot use kubernetes.io/metadata.name
```
