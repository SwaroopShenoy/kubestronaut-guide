# Section III: CKA — Certified Kubernetes Administrator

CKA is the hands-on administration exam. You run tasks on live clusters: fixing broken nodes and control-plane components, configuring access, networking workloads and storage, and installing and upgrading clusters. Troubleshooting carries the largest weight, so the topics on diagnosis come first.

> **Read first: status and limits**
>
> This guide is an independent guide to the CNCF Kubernetes certifications. It is **not** an official CNCF or Linux Foundation resource and is not endorsed by them.
>
> - **Not a complete or current source of truth.** Domains and weights were checked against the official CNCF curriculum PDFs as of 2026-10-05. Exam formats, passing scores, allowed resources, Kubernetes versions and tool behaviour change, and may already differ from what is written here.
> - **Not all commands are tested.** Commands, flags and YAML were written from knowledge and have not all been run on a live cluster. Verify before you rely on them.
> - **Reading aid, not a substitute for practice.** Use it alongside the official curriculum, the official documentation and hands-on practice. Do not use it as your only preparation material.
> - **Open questions are marked.** Items labelled "not verified" or "unconfirmed" are open. Treat them as questions, not facts.
> - **No warranty.** The author accepts no responsibility for exam results, production changes, or decisions made from this content. Check the official sources yourself.

Certified Kubernetes Administrator — hub document. Start here; each domain links to topic docs.

## Exam at a glance

| Item | Value |
|---|---|
| Format | Performance-based, online, proctored; tasks run on the command line in live clusters |
| Duration | 2 hours |
| Passing score | 66% (not stated on the official CNCF page; verify) |
| Cost | $445, includes one free retake (CNCF page) |
| Prerequisite | None |
| Kubernetes version | v1.35 (Linux Foundation CKA and CKAD tips page) |
| Pre-installed tools | kubectl, yq, curl, wget and man pages on the SSH hosts |
| Allowed documentation | kubernetes.io/docs, kubernetes.io/blog, helm.sh/docs, gateway-api.sigs.k8s.io (Linux Foundation resources-allowed page) |

## Domains and weights

Weights from the official CNCF exam curriculum PDF.

| # | Domain | Weight | Topic docs |
|---|---|---|---|
| 1 | Troubleshooting | 30% | [Troubleshooting method](docs/01-troubleshooting/troubleshooting-method.md) · [Control plane and nodes](docs/01-troubleshooting/control-plane-and-nodes.md) · [Services, DNS and networking](docs/01-troubleshooting/services-dns-and-networking.md) · [Monitoring and logs](docs/01-troubleshooting/monitoring-and-logs.md) |
| 2 | Cluster Architecture, Installation and Configuration | 25% | [RBAC](docs/02-cluster-architecture-installation-config/rbac.md) · [kubeadm install and upgrade](docs/02-cluster-architecture-installation-config/kubeadm-install-and-upgrade.md) · [Certificates and kubeconfig](docs/02-cluster-architecture-installation-config/certificates-and-kubeconfig.md) · [etcd backup and restore](docs/02-cluster-architecture-installation-config/etcd-backup-restore.md) · [CRDs and operators](docs/02-cluster-architecture-installation-config/crds-and-operators.md) · [Highly available control plane](docs/02-cluster-architecture-installation-config/high-availability-control-plane.md) · [Extension interfaces (CNI, CSI, CRI)](docs/02-cluster-architecture-installation-config/extension-interfaces.md) |
| 3 | Services and Networking | 20% | [Services and DNS](docs/03-services-networking/services-and-dns.md) · [NetworkPolicy](docs/03-services-networking/network-policy.md) · [Ingress and Gateway API](docs/03-services-networking/ingress-and-gateway-api.md) |
| 4 | Workloads and Scheduling | 15% | [Deployments and scaling](docs/04-workloads-scheduling/deployments-and-scaling.md) · [Workload types](docs/04-workloads-scheduling/workload-types.md) · [Scheduling](docs/04-workloads-scheduling/scheduling.md) · [Helm and Kustomize](docs/04-workloads-scheduling/helm-and-kustomize.md) · [Configuration, autoscaling and self-healing](docs/04-workloads-scheduling/configuration-and-autoscaling.md) |
| 5 | Storage | 10% | [Storage](docs/05-storage/storage.md) |

Reference material (not a domain):

- [kubectl speed and vim](reference/kubectl-speed-and-vim.md)
- [API discovery and documentation navigation](reference/api-discovery-and-docs.md)

## Exam habits

- Set the context and namespace from each task before typing (`kubectl config use-context`).
- Generate YAML with `--dry-run=client -o yaml`, then edit. Do not write everything from scratch.
- Verify every change with a command that proves the behavior, not just that `apply` succeeded.
- Flag hard tasks and return; partial credit is per task.

## Key ideas at a glance

- Walk a node from NotReady to Ready (kubelet, swap, certs, disk)
- Find and fix a broken static pod manifest
- Create Role/RoleBinding and prove it with `kubectl auth can-i`
- Take an etcd snapshot and verify it
- Upgrade a kubeadm control plane and one worker in order
- Create a Service, find why it has no endpoints, fix the selector
- Create a default-deny NetworkPolicy and an allow rule, with DNS
- Create a Deployment with resources, scale it, roll it back
- Create a PV/PVC pair and a Pod that mounts it
- Install a Helm chart, upgrade it, roll it back

## Quick command reference

```bash
kubectl config use-context <ctx>
kubectl get pods -A -o wide
kubectl describe pod <pod> | tail -20         # events are at the bottom
kubectl logs <pod> --previous
kubectl auth can-i --list --as=<subject>
kubectl get events -A --sort-by=.lastTimestamp
kubectl explain <resource>.<field>
```

## Document map

```
README.md   this hub
docs/01-troubleshooting/            method, control plane and nodes, services and networking
docs/02-cluster-architecture.../    RBAC, kubeadm, certificates, etcd, CRDs
docs/03-services-networking/        services and DNS, NetworkPolicy, ingress and Gateway API
docs/04-workloads-scheduling/       deployments, workload types, scheduling, Helm and Kustomize
docs/05-storage/                    PV, PVC, StorageClass
reference/                          kubectl speed, API discovery
notes/                              sources and verification
```

---

<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../LICENSE)</sub>
