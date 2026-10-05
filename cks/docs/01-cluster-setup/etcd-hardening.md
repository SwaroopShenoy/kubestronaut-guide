# etcd Hardening

Up: [CKS README](../../README.md) · Domain 1 — Cluster Setup (15%) · Prev: [Ingress TLS, node metadata and binary verification](ingress-tls-and-node-metadata.md) · Next: [RBAC](../02-cluster-hardening/rbac.md)

etcd holds every object in the cluster, including Secrets. Whoever can read or write etcd controls the cluster, so it is secured separately from the API server.

Official curriculum topic: "Use CIS benchmark to review the security configuration of Kubernetes components (etcd, kubelet, kubedns, kubeapi)". The etcd documentation (etcd.io/docs) is on the exam's allowed-resources list.

## What to protect

| Concern | Control |
|---|---|
| Who can connect | Client certificate authentication; only the API server holds a client certificate |
| Traffic in transit | TLS for client (2379) and peer (2380) connections |
| Data at rest on disk | Restrictive permissions on the data directory; encryption of Secrets (see [Secrets and encryption at rest](../04-microservice-vulnerabilities/secrets-and-encryption-at-rest.md)) |
| Network reach | Firewall 2379 and 2380 to the control-plane nodes |
| Recovery | Regular, protected snapshots |

## Settings to check in the etcd manifest

On kubeadm clusters etcd is a static pod defined in `/etc/kubernetes/manifests/etcd.yaml`.

```bash
sudo grep -E 'client-cert-auth|peer-client-cert-auth|auto-tls|cert-file|key-file|trusted-ca-file|listen-client-urls|data-dir' /etc/kubernetes/manifests/etcd.yaml
```

Expected values:

```yaml
- --client-cert-auth=true
- --peer-client-cert-auth=true
- --cert-file=/etc/kubernetes/pki/etcd/server.crt
- --key-file=/etc/kubernetes/pki/etcd/server.key
- --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
- --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
- --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
- --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

Self-signed automatic TLS (`--auto-tls`, `--peer-auto-tls`) should not be enabled. Where the flags appear, set them to `false`.

The `listen-client-urls` should be the loopback address and the node's own address, not `0.0.0.0` on an exposed interface.

## Data directory permissions

```bash
sudo ls -ld /var/lib/etcd
sudo chmod 700 /var/lib/etcd
sudo chown -R root:root /var/lib/etcd
```

The CIS benchmark expects mode 700 and ownership by the user that runs etcd. On kubeadm clusters etcd runs as root.

## Restricting network access

Only control-plane nodes and the API server should reach etcd.

```bash
sudo ufw allow from 10.0.0.5 to any port 2379 proto tcp
sudo ufw allow from 10.0.0.5 to any port 2380 proto tcp
sudo ufw deny 2379
sudo ufw deny 2380
```

Add the allow rules before the deny rules and keep your own SSH access open.

## Checking that etcd answers only to authorised clients

```bash
sudo ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

A request without a valid client certificate should be refused:

```bash
curl -sk https://127.0.0.1:2379/health
```

With `--client-cert-auth=true` this fails the TLS handshake instead of returning a result.

## API server side

The API server's connection to etcd is set by these flags in `kube-apiserver.yaml`:

```yaml
- --etcd-servers=https://127.0.0.1:2379
- --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
- --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
- --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
```

## Backup

Snapshots contain every Secret in plain form unless encryption at rest is enabled. Store them with restricted permissions and off the node. The snapshot and restore procedure is in [etcd backup and restore](../../../cka/docs/02-cluster-architecture-installation-config/etcd-backup-restore.md).

## Common mistakes

- Exposing the client port on all interfaces with no firewall.
- Leaving `--client-cert-auth` unset, which accepts any client that can reach the port.
- Encrypting Secrets in the API server but leaving old snapshots unencrypted.
- Changing the manifest with a typo and losing the cluster's database process.

---

Prev: [Ingress TLS, node metadata and binary verification](ingress-tls-and-node-metadata.md) · Next: [RBAC](../02-cluster-hardening/rbac.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
