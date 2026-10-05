# Cluster Upgrades for Security

Up: [CKS hub](../../CKS_2026_Complete_Crash_Course.md) · Domain 2 — Cluster Hardening (15%) · Prev: [Service accounts and API access](service-accounts-and-api-access.md) · Next: [Host hardening](../03-system-hardening/host-hardening.md)

This doc is **new**. Keeping control-plane and node components patched is part of cluster hardening in the curriculum summaries we checked; confirm the exact scope against the official curriculum.

## Why it matters

Known CVEs in kube-apiserver, kubelet, or the container runtime are fixed in specific releases. An unpatched control plane is a hardening finding even if every other control is in place.

## Rules

- Upgrade one minor version at a time. Skipping minors is not supported.
- Control plane first, then worker nodes.
- kubelet must not be newer than kube-apiserver. Node components may lag the control plane by a bounded number of minor versions (the version-skew policy).
- Drain nodes before upgrading their kubelet.

## kubeadm workflow (control-plane node)

```bash
# Check current and available versions
kubeadm version
kubectl get nodes

sudo apt-get update
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.34.x-*    # pick the target patch release
sudo apt-mark hold kubeadm

sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.34.x

sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.34.x-* kubectl=1.34.x-*
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

## Worker node

From a machine with kubectl access:

```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

On the node:

```bash
sudo apt-get update
sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-get install -y kubeadm=1.34.x-* kubelet=1.34.x-* kubectl=1.34.x-*
sudo apt-mark hold kubeadm kubelet kubectl
sudo kubeadm upgrade node
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Back from kubectl:

```bash
kubectl uncordon <node>
kubectl get nodes      # confirm the new VERSION column
```

Replace `1.34.x` with the target patch release. Package names depend on the distribution; follow the install docs for your environment.

## Verifying after an upgrade

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
kubeadm version
```

Re-run kube-bench after upgrades. Some remediations change between versions, and an upgrade can reset config files that kubeadm regenerates.

## Common mistakes

- Upgrading kubelet before the control plane.
- Forgetting to `uncordon` the node, leaving it unschedulable.
- Editing manifests in `/etc/kubernetes/manifests` and then running `kubeadm upgrade`, which can overwrite them. Keep a backup copy of any custom flags.

## Practice

1. Read `kubeadm upgrade plan` output on a lab cluster and identify the target version and any warnings.
2. Drain and uncordon a worker node; confirm pods rescheduled.
3. Write down which steps must happen on which machine (control plane, worker, workstation).
