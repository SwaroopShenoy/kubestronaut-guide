# Sources and Verification: CKA

Up: [CKA README](../README.md) · Notes

This page records where the figures in this section come from and what has and has not been checked.

## Official sources

- CNCF CKA Exam Curriculum v1.35 (PDF): https://raw.githubusercontent.com/cncf/curriculum/master/CKA_Curriculum_v1.35.pdf
- CNCF CKA certification page: https://www.cncf.io/training/certification/cka/
- Linux Foundation CKA and CKAD tips: https://docs.linuxfoundation.org/tc-docs/certification/tips-cka-and-ckad
- Linux Foundation exam resources allowed: https://docs.linuxfoundation.org/tc-docs/certification/certification-resources-allowed
- All checked on 2026-10-05.

## Exam facts

| Item | Value | Source |
|---|---|---|
| Duration | 2 hours | CNCF page |
| Kubernetes version | v1.35 | Linux Foundation tips page |
| Pre-installed tools | kubectl, yq, curl, wget, man pages | Linux Foundation tips page |
| Allowed documentation | kubernetes.io/docs and blog, helm.sh/docs, gateway-api.sigs.k8s.io | Linux Foundation resources-allowed page |

## Curriculum summary

| Domain | Weight |
|---|---|
| Troubleshooting | 30% |
| Cluster Architecture, Installation and Configuration | 25% |
| Services and Networking | 20% |
| Workloads and Scheduling | 15% |
| Storage | 10% |

## Curriculum coverage map

| Curriculum item | Where it is covered |
|---|---|
| Troubleshoot clusters and nodes | [Control plane and nodes](../docs/01-troubleshooting/control-plane-and-nodes.md) |
| Troubleshoot cluster components | [Control plane and nodes](../docs/01-troubleshooting/control-plane-and-nodes.md) |
| Monitor cluster and application resource usage | [Monitoring and logs](../docs/01-troubleshooting/monitoring-and-logs.md) |
| Manage and evaluate container output streams | [Monitoring and logs](../docs/01-troubleshooting/monitoring-and-logs.md) |
| Troubleshoot services and networking | [Services, DNS and networking](../docs/01-troubleshooting/services-dns-and-networking.md) |
| Manage RBAC | [RBAC](../docs/02-cluster-architecture-installation-config/rbac.md) |
| Prepare infrastructure; create and manage clusters with kubeadm | [kubeadm install and upgrade](../docs/02-cluster-architecture-installation-config/kubeadm-install-and-upgrade.md) |
| Manage the cluster lifecycle | [kubeadm install and upgrade](../docs/02-cluster-architecture-installation-config/kubeadm-install-and-upgrade.md), [etcd backup and restore](../docs/02-cluster-architecture-installation-config/etcd-backup-restore.md), [Certificates and kubeconfig](../docs/02-cluster-architecture-installation-config/certificates-and-kubeconfig.md) |
| Highly available control plane | [Highly available control plane](../docs/02-cluster-architecture-installation-config/high-availability-control-plane.md) |
| Helm and Kustomize to install cluster components | [Helm and Kustomize](../docs/04-workloads-scheduling/helm-and-kustomize.md) |
| Extension interfaces (CNI, CSI, CRI) | [Extension interfaces](../docs/02-cluster-architecture-installation-config/extension-interfaces.md) |
| CRDs, operators | [CRDs and operators](../docs/02-cluster-architecture-installation-config/crds-and-operators.md) |
| Connectivity between Pods | [Services, DNS and networking](../docs/01-troubleshooting/services-dns-and-networking.md), [NetworkPolicy](../docs/03-services-networking/network-policy.md) |
| Define and enforce Network Policies | [NetworkPolicy](../docs/03-services-networking/network-policy.md) |
| ClusterIP, NodePort, LoadBalancer, endpoints | [Services and DNS](../docs/03-services-networking/services-and-dns.md) |
| Gateway API | [Ingress and Gateway API](../docs/03-services-networking/ingress-and-gateway-api.md) |
| Ingress controllers and resources | [Ingress and Gateway API](../docs/03-services-networking/ingress-and-gateway-api.md) |
| CoreDNS | [Services and DNS](../docs/03-services-networking/services-and-dns.md) |
| Deployments, rolling updates and rollbacks | [Deployments and scaling](../docs/04-workloads-scheduling/deployments-and-scaling.md) |
| ConfigMaps and Secrets | [Configuration, autoscaling and self-healing](../docs/04-workloads-scheduling/configuration-and-autoscaling.md) |
| Workload autoscaling | [Configuration, autoscaling and self-healing](../docs/04-workloads-scheduling/configuration-and-autoscaling.md) |
| Primitives for self-healing deployments | [Configuration, autoscaling and self-healing](../docs/04-workloads-scheduling/configuration-and-autoscaling.md), [Workload types](../docs/04-workloads-scheduling/workload-types.md) |
| Pod admission and scheduling (limits, node affinity) | [Scheduling](../docs/04-workloads-scheduling/scheduling.md), [Configuration, autoscaling and self-healing](../docs/04-workloads-scheduling/configuration-and-autoscaling.md) |
| Storage classes and dynamic provisioning | [Storage](../docs/05-storage/storage.md) |
| Volume types, access modes, reclaim policies | [Storage](../docs/05-storage/storage.md) |
| PersistentVolumes and claims | [Storage](../docs/05-storage/storage.md) |

## Not verified

- Passing score and number of tasks.
- Commands and YAML were written from documented behaviour and have not all been executed on a live cluster.
- The curriculum version reviewed is v1.35.

---

<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../LICENSE)</sub>
