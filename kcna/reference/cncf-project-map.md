# CNCF Project Map

Up: [KCNA hub](../README.md) · Reference

Use this to map a problem to a project name. Project maturity (graduated, incubating, sandbox) changes over time; check [landscape.cncf.io](https://landscape.cncf.io/) for the current status before relying on it.

| Problem | Projects |
|---|---|
| Container orchestration | Kubernetes |
| Container runtime | containerd, CRI-O |
| Pod networking (CNI) | Calico, Cilium, Flannel, Weave Net |
| Cluster DNS | CoreDNS |
| Service proxy | Envoy |
| Service mesh | Istio, Linkerd |
| Ingress controller | Contour, NGINX Ingress |
| Metrics and alerting | Prometheus, Alertmanager |
| Dashboards | Grafana (not a CNCF project; often paired with Prometheus) |
| Logging | Fluentd, Fluent Bit, Loki (Grafana) |
| Tracing and telemetry standard | Jaeger, OpenTelemetry |
| Long-term metrics storage | Thanos, Cortex |
| Storage orchestration | Rook |
| Package management | Helm |
| GitOps | Argo CD, Flux |
| CI/CD pipelines | Tekton, Argo Workflows |
| Policy enforcement | Open Policy Agent (OPA), Kyverno (CNCF project) |
| Runtime threat detection | Falco |
| Container registry | Harbor |
| Serverless | Knative |
| Event-driven autoscaling | KEDA |
| Workload identity | SPIFFE and SPIRE |

## Easy confusions

- Envoy is a proxy. Istio is a service mesh that uses Envoy.
- OpenTelemetry merges OpenTracing and OpenCensus.
- Prometheus does not store data long term by itself.
- Grafana is not a CNCF project; Grafana and Loki come from the same company.
- Helm is a package manager; Argo CD and Flux deploy from Git.

---

<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../LICENSE)</sub>
