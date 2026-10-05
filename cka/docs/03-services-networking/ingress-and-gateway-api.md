# Ingress and Gateway API

Up: [CKA hub](../../CKA_2026_Complete_Crash_Course.md) · Domain 3 — Services and Networking (20%) · Prev: [NetworkPolicy](network-policy.md) · Next: [Deployments and scaling](../04-workloads-scheduling/deployments-and-scaling.md)

## Ingress

Ingress routes HTTP(S) from outside the cluster to Services. It needs an ingress controller installed in the cluster; the Ingress object alone does nothing.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx        # must match an installed IngressClass
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

Create it imperatively for speed:

```bash
kubectl create ingress web --rule="web.example.com/=web:80" --class=nginx
```

Check:

```bash
kubectl get ingress
kubectl describe ingress web
kubectl get ingressclass
```

Common failures: wrong `ingressClassName`, backend Service name or port wrong, no controller running.

## Gateway API

Gateway API is a separate set of CRDs (not built into Kubernetes) that is more expressive than Ingress. Check whether the cluster has it:

```bash
kubectl api-resources | grep gateway.networking.k8s.io
```

Core resources:

- **GatewayClass**: which controller implements gateways.
- **Gateway**: listeners (port, protocol) using a GatewayClass.
- **HTTPRoute**: path or host rules attaching to a Gateway, sending traffic to Services.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
spec:
  gatewayClassName: <class-from-the-cluster>
  listeners:
  - name: http
    protocol: HTTP
    port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: web-gateway
  hostnames: ["web.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api
      port: 8080
```

Check:

```bash
kubectl get gateway,httproute
kubectl describe httproute web-route
```

Look at the route's status conditions: `Accepted` and `ResolvedRefs` show whether the parent and backends were found.

## Which to use

If the task names an Ingress, use Ingress. If it names a Gateway or HTTPRoute, use Gateway API. Do not mix them for the same host.

## Common mistakes

- Using `gatewayClassName` that does not exist in the cluster.
- Forgetting `parentRefs` on an HTTPRoute.
- Wrong backend port in Ingress or HTTPRoute.

## Quick reference

```bash
kubectl get ingress,ingressclass
kubectl get gateway,httproute
kubectl describe httproute <name>
```
