# Audit Logging Masterclass
## CKS Deep Dive - Track Who Did What, When, and How

---

# Section 1: Audit Logging Fundamentals

## What is Audit Logging?

**Audit Logging** = K8s records EVERYTHING that happens in the cluster.

Every API request is logged: who did it, what resource, when, result.

**Think of it as**: Security camera + detailed ledger for cluster activity.

**Why it matters for CKS**:
- Detect unauthorized access attempts
- Track who modified critical resources
- Compliance (HIPAA, PCI-DSS, SOC2)
- Forensics after a breach
- Exam tests: Enable + parse + analyze logs

---

## What Gets Logged?

**Everything**:
- kubectl create/update/delete/patch operations
- Pod exec commands
- Secret access (dangerous!)
- RBAC changes
- Node registration
- Controller actions
- etcd operations

**Who's logging**:
- kube-apiserver (central audit point)
- Every request passes through → gets logged

---

## Audit Policy (What to Log, What to Skip)

Audit is **noisy** (millions of events/day). You configure what to log via **Audit Policy**.

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy

# Rules are evaluated top-to-bottom, first match wins
rules:

# Rule 1: Log Secret access (HIGH SECURITY)
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["secrets"]
  # This rule logs full request + response

# Rule 2: Log Pod exec (SUSPICIOUS)
- level: RequestResponse
  verbs: ["create"]
  resources:
  - group: ""
      resources: ["pods/exec", "pods/attach"]

# Rule 3: Log RBAC changes
- level: Metadata
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["clusterrolebindings", "rolebindings"]

# Rule 4: Log all Pod creation (medium priority)
- level: Metadata
  verbs: ["create"]
  resources:
  - group: ""
    resources: ["pods"]

# Rule 5: Catch-all (everything else, minimal logging)
- level: Metadata
  omitStages:
  - RequestReceived  # Don't log on request arrival, only on response

# Omit rules (what NOT to log)
- level: None
  verbs: ["watch", "list"]  # Skip watch/list (too noisy)
- level: None
  resources:
  - group: ""
    resources: ["events"]  # Skip events (spam)
```

---

## Audit Log Levels

| Level | What Gets Logged | Use When |
|-------|---|---|
| **None** | Nothing | Don't log this (skip events, watch, list) |
| **Metadata** | User, verb, resource, result | Most things (default) |
| **RequestResponse** | Metadata + full request + response body | Sensitive ops (secrets, RBAC) |
| **Request** | Metadata + request body (no response) | Rare, debugging |

**Memory rule**: `None < Metadata < Request < RequestResponse` (detail level)

---

# Section 2: Enabling Audit Logging

## Step 1: Create Audit Policy

```bash
# On control plane node
sudo cat <<EOF > /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# CRITICAL: Secret access
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["secrets"]

# HIGH: Pod exec/attach (suspicious activity)
- level: RequestResponse
  verbs: ["create"]
  resources:
  - group: ""
    resources: ["pods/exec", "pods/attach"]

# MEDIUM: RBAC changes
- level: Metadata
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["clusterrolebindings", "rolebindings", "clusterroles", "roles"]

# MEDIUM: Pod operations
- level: Metadata
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["pods"]

# LOW: Most other operations
- level: Metadata
  omitStages:
  - RequestReceived

# SKIP: Watch, list (too noisy)
- level: None
  verbs: ["watch", "list"]

# SKIP: Events (spam)
- level: None
  resources:
  - group: ""
    resources: ["events"]
EOF

sudo chmod 600 /etc/kubernetes/audit-policy.yaml
```

## Step 2: Configure kube-apiserver

Edit kube-apiserver manifest:

```bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add these flags:

```yaml
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver
    # ... existing flags ...
    
    # Audit logging flags
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-maxage=30                    # Keep logs 30 days
    - --audit-log-maxbackup=10                 # Keep 10 backup files
    - --audit-log-maxsize=100                  # Rotate when file > 100 MB
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    
    volumeMounts:
    - name: audit
      mountPath: /var/log/kubernetes/audit
    - name: audit-policy
      mountPath: /etc/kubernetes/audit-policy.yaml
      readOnly: true
  
  volumes:
  - name: audit
    hostPath:
      path: /var/log/kubernetes/audit
      type: DirectoryOrCreate
  - name: audit-policy
    hostPath:
      path: /etc/kubernetes/audit-policy.yaml
      type: File
```

## Step 3: Restart kubelet

```bash
sudo systemctl restart kubelet

# Wait 30 seconds for API server to restart
sleep 30

# Verify API server is running
k get pods -n kube-system | grep kube-apiserver
```

