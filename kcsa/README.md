# Part II: KCSA — Kubernetes and Cloud Native Security Associate

KCSA moves from how Kubernetes works to how it can be attacked and defended. It asks you to reason about trust boundaries, identities, policies and threats across the cloud, the cluster and the workload. It is multiple choice, and the questions reward understanding over memorisation.

> **Read first: status and limits**
>
> This book is an independent guide to the CNCF Kubernetes certifications. It is **not** an official CNCF or Linux Foundation resource and is not endorsed by them.
>
> - **Not a complete or current source of truth.** Domains and weights were checked against the official CNCF curriculum PDFs as of 2026-10-05. Exam formats, passing scores, allowed resources, Kubernetes versions and tool behaviour change, and may already differ from what is written here.
> - **Not all commands are tested.** Commands, flags and YAML were written from knowledge and have not all been run on a live cluster. Verify before you rely on them.
> - **Reading aid, not a substitute for practice.** Use it alongside the official curriculum, the official documentation and hands-on practice. Do not use it as your only preparation material.
> - **Open questions are marked.** Items labelled "not verified" or "unconfirmed" are open. Treat them as questions, not facts.
> - **No warranty.** The author accepts no responsibility for exam results, production changes, or decisions made from this content. Check the official sources yourself.

Kubernetes and Cloud Native Security Associate — hub document. Start here; each domain links to topic docs.

## Exam at a glance

| Item | Value |
|---|---|
| Format | Online, proctored, multiple choice; no terminal tasks |
| Duration | Not stated on the official CNCF page; verify |
| Questions | Not stated on the official CNCF page; verify |
| Passing score | Not stated on the official CNCF page; verify |
| Cost | $250, includes one free retake (CNCF page) |
| Prerequisite | None |

## Domains and weights

Weights from the official CNCF exam curriculum PDF.

| # | Domain | Weight | Topic docs |
|---|---|---|---|
| 1 | Overview of Cloud Native Security | 14% | [The 4Cs and shared responsibility](docs/01-overview-cloud-native-security/four-cs-and-shared-responsibility.md) · [Isolation and workload security](docs/01-overview-cloud-native-security/isolation-and-workload-security.md) |
| 2 | Kubernetes Cluster Component Security | 22% | [Control plane security](docs/02-kubernetes-cluster-component-security/control-plane-security.md) · [etcd and node security](docs/02-kubernetes-cluster-component-security/etcd-and-node-security.md) · [Runtime, networking, client and storage](docs/02-kubernetes-cluster-component-security/components-runtime-networking-storage.md) |
| 3 | Kubernetes Security Fundamentals | 22% | [RBAC and ServiceAccounts](docs/03-kubernetes-security-fundamentals/rbac-and-serviceaccounts.md) · [Pod security and NetworkPolicy](docs/03-kubernetes-security-fundamentals/pod-security-and-networkpolicy.md) · [Secrets](docs/03-kubernetes-security-fundamentals/secrets.md) · [Authentication, isolation and segmentation](docs/03-kubernetes-security-fundamentals/authentication-isolation-segmentation.md) |
| 4 | Kubernetes Threat Model | 16% | [Threat model and attack paths](docs/04-kubernetes-threat-model/threat-model-and-attack-paths.md) |
| 5 | Platform Security | 16% | [Supply chain and images](docs/05-platform-security/supply-chain-and-images.md) · [Admission and policy](docs/05-platform-security/admission-and-policy.md) · [Runtime security](docs/05-platform-security/runtime-security.md) · [Mesh, PKI, connectivity and observability](docs/05-platform-security/mesh-pki-connectivity-observability.md) |
| 6 | Compliance and Security Frameworks | 10% | [CIS benchmark and audit logging](docs/06-compliance-and-frameworks/cis-benchmark-and-audit.md) · [Frameworks and regulations](docs/06-compliance-and-frameworks/frameworks-and-regulations.md) · [Threat modeling frameworks and automation](docs/06-compliance-and-frameworks/threat-modeling-and-automation.md) |

Reference:

- [Security tools map](reference/security-tools-map.md)

## Key ideas at a glance

- Explain the 4Cs in order and give one control per layer
- Apply STRIDE to a Kubernetes example
- Name the API server's authentication, authorization and admission steps, in order
- List the kubelet and etcd hardening settings the benchmark asks for
- Explain why RBAC wildcards and `cluster-admin` bindings are risks
- Name the three Pod Security Standards levels and what restricted requires
- Explain why base64 is not encryption and what etcd encryption at rest does
- Describe image signing and why verification must check the digest
- State what Falco detects and what a scanner like Trivy does instead
- Explain what the audit log records and at which levels

## Document map

```
README.md                         this hub
docs/01-overview-cloud-native-security/       4Cs, shared responsibility
docs/02-kubernetes-cluster-component-security/  control plane, etcd, nodes
docs/03-kubernetes-security-fundamentals/     RBAC, pod security, NetworkPolicy, secrets
docs/04-kubernetes-threat-model/              STRIDE, attack paths, response
docs/05-platform-security/                    supply chain, admission, runtime
docs/06-compliance-and-frameworks/            CIS, audit, regulations
reference/                                    security tools map
notes/                                        sources and verification
```
