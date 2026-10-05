# CKAD Complete Crash Course

Certified Kubernetes Application Developer — hub document. Start here; each domain links to topic docs.

Source material: the 2026 guide and the 2025 crash course (archived in [_archive](_archive/)). This hub supersedes both.

Many CKAD topics are shared with CKA. Those link to the CKA docs rather than repeating them.

## Exam at a glance

| Item | Value |
|---|---|
| Format | Performance-based, online, proctored; tasks run on the command line in live clusters |
| Duration | 2 hours |
| Passing score | 66% (stated in both source guides; not on the CNCF page checked) |
| Cost | $445, includes one free retake (CNCF page) |
| Prerequisite | None |
| Kubernetes version | Not verified; check the version the exam runs |

## Domains and weights

Weights from the CNCF CKAD certification page.

| # | Domain | Weight | Topic docs |
|---|---|---|---|
| 1 | Application Design and Build | 20% | [Multi-container patterns](docs/01-application-design-build/multi-container-patterns.md) · [Jobs and CronJobs](docs/01-application-design-build/jobs-and-cronjobs.md) · [Container images](docs/01-application-design-build/container-images.md) · [Volumes and workload choice](docs/01-application-design-build/volumes-and-workload-choice.md) |
| 2 | Application Deployment | 20% | [Deployment strategies](docs/02-application-deployment/deployment-strategies.md) · [Helm and Kustomize](../cka/docs/04-workloads-scheduling/helm-and-kustomize.md) (shared with CKA) |
| 3 | Application Observability and Maintenance | 15% | [Probes](docs/03-application-observability-maintenance/probes.md) · [Logs, debugging and API deprecations](docs/03-application-observability-maintenance/logs-debugging-and-deprecations.md) |
| 4 | Application Environment, Configuration and Security | 25% | [ConfigMaps and Secrets](docs/04-application-environment-config-security/configmaps-and-secrets.md) · [SecurityContext, quotas and limits](docs/04-application-environment-config-security/security-context-quotas-limits.md) · [ServiceAccounts and RBAC](docs/04-application-environment-config-security/serviceaccounts-and-rbac.md) |
| 5 | Services and Networking | 20% | [Services, NetworkPolicy and Ingress](docs/05-services-networking/services-networking.md) |

Shared with CKA (read those docs too):

- [Deployments and scaling](../cka/docs/04-workloads-scheduling/deployments-and-scaling.md)
- [Services and DNS](../cka/docs/03-services-networking/services-and-dns.md)
- [NetworkPolicy](../cka/docs/03-services-networking/network-policy.md)
- [Ingress and Gateway API](../cka/docs/03-services-networking/ingress-and-gateway-api.md)
- [CRDs and operators](../cka/docs/02-cluster-architecture-installation-config/crds-and-operators.md)
- [kubectl speed and vim](../cka/reference/kubectl-speed-and-vim.md)

## Study order

1. **Speed and basics:** [kubectl speed](../cka/reference/kubectl-speed-and-vim.md), [Deployments](../cka/docs/04-workloads-scheduling/deployments-and-scaling.md).
2. **Multi-container and probes (35% combined):** [multi-container patterns](docs/01-application-design-build/multi-container-patterns.md), [probes](docs/03-application-observability-maintenance/probes.md).
3. **Configuration and security (25%):** [ConfigMaps and Secrets](docs/04-application-environment-config-security/configmaps-and-secrets.md), [SecurityContext](docs/04-application-environment-config-security/security-context-quotas-limits.md).
4. **Jobs, deployments, services, debugging:** remaining docs.
5. **Mock exams** under timed conditions.

## Pre-exam checklist

- [ ] Add a sidecar sharing an emptyDir to an existing Pod
- [ ] Add an init container that waits for a Service
- [ ] Add liveness, readiness and startup probes with correct paths and timings
- [ ] Create a Job with completions and parallelism; create a CronJob with a correct schedule
- [ ] Mount a ConfigMap as files and a Secret as env
- [ ] Set container securityContext so the app runs as a non-root UID
- [ ] Create a ResourceQuota and show a pod rejected by it
- [ ] Do a rolling update, roll back, and do a blue/green switch with a Service selector
- [ ] Fix a NetworkPolicy egress rule that forgot DNS
- [ ] Fix a deprecated apiVersion in a manifest

## Quick command reference

```bash
kubectl run app --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl logs <pod> -c <container> --previous
kubectl describe pod <pod> | sed -n '/Events:/,$p'
kubectl create job j --image=busybox -- sh -c 'echo ok'
kubectl create cronjob c --image=busybox --schedule="*/5 * * * *" -- sh -c 'date'
kubectl create configmap cfg --from-literal=KEY=value
kubectl create secret generic creds --from-literal=password=secret
kubectl create quota q --hard=cpu=2,memory=2Gi,pods=5
```

## Document map

```
CKAD_2026_Complete_Crash_Course.md   this hub
docs/01-application-design-build/        multi-container, jobs, images, volumes
docs/02-application-deployment/          strategies (Helm and Kustomize shared with CKA)
docs/03-application-observability-.../   probes, logs, deprecations
docs/04-application-environment-.../     ConfigMaps, Secrets, security context, quotas, RBAC
docs/05-services-networking/             services, NetworkPolicy, ingress
notes/                                   scope, corrections, open questions
_archive/                                original 2025 and 2026 guides, unmodified
```
