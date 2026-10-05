# Certificates and kubeconfig

Up: [CKA hub](../../README.md) · Domain 2 — Cluster Architecture (25%) · Prev: [kubeadm install and upgrade](kubeadm-install-and-upgrade.md) · Next: [etcd backup and restore](etcd-backup-restore.md)

## Where kubeadm keeps certificates

```
/etc/kubernetes/pki/
  ca.crt / ca.key                    cluster CA
  apiserver.crt / apiserver.key
  apiserver-kubelet-client.crt / .key
  front-proxy-ca.crt / .key
  front-proxy-client.crt / .key
  sa.key / sa.pub                    service account signing keys
  etcd/
    ca.crt / ca.key
    server.crt / server.key
    peer.crt / peer.key
    healthcheck-client.crt / .key
```

## Check expiry

```bash
sudo kubeadm certs check-expiration
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -subject -enddate
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A1 'Subject Alternative Name'
```

Check the subject alternative names when the API server rejects a client with a hostname or IP mismatch.

## Renew certificates

```bash
sudo kubeadm certs renew all              # or a single name, e.g. apiserver
sudo systemctl restart kubelet            # reload control-plane pods
```

Restarting kubelet is enough for static pods. Some tasks instead say to move the manifest out and back; either approach restarts the component.

## Kubelet certificates on workers

The kubelet client certificate rotates automatically when rotation is enabled. Check it on the node:

```bash
ls -l /var/lib/kubelet/pki/
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -enddate
```

If the kubelet cannot authenticate, compare the kubeconfig and the certificate dates before renewing anything.

## CertificateSigningRequests

```bash
kubectl get csr
kubectl describe csr <name>
kubectl certificate approve <name>
kubectl certificate deny <name>
```

## kubeconfig

A kubeconfig has three parts: clusters (server and CA), users (credentials), and contexts (a cluster plus a user plus a namespace).

```bash
kubectl config view --minify                       # current context only
kubectl config get-contexts
kubectl config use-context <ctx>
kubectl config set-context --current --namespace=<ns>
kubectl --context=<ctx> get pods                  # one-off, without switching
```

Use several kubeconfigs together:

```bash
export KUBECONFIG=~/.kube/config:/path/other.conf
kubectl config view --flatten > /tmp/merged.conf   # if you need one file
```

## Create a user with a client certificate

```bash
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -out jane.csr -subj "/CN=jane/O=developers"
sudo openssl x509 -req -in jane.csr \
  -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial -out jane.crt -days 365

kubectl config set-credentials jane --client-certificate=jane.crt --client-key=jane.key --embed-certs=true
kubectl config set-cluster kubernetes --server=https://<api-server>:6443 --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true
kubectl config set-context jane@kubernetes --cluster=kubernetes --user=jane
```

The user has no permissions until a RoleBinding exists. See [RBAC](rbac.md).

## Troubleshooting kubeconfig

| Error | Likely cause |
|---|---|
| `x509: certificate signed by unknown authority` | Wrong CA in the kubeconfig |
| `x509: certificate has expired` | Renew the cert |
| `connection refused` | Wrong `server:` address or API server down |
| `forbidden` | Authentication works; RBAC is missing |
| `context ... does not exist` | Typo in context name or wrong kubeconfig file |

## Quick reference

```bash
sudo kubeadm certs check-expiration
sudo kubeadm certs renew all
kubectl certificate approve <csr>
kubectl config view --minify
```
