# kubeadm Install and Upgrade

Up: [CKA hub](../../README.md) · Domain 2 — Cluster Architecture (25%) · Prev: [RBAC](rbac.md) · Next: [Certificates and kubeconfig](certificates-and-kubeconfig.md)

kubeadm is the standard tool for building and changing a Kubernetes cluster by hand. This topic walks through installing a control plane, joining workers, and upgrading in the order that keeps the cluster safe.

## Install a control plane

Prerequisites on every node: container runtime (containerd) running, kubelet, kubeadm and kubectl installed at matching versions, swap off.

```bash
sudo swapoff -a
sudo kubeadm init \
  --apiserver-advertise-address=<control-plane-ip> \
  --pod-network-cidr=<cidr-required-by-your-CNI>
```

Set up kubectl for your user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install a CNI plugin. The exam gives you the plugin and its manifest or instructions; apply exactly what the task says. Nodes stay NotReady until the CNI is running.

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

Join a worker. `kubeadm init` prints the join command; regenerate it if the token expired:

```bash
kubeadm token create --print-join-command
# run the printed command as root on the worker
```

## Upgrade a cluster

Rules:

- One minor version at a time.
- Control plane first, then workers.
- Drain each node before upgrading its kubelet, uncordon after.

Control-plane node:

```bash
sudo apt-get update
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=<target>-*
sudo apt-mark hold kubeadm

sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v<target>

sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=<target>-* kubectl=<target>-*
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Worker node (drain from a machine with kubectl, upgrade on the node):

```bash
# from kubectl
kubectl drain <worker> --ignore-daemonsets --delete-emptydir-data

# on the worker
sudo apt-get update
sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-get install -y kubeadm=<target>-* kubelet=<target>-* kubectl=<target>-*
sudo apt-mark hold kubeadm kubelet kubectl
sudo kubeadm upgrade node
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# from kubectl
kubectl uncordon <worker>
kubectl get nodes
```

Replace `<target>` with the exact patch version from `kubeadm upgrade plan`. Package naming differs across distributions; follow the install docs if apt names differ.

## Verify

```bash
kubectl get nodes -o wide         # VERSION column shows each node
kubeadm version
kubectl get pods -n kube-system
```

## Common mistakes

- Upgrading kubelet on the control plane before running `kubeadm upgrade apply`.
- Forgetting `kubeadm upgrade node` on workers; kubelet then runs with an outdated config.
- Leaving a worker cordoned after upgrade.

## Quick reference

```bash
kubeadm token create --print-join-command
kubeadm upgrade plan
kubeadm upgrade apply v<version>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```

---

Prev: [RBAC](rbac.md) · Next: [Certificates and kubeconfig](certificates-and-kubeconfig.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
