# Ingress TLS, Node Metadata and Binary Verification

Up: [CKS hub](../../README.md) · Domain 1 — Cluster Setup (15%) · Prev: [CIS benchmark and kube-bench](cis-benchmark-kube-bench.md) · Next: [etcd hardening](etcd-hardening.md)

Three things define a cluster's outer edge: the encryption on incoming traffic, the metadata endpoint that cloud instances expose, and the binaries the cluster runs. This topic covers how to secure all three.

## 1. TLS on Ingress

Ingress terminates TLS with a secret of type `kubernetes.io/tls`, which must contain `tls.crt` and `tls.key`.

```bash
# Self-signed certificate for a lab
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout tls.key -out tls.crt -subj "/CN=api.example.com"

kubectl create secret tls api-tls --cert=tls.crt --key=tls.key -n production
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  namespace: production
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 8080      # plain HTTP inside the cluster, TLS ends at the ingress
```

Check it:

```bash
kubectl get ingress api -n production
kubectl describe ingress api -n production
```

Pitfalls:

- The backend port is the Service port, not 443 unless the Service listens on 443. TLS terminates at the ingress; the backend usually speaks plain HTTP.
- Ingress TLS does not protect pod-to-pod traffic. For that you need mTLS or a sidecar, which is outside the core scope.
- Ingress controllers vary. If a lab uses a specific controller, its `ingressClassName` must match.

### NGINX Ingress annotations for TLS

The NGINX Ingress Controller documentation is on the exam's allowed-resources list. Settings are mostly annotations on the Ingress:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

| Annotation | Effect |
|---|---|
| `ssl-redirect` | Redirects HTTP to HTTPS when TLS is configured (on by default when a `tls` section exists) |
| `force-ssl-redirect` | Redirects to HTTPS even when TLS is terminated before the controller |
| `backend-protocol: "HTTPS"` | Talks to the backend Service over TLS |

Verify the certificate the controller serves:

```bash
curl -vk --resolve api.example.com:443:<ingress-ip> https://api.example.com/ 2>&1 | grep -E 'subject|issuer|expire'
```

If the controller serves its default "Kubernetes Ingress Controller Fake Certificate", the `secretName` is wrong or the Secret is in a different namespace from the Ingress.

## 2. Protecting cloud instance metadata

Cloud instances expose a metadata endpoint at `169.254.169.254`. A compromised pod that reaches it can often obtain node credentials.

Two layers:

**Pod layer (works without node access):** a NetworkPolicy egress rule that excludes the metadata IP.

```yaml
egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0
      except:
      - 169.254.169.254/32
```

**Node layer:** pod traffic is forwarded, so rules on the `OUTPUT` chain do not catch it. Use `FORWARD` (or `DOCKER-USER` if Docker is the runtime):

```bash
sudo iptables -I FORWARD -d 169.254.169.254/32 -j DROP
```

Verify from a pod:

```bash
kubectl run metadata-test --rm -it --image=curlimages/curl:8.10.1 -- \
  curl -m 3 http://169.254.169.254/latest/meta-data/ || echo "blocked"
```

On AWS, IMDSv2 with a hop limit of 1 also stops containers from reaching the endpoint through the node's network namespace.

## 3. Verifying Kubernetes binaries

Download the per-binary checksum from the same release path and compare:

```bash
VER=v1.34.0            # use the version actually installed: kubelet --version
ARCH=amd64
curl -LO "https://dl.k8s.io/release/${VER}/bin/linux/${ARCH}/kubelet"
curl -LO "https://dl.k8s.io/release/${VER}/bin/linux/${ARCH}/kubelet.sha256"
echo "$(cat kubelet.sha256)  kubelet" | sha256sum --check
# kubelet: OK
```

If the check fails, do not run the binary. Repeat for `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kubectl`, and `kubeadm` as needed.

The checksum file name and path are section of the release layout; if the URL 404s, look up the current layout in the Kubernetes install docs rather than guessing.

## Quick reference

```bash
kubectl create secret tls <name> --cert=tls.crt --key=tls.key -n <ns>
sudo iptables -I FORWARD -d 169.254.169.254/32 -j DROP
echo "$(cat X.sha256)  X" | sha256sum --check
```

---

Prev: [CIS benchmark and kube-bench](cis-benchmark-kube-bench.md) · Next: [etcd hardening](etcd-hardening.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
