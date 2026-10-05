# Services, NetworkPolicy and Ingress

Up: [CKAD hub](../../README.md) · Domain 5 — Services and Networking (20%) · Prev: [ServiceAccounts and RBAC](../04-application-environment-config-security/serviceaccounts-and-rbac.md)

Full treatments are in the CKA docs. This page lists the CKAD checks, with links.

- Service types, endpoints and DNS: [Services and DNS](../../../cka/docs/03-services-networking/services-and-dns.md)
- NetworkPolicy: [NetworkPolicy](../../../cka/docs/03-services-networking/network-policy.md)
- Ingress and Gateway API: [Ingress and Gateway API](../../../cka/docs/03-services-networking/ingress-and-gateway-api.md)

## The CKAD checks

1. **Service has endpoints.** `kubectl get endpoints <svc>` lists pod IPs. Empty means the selector does not match pod labels, or pods are not Ready.
2. **Port chain is correct.** Service `port` → Service `targetPort` → container port. A wrong middle value gives connection refused.
3. **Pod reaches the Service by name.** `kubectl run t --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://<svc>:<port>`.
4. **NetworkPolicy allows the path.** Ingress on the target, egress on the source, and DNS (UDP and TCP 53) for any restricted egress.

## Minimal egress policy that keeps DNS working

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-egress
spec:
  podSelector:
    matchLabels: {app: myapp}
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
    - ipBlock:
        cidr: 203.0.113.0/24
    ports:
    - {protocol: TCP, port: 443}
```

## Common debugging sequence

```bash
kubectl get svc <svc> -o wide
kubectl get endpoints <svc>
kubectl get pods -l <selector> --show-labels
kubectl get netpol -n <ns>
kubectl describe netpol <name> -n <ns>
```

## Common mistakes

- Selector typo: the Service exists, endpoints are empty.
- `targetPort` set to the Service port.
- NetworkPolicy egress rule without DNS: names do not resolve.
- Assuming a ClusterIP is reachable from outside the cluster.

## Quick reference

```bash
kubectl get endpoints <svc>
kubectl expose deploy <n> --port=<p> --target-port=<tp>
kubectl get netpol -A
```
