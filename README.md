# Kubestronaut Crash Course

Root document for the five CNCF Kubernetes certifications. Each certification has its own folder with a hub document, topic docs by exam domain, reference material, and scope notes.

Kubestronaut is the designation for holding all five: KCNA, KCSA, CKA, CKAD and CKS. CKS requires that you have passed the CKA at some point before registering; the CKA does not need to be active (CNCF page).

## Certifications

| Cert | Hub | Domains and weights |
|---|---|---|
| KCNA — Kubernetes and Cloud Native Associate | [kcna/KCNA_Crash_Course.md](kcna/KCNA_Crash_Course.md) | Fundamentals 44% · Container Orchestration 28% · Application Delivery 16% · Cloud Native Architecture 12% |
| KCSA — Kubernetes and Cloud Native Security Associate | [kcsa/KCSA_Crash_Course.md](kcsa/KCSA_Crash_Course.md) | Cloud Native Security 14% · Cluster Component Security 22% · Security Fundamentals 22% · Threat Model 16% · Platform Security 16% · Compliance 10% |
| CKA — Certified Kubernetes Administrator | [cka/CKA_2026_Complete_Crash_Course.md](cka/CKA_2026_Complete_Crash_Course.md) | Troubleshooting 30% · Cluster Architecture 25% · Services and Networking 20% · Workloads and Scheduling 15% · Storage 10% |
| CKAD — Certified Kubernetes Application Developer | [ckad/CKAD_2026_Complete_Crash_Course.md](ckad/CKAD_2026_Complete_Crash_Course.md) | Application Design and Build 20% · Application Deployment 20% · Observability and Maintenance 15% · Environment, Configuration and Security 25% · Services and Networking 20% |
| CKS — Certified Kubernetes Security Specialist | [cks/CKS_2026_Complete_Crash_Course.md](cks/CKS_2026_Complete_Crash_Course.md) | Cluster Setup 10% · Cluster Hardening 15% · System Hardening 15% · Minimize Microservice Vulnerabilities 20% · Supply Chain 20% · Monitoring, Logging and Runtime 20% |

Weights are from the CNCF certification pages. Exam facts such as passing scores and question counts are marked as unverified inside each hub where the official page did not state them.

## Shared material

Several certs overlap heavily:

- CKA and CKAD share Deployments, Services, NetworkPolicy, Ingress, Helm and Kustomize. The CKAD docs link to the CKA docs for these.
- CKA, CKAD and CKS share RBAC and NetworkPolicy. CKS goes deeper into hardening; CKA and CKAD cover the everyday use.
- KCNA and KCSA are conceptual; their topics are the same ideas CKA, CKAD and CKS test hands-on.

## Suggested order

One common order is KCNA, then CKA, CKAD, KCSA, and CKS last. This is a suggestion, not a requirement, except that CKS needs CKA passed first. Check current prerequisites before booking.

## Folder layout

Each cert folder follows the same structure:

- `<CERT>_Crash_Course.md` — hub: exam facts, domain table, reading order, checklist
- `docs/` — one folder per exam domain, with topic docs that each link up to the hub and to the previous and next topic
- `reference/` — tool and command references
- `notes/` — what was confirmed, what the source guides got wrong, and what remains open
- `_archive/` — the original guides, unmodified
