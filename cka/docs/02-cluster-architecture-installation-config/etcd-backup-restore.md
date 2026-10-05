# etcd Backup and Restore

Up: [CKA hub](../../README.md) · Domain 2 — Cluster Architecture (25%) · Prev: [Certificates and kubeconfig](certificates-and-kubeconfig.md) · Next: [CRDs and operators](crds-and-operators.md)

Everything the cluster knows is stored in one database, etcd. This chapter covers how to save that database, check the copy, and restore it when things go badly wrong.

etcd holds all cluster state. Losing it means losing the cluster, so backups are a core skill.

## Find the endpoints and certificates

On kubeadm clusters, read them from the etcd static pod:

```bash
sudo grep -E 'listen-client-urls|cert-file|key-file|trusted-ca-file|data-dir' /etc/kubernetes/manifests/etcd.yaml
```

Common values: endpoint `https://127.0.0.1:2379`, CA `/etc/kubernetes/pki/etcd/ca.crt`, client cert `/etc/kubernetes/pki/etcd/server.crt`, key `/etc/kubernetes/pki/etcd/server.key`, data dir `/var/lib/etcd`.

If `etcdctl` is not installed on the node, run it from the etcd pod:

```bash
kubectl -n kube-system exec etcd-<node> -- etcdctl version
```

## Snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot save /var/backups/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Verify the snapshot:

```bash
ETCDCTL_API=3 etcdutl snapshot status /var/backups/etcd-snapshot.db --write-out=table
```

Use `etcdutl` for snapshot status and restore. `etcdctl snapshot restore` is deprecated in current etcd releases.

## Restore

Restore writes a new data directory. Then the etcd static pod must point at it.

```bash
# 1. Stop the API server and etcd by moving their manifests away
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# 2. Restore to a new directory
sudo etcdutl snapshot restore /var/backups/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --name=<etcd-member-name> \
  --initial-cluster=<name>=https://<node-ip>:2380 \
  --initial-advertise-peer-urls=https://<node-ip>:2380

# 3. Point etcd at the restored data
sudo sed -i 's|/var/lib/etcd|/var/lib/etcd-restored|g' /tmp/etcd.yaml

# 4. Put the manifests back
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 5. Verify
kubectl get nodes
kubectl get pods -A
```

Values for `--name` and `--initial-cluster` must match the etcd manifest (`--name` and `--initial-cluster` flags). Read them from the manifest before restoring.

Check that the `hostPath` volume for the data directory in `etcd.yaml` points at the restored path; that is the step most often missed.

## Common mistakes

- Forgetting `--endpoints` or the certificate flags. Every etcdctl call needs them.
- Restoring to `/var/lib/etcd` directly while etcd still runs. Use a new directory, then switch.
- Mismatched `--name` or `--initial-cluster` values.
- Restoring but not moving the API server manifest back.

## Quick reference

```bash
ETCDCTL_API=3 etcdctl snapshot save <file> --endpoints=... --cacert=... --cert=... --key=...
ETCDCTL_API=3 etcdutl snapshot status <file> --write-out=table
etcdutl snapshot restore <file> --data-dir=<new-dir> ...
```
