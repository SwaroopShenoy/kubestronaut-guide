# Frameworks and Regulations

Up: [KCSA hub](../../README.md) · Domain 6 — Compliance and Security Frameworks (10%) · Prev: [CIS benchmark and audit](cis-benchmark-and-audit.md)

Regulations and frameworks describe what an organisation must show, not just what it does. This topic maps the main ones to the controls covered earlier in the guide.

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

A framework is a set of requirements; a Kubernetes setting is evidence that you meet section of one. Passing a CIS check does not make a cluster compliant with a regulation on its own.

---

Prev: [CIS benchmark and audit](cis-benchmark-and-audit.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
