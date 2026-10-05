# CKS Complete Crash Course

Certified Kubernetes Security Specialist — hub document. Start here; each domain links to a topic doc with full detail.

Last reviewed: 2026-10-05. Exam facts confirmed against the CNCF certification page. See [scope and review notes](docs/notes/scope-and-review-notes.md) for what is confirmed, what is assumed, and what was corrected from the earlier version.

## Exam at a glance

| Item | Value |
|---|---|
| Format | Performance-based, online, proctored; tasks run on the command line in a live cluster |
| Duration | 2 hours |
| Prerequisite | CKA passed at any time before registering (CKA need not be active) |
| Validity | 2 years |
| Passing score | Not verified in this review (course earlier stated 67%; confirm) |
| Task count | Not verified (course earlier stated 15–20; confirm) |

## Domains and weights

Weights are from the CNCF CKS certification page.

| # | Domain | Weight | Topic docs |
|---|---|---|---|
| 1 | Cluster Setup | 10% | [Network policy](docs/01-cluster-setup/network-policy.md) · [CIS benchmark and kube-bench](docs/01-cluster-setup/cis-benchmark-kube-bench.md) · [Ingress TLS, node metadata, binary verification](docs/01-cluster-setup/ingress-tls-and-node-metadata.md) |
| 2 | Cluster Hardening | 15% | [RBAC](docs/02-cluster-hardening/rbac.md) · [Service accounts and API access](docs/02-cluster-hardening/service-accounts-and-api-access.md) · [Cluster upgrades](docs/02-cluster-hardening/cluster-upgrades.md) |
| 3 | System Hardening | 15% | [Host hardening](docs/03-system-hardening/host-hardening.md) · [AppArmor and seccomp](docs/03-system-hardening/apparmor-and-seccomp.md) |
| 4 | Minimize Microservice Vulnerabilities | 20% | [Security context and PSS](docs/04-microservice-vulnerabilities/security-context-and-pss.md) · [Secrets and encryption at rest](docs/04-microservice-vulnerabilities/secrets-and-encryption-at-rest.md) · [Runtime sandboxes](docs/04-microservice-vulnerabilities/runtime-sandboxes.md) |
| 5 | Supply Chain Security | 20% | [Image scanning with Trivy](docs/05-supply-chain-security/image-scanning-trivy.md) · [Image signing with Cosign](docs/05-supply-chain-security/image-signing-cosign.md) · [Admission control](docs/05-supply-chain-security/admission-control.md) · [Base images and Dockerfiles](docs/05-supply-chain-security/base-images-and-dockerfiles.md) |
| 6 | Monitoring, Logging and Runtime Security | 20% | [Audit logging](docs/06-monitoring-logging-runtime/audit-logging.md) · [Falco](docs/06-monitoring-logging-runtime/falco.md) · [jq for audit data](docs/06-monitoring-logging-runtime/jq-for-audit-data.md) |

Supporting references (read as needed, not separate exam domains):

- [Rego basics](docs/reference/rego-basics.md)
- [OPA Gatekeeper](docs/reference/opa-gatekeeper.md)

## Scope summary

Your question about Rego, seccomp and AppArmor is answered in full in the notes. Short version:

- **Use** seccomp and AppArmor profiles on pods, and confirm they are loaded and working: expected.
- **Author** them from scratch, or write Rego from scratch: not expected under exam time. Read and adapt them.

Other items in the earlier course that are not core: PodSecurityPolicy (removed in 1.25), Istio mTLS (not confirmed by any source checked), Kyverno and Gatekeeper internals (treat as reference).

## Study order

Work through the domains in weight order, and do the practice blocks on a live lab each time.

1. **Foundations:** [RBAC](docs/02-cluster-hardening/rbac.md), [Security context and PSS](docs/04-microservice-vulnerabilities/security-context-and-pss.md), [Network policy](docs/01-cluster-setup/network-policy.md). These show up in almost every task.
2. **Supply chain:** [Trivy](docs/05-supply-chain-security/image-scanning-trivy.md), [Cosign](docs/05-supply-chain-security/image-signing-cosign.md), [Admission control](docs/05-supply-chain-security/admission-control.md).
3. **Cluster and host:** [CIS benchmark](docs/01-cluster-setup/cis-benchmark-kube-bench.md), [Secrets and encryption](docs/04-microservice-vulnerabilities/secrets-and-encryption-at-rest.md), [Host hardening](docs/03-system-hardening/host-hardening.md), [Service accounts](docs/02-cluster-hardening/service-accounts-and-api-access.md), [Cluster upgrades](docs/02-cluster-hardening/cluster-upgrades.md).
4. **Runtime:** [Audit logging](docs/06-monitoring-logging-runtime/audit-logging.md), [jq](docs/06-monitoring-logging-runtime/jq-for-audit-data.md), [Falco](docs/06-monitoring-logging-runtime/falco.md), [AppArmor and seccomp](docs/03-system-hardening/apparmor-and-seccomp.md), [Runtime sandboxes](docs/04-microservice-vulnerabilities/runtime-sandboxes.md), [Ingress and metadata](docs/01-cluster-setup/ingress-tls-and-node-metadata.md), [Base images](docs/05-supply-chain-security/base-images-and-dockerfiles.md).

