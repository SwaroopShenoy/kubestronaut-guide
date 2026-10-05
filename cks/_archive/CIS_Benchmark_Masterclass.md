# CIS Kubernetes Benchmark Masterclass
## CKS Deep Dive - What It Is, What Fails, How to Fix It

---

# What is CIS Benchmark?

**CIS** = Center for Internet Security  
**Benchmark** = Security hardening guidelines for Kubernetes

Think of it as: "Here are security best practices. Check your cluster against them."

**Why it matters for CKS**:
- Automated scanning (kube-bench) tests most of it
- Exam will ask you to fix violations
- Real clusters get audited against CIS regularly

---

# How kube-bench Works

```bash
# Install kube-bench
wget https://github.com/aquasecurity/kube-bench/releases/download/v0.6.10/kube-bench_0.6.10_linux_x86_64.tar.gz
tar xzf kube-bench_0.6.10_linux_x86_64.tar.gz
sudo mv kube-bench /usr/local/bin

# Run CIS checks
sudo kube-bench run --targets master,node,policies

# Output format:
# [PASS] 1.1.1 Ensure that the --cert-file and --key-file arguments are set
# [FAIL] 1.2.1 Ensure that the --authorization-mode argument is not set to AlwaysAllow
# [WARN] 1.2.2 Ensure that the --authorization-mode argument includes RBAC
# [INFO] 1.2.3 Ensure that the admission control plugin SecurityContextDeny is not set
```

**Status meanings**:
- **PASS**: Check passed, configuration is secure
- **FAIL**: Configuration violates security best practice (FIX THIS)
- **WARN**: Not critical but should review
- **INFO**: Informational, no action needed

---

# CIS Sections (What Gets Tested)

## Section 1: Control Plane Node Configuration

### 1.1: API Server

These checks verify kube-apiserver security flags.

#### 1.1.1 - 1.1.6: Certificate & Key Files

```bash
# CHECK: Does API server have --cert-file and --key-file?
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -E "cert-file|key-file"

# ✅ PASS: Should see
# - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
# - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key

# ❌ FAIL: If missing, add to manifest
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Add lines under spec.containers[0].command:
# - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
# - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key

# Restart kubelet (auto-restarts pod)
sudo systemctl restart kubelet
```

#### 1.2.1: Authorization Mode

```bash
# CHECK: Is AlwaysAllow disabled?
grep authorization-mode /etc/kubernetes/manifests/kube-apiserver.yaml

# ✅ PASS: Should see
# --authorization-mode=Node,RBAC

# ❌ FAIL: If set to AlwaysAllow or missing RBAC
# FIX: Edit manifest
# Change: --authorization-mode=AlwaysAllow
# To: --authorization-mode=Node,RBAC
```

#### 1.2.5: Request Timeout

```bash
# CHECK: --request-timeout set to reasonable value?
grep request-timeout /etc/kubernetes/manifests/kube-apiserver.yaml

# ✅ PASS: Should see
# --request-timeout=300s

# ❌ FAIL: If missing or 0
# FIX: Add
# --request-timeout=300s
```

#### 1.2.8: Kubelet Certificate Authority

```bash
# CHECK: Does API server know kubelet's CA?
grep kubelet-client-certificate /etc/kubernetes/manifests/kube-apiserver.yaml

# ✅ PASS: Should see
# --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
# --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
```

#### 1.2.9: Kubelet HTTPS

```bash
# CHECK: Does kubelet use HTTPS?
grep -E "kubelet-https|kubelet-port" /etc/kubernetes/manifests/kube-apiserver.yaml

# ✅ PASS: Should NOT see --kubelet-https=false
# Should NOT see --kubelet-port=10255

# ❌ FAIL: If kubelet-https=false
# FIX: Remove the flag (defaults to true)
```

#### 1.2.31: Disable Insecure Port

```bash
# CHECK: Is insecure port disabled?
grep insecure /etc/kubernetes/manifests/kube-apiserver.yaml

# ✅ PASS: Should see
# --insecure-port=0

# ❌ FAIL: If insecure-port=8080 or missing
# FIX: Add --insecure-port=0
```

#### 1.2.33: Encryption at Rest

