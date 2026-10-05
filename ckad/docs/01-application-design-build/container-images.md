# Container Images

Up: [CKAD hub](../../README.md) · Domain 1 — Application Design and Build (20%) · Prev: [Jobs and CronJobs](jobs-and-cronjobs.md) · Next: [Volumes and workload choice](volumes-and-workload-choice.md)

An application starts as an image. This topic covers the Dockerfile instructions that shape an image, multi-stage builds that keep images small, and habits that keep them safe.

The CKAD curriculum lists defining and building container images. Confirm current scope against the official curriculum.

## Dockerfile essentials

| Instruction | Purpose |
|---|---|
| `FROM` | Base image (pin a version) |
| `WORKDIR` | Working directory for later steps |
| `COPY` / `ADD` | Copy files; `ADD` also extracts archives and fetches URLs (prefer `COPY`) |
| `RUN` | Run a command at build time; each one creates a layer |
| `ENV` / `ARG` | Build-time or runtime environment |
| `EXPOSE` | Documents a port; does not publish it |
| `USER` | User for later instructions and the container process |
| `ENTRYPOINT` / `CMD` | Main process; `CMD` is overridable, `ENTRYPOINT` is the fixed command |

`CMD` and `ENTRYPOINT` in exec form (`["app", "arg"]`) are preferred over shell form.

## Multi-stage build

Build with a full toolchain, ship only the result.

```dockerfile
FROM golang:1.23 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

## Practices that exam tasks check

- Pin base image tags; avoid `latest`.
- Run as a non-root `USER` so `runAsNonRoot` passes.
- Combine `RUN` steps that install and clean up in one layer.
- Put rarely-changing steps (dependencies) before frequently-changing ones (source) to use the cache.
- Use `.dockerignore` for `.git`, build output and secret files.

## Build and run

```bash
docker build -t myapp:1.0 .
docker build -t myapp:1.0 -f Dockerfile.prod .
docker run --rm -p 8080:8080 myapp:1.0
docker inspect --format '{{.Config.User}}' myapp:1.0
```

Load into a kind cluster for testing without a registry:

```bash
kind load docker-image myapp:1.0
```

Use the image in a pod with `imagePullPolicy: IfNotPresent` so the cluster uses the local copy.

## Common mistakes

- Using `latest` and expecting a rebuilt image to roll out (the tag is unchanged; a Deployment sees no change).
- Writing `ENTRYPOINT` in shell form and then wondering why signals do not reach the app.
- Leaving `USER` unset and then setting `runAsNonRoot` in the pod (the pod fails to start).

## Quick reference

```bash
docker build -t <name>:<tag> .
docker inspect --format '{{.Config.User}}' <image>
kind load docker-image <image>
```

---

Prev: [Jobs and CronJobs](jobs-and-cronjobs.md) · Next: [Volumes and workload choice](volumes-and-workload-choice.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
