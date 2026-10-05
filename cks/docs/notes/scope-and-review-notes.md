# Scope and Review Notes

Up: [CKS hub](../../CKS_2026_Complete_Crash_Course.md) · Read this before studying the topic docs.

This file records what was checked, what was corrected from the original course, what is assumed rather than confirmed, and what the syllabus is missing.

## 1. Confirmed against official sources

| Item | Status | Source |
|---|---|---|
| Domain weights: Cluster Setup 10%, Cluster Hardening 15%, System Hardening 15%, Minimize Microservice Vulnerabilities 20%, Supply Chain Security 20%, Monitoring/Logging/Runtime Security 20% | Confirmed (sums to 100) | CNCF CKS certification page |
| Duration 2 hours, performance-based, command line | Confirmed | CNCF CKS certification page |
| Prerequisite: CKA passed at any time before registering; CKA need not be active | Confirmed | CNCF CKS certification page and CNCF sample path PDF |
| Certification valid 2 years | Confirmed | CNCF CKS certification page |
| Exam fee $445 with one free retake | Confirmed at time of check; prices change | CNCF CKS certification page |

Not confirmed in this review:

- Passing score. The original course says 67%. The CNCF page checked did not state it. Verify.
- Number of tasks. The original says 15–20. Not verified.
- The exact competency bullets under each domain. The CNCF curriculum PDF is the authority; the Linux Foundation sample-path PDF checked did not contain it, and the curriculum file on GitHub returned a 404 during this review. Pull the current curriculum PDF from the cncf/curriculum repository and compare each topic in this course against it.

## 2. Your scope question: rego, seccomp, AppArmor

Your assumption is mostly right, with one correction.

- **Writing Rego policies from scratch** — not expected to be a core skill. Reading and adapting a template is the realistic level. See [Rego basics](../reference/rego-basics.md).
- **Writing seccomp and AppArmor profiles from scratch** — same: understand the format and be able to load and apply one. See [AppArmor and seccomp](../03-system-hardening/apparmor-and-seccomp.md).
- **Using them correctly** — this is the part that is expected. Applying `RuntimeDefault` or a `Localhost` profile, applying an AppArmor profile to a pod, confirming it is loaded on the node, and reading why a pod was blocked.

Correction: the earlier course treated "AppArmor and seccomp" as entirely in scope with full authoring examples. The authoring examples are kept for understanding, but they should not be memorized as exam steps.

Gatekeeper is not named in the curriculum summaries checked. It is kept as reference material; see [OPA Gatekeeper](../reference/opa-gatekeeper.md).

## 3. Corrections to the original course

Factual errors fixed in this rewrite. Each one could have cost points or caused a bad change in a real cluster.

