# Gateway API and TLS

Up: [CKS README](../../README.md) · Domain 1 — Cluster Setup (15%) · Prev: [Ingress TLS, node metadata, binary verification](ingress-tls-and-node-metadata.md) · Next: [etcd hardening](etcd-hardening.md)

Ingress is the established way to bring HTTP traffic into a cluster, and Gateway API is its successor. This topic covers where each stands, and how to terminate TLS and limit who can attach routes with Gateway API.

## Where this fits in CKS

The CKS curriculum says "Properly set up Ingress objects with TLS". It does not name Gateway API, and the CKS allowed-resources list names the NGINX Ingress Controller guide and not the Gateway API site, although `kubernetes.io/docs` has Gateway API concept pages. So treat Ingress with TLS as the examined item, and this topic as supporting knowledge. The [CKA](../../../cka/docs/03-services-networking/ingress-and-gateway-api.md) curriculum does name Gateway API.

## Status of Ingress

- The Kubernetes documentation says the Ingress API is stable but frozen: it is generally available, no further changes will be made to it, and the project has no plans to remove it. It recommends Gateway instead of Ingress. "Soft deprecated" describes this fairly, but the project's own word is *frozen*, not deprecated.
- The NGINX ingress controller (`kubernetes/ingress-nginx`, the controller behind the exam's allowed NGINX guide) was retired on 2026-03-24. It receives no more releases, bug fixes or security patches. Ingress as an API is unaffected; this is about one controller. Other controllers continue.

## The model

| Object | Owned by | Purpose |
|---|---|---|
| GatewayClass | The infrastructure provider | Names the controller that implements gateways |
| Gateway | Cluster or platform operators | Defines listeners: ports, protocols, hostnames and TLS |
| HTTPRoute (and other routes) | Application teams | Maps requests to Services and attaches to a Gateway |

Splitting these roles is a security feature: the team that owns the certificate and the listener is not the team that publishes routes.

Gateway API is a set of custom resources, not part of core Kubernetes. A cluster needs the CRDs installed and a controller that implements them:

```bash
kubectl api-resources | grep gateway.networking.k8s.io
kubectl get gatewayclass
```

## Terminating TLS on a listener

The certificate lives in a Secret of type `kubernetes.io/tls`, the same as for Ingress:

```bash
kubectl create secret tls api-tls --cert=tls.crt --key=tls.key -n gateway-infra
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: gateway-infra
spec:
  gatewayClassName: <class-from-the-cluster>
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: api.example.com
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        group: ""
        name: api-tls
```

`mode: Terminate` is the default for HTTPS listeners. A `Passthrough` mode exists for TLS routes, where the Gateway forwards the encrypted stream and does not read it.

## Certificates in another namespace

A Gateway may reference a Secret in another namespace only if the Secret's namespace allows it with a ReferenceGrant:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway-certs
  namespace: certs
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: gateway-infra
  to:
  - group: ""
    kind: Secret
```

The Gateway then names the namespace in `certificateRefs` (`namespace: certs`). Without the grant, the listener reports the reference as not permitted. Check the installed API version with `kubectl api-resources | grep referencegrant`.

## Controlling who can attach routes

By default a listener accepts routes from its own namespace. Widen or narrow this with `allowedRoutes`:

```yaml
listeners:
- name: https
  protocol: HTTPS
  port: 443
  tls:
    certificateRefs:
    - name: api-tls
  allowedRoutes:
    namespaces:
      from: Selector
      selector:
        matchLabels:
          gateway-access: "true"
```

| `from` | Routes accepted from |
|---|---|
| `Same` | The Gateway's namespace (default) |
| `Selector` | Namespaces whose labels match |
| `All` | Every namespace (avoid on shared clusters) |

## A route, and redirecting HTTP to HTTPS

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api
  namespace: shop
spec:
  parentRefs:
  - name: web-gateway
    namespace: gateway-infra
    sectionName: https
  hostnames:
  - api.example.com
  rules:
  - backendRefs:
    - name: api
      port: 8080
```

A redirect is a route on a plain HTTP listener with a `RequestRedirect` filter:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-redirect
  namespace: gateway-infra
spec:
  parentRefs:
  - name: web-gateway
    sectionName: http
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
```

This needs a listener named `http` on port 80 in the same Gateway.

## Checking it works

```bash
kubectl get gateway -n gateway-infra
kubectl describe gateway web-gateway -n gateway-infra
kubectl get httproute -A
kubectl describe httproute api -n shop
```

Read the status conditions. On the Gateway, `Accepted` and `Programmed` should be `True`, and each listener has its own `ResolvedRefs` condition: `False` means the certificate Secret is missing, has the wrong type, or is not permitted by a ReferenceGrant. On the route, `Accepted` shows whether the listener's `allowedRoutes` let it attach.

From outside, confirm the certificate that is served:

```bash
curl -vk --resolve api.example.com:443:<gateway-address> https://api.example.com/ 2>&1 | grep -E 'subject|issuer|expire'
```

## From Ingress to Gateway

| Ingress | Gateway API |
|---|---|
| `ingressClassName` | `gatewayClassName` on the Gateway |
| `spec.tls[].secretName` | `listeners[].tls.certificateRefs` |
| `spec.rules[].host` | `hostname` on the listener and `hostnames` on the route |
| Paths to a backend Service | `rules[].matches` and `backendRefs` on an HTTPRoute |
| Controller-specific annotations | Typed fields and filters in the API |
| One object per application | Gateway owned by operators, routes owned by teams |

The `ingress2gateway` tool can convert existing Ingress manifests into a starting point; review its output before applying it.

## Common mistakes

- Creating a Gateway with a `gatewayClassName` that does not exist, so nothing programs it.
- Putting the certificate Secret in a namespace the Gateway cannot read, with no ReferenceGrant.
- A Secret that is not of type `kubernetes.io/tls`, or that lacks `tls.crt` or `tls.key`.
- Setting `allowedRoutes.namespaces.from: All` on a shared Gateway.
- Forgetting `sectionName` on the route, so it attaches to a listener you did not intend.
- Assuming the retirement of the NGINX ingress controller removes the Ingress API. It does not.

---

Prev: [Ingress TLS, node metadata, binary verification](ingress-tls-and-node-metadata.md) · Next: [etcd hardening](etcd-hardening.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
