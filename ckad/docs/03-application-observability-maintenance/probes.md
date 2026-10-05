# Probes

Up: [CKAD hub](../../README.md) · Domain 3 — Application Observability and Maintenance (15%) · Prev: [Deployment strategies](../02-application-deployment/deployment-strategies.md) · Next: [Logs, debugging and deprecations](logs-debugging-and-deprecations.md)

Kubernetes cannot know whether an application is healthy unless you tell it how to check. This topic covers liveness, readiness and startup probes, and how to tune them so they help rather than hurt.

## The three probes

| Probe | Question | Failure action |
|---|---|---|
| Liveness | Is the container stuck or dead? | Restart the container |
| Readiness | Can it take traffic now? | Remove the pod from Service endpoints |
| Startup | Has it finished starting? | Holds liveness and readiness until it passes; failure restarts the container |

## Probe methods

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: X-Probe
      value: liveness
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3

readinessProbe:
  tcpSocket:
    port: 5432
  periodSeconds: 5

startupProbe:
  exec:
    command: ["cat", "/tmp/started"]
  periodSeconds: 5
  failureThreshold: 30       # 30 x 5s = 150s to start
```

`exec` runs a command in the container; success is exit code 0. `tcpSocket` checks a port opens. `httpGet` is success for codes 200–399.

## Parameters

| Field | Default | Meaning |
|---|---|---|
| `initialDelaySeconds` | 0 | Wait before the first probe |
| `periodSeconds` | 10 | Interval |
| `timeoutSeconds` | 1 | Per-probe timeout |
| `failureThreshold` | 3 | Failures before the action |
| `successThreshold` | 1 | Successes to count as healthy (must be 1 for liveness and startup) |

## Slow-starting app

Use a startup probe so liveness does not kill the app during boot:

```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  periodSeconds: 10
  failureThreshold: 30
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  periodSeconds: 10
```

## Graceful shutdown

A `preStop` hook gives the app time to finish requests before the container receives SIGTERM:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
```

This is a shutdown hook, not a fix for crashes. Do not confuse it with probe tuning.

## Debugging probes

```bash
kubectl describe pod <pod> | grep -iE 'liveness|readiness|startup|unhealthy'
kubectl get events --field-selector involvedObject.name=<pod>
kubectl exec <pod> -- wget -qO- http://localhost:8080/healthz
kubectl port-forward pod/<pod> 8080:8080    # then curl localhost:8080/healthz
```

Common failures:

- Wrong path or port: the probe returns 404 or connection refused. Check the app's real health endpoint.
- Too aggressive liveness: the container restarts in a loop. Add a startup probe or increase `initialDelaySeconds` and `failureThreshold`.
- Readiness probe pointing at a dependency: the pod stays unready when the dependency is down. Use a check that the app itself can serve.

## Common mistakes

- Using liveness to check a dependency; a database outage then restarts the app repeatedly.
- Setting `successThreshold` above 1 on liveness (the API rejects it).
- Setting the probe port to the Service port instead of the container port.

## Quick reference

```bash
kubectl describe pod <pod> | grep -A3 -i probe
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}{"\n"}'
```
