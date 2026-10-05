# etcd and Node Security

Up: [KCSA hub](../../KCSA_Crash_Course.md) · Domain 2 — Kubernetes Cluster Component Security (22%) · Prev: [Control plane security](control-plane-security.md) · Next: [RBAC and ServiceAccounts](../03-kubernetes-security-fundamentals/rbac-and-serviceaccounts.md)

## etcd

etcd stores all cluster state, including Secrets. Whoever can read etcd can read every Secret.

Protections:

- **Encryption at rest**: the API server encrypts configured resources (typically Secrets) before writing them to etcd, using an EncryptionConfiguration. Existing data must be rewritten to be encrypted.
- **Mutual TLS**: the API server and etcd authenticate each other with client certificates.
- **Network isolation**: only control-plane components reach the etcd client port.
- **Backups**: encrypted, stored separately, and tested by restoring.

Encryption providers: `aescbc`, `secretbox`, `kms` (external key management). The `identity` provider stores plaintext and must be last in the list if used as a fallback.

## Node security

The kubelet is the most sensitive node component: it can start containers and read node-level resources.

Kubelet hardening:

- Anonymous auth disabled (`authentication.anonymous.enabled: false`).
- Webhook authorization (`authorization.mode: Webhook`), not AlwaysAllow.
- Read-only port disabled (`readOnlyPort: 0`).
- Certificate rotation enabled.
- Protect kernel defaults.

Node OS hardening:

- Minimal operating system images where possible.
- Remove unneeded services and packages; restrict SSH to keys; disable root login.
- Patch regularly.
- Firewall the kubelet API (10250) and etcd ports.
- Use the container runtime's security features: seccomp, AppArmor or SELinux, user namespaces.

## kube-proxy

Runs on every node and programs Service routing rules. Keep it updated and restrict its credentials to what it needs.

## Practice questions

- Why is base64 in Secrets not a security control? (It is encoding, reversible by anyone who reads the object)
- Which setting disables unauthenticated kubelet access? (`anonymous.enabled: false`)
- What must be true of an identity provider listed last in an EncryptionConfiguration? (It is the plaintext fallback; it does not encrypt new writes)
- Which port should a firewall restrict on each node? (Kubelet API, 10250)
