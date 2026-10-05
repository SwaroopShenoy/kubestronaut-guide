# Security Tools Map

Up: [KCSA hub](../KCSA_Crash_Course.md) · Reference

| Problem | Tool | Category |
|---|---|---|
| Find CVEs in images and filesystems | Trivy, Grype | Scanner |
| Sign and verify images | Cosign (Sigstore), Notary | Supply chain |
| Generate SBOMs | Syft, Trivy | Supply chain |
| Policy as code (Rego) | OPA Gatekeeper | Admission |
| Policy as code (YAML) | Kyverno | Admission |
| Runtime threat detection | Falco | Runtime |
| eBPF enforcement and observability | Tetragon | Runtime |
| CIS benchmark checks | kube-bench | Compliance |
| Cluster configuration audit | Popeye | Compliance |
| Secret storage and rotation | HashiCorp Vault, cloud secret managers | Secrets |
| Secrets from external stores into Kubernetes | External Secrets Operator | Secrets |
| Encrypt Secrets stored in Git | Sealed Secrets | Secrets |
| Network policy enforcement | Calico, Cilium | Network |
| Service mesh mTLS | Istio, Linkerd | Network |
| Private registry with scanning and RBAC | Harbor | Registry |

Tools change frequently; treat this as a map of categories, not a list of current recommendations.
