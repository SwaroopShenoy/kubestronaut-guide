# CKA Complete Crash Course

Certified Kubernetes Administrator — hub document. Start here; each domain links to topic docs.

Source material: the 2026 guide and the 2025 crash course (archived in [_archive](_archive/)). This hub supersedes both.

## Exam at a glance

| Item | Value |
|---|---|
| Format | Performance-based, online, proctored; tasks run on the command line in live clusters |
| Duration | 2 hours |
| Passing score | 66% (confirmed by the 2025 and 2026 guides; not on the CNCF page checked) |
| Cost | $445, includes one free retake (CNCF page) |
| Prerequisite | None |
| Kubernetes version | Not verified. The 2026 guide targets v1.35; check the version the exam currently runs |
| Allowed documentation | Not verified. Check the current exam resources page before exam day |

## Domains and weights

Weights from the CNCF CKA certification page.

| # | Domain | Weight | Topic docs |
|---|---|---|---|
| 1 | Troubleshooting | 30% | [Troubleshooting method](docs/01-troubleshooting/troubleshooting-method.md) · [Control plane and nodes](docs/01-troubleshooting/control-plane-and-nodes.md) · [Services, DNS and networking](docs/01-troubleshooting/services-dns-and-networking.md) |
| 2 | Cluster Architecture, Installation and Configuration | 25% | [RBAC](docs/02-cluster-architecture-installation-config/rbac.md) · [kubeadm install and upgrade](docs/02-cluster-architecture-installation-config/kubeadm-install-and-upgrade.md) · [Certificates and kubeconfig](docs/02-cluster-architecture-installation-config/certificates-and-kubeconfig.md) · [etcd backup and restore](docs/02-cluster-architecture-installation-config/etcd-backup-restore.md) · [CRDs and operators](docs/02-cluster-architecture-installation-config/crds-and-operators.md) |
| 3 | Services and Networking | 20% | [Services and DNS](docs/03-services-networking/services-and-dns.md) · [NetworkPolicy](docs/03-services-networking/network-policy.md) · [Ingress and Gateway API](docs/03-services-networking/ingress-and-gateway-api.md) |
| 4 | Workloads and Scheduling | 15% | [Deployments and scaling](docs/04-workloads-scheduling/deployments-and-scaling.md) · [Workload types](docs/04-workloads-scheduling/workload-types.md) · [Scheduling](docs/04-workloads-scheduling/scheduling.md) · [Helm and Kustomize](docs/04-workloads-scheduling/helm-and-kustomize.md) |
| 5 | Storage | 10% | [Storage](docs/05-storage/storage.md) |

Reference material (not a domain):

- [kubectl speed and vim](reference/kubectl-speed-and-vim.md)
- [API discovery and documentation navigation](reference/api-discovery-and-docs.md)

## Study order

Start with the highest-weight domain, since troubleshooting touches everything else.

1. **Speed foundations:** [kubectl speed](reference/kubectl-speed-and-vim.md), [Deployments](docs/04-workloads-scheduling/deployments-and-scaling.md), [Services and DNS](docs/03-services-networking/services-and-dns.md).
2. **Troubleshooting (30%):** all three troubleshooting docs. Break things on purpose and fix them against a timer.
3. **Cluster architecture (25%):** [RBAC](docs/02-cluster-architecture-installation-config/rbac.md), [kubeadm](docs/02-cluster-architecture-installation-config/kubeadm-install-and-upgrade.md), [certificates](docs/02-cluster-architecture-installation-config/certificates-and-kubeconfig.md), [etcd](docs/02-cluster-architecture-installation-config/etcd-backup-restore.md).
4. **Networking, workloads and storage:** remaining topic docs.
5. **Mock exams:** timed sessions after the domains above.

Time plan:

- **3–4 weeks:** one or two domains per week, daily 1–2 hours of lab work, final week timed mocks.
- **1 week (refresher only):** troubleshooting and RBAC drills, then mocks.

## Exam habits

- Set the context and namespace from each task before typing (`kubectl config use-context`).
- Generate YAML with `--dry-run=client -o yaml`, then edit. Do not write everything from scratch.
- Verify every change with a command that proves the behavior, not just that `apply` succeeded.
- Flag hard tasks and return; partial credit is per task.

## Pre-exam checklist

- [ ] Walk a node from NotReady to Ready (kubelet, swap, certs, disk)
- [ ] Find and fix a broken static pod manifest
- [ ] Create Role/RoleBinding and prove it with `kubectl auth can-i`
- [ ] Take an etcd snapshot and verify it
- [ ] Upgrade a kubeadm control plane and one worker in order
- [ ] Create a Service, find why it has no endpoints, fix the selector
- [ ] Create a default-deny NetworkPolicy and an allow rule, with DNS
- [ ] Create a Deployment with resources, scale it, roll it back
- [ ] Create a PV/PVC pair and a Pod that mounts it
- [ ] Install a Helm chart, upgrade it, roll it back

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
CKA_2026_Complete_Crash_Course.md   this hub
docs/01-troubleshooting/            method, control plane and nodes, services and networking
docs/02-cluster-architecture.../    RBAC, kubeadm, certificates, etcd, CRDs
docs/03-services-networking/        services and DNS, NetworkPolicy, ingress and Gateway API
docs/04-workloads-scheduling/       deployments, workload types, scheduling, Helm and Kustomize
docs/05-storage/                    PV, PVC, StorageClass
reference/                          kubectl speed, API discovery
notes/                              scope, corrections, open questions
_archive/                           original 2025 and 2026 guides, unmodified
```
