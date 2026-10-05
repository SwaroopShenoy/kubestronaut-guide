# Sources and Verification: CKA

Up: [CKA README](../README.md) · Notes

This page records where the figures in this part come from and what has and has not been checked.

## Official sources

- CNCF CKA Exam Curriculum v1.35 (PDF): https://raw.githubusercontent.com/cncf/curriculum/master/CKA_Curriculum_v1.35.pdf
- CNCF CKA certification page: https://www.cncf.io/training/certification/cka/
- Both were checked on 2026-10-05.

## Curriculum summary

| Domain | Weight |
|---|---|
| Troubleshooting | 30% |
| Cluster Architecture, Installation and Configuration | 25% |
| Services and Networking | 20% |
| Workloads and Scheduling | 15% |
| Storage | 10% |

The curriculum also lists Helm and Kustomize for installing cluster components, extension interfaces (CNI, CSI, CRI), CRDs and operators, and highly available control planes. Each has a chapter in this book.

## Not verified

- Passing score, number of tasks, and allowed documentation sites. The CNCF page checked does not state them.
- The Kubernetes version the exam runs. Chapters target current stable releases; check the version before exam day.
- Commands and YAML were written from documented behaviour and have not all been executed on a live cluster.
- Curriculum v1.35 is the version reviewed.
