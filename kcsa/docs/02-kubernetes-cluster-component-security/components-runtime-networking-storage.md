# Container Runtime, kube-proxy, Networking, Client and Storage Security

Up: [KCSA README](../../README.md) · Domain 2 — Kubernetes Cluster Component Security (22%) · Prev: [etcd and node security](etcd-and-node-security.md) · Next: [RBAC and ServiceAccounts](../03-kubernetes-security-fundamentals/rbac-and-serviceaccounts.md)

Every component of a cluster is a potential target, and every component can be hardened. This chapter covers the controller manager, scheduler, runtime, kube-proxy, networking, client credentials and storage.

Official curriculum topics covered here: controller manager, scheduler, container runtime, kube-proxy, pod, container networking, client security and storage. The API server, kubelet, etcd and node topics are in [control plane security](control-plane-security.md) and [etcd and node security](etcd-and-node-security.md).

## Controller manager and scheduler

- They authenticate to the API server with their own credentials, not cluster-admin.
- They should not expose unauthenticated endpoints. Bind their metrics and health ports to localhost where the platform allows.
- A compromised controller manager can create or change workloads, so its RBAC is powerful. Keep its credentials protected.

## Container runtime

- Runs containers on each node. containerd and CRI-O are the common runtimes.
- Use runtime security features: seccomp, AppArmor or SELinux, user namespaces, and cgroup limits.
- Keep the runtime patched. Runtime escape bugs are cluster-wide risks.
- Limit who can talk to the runtime socket. Access to it is root-equivalent on the node.

## kube-proxy

- Runs on every node and programs Service routing rules.
- Keep it updated and limit its credentials to what it needs.

## Pod

- A pod's security context and the namespace's Pod Security level decide what the pod may do. See [pod security and NetworkPolicy](../03-kubernetes-security-fundamentals/pod-security-and-networkpolicy.md).

## Container networking

- The CNI plugin provides pod networking. Choose a plugin that enforces NetworkPolicy.
- Default-deny NetworkPolicy limits lateral movement after a compromise.
- Encrypt node-to-node and pod-to-pod traffic where the CNI supports it (for example WireGuard or IPsec in Cilium).

## Client security

- Client certificates and kubeconfig files are credentials. Protect them with file permissions and do not commit them.
- Short-lived tokens are safer than long-lived ones.
- Revoke access by removing bindings and rotating credentials, not by deleting kubeconfig files on one laptop.

## Storage

- Restrict which pods can mount which volumes. A hostPath mount is a host-access risk.
- Encrypt data at rest in the storage backend, and enable encryption for Secrets in etcd.
- Use StorageClass parameters to enforce encryption where the provider supports it.
