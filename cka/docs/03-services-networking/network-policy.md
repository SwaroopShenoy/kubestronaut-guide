# NetworkPolicy

Up: [CKA hub](../../README.md) · Domain 3 — Services and Networking (20%) · Prev: [Services and DNS](services-and-dns.md) · Next: [Ingress and Gateway API](ingress-and-gateway-api.md)

By default, every pod in a cluster can reach every other pod. This chapter shows how NetworkPolicy turns that open network into one where only the flows you name are allowed.

The full security treatment (selector semantics, debugging, zero-trust patterns) is in [CKS NetworkPolicy](../../../cks/docs/01-cluster-setup/network-policy.md). This page covers what the CKA task usually asks for.

## Behavior

- No policy selects a pod → all traffic allowed.
- Once a policy selects a pod for a direction, that direction is default-deny except for what the policies allow.
- Policies are additive. There are no deny rules.
- A policy only works if the CNI enforces NetworkPolicy (Calico, Cilium and similar do; some basic setups do not).

## Default deny

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: prod
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

## Allow ingress from one app

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: prod
spec:
  podSelector:
    matchLabels: {app: backend}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels: {app: frontend}
    ports:
    - {protocol: TCP, port: 8080}
```

## Egress with DNS

Any egress policy must allow DNS, or names will not resolve:

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
  - {protocol: UDP, port: 53}
  - {protocol: TCP, port: 53}
- to:
  - podSelector:
      matchLabels: {app: database}
  ports:
  - {protocol: TCP, port: 5432}
```

Namespace selectors use the automatic label `kubernetes.io/metadata.name`.

## Test

```bash
kubectl get netpol -n prod
kubectl describe netpol allow-frontend-to-backend -n prod
kubectl run t -n prod --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 3 http://backend:8080
```

## Common mistakes

- Adding `Egress` to `policyTypes` with no egress rules: the pod loses all outbound traffic.
- Forgetting DNS when restricting egress.
- Writing `podSelector` and `namespaceSelector` in separate list items when you meant both to match (that is OR, not AND).
- Forgetting `ports`: omitting them allows every port.

## Quick reference

```bash
kubectl get netpol -A
kubectl describe netpol <name> -n <ns>
kubectl get pods --show-labels -n <ns>
```
