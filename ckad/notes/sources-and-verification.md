# Sources and Verification: CKAD

Up: [CKAD README](../README.md) · Notes

This page records where the figures in this section come from and what has and has not been checked.

## Official sources

- CNCF CKAD Exam Curriculum v1.35 (PDF): https://raw.githubusercontent.com/cncf/curriculum/master/CKAD_Curriculum_v1.35.pdf
- CNCF CKAD certification page: https://www.cncf.io/training/certification/ckad/
- Both were checked on 2026-10-05. The repository also holds a v1.37 curriculum, which was not reviewed.

## Curriculum summary

| Domain | Weight |
|---|---|
| Application Design and Build | 20% |
| Application Deployment | 20% |
| Application Observability and Maintenance | 15% |
| Application Environment, Configuration and Security | 25% |
| Services and Networking | 20% |

Container image building, CRDs and operators, Kustomize, Helm, and API deprecations are listed in the curriculum, and each has a topic.

## Not verified

- Passing score, number of tasks, and allowed documentation. The CNCF page checked does not state the passing score or task count.
- The Kubernetes version the exam runs.
- Commands and YAML have not all been executed on a live cluster.
