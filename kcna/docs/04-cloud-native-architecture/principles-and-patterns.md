# Principles and Patterns

Up: [KCNA hub](../../README.md) · Domain 4 — Cloud Native Architecture (12%) · Prev: [Packaging with Helm and Kustomize](../03-cloud-native-application-delivery/packaging-helm-kustomize.md) · Next: [Observability](observability.md)

Cloud native design is a set of habits: small services, declared configuration, and systems built to be replaced. This topic covers the principles and the patterns that follow from them.

## What cloud native means

The CNCF definition describes technologies that let organizations build and run scalable applications in modern, dynamic environments (public, private and hybrid clouds). Containers, service meshes, microservices, immutable infrastructure and declarative APIs are examples.

## Microservices

- Small, independently deployable services, each owning its data.
- Benefits: independent releases, technology choice per service, fault isolation, targeted scaling.
- Costs: distributed-system complexity, network latency, harder testing, data consistency across services.

Communication: synchronous (REST, gRPC) or asynchronous (message queues, event streams).

## The twelve-factor app

1. Codebase: one codebase tracked in version control, many deploys.
2. Dependencies: declared explicitly and isolated.
3. Config: stored in the environment.
4. Backing services: treated as attached resources.
5. Build, release, run: strictly separated.
6. Processes: stateless; state lives in backing services.
7. Port binding: the app exports its service by binding a port.
8. Concurrency: scale out by adding processes.
9. Disposability: fast startup, graceful shutdown.
10. Dev/prod parity: keep environments similar.
11. Logs: treated as event streams written to stdout.
12. Admin processes: one-off tasks run in the same environment.

Twelve-factor applies to monoliths too; microservices benefit from it most.

## Service mesh

A service mesh moves service-to-service concerns (routing, mTLS, retries, timeouts, telemetry) out of application code into sidecar proxies. Examples: Istio (Envoy-based), Linkerd. Costs: extra latency, extra resource use per pod.

## Serverless

Run code without managing servers; event-driven and scales to zero. Knative provides a Kubernetes-based serverless platform (serving and eventing). Cold starts add latency on the first request.

## API gateway

A single entry point for clients: routing, authentication, rate limiting and protocol translation. Examples in Kubernetes: Kong, Ambassador or Emissary, and ingress-based gateways.

## Autoscaling

Covered in [scheduling and scaling](../02-container-orchestration/scheduling-and-scaling.md): HPA, VPA, Cluster Autoscaler, KEDA.
