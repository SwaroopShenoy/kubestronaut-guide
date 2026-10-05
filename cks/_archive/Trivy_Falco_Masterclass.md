# Trivy + Falco Masterclass
## CKS Deep Dive - Image Security & Runtime Threat Detection

---

# PART 1: TRIVY (Image Scanning)

## What is Trivy?

**Trivy** = Image vulnerability scanner (checks for CVEs).

Scans container images for security problems BEFORE deployment.

**Why it matters for CKS**:
- Exam tests: Can you identify vulnerable images?
- Exam tests: Can you interpret Trivy output?
- Exam tests: Can you block vulnerable images from deploying?

---

## Installing Trivy

```bash
# Download and install
wget https://github.com/aquasecurity/trivy/releases/download/v0.46.0/trivy_0.46.0_Linux-64bit.tar.gz
tar xzf trivy_0.46.0_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin

# Verify
trivy version
```

---

## Basic Trivy Scanning

### Scan Local Image

```bash
# Scan image already pulled locally
trivy image nginx:1.25

# Output:
# nginx (debian 12.1)
# ════════════════════════════════════════════════════════════════════
# Total: 42 (CRITICAL: 3, HIGH: 8, MEDIUM: 15, LOW: 16)
#
# CRITICAL [0-9]
# ════════════════════════════════════════════════════════════════════
# OpenSSL: CVE-2023-52356
#   Severity: CRITICAL
#   Description: Heap memory corruption in OpenSSL
#   Fixed Version: 3.0.13
```

### Scan from Registry (without pulling)

```bash
# Scan directly from Docker Hub
trivy image docker.io/library/nginx:latest

# Scan from private registry
trivy image myregistry.azurecr.io/myapp:v1
```

### Scan Local Dockerfile/Directory

```bash
# Scan all images in directory
trivy image --input=/path/to/image.tar

# Scan during build (before push)
docker build -t myapp:v1 .
trivy image myapp:v1
# If CRITICAL found: don't push
```

---

## Interpreting Trivy Output

### Severity Levels

```
CRITICAL: Immediate exploit risk, patch NOW
HIGH:     Serious vulnerability, patch ASAP
MEDIUM:   Moderate risk, plan patching
LOW:      Minor issues, patch eventually
```

### Real Example Output

```
# Image: nginx:1.25
# ════════════════════════════════════════════════════════════════════
# 
# CRITICAL (3)
# ════════════════════════════════════════════════════════════════════
# CVE-2023-52356  OpenSSL        0-9              3.0.0 -> 3.0.13
#   Description: Heap memory corruption in OpenSSL
#   Severity: CRITICAL
#   Link: https://nvd.nist.gov/vuln/detail/CVE-2023-52356
# 
# CVE-2023-44487  HTTP/2         Unspecified      Vulnerable -> N/A
#   Description: HTTP/2 rapid reset attack
#   Severity: CRITICAL
# 
# HIGH (8)
# ════════════════════════════════════════════════════════════════════
# CVE-2023-46604  Log4j           2.0 -> 2.20.0   
#   Description: Remote code execution in Log4j
#   Severity: HIGH
#
# MEDIUM (15)
# ... (15 medium severity issues)
#
# LOW (16)
# ... (16 low severity issues)
```

**Reading tips**:
- `3.0.0 -> 3.0.13` = Fixed in version 3.0.13 (upgrade from current)
- `Unspecified` = No fixed version yet (mitigation required)
- `Vulnerable -> N/A` = No patch available

---

## Trivy Output Formats

### JSON Output (Parse Programmatically)

```bash
# Output as JSON
trivy image nginx:1.25 -f json -o results.json

# Parse with jq
cat results.json | jq '.Results[] | select(.Severity=="CRITICAL")'
```

### SARIF Output (Security reporting)

```bash
# GitHub Actions / SARIF format
trivy image nginx:1.25 -f sarif -o results.sarif
```

### Table Output (Human readable, default)

```bash
trivy image nginx:1.25 -f table
```

---

## Filtering Results

### Show Only CRITICAL Vulnerabilities

```bash
# Flag: --severity
trivy image --severity CRITICAL nginx:1.25

# Only show images with CRITICAL or HIGH
trivy image --severity CRITICAL,HIGH nginx:1.25
```

