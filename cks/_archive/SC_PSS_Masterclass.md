# SecurityContext & Pod Security Standards Masterclass
## CKS Deep Dive - Container Isolation & Hardening

---

# Section 1: SecurityContext (SC)

## What is SecurityContext?

**SecurityContext** = Tell the OS what permissions a container can have.

Think of it as: "Lock down what this container can do at the Linux/OS level."

**Where it's applied**:
- **Pod-level**: Applies to ALL containers in the Pod (unless overridden)
- **Container-level**: Applies to specific container (overrides Pod-level)

---

## SecurityContext Fields (Complete Reference)

### 1. runAsUser / runAsNonRoot

**Purpose**: What user ID does the container run as?

```yaml
# Pod-level (applies to all containers)
spec:
  securityContext:
    runAsUser: 1000          # Run as user ID 1000 (NOT root)
    runAsNonRoot: true       # ENFORCE non-root (fail if not possible)

# Container-level (overrides Pod)
spec:
  containers:
  - name: app
    image: myapp
    securityContext:
      runAsUser: 2000        # This container runs as 2000
      runAsNonRoot: true     # This container must be non-root
```

**Why it matters**:
- ✅ Root (UID 0) can do ANYTHING (escape container, modify files, etc.)
- ✅ Non-root (UID 1000+) has limited permissions
- **Exam will test this heavily**

**Verify**:
```bash
k exec <pod> -- id
# Output: uid=1000(app) gid=1000(app) groups=1000(app)
# NOT: uid=0(root)
```

**Common issue**:
```yaml
# ❌ WRONG - App runs as root
spec:
  containers:
  - name: app
    image: nginx  # Often defaults to root

# ✅ RIGHT - Force non-root
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101  # nginx user ID
```

---

### 2. fsGroup

**Purpose**: What group owns mounted volumes?

```yaml
spec:
  securityContext:
    fsGroup: 2000             # Volumes owned by GID 2000
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: data
      mountPath: /data       # This mount has GID 2000
  volumes:
  - name: data
    emptyDir: {}
```

**Why it matters**:
- Multiple containers in Pod need to share volumes
- fsGroup ensures all can read/write

**Verify**:
```bash
k exec <pod> -- ls -la /data
# Ownership: -rw-rwx--- root 2000
# Group is 2000
```

---

### 3. allowPrivilegeEscalation

**Purpose**: Can container use `sudo` or other privilege escalation tricks?

```yaml
# ✅ SECURE - Deny escalation
securityContext:
  allowPrivilegeEscalation: false

# ❌ INSECURE - Allow escalation
securityContext:
  allowPrivilegeEscalation: true
```

**Real-world scenario**:
```bash
# Inside container, try to escalate
k exec <pod> -- sudo whoami
# If allowPrivilegeEscalation: false → DENIED
# If true → Works (BAD)
```

**When to use**:
- **Always set to false** unless you have a specific reason
- Even with non-root user, privilege escalation is risky

---

### 4. readOnlyRootFilesystem

**Purpose**: Container can't write to disk (read-only filesystem).

```yaml
# ✅ SECURE - Read-only filesystem
securityContext:
  readOnlyRootFilesystem: true

# ❌ INSECURE - Can write anywhere
securityContext:
  readOnlyRootFilesystem: false
```

**Practical example**:
```yaml
spec:
  securityContext:
    readOnlyRootFilesystem: true
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx  # Only place container can write
    - name: cache
      mountPath: /var/cache/nginx
  volumes:
  - name: logs
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

**Verify**:
```bash
k exec <pod> -- touch /test
# Error: Read-only file system

k exec <pod> -- touch /var/log/nginx/test
# Success (mounted volume, writable)
```

**Why it matters**:
- Prevents malware from writing backdoors to disk
- Forces logs to go to stdout (good for Kubernetes)
- Containers should be stateless anyway

---

### 5. Capabilities

**Purpose**: Which Linux capabilities can the container use?

**What are capabilities?**
- Linux splits root powers into granular permissions
- Examples: CAP_NET_ADMIN, CAP_SYS_ADMIN, CAP_CHOWN, etc.

```yaml
# Drop ALL capabilities (most secure)
securityContext:
  capabilities:
    drop:
    - ALL

