# AppArmor & Seccomp Masterclass
## CKS Deep Dive - OS-Level Container Restriction

---

# Section 1: AppArmor Fundamentals

## What is AppArmor?

**AppArmor** = Linux kernel security module. Restricts what processes can access (files, network, capabilities).

**Think of it as**: Firewall for file/system access. "This process can only read /app, write to /tmp, make network connections to port 443."

**Why it matters for CKS**:
- Prevent container escapes
- Limit damage if container compromised
- Kernel-level enforcement (can't be bypassed from container)

---

## AppArmor Profile Basics

```
/etc/apparmor.d/docker-default   # Default Docker profile
/etc/apparmor.d/my-profile        # Custom profiles
```

### Profile Syntax

```
/etc/apparmor.d/container-app:
  # Allow rules
  /app/** r,               # Read anything under /app
  /tmp/** rw,              # Read-write anything under /tmp
  /var/log/app.log w,      # Write to log file
  
  # Deny rules
  deny /etc/** rwk,        # Deny all access to /etc
  
  # Network
  network inet dgram,      # Allow UDP
  network inet stream,     # Allow TCP
```

### Access Modes

| Mode | Meaning |
|------|---------|
| `r` | Read |
| `w` | Write |
| `x` | Execute |
| `k` | Lock |
| `a` | Append |
| `ux` | Unconfined execute |
| `Ux` | Unconfined execute (inherit) |
| `px` | Confined execute |
| `Px` | Confined execute (inherit) |

---

# Section 2: Using AppArmor in Kubernetes

## Step 1: Load AppArmor Profile on Node

```bash
# On node
sudo cat <<EOF > /etc/apparmor.d/my-app
#include <tunables/global>

profile my-app flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  
  # Allow reading app files
  /app/** r,
  
  # Allow writing logs
  /var/log/** w,
  
  # Allow temp files
  /tmp/** rw,
  
  # Deny system access
  deny /etc/** rwk,
  deny /root/** rwk,
  
  # Network
  network inet stream,
  network inet dgram,
}
EOF

# Load profile
sudo apparmor_parser -r /etc/apparmor.d/my-app

# Verify loaded
sudo aa-status | grep my-app
# Output: my-app (enforce mode)
```

## Step 2: Apply Profile to Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  annotations:
    # Reference the profile
    container.apparmor.security.beta.kubernetes.io/app: localhost/my-app
spec:
  containers:
  - name: app
    image: myapp:v1
```

## Step 3: Verify Profile Applied

```bash
# Inside pod, try to access restricted file
k exec -it pod/app-pod -- cat /etc/passwd
# Error: Permission denied (blocked by AppArmor)

# Try to write to log (allowed)
k exec -it pod/app-pod -- echo "test" > /var/log/app.log
# Works!
```

---

## Real Exam Scenario: AppArmor

**Question**: Create AppArmor profile restricting pod to /app and /tmp only. Block /etc access.

```bash
# 1. Create profile
sudo cat <<EOF > /etc/apparmor.d/restricted-app
#include <tunables/global>

profile restricted-app flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  
  # Allow app directory
  /app/** rw,
  
  # Allow temp
  /tmp/** rw,
  
  # Deny everything else
  deny /etc/** rwk,
  deny /root/** rwk,
  deny /var/** rwk,
  
  # Allow basic network
  network inet stream,
  network inet dgram,
}
EOF

# 2. Load
sudo apparmor_parser -r /etc/apparmor.d/restricted-app

# 3. Verify
sudo aa-status | grep restricted-app

# 4. Apply to pod
k apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/restricted-app
spec:
  containers:
  - name: app
    image: myapp:v1
EOF

# 5. Test
k exec -it pod/restricted-pod -- cat /etc/passwd
# Denied

k exec -it pod/restricted-pod -- touch /tmp/test
# Works
```

---

# Section 3: Seccomp Fundamentals

## What is Seccomp?

**Seccomp** = Secure computing mode. Filter which system calls (syscalls) container can make.

**Think of it as**: Whitelist/blacklist of kernel functions. "Container can call open(), read(), write() but NOT ptrace(), mount(), etc."

**Why it matters**:
- Block privilege escalation (ptrace can debug as root)
- Block container escapes (mount, unshare, etc.)
- Block dangerous operations (perfopen, etc.)

---

## Seccomp vs AppArmor

| Feature | AppArmor | Seccomp |
|---|---|---|
| **What it controls** | File/network access | Syscalls (kernel functions) |
| **Granularity** | Coarse (paths, ports) | Fine (specific syscalls) |
| **Enforcement point** | Filesystem | Syscall interception |
| **Use for** | File restrictions | Kernel attack prevention |

---

# Section 4: Using Seccomp in Kubernetes

## Seccomp Profiles

### RuntimeDefault (Recommended)

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

**What it does**: Uses Docker/Kubernetes default profile (blocks ptrace, mount, etc.)

### Localhost (Custom Profile)

```yaml
securityContext:
  seccompProfile:
    type: Localhost
    localhostProfile: my-seccomp.json
```

**You provide JSON file on node**.

### Unconfined (No Filtering)

```yaml
securityContext:
  seccompProfile:
    type: Unconfined
```

**Dangerous - allows all syscalls.**

---

## Step 1: Create Seccomp Profile (Optional)

```bash
# Default profile is usually sufficient
# But if you need custom...

cat <<EOF > /var/lib/kubelet/seccomp/my-profile.json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "defaultErrnoRet": 1,
  "archMap": [
    {
      "architecture": "SCMP_ARCH_X86_64",
      "subArchitectures": [
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
      ]
    }
  ],
  "syscalls": [
    {
      "names": ["ptrace", "mount", "umount2", "perfopen", "syslog"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
EOF
```

## Step 2: Apply Seccomp to Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault  # Use default (blocks dangerous syscalls)
  
  containers:
  - name: app
    image: myapp:v1
```

## Step 3: Test (Try Blocked Syscall)

```bash
# Try to trace another process (ptrace - blocked)
k exec -it pod/secure-pod -- strace -c sleep 1
# Error: strace: trace: ptrace(PTRACE_TRACEME): Operation not permitted

# Normal syscalls still work
k exec -it pod/secure-pod -- echo "test"
# Works! (write syscall allowed)
```

---

## Real Exam Scenario: Seccomp

**Question**: Apply seccomp profile to prevent ptrace (process debugging).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: protected-pod
spec:
  # Apply RuntimeDefault seccomp
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  
  containers:
  - name: app
    image: myapp:v1
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
```

**Result**: Pod can't call ptrace. Prevents container escape via process debugging.

---

# Section 5: AppArmor + Seccomp Together

## Defense in Depth

```
User tries to escape container
    ↓
SecurityContext (runAsNonRoot, no escalation)
    ↓
AppArmor (block /etc, /root file access)
    ↓
Seccomp (block ptrace, mount, perfopen syscalls)
    ↓
Result: Escape FAILS at multiple layers
```

## Combined Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/my-profile
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  
  containers:
  - name: app
    image: myapp:v1
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
```

**Protections**:
- ✅ Non-root user (can't do root operations)
- ✅ No privilege escalation (can't sudo)
- ✅ No capabilities (can't use special powers)
- ✅ AppArmor (file/network restricted)
- ✅ Seccomp (dangerous syscalls blocked)
- ✅ Read-only filesystem (can't write backdoors)

---

# Section 6: Debugging

## AppArmor Issues

```bash
# Check if profile loaded
sudo aa-status | grep my-profile

# Check denials in kernel log
sudo tail -f /var/log/syslog | grep apparmor

# Pod denied by AppArmor?
k describe pod <pod>
# Look for: "apparmor enforcement"

# Reload profile
sudo apparmor_parser -r /etc/apparmor.d/my-profile

# Put profile in complain mode (log only, don't deny)
sudo aa-complain /etc/apparmor.d/my-profile
```

## Seccomp Issues

```bash
# Check if seccomp loaded
ls /var/lib/kubelet/seccomp/

# Check syscall denials
journalctl -u kubelet -f | grep seccomp

# Pod blocked by seccomp?
k describe pod <pod>
# Look for: "seccomp" in events

# Try strace to see syscalls
k exec -it pod/test -- strace ls
# If blocked: "ptrace not allowed"
```

---

# Pre-Exam Checklist

- ✅ Understand: AppArmor blocks file/network, Seccomp blocks syscalls
- ✅ Load AppArmor profile on node
- ✅ Apply AppArmor to pod (annotation)
- ✅ Test profile (denied + allowed access)
- ✅ Apply Seccomp (RuntimeDefault)
- ✅ Test Seccomp (blocked syscall shows error)
- ✅ Debug when denied (check logs, reload profile)
- ✅ Speed: Should take <15 min to create + test profile

---

# Speed Targets for Exam

- **Create AppArmor profile**: <5 min (copy template, modify paths)
- **Load profile on node**: <1 min
- **Apply to pod**: <2 min
- **Test (verify denied/allowed)**: <3 min
- **Apply Seccomp**: <1 min (just add to SecurityContext)
- **Debug failure**: <5 min

---

# Cheat Sheet

## AppArmor Quick Template

```
/etc/apparmor.d/my-app:
  #include <tunables/global>
  
  profile my-app flags=(attach_disconnected,mediate_deleted) {
    #include <abstractions/base>
    
    /app/** rw,                    # Read-write app dir
    /tmp/** rw,                    # Temp files
    deny /etc/** rwk,              # Deny system config
    deny /root/** rwk,             # Deny root home
    
    network inet stream,
    network inet dgram,
  }
```

## Pod Annotation

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/my-app
```

## Seccomp in Pod

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