## Step 4: Verify Logs Are Being Written

```bash
# Check if audit logs exist
ls -la /var/log/kubernetes/audit/

# Should see: audit.log

# Check logs
tail -f /var/log/kubernetes/audit/audit.log

# Should see JSON events (one per line)
```

---

# Section 3: Parsing Audit Logs with jq

Audit logs are **JSON, one event per line**. Use `jq` to parse.

## Basic jq Queries

### Find All Secret Access

```bash
cat /var/log/kubernetes/audit/audit.log | \
  jq 'select(.objectRef.resource=="secrets")'

# Output:
{
  "level": "RequestResponse",
  "timestamp": "2026-09-25T10:15:30.123456Z",
  "user": {
    "username": "alice",
    "uid": "system:serviceaccount:default:admin"
  },
  "verb": "create",
  "objectRef": {
    "resource": "secrets",
    "namespace": "production",
    "name": "db-password"
  },
  "requestObject": {"kind": "Secret", "metadata": {"name": "db-password"}, ...},
  "responseStatus": {"code": 201}
}
```

### Find Who Deleted What

```bash
cat /var/log/kubernetes/audit/audit.log | \
  jq 'select(.verb=="delete")'

# Shows all delete operations with who did it
```

### Find Pod Exec Commands (Suspicious!)

```bash
cat /var/log/kubernetes/audit/audit.log | \
  jq 'select(.objectRef.resource=="pods/exec")'

# Output shows: who exec'd into which pod, when
```

### Find Failed Operations (Errors)

```bash
cat /var/log/kubernetes/audit/audit.log | \
  jq 'select(.responseStatus.code >= 400)'

# Shows all failed API calls (401 Unauthorized, 403 Forbidden, etc.)
```

### Find Specific User's Actions

```bash
cat /var/log/kubernetes/audit/audit.log | \
  jq 'select(.user.username=="alice")'

# Everything alice did
```

### Find RBAC Changes

```bash
cat /var/log/kubernetes/audit/audit.log | \
  jq 'select(.objectRef.resource=="clusterrolebindings" or .objectRef.resource=="rolebindings")'

# Who changed permissions
```

### Parse and Pretty-Print User Actions

```bash
cat /var/log/kubernetes/audit/audit.log | \
  jq '{
    timestamp: .requestReceivedTimestamp,
    user: .user.username,
    action: .verb,
    resource: .objectRef.resource,
    name: .objectRef.name,
    namespace: .objectRef.namespace,
    status: .responseStatus.code
  }'

# Output:
{
  "timestamp": "2026-09-25T10:15:30.123456Z",
  "user": "alice",
  "action": "create",
  "resource": "secrets",
  "name": "db-password",
  "namespace": "production",
  "status": 201
}
```

---

# Section 4: Real Exam Scenarios

## Scenario 1: Enable Audit Logging

**Question**: Enable audit logging to track secret access and RBAC changes.

```bash
# 1. Create policy
sudo cat <<EOF > /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["secrets"]

- level: Metadata
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["clusterrolebindings", "rolebindings"]

- level: Metadata
EOF

# 2. Edit kube-apiserver
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Add audit flags (see Step 2 above)

# 3. Restart
sudo systemctl restart kubelet

# 4. Verify
sleep 30
k get pods -n kube-system | grep apiserver
```

## Scenario 2: Find Who Created a Secret

**Question**: A secret "db-password" was created. When? By whom? Show the request.

```bash
# Query audit logs
cat /var/log/kubernetes/audit/audit.log | \
  jq 'select(.objectRef.resource=="secrets" and .objectRef.name=="db-password" and .verb=="create")'

# Output shows:
# - timestamp (when)
# - user.username (who)
# - requestObject (what was created)
# - responseStatus (success/fail)
```

## Scenario 3: Detect Pod Exec (Suspicious Activity)

**Question**: Find all instances of someone running shell commands inside pods (potential attack).

```bash
# Query
cat /var/log/kubernetes/audit/audit.log | \
  jq 'select(.objectRef.resource=="pods/exec") | {
    timestamp: .requestReceivedTimestamp,
    user: .user.username,
    pod: .objectRef.name,
    namespace: .objectRef.namespace,
    command: .requestObject.spec.command
  }'

# If found: pod was compromised or under investigation
```

## Scenario 4: Find Unauthorized Access Attempts

**Question**: Show all RBAC operations by non-system users.

```bash
cat /var/log/kubernetes/audit/audit.log | \
  jq 'select(
    (.objectRef.resource=="clusterrolebindings" or .objectRef.resource=="rolebindings") and
    (.user.username | contains("system") | not)
  ) | {
    timestamp: .requestReceivedTimestamp,
    user: .user.username,
    action: .verb,
    resource: .objectRef.resource,
    result: .responseStatus.code
  }'

# Shows non-system users changing permissions (audit trail)
```

