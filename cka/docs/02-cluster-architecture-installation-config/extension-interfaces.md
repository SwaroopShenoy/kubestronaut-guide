# Extension Interfaces: CNI, CSI and CRI

Up: [CKA README](../../README.md) · Domain 2 — Cluster Architecture, Installation and Configuration (25%) · Prev: [Highly available control plane](high-availability-control-plane.md) · Next: [Services and DNS](../03-services-networking/services-and-dns.md)

Kubernetes delegates networking, storage and container execution to plug-ins through three standard interfaces. Knowing where each plug-in lives lets you tell which part is broken.

Official curriculum topic: "Understand extension interfaces (CNI, CSI, CRI, etc.)".

## CRI: the container runtime interface

The kubelet talks to a runtime (containerd or CRI-O) through CRI. `crictl` is the CRI command line client and works even when the API server is down.

```bash
sudo crictl ps -a                    # containers, including exited ones
sudo crictl pods                     # pod sandboxes
sudo crictl images
sudo crictl logs <container-id>
sudo crictl inspect <container-id>
```

`crictl` reads its endpoint from `/etc/crictl.yaml`:

```yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
```

Runtime configuration lives in `/etc/containerd/config.toml`. The kubelet and the runtime must use the same cgroup driver, normally `systemd`; a mismatch makes pods fail or restart.

```bash
sudo systemctl status containerd --no-pager
grep -n SystemdCgroup /etc/containerd/config.toml
grep -n cgroupDriver /var/lib/kubelet/config.yaml
```

## CNI: the container network interface

A CNI plug-in gives each pod an IP address and connects pods across nodes. Without one, nodes stay `NotReady`.

```bash
ls /etc/cni/net.d/                   # active network configuration
ls /opt/cni/bin/                     # plug-in binaries
kubectl get pods -n kube-system -o wide | grep -iE 'calico|cilium|flannel|weave'
kubectl describe node <node> | grep -i networkunavailable
```

Common plug-ins: Calico, Cilium, Flannel, Weave Net. Whether NetworkPolicy is enforced depends on the plug-in; Flannel alone does not enforce it.

When installing a cluster with kubeadm, apply the plug-in's manifest after `kubeadm init`, and make sure the pod network range passed to `kubeadm init` matches what the plug-in expects.

## CSI: the container storage interface

A CSI driver lets Kubernetes create, attach and mount volumes from a storage system. The driver runs as pods in the cluster and registers with each node.

```bash
kubectl get csidrivers
kubectl get csinodes
kubectl get storageclass
kubectl get pods -n kube-system | grep -i csi
```

A StorageClass names the driver in its `provisioner` field. A PVC that stays `Pending` often points to a missing or unhealthy driver. See [Storage](../05-storage/storage.md).

## Other extension points

- **CRDs and operators** extend the API itself: [CRDs and operators](crds-and-operators.md).
- **Admission webhooks** extend request validation and mutation.
- **Device plug-ins** expose hardware such as GPUs to pods.
- **Scheduler plug-ins** extend placement logic.

## Where to look first

| Symptom | Interface to check |
|---|---|
| Node `NotReady`, pods have no IP | CNI |
| Containers will not start, image pulls fail on one node | CRI (runtime and its configuration) |
| PVC `Pending`, volume will not attach | CSI driver and StorageClass |

## Common mistakes

- Running `docker ps` on a node that uses containerd; use `crictl`.
- Mismatched cgroup drivers between kubelet and containerd.
- Installing a CNI plug-in with a pod network range that differs from the one given to kubeadm.
- Assuming NetworkPolicy works with a plug-in that does not enforce it.

---

Prev: [Highly available control plane](high-availability-control-plane.md) · Next: [Services and DNS](../03-services-networking/services-and-dns.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
