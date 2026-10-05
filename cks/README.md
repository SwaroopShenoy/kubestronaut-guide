# Section V: CKS — Certified Kubernetes Security Specialist

CKS is the advanced security exam. It assumes the CKA and goes deeper into hardening clusters, securing workloads, protecting the supply chain and detecting threats at runtime. Like CKA, it is performance-based on a live cluster.

> **Read first: status and limits**
>
> This guide is an independent guide to the CNCF Kubernetes certifications. It is **not** an official CNCF or Linux Foundation resource and is not endorsed by them.
>
> - **Not a complete or current source of truth.** Domains and weights were checked against the official CNCF curriculum PDFs as of 2026-10-05. Exam formats, passing scores, allowed resources, Kubernetes versions and tool behaviour change, and may already differ from what is written here.
> - **Not all commands are tested.** Commands, flags and YAML were written from knowledge and have not all been run on a live cluster. Verify before you rely on them.
> - **Reading aid, not a substitute for practice.** Use it alongside the official curriculum, the official documentation and hands-on practice. Do not use it as your only preparation material.
> - **Open questions are marked.** Items labelled "not verified" or "unconfirmed" are open. Treat them as questions, not facts.
> - **No warranty.** The author accepts no responsibility for exam results, production changes, or decisions made from this content. Check the official sources yourself.

Certified Kubernetes Security Specialist — hub document. Start here; each domain links to a topic doc with full detail.

Last reviewed: 2026-10-05 against the official CNCF curriculum. See [sources and verification](docs/notes/sources-and-verification.md) for the official sources, what was checked, and what is not verified from the earlier version.

## Exam at a glance

| Item | Value |
|---|---|
| Format | Performance-based, online, proctored; tasks run on the command line in a live cluster |
| Duration | 2 hours |
| Prerequisite | CKA passed at any time before registering (CKA need not be active) |
| Validity | 2 years |
| Passing score | Not stated on the official pages checked; verify |
| Task count | Not stated on the official pages checked; verify |

## Domains and weights

Weights are from the official CNCF CKS exam curriculum PDF (v1.34). The CNCF certification page lists different weights for two domains; see the sources page.

| # | Domain | Weight | Topic docs |
|---|---|---|---|
| 1 | Cluster Setup | 15% | [Network policy](docs/01-cluster-setup/network-policy.md) · [CIS benchmark and kube-bench](docs/01-cluster-setup/cis-benchmark-kube-bench.md) · [Ingress TLS, node metadata, binary verification](docs/01-cluster-setup/ingress-tls-and-node-metadata.md) |
| 2 | Cluster Hardening | 15% | [RBAC](docs/02-cluster-hardening/rbac.md) · [Service accounts and API access](docs/02-cluster-hardening/service-accounts-and-api-access.md) · [Cluster upgrades](docs/02-cluster-hardening/cluster-upgrades.md) |
| 3 | System Hardening | 10% | [Host hardening](docs/03-system-hardening/host-hardening.md) · [AppArmor and seccomp](docs/03-system-hardening/apparmor-and-seccomp.md) |
| 4 | Minimize Microservice Vulnerabilities | 20% | [Security context and PSS](docs/04-microservice-vulnerabilities/security-context-and-pss.md) · [Secrets and encryption at rest](docs/04-microservice-vulnerabilities/secrets-and-encryption-at-rest.md) · [Runtime sandboxes and isolation](docs/04-microservice-vulnerabilities/runtime-sandboxes.md) · [Pod-to-pod encryption (Cilium, Istio)](docs/04-microservice-vulnerabilities/pod-to-pod-encryption.md) |
| 5 | Supply Chain Security | 20% | [Image scanning with Trivy](docs/05-supply-chain-security/image-scanning-trivy.md) · [Image signing with Cosign](docs/05-supply-chain-security/image-signing-cosign.md) · [Admission control](docs/05-supply-chain-security/admission-control.md) · [Base images and Dockerfiles](docs/05-supply-chain-security/base-images-and-dockerfiles.md) · [Static analysis (Kubesec, KubeLinter)](docs/05-supply-chain-security/static-analysis.md) |
| 6 | Monitoring, Logging and Runtime Security | 20% | [Audit logging](docs/06-monitoring-logging-runtime/audit-logging.md) · [Falco](docs/06-monitoring-logging-runtime/falco.md) · [jq for audit data](docs/06-monitoring-logging-runtime/jq-for-audit-data.md) |

Supporting references (read as needed, not separate exam domains):

- [Rego basics](docs/reference/rego-basics.md)
- [OPA Gatekeeper](docs/reference/opa-gatekeeper.md)

