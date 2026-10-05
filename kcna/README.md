# Section I: KCNA — Kubernetes and Cloud Native Associate

KCNA is the entry point. It tests whether you understand what Kubernetes is, how its sections fit together, and the wider cloud native ecosystem around it. There are no terminal tasks; the exam is multiple choice, so the work here is building a clear mental model of each concept.

> **Read first: status and limits**
>
> This guide is an independent guide to the CNCF Kubernetes certifications. It is **not** an official CNCF or Linux Foundation resource and is not endorsed by them.
>
> - **Not a complete or current source of truth.** Domains and weights were checked against the official CNCF curriculum PDFs as of 2026-10-05. Exam formats, passing scores, allowed resources, Kubernetes versions and tool behaviour change, and may already differ from what is written here.
> - **Not all commands are tested.** Commands, flags and YAML were written from knowledge and have not all been run on a live cluster. Verify before you rely on them.
> - **Reading aid, not a substitute for practice.** Use it alongside the official curriculum, the official documentation and hands-on practice. Do not use it as your only preparation material.
> - **Open questions are marked.** Items labelled "not verified" or "unconfirmed" are open. Treat them as questions, not facts.
> - **No warranty.** The author accepts no responsibility for exam results, production changes, or decisions made from this content. Check the official sources yourself.

Kubernetes and Cloud Native Associate — hub document. Start here; each domain links to topic docs.

## Exam at a glance

| Item | Value |
|---|---|
| Format | Online, proctored, multiple choice; no terminal tasks |
| Duration | Not stated on the official CNCF page; verify |
| Questions | Not stated on the official CNCF page; verify |
| Passing score | Not stated on the official CNCF page; verify |
| Cost | $250, includes one free retake (CNCF page) |
| Prerequisite | None |

## Domains and weights

Weights from the official CNCF exam curriculum PDF.

| # | Domain | Weight | Topic docs |
|---|---|---|---|
| 1 | Kubernetes Fundamentals | 44% | [Architecture](docs/01-kubernetes-fundamentals/kubernetes-architecture.md) · [API objects and workloads](docs/01-kubernetes-fundamentals/api-objects-and-workloads.md) · [Services, storage and configuration](docs/01-kubernetes-fundamentals/services-storage-config.md) · [Containerization and administration](docs/01-kubernetes-fundamentals/containerization-and-administration.md) |
| 2 | Container Orchestration | 28% | [Runtimes, networking and interfaces](docs/02-container-orchestration/runtimes-networking-interfaces.md) · [Scheduling and scaling](docs/02-container-orchestration/scheduling-and-scaling.md) · [Security basics](docs/02-container-orchestration/security-basics.md) · [Troubleshooting and debugging](docs/02-container-orchestration/troubleshooting-and-debugging.md) |
| 3 | Cloud Native Application Delivery | 16% | [CI/CD and GitOps](docs/03-cloud-native-application-delivery/cicd-and-gitops.md) · [Packaging with Helm and Kustomize](docs/03-cloud-native-application-delivery/packaging-helm-kustomize.md) |
| 4 | Cloud Native Architecture | 12% | [Principles and patterns](docs/04-cloud-native-architecture/principles-and-patterns.md) · [Observability](docs/04-cloud-native-architecture/observability.md) · [Ecosystem and community](docs/04-cloud-native-architecture/cloud-native-community.md) |

Reference:

- [CNCF project map](reference/cncf-project-map.md)

## Key ideas at a glance

- Name the control-plane and node components and what each does
- Explain the difference between Deployment, StatefulSet, DaemonSet, Job and CronJob
- Choose the right Service type for a scenario
- Explain the CRI, CNI and CSI interfaces with an example each
- Describe RBAC: Role, ClusterRole, RoleBinding, ClusterRoleBinding, subjects
- Name the three Pod Security Standards levels
- Explain GitOps in terms of desired state and reconciliation
- Map common CNCF projects to their purpose (see the project map)
- Explain metrics, logs and traces, and which tool serves each

## Document map

```
README.md              this hub
docs/01-kubernetes-fundamentals/   architecture, API objects, services/storage/config
docs/02-container-orchestration/   runtimes and networking, scheduling, security basics
docs/03-cloud-native-application-delivery/   CI/CD and GitOps, Helm and Kustomize
docs/04-cloud-native-architecture/ principles, observability
reference/                         CNCF project map
notes/                             sources and verification
```

---

<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../LICENSE)</sub>
