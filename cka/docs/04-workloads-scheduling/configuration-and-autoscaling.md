# Configuration, Autoscaling and Self-Healing

Up: [CKA README](../../README.md) · Domain 4 — Workloads and Scheduling (15%) · Prev: [Helm and Kustomize](helm-and-kustomize.md) · Next: [Storage](../05-storage/storage.md)

Applications need settings, the right number of replicas, and a way to recover on their own. This topic covers the curriculum items for configuring applications with ConfigMaps and Secrets, workload autoscaling, pod admission limits, and self-healing primitives.

Official curriculum topics: "Use ConfigMaps and Secrets to configure applications", "Configure workload autoscaling", "Understand the primitives used to create robust, self-healing, application deployments", "Configure Pod admission and scheduling (limits, node affinity, etc.)".

## ConfigMaps and Secrets

```bash
kubectl create configmap app-config --from-literal=LOG_LEVEL=debug --from-file=app.conf
kubectl create secret generic db-creds --from-literal=username=admin --from-literal=password='s3cr3t'
```

Consume them as environment variables or as files:

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    envFrom:
    - configMapRef:
        name: app-config
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
    volumeMounts:
    - name: config
      mountPath: /etc/app
  volumes:
  - name: config
    configMap:
      name: app-config
```

Environment variables are read when the container starts. Files from a mounted ConfigMap or Secret are updated in place after a delay, unless the mount uses `subPath`. A Deployment can be restarted to pick up changes:

```bash
kubectl rollout restart deployment/<name>
```

More examples are in [ConfigMaps and Secrets](../../../ckad/docs/04-application-environment-config-security/configmaps-and-secrets.md) in the CKAD section.

## Horizontal Pod Autoscaler

The HPA changes the replica count of a Deployment, StatefulSet or ReplicaSet based on observed metrics. It needs metrics-server for CPU and memory, and the target pods must set resource requests, because utilisation is measured against the request.

```bash
kubectl autoscale deployment web --min=2 --max=10 --cpu-percent=80
kubectl get hpa
kubectl describe hpa web
```

The same object in YAML:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

Scaling behaviour can be tuned with a `behavior` block, for example a stabilisation window to stop replicas flapping.

If the HPA shows `<unknown>` for its target, metrics-server is missing or the pods have no CPU request:

```bash
kubectl get pods -n kube-system | grep metrics-server
kubectl top pods
```

## Vertical scaling and cluster scaling (concepts)

- The Vertical Pod Autoscaler adjusts requests and limits and usually restarts pods.
- The Cluster Autoscaler adds or removes nodes when pods cannot be scheduled.
- Do not drive HPA and VPA from the same metric on one workload.

## Limits that admit or reject pods

Requests drive scheduling; limits cap use. A namespace can enforce defaults and totals:

```bash
kubectl create quota compute --hard=requests.cpu=4,requests.memory=8Gi,pods=10 -n dev
kubectl describe quota compute -n dev
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: dev
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 200m
      memory: 256Mi
```

A pod that would exceed the quota is rejected at creation with an `exceeded quota` message.

## Self-healing primitives

| Primitive | What it does |
|---|---|
| ReplicaSet (through a Deployment) | Replaces pods that disappear |
| `restartPolicy` | Restarts failed containers in a pod (`Always`, `OnFailure`, `Never`) |
| Liveness probe | Restarts a container that is stuck |
| Readiness probe | Removes a pod from Service endpoints until it is ready |
| Startup probe | Gives slow applications time to start before liveness applies |
| PodDisruptionBudget | Limits how many replicas voluntary disruptions, such as a node drain, may remove |

Probe syntax is in [Probes](../../../ckad/docs/03-application-observability-maintenance/probes.md).

A PodDisruptionBudget:

```bash
kubectl create poddisruptionbudget web-pdb --selector=app=web --min-available=2
kubectl get pdb
```

A drain that would violate the budget waits or fails until enough replicas are available elsewhere.

## Common mistakes

- Creating an HPA for pods with no CPU request, so it never scales.
- Expecting a changed ConfigMap to update environment variables in running pods.
- Setting `maxReplicas` below the replica count the application already needs.
- Draining a node that holds the only replica of a workload protected by a strict PodDisruptionBudget.

---

Prev: [Helm and Kustomize](helm-and-kustomize.md) · Next: [Storage](../05-storage/storage.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
