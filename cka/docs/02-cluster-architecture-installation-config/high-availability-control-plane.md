# Highly Available Control Plane

Up: [CKA README](../../README.md) · Domain 2 — Cluster Architecture, Installation and Configuration (25%) · Prev: [CRDs and operators](crds-and-operators.md)

A single control-plane node is a single point of failure. This topic explains how several control-plane nodes, a shared etcd quorum and a load balancer keep the API available when one of them fails.

Official curriculum topic covered here: "Implement and configure a highly-available control plane" and "Prepare underlying infrastructure for installing a Kubernetes cluster". For single-node install and upgrades see [kubeadm install and upgrade](kubeadm-install-and-upgrade.md).

## What high availability means

- Several control-plane nodes run kube-apiserver, kube-controller-manager and kube-scheduler.
- Only one controller manager and one scheduler are active at a time (leader election); the others stand by.
- The API server runs on every control-plane node, behind a load balancer or virtual IP that clients use.
- etcd must keep a quorum. A three-member cluster survives one failure; a five-member cluster survives two.

## etcd topology

| Topology | Layout | Trade-off |
|---|---|---|
| Stacked | etcd runs on the control-plane nodes | Simpler; losing a node loses an etcd member and an API server together |
| External | etcd runs on its own dedicated nodes | More machines; control-plane and data failures separate |

## Load balancer in front of the API server

- Clients and nodes point at one stable address (for example a load balancer DNS name or virtual IP) on port 6443.
- The load balancer health-checks each API server's `/readyz` endpoint and removes unhealthy ones.
- The control-plane certificates must include that address in their Subject Alternative Names when the cluster is created (kubeadm's control-plane endpoint setting).

## Infrastructure prerequisites

- Consistent OS and container runtime on every node.
- Swap disabled, required ports open (6443, 2379–2380, 10250, and the controller and scheduler ports on the control plane).
- Synchronized clocks (certificates and etcd depend on time).
- Stable hostnames and IP addresses.

## Joining additional control-plane nodes

With kubeadm, the join command for a control-plane node includes a certificate key and the control-plane endpoint. Generate them on an existing control-plane node when joining more members. Follow the official kubeadm high-availability guide for the exact flags for your version.

## Checking the result

```bash
kubectl get nodes -l node-role.kubernetes.io/control-plane
kubectl get pods -n kube-system -o wide
```

Confirm that each control-plane node runs the API server and that etcd reports healthy members.

## Common mistakes

- A load balancer address missing from the certificate SANs, so clients fail TLS validation.
- An even number of etcd members, which adds cost without improving fault tolerance.
- Forgetting that stacked etcd on two nodes cannot survive one failure.
