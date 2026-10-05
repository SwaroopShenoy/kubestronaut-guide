# Services and DNS

Up: [CKA hub](../../README.md) · Domain 3 — Services and Networking (20%) · Next: [NetworkPolicy](network-policy.md)

Pods come and go, and their addresses change with them. Services give them a stable name and address, and cluster DNS makes those names usable. This chapter explains how the two work together.

## Service types

| Type | Reach | Notes |
|---|---|---|
| ClusterIP (default) | Inside the cluster | Stable virtual IP |
| NodePort | Node IP + static port (30000–32767) | Port opened on every node |
| LoadBalancer | External IP from a cloud or load-balancer provider | Builds on NodePort |
| ExternalName | DNS CNAME to an external name | No proxying |
| Headless (`clusterIP: None`) | Returns pod IPs directly | Used by StatefulSets |

## Create Services quickly

```bash
kubectl expose deployment web --port=80 --target-port=8080
kubectl expose deployment web --type=NodePort --port=80 --target-port=8080
kubectl create service clusterip web --tcp=80:8080
kubectl create service externalname db --external-name=db.example.com
```

Verify with the endpoints, not only the Service:

```bash
kubectl get svc web
kubectl get endpoints web
```

## Service YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: web          # must match pod labels exactly
  ports:
  - port: 80          # Service port
    targetPort: 8080  # container port
    nodePort: 30080   # optional, must be in the NodePort range
```

## DNS

CoreDNS serves cluster names:

- Service: `<svc>.<namespace>.svc.cluster.local`
- Short names work from the same namespace: `<svc>`; cross-namespace: `<svc>.<namespace>`
- Pod: `<ip-with-dashes>.<namespace>.pod.cluster.local`

Test DNS:

```bash
kubectl run dns-test --rm -it --image=busybox:1.36 --restart=Never -- nslookup web.default.svc.cluster.local
```

CoreDNS config: ConfigMap `coredns` in `kube-system`.

## Headless services and StatefulSets

A StatefulSet with a headless Service gives each pod a stable name: `<pod>.<svc>.<namespace>.svc.cluster.local`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  clusterIP: None
  selector:
    app: db
  ports:
  - port: 5432
```

## Common mistakes

- Selector typo or label mismatch: the Service exists, the endpoints are empty.
- `targetPort` set to the Service port rather than the container port.
- Expecting a NodePort to work on a node that has no kube-proxy running.
- Testing with `curl` from the node instead of from inside the cluster.

## Quick reference

```bash
kubectl expose deploy <name> --port=<p> --target-port=<tp> [--type=NodePort]
kubectl get endpoints <svc>
kubectl run t --rm -it --image=busybox:1.36 --restart=Never -- nslookup <svc>
```
