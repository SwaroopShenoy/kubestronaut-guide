# Logs, Debugging and API Deprecations

Up: [CKAD hub](../../CKAD_2026_Complete_Crash_Course.md) · Domain 3 — Application Observability and Maintenance (15%) · Prev: [Probes](probes.md) · Next: [ConfigMaps and Secrets](../04-application-environment-config-security/configmaps-and-secrets.md)

## Logs

```bash
kubectl logs <pod>
kubectl logs <pod> -f                      # follow
kubectl logs <pod> --previous              # last crashed container
kubectl logs <pod> -c <container>          # multi-container pod
kubectl logs <pod> --tail=50 --since=1h
kubectl logs deploy/<name> --all-containers=true
kubectl logs -l app=<label> --prefix       # all matching pods, prefixed
```

## Resource usage

Requires metrics-server.

```bash
kubectl top pods --sort-by=memory
kubectl top pod <pod> --containers
kubectl top nodes
```

## Describe and events

```bash
kubectl describe pod <pod>                  # Events section at the bottom
kubectl get events --sort-by=.lastTimestamp
kubectl get events --field-selector type=Warning
kubectl get events --field-selector involvedObject.name=<pod>
```

## Exit codes and last state

```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}{"\n"}'
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}{"\n"}'
```

`OOMKilled` means the memory limit was reached. Exit code 137 is SIGKILL, often OOM.

## Exec and port-forward

```bash
kubectl exec -it <pod> -- sh
kubectl exec <pod> -c <container> -- env
kubectl port-forward pod/<pod> 8080:8080
kubectl port-forward svc/<svc> 8080:80
```

## Triage order

1. `kubectl get pod` for status and restart count.
2. `kubectl describe pod` for events.
3. `kubectl logs --previous` for the crash.
4. Check the spec: image, command, env, mounts, probes.
5. Exec in if the container still runs.

## API deprecations

The exam may ask you to update a manifest that uses an API version removed from the cluster.

| Old | Current |
|---|---|
| `extensions/v1beta1` Deployment | `apps/v1` |
| `networking.k8s.io/v1beta1` Ingress | `networking.k8s.io/v1` |
| `policy/v1beta1` PodSecurityPolicy | Removed; use Pod Security Admission labels |
| `batch/v1beta1` CronJob | `batch/v1` |

Find the current version for a kind:

```bash
kubectl api-resources | grep -i <kind>
kubectl explain <kind>
```

`kubectl convert` is a separate plugin and is not guaranteed on the exam. Edit the apiVersion and any fields that changed (for example, `apps/v1` Deployments require `spec.selector`).

## Common mistakes

- Reading `logs` without `--previous` after a crash, and seeing only the new container's startup.
- Looking at `logs` for a pod with several containers without `-c`.
- Changing a Deployment's `apiVersion` but not adding the required `spec.selector`.

## Practice

1. Make a container exit with code 3; read the exit code from `jsonpath` and the reason from `describe`.
2. Take a Deployment manifest with `extensions/v1beta1` and convert it to `apps/v1`; apply it.

## Quick reference

```bash
kubectl logs <pod> --previous -c <container>
kubectl describe pod <pod> | sed -n '/Events:/,$p'
kubectl top pods --sort-by=memory
kubectl api-resources | grep -i <kind>
```
