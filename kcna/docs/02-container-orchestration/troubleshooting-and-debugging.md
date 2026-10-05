# Troubleshooting and Debugging

Up: [KCNA README](../../README.md) · Domain 2 — Container Orchestration (28%, Troubleshooting) · Prev: [Security basics](security-basics.md) · Next: [CI/CD and GitOps](../03-cloud-native-application-delivery/cicd-and-gitops.md)

Most problems announce themselves with a status. This topic explains what the common statuses mean and where the evidence for each one lives.

Official curriculum topics covered here: troubleshooting (listed under Container Orchestration) and debugging (listed under Application Delivery). KCNA tests concepts, so this page is about what each signal means, not command syntax.

## Start from the symptom

| Symptom | What it usually means |
|---|---|
| Pod stays Pending | The scheduler cannot place it: resources, taints, selectors or an unbound volume |
| Pod in CrashLoopBackOff | The container starts and exits repeatedly; check its logs and exit code |
| ImagePullBackOff | The image name, tag or registry credentials are wrong, or the registry is unreachable |
| Pod Running but not Ready | A readiness probe is failing; the pod gets no Service traffic |
| Service has no endpoints | The Service selector matches no ready pods |
| Node NotReady | The kubelet, container runtime or network on that node is unhealthy |

## Signals to read

- **Events:** Kubernetes records why it took an action (scheduling failures, image pulls, probe failures). Check them first.
- **Logs:** application output from each container. The previous container's logs matter after a crash.
- **Status and conditions:** the phase of a pod and the conditions on a node show the current state.
- **Metrics:** resource use over time shows pressure (CPU throttling, memory limits reached).

## Debugging approach

1. Confirm the scope: one pod, one workload, one node, or the whole cluster?
2. Read events, then logs, then the object's spec.
3. Form one hypothesis and test it.
4. Change one thing, then verify.

Ephemeral debug containers let you attach a troubleshooting container to a running pod without changing its spec. Use them when the application image has no shell.