```bash
# CHECK: Are resources encrypted at rest?
grep encryption /etc/kubernetes/manifests/kube-apiserver.yaml

# ✅ PASS: Should see
# --encryption-provider-config=/etc/kubernetes/encryption.yaml

# ❌ FAIL: If missing
# Need to create encryption config (covered in secrets section)
```

### 1.3: Controller Manager

```bash
# CHECK: Controller manager hardening
cat /etc/kubernetes/manifests/kube-controller-manager.yaml

# Key checks:
# --terminated-pod-gc-threshold=10 (exist)
# --use-service-account-credentials=true
# --service-account-private-key-file=/etc/kubernetes/pki/sa.key
```

### 1.4: Scheduler

```bash
# CHECK: Scheduler binding
grep bind-address /etc/kubernetes/manifests/kube-scheduler.yaml

# ✅ PASS: Should see
# --bind-address=127.0.0.1 (NOT 0.0.0.0)

# ❌ FAIL: If 0.0.0.0 (exposed to network)
# FIX: Change to 127.0.0.1
```

---

## Section 2: etcd

### 2.1: etcd Server Configuration

```bash
# CHECK: etcd security
cat /etc/kubernetes/manifests/etcd.yaml

# Key checks:
# --cert-file (has TLS cert)
# --key-file (has TLS key)
# --client-cert-auth=true (client auth required)
# --peer-cert-file (peer TLS)
# --peer-client-cert-auth=true

# ✅ All should be present

# ❌ FAIL Example: client-cert-auth missing
# FIX: Add --client-cert-auth=true
```

### 2.2: etcd Data Permissions

```bash
# CHECK: Who can read etcd database?
ls -la /var/lib/etcd/

# ✅ PASS: Should see
# drwx------ (700 permissions, only root)

# ❌ FAIL: If world-readable or group-readable
# FIX:
sudo chmod 700 /var/lib/etcd/
```

---

## Section 3: Control Plane General

### 3.1: RBAC

```bash
# CHECK: RBAC enabled
grep authorization-mode /etc/kubernetes/manifests/kube-apiserver.yaml
# Should include RBAC

# ✅ PASS: --authorization-mode=Node,RBAC

# ❌ FAIL: Missing RBAC
# FIX: Add RBAC to authorization-mode
```

### 3.2: Logging Levels

```bash
# CHECK: Verbose logging for debugging
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep "\-v="

# ✅ WARN/PASS: Should see
# --v=2 (moderate logging, not verbose)

# ❌ FAIL: If --v=4 or higher (too verbose)
# FIX: Change to --v=2
```

---

## Section 4: Worker Node Configuration

### 4.1: Kubelet Configuration

Most failures happen here. This is exam-heavy.

#### 4.1.1: Kubelet Configuration File

```bash
# CHECK: Does kubelet use config file?
ps aux | grep kubelet | grep -o "config=[^ ]*"
# Should see: --config=/var/lib/kubelet/config.yaml

# ✅ PASS: Config file exists and is used

# ❌ FAIL: If running with flags instead of config
# FIX: Move flags to config file at /var/lib/kubelet/config.yaml
```

#### 4.1.2: Kubelet Client Certificate Authority

```bash
# CHECK: In kubelet config
cat /var/lib/kubelet/config.yaml | grep serverTLSBootstrap

# ✅ PASS: Should see
# serverTLSBootstrap: true

# This allows kubelet to bootstrap its own certs
```

#### 4.1.6: Kubelet Authorization Mode

```bash
# CHECK: In kubelet config
cat /var/lib/kubelet/config.yaml | grep -A 2 "authorization:"

# ✅ PASS: Should see
# authorization:
#   mode: Webhook

# ❌ FAIL: If AlwaysAllow
# FIX: Edit config, change to Webhook
# Restart kubelet
sudo systemctl restart kubelet
```

#### 4.1.7: Kubelet Read-Only Port DISABLED

```bash
# CHECK: Is read-only port disabled?
cat /var/lib/kubelet/config.yaml | grep readOnlyPort

# ✅ PASS: Should see
# readOnlyPort: 0

# ❌ FAIL: If readOnlyPort: 10255
# FIX: Change to 0
# This exposes kubelet metrics without auth!

# Restart kubelet
sudo systemctl restart kubelet
```

