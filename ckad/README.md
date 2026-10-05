# Section IV: CKAD — Certified Kubernetes Application Developer

CKAD is the developer exam. It is about building, configuring, deploying and debugging applications on Kubernetes, using the resources developers touch every day: pods, deployments, jobs, probes, configuration and services. Like CKA, it is performance-based on a live cluster.

> **Read first: status and limits**
>
> This guide is an independent guide to the CNCF Kubernetes certifications. It is **not** an official CNCF or Linux Foundation resource and is not endorsed by them.
>
> - **Not a complete or current source of truth.** Domains and weights were checked against the official CNCF curriculum PDFs as of 2026-10-05. Exam formats, passing scores, allowed resources, Kubernetes versions and tool behaviour change, and may already differ from what is written here.
> - **Not all commands are tested.** Commands, flags and YAML were written from knowledge and have not all been run on a live cluster. Verify before you rely on them.
> - **Reading aid, not a substitute for practice.** Use it alongside the official curriculum, the official documentation and hands-on practice. Do not use it as your only preparation material.
> - **Open questions are marked.** Items labelled "not verified" or "unconfirmed" are open. Treat them as questions, not facts.
> - **No warranty.** The author accepts no responsibility for exam results, production changes, or decisions made from this content. Check the official sources yourself.

Certified Kubernetes Application Developer — hub document. Start here; each domain links to topic docs.

Many CKAD topics are shared with CKA. Those link to the CKA docs rather than repeating them.

## Exam at a glance

| Item | Value |
|---|---|
| Format | Performance-based, online, proctored; tasks run on the command line in live clusters |
| Duration | 2 hours |
| Passing score | 66% (not stated on the official CNCF page; verify) |
| Cost | $445, includes one free retake (CNCF page) |
| Prerequisite | None |
| Kubernetes version | Not verified; check the version the exam runs |

## Domains and weights

Weights from the official CNCF exam curriculum PDF.

| # | Domain | Weight | Topic docs |
|---|---|---|---|
| 1 | Application Design and Build | 20% | [Multi-container patterns](docs/01-application-design-build/multi-container-patterns.md) · [Jobs and CronJobs](docs/01-application-design-build/jobs-and-cronjobs.md) · [Container images](docs/01-application-design-build/container-images.md) · [Volumes and workload choice](docs/01-application-design-build/volumes-and-workload-choice.md) |
| 2 | Application Deployment | 20% | [Deployment strategies](docs/02-application-deployment/deployment-strategies.md) · [Helm and Kustomize](../cka/docs/04-workloads-scheduling/helm-and-kustomize.md) (shared with CKA) |
| 3 | Application Observability and Maintenance | 15% | [Probes](docs/03-application-observability-maintenance/probes.md) · [Logs, debugging and API deprecations](docs/03-application-observability-maintenance/logs-debugging-and-deprecations.md) |
| 4 | Application Environment, Configuration and Security | 25% | [ConfigMaps and Secrets](docs/04-application-environment-config-security/configmaps-and-secrets.md) · [SecurityContext, quotas and limits](docs/04-application-environment-config-security/security-context-quotas-limits.md) · [ServiceAccounts and RBAC](docs/04-application-environment-config-security/serviceaccounts-and-rbac.md) · [Extending Kubernetes (CRDs, operators)](../cka/docs/02-cluster-architecture-installation-config/crds-and-operators.md) |
| 5 | Services and Networking | 20% | [Services, NetworkPolicy and Ingress](docs/05-services-networking/services-networking.md) |

Shared with CKA (read those docs too):

- [Deployments and scaling](../cka/docs/04-workloads-scheduling/deployments-and-scaling.md)
- [Services and DNS](../cka/docs/03-services-networking/services-and-dns.md)
- [NetworkPolicy](../cka/docs/03-services-networking/network-policy.md)
- [Ingress and Gateway API](../cka/docs/03-services-networking/ingress-and-gateway-api.md)
- [CRDs and operators](../cka/docs/02-cluster-architecture-installation-config/crds-and-operators.md)
- [kubectl speed and vim](../cka/reference/kubectl-speed-and-vim.md)

## Key ideas at a glance

- Add a sidecar sharing an emptyDir to an existing Pod
- Add an init container that waits for a Service
- Add liveness, readiness and startup probes with correct paths and timings
- Create a Job with completions and parallelism; create a CronJob with a correct schedule
- Mount a ConfigMap as files and a Secret as env
- Set container securityContext so the app runs as a non-root UID
- Create a ResourceQuota and show a pod rejected by it
- Do a rolling update, roll back, and do a blue/green switch with a Service selector
- Fix a NetworkPolicy egress rule that forgot DNS
- Fix a deprecated apiVersion in a manifest

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
README.md   this hub
docs/01-application-design-build/        multi-container, jobs, images, volumes
docs/02-application-deployment/          strategies (Helm and Kustomize shared with CKA)
docs/03-application-observability-.../   probes, logs, deprecations
docs/04-application-environment-.../     ConfigMaps, Secrets, security context, quotas, RBAC
docs/05-services-networking/             services, NetworkPolicy, ingress
notes/                                   sources and verification
```

---

<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../LICENSE)</sub>