### Exit with Code if Vulnerabilities Found

```bash
# Useful for CI/CD gates
trivy image nginx:1.25

# Exit codes:
# 0 = No vulnerabilities found
# 1 = Vulnerabilities found

# Use in scripts:
if trivy image myapp:v1; then
  docker push myapp:v1
else
  echo "Image has vulnerabilities, not pushing"
  exit 1
fi
```

---

## Real Exam Scenarios: Trivy

### Scenario 1: Scan Image, Identify Vulnerabilities

**Question**: Scan myapp:v1 image. How many CRITICAL CVEs?

```bash
# Run scan
trivy image myapp:v1

# Count CRITICAL
trivy image --severity CRITICAL myapp:v1 | grep CVE | wc -l

# Answer: "3 CRITICAL CVEs found"
```

### Scenario 2: Find Fixed Version for Vulnerability

**Question**: nginx:1.25 has CVE-2023-52356. What version fixes it?

```bash
trivy image nginx:1.25 | grep -A 3 "CVE-2023-52356"
# Output shows: Fixed Version: 3.0.13

# Answer: "Upgrade OpenSSL to 3.0.13"
```

### Scenario 3: Scan and Reject if CRITICAL Found

**Question**: Implement gate to block images with CRITICAL CVEs.

```bash
#!/bin/bash
IMAGE=$1

trivy image --severity CRITICAL $IMAGE
if [ $? -eq 0 ]; then
  echo "✓ Image approved for deployment"
  exit 0
else
  echo "✗ Image has CRITICAL vulnerabilities"
  exit 1
fi
```

### Scenario 4: Compare Two Image Versions

**Question**: nginx:1.24 vs nginx:1.25 - which is more secure?

```bash
# Scan both
trivy image nginx:1.24 > scan-1.24.txt
trivy image nginx:1.25 > scan-1.25.txt

# Compare
diff scan-1.24.txt scan-1.25.txt

# Or count:
echo "1.24 CRITICAL count:"
grep "CRITICAL" scan-1.24.txt | wc -l

echo "1.25 CRITICAL count:"
grep "CRITICAL" scan-1.25.txt | wc -l
```

---

## Trivy in CI/CD (How to Use in Real Pipeline)

### GitHub Actions Example

```yaml
name: Container Security Scan

on: [push]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
    - name: Build image
      run: docker build -t myapp:${{ github.sha }} .
    
    - name: Run Trivy scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: myapp:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload scan results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Fail if CRITICAL found
      run: |
        trivy image --severity CRITICAL myapp:${{ github.sha }}
```

**Result**: If CRITICAL CVEs found, pipeline fails, image not pushed. ✅

---

# PART 2: FALCO (Runtime Security)

## What is Falco?

**Falco** = Runtime security monitoring. Detects suspicious behavior WHILE container is running.

**Think of it as**: Security camera inside containers, watching for:
- Unauthorized process execution
- File system tampering
- Network anomalies
- Privilege escalation attempts

**Why it matters for CKS**:
- Exam tests: Deploy Falco
- Exam tests: Understand Falco rules
- Exam tests: Interpret Falco alerts
- Exam tests: Respond to detected threats

---

## Installing Falco

### Method 1: Install on Node (Host)

```bash
# Add repo
curl -s https://falco.org/repo/falcosecurity-3672BA8F.asc | apt-key add -
echo "deb https://download.falco.org/packages/deb stable main" | tee /etc/apt/sources.list.d/falcosecurity.list

# Install
sudo apt-get update
sudo apt-get install -y falco

# Start
sudo systemctl start falco
sudo systemctl enable falco

# View logs
sudo journalctl -u falco -f
```

### Method 2: Kubernetes DaemonSet (Recommended for CKS)

```bash
# Deploy Falco to all nodes
k apply -f https://raw.githubusercontent.com/falcosecurity/falco/master/deploy/kubernetes/falco-daemonset.yaml

# Verify
k get daemonset -n falco
k get pods -n falco

# View alerts
k logs -f -l app=falco -n falco
```

---

## Falco Rules (How Falco Detects Threats)

