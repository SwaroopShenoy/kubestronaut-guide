# Sources and Verification: KCSA

Up: [KCSA README](../README.md) · Notes

This page records where the figures in this section come from and what has and has not been checked.

## Official sources

- CNCF KCSA Exam Curriculum (PDF): https://raw.githubusercontent.com/cncf/curriculum/master/KCSA%20Curriculum.pdf
- CNCF KCSA certification page: https://www.cncf.io/training/certification/kcsa/
- Both were checked on 2026-10-05.

## Curriculum summary

| Domain | Weight |
|---|---|
| Overview of Cloud Native Security | 14% |
| Kubernetes Cluster Component Security | 22% |
| Kubernetes Security Fundamentals | 22% |
| Kubernetes Threat Model | 16% |
| Platform Security | 16% |
| Compliance and Security Frameworks | 10% |

## Curriculum coverage map

| Curriculum item | Where it is covered |
|---|---|
| 4Cs; cloud provider and infrastructure security | [The 4Cs and shared responsibility](../docs/01-overview-cloud-native-security/four-cs-and-shared-responsibility.md) |
| Controls and frameworks | [Frameworks and regulations](../docs/06-compliance-and-frameworks/frameworks-and-regulations.md) |
| Isolation techniques; artifact repository and image security; workload and code security | [Isolation and workload security](../docs/01-overview-cloud-native-security/isolation-and-workload-security.md) |
| API server; kubelet; etcd | [Control plane security](../docs/02-kubernetes-cluster-component-security/control-plane-security.md), [etcd and node security](../docs/02-kubernetes-cluster-component-security/etcd-and-node-security.md) |
| Controller manager; scheduler; container runtime; kube-proxy; pod; container networking; client security; storage | [Runtime, networking, client and storage](../docs/02-kubernetes-cluster-component-security/components-runtime-networking-storage.md) |
| Pod Security Standards and Admissions; network policy | [Pod security and NetworkPolicy](../docs/03-kubernetes-security-fundamentals/pod-security-and-networkpolicy.md) |
| Authentication; isolation and segmentation | [Authentication, isolation and segmentation](../docs/03-kubernetes-security-fundamentals/authentication-isolation-segmentation.md) |
| Secrets | [Secrets](../docs/03-kubernetes-security-fundamentals/secrets.md) |
| Audit logging | [CIS benchmark and audit](../docs/06-compliance-and-frameworks/cis-benchmark-and-audit.md), [Control plane security](../docs/02-kubernetes-cluster-component-security/control-plane-security.md) |
| Trust boundaries and data flow; persistence; denial of service; malicious code execution; attacker on the network; sensitive data; privilege escalation | [Threat model and attack paths](../docs/04-kubernetes-threat-model/threat-model-and-attack-paths.md) |
| Supply chain security; image repository | [Supply chain and images](../docs/05-platform-security/supply-chain-and-images.md) |
| Observability; service mesh; PKI; connectivity | [Mesh, PKI, connectivity and observability](../docs/05-platform-security/mesh-pki-connectivity-observability.md) |
| Admission control | [Admission and policy](../docs/05-platform-security/admission-and-policy.md) |
| Compliance frameworks | [Frameworks and regulations](../docs/06-compliance-and-frameworks/frameworks-and-regulations.md), [CIS benchmark and audit](../docs/06-compliance-and-frameworks/cis-benchmark-and-audit.md) |
| Threat modeling frameworks; supply chain compliance; automation and tooling | [Threat modeling and automation](../docs/06-compliance-and-frameworks/threat-modeling-and-automation.md) |

## Not verified

- Duration, number of questions and passing score. The CNCF page checked does not state them.
- Commands and tool behaviour have not all been executed on a live cluster.

---

<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../LICENSE)</sub>
