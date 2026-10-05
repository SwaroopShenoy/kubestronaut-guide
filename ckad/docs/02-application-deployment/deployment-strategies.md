# Deployment Strategies

Up: [CKAD hub](../../CKAD_2026_Complete_Crash_Course.md) · Domain 2 — Application Deployment (20%) · Prev: [Volumes and workload choice](../01-application-design-build/volumes-and-workload-choice.md) · Next: [Probes](../03-application-observability-maintenance/probes.md)

Deployments and Helm/Kustomize basics are in the CKA docs: [Deployments and scaling](../../../cka/docs/04-workloads-scheduling/deployments-and-scaling.md) and [Helm and Kustomize](../../../cka/docs/04-workloads-scheduling/helm-and-kustomize.md). This page covers the strategies CKAD asks you to build by hand.

## Rolling update (built in)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # extra pods allowed during rollout
      maxUnavailable: 0   # never drop below desired count
  minReadySeconds: 10     # pod must stay Ready this long before it counts
```

```bash
kubectl set image deploy/web web=nginx:1.28
kubectl rollout status deploy/web
kubectl rollout undo deploy/web
```

## Recreate

All old pods stop before new ones start. Use when two versions must not run together.

```yaml
spec:
  strategy:
    type: Recreate
```

## Blue/green with a Service selector

Run two Deployments with different `version` labels. The Service selects one.

Write the blue Deployment with the label on the pod template, so the pods carry it:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue
spec:
  replicas: 3
  selector:
    matchLabels: {app: myapp, version: blue}
  template:
    metadata:
      labels: {app: myapp, version: blue}
    spec:
      containers:
      - name: app
        image: myapp:1.0
        ports:
        - containerPort: 8080
```

Expose it with a Service that selects both labels:

```bash
kubectl expose deployment blue --name=app --port=80 --target-port=8080 --selector=app=myapp,version=blue
```

`kubectl label deploy` only labels the Deployment object, not its pods, so the Service would select nothing.

Deploy green the same way with `version: green` and `myapp:2.0`, test it through a temporary Service or port-forward, then switch:

```bash
kubectl set selector svc app 'app=myapp,version=green'
```

Roll back by setting the selector back to `version=blue`. Delete the old Deployment when you are confident.

## Canary by replica ratio

Both Deployments share a label the Service selects (for example `app=web`). Traffic splits roughly by pod count.

```bash
kubectl create deployment web-stable --image=myapp:1.0 --replicas=9
kubectl create deployment web-canary --image=myapp:2.0 --replicas=1
# both pod templates need the label app=web
kubectl scale deploy web-canary --replicas=3       # about 25 percent
kubectl delete deploy web-stable                   # after full promotion
```

This is approximate, since traffic is spread per request, not per pod. Note this in your answer if the task asks for an exact percentage.

## Common mistakes

- Setting labels on the Deployment object but not the pod template; the Service selects nothing.
- Blue/green selector typo (`version=Green`): the Service has no endpoints.
- Forgetting `maxUnavailable: 0` when the task demands zero downtime; the default allows one pod to be down.
- Editing a pod to change its version; the Deployment recreates it with the old image.

## Practice

1. Set a rolling update with `maxUnavailable: 0`; change the image; watch `rollout status`.
2. Build blue/green: two Deployments, one Service; switch; roll back.
3. Build a 9:1 canary; scale to 3:1; promote.

## Quick reference

```bash
kubectl rollout status deploy/<n>
kubectl rollout undo deploy/<n>
kubectl set selector svc <svc> 'version=<v>'
kubectl get endpoints <svc>
```