## Scope summary

The official curriculum sets the scope for Rego, seccomp and AppArmor:

- **Use** seccomp and AppArmor profiles on pods, and confirm they are loaded and working: expected.
- **Author** them from scratch, or write Rego from scratch: not expected under exam time. Read and adapt them.

Pod-to-pod encryption with Cilium or Istio is named in the official curriculum and is covered. PodSecurityPolicy was removed in Kubernetes 1.25 and is not studied. Kyverno and Gatekeeper are named as tools only in the community guides; their internals are reference material.

## Exam habits

- Read each task fully; note the cluster context and namespace before typing.
- Use `kubectl config use-context <ctx>` when a task names one, and check which node you are on before editing a manifest.
- Flag hard tasks and return; partial credit is per task.
- Verify every change with a command that proves the behavior, not just that the YAML applied.
- Keep a short written record of what you changed when a task has several sections.

## Key ideas at a glance

- NetworkPolicy: ingress, egress, selector AND vs OR, DNS egress, default-deny
- PSA labels, the three modes, restricted requirements, reading a rejection
- SecurityContext fields and what each one blocks
- RBAC: Role vs ClusterRole, bindings, `auth can-i`, dangerous verbs
- Service account auto-mount and TokenRequest
- kube-bench targets and reading remediations; kubelet config field names
- Encryption at rest: provider order, mount, re-encryption
- Audit policy levels, rule order, log format, jq filters
- Trivy severity gating with `--exit-code 1`
- Cosign: sign and verify by digest
- AppArmor and seccomp: apply, load, verify
- Falco: check running, read an alert, edit a rule in `rules.d/`
- Upgrade order: control plane, then nodes

## Common mistakes (summary)

| Mistake | Fix |
|---|---|
| Restricted PSS requirement assumed to include read-only root FS | Restricted does not require it; read-only root FS is good practice |
| Egress policy without DNS | UDP and TCP 53 to kube-dns |
| `policyTypes` lists Egress with no egress rules | Pod loses all outbound |
| Audit exec filter on resource `pods/exec` | Filter `pods` with subresource `exec` |
| `jq '.[]'` on audit.log | Use `jq -c` for each line |
| Trivy passes with findings | Add `--exit-code 1` |
| Verifying Cosign signatures by tag | Verify by digest |
| Seccomp/AppArmor profile missing on the scheduled node | Load the profile on every node the pod can land on |

## Gotchas that cost time

- The API server is a static pod. A YAML error in its manifest takes the API offline; use `crictl` to recover.
- Namespace labels: use `kubernetes.io/metadata.name` for namespace selectors.
- PSA rejections for Deployments appear on the ReplicaSet, not the Deployment.
- NetworkPolicy does nothing without a supporting CNI.
- `runAsNonRoot` fails for images that default to root without an explicit `runAsUser`.

## Adjacent topics (not in the official curriculum)

- **PodSecurityPolicy:** removed in Kubernetes 1.25. Do not study it except to recognize old manifests.
- **Dockerfile security:** partly in scope via base-image footprint; see [Base images](docs/05-supply-chain-security/base-images-and-dockerfiles.md).

## Quick command reference

```bash
# Network
kubectl get netpol -A
kubectl describe netpol <name> -n <ns>

# PSA
kubectl label ns <ns> pod-security.kubernetes.io/enforce=restricted

# RBAC
kubectl auth can-i --list --as=<subject> -n <ns>
kubectl get clusterrolebindings -o wide

# Pod identity and runtime
kubectl exec <pod> -- id
grep Seccomp /proc/self/status        # inside the container

# CIS
sudo kube-bench run --targets control-plane
sudo kube-bench run --targets node

# Images
trivy image --severity CRITICAL --exit-code 1 --ignore-unfixed <image>
cosign verify --key cosign.pub <registry>/<image>@<digest>

# Audit
sudo jq -c 'select(.objectRef.subresource=="exec")' /var/log/kubernetes/audit/audit.log

# Falco
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=100
```

## Document map

```
README.md          this hub
docs/
  01-cluster-setup/                        network policy, CIS, ingress and metadata
  02-cluster-hardening/                    RBAC, service accounts, upgrades
  03-system-hardening/                     host hardening, AppArmor and seccomp
  04-microservice-vulnerabilities/         security context and PSS, secrets, sandboxes
  05-supply-chain-security/                Trivy, Cosign, admission control, base images
  06-monitoring-logging-runtime/           audit logging, Falco, jq
  reference/                               Rego basics, OPA Gatekeeper
  notes/                                   sources and verification
```