#### 4.1.8: Kubelet Anonymous Auth DISABLED

```bash
# CHECK: Can anonymous users access kubelet?
cat /var/lib/kubelet/config.yaml | grep anonymousAuth

# ✅ PASS: Should see
# anonymousAuth: false

# ❌ FAIL: If true or missing
# FIX: Add/change to false
sudo systemctl restart kubelet
```

#### 4.1.10: Kubelet Event QPS

```bash
# CHECK: Event rate limiting
cat /var/lib/kubelet/config.yaml | grep eventRecordQPS

# ✅ PASS: Should see
# eventRecordQPS: 5 (not 0)

# ❌ FAIL: If 0 (unlimited events)
# FIX: Set to reasonable value (5 is common)
```

#### 4.1.12: Kubelet Event TTL

```bash
# CHECK: How long events are kept
cat /var/lib/kubelet/config.yaml | grep eventRecordQPS

# Should exist, reasonable value prevents log spam
```

#### 4.2.1: Kubelet File Permissions

```bash
# CHECK: kubeconfig permissions
ls -la /etc/kubernetes/kubelet.conf

# ✅ PASS: Should see
# -rw------- (600 permissions)

# ❌ FAIL: If -rw-r--r-- (world readable)
# FIX:
sudo chmod 600 /etc/kubernetes/kubelet.conf
```

#### 4.2.2: Kubelet Certificate Authorities File

```bash
# CHECK: CA cert file permissions
ls -la /var/lib/kubelet/pki/ca.crt 2>/dev/null || ls -la /etc/kubernetes/pki/ca.crt

# ✅ PASS: Should be readable by kubelet only
# Not world-writable
```

#### 4.2.7: Kubelet Make Read-Only Filesystem

```bash
# CHECK: Pod root filesystem is read-only?
cat /var/lib/kubelet/config.yaml | grep readOnlyRootFilesystem

# ⚠️  This is pod-level, not kubelet-level
# For PASS: Pods should have readOnlyRootFilesystem: true
# But exam might not check this at kubelet config level
```

---

## Section 5: Policies

### 5.1: RBAC Policy

```bash
# CHECK: Excessive permissions
k get clusterrolebindings -o json | jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'

# ✅ PASS: Should only see system accounts
# - system:masters
# - system:kube-*

# ❌ FAIL: If regular users bound to cluster-admin
# FIX: Remove binding
k delete clusterrolebinding <name>
```

### 5.2: Pod Security Policy / Pod Security Standards

```bash
# CHECK: Is PSP or PSS enabled?
grep pod-security /etc/kubernetes/manifests/kube-apiserver.yaml

# ✅ PASS: Should see
# --pod-security-policy=restricted
# OR namespace labels for PSS

# ❌ FAIL: If nothing
# FIX: Enable Pod Security Standards
k label namespace kube-system pod-security.kubernetes.io/enforce=baseline
k label namespace default pod-security.kubernetes.io/enforce=restricted
```

### 5.3: Network Policies

```bash
# CHECK: Are network policies defined?
k get networkpolicies --all-namespaces

# ✅ PASS: Should see default-deny policies

# ❌ FAIL: If none (all pods can talk)
# FIX: Create NetworkPolicy (covered in networking section)
```

### 5.4: Secrets

```bash
# CHECK: Are secrets encrypted at rest?
grep encryption /etc/kubernetes/manifests/kube-apiserver.yaml

# ✅ PASS: Should see --encryption-provider-config

# ❌ FAIL: If missing
# FIX: Configure etcd encryption
```

### 5.5: Privileged Access

```bash
# CHECK: Who has admin access?
k get clusterrolebindings | grep cluster-admin

# ✅ PASS: Only service accounts should have it

# ❌ FAIL: If regular users
# FIX: Remove excess bindings
```

---

# Real Exam Scenarios

## Scenario 1: Fix Multiple CIS Failures

**Question**: Run kube-bench, fix top 5 failures on master node.

