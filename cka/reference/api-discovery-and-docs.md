# API Discovery and Documentation Navigation

Up: [CKA hub](../README.md) · Reference

## Find a resource

```bash
kubectl api-resources                       # name, short name, API version, namespaced, kind
kubectl api-resources --namespaced=false
kubectl api-resources | grep -i polic
kubectl api-versions
```

## Learn fields without the docs

```bash
kubectl explain pod.spec.containers
kubectl explain deploy.spec.strategy.rollingUpdate --recursive
```

## Documentation navigation

Know where these live on kubernetes.io before the exam, and check the current allowed-resources list for the exam itself:

- Workloads: Pods, Deployments, StatefulSets, DaemonSets, Jobs, CronJobs
- Services and networking: Service, NetworkPolicy, Ingress, Gateway API
- Storage: PersistentVolumes, StorageClass
- Configuration: ConfigMap, Secret
- Access control: RBAC, ServiceAccount
- Cluster administration: kubeadm, etcd backup, upgrade

Search for the resource name plus the word "example". The example YAML is usually the fastest starting point.

---

<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../LICENSE)</sub>
