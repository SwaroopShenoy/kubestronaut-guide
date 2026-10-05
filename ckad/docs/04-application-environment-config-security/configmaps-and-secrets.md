# ConfigMaps and Secrets

Up: [CKAD hub](../../README.md) · Domain 4 — Application Environment, Configuration and Security (25%) · Next: [SecurityContext, quotas and limits](security-context-quotas-limits.md)

Configuration should live outside the image. This topic shows how to supply settings and secrets to an application as environment variables and as files, and what happens when they change.

## Create

```bash
kubectl create configmap app-config --from-literal=LOG_LEVEL=debug --from-literal=DB_HOST=postgres
kubectl create configmap app-config --from-file=app.conf
kubectl create secret generic db-creds --from-literal=username=admin --from-literal=password='s3cr3t'
kubectl create secret generic tls --from-file=tls.crt --from-file=tls.key
kubectl create secret docker-registry regcred --docker-server=<server> --docker-username=<u> --docker-password=<p>
```

Inspect:

```bash
kubectl get configmap app-config -o yaml
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d; echo
```

## Consume as environment variables

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    envFrom:                       # every key becomes an env var
    - configMapRef:
        name: app-config
    env:
    - name: DB_PASSWORD            # one key
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
```

## Consume as files

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: config
      mountPath: /etc/app
      readOnly: true
    - name: creds
      mountPath: /etc/creds
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
      items:                       # optional: only these keys, with chosen names
      - key: app.conf
        path: app.conf
  - name: creds
    secret:
      secretName: db-creds
      defaultMode: 0400
```

Each key becomes a file under `mountPath`.

## Updates

- Environment variables are read at container start. Changing the ConfigMap does not change running pods; restart them with `kubectl rollout restart deploy/<name>`.
- Volume-mounted ConfigMaps and Secrets update in place after a delay (kubelet sync). They do not update when `subPath` is used.

## Multiple sources in one pod

Pick keys from different sources explicitly rather than relying on merge behavior. Use `items` in volume definitions to choose which keys land as files.

## Common mistakes

- Creating a Secret in one namespace and referencing it from a pod in another. Secrets are namespace-scoped.
- Typo in `key` for `secretKeyRef` or `configMapKeyRef`; the pod fails with `CreateContainerConfigError`.
- Expecting `envFrom` to pick up a ConfigMap change without a restart.
- Putting secrets in a ConfigMap. Use a Secret.

## Quick reference

```bash
kubectl create configmap <n> --from-literal=K=V
kubectl create secret generic <n> --from-literal=K=V
kubectl exec <pod> -- ls /etc/<mount>
kubectl rollout restart deploy/<n>
```

---

Next: [SecurityContext, quotas and limits](security-context-quotas-limits.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
