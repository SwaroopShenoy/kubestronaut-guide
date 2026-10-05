# CKAD Scope and Review Notes

Up: [CKAD hub](../CKAD_2026_Complete_Crash_Course.md) · Read before studying the topic docs.

## Confirmed against the CNCF CKAD page

| Item | Status |
|---|---|
| Domains and weights: Application Design and Build 20%, Application Deployment 20%, Application Observability and Maintenance 15%, Application Environment, Configuration and Security 25%, Services and Networking 20% | Confirmed |
| Duration about 2 hours, performance-based, online and proctored | Confirmed |
| Cost $445 with one free retake | Confirmed |
| Prerequisite | None stated on the page |

Not confirmed:

- Passing score 66%: stated in both source guides; not on the CNCF page checked.
- Task count (15–20): stated in the source guides only.
- Container image building in scope: included from the source guides; confirm against the curriculum.
- Allowed documentation sites: check the current exam resources page.
- Kubernetes version (1.34 or 1.35): the two source guides disagree; check the exam's version.

## Corrections to the source guides

| Source | Claim | Correction |
|---|---|---|
| 2026 guide | Native sidecars "new in v1.35, tested on exam" | Feature predates v1.35; confirm for the exam version |
| 2026 guide | "Real exam feedback" on egress DNS, CronJob syntax, ConfigMap merging | Anecdotes, unverified; the underlying mechanisms are real and are covered |
| 2026 guide | "ConfigMaps don't auto-reload; pod must restart" | Partly wrong. Volume-mounted ConfigMaps update in place (not with subPath); env vars need a restart. See [ConfigMaps and Secrets](../docs/04-application-environment-config-security/configmaps-and-secrets.md) |
| 2026 guide | `kubectl label deploy blue version=blue` to switch blue/green | Labels the Deployment object, not its pods. Use the pod template. See [Deployment strategies](../docs/02-application-deployment/deployment-strategies.md) |
| 2026 guide | Canary "25%" via replica counts | Approximate. Traffic is per request, not per pod; the doc now says so |
| 2026 guide | `lifecycle.preStop` listed under CrashLoopBackOff fixes | preStop is for graceful shutdown, not crash fixes; moved to [Probes](../docs/03-application-observability-maintenance/probes.md) |
| 2026 guide | `kubectl convert` | Plugin, not guaranteed; replaced with manual apiVersion edits |
| 2026 guide | `kubectl edit pod` "not recommended on exam" | Removed; edit the controller instead |
| 2025 guide | Deployments as `extensions/v1beta1`, Ingress `v1beta1` | Historical; removed in current Kubernetes. Kept as deprecation examples only |
| 2025 guide | "Questions are typically FASTER than CKA" | Opinion; removed |
| 2025 guide | Kubernetes 1.34 | Superseded |
| Both | "Your sensei's prediction", score predictions, "Ganbatte" | Removed |

## Gaps

- Probe and deployment content was thorough; the observability domain's metrics and monitoring coverage was brief. Expand from the official curriculum if the task list shows more.
- Dockerfile content is included as a CKAD topic; confirm its weight in the curriculum.

## Next steps

1. Check each bullet of the current CKAD curriculum against a topic doc.
2. Run each practice block on a lab cluster.
3. Do two timed mocks after finishing the five domains.