### Rule Structure

```yaml
# /etc/falco/rules.d/custom-rules.yaml

- rule: Unauthorized Shell Access
  desc: Detects shell spawned in production container
  
  condition: >
    spawned_process and                           # A process was spawned
    container and                                 # Inside a container
    container.labels["environment"] == "prod" and # In prod namespace
    proc.name in (bash, sh, /bin/bash, /bin/sh)  # Process is shell
  
  output: >
    ALERT: Shell spawned in production
    (container=%container.name user=%user.name process=%proc.name)
  
  priority: WARNING
  tags: [shell_access, unauthorized]
```

### Condition Fields Explained

```yaml
# Process fields
spawned_process          # Process was created
proc.name                # Name of process (bash, nginx, etc.)
proc.args                # Arguments passed to process
proc.uid                 # User ID running process
user.name                # Username running process

# Container fields
container                # True if in container
container.name           # Container name
container.image          # Container image
container.labels         # Container labels
container.privileged     # Is it privileged?

# File system fields
open                     # File was opened
write                    # File was written
fd.name                  # File path
fd.directory             # File directory

# Network fields
outbound                 # Outbound connection
inbound                  # Inbound connection
connection               # Any connection
fd.sip                   # Source IP
fd.dip                   # Destination IP
fd.sport                 # Source port
fd.dport                 # Destination port
```

---

## Built-in Falco Rules (What Gets Detected)

### Category 1: Process Execution

```yaml
- rule: Shell in Container
  condition: >
    spawned_process and container and
    proc.name in (bash, sh, /bin/bash, /bin/sh)
  output: >
    ALERT: Shell spawned in container
    (container=%container.name process=%proc.name)
  priority: WARNING
```

