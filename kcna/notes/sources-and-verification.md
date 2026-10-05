# Sources and Verification: KCNA

Up: [KCNA README](../README.md) · Notes

This page records where the figures in this section come from and what has and has not been checked.

## Official sources

- CNCF KCNA Exam Curriculum (PDF): https://raw.githubusercontent.com/cncf/curriculum/master/KCNA_Curriculum.pdf
- CNCF KCNA certification page: https://www.cncf.io/training/certification/kcna/
- Both were checked on 2026-10-05.

## Curriculum summary

| Domain | Weight | Topics in the curriculum |
|---|---|---|
| Kubernetes Fundamentals | 44% | Core concepts, administration, scheduling, containerization |
| Container Orchestration | 28% | Networking, security, troubleshooting, storage |
| Cloud Native Application Delivery | 16% | Application delivery, debugging |
| Cloud Native Architecture | 12% | Observability, ecosystem and principles, community and collaboration |

## Curriculum coverage map

| Curriculum item | Where it is covered |
|---|---|
| Kubernetes core concepts | [Architecture](../docs/01-kubernetes-fundamentals/kubernetes-architecture.md), [API objects and workloads](../docs/01-kubernetes-fundamentals/api-objects-and-workloads.md) |
| Administration | [Containerization and administration](../docs/01-kubernetes-fundamentals/containerization-and-administration.md) |
| Scheduling | [Scheduling and scaling](../docs/02-container-orchestration/scheduling-and-scaling.md) |
| Containerization | [Containerization and administration](../docs/01-kubernetes-fundamentals/containerization-and-administration.md) |
| Networking | [Runtimes, networking and interfaces](../docs/02-container-orchestration/runtimes-networking-interfaces.md), [Services, storage and configuration](../docs/01-kubernetes-fundamentals/services-storage-config.md) |
| Security | [Security basics](../docs/02-container-orchestration/security-basics.md) |
| Troubleshooting | [Troubleshooting and debugging](../docs/02-container-orchestration/troubleshooting-and-debugging.md) |
| Storage | [Services, storage and configuration](../docs/01-kubernetes-fundamentals/services-storage-config.md) |
| Application delivery | [CI/CD and GitOps](../docs/03-cloud-native-application-delivery/cicd-and-gitops.md), [Packaging with Helm and Kustomize](../docs/03-cloud-native-application-delivery/packaging-helm-kustomize.md) |
| Debugging | [Troubleshooting and debugging](../docs/02-container-orchestration/troubleshooting-and-debugging.md) |
| Observability | [Observability](../docs/04-cloud-native-architecture/observability.md) |
| Cloud native ecosystem and principles | [Principles and patterns](../docs/04-cloud-native-architecture/principles-and-patterns.md), [CNCF project map](../reference/cncf-project-map.md) |
| Community and collaboration | [Ecosystem and community](../docs/04-cloud-native-architecture/cloud-native-community.md) |

## Not verified

- Duration, number of questions and passing score. The CNCF page checked does not state them.
- Validity period.
- Project maturity levels in the CNCF landscape change over time. Check the landscape site before relying on them.

---

<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../LICENSE)</sub>
