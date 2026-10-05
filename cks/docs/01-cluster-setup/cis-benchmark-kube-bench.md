# CIS Benchmark and kube-bench

Up: [CKS hub](../../CKS_2026_Complete_Crash_Course.md) · Domain 1 — Cluster Setup (10%) · Prev: [Network policy](network-policy.md) · Next: [Ingress, TLS and node metadata](ingress-tls-and-node-metadata.md)

## Exam scope

**In scope:** running kube-bench, reading PASS/FAIL/WARN/INFO, and fixing a small number of failing checks by editing the component's configuration (static pod manifests and the kubelet config file).

**Not required:** memorizing check IDs or the full benchmark. Check IDs differ between benchmark versions, so always use the ID that kube-bench prints and read its remediation text.

## What CIS is

The CIS Kubernetes Benchmark is a community-agreed list of hardening checks for the control plane, worker nodes, etcd, and policies. kube-bench is the open-source tool that runs those checks against a real node.

## Running kube-bench

```bash
# Check the built-in target names for your version first
kube-bench run --help

# Run on a control-plane node
sudo kube-bench run --targets control-plane

# Run on a worker node
sudo kube-bench run --targets node

# Save results for review
sudo kube-bench run --targets control-plane > bench.txt
grep -E "\[FAIL\]|\[WARN\]" bench.txt
```

Status meanings:

| Status | Meaning | Action |
|---|---|---|
| PASS | Check satisfied | None |
| FAIL | Violates a recommendation | Fix it |
| WARN | Needs manual review | Read the remediation; decide |
| INFO | Informational | None |

Each FAIL line includes a remediation block. Start there, not with memory.

## Where the configuration lives

| Component | File |
|---|---|
| kube-apiserver | `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| kube-controller-manager | `/etc/kubernetes/manifests/kube-controller-manager.yaml` |
| kube-scheduler | `/etc/kubernetes/manifests/kube-scheduler.yaml` |
| etcd | `/etc/kubernetes/manifests/etcd.yaml` |
| kubelet config | `/var/lib/kubelet/config.yaml` (confirm with `ps -ef \| grep kubelet`) |
| kubelet flags/service | `/etc/systemd/system/kubelet.service.d/` |

Static pod manifests are picked up automatically by the kubelet once saved. Do not run `kubectl apply` on them. To confirm a change took, wait for the component pod to restart (`kubectl get pods -n kube-system`) and rerun kube-bench on that check.

## Control-plane fixes worth knowing

These are settings that CIS-style checks and the exam commonly target. The examples show the flag form; verify against the remediation text kube-bench prints for your version.

```yaml
# kube-apiserver.yaml, under spec.containers[0].command
- --anonymous-auth=false
- --authorization-mode=Node,RBAC        # never AlwaysAllow
- --enable-admission-plugins=NodeRestriction
- --profiling=false
- --encryption-provider-config=/etc/kubernetes/enc/encryption.yaml   # see secrets doc
```

Notes:

- `--insecure-port` was removed from kube-apiserver. If you see a remediation that says to set it to 0 on a current cluster, the check is outdated; read the version-specific text.
- Mounting the encryption config into the API server pod needs a `hostPath` volume and a `volumeMount` as well as the flag. See [Secrets and encryption at rest](../04-microservice-vulnerabilities/secrets-and-encryption-at-rest.md).

etcd and file permissions:

```bash
sudo chmod 700 /var/lib/etcd
sudo chmod 600 /etc/kubernetes/admin.conf /etc/kubernetes/scheduler.conf /etc/kubernetes/controller-manager.conf
```

## Kubelet fixes

Modern kubelets read settings from the config file. The config-file field names differ from the old flags:

```yaml
# /var/lib/kubelet/config.yaml
authentication:
  anonymous:
    enabled: false          # flag form: --anonymous-auth=false
  webhook:
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook             # never AlwaysAllow
readOnlyPort: 0             # disables the unauthenticated read-only port
protectKernelDefaults: true
```

After editing:

```bash
sudo systemctl daemon-reload    # only if you edited a unit file
sudo systemctl restart kubelet
sudo systemctl status kubelet --no-pager
```

Verify the read-only port is closed:

```bash
sudo ss -tlnp | grep 10255 || echo "10255 closed"
```

## Workflow for an exam task

1. `kube-bench run --targets <target>` and capture FAILs.
2. Pick the two or three that are quick (anonymous auth, authorization mode, readOnlyPort, file permissions).
3. For each: open the config file, change one thing, save.
4. Wait for the component to restart, then rerun kube-bench and grep for the check.
5. Do not chase every FAIL. Partial credit is per task, not per check, so fix what the task asks for.

## Common mistakes

- Editing a manifest then restarting the wrong thing. For static pods the kubelet re-reads the manifest; restarting kubelet is only needed for kubelet config changes.
- Setting the value in the wrong file: API server flags go in the manifest, kubelet settings go in `config.yaml`.
- Writing a YAML syntax error in a static pod manifest. The pod disappears and you lose the API server. Re-check with `kubectl get pods -n kube-system` and `crictl ps` if the API stops responding.
- Assuming a check is still valid. Remediations that mention removed flags are outdated.

## Quick reference

```bash
sudo kube-bench run --targets control-plane
sudo kube-bench run --targets node
grep -n "authorization-mode\|anonymous-auth" /etc/kubernetes/manifests/kube-apiserver.yaml
sudo grep -n "anonymous\|authorization\|readOnlyPort" /var/lib/kubelet/config.yaml
```
