# Sources and Verification: CKS

Up: [CKS README](../../README.md) · Notes

This page records where the figures in this section come from and what has and has not been checked.

## Official sources

- CNCF CKS Exam Curriculum v1.34 (PDF): https://raw.githubusercontent.com/cncf/curriculum/master/CKS_Curriculum%20v1.34.pdf
- CNCF CKS certification page: https://www.cncf.io/training/certification/cks/
- Linux Foundation CKS important instructions: https://docs.linuxfoundation.org/tc-docs/certification/important-instructions-cks
- Linux Foundation exam resources allowed: https://docs.linuxfoundation.org/tc-docs/certification/certification-resources-allowed
- All checked on 2026-10-05.

## Exam facts

| Item | Value | Source |
|---|---|---|
| Duration | 2 hours | CNCF page, Linux Foundation instructions |
| Tasks | 15–20 performance-based tasks | Linux Foundation instructions |
| Kubernetes version | v1.35 | Linux Foundation instructions |
| Pre-installed tools | kubectl (alias `k`), yq, curl, wget, man pages | Linux Foundation instructions |
| Prerequisite | CKA passed at any time; it need not be active | CNCF page |
| Allowed documentation | kubernetes.io/docs and blog, falco.org/docs, kubernetes-sigs.github.io/bom, etcd.io/docs, NGINX Ingress Controller configuration guide, docs.cilium.io, istio.io/latest/docs | Linux Foundation resources-allowed page |

## Curriculum summary

| Domain | Weight (curriculum v1.34) |
|---|---|
| Cluster Setup | 15% |
| Cluster Hardening | 15% |
| System Hardening | 10% |
| Minimize Microservice Vulnerabilities | 20% |
| Supply Chain Security | 20% |
| Monitoring, Logging and Runtime Security | 20% |

The CNCF certification page lists Cluster Setup at 10% and System Hardening at 15%. The curriculum PDF and community reports of the curriculum update (Cluster Setup rising from 10% to 15%) agree with each other, so this guide uses the PDF. Confirm the current figures with CNCF.

## Curriculum coverage map

| Curriculum item | Where it is covered |
|---|---|
| Network security policies to restrict cluster level access | [Network policy](../01-cluster-setup/network-policy.md) |
| CIS benchmark for etcd, kubelet, kubedns, kubeapi | [CIS benchmark](../01-cluster-setup/cis-benchmark-kube-bench.md), [etcd hardening](../01-cluster-setup/etcd-hardening.md) |
| Ingress objects with TLS | [Ingress TLS](../01-cluster-setup/ingress-tls-and-node-metadata.md) |
| Protect node metadata and endpoints | [Ingress TLS, node metadata](../01-cluster-setup/ingress-tls-and-node-metadata.md) |
| Verify platform binaries before deploying | [Ingress TLS, node metadata](../01-cluster-setup/ingress-tls-and-node-metadata.md) |
| RBAC to minimise exposure | [RBAC](../02-cluster-hardening/rbac.md) |
| Service account caution (disable defaults, minimise permissions) | [Service accounts and API access](../02-cluster-hardening/service-accounts-and-api-access.md) |
| Restrict access to the Kubernetes API | [Service accounts and API access](../02-cluster-hardening/service-accounts-and-api-access.md) |
| Upgrade Kubernetes to avoid vulnerabilities | [Cluster upgrades](../02-cluster-hardening/cluster-upgrades.md) |
| Minimise host OS footprint | [Host hardening](../03-system-hardening/host-hardening.md) |
| Least-privilege identity and access management | [Host hardening](../03-system-hardening/host-hardening.md) |
| Minimise external access to the network | [Host hardening](../03-system-hardening/host-hardening.md) |
| Kernel hardening tools such as AppArmor and seccomp | [AppArmor and seccomp](../03-system-hardening/apparmor-and-seccomp.md) |
| Appropriate pod security standards | [Security context and PSS](../04-microservice-vulnerabilities/security-context-and-pss.md) |
| Manage Kubernetes secrets | [Secrets and encryption at rest](../04-microservice-vulnerabilities/secrets-and-encryption-at-rest.md) |
| Isolation techniques (multi-tenancy, sandboxed containers) | [Runtime sandboxes](../04-microservice-vulnerabilities/runtime-sandboxes.md) |
| Pod-to-pod encryption (Cilium, Istio) | [Pod-to-pod encryption](../04-microservice-vulnerabilities/pod-to-pod-encryption.md) |
| Minimise base image footprint | [Base images and Dockerfiles](../05-supply-chain-security/base-images-and-dockerfiles.md) |
| Understand the supply chain (SBOM, CI/CD, artifact repositories) | [SBOM and the supply chain](../05-supply-chain-security/sbom-and-supply-chain.md) |
| Secure the supply chain (permitted registries, sign and validate) | [Admission control](../05-supply-chain-security/admission-control.md), [Image signing](../05-supply-chain-security/image-signing-cosign.md), [SBOM and the supply chain](../05-supply-chain-security/sbom-and-supply-chain.md) |
| Static analysis of workloads and images (Kubesec, KubeLinter) | [Static analysis](../05-supply-chain-security/static-analysis.md), [Image scanning](../05-supply-chain-security/image-scanning-trivy.md) |
| Behavioural analytics to detect malicious activity | [Falco](../06-monitoring-logging-runtime/falco.md) |
| Detect threats across infrastructure, apps, networks, data, users and workloads | [Falco](../06-monitoring-logging-runtime/falco.md), [Audit logging](../06-monitoring-logging-runtime/audit-logging.md) |
| Investigate phases of an attack and the actors | [Audit logging](../06-monitoring-logging-runtime/audit-logging.md) |
| Ensure immutability of containers at runtime | [Container immutability](../06-monitoring-logging-runtime/runtime-immutability.md) |
| Use Kubernetes audit logs to monitor access | [Audit logging](../06-monitoring-logging-runtime/audit-logging.md), [jq for audit data](../06-monitoring-logging-runtime/jq-for-audit-data.md) |

Tools on the allowed-resources list and where they appear: Falco ([Falco](../06-monitoring-logging-runtime/falco.md)), bom ([SBOM](../05-supply-chain-security/sbom-and-supply-chain.md)), etcd ([etcd hardening](../01-cluster-setup/etcd-hardening.md)), NGINX Ingress ([Ingress TLS](../01-cluster-setup/ingress-tls-and-node-metadata.md)), Cilium and Istio ([Pod-to-pod encryption](../04-microservice-vulnerabilities/pod-to-pod-encryption.md)).

## What the curriculum does and does not say

- It does not name Trivy, kube-bench, Cosign, OPA Gatekeeper or Kyverno. They appear in this guide as common examples of the outcomes it describes.
- It does not state that writing Rego policies is required, and OPA documentation is not on the allowed-resources list.
- It does not state that writing AppArmor or seccomp profiles from scratch is required. It says to "appropriately use" them.
- Falco documentation is on the allowed-resources list, so Falco is used in the exam. The curriculum says "detect" and does not describe how far rule editing goes.

## Not verified

- Passing score.
- Commands and YAML were written from documented behaviour and have not all been executed on a live cluster.
- The curriculum version reviewed is v1.34; newer versions were not reviewed.

---

<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