## Scenario 5: Compliance Report - Track All Changes

**Question**: Generate compliance report of all infrastructure changes in last hour.

```bash
# Get changes in last hour (filter by timestamp)
# Parse with jq to show: who, when, what, result

cat /var/log/kubernetes/audit/audit.log | \
  jq 'select(
    .verb=="create" or .verb=="delete" or .verb=="patch" or .verb=="update"
  ) | {
    time: .requestReceivedTimestamp,
    user: .user.username,
    verb: .verb,
    resource: .objectRef.resource,
    name: .objectRef.name,
    namespace: .objectRef.namespace,
    status: .responseStatus.code
  }' | \
  head -20  # Last 20 changes

# Export to CSV for compliance
cat /var/log/kubernetes/audit/audit.log | \
  jq -r '[.requestReceivedTimestamp, .user.username, .verb, .objectRef.resource, .objectRef.name, .responseStatus.code] | @csv' > audit-report.csv
```

---

# Section 5: Audit Logging Best Practices

## What to Log (Priority Order)

| Priority | What | Why |
|---|---|---|
| **CRITICAL** | Secret create/update/delete | Attackers seek creds |
| **CRITICAL** | Pod exec/attach | Remote code execution |
| **HIGH** | RBAC changes | Privilege escalation |
| **HIGH** | Service account token access | Identity theft |
| **MEDIUM** | Pod creation | Malicious workloads |
| **MEDIUM** | Config/Deployment updates | Supply chain attacks |
| **LOW** | Normal reads (get, list) | Too noisy, low risk |
| **SKIP** | Watch, events | Extreme spam |

## Log Retention Policy

```yaml
# In kube-apiserver flags:
--audit-log-maxage=30         # Keep 30 days
--audit-log-maxbackup=10      # Keep 10 files max
--audit-log-maxsize=100       # Rotate at 100 MB
```

**Why**: Compliance requires 30-90 day retention. Disk fills quickly.

## Centralized Logging (Forward Logs)

Don't keep logs only on one node. Forward to SIEM:

```bash
# Option 1: Forward to Syslog
# Configure: syslog server, send audit.log there

# Option 2: Forward to CloudWatch/Splunk/ELK
# Ship logs to centralized system
# Can then: search, alert, correlate across nodes
```

---

# Section 6: Cheat Sheet

## Quick Audit Policy (Copy-Paste)

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Secrets (RequestResponse = full logging)
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["secrets"]

# Pod exec (catches remote access)
- level: RequestResponse
  verbs: ["create"]
  resources:
  - group: ""
    resources: ["pods/exec", "pods/attach"]

# RBAC (permission changes)
- level: Metadata
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["clusterrolebindings", "rolebindings", "clusterroles", "roles"]

# Default (everything else)
- level: Metadata

# Skip (too noisy)
- level: None
  verbs: ["watch", "list"]
- level: None
  resources:
  - group: ""
    resources: ["events"]
```

## Quick jq Commands

```bash
# Secrets accessed
jq 'select(.objectRef.resource=="secrets")'

# Pod exec (shell access)
jq 'select(.objectRef.resource=="pods/exec")'

# RBAC changes
jq 'select(.objectRef.resource | contains("role"))'

# Failed operations
jq 'select(.responseStatus.code >= 400)'

# Specific user
jq 'select(.user.username=="alice")'

# Formatted output
jq '{time: .requestReceivedTimestamp, user: .user.username, verb: .verb, resource: .objectRef.resource, result: .responseStatus.code}'

# Export as CSV
jq -r '[.requestReceivedTimestamp, .user.username, .verb, .objectRef.resource] | @csv'
```

## Quick Enable Steps

```bash
# 1. Create policy
sudo vi /etc/kubernetes/audit-policy.yaml

# 2. Edit kube-apiserver
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Add audit flags + volumes

# 3. Restart
sudo systemctl restart kubelet

# 4. Test
tail -f /var/log/kubernetes/audit/audit.log
```

---

# Pre-Exam Checklist

- ✅ Understand audit log levels (None, Metadata, Request, RequestResponse)
- ✅ Create audit policy for secrets + RBAC + pod exec
- ✅ Enable audit logging on kube-apiserver
- ✅ Verify logs are being written
- ✅ Parse logs with jq (at least 5 queries)
- ✅ Find specific events by user/resource/verb
- ✅ Export logs for compliance (CSV)
- ✅ Speed: Should take <15 min to enable + verify

