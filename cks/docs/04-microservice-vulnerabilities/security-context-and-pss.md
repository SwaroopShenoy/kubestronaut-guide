# Security Context and Pod Security Standards

Up: [CKS hub](../../README.md) · Domain 4 — Minimize Microservice Vulnerabilities (20%) · Prev: [AppArmor and seccomp](../03-system-hardening/apparmor-and-seccomp.md) · Next: [Secrets and encryption at rest](secrets-and-encryption-at-rest.md)

A pod's security context and its namespace's Pod Security level decide how much damage a compromised container can do. This topic covers both, with the requirements of each level.

Pod Security Admission (PSA) replaced PodSecurityPolicy, which was removed in Kubernetes 1.25. Do not use PSP in new work.

## 1. SecurityContext

Set at pod level (applies to all containers unless overridden) or container level (overrides).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened
  namespace: production
spec:
  securityContext:              # pod level
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:1.0
    securityContext:            # container level
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir:
```

Field reference:

| Field | Level | Effect |
|---|---|---|
| `runAsNonRoot` | pod or container | Kubelet refuses to start the container if its image resolves to UID 0 |
| `runAsUser` / `runAsGroup` | both | UID/GID the process runs as |
| `fsGroup` | pod | Group ownership applied to volumes; pods can write to them |
| `allowPrivilegeEscalation` | container | Sets `no_new_privs`; blocks setuid escalation |
| `readOnlyRootFilesystem` | container | Root filesystem is read-only; use `emptyDir` for writable paths |
| `capabilities.drop` | container | Remove Linux capabilities; `ALL` then add back only what is needed |
| `privileged` | container | Full host-level access. Must be false or absent |
| `seccompProfile` | both | Syscall filter (see AppArmor and seccomp doc) |

### Verify

```bash
kubectl exec hardened -n production -- id                      # uid=10001, not 0
kubectl exec hardened -n production -- touch /usr/x            # Read-only file system
kubectl exec hardened -n production -- touch /tmp/x            # works (emptyDir)
kubectl exec hardened -n production -- grep Cap /proc/self/status
```

### Images that run as root

`runAsNonRoot: true` fails for images whose user is root unless you also set `runAsUser`. The kubelet error reads `container has runAsNonRoot and image will run as root`. Fixes: set `runAsUser` to a non-zero UID, or use an image built to run as non-root.

Some official images need changes to run non-root. `nginx` listens on port 80 by default and its master process starts as root. Use `nginxinc/nginx-unprivileged` (listens on 8080) or change the listen port and writable paths.

## 2. Pod Security Admission

PSA enforces one of three profiles per namespace, set by labels.

| Profile | Summary |
|---|---|
| `privileged` | No restrictions |
| `baseline` | Blocks known privilege escalations: privileged containers, host namespaces, hostPath volumes, most dangerous capabilities, unconfined seccomp/AppArmor, unsafe sysctls |
| `restricted` | Baseline plus: `runAsNonRoot: true`; `allowPrivilegeEscalation: false`; `capabilities.drop: ["ALL"]` (only `NET_BIND_SERVICE` may be added back); seccomp `RuntimeDefault` or `Localhost`; restricted volume types only |

`readOnlyRootFilesystem` is **not** section of restricted. Do not claim it is.

Modes:

| Mode | Behavior |
|---|---|
| `enforce` | Reject non-compliant pods |
| `audit` | Allow, record in audit log |
| `warn` | Allow, return a warning to the client |

Labels:

```bash
kubectl label ns production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

Use a dry run to see what would be rejected before enforcing:

```bash
kubectl label --dry-run=server --overwrite ns production pod-security.kubernetes.io/enforce=restricted
```

The dry-run output lists existing pods that would violate the new level.

### What the rejection looks like

For a bare pod, `kubectl apply` fails immediately with `violates PodSecurity "restricted:latest"` and the specific fields.

For a Deployment, the Deployment object is created, but the ReplicaSet cannot create pods. Look at the ReplicaSet events, not the Deployment:

```bash
kubectl get deploy,rs -n production
kubectl describe rs -n production | grep -i -A3 "violates\|forbidden"
```

### Do not label kube-system as restricted

System components run privileged. Labeling `kube-system` as enforced at `restricted` breaks them. The common pattern is `privileged` on `kube-system`, and `restricted` or `baseline` on workload namespaces.

### Testing PSA without a live pod

Use `--dry-run=server` with a manifest that deliberately violates the profile:

```bash
kubectl run test --image=nginx --dry-run=server -n production \
  --overrides='{"spec":{"containers":[{"name":"test","image":"nginx","securityContext":{"privileged":true}}]}}'
```

The error text names the rule that fails.

### Cluster-wide defaults and exemptions

Namespace labels set the level per namespace. To set a default for namespaces that have no label, and to exempt certain users, runtime classes or namespaces, configure the PodSecurity plugin in an admission configuration file:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: baseline
      enforce-version: latest
      audit: restricted
      audit-version: latest
      warn: restricted
      warn-version: latest
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces:
      - kube-system
```

The API server reads the file through `--admission-control-config-file`, with the file mounted into the static pod like the other configuration files. That flag takes one file for all admission plugins, so if another plugin such as ImagePolicyWebhook is already configured, add the PodSecurity entry to the same `plugins` list.

The `-version` fields pin the Pod Security Standards to a Kubernetes version, for example `v1.35`, so a cluster upgrade does not tighten enforcement unexpectedly. `latest` follows the installed version. Namespace labels such as `pod-security.kubernetes.io/enforce-version` do the same per namespace.

An exemption removes a namespace or user from checking entirely, so keep the list short and review it.

## 3. Fixing a non-compliant Deployment

Typical PSA failures and the field to add:

| Failure message fragment | Fix |
|---|---|
| `runAsNonRoot != true` | `runAsNonRoot: true` plus `runAsUser` |
| `allowPrivilegeEscalation != false` | container `allowPrivilegeEscalation: false` |
| `unrestricted capabilities` | `capabilities.drop: ["ALL"]` |
| `seccompProfile` | pod `seccompProfile.type: RuntimeDefault` |
| `restricted volume types` (hostPath) | replace with `emptyDir` or a PVC |

Patch in place:

```bash
kubectl patch deploy app -n production --type=strategic -p '
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
'
```

## Common mistakes

- Setting `readOnlyRootFilesystem` and forgetting an `emptyDir` for `/tmp` or the app's cache directory. The app fails at startup.
- Dropping ALL capabilities and wondering why a process cannot bind port 80. Add `NET_BIND_SERVICE` or use a higher port.
- Setting the pod-level `runAsNonRoot` while the image's user is root and no `runAsUser` is given.
- Thinking PSA checks an already-running pod. It checks at creation and update of pods (and updates to pod templates through controllers).

## Quick reference

```bash
kubectl label ns <ns> pod-security.kubernetes.io/enforce=restricted
kubectl exec <pod> -- id
kubectl get pod <pod> -o jsonpath='{.spec.securityContext}{"\n"}'
```

---

Prev: [AppArmor and seccomp](../03-system-hardening/apparmor-and-seccomp.md) · Next: [Secrets and encryption at rest](secrets-and-encryption-at-rest.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
