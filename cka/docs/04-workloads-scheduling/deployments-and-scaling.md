# Deployments and Scaling

Up: [CKA hub](../../README.md) · Domain 4 — Workloads and Scheduling (15%) · Next: [Workload types](workload-types.md)

A Deployment keeps a set of identical pods running, updates them without downtime, and rolls them back when an update goes wrong. This chapter covers how to create, scale, update and recover a Deployment.

## Create and inspect

```bash
kubectl create deployment web --image=nginx:1.27 --replicas=3
kubectl get deploy,rs,pods -l app=web
kubectl describe deploy web
```

Generate YAML for editing:

```bash
kubectl create deployment web --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml > web.yaml
```

## Resources, probes and env in a manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: {app: web}
  template:
    metadata:
      labels: {app: web}
    spec:
      containers:
      - name: web
        image: nginx:1.27
        resources:
          requests: {cpu: 200m, memory: 256Mi}
          limits: {cpu: 500m, memory: 512Mi}
        env:
        - name: MODE
          value: prod
```

The `selector` must match the template labels, and it cannot be changed after creation.

## Scale

```bash
kubectl scale deploy web --replicas=5
kubectl autoscale deploy web --min=2 --max=10 --cpu-percent=80
kubectl get hpa
```

HPA needs metrics-server to report CPU and memory.

## Update and roll back

```bash
kubectl set image deploy/web web=nginx:1.28
kubectl rollout status deploy/web
kubectl rollout history deploy/web
kubectl rollout undo deploy/web                  # previous revision
kubectl rollout undo deploy/web --to-revision=2
kubectl rollout pause deploy/web
kubectl rollout resume deploy/web
```

Rolling update tuning:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

## Set resources and env without editing YAML

```bash
kubectl set resources deploy web --requests=cpu=200m,memory=256Mi --limits=cpu=500m,memory=512Mi
kubectl set env deploy web MODE=prod
kubectl set serviceaccount deploy web builder
```

## Common mistakes

- Editing a pod that a Deployment owns; the ReplicaSet recreates it. Edit the Deployment.
- Changing the selector on an existing Deployment (the API rejects it).
- Missing `--record`-style history: use `kubectl annotate deploy web kubernetes.io/change-cause="..."` if the task asks for a change reason.

## Quick reference

```bash
kubectl create deploy <n> --image=<i> --replicas=<r> --dry-run=client -o yaml
kubectl set image deploy/<n> <container>=<image>
kubectl rollout undo deploy/<n> [--to-revision=<r>]
```
