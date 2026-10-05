# Base Images and Dockerfiles

Up: [CKS hub](../../CKS_2026_Complete_Crash_Course.md) · Domain 5 — Supply Chain Security (20%) · Prev: [Admission control](admission-control.md) · Next: [Audit logging](../06-monitoring-logging-runtime/audit-logging.md)

This doc is **new**. The original course said Dockerfile security is out of scope. The curriculum summaries we checked list minimizing base image footprint under Supply Chain Security, so this topic is probably in scope. Confirm against the official curriculum.

## Why it matters

Every package in an image is attack surface and a source of CVEs. A smaller image with fewer packages has fewer findings, and a non-root default limits what an exploit can do.

## Checklist

- Use a minimal base (distroless, alpine, or a vendor's minimal image). Pin a version.
- Use a multi-stage build so compilers and build tools do not ship.
- Set a non-root `USER` in the image so `runAsNonRoot` can pass without a runtime override.
- Do not put secrets in `ARG`, `ENV`, or `COPY`; they persist in image history.
- Remove package-manager caches in the same `RUN` layer.
- Use `.dockerignore` so `.git`, `.env`, and key files do not enter the build context.

## Example

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.23 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

- Build stage has the toolchain; the final image has only the binary.
- `:nonroot` distroless variant runs as a non-root UID.
- The explicit `USER` numeric ID helps Kubernetes verify `runAsNonRoot` without guessing from a name.

Check the result:

```bash
docker build -t app:1.0 .
docker inspect --format '{{.Config.User}}' app:1.0
trivy image --severity HIGH,CRITICAL app:1.0
```

## Anti-patterns

```dockerfile
ENV DB_PASSWORD=supersecret          # persists in image history
RUN apt-get update                   # cache left in the layer
RUN apt-get install -y curl          # build tools shipped to production
FROM ubuntu:latest                   # unpinned and full distro
# no USER line                       # runs as root
```

## Linting

```bash
hadolint Dockerfile
```

hadolint flags missing `USER`, unpinned tags, and `apt-get` without `--no-install-recommends`. Run it in CI next to Trivy.

## Practice

1. Take a Dockerfile that runs as root and ships a compiler; rewrite it as multi-stage with a non-root user.
2. Compare the Trivy finding counts before and after.

## Quick reference

```bash
docker inspect --format '{{.Config.User}}' <image>
hadolint Dockerfile
```
