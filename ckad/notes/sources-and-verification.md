# Sources and Verification: CKAD

Up: [CKAD README](../README.md) · Notes

This page records where the figures in this section come from and what has and has not been checked.

## Official sources

- CNCF CKAD Exam Curriculum v1.35 and v1.37 (PDF): https://raw.githubusercontent.com/cncf/curriculum/master/CKAD_Curriculum_v1.37.pdf
- CNCF CKAD certification page: https://www.cncf.io/training/certification/ckad/
- Linux Foundation CKA and CKAD tips: https://docs.linuxfoundation.org/tc-docs/certification/tips-cka-and-ckad
- Linux Foundation exam resources allowed: https://docs.linuxfoundation.org/tc-docs/certification/certification-resources-allowed
- All checked on 2026-10-05. The v1.35 and v1.37 curriculum files are byte-identical.

## Exam facts

| Item | Value | Source |
|---|---|---|
| Duration | About 2 hours | CNCF page |
| Kubernetes version | v1.37 | Linux Foundation tips page |
| Pre-installed tools | kubectl, yq, curl, wget, man pages | Linux Foundation tips page |
| Allowed documentation | kubernetes.io/docs and blog, helm.sh/docs | Linux Foundation resources-allowed page |

## Curriculum summary

| Domain | Weight |
|---|---|
| Application Design and Build | 20% |
| Application Deployment | 20% |
| Application Observability and Maintenance | 15% |
| Application Environment, Configuration and Security | 25% |
| Services and Networking | 20% |

## Curriculum coverage map

| Curriculum item | Where it is covered |
|---|---|
| Define, build and modify container images | [Container images](../docs/01-application-design-build/container-images.md) |
| Choose the right workload resource | [Volumes and workload choice](../docs/01-application-design-build/volumes-and-workload-choice.md), [Jobs and CronJobs](../docs/01-application-design-build/jobs-and-cronjobs.md) |
| Multi-container Pod design patterns | [Multi-container patterns](../docs/01-application-design-build/multi-container-patterns.md) |
| Persistent and ephemeral volumes | [Volumes and workload choice](../docs/01-application-design-build/volumes-and-workload-choice.md) |
| Deployment strategies (blue/green, canary) | [Deployment strategies](../docs/02-application-deployment/deployment-strategies.md) |
| Deployments and rolling updates | [Deployment strategies](../docs/02-application-deployment/deployment-strategies.md) |
| Helm; Kustomize | [Helm and Kustomize](../../cka/docs/04-workloads-scheduling/helm-and-kustomize.md) |
| API deprecations | [Logs, debugging and deprecations](../docs/03-application-observability-maintenance/logs-debugging-and-deprecations.md) |
| Probes and health checks | [Probes](../docs/03-application-observability-maintenance/probes.md) |
| Built-in CLI tools to monitor applications; container logs; debugging | [Logs, debugging and deprecations](../docs/03-application-observability-maintenance/logs-debugging-and-deprecations.md) |
| CRDs and operators | [CRDs and operators](../../cka/docs/02-cluster-architecture-installation-config/crds-and-operators.md) |
| Authentication, authorization and admission control | [Authentication, authorization and admission control](../docs/04-application-environment-config-security/auth-and-admission.md) |
| Requests, limits, quotas; resource requirements | [SecurityContext, quotas and limits](../docs/04-application-environment-config-security/security-context-quotas-limits.md) |
| ConfigMaps; Secrets | [ConfigMaps and Secrets](../docs/04-application-environment-config-security/configmaps-and-secrets.md) |
| ServiceAccounts | [ServiceAccounts and RBAC](../docs/04-application-environment-config-security/serviceaccounts-and-rbac.md) |
| Application security (SecurityContexts, capabilities) | [SecurityContext, quotas and limits](../docs/04-application-environment-config-security/security-context-quotas-limits.md) |
| NetworkPolicies; services; Ingress | [Services, NetworkPolicy and Ingress](../docs/05-services-networking/services-networking.md) |

## Not verified

- Passing score and number of tasks.
- Commands and YAML were written from documented behaviour and have not all been executed on a live cluster.

---

<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../LICENSE)</sub>