```bash
# 1. Run kube-bench, capture failures
sudo kube-bench run --targets master > bench-results.txt
grep FAIL bench-results.txt

# Common failures:
# [FAIL] 1.2.1 Ensure authorization-mode is not AlwaysAllow
# [FAIL] 1.2.31 Ensure insecure-port is disabled
# [FAIL] 1.2.33 Encryption at rest is configured
# [FAIL] 4.1.7 kubelet read-only-port is disabled
# [FAIL] 4.1.8 kubelet anonymous auth disabled

# 2. Fix 1.2.1: Authorization Mode
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Change: --authorization-mode=AlwaysAllow
# To: --authorization-mode=Node,RBAC
# Save and restart (kubelet auto-restarts)
sudo systemctl restart kubelet

# 3. Fix 1.2.31: Insecure Port
# Add to kube-apiserver.yaml:
# --insecure-port=0
# Restart

# 4. Fix 1.2.33: Encryption at Rest
# Create /etc/kubernetes/encryption.yaml
# Add --encryption-provider-config=/etc/kubernetes/encryption.yaml
# Restart

# 5. Fix 4.1.7 & 4.1.8: Kubelet Config
sudo vi /var/lib/kubelet/config.yaml
# Change: readOnlyPort: 10255 → readOnlyPort: 0
# Change: anonymousAuth: true → anonymousAuth: false
# Restart:
sudo systemctl restart kubelet

# 6. Verify fixes
sudo kube-bench run --targets master | grep -E "PASS|FAIL" | grep "1.2.1\|1.2.31\|1.2.33\|4.1.7\|4.1.8"
```

## Scenario 2: Identify and Document Non-Compliance

**Question**: Run CIS audit, document what's not compliant and why it matters.

```bash
# Run with JSON output for parsing
sudo kube-bench run --targets master,node -j > cis-report.json

# Parse for failures
cat cis-report.json | jq '.[] | select(.results[] | select(.test_number=="1.2.1")) | .results[] | select(.test_number=="1.2.1")'

# Document format:
# Check ID: 1.2.1
# Check Name: Ensure that the --authorization-mode argument is not set to AlwaysAllow
# Current Status: FAIL
# Severity: CRITICAL
# Impact: Without RBAC, any authenticated user can do anything
# Remediation: Add --authorization-mode=Node,RBAC to kube-apiserver
# Verification: grep authorization-mode kube-apiserver.yaml
```

---

# Top 10 Most-Tested CIS Checks on Exam

Based on real exam reports, these are what appears most:

| # | Check | What It Does | Fix | Difficulty |
|---|-------|-------------|-----|------------|
| 1 | 1.2.1 | Authorization mode not AlwaysAllow | Set to Node,RBAC | Easy |
| 2 | 1.2.31 | Insecure port disabled | Set to 0 | Easy |
| 3 | 4.1.7 | Kubelet read-only port disabled | Set readOnlyPort: 0 | Easy |
| 4 | 4.1.8 | Kubelet anonymous auth disabled | Set anonymousAuth: false | Easy |
| 5 | 1.2.33 | Encryption at rest configured | Create encryption.yaml | Medium |
| 6 | 1.2.9 | Kubelet HTTPS | Don't disable HTTPS | Easy |
| 7 | 4.1.6 | Kubelet authorization mode | Set to Webhook | Medium |
| 8 | 2.1.x | etcd security | TLS certs, client auth | Medium |
| 9 | 5.2.x | Pod Security Standards | Label namespaces | Easy |
| 10 | 3.1.x | RBAC enabled | Add to authorization-mode | Easy |

---

# Quick Fix Checklist (Save This)

```bash
# === KUBE-APISERVER CRITICAL FIXES ===

# 1. Authorization mode
--authorization-mode=Node,RBAC

# 2. Disable insecure port
--insecure-port=0

# 3. Enable encryption
--encryption-provider-config=/etc/kubernetes/encryption.yaml

# 4. TLS certificates
--tls-cert-file=/etc/kubernetes/pki/apiserver.crt
--tls-private-key-file=/etc/kubernetes/pki/apiserver.key

# 5. Kubelet client auth
--kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
--kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key

# === KUBELET CRITICAL FIXES ===

# In /var/lib/kubelet/config.yaml:

# Disable read-only port
readOnlyPort: 0

# Disable anonymous auth
anonymousAuth: false

# Enable webhook auth
authorization:
  mode: Webhook

# Enable client cert auth
clientCAFile: /etc/kubernetes/pki/ca.crt

# === CONTROLLER MANAGER ===

--use-service-account-credentials=true
--service-account-private-key-file=/etc/kubernetes/pki/sa.key

# === SCHEDULER ===

--bind-address=127.0.0.1  # NOT 0.0.0.0

# === ETCD ===

--client-cert-auth=true
--peer-client-cert-auth=true
--cert-file=/etc/kubernetes/pki/etcd/server.crt
--key-file=/etc/kubernetes/pki/etcd/server.key

# === FILE PERMISSIONS ===

chmod 600 /etc/kubernetes/kubelet.conf
chmod 600 /etc/kubernetes/scheduler.conf
chmod 600 /etc/kubernetes/controller-manager.conf
chmod 700 /var/lib/etcd/
```

