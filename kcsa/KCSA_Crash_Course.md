# KCSA Crash Course

Kubernetes and Cloud Native Security Associate — hub document. Start here; each domain links to topic docs.

Source material: the 2025 crash course (archived in [_archive](_archive/)). This hub supersedes it.

## Exam at a glance

| Item | Value |
|---|---|
| Format | Online, proctored, multiple choice; no terminal tasks |
| Duration | Not on the CNCF page checked (source guide says 90 minutes) |
| Questions | Not on the CNCF page checked (source guide says 60) |
| Passing score | Not on the CNCF page checked (source guide says 75%) |
| Cost | $250, includes one free retake (CNCF page) |
| Prerequisite | None |

## Domains and weights

Weights from the CNCF KCSA certification page. The 2025 guide's estimated weights were wrong and have been replaced.

| # | Domain | Weight | Topic docs |
|---|---|---|---|
| 1 | Overview of Cloud Native Security | 14% | [The 4Cs and shared responsibility](docs/01-overview-cloud-native-security/four-cs-and-shared-responsibility.md) |
| 2 | Kubernetes Cluster Component Security | 22% | [Control plane security](docs/02-kubernetes-cluster-component-security/control-plane-security.md) · [etcd and node security](docs/02-kubernetes-cluster-component-security/etcd-and-node-security.md) |
| 3 | Kubernetes Security Fundamentals | 22% | [RBAC and ServiceAccounts](docs/03-kubernetes-security-fundamentals/rbac-and-serviceaccounts.md) · [Pod security and NetworkPolicy](docs/03-kubernetes-security-fundamentals/pod-security-and-networkpolicy.md) · [Secrets](docs/03-kubernetes-security-fundamentals/secrets.md) |
| 4 | Kubernetes Threat Model | 16% | [Threat model and attack paths](docs/04-kubernetes-threat-model/threat-model-and-attack-paths.md) |
| 5 | Platform Security | 16% | [Supply chain and images](docs/05-platform-security/supply-chain-and-images.md) · [Admission and policy](docs/05-platform-security/admission-and-policy.md) · [Runtime security](docs/05-platform-security/runtime-security.md) |
| 6 | Compliance and Security Frameworks | 10% | [CIS benchmark and audit logging](docs/06-compliance-and-frameworks/cis-benchmark-and-audit.md) · [Frameworks and regulations](docs/06-compliance-and-frameworks/frameworks-and-regulations.md) |

Reference:

- [Security tools map](reference/security-tools-map.md)

## Pre-exam checklist

- [ ] Explain the 4Cs in order and give one control per layer
- [ ] Apply STRIDE to a Kubernetes example
- [ ] Name the API server's authentication, authorization and admission steps, in order
- [ ] List the kubelet and etcd hardening settings the benchmark asks for
- [ ] Explain why RBAC wildcards and `cluster-admin` bindings are risks
- [ ] Name the three Pod Security Standards levels and what restricted requires
- [ ] Explain why base64 is not encryption and what etcd encryption at rest does
- [ ] Describe image signing and why verification must check the digest
- [ ] State what Falco detects and what a scanner like Trivy does instead
- [ ] Explain what the audit log records and at which levels

## Document map

```
KCSA_Crash_Course.md                         this hub
docs/01-overview-cloud-native-security/       4Cs, shared responsibility
docs/02-kubernetes-cluster-component-security/  control plane, etcd, nodes
docs/03-kubernetes-security-fundamentals/     RBAC, pod security, NetworkPolicy, secrets
docs/04-kubernetes-threat-model/              STRIDE, attack paths, response
docs/05-platform-security/                    supply chain, admission, runtime
docs/06-compliance-and-frameworks/            CIS, audit, regulations
reference/                                    security tools map
notes/                                        scope, corrections, open questions
_archive/                                     original 2025 guide, unmodified
```
