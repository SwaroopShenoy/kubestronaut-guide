# KCSA Scope and Review Notes

Up: [KCSA hub](../README.md) · Read before studying the topic docs.

## Official curriculum check

Checked against the official KCSA Exam Curriculum PDF in the cncf/curriculum repository. The six domain weights match (14/22/22/16/16/10). The topic lists in each domain were mapped to docs in this guide; the earlier version missed several listed topics (isolation techniques, container runtime, kube-proxy, container networking, client security, storage, authentication, service mesh, PKI, connectivity, persistence, network attacker, threat modeling frameworks, automation and tooling). Those now have docs or sections. Writing policies or rules is not in the curriculum.

## Confirmed against the CNCF KCSA page

| Item | Status |
|---|---|
| Domains and weights: Overview of Cloud Native Security 14%, Kubernetes Cluster Component Security 22%, Kubernetes Security Fundamentals 22%, Kubernetes Threat Model 16%, Platform Security 16%, Compliance and Security Frameworks 10% | Confirmed |
| Cost $250 with one free retake | Confirmed |
| Online, proctored, multiple choice | Confirmed |
| Duration, question count, passing score | Not on the page checked |

Not confirmed: 90 minutes, 60 questions, 75% passing score, three-year validity (all from the source guide or third-party sites).

## Corrections to the 2025 source guide

| Claim | Correction |
|---|---|
| Estimated weights: Overview ~25%, Cluster Component ~25%, Fundamentals ~20%, Threat Model ~15%, Platform ~10%, Compliance ~5% | Replaced with the official 14/22/22/16/16/10. The guide's estimates were not from CNCF and were wrong |
| "KCSA is harder than KCNA; conceptual CKS" | Opinion; removed |
| `--insecure-port=0` as a hardening setting | Flag removed from the API server; obsolete |
| `kube-bench run --targets master` | Target names change between kube-bench versions; check `--help` |
| "Kyverno validationFailureAction: enforce" | Field names vary by Kyverno version; the concept is kept without the field |
| SLSA "L1 to L4" descriptions | Level definitions changed between SLSA versions; doc says to check the current spec |
| Dynamic audit as a backend | AuditSink-based dynamic audit was removed; webhook backend is the current approach |
| "Sysdig, Aqua, Prisma Cloud" product lists | Commercial products; removed from the core topics |
| Third-party mock-exam URL and course vendors | Removed; not part of the curriculum |
| "Pre-exam scheduling strategy: KCNA then KCSA same day" | Removed; scheduling is a personal choice |
| Emoji headers, "Sensei Mode", "Ganbatte" | Removed |

## Gaps

- The source guide covered MITRE ATT&CK for containers only by name; the threat model doc now gives the categories.
- Compliance content is recognition-level; the frameworks doc does not go into legal detail.