# Drop most, add back only what's needed
securityContext:
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE  # Allow binding to ports < 1024
```

**Common capabilities**:

| Capability | What It Allows | Dangerous? |
|------------|---|---|
| `CAP_SYS_ADMIN` | Lots of system admin powers | ⚠️ VERY |
| `CAP_NET_ADMIN` | Network configuration | ⚠️ HIGH |
| `CAP_SYS_PTRACE` | Debug other processes | ⚠️ HIGH |
| `CAP_CHOWN` | Change file ownership | ⚠️ MEDIUM |
| `CAP_NET_BIND_SERVICE` | Bind to ports < 1024 | ✅ LOW |
| `CAP_NET_RAW` | Create raw sockets | ⚠️ MEDIUM |

**Real-world example**:
```yaml
# nginx needs to bind to port 80 (< 1024)
# It's not root, so it needs this capability
spec:
  securityContext:
    runAsUser: 101  # nginx non-root user
    capabilities:
      drop:
      - ALL
      add:
      - NET_BIND_SERVICE
```

**Verify**:
```bash
# See what capabilities container has
k exec <pod> -- grep Cap /proc/self/status
# CapEff: 0000000000000400 (NET_BIND_SERVICE only)

# Try something it can't do
k exec <pod> -- ip link add dummy0 type dummy
# Error: Operation not permitted (no CAP_NET_ADMIN)
```

---

### 6. seccompProfile

**Purpose**: Filter which system calls container can make.

```yaml
# Use default profile (blocks dangerous syscalls)
securityContext:
  seccompProfile:
    type: RuntimeDefault

# Use custom profile (advanced)
securityContext:
  seccompProfile:
    type: Localhost
    localhostProfile: my-profile.json
```

**Common syscalls blocked by default**:
- `ptrace` - Process debugging
- `perfopen` - Performance monitoring
- `chroot` - Change root directory
- `keyctl` - Key management

**Verify**:
```bash
# Try blocked syscall
k exec <pod> -- strace -e trace=ptrace echo
# Error: ptrace is blocked
```

---

### 7. seLinuxOptions

**Purpose**: Apply SELinux labels to container.

```yaml
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
    type: "container_t"
```

**When to use**: If your cluster runs SELinux (usually not for CKS exam).

---

### 8. windowsOptions

**Purpose**: Windows-specific settings (not for CKS, skip).

---

## SecurityContext Examples

### Example 1: Maximum Security Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
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
        add:
        - NET_BIND_SERVICE
      readOnlyRootFilesystem: true
    
    volumeMounts:
    - name: logs
      mountPath: /app/logs
  
  volumes:
  - name: logs
    emptyDir: {}
```

**What this achieves**:
- ✅ Non-root (UID 1000)
- ✅ Can't escalate privileges
- ✅ Read-only filesystem (writes only to /app/logs)
- ✅ No dangerous capabilities
- ✅ Seccomp filters bad syscalls

### Example 2: Container Overriding Pod Context

```yaml
spec:
  securityContext:
    runAsUser: 1000
  
  containers:
  - name: app1
    image: app1
    # Uses Pod context: runAsUser 1000
  
  - name: app2
    image: app2
    securityContext:
      runAsUser: 2000  # Overrides Pod: runs as 2000
```

**Result**: app1 runs as 1000, app2 runs as 2000.

---

## Debugging SecurityContext Issues

### Issue 1: Pod Complains About Running as Root

```yaml
# Pod definition
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - name: app
    image: ubuntu  # Defaults to root
```

**Error**:
```
Error creating pod: container has runAsNonRoot and image will run as root
```

**Fix 1**: Use app that supports non-root
```yaml
image: myapp:v1  # Built to run as non-root
```

**Fix 2**: Explicitly set runAsUser in image
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    image: ubuntu  # Now will run as UID 1000
```

### Issue 2: "Permission Denied" Writing to Volume

```bash
k logs <pod>
# Error: permission denied: /data/app.log
```

**Cause**: fsGroup not set, container can't write.

**Fix**:
```yaml
spec:
  securityContext:
    fsGroup: 2000  # Volumes now writable by container
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /data
```

### Issue 3: "Operation Not Permitted" on Network

```bash
k exec <pod> -- ip link add dummy0 type dummy
# Error: Operation not permitted
```

**Cause**: Missing CAP_NET_ADMIN capability.

**Fix**:
```yaml
securityContext:
  capabilities:
    drop:
    - ALL
    add:
    - NET_ADMIN  # Add the capability
