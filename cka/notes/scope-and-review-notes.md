# CKA Scope and Review Notes

Up: [CKA hub](../CKA_2026_Complete_Crash_Course.md) · Read before studying the topic docs.

## Confirmed against the CNCF CKA page

| Item | Status |
|---|---|
| Domains and weights: Troubleshooting 30%, Cluster Architecture/Installation/Configuration 25%, Services and Networking 20%, Workloads and Scheduling 15%, Storage 10% | Confirmed |
| Duration 2 hours, performance-based, online and proctored | Confirmed |
| Cost $445 with one free retake | Confirmed |
| Prerequisite | None stated on the page |

Not confirmed:

- Passing score 66%: stated in both source guides; not on the CNCF page checked.
- Task count (15–20): stated in both source guides; not on the CNCF page.
- Allowed documentation sites and any extra browser tabs: stated in the source guides; check the current exam resources page.
- Kubernetes version (v1.35): the 2026 guide targets it; check the version the exam runs.
- Validity period: not verified for CKA.

## Corrections to the source guides

| Source | Claim | Correction |
|---|---|---|
| 2026 guide | `kubeadm certs renew kubelet-client-current` | Not a kubeadm cert name. Kubelet client certs rotate on the node; use `kubeadm certs renew all` for control-plane certs |
| 2026 guide | "Watch for `--kubeconfig vs --kubelet-config`" as an error | Invented example; removed |
| 2026 guide | etcd restore with `etcdctl snapshot restore` | Current etcd uses `etcdutl snapshot restore`; snapshot save stays `etcdctl` |
| 2026 guide | Init containers as sidecars marked "NEW in v1.35 (July 2026)" | Native sidecars (init containers with `restartPolicy: Always`) were introduced earlier and are not new in v1.35. Confirm the feature state for your version |
| 2026 guide | "cgroup v2 — all nodes now use cgroup v2"; "exam question: kubelet metrics" | Unsupported exam claim; removed |
| 2026 guide | CEL validation under a CRD `spec.validationRules` | Not a real field. Use `x-kubernetes-validations` in the OpenAPI schema |
| 2026 guide | "Gateway API graduated to v1 in K8s 1.32" presented as built-in | Gateway API is a separate CRD set, not built into Kubernetes; install per cluster |
| 2026 guide | Kustomize `commonLabels` | Deprecated; use `labels` |
| 2026 guide | `kubectl create ... --record`-style history | Removed flag; use `kubernetes.io/change-cause` annotation |
| 2026 guide | Imperative `kubectl top` requires "metrics server" | Correct, kept |
| 2025 guide | Weights 25/15/20/10/30 in a different order; Helm and Kustomize marked "NEW 2025" | Weights consistent with CNCF; kept |
| 2025 guide | Kubernetes 1.32 examples | Superseded; kept only in archive |
| Both | "PSI remote browser, 6 different clusters" | Not verified; not carried into the hub |
| Both | Emoji headers, "Sensei mode", "Ganbatte" | Removed |

## Gaps in the source guides

- Helm and Kustomize had no scope statement; treated as in scope per the 2025 update, confirm against the curriculum.
- Troubleshooting was thin on the kubelet and static pod failure modes; expanded in [control plane and nodes](../docs/01-troubleshooting/control-plane-and-nodes.md).
- No storage troubleshooting detail; added in [storage](../docs/05-storage/storage.md).

## Next steps

1. Pull the current CKA curriculum from the cncf/curriculum repository and check every bullet against a topic doc.
2. Build a lab cluster (kubeadm on three VMs, or kind for workload tasks) and run each practice block.
3. Do timed mocks after troubleshooting and cluster architecture are both comfortable.