| Area | Original claim | Correction |
|---|---|---|
| Domain structure | Six domains with 15/20/20/15/15/15 weights and different names | Official names and weights (see hub) |
| PodSecurityPolicy | Presented as a live control with manifests | Removed in Kubernetes 1.25. Removed from the study path |
| PSS Baseline | Said to require `runAsNonRoot` | Baseline does not; restricted does |
| PSS Restricted | Said to require `readOnlyRootFilesystem` | Restricted does not require it |
| PSS labels | `k run ... --as root` used to demonstrate rejection | `--as` impersonates a user. Use `--dry-run=server` with a violating spec |
| PSS on kube-system | Labeled `enforce=baseline`, then `privileged` in another section | Contradictory; system namespaces should not be enforced at restricted |
| RBAC default SA | Said to be able to create pods and delete secrets | Not true on current clusters. Verify with `kubectl auth can-i --list` |
| RBAC kubelet | ClusterRole with a `namespaces:` field | Invalid RBAC field; kubelets use the Node authorizer |
| RBAC cluster-admin | Recreated the cluster-admin binding | Risks locking out access. Delete specific bindings instead |
| Audit policy | `level: RequestReceivedTimestamp` | Invalid level. Valid: None, Metadata, Request, RequestResponse |
| Audit exec filter | `objectRef.resource == "pods/exec"` | Resource is `pods`; use `objectRef.subresource == "exec"`. Appears in several docs |
| Audit log parsing | `jq '.[]'` on the log | Log is newline-delimited JSON; use `jq -c` or `jq -s` |
| Audit file writing | `sudo cat <<EOF > file` | Redirect runs without sudo; use `sudo tee` |
| Audit field name | `.timestamp` in sample output | Field is `requestReceivedTimestamp` |
| Trivy exit code | Said to exit 1 on findings by default | Exits 0 unless `--exit-code 1` |
| Trivy flags | `--skip-check`, `--remote` | Not valid flags; use `--ignorefile` |
| Trivy outputs | Fabricated CVE examples (e.g., Log4j in nginx) | Removed; practice reading real output |
| Trivy / GitHub Action | `trivy-action@master` | Pin a release tag |
| Kubelet config | `anonymousAuth: false`; `readOnlyPort` said to be 10255 by default | Config field is `authentication.anonymous.enabled`; read-only port is disabled by default in current kubelets |
| Kubelet metadata | iptables on OUTPUT chain to block pods | Pod traffic traverses FORWARD; use FORWARD or pod-level NetworkPolicy |
| Insecure port | Set `--insecure-port=0` | Flag removed from kube-apiserver; the remediation is outdated |
| CIS check IDs | Specific IDs presented as authoritative | Unverified; use IDs printed by kube-bench |
| CIS "top 10 most tested" | Claimed "based on real exam reports" | Unsupported; removed |
| Checksums | Used a SHA256SUMS file with `grep` | Use the per-binary `.sha256` file and `sha256sum --check` |
| Seccomp profile | Example `defaultAction: SCMP_ACT_ERRNO` with a few allowed calls | Would break most containers; blocklist (default allow) is the practical pattern |
| Seccomp claim | RuntimeDefault "blocks ptrace" | Depends on runtime and kernel; verify on the cluster |
| AppArmor annotation | Presented as the only way | Deprecated in 1.30+; use `securityContext.appArmorProfile` |
| Rego syntax | `AND`/`OR` keywords, `(a, b)` membership, `kind == "Pod" { }` blocks, pipe operator `| length` | Not valid Rego |
| Gatekeeper enforcement | `enforcementAction: audit` | Valid values: deny, dryrun, warn |
| Gatekeeper image signing | Mock rego that hard-codes an image | Not a real verification; use an actual verifier |
| ImagePolicyWebhook | `kind: ImagePolicyWebhook` manifest and `defaultAllow: true` | Configured via AdmissionConfiguration; `defaultAllow: true` is fail-open |
| ValidatingAdmissionPolicy | `v1beta1` manifest with no binding | `v1` (GA 1.30) with a binding |
| Cosign | `GITHUB_TOKEN` as identity token; `COSIGN_EXPERIMENTAL` env; "Verification successful!" text | Keyless needs an OIDC token via `id-token: write`; verify by exit code |
| Falco install | `apt-key` and a daemonset URL | Deprecated; use official chart or package |
| Falco outputs | `webhook_url` in outputs.yaml | Use `http_output` in falco.yaml |
| Falco fabricated output | Invented alert text presented as real | Replaced with a pattern to read |
| Ingress | Backend port 443 with TLS terminated at ingress | Backend is usually plain HTTP on the Service port |
| RuntimeClass | Handler `gvisor` | gVisor's runtime handler is `runsc` in typical containerd config |
| "2026 exam trap" | Claimed as a real reported exam gotcha | Unverifiable; the mechanism (DNS egress) is real; the claim was removed |
| Dockerfile security | Listed as "not on CKS" | Base image footprint is listed under supply chain in the summaries checked; moved into scope with a verify note |
| Emojis and "Go crush it" | Decorative | Removed |

## 4. Missing topics (added)

From the syllabus areas that the original course did not cover, or covered only in passing:

- Service account token handling: auto-mount, TokenRequest, legacy token Secrets. [Service accounts and API access](../02-cluster-hardening/service-accounts-and-api-access.md)
- Cluster upgrades and version skew. [Cluster upgrades](../02-cluster-hardening/cluster-upgrades.md)
- Ingress TLS, metadata protection, binary verification. [Ingress, TLS and node metadata](../01-cluster-setup/ingress-tls-and-node-metadata.md)
- Host hardening: attack surface, SSH, kernel modules, firewall, patching. [Host hardening](../03-system-hardening/host-hardening.md)
- Encryption at rest as its own topic, including re-encryption and key rotation. [Secrets and encryption at rest](../04-microservice-vulnerabilities/secrets-and-encryption-at-rest.md)
- RuntimeClass and sandboxes. [Runtime sandboxes](../04-microservice-vulnerabilities/runtime-sandboxes.md)
- Admission control with ValidatingAdmissionPolicy and ImagePolicyWebhook. [Admission control](../05-supply-chain-security/admission-control.md)
- Base images and Dockerfile hygiene. [Base images and Dockerfiles](../05-supply-chain-security/base-images-and-dockerfiles.md)
- Falco split into its own doc with a corrected rule model. [Falco](../06-monitoring-logging-runtime/falco.md)

## 5. Still uncertain — check before exam day

- Exact competency bullets per domain (see section 1).
- Whether Istio mTLS is in scope. The original course had a section on it; no official source checked confirmed it. It is not in the topic docs. Add it back only if the curriculum lists service mesh or pod-to-pod encryption.
- Whether Gatekeeper, Kyverno, and Cosign are named or only implied. The docs treat them as tools to understand, not to master.
- Kubernetes version of the exam environment. Commands here are written for current releases (1.30+); check the version the exam uses.
- Tool versions: Trivy, Falco, Cosign, kube-bench all change flags. Confirm with `--help` on the installed version.
- The exam's list of permitted documentation sites. Check the current list from CNCF before exam day; it determines which references you can open.

## 6. Recommended next steps

1. Pull the current official CKS curriculum PDF and map each bullet to a doc in this folder. Add a row for anything unmapped.
2. Run each "Practice" block on a lab cluster. Read-only study is not enough for CKS.
3. Do the timed drills in the hub after completing Domains 1–3, not before.