---

# How to Quickly Find & Fix Issues

## Time-Saving Workflow

```bash
# 1. Run kube-bench, save output
sudo kube-bench run --targets master > /tmp/bench.txt

# 2. Extract just failures
grep FAIL /tmp/bench.txt | head -10

# 3. For each failure, map to config file:
# Check ID 1.2.X → /etc/kubernetes/manifests/kube-apiserver.yaml
# Check ID 1.3.X → /etc/kubernetes/manifests/kube-controller-manager.yaml
# Check ID 1.4.X → /etc/kubernetes/manifests/kube-scheduler.yaml
# Check ID 2.X.X → /etc/kubernetes/manifests/etcd.yaml
# Check ID 4.1.X → /var/lib/kubelet/config.yaml
# Check ID 4.2.X → File permissions in /etc/kubernetes/

# 4. Edit config
sudo vi <file>

# 5. Restart
sudo systemctl restart kubelet  # For static pods
# Wait ~30 seconds for pod to restart

# 6. Verify
sudo kube-bench run --targets master | grep "1.2.31"  # Should now PASS
```

---

# Common Gotchas

❌ **Edited manifest but pod didn't restart**  
→ Need to wait 30 seconds, kubelet watches the directory  
→ Or manually restart: `sudo systemctl restart kubelet`

❌ **Can't find kubelet config**  
→ Check: `ps aux | grep kubelet | grep config`  
→ Usually at `/var/lib/kubelet/config.yaml`

❌ **Changed config but kube-bench still fails**  
→ Some changes require kubelet restart  
→ `sudo systemctl restart kubelet` and wait 30s

❌ **readOnlyPort shows FAIL even after setting to 0**  
→ Must restart kubelet  
→ Verify: `sudo netstat -tlnp | grep kubelet` should NOT show 10255

❌ **anonymousAuth: false but kube-bench still fails**  
→ Verify it's actually in config: `cat /var/lib/kubelet/config.yaml | grep anon`  
→ Might need systemctl restart + wait 1 min

---

# Pro Tips for Exam

1. **Know the file locations by heart**:
   - API server: `/etc/kubernetes/manifests/kube-apiserver.yaml`
   - Kubelet config: `/var/lib/kubelet/config.yaml`
   - Kubelet credentials: `/etc/kubernetes/kubelet.conf`
   - etcd: `/etc/kubernetes/manifests/etcd.yaml`

2. **Run kube-bench first**:
   - It tells you exactly what's wrong
   - Copy the check ID from output
   - Google/docs: "CIS Kubernetes 1.X.X" for remediation

3. **Restart kubelet when unsure**:
   - Most config changes need kubelet restart
   - It's safe, takes 30 seconds
   - Pods auto-restart, no downtime

4. **Test your fixes**:
   ```bash
   # Re-run kube-bench on that specific check
   sudo kube-bench run --targets master | grep "1.2.1"
   # Should now show [PASS]
   ```

5. **Document what you changed**:
   - CKS has a note-taking feature in exam
   - Write down what you changed and why
   - Helps with confidence + if you need to re-check

---

# Pre-Exam Practice

```bash
# On your test cluster:

# 1. Run full CIS audit
sudo kube-bench run --targets master,node

# 2. Pick top 5 FAILs
# 3. Fix each one
# 4. Time yourself (should take <15 min per fix)
# 5. Verify with kube-bench again
# 6. Repeat until all PASS

# Target: You should be able to identify + fix any CIS failure in <3 minutes
```

