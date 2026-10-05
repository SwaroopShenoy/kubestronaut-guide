# Secrets and Encryption at Rest

Up: [CKS hub](../../README.md) · Domain 4 — Minimize Microservice Vulnerabilities (20%) · Prev: [Security context and PSS](security-context-and-pss.md) · Next: [Runtime sandboxes](runtime-sandboxes.md)

Secrets are encoded by default, not encrypted. This topic explains what that means in practice, how to consume secrets more safely, and how to encrypt them in etcd.

## 1. Base64 is not encryption

`kubectl create secret` stores values base64-encoded. Anyone who can read the Secret object can decode it:

```bash
kubectl get secret db -n production -o jsonpath='{.data.password}' | base64 -d
```

Two protections matter: encryption at rest in etcd, and limiting who can read Secrets through RBAC and audit.

## 2. Consuming Secrets safely

Prefer volume mounts over environment variables. Environment variables appear in `kubectl describe`, in process listings of child processes, and in crash dumps.

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: db
      mountPath: /etc/db
      readOnly: true
  volumes:
  - name: db
    secret:
      secretName: db
      defaultMode: 0400
```

Mount with `readOnly: true` and `defaultMode: 0400` so only the owner can read the files.

Do not:

- hardcode values in manifests checked into Git
- bake secrets into images (`ENV` or `COPY` in a Dockerfile persists in layers)
- store secrets in ConfigMaps (ConfigMaps are not access-controlled as sensitive data and have no secret-specific handling)

## 3. Encryption at rest (EncryptionConfiguration)

The API server encrypts resources listed in an `EncryptionConfiguration` before writing them to etcd. The first provider in the list is used for writing; all listed providers can read.

```yaml
# /etc/kubernetes/enc/encryption.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64 of 32 random bytes>
  - identity: {}          # plaintext fallback for reads of old data; keep last
```

Generate a key:

```bash
head -c 32 /dev/urandom | base64
```

Wire it into the API server. Two sections are required: the flag and a volume mount, because the API server runs as a static pod.

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/enc/encryption.yaml
    volumeMounts:
    - name: enc
      mountPath: /etc/kubernetes/enc
      readOnly: true
  volumes:
  - name: enc
    hostPath:
      path: /etc/kubernetes/enc
      type: DirectoryOrCreate
```

The kubelet restarts the API server pod when the manifest changes. Wait for it to come back, then check the API still answers.

### Re-encrypt existing Secrets

Turning encryption on does not rewrite existing objects. Force a rewrite:

```bash
kubectl get secrets -A -o json | kubectl replace -f -
```

Verify in etcd (on a control-plane node, using the etcd certificates from the kubeadm layout):

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/<name> | hexdump -C | head
```

Encrypted values begin with the provider prefix (for example `k8s:enc:aescbc:v1:key1`). Plaintext shows the JSON.

### Keys and rotation

- Add a new key as the first entry, restart the API server, then re-run the `replace` command.
- Remove the old key only after all Secrets have been rewritten.
- Protect the encryption file: it contains the key. Set `chmod 600` and owner root.

### KMS providers (concept)

Production clusters use a KMS plugin so the key lives outside the node. The CKS scope is the local provider above; know that KMS exists and why.

## 4. Who can read Secrets

RBAC verbs `get`, `list`, and `watch` on `secrets` all expose values. A `list` returns full objects with data. Use audit logging to track these reads. Note that many default audit policies log secret writes at high detail but only metadata for reads; add a rule for `get` and `list` on secrets if your task requires it.

## Common mistakes

- Enabling encryption without the volume mount. The API server fails to start.
- Putting `identity: {}` first. New writes stay in plaintext.
- Forgetting to re-run the `replace` step, so older Secrets remain plaintext in etcd.
- Deleting the old key before re-encrypting. Old data becomes unreadable.

## Quick reference

```bash
kubectl get secrets -A -o json | kubectl replace -f -
head -c 32 /dev/urandom | base64
grep encryption-provider-config /etc/kubernetes/manifests/kube-apiserver.yaml
```