```

---

# Section 2: Pod Security Standards (PSS)

## What is PSS?

**Pod Security Standards** = Kubernetes-level enforcement of security best practices.

Replaces deprecated PodSecurityPolicy (PSP).

**How it works**: Label your namespace, Kubernetes auto-checks Pods against that level.

---

## Three Levels (Restricted → Baseline → Unrestricted)

### Level 1: Restricted (Most Secure)

**Rules enforced**:
- ✅ No privileged containers
- ✅ No privilege escalation
- ✅ No root user
- ✅ No dangerous capabilities
- ✅ Read-only root filesystem
- ✅ No host networking/IPC/PID
- ✅ Limited volume types (no hostPath)

**When to use**: Production, multi-tenant, security-critical clusters.

```yaml
# Pod must look like this to pass Restricted
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  
  containers:
  - name: app
    image: myapp
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
```

### Level 2: Baseline (Minimal)

**Rules enforced**:
- ✅ No privileged containers
- ✅ No root user allowed (runAsNonRoot: true)
- ✅ Some capabilities can be added back
- ✅ Read-only root filesystem NOT required
- ✅ Can use more volume types

**When to use**: Most clusters, default for new namespaces.

### Level 3: Unrestricted (No Enforcement)

**Rules enforced**: None. Anything goes.

**When to use**: Never. Unless debugging.

---

## How to Enforce PSS

### Method 1: Label Namespace (Most Common)

```bash
# Apply enforcement
k label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# What these mean:
# enforce=restricted → REJECT Pods that violate Restricted
# audit=restricted → Log violations (don't reject)
# warn=restricted → Warn users in API response
```

**Three modes** (can be different levels):

| Mode | Effect |
|------|--------|
| `enforce` | REJECT non-compliant Pods |
| `audit` | ALLOW but LOG violations |
| `warn` | ALLOW but WARN in response |

**Example**:
```bash
# Reject if violates Restricted, but warn if Baseline
k label namespace default \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=baseline
```

### Method 2: Apply to Multiple Namespaces

```bash
# Label all namespaces at once
for ns in default kube-system kube-public; do
  k label namespace $ns \
    pod-security.kubernetes.io/enforce=baseline \
    --overwrite
done

# Verify
k get ns --show-labels | grep pod-security
```

### Method 3: Label System Namespaces Differently

System namespaces need to run privileged components.

```bash
# Most namespaces: Baseline
k label namespace production \
  pod-security.kubernetes.io/enforce=baseline

# System namespaces: Allow privilege (they need it)
k label namespace kube-system \
  pod-security.kubernetes.io/enforce=privileged

k label namespace kube-public \
  pod-security.kubernetes.io/enforce=privileged
```

---

## Real Exam Scenarios

### Scenario 1: Enforce Restricted on Production Namespace

**Question**: Apply Restricted PSS to production namespace. Create Pod, verify it's rejected.

```bash
# 1. Label namespace
k label namespace production \
  pod-security.kubernetes.io/enforce=restricted

# 2. Try to create non-compliant Pod
k run insecure --image=nginx -n production
# Error: violates PodSecurityStandard "restricted"

# 3. Create compliant Pod
cat <<EOF | k apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
    fsGroup: 101
    seccompProfile:
      type: RuntimeDefault
  
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
      readOnlyRootFilesystem: true
    
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  
  volumes:
  - name: logs
    emptyDir: {}
EOF

# 4. Verify it's created
k get pods -n production
# secure-app should be Running
```

### Scenario 2: Mix Enforcement Levels

**Question**: Different namespaces need different security levels. Configure appropriately.

```bash
# Development: Relaxed (Baseline)
k label namespace development \
  pod-security.kubernetes.io/enforce=baseline

# Production: Strict (Restricted)
k label namespace production \
  pod-security.kubernetes.io/enforce=restricted

