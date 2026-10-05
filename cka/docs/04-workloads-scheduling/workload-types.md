# Workload Types

Up: [CKA hub](../../README.md) · Domain 4 — Workloads and Scheduling (15%) · Prev: [Deployments and scaling](deployments-and-scaling.md) · Next: [Scheduling](scheduling.md)

## Choosing a type

| Need | Resource |
|---|---|
| Stateless app, scaling, rolling updates | Deployment |
| Stable identity and per-pod storage | StatefulSet |
| One pod per node (agents, log collectors) | DaemonSet |
| Run to completion once | Job |
| Run on a schedule | CronJob |

## StatefulSet

Pods are created in order (`web-0`, `web-1`, ...) and keep their names and their PVCs.

```bash
kubectl scale sts web --replicas=3
kubectl delete sts web --cascade=orphan     # delete the controller, keep the pods
```

A StatefulSet needs a headless Service for stable DNS names (see [Services and DNS](../03-services-networking/services-and-dns.md)). Write the manifest from the docs; there is no imperative shortcut.

## DaemonSet

```bash
kubectl set image ds/logger logger=fluentbit:3.0
kubectl rollout status ds/logger
```

DaemonSet pods run on every node that matches the scheduling constraints. Control-plane nodes are skipped unless they tolerate the control-plane taint.

## Job

```bash
kubectl create job pi --image=perl:5.38 -- perl -Mbignum=bpi -wle 'print bpi(200)'
kubectl wait --for=condition=complete job/pi --timeout=120s
kubectl logs job/pi
```

Job manifest with completions and parallelism:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch
spec:
  completions: 5
  parallelism: 2
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never      # Never or OnFailure only
      containers:
      - name: work
        image: busybox:1.36
        command: ["sh", "-c", "echo done"]
```

## CronJob

```bash
kubectl create cronjob backup --image=busybox:1.36 --schedule="0 2 * * *" -- sh -c 'echo backup'
kubectl create job --from=cronjob/backup backup-manual
kubectl get cronjob
```

Schedule format: `minute hour day-of-month month day-of-week`. `0 2 * * *` runs at 02:00 daily. A typo in `schedule` produces a CronJob that never runs.

## Common mistakes

- `restartPolicy: Always` on a Job pod template (rejected).
- Expecting a StatefulSet to create pods in parallel (it does not, by default).
- Forgetting that deleting a Job removes its pods unless `--cascade=orphan`.

## Quick reference

```bash
kubectl create job <n> --image=<i> -- <cmd>
kubectl create cronjob <n> --image=<i> --schedule="<cron>" -- <cmd>
kubectl get deploy,sts,ds,job,cronjob
```
