# Jobs and CronJobs

Up: [CKAD hub](../../README.md) · Domain 1 — Application Design and Build (20%) · Prev: [Multi-container patterns](multi-container-patterns.md) · Next: [Container images](container-images.md)

Not every workload runs forever. Jobs run work to completion, and CronJobs run it on a schedule. This topic covers completions, parallelism, retries and cron syntax.

## Job

Runs pods until a set number of completions succeed.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  completions: 3          # three successful pods required
  parallelism: 2          # at most two at a time
  backoffLimit: 2         # retries before the Job fails
  activeDeadlineSeconds: 300
  template:
    spec:
      restartPolicy: Never   # Never or OnFailure only
      containers:
      - name: pi
        image: perl:5.38
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(200)"]
```

Imperative start:

```bash
kubectl create job pi --image=perl:5.38 --dry-run=client -o yaml -- perl -Mbignum=bpi -wle 'print bpi(200)' > job.yaml
# edit job.yaml to add completions and parallelism, then:
kubectl apply -f job.yaml
```

Check:

```bash
kubectl get job pi
kubectl wait --for=condition=complete job/pi --timeout=300s
kubectl logs job/pi
kubectl get pods -l job-name=pi
```

The `COMPLETIONS` column shows progress, for example `1/3`.

## CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"            # 02:00 daily
  concurrencyPolicy: Forbid        # skip a run if the last one is still going
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: busybox:1.36
            command: ["sh", "-c", "date"]
```

Cron format: `minute hour day-of-month month day-of-week`.

| Expression | Meaning |
|---|---|
| `0 2 * * *` | 02:00 every day |
| `*/5 * * * *` | every 5 minutes |
| `0 0 * * 0` | midnight every Sunday |
| `0 9-17 * * 1-5` | hourly from 09:00 to 17:00, Monday to Friday |

Trigger a run manually:

```bash
kubectl create job backup-manual --from=cronjob/backup
```

## Troubleshooting

| Symptom | Check |
|---|---|
| Job never completes | `describe job`; pod logs; `backoffLimit` reached |
| Pods rejected at creation | `restartPolicy: Always` in the template |
| CronJob never fires | `schedule` syntax; `kubectl describe cronjob` |
| Old pods pile up | History limits not set |

## Common mistakes

- `restartPolicy: Always` in a Job or CronJob template.
- Typo in `schedule`: the CronJob is accepted and never runs.
- Forgetting `parallelism` when the task asks for concurrent runs.

## Quick reference

```bash
kubectl create job <n> --image=<i> -- <cmd>
kubectl create cronjob <n> --image=<i> --schedule="<cron>" -- <cmd>
kubectl create job <n> --from=cronjob/<cj>
kubectl wait --for=condition=complete job/<n>
```
