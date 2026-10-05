# Scope and Review Notes

Up: [CKS hub](../../README.md) · Read this before studying the topic docs.

This file records what was checked, what was corrected from the original course, what is assumed rather than confirmed, and what the syllabus is missing.

## 1. Confirmed against official sources

| Item | Status | Source |
|---|---|---|
| Domain weights per the official curriculum PDF (CKS v1.34): Cluster Setup 15%, Cluster Hardening 15%, System Hardening 10%, Minimize Microservice Vulnerabilities 20%, Supply Chain Security 20%, Monitoring/Logging/Runtime Security 20% | Confirmed from the curriculum PDF | CNCF CKS Exam Curriculum PDF |
| Weights on the CNCF certification page checked earlier: Cluster Setup 10%, System Hardening 15% | Conflicts with the curriculum PDF for two domains. The curriculum PDF is used here; confirm with CNCF before relying on either | CNCF CKS certification page |
| Duration 2 hours, performance-based, command line | Confirmed | CNCF CKS certification page |
| Prerequisite: CKA passed at any time before registering; CKA need not be active | Confirmed | CNCF CKS certification page and CNCF sample path PDF |
| Certification valid 2 years | Confirmed | CNCF CKS certification page |
| Exam fee $445 with one free retake | Confirmed at time of check; prices change | CNCF CKS certification page |

Not confirmed in this review:

- Passing score. The original course says 67%. The CNCF page checked did not state it. Verify.
- Number of tasks. The original says 15–20. Not verified.
- The exact competency bullets under each domain. The CNCF curriculum PDF is the authority; the Linux Foundation sample-path PDF checked did not contain it, and the curriculum file on GitHub returned a 404 during this review. Pull the current curriculum PDF from the cncf/curriculum repository and compare each topic in this course against it.

## 2. Scope question: rego, Falco, seccomp, AppArmor

What the official CKS curriculum (v1.34 PDF) says:

- System Hardening: "Appropriately use kernel hardening tools such as AppArmor, seccomp". The verb is *use*.
- Monitoring, Logging and Runtime Security: "Perform behavioral analytics", "Detect threats", "Investigate and identify phases of attack", "Ensure immutability of containers at runtime", "Use Kubernetes audit logs". No tool is named.
- Microservice Vulnerabilities: "Use appropriate pod security standards", "Manage kubernetes secrets", "Understand and implement isolation techniques", "Implement Pod-to-Pod encryption (Cilium, Istio)".
- Rego, OPA Gatekeeper, Kyverno and Falco rule authoring are **not mentioned** anywhere in the curriculum.

Conclusion: the official curriculum does not say writing Falco rules, Rego policies, or AppArmor or seccomp profiles is expected. It asks you to use kernel hardening tools and detect threats. Some community blogs say Falco rule writing is almost certain on the exam; those claims are unverified and not from the curriculum. The safe plan is to read and modify an existing rule or profile, and not to rely on writing one from scratch.

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
- Pod-to-pod encryption (Cilium, Istio) is in the official curriculum and is covered in [Pod-to-pod encryption](../04-microservice-vulnerabilities/pod-to-pod-encryption.md).
- Whether Gatekeeper, Kyverno, and Cosign are named or only implied. The docs treat them as tools to understand, not to master.
- Kubernetes version of the exam environment. Commands here are written for current releases (1.30+); check the version the exam uses.
- Tool versions: Trivy, Falco, Cosign, kube-bench all change flags. Confirm with `--help` on the installed version.
- The exam's list of permitted documentation sites. Check the current list from CNCF before exam day; it determines which references you can open.
