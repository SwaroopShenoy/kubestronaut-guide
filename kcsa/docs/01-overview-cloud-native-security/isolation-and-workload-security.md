# Isolation Techniques and Workload Security

Up: [KCSA README](../../README.md) · Domain 1 — Overview of Cloud Native Security (14%) · Prev: [The 4Cs and shared responsibility](four-cs-and-shared-responsibility.md) · Next: [Control plane security](../02-kubernetes-cluster-component-security/control-plane-security.md)

Isolation decides how far a compromise can travel. This chapter covers the layers of isolation, workload hardening, and how images and artifacts should be handled.

Official curriculum topics covered here: isolation techniques, artifact repository and image security, and workload and application code security.

## Isolation techniques

Isolation limits what a compromised workload can reach. Layers, from weakest to strongest:

| Technique | What it isolates | Limits |
|---|---|---|
| Namespaces | Names, quotas and RBAC scope | Not a security boundary on their own; pods in one namespace share a kernel |
| NetworkPolicy | Pod-to-pod traffic | Needs an enforcing CNI |
| Pod Security Admission | Privilege of pods in a namespace | Only blocks settings; does not isolate runtime |
| Separate node pools or taints | Workloads on different machines | Cost and scheduling complexity |
| Sandboxed runtimes (gVisor, Kata) | Kernel exposure of a container | Performance cost; needs runtime support |
| Separate clusters | Control plane and blast radius | Operational overhead |

Multi-tenancy: a shared cluster with several teams needs namespaces plus RBAC, quotas, NetworkPolicy, and PSA together. Namespaces alone are not enough for hostile tenants.

## Workload and application code security

- Run as non-root with a read-only root filesystem and dropped capabilities.
- Do not run privileged containers or mount the host filesystem.
- Keep secrets out of images and source code; inject them at runtime.
- Scan dependencies (software composition analysis) and code (static analysis) in CI.
- Keep base images and language runtimes patched.

## Artifact repositories and image security

- Use a private registry with access control; allow only approved registries (admission policy).
- Sign images and verify signatures before running them.
- Scan images for known vulnerabilities and rebuild regularly.
- Pin images by digest where possible.
