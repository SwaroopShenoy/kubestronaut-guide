# Frameworks and Regulations

Up: [KCSA hub](../../KCSA_Crash_Course.md) · Domain 6 — Compliance and Security Frameworks (10%) · Prev: [CIS benchmark and audit](cis-benchmark-and-audit.md)

The exam tests recognition: what each framework is for and which Kubernetes controls map to it. It does not expect legal detail.

| Framework | Scope | Kubernetes-relevant themes |
|---|---|---|
| CIS Benchmarks | Hardening guidance for OSes and Kubernetes | Control plane, node, and policy checks |
| PCI DSS | Payment card data | Encryption, access control, logging, network segmentation |
| HIPAA | US health information | Encryption in transit and at rest, access controls, audit trails |
| SOC 2 | Service organization trust criteria | Security, availability, confidentiality; access reviews; monitoring |
| GDPR | EU personal data | Data protection by design, access limits, breach response, data residency |
| NIST SP 800-190 | Application container security | Image, registry, orchestrator, and runtime guidance |

## Mapping controls to frameworks

| Control | Frameworks it supports |
|---|---|
| RBAC least privilege | PCI DSS, HIPAA, SOC 2 |
| Encryption at rest (etcd) | PCI DSS, HIPAA, GDPR |
| Audit logging | All of the above |
| NetworkPolicy segmentation | PCI DSS, SOC 2 |
| Image scanning and signing | SOC 2, NIST SP 800-190 |

A framework is a set of requirements; a Kubernetes setting is evidence that you meet part of one. Passing a CIS check does not make a cluster compliant with a regulation on its own.
