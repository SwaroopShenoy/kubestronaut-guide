# Troubleshooting Method

Up: [CKA hub](../../CKA_2026_Complete_Crash_Course.md) · Domain 1 — Troubleshooting (30%) · Next: [Control plane and nodes](control-plane-and-nodes.md)

Troubleshooting is the largest CKA domain. Use one method every time instead of guessing.

## The loop

1. **Scope the symptom.** What is failing: the cluster API, a node, a pod, a service, DNS?
2. **Find the layer.** Control plane, node, pod, network, storage.
3. **Read the evidence.** `describe` events, logs, node conditions.
4. **Fix the smallest thing** that explains the evidence.
5. **Verify** with a command that tests the behavior the task asked for.

## Start with the cluster

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
kubectl get events -A --sort-by=.lastTimestamp | tail -30
```

If `kubectl` itself fails with "connection refused" or a certificate error, skip to [control plane and nodes](control-plane-and-nodes.md). Everything else needs a working API server.

## Pod states and what they usually mean

| State | First place to look |
|---|---|
| Pending | `describe` Events: insufficient CPU/memory, taints, unbound PVC, nodeSelector mismatch |
| ImagePullBackOff / ErrImagePull | Image name or tag, registry auth (`imagePullSecrets`), registry reachability |
| CrashLoopBackOff | `logs --previous`, exit code, command/args, probes killing the container |
| OOMKilled | Memory limit too low or a leak; `describe` shows last state reason |
| Running but not Ready | Readiness probe failing; `describe` shows probe failures |
| Unknown | Node lost contact; check the node |

```bash
kubectl describe pod <pod> | sed -n '/Events:/,$p'
kubectl logs <pod> --previous
kubectl logs <pod> -c <container>          # multi-container pods
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.exitCode}{"\n"}'
```

## Pods that are Pending

```bash
kubectl describe pod <pod> | grep -A5 Events
kubectl get pod <pod> -o yaml | grep -A4 -E 'nodeSelector|tolerations|affinity|resources'
kubectl describe node <node> | grep -A6 -E 'Taints|Allocated resources'
kubectl get pvc -n <ns>
```

Typical fixes: lower the request, add a toleration or remove a bad taint (`kubectl taint node <n> key:NoSchedule-`), correct the nodeSelector label, bind or create the PVC.

## Resource pressure

```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory
kubectl describe node <node> | grep -A6 Conditions
```

If `kubectl top` fails, metrics-server is missing or broken. Check `kubectl get pods -n kube-system | grep metrics`.

## Workflow for a task

1. Read the task fully and note the context and namespace.
2. Run one `get` and one `describe` on the failing object.
3. Form one hypothesis from the Events and logs.
4. Fix it in the smallest possible edit (`kubectl edit`, or patch a single field).
5. Run the verification command the task names. If none, run one yourself.

## Common mistakes

- Reading only `describe` headers and missing the Events block at the bottom.
- Fixing a symptom in a Deployment when the pod template in the Deployment is the cause (fix the template, not the pod).
- Editing a pod that a controller will recreate. Edit the Deployment, StatefulSet or DaemonSet instead.
- Declaring success without testing from inside the cluster.

## Quick reference

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -30
kubectl describe <kind> <name>
kubectl logs <pod> --previous
kubectl get endpoints <svc>
```
