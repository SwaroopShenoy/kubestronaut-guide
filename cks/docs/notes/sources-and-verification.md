# Sources and Verification: CKS

Up: [CKS README](../../README.md) · Notes

This page records where the figures in this section come from and what has and has not been checked.

## Official sources

- CNCF CKS Exam Curriculum v1.34 (PDF): https://raw.githubusercontent.com/cncf/curriculum/master/CKS_Curriculum%20v1.34.pdf
- CNCF CKS certification page: https://www.cncf.io/training/certification/cks/
- Both were checked on 2026-10-05.

## Curriculum summary

| Domain | Weight (curriculum v1.34) |
|---|---|
| Cluster Setup | 15% |
| Cluster Hardening | 15% |
| System Hardening | 10% |
| Minimize Microservice Vulnerabilities | 20% |
| Supply Chain Security | 20% |
| Monitoring, Logging and Runtime Security | 20% |

## Conflicting figures

The CNCF certification page checked on the same date lists Cluster Setup at 10% and System Hardening at 15%. This guide follows the curriculum PDF. Confirm the current figures with CNCF before planning around either.

## What the curriculum does not state

- It does not name Falco, OPA Gatekeeper, Kyverno, Trivy or cosign. It describes outcomes, and those tools are examples.
- It does not state that writing Rego or Falco rules is required.
- It does not state that writing AppArmor or seccomp profiles from scratch is required. It asks candidates to "appropriately use" kernel hardening tools.

## Not verified

- Passing score, number of tasks, exam duration, allowed documentation, and the Kubernetes version the exam runs.
- Commands and YAML in this guide were written from documented behaviour. They have not all been executed on a live cluster.
- The curriculum version used is v1.34. Newer versions were not reviewed.