Time plan:

- **1 week:** one domain per day, then a timed mock on day 7.
- **2 weeks:** week 1 learn and practice each domain; week 2 timed drills.
- **4 weeks:** weeks 1–2 domains with labs; week 3 drills and gap analysis; week 4 final drills and review of the notes.

## Timed task patterns

Most tasks are one of these. Practice each until the first working command takes under two minutes.

| Pattern | Start here |
|---|---|
| Make a pod compliant with restricted PSS | [Security context and PSS](docs/04-microservice-vulnerabilities/security-context-and-pss.md) |
| Default-deny plus specific allows, with DNS | [Network policy](docs/01-cluster-setup/network-policy.md) |
| Remove an over-broad binding and prove it | [RBAC](docs/02-cluster-hardening/rbac.md) |
| Fix CIS FAILs in control-plane or kubelet config | [CIS benchmark](docs/01-cluster-setup/cis-benchmark-kube-bench.md) |
| Enable audit logging and answer a who/what question | [Audit logging](docs/06-monitoring-logging-runtime/audit-logging.md) |
| Scan an image and gate on severity | [Trivy](docs/05-supply-chain-security/image-scanning-trivy.md) |
| Enable encryption at rest | [Secrets and encryption](docs/04-microservice-vulnerabilities/secrets-and-encryption-at-rest.md) |
| Apply a seccomp or AppArmor profile | [AppArmor and seccomp](docs/03-system-hardening/apparmor-and-seccomp.md) |
| Confirm Falco alerts on a trigger | [Falco](docs/06-monitoring-logging-runtime/falco.md) |

## Exam habits

- Read each task fully; note the cluster context and namespace before typing.
- Use `kubectl config use-context <ctx>` when a task names one, and check which node you are on before editing a manifest.
- Flag hard tasks and return; partial credit is per task.
- Verify every change with a command that proves the behavior, not just that the YAML applied.
- Keep a short written record of what you changed when a task has several parts.

## Pre-exam checklist

Knowledge:

- [ ] NetworkPolicy: ingress, egress, selector AND vs OR, DNS egress, default-deny
- [ ] PSA labels, the three modes, restricted requirements, reading a rejection
- [ ] SecurityContext fields and what each one blocks
- [ ] RBAC: Role vs ClusterRole, bindings, `auth can-i`, dangerous verbs
- [ ] Service account auto-mount and TokenRequest
- [ ] kube-bench targets and reading remediations; kubelet config field names
- [ ] Encryption at rest: provider order, mount, re-encryption
- [ ] Audit policy levels, rule order, log format, jq filters
- [ ] Trivy severity gating with `--exit-code 1`
- [ ] Cosign: sign and verify by digest
- [ ] AppArmor and seccomp: apply, load, verify
- [ ] Falco: check running, read an alert, edit a rule in `rules.d/`
- [ ] Upgrade order: control plane, then nodes

Practice:

- [ ] Restricted-compliant pod in under 5 minutes
- [ ] Default-deny with DNS in under 5 minutes
- [ ] Audit query answered in under 3 minutes
- [ ] One CIS FAIL fixed and verified in under 5 minutes

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

## Adjacent topics (not confirmed as exam scope)

- **Istio / mTLS:** the earlier course covered it; no source checked confirmed it is tested. Read the official curriculum before spending time here.
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
CKS_2026_Complete_Crash_Course.md          this hub
docs/
  01-cluster-setup/                        network policy, CIS, ingress and metadata
  02-cluster-hardening/                    RBAC, service accounts, upgrades
  03-system-hardening/                     host hardening, AppArmor and seccomp
  04-microservice-vulnerabilities/         security context and PSS, secrets, sandboxes
  05-supply-chain-security/                Trivy, Cosign, admission control, base images
  06-monitoring-logging-runtime/           audit logging, Falco, jq
  reference/                               Rego basics, OPA Gatekeeper
  notes/                                   scope, corrections and open questions
_archive/                                  the original 12 files, unmodified
```

---

You have the material. Now run the practice blocks on a lab cluster.
