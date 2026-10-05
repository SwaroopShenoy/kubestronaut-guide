# KCNA Scope and Review Notes

Up: [KCNA hub](../README.md) · Read before studying the topic docs.

## Official curriculum check

Checked against the official KCNA Exam Curriculum PDF in the cncf/curriculum repository. The four domain weights match (44/28/16/12). The PDF lists Observability under Cloud Native Architecture, so the placement in this guide is correct; the earlier note that observability was "not a listed domain" was wrong. The PDF also lists Troubleshooting under Container Orchestration, Debugging under Application Delivery, and Cloud Native Community and Collaboration under Architecture. Those are now covered.

## Confirmed against the CNCF KCNA page

| Item | Status |
|---|---|
| Domains and weights: Kubernetes Fundamentals 44%, Container Orchestration 28%, Cloud Native Application Delivery 16%, Cloud Native Architecture 12% | Confirmed |
| Cost $250 with one free retake | Confirmed |
| Online, proctored, multiple choice | Confirmed |
| Duration, question count, passing score | Not on the page checked |

Not confirmed (stated only in the source guide or third-party sites):

- 60 questions, 90 minutes, 75% passing score.
- Three-year validity.

## Corrections to the 2025 source guide

| Claim | Correction |
|---|---|
| Five domains: 46/22/16/8/8 | Official page lists four domains, weights 44/28/16/12. Observability was folded into Cloud Native Architecture and flagged |
| Domain weights from third-party sites in one search (44/28/16/12 vs 46/22/16/8/8) | The official page matches 44/28/16/12 |
| "Docker is NOT a runtime for K8s anymore" | Accurate in spirit; the precise fact is that dockershim was removed in 1.24 |
| "kube-proxy doesn't actually proxy traffic in default mode" | Simplified; kube-proxy programs iptables or IPVS rules that the kernel applies |
| "ServiceAccounts are for Pods, not humans" | Correct, kept |
| "Scheduler doesn't start Pods, kubelet does" | Correct, kept |
| "Argo CD vs Flux: Argo is UI and pull-based; Flux is CLI-focused and also pull-based" | Simplified; both are pull-based GitOps tools |
| "Istio recently graduated" and similar maturity claims | Project status changes; the project map now points to the landscape site |
| "Helm 2 vs 3: Tiller removed" | Correct, kept |
| "Prometheus, Fluentd, Jaeger graduated" lists | Kept in the map as examples; verify current status |
| Practice exam, "CNCF project quiz", "KodeKloud" and "Udemy" references | Removed; not part of the curriculum |
| "Post-exam next steps" and "Golden Kubestronaut" | Removed from this doc; belongs in the root guide |
| Emoji headers, "You've got this", "Ganbatte" tone | Removed |

## Gaps

- The source guide did not cover CSI in depth; expanded in [runtimes, networking and interfaces](../docs/02-container-orchestration/runtimes-networking-interfaces.md).
