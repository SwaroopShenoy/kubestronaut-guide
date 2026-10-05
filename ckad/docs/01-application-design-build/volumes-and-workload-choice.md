# Volumes and Workload Choice

Up: [CKAD hub](../../README.md) · Domain 1 — Application Design and Build (20%) · Prev: [Container images](container-images.md) · Next: [Deployment strategies](../02-application-deployment/deployment-strategies.md)

Choosing the right workload type and the right kind of storage shapes how an application behaves under load and failure. This topic covers both choices and the volume types that support them.

## Pick the workload type from the requirement

| Requirement in the task | Resource |
|---|---|
| Stateless web app, scale, rolling updates | Deployment |
| Stable names and per-pod storage | StatefulSet |
| One pod per node | DaemonSet |
| Run once to completion | Job |
| Run on a schedule | CronJob |

Read the wording: "scheduled", "once", "every node", "stable identity" usually map directly.

## Volume types

```yaml
volumes:
- name: cache
  emptyDir: {}                    # lives as long as the pod
- name: data
  persistentVolumeClaim:
    claimName: data-claim         # see the CKA storage doc
- name: config
  configMap:
    name: app-config
    items:                        # only selected keys
    - key: app.conf
      path: app.conf
- name: certs
  secret:
    secretName: tls-secret
- name: node-logs
  hostPath:
    path: /var/log
    type: Directory               # avoid outside labs
```

`emptyDir` with `medium: Memory` gives a tmpfs-backed volume for sensitive scratch data.

## Mount a volume

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: cache
      mountPath: /var/cache/app
    - name: config
      mountPath: /etc/app
      readOnly: true
  volumes:
  - name: cache
    emptyDir: {}
  - name: config
    configMap:
      name: app-config
```

## Read-only root filesystem with writable paths

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

## Common mistakes

- Mounting a volume under the wrong container name in `volumeMounts`.
- Using `hostPath` when the task asks for ephemeral shared storage (use `emptyDir`).
- Forgetting that an emptyDir is wiped when the pod is deleted.

## Quick reference

```bash
kubectl exec <pod> -c <container> -- ls <mountPath>
kubectl describe pod <pod> | grep -A3 Mounts
```
