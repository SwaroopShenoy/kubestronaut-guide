# KCNA Crash Course

Kubernetes and Cloud Native Associate — hub document. Start here; each domain links to topic docs.

Source material: the 2025 weekend crash course (archived in [_archive](_archive/)). This hub supersedes it.

## Exam at a glance

| Item | Value |
|---|---|
| Format | Online, proctored, multiple choice; no terminal tasks |
| Duration | 90 minutes (source guide; not verified on the CNCF page) |
| Questions | 60 (source guide; not verified) |
| Passing score | 75% (source guide; not verified) |
| Cost | $250, includes one free retake (CNCF page) |
| Prerequisite | None |

## Domains and weights

Weights from the CNCF KCNA certification page. The 2025 guide used a different five-domain split (46/22/16/8/8) that does not match the official page; this hub follows the official four domains.

| # | Domain | Weight | Topic docs |
|---|---|---|---|
| 1 | Kubernetes Fundamentals | 44% | [Architecture](docs/01-kubernetes-fundamentals/kubernetes-architecture.md) · [API objects and workloads](docs/01-kubernetes-fundamentals/api-objects-and-workloads.md) · [Services, storage and configuration](docs/01-kubernetes-fundamentals/services-storage-config.md) |
| 2 | Container Orchestration | 28% | [Runtimes, networking and interfaces](docs/02-container-orchestration/runtimes-networking-interfaces.md) · [Scheduling and scaling](docs/02-container-orchestration/scheduling-and-scaling.md) · [Security basics](docs/02-container-orchestration/security-basics.md) |
| 3 | Cloud Native Application Delivery | 16% | [CI/CD and GitOps](docs/03-cloud-native-application-delivery/cicd-and-gitops.md) · [Packaging with Helm and Kustomize](docs/03-cloud-native-application-delivery/packaging-helm-kustomize.md) |
| 4 | Cloud Native Architecture | 12% | [Principles and patterns](docs/04-cloud-native-architecture/principles-and-patterns.md) · [Observability](docs/04-cloud-native-architecture/observability.md) |

Reference:

- [CNCF project map](reference/cncf-project-map.md)

## Pre-exam checklist

- [ ] Name the control-plane and node components and what each does
- [ ] Explain the difference between Deployment, StatefulSet, DaemonSet, Job and CronJob
- [ ] Choose the right Service type for a scenario
- [ ] Explain the CRI, CNI and CSI interfaces with an example each
- [ ] Describe RBAC: Role, ClusterRole, RoleBinding, ClusterRoleBinding, subjects
- [ ] Name the three Pod Security Standards levels
- [ ] Explain GitOps in terms of desired state and reconciliation
- [ ] Map common CNCF projects to their purpose (see the project map)
- [ ] Explain metrics, logs and traces, and which tool serves each

## Document map

```
KCNA_Crash_Course.md              this hub
docs/01-kubernetes-fundamentals/   architecture, API objects, services/storage/config
docs/02-container-orchestration/   runtimes and networking, scheduling, security basics
docs/03-cloud-native-application-delivery/   CI/CD and GitOps, Helm and Kustomize
docs/04-cloud-native-architecture/ principles, observability
reference/                         CNCF project map
notes/                             scope, corrections, open questions
_archive/                          original 2025 guide, unmodified
```
