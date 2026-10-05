# Multi-Container Patterns

Up: [CKAD hub](../../CKAD_2026_Complete_Crash_Course.md) · Domain 1 — Application Design and Build (20%) · Next: [Jobs and CronJobs](jobs-and-cronjobs.md)

Four patterns cover most multi-container questions. Each container in a pod shares the network namespace (same IP, `localhost`) and any volumes you mount into both.

## Sidecar

A helper runs next to the app and shares a volume with it. Typical use: ship logs, sync config.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logger
spec:
  containers:
  - name: web
    image: nginx:1.27
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  - name: log-shipper
    image: busybox:1.36
    command: ["sh", "-c", "tail -F /logs/access.log"]
    volumeMounts:
    - name: logs
      mountPath: /logs
      readOnly: true
  volumes:
  - name: logs
    emptyDir: {}
```

nginx writes `access.log` to `/var/log/nginx`; the sidecar reads it through the shared `emptyDir`.

## Init container

Runs to completion before the app containers start. Init containers run in order, and each must succeed.

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ["sh", "-c", "until nslookup db.default.svc.cluster.local; do sleep 2; done"]
  containers:
  - name: app
    image: myapp:1.0
```

Debug init containers by name:

```bash
kubectl logs <pod> -c wait-for-db
kubectl describe pod <pod>      # Init Containers section shows state
```

## Native sidecar (init container with restartPolicy: Always)

An init container with `restartPolicy: Always` starts before the app and keeps running alongside it. It restarts if it fails. This is the form to use when a sidecar must start before the app and must not stop the pod.

```yaml
spec:
  initContainers:
  - name: log-agent
    image: fluent/fluent-bit:3.0
    restartPolicy: Always
    volumeMounts:
    - name: logs
      mountPath: /logs
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  volumes:
  - name: logs
    emptyDir: {}
```

Native sidecars are not new in the current release. Check the feature state for the cluster version in use; the 2026 source guide called this "new in v1.35" and that label is not confirmed.

## Adapter

Normalizes another container's output for a consumer. The app writes a format the platform does not understand; the adapter reads the shared volume and rewrites it.

Structure is the same as the sidecar: shared volume, two containers. The difference is the job of the second container (transform, not ship).

## Ambassador

The app talks to `localhost`; the ambassador container proxies to the real external endpoint. Lets you change the external location without changing the app.

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DB_HOST
      value: localhost
    - name: DB_PORT
      value: "5432"
  - name: db-proxy
    image: <proxy-image>           # forwards localhost:5432 to the real database
    env:
    - name: TARGET
      value: db.example.com:5432
```

## Choosing a pattern

| Need | Pattern |
|---|---|
| Share logs or files with a helper that keeps running | Sidecar (or native sidecar) |
| Prepare data or wait for a dependency before start | Init container |
| Change the format of another container's output | Adapter |
| Make an external service look local | Ambassador |

## Common mistakes

- Mounting the shared volume in one container but not the other.
- Using `readOnly: true` on the producer and getting a write error.
- Putting an init container under `containers:`.
- Looking for logs of an init container with `kubectl logs <pod>` (you need `-c <name>`).

## Practice

1. Add a sidecar that reads a shared log file; confirm the sidecar can see the file.
2. Add an init container that fails until a Service exists; create the Service; watch the pod start.
3. Convert the sidecar to a native sidecar and compare the pod status.

## Quick reference

```bash
kubectl logs <pod> -c <container>
kubectl describe pod <pod> | grep -A6 'Init Containers'
kubectl exec <pod> -c <container> -- ls /shared
```
