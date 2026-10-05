# Pod-to-Pod Encryption

Up: [CKS README](../../README.md) · Domain 2 — Minimize Microservice Vulnerabilities (20%) · Prev: [Runtime sandboxes](runtime-sandboxes.md) · Next: [Base images and Dockerfiles](../05-supply-chain-security/base-images-and-dockerfiles.md)

Network policy decides which pods may talk; it does not hide what they say. This topic covers encrypting pod-to-pod traffic with Cilium and with Istio's mutual TLS.

Official curriculum topic: "Implement Pod-to-Pod encryption (Cilium, Istio)".

## Why encrypt pod-to-pod traffic

NetworkPolicy controls which pods may talk. It does not encrypt the traffic. An attacker who can sniff the node network can read unencrypted traffic between pods, and can impersonate a service unless each side proves its identity.

Two approaches appear in the curriculum: encryption in the network layer (Cilium) and mutual TLS at the service layer (Istio).

## Option 1: Encryption in the CNI (Cilium)

Cilium can encrypt traffic between nodes using WireGuard or IPsec, transparently to pods. Enable it through the Cilium installation (Helm values or the cilium CLI), then verify:

```bash
cilium status
cilium encrypt status        # reports whether encryption is enabled
```

Limits: this encrypts node-to-node traffic and proves node identity, not per-workload identity.

## Option 2: Mutual TLS with a service mesh (Istio)

Istio injects a sidecar proxy next to each pod. The proxies set up mTLS with certificates issued to each workload identity.

Enable sidecar injection for a namespace:

```bash
kubectl label namespace shop istio-injection=enabled
```

Require strict mTLS for the namespace:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: shop
spec:
  mtls:
    mode: STRICT
```

Check it:

```bash
kubectl get peerauthentication -n shop
istioctl x describe pod <pod> -n shop        # shows mTLS status for the pod
```

Pods in a namespace without injection cannot talk to strict mTLS workloads. Plan migration carefully.

## Choosing

| Need | Use |
|---|---|
| Encrypt all node-to-node traffic with no app change | Cilium encryption |
| Workload identity and authorization between services | Istio mTLS and authorization policy |
| Both | Layer them; they solve different problems |

## Common mistakes

- Assuming NetworkPolicy encrypts traffic. It does not.
- Setting STRICT mode before every client has a sidecar, which breaks traffic.
- Forgetting that Cilium encryption is per node pair, not per pod identity.
- Skipping the check: confirm encryption and mTLS are active with the tool's status command.