# Audit only (log violations, don't reject)
k label namespace staging \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=restricted

# Verify labels
k get ns --show-labels
# Shows which namespaces have PSS enforcement
```

### Scenario 3: Fix Deployment Violation

**Question**: Deployment in prod is rejected by Restricted PSS. Fix it.

```bash
# Get the Deployment
k get deployment app -n production -o yaml

# Error message shows: pod does not fit on any node (PSS violation)
# Likely issues:
# - Running as root
# - allowPrivilegeEscalation: true
# - readOnlyRootFilesystem missing

# Fix
k edit deployment app -n production

# Add securityContext:
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
      - name: app
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
        volumeMounts:
        - name: logs
          mountPath: /tmp
      volumes:
      - name: logs
        emptyDir: {}

# Save and verify
k get pods -n production
# Pods should now be Running
```

---

## Common SC + PSS Exam Questions

### Question Type 1: "Secure This Pod"

```yaml
# Given insecure Pod:
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: web
    image: nginx:latest
    # No SecurityContext!
```

**You must add**:
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
    fsGroup: 101
    seccompProfile:
      type: RuntimeDefault
  
  containers:
  - name: web
    image: nginx:latest
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
      readOnlyRootFilesystem: true
    
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  
  volumes:
  - name: logs
    emptyDir: {}
```

**Time**: Should take <5 minutes.

### Question Type 2: "Enforce Security Policy"

```bash
# Given: Namespace 'payment' needs Restricted enforcement

# You run:
k label namespace payment \
  pod-security.kubernetes.io/enforce=restricted

# Then verify non-compliant Pods are rejected
```

**Time**: <2 minutes.

### Question Type 3: "Debug PSS Violation"

```bash
# Deployment pods not starting
k describe pod <pod>
# Warning: violates PodSecurityStandard "restricted"

# You check Deployment YAML
k get deployment <name> -o yaml

# Find the issue (no securityContext, root user, etc.)
# Fix it
k edit deployment <name>
# Add/modify securityContext
# Save and verify pods restart
```

**Time**: <10 minutes.

---

## Cheat Sheet: SC Field Quick Reference

```yaml
# === POD-LEVEL ===
spec:
  securityContext:
    runAsUser: 1000                    # UID to run as
    runAsNonRoot: true                 # ENFORCE non-root
    fsGroup: 2000                      # Volume group ownership
    seccompProfile:
      type: RuntimeDefault             # Syscall filtering
    seLinuxOptions:                    # SELinux labels
      level: "s0:c123,c456"

# === CONTAINER-LEVEL ===
spec:
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false  # Block sudo/escalation
      capabilities:                    # Linux capabilities
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
      readOnlyRootFilesystem: true     # Immutable disk
      runAsUser: 1000                  # Override Pod-level
```

---

## Cheat Sheet: PSS Label Quick Reference

```bash
# === RESTRICT TO MAXIMUM SECURITY ===
k label namespace prod \
  pod-security.kubernetes.io/enforce=restricted

# === ALLOW BASELINE (COMMON) ===
k label namespace dev \
  pod-security.kubernetes.io/enforce=baseline

# === ALLOW ANYTHING (SYSTEM NS) ===
k label namespace kube-system \
  pod-security.kubernetes.io/enforce=privileged

# === THREE MODES ===
# enforce=X → REJECT violators
# audit=X → ALLOW but LOG violators
# warn=X → ALLOW but WARN users

# === EXAMPLE: DIFFERENT LEVELS ===
k label namespace prod \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=baseline
```

---

## Pre-Exam Validation

```bash
# 1. Create secure Pod quickly
time k run secure --image=nginx -o yaml > pod.yaml
# Add SC fields manually
# Should take <3 min

# 2. Apply PSS labels
time k label namespace default pod-security.kubernetes.io/enforce=restricted
# Should take <30 sec

# 3. Verify PSS rejection
k run root-app --image=ubuntu -n default -- sleep 3600
# Should be REJECTED

# 4. Create compliant Pod
# Should PASS

# 5. Debug PSS violations
k describe pod <violating-pod>
# Should see PSS warning in output
```

**Target**: You should master SC + PSS in <2 weeks of practice.

