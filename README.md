# Kubestronaut Crash Course

> **Read first: status and limits**
>
> This is a personal study guide, written to organise notes for the CNCF Kubernetes certifications. It is **not** an official CNCF or Linux Foundation resource and is not endorsed by them.
>
> - **Not a complete or current source of truth.** Domains and weights were checked against the official CNCF curriculum PDFs as of 2026-10-05. Exam formats, passing scores, allowed resources, Kubernetes versions and tool behaviour change, and may already differ from what is written here.
> - **Not all commands are tested.** Commands, flags and YAML were written from knowledge and have not all been run on a live cluster. Verify before you rely on them.
> - **Reading aid, not a course.** Use it alongside the official curriculum, the official documentation and hands-on practice. Do not use it as your only preparation material.
> - **Open questions are marked.** Items labelled "not verified" or "unconfirmed" are open. Treat them as questions, not facts.
> - **No warranty.** The author accepts no responsibility for exam results, production changes, or decisions made from this content. Check the official sources yourself.

Root document for the five CNCF Kubernetes certifications. Each certification has its own folder with a hub document, topic docs by exam domain, reference material, and scope notes.

Kubestronaut is the designation for holding all five: KCNA, KCSA, CKA, CKAD and CKS. CKS requires that you have passed the CKA at some point before registering; the CKA does not need to be active (CNCF page).

## Certifications

| Cert | Hub | Domains and weights |
|---|---|---|
| KCNA — Kubernetes and Cloud Native Associate | [kcna/README.md](kcna/README.md) | Fundamentals 44% · Container Orchestration 28% · Application Delivery 16% · Cloud Native Architecture 12% |
| KCSA — Kubernetes and Cloud Native Security Associate | [kcsa/README.md](kcsa/README.md) | Cloud Native Security 14% · Cluster Component Security 22% · Security Fundamentals 22% · Threat Model 16% · Platform Security 16% · Compliance 10% |
| CKA — Certified Kubernetes Administrator | [cka/README.md](cka/README.md) | Troubleshooting 30% · Cluster Architecture 25% · Services and Networking 20% · Workloads and Scheduling 15% · Storage 10% |
| CKAD — Certified Kubernetes Application Developer | [ckad/README.md](ckad/README.md) | Application Design and Build 20% · Application Deployment 20% · Observability and Maintenance 15% · Environment, Configuration and Security 25% · Services and Networking 20% |
| CKS — Certified Kubernetes Security Specialist | [cks/README.md](cks/README.md) | Cluster Setup 15% · Cluster Hardening 15% · System Hardening 10% · Minimize Microservice Vulnerabilities 20% · Supply Chain 20% · Monitoring, Logging and Runtime 20% |

Weights are from the official CNCF exam curriculum PDFs. Exam facts such as passing scores and question counts are marked as unverified inside each hub where the official page did not state them.

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
