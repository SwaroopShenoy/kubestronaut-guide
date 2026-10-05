# Observability

Up: [KCNA hub](../../README.md) · Domain 4 — Cloud Native Architecture (12%) · Prev: [Principles and patterns](principles-and-patterns.md)

The 2025 source guide had a separate Observability domain at 8%. The CNCF KCNA page lists four domains and does not include observability as its own domain. Observability content is kept here because it is commonly tested as a concept; confirm against the current KCNA curriculum.

## Metrics, logs and traces

| Signal | What it is | Typical tool |
|---|---|---|
| Metrics | Numbers over time (requests, CPU, error rates) | Prometheus, with Grafana for dashboards |
| Logs | Discrete event records | Fluentd or Fluent Bit, Loki, Elasticsearch |
| Traces | One request's path across services | Jaeger, OpenTelemetry |

## Prometheus

- Time-series database with a pull model: it scrapes targets on an interval.
- Query language: PromQL.
- Alerting through Alertmanager.
- Metric types: counter (only goes up), gauge (goes up and down), histogram (buckets of observations), summary (client-side quantiles).
- Pushgateway exists for short-lived batch jobs that cannot be scraped.
- Prometheus is not for long-term storage on its own; projects like Thanos or Cortex add that.

## Kubernetes metrics

- **metrics-server**: CPU and memory of pods and nodes. Needed for `kubectl top` and HPA.
- **kube-state-metrics**: metrics about the state of objects (Deployments desired vs available). A different job from metrics-server.

## Logging

- Applications write logs to stdout and stderr. A node-level agent (DaemonSet) or a sidecar collects and forwards them.
- Fluentd is a CNCF graduated project; Fluent Bit is its lightweight sibling.
- Loki indexes labels rather than full log text, so it is cheaper to run.

## Tracing

- A trace is the full journey of a request; a span is one operation within it.
- Context propagation passes trace IDs between services.
- OpenTelemetry is the vendor-neutral standard for traces, metrics and logs. It replaced OpenTracing and OpenCensus.
- Tracing needs instrumentation in the application or a library.

## Distinctions that get tested

- Metrics are aggregated numbers; logs are discrete events; traces follow one request.
- metrics-server (resource usage) vs kube-state-metrics (object state).
- Prometheus pulls; applications expose a metrics endpoint.
