# AppArmor and Seccomp

Up: [CKS hub](../../README.md) · Domain 3 — System Hardening (10%) · Prev: [Host hardening](host-hardening.md) · Next: [Security context and PSS](../04-microservice-vulnerabilities/security-context-and-pss.md)

## Scope: what to learn and what not to

Confirm against the official curriculum, but the working assumption for this course is:

| Skill | Expected level |
|---|---|
| Apply an existing seccomp profile (`RuntimeDefault`, `Localhost`) to a pod | **Must** |
| Apply an existing AppArmor profile to a pod, and confirm it is loaded on the node | **Must** |
| Check whether AppArmor/seccomp is active and why a pod was blocked | **Should** |
| Load a ready-made AppArmor profile file with `apparmor_parser` | **Should** |
| Write an AppArmor or seccomp profile from scratch | **Understand the format; do not expect to author one under time pressure** |

Those levels are an assessment, not a confirmed exam blueprint. If the official curriculum lists something different, follow the official list.

## Seccomp

Seccomp filters system calls. Three profile types on a pod:

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault           # the runtime's built-in profile; the safe default
# or
    type: Localhost
    localhostProfile: profiles/restrict.json   # relative to /var/lib/kubelet/seccomp/
# or
    type: Unconfined               # no filtering; do not use
```

### Localhost profile file

The path is relative to the kubelet seccomp root, `/var/lib/kubelet/seccomp/`:

```bash
sudo mkdir -p /var/lib/kubelet/seccomp/profiles
sudo tee /var/lib/kubelet/seccomp/profiles/deny-ptrace.json >/dev/null <<'EOF'
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "names": ["ptrace", "mount", "umount2", "keyctl", "unshare"],
      "action": "SCMP_ACT_ERRNO",
      "args": [],
      "comments": "block common escape and debugging primitives"
    }
  ]
}
EOF
```

A blocklist (default allow, deny specific calls) is the practical pattern. An allowlist that is too narrow breaks the container in ways that look like application bugs.

### Verifying

Whether `ptrace` is blocked by `RuntimeDefault` depends on the runtime and kernel. Do not assume a fixed list; test on the cluster:

```bash
kubectl exec <pod> -- sh -c 'cat /proc/self/status | grep Seccomp'
# Seccomp: 2   means filter mode is active
```

The status field `Seccomp: 0` means no filtering; `2` means a filter is attached.

### Pitfalls

- Missing `profiles/` directory or wrong relative path. The pod stays in a pending or error state with a seccomp-related event. Check with `kubectl describe pod`.
- The profile must exist on **the node the pod is scheduled to**. For a multi-node cluster, copy it everywhere or pin the pod with `nodeSelector`.
- Older annotation form (`seccomp.security.alpha.kubernetes.io/pod`) is deprecated. Use `securityContext.seccompProfile`.

## AppArmor

AppArmor confines a program's file access, network use and capabilities by path. Profiles are loaded into the kernel on each node.

### Check the node

```bash
cat /sys/module/apparmor/parameters/enabled    # Y means enabled
sudo aa-status
```

### Write and load a profile

A minimal profile for a container that may read `/app`, write `/tmp`, and must not touch `/etc` or `/root`:

```bash
sudo tee /etc/apparmor.d/k8s-app >/dev/null <<'EOF'
#include <tunables/global>

profile k8s-app flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  /app/** r,
  /tmp/** rw,
  deny /etc/** rwklx,
  deny /root/** rwklx,
  network inet stream,
  network inet dgram,
}
EOF

sudo apparmor_parser -r -W /etc/apparmor.d/k8s-app   # load or reload
sudo aa-status | grep k8s-app                          # confirm enforce mode
```

Use `aa-complain` to log violations without blocking while you build a profile, then `aa-enforce` to switch to enforce mode.

### Apply to a pod

Current Kubernetes (1.30 and later) uses a field:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  securityContext:
    appArmorProfile:
      type: Localhost
      localhostProfile: k8s-app
  containers:
  - name: app
    image: myapp:1.0
```

Older clusters use an annotation, which is deprecated but still appears in older material and labs:

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/k8s-app
```

Check the server version (`kubectl version`) before choosing. Use one form per pod; setting both is redundant and confusing to read.

### Verify

```bash
kubectl exec app -- cat /etc/shadow        # expect: Permission denied
kubectl exec app -- touch /tmp/ok          # expect: success
sudo journalctl -k | grep -i apparmor      # denials are logged here
```

### Pitfalls

- Profile not loaded on the node the pod runs on. The pod fails to start.
- Profile name in the pod does not match the name declared in the profile file.
- The profile is loaded but in complain mode, so nothing is blocked. Check `aa-status` for "enforce mode".
- Overly broad abstractions that grant more than intended.

## SELinux (concept only)

Some distributions use SELinux instead of AppArmor. The pod field is `securityContext.seLinuxOptions` with `level`, `role`, `type`, `user`. Learn that it exists; don't assume it is the lab's MAC system without checking `getenforce`.

## Quick reference

```bash
aa-status
sudo apparmor_parser -r -W /etc/apparmor.d/<profile>
ls /var/lib/kubelet/seccomp/
grep Seccomp /proc/self/status
```
