# Monitoring Resource Usage and Reading Logs

Up: [CKA README](../../README.md) · Domain 1 — Troubleshooting (30%) · Prev: [Services, DNS and networking](services-dns-and-networking.md) · Next: [RBAC](../02-cluster-architecture-installation-config/rbac.md)

Two curriculum items sit under troubleshooting: monitor cluster and application resource usage, and manage and evaluate container output streams. Both come down to knowing which command shows which signal.

Official curriculum topics: "Monitor cluster and application resource usage" and "Manage and evaluate container output streams".

## Resource usage with metrics-server

`kubectl top` reads from metrics-server. If it fails, check that first.

```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory
kubectl top pod <pod> --containers
kubectl get pods -n kube-system | grep metrics-server
```

A common error is `Metrics API not available`, which means metrics-server is missing, not ready, or cannot reach the kubelets. Check its logs:

```bash
kubectl logs -n kube-system deployment/metrics-server
```

Node-level checks when `kubectl` is not enough:

```bash
df -h               # disk
free -h             # memory
uptime              # load
sudo crictl stats   # per-container usage from the runtime
```

Compare usage with what was requested:

```bash
kubectl describe node <node> | sed -n '/Allocated resources/,/Events/p'
```

## Container logs

```bash
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs <pod> --previous             # the last crashed instance
kubectl logs <pod> -f                     # follow
kubectl logs <pod> --tail=100 --since=15m
kubectl logs -l app=web --all-containers=true --prefix
kubectl logs deployment/web
kubectl logs job/<job>
```

Containers should write to standard output and standard error. The runtime stores those streams on the node:

```bash
ls /var/log/pods/
ls /var/log/containers/
```

If the API server is down, read the files directly or use `crictl logs <container-id>`.

## Events

Events record what the control plane did and why.

```bash
kubectl get events -A --sort-by=.lastTimestamp
kubectl get events --field-selector type=Warning
kubectl get events --field-selector involvedObject.name=<pod>
kubectl describe pod <pod> | sed -n '/Events:/,$p'
```

Events expire after about an hour, so read them soon after a failure.

## Control-plane and node logs

```bash
kubectl logs -n kube-system kube-apiserver-<node>
kubectl logs -n kube-system etcd-<node>
sudo journalctl -u kubelet -n 100 --no-pager
sudo journalctl -u containerd -n 100 --no-pager
```

When the API server does not answer, use `crictl ps -a` and `crictl logs` on the control-plane node.

## Reading a log quickly

- Start from the end of the log and the timestamp of the failure.
- Look for the first error, not the loudest one; later errors are often consequences.
- For a crash loop, compare `--previous` logs with the current ones.
- For a multi-container pod, read each container separately.

## Common mistakes

- Running `kubectl logs` on a crashed pod without `--previous`.
- Forgetting `-c` on a multi-container pod.
- Treating `kubectl top` failures as application problems when metrics-server is the cause.
- Reading events too late; they age out.

---

Prev: [Services, DNS and networking](services-dns-and-networking.md) · Next: [RBAC](../02-cluster-architecture-installation-config/rbac.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