**Triggers when**: Any shell spawns in a container (usually means someone SSH'd in or exploit ran).

### Category 2: File System Tampering

```yaml
- rule: Write to System Files
  condition: >
    write and container and
    fd.name glob /etc/*
  output: >
    ALERT: Write to /etc (container=%container.name file=%fd.name user=%user.name)
  priority: CRITICAL
```

**Triggers when**: Container tries to modify /etc (config files).

### Category 3: Privilege Escalation

```yaml
- rule: Sudo Without Tty
  condition: >
    spawned_process and container and
    proc.name == sudo and
    not proc_interact
  output: >
    ALERT: Sudo executed (container=%container.name user=%user.name)
  priority: WARNING
```

**Triggers when**: Container tries to use sudo (escape attempt).

### Category 4: Network Anomalies

```yaml
- rule: Suspicious Outbound Connection
  condition: >
    outbound and container and
    fd.dip not in (allowed_ips) and
    container.labels["network"] == "restricted"
  output: >
    ALERT: Outbound connection to suspicious IP
    (container=%container.name destination=%fd.dip)
  priority: HIGH
```

**Triggers when**: Container connects to unexpected IP.

### Category 5: Cryptomining Detection

```yaml
- rule: Crypto Mining Process
  condition: >
    spawned_process and container and
    proc.name in (xmrig, monero-miner, ethminer)
  output: >
    ALERT: Cryptominer detected
    (container=%container.name process=%proc.name)
  priority: CRITICAL
```

**Triggers when**: Known cryptomining tools run.

---

## Falco Alert Levels

| Priority | Severity | Response |
|----------|----------|----------|
| `EMERGENCY` | System unusable | IMMEDIATE shutdown |
| `ALERT` | Action required immediately | Page on-call engineer |
| `CRITICAL` | Critical condition | Investigate immediately |
| `ERROR` | Error condition | Investigate within 1 hour |
| `WARNING` | Warning condition | Monitor, investigate |
| `NOTICE` | Normal but significant | Log for audit |
| `INFORMATIONAL` | Informational | Log only |
| `DEBUG` | Debug level | Development only |

---

## Real Exam Scenarios: Falco

### Scenario 1: Deploy Falco and Detect Threat

**Question**: Deploy Falco, run cryptominer in container, capture alert.

```bash
# 1. Deploy Falco
k apply -f https://raw.githubusercontent.com/falcosecurity/falco/master/deploy/kubernetes/falco-daemonset.yaml

# 2. Verify it's running
k get pods -n falco

# 3. Deploy pod with cryptominer
k run attacker --image=ubuntu -it -- xmrig -o pool.monero.com

# 4. Check Falco logs
k logs -f -l app=falco -n falco | grep "xmrig"

# Output should show ALERT about cryptominer detected
```

### Scenario 2: Detect Unauthorized Shell

**Question**: Detect when someone shells into a production pod.

```bash
# 1. Falco is running (already deployed)

# 2. Shell into production pod
k exec -it deployment/api -n production -- bash

# 3. Falco alerts
k logs -f -l app=falco -n falco | grep "Shell spawned"

# Output: "ALERT: Shell spawned in production (container=api-xxx user=root process=/bin/bash)"
```

### Scenario 3: Write Sensitive File Detection

**Question**: Container writes to /etc/passwd, Falco detects.

```bash
# 1. Inside container
k exec -it deployment/app -- bash

# 2. Attacker tries to modify /etc
bash-4.2# echo "hacker:x:0:0::/:/bin/bash" >> /etc/passwd

# 3. Falco alerts
k logs -f -l app=falco -n falco | grep "Write to /etc"

# Output: "ALERT: Write to /etc (container=app-xxx file=/etc/passwd user=root)"
```

### Scenario 4: Suspicious Network Connection

**Question**: Pod makes outbound connection to known C2 server, Falco detects.

```bash
# Custom Falco rule to detect suspicious IPs:
cat <<EOF > /etc/falco/rules.d/custom.yaml
- rule: Outbound to Suspicious IP
  desc: Detects connection to known C2 server
  condition: >
    outbound and container and
    fd.dip == "203.0.113.10"  # Known attacker IP
  output: >
    ALERT: Suspicious outbound connection
    (container=%container.name destination=%fd.dip port=%fd.dport)
  priority: CRITICAL
EOF

# Restart Falco
sudo systemctl restart falco

# Test: Pod connects to malicious IP
k exec deployment/app -- curl 203.0.113.10

# Falco alerts
sudo journalctl -u falco -f | grep "Suspicious outbound"
```

---

## Falco Macros & Lists (For Custom Rules)

### Macros (Reusable Conditions)

```yaml
# Define macro
- macro: shell_procs
  condition: >
    proc.name in (bash, sh, /bin/bash, /bin/sh, zsh, ksh)

# Use macro in rule
- rule: Unauthorized Shell
  condition: >
    spawned_process and container and
    shell_procs
  output: >
    ALERT: Unauthorized shell spawned
  priority: WARNING
```

### Lists (Reusable Collections)

```yaml
# Define list
- list: sensitive_files
  items:
  - /etc/passwd
  - /etc/shadow
  - /etc/sudoers
  - /root/.ssh/authorized_keys

# Use list in rule
- rule: Write to Sensitive Files
  condition: >
    write and container and
    fd.name in (sensitive_files)
  output: >
    ALERT: Sensitive file modified (file=%fd.name)
  priority: CRITICAL
```

---

## Falco Performance & Troubleshooting

### Check Falco Status

```bash
# On node
sudo systemctl status falco

# Check logs
sudo journalctl -u falco -n 50  # Last 50 lines
sudo journalctl -u falco -f     # Follow live

# In Kubernetes
k logs -f pod/falco-xyz -n falco
```

### Falco Not Alerting?

```bash
# 1. Check if running
k get daemonset -n falco

# 2. Check logs for errors
k logs -f pod/falco-xyz -n falco | grep -i error

# 3. Verify rules loaded
k exec pod/falco-xyz -n falco -- falco -i

# 4. Check if syscalls available
# On node:
sudo cat /proc/sys/kernel/ftrace_enabled
# Should be 1

# 5. Restart Falco
k delete pod -l app=falco -n falco
# DaemonSet will recreate
```

### High CPU/Memory Usage?

```bash
# Falco watching too many events, tune rules:
# Remove/disable low-value rules
# Add filters to reduce noise
# Example: Only alert on prod namespaces
- rule: My Rule
  condition: >
    ... and
    container.labels["namespace"] == "production"
```

---

## Integrating Falco with Other Systems

### Send Alerts to Syslog

```yaml
# /etc/falco/falco.yaml
syslog:
  enabled: true
  facility: LOG_USER
```

### Send Alerts to Webhook (Slack/Teams)

```bash
# Configure webhook output
cat <<EOF > /etc/falco/outputs.yaml
outputs:
  - name: webhook
    enabled: true
    webhook_url: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
    custom_headers: []
EOF

# Falco sends each alert to Slack in real-time
```

### Send to SIEM (Splunk)

```bash
# Forward syslog to Splunk
# Falco → syslog → Splunk
# Splunk indexes and correlates alerts
```

---

## Pre-Exam Falco Checklist

- ✅ Install Falco (daemonset or systemd)
- ✅ Verify it's running (check logs)
- ✅ Understand rule structure (condition, output, priority)
- ✅ Know built-in rule categories (process, file, network, privilege)
- ✅ Create custom rule for your use case
- ✅ Test rule (trigger the condition, verify alert)
- ✅ Know alert priorities (CRITICAL > WARNING > NOTICE)
- ✅ Know how to query Falco logs (k logs, journalctl, grep)

---

# PART 3: Trivy + Falco Together (Defense in Depth)

## Complete Security Pipeline

```
1. BUILD STAGE (Trivy)
   ├─ docker build -t myapp:v1 .
   └─ trivy image myapp:v1
      ├─ If CRITICAL: REJECT, don't push
      └─ If OK: Push to registry

2. DEPLOYMENT STAGE (SecurityContext + PSS)
   ├─ Pod Security Standards labels
   ├─ SecurityContext (non-root, no escalation, etc.)
   └─ NetworkPolicy (zero-trust)

3. RUNTIME STAGE (Falco)
   ├─ Falco daemonset monitoring all pods
   ├─ Detect: shell access, file tampering, suspicious connections
   └─ Alert: Send to security team
```

---

## Real Exam Scenario: Full Security Posture

**Question**: Implement end-to-end container security for production namespace.

```bash
# 1. Scan image with Trivy
trivy image myapp:v1 --severity CRITICAL
# Result: No CRITICAL CVEs, safe to deploy

# 2. Deploy with secure config
k apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
      - name: app
        image: myapp:v1
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
EOF

# 3. Enable PSS on namespace
k label namespace production pod-security.kubernetes.io/enforce=restricted

# 4. Verify Falco is running
k get daemonset -n falco

# 5. Test: Try to shell into pod
k exec -it deployment/myapp -n production -- bash
# Falco detects and alerts

# 6. Check Falco alert
k logs -f -l app=falco -n falco | grep "Shell spawned"
```

---

## Cheat Sheet: Trivy Commands

```bash
# Basic scan
trivy image nginx:1.25

# Only CRITICAL
trivy image --severity CRITICAL nginx:1.25

# JSON output
trivy image -f json -o results.json nginx:1.25

# Skip certain CVEs
trivy image --skip-check CVE-2023-1234 nginx:1.25

# Scan tarball
trivy image --input image.tar

# Scan directory
trivy rootfs /var/lib/docker

# List vulnerabilities
trivy image -f table nginx:1.25
```

---

## Cheat Sheet: Falco Commands

```bash
# Install
sudo apt-get install falco

# Start
sudo systemctl start falco

# Follow logs (on node)
sudo journalctl -u falco -f

# Follow logs (Kubernetes)
k logs -f -l app=falco -n falco

# List rules
sudo falco -i

# Test rule
sudo falco -r /etc/falco/rules.yaml

# Restart
sudo systemctl restart falco
```

---

## Pre-Exam Validation

```bash
# 1. Scan image quickly
time trivy image nginx:1.25
# Should take <30 sec

# 2. Interpret output
trivy image nginx:1.25 | grep CRITICAL
# Should identify vulnerability count instantly

# 3. Deploy Falco
time k apply -f falco-daemonset.yaml
# Should take <1 min

# 4. Verify Falco running
k get pods -n falco
# Should see daemonset pods on all nodes

# 5. Trigger alert
k exec -it deployment/test -- bash
# Check: k logs -f -l app=falco -n falco
# Should see alert within seconds

# If all working: You're ready 💪
```

