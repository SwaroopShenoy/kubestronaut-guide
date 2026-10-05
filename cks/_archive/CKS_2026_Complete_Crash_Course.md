# CKS 2026 Complete Crash Course
## Certified Kubernetes Security Specialist - All Domains Deep Dive

**Format**: Performance-based, 2 hours, 15-20 tasks, 67% pass  
**Domains**: 6 equally weighted (15% each)  
**Prerequisites**: Active CKA certificate

---

# Domain 1: Cluster Setup & Hardening (15%)

## 1.1 Network Policies (HIGHEST VALUE)

**Default state**: All pods can talk to all pods (INSECURE).

### Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  # No ingress rules = deny all
```

### Allow Specific Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - namespaceSelector:
        matchLabels:
          name: external
    ports:
    - protocol: TCP
      port: 8080
```

### Deny Egress to Untrusted Namespaces

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-safe-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  egress:
  # Allow to Kubernetes DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
  # Allow to internal services
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 443
  # Explicit: no access to other namespaces, internet
```

### Test NetworkPolicy

```bash
# Deploy test pods
k run test-pod --image=busybox -it -- sh

# From test pod, try to reach blocked service
k exec test-pod -- wget -O- http://backend-service:8080
# Should timeout if blocked

# If doesn't work, check:
k describe networkpolicy allow-from-frontend
k get networkpolicies
```

---

## 1.2 CIS Kubernetes Benchmark

<cite index="21-1">CIS benchmark review is section of cluster setup security.</cite>

**What it is**: Hardening guidelines from Center for Internet Security.

### Check CIS Compliance

```bash
# Install kube-bench (automated CIS checker)
# Direct download:
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.6.10/kube-bench_0.6.10_linux_x86_64.tar.gz | tar -xz
sudo mv kube-bench /usr/local/bin

# Run CIS checks
sudo kube-bench run --targets master,node,policies

# Output: PASS, FAIL, WARN, INFO
```

### Common CIS Failures & Fixes

**1.2.1: API server not enforcing RBAC**
```bash
# Check kube-apiserver manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep authorization-mode

# Should have: --authorization-mode=RBAC
# If missing, add it and restart
```

**1.4.1: Controller manager logging level**
```bash
# Check
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep v=

# Should be: --v=2 (not 4 or higher)
```

**4.1.1: Kubelet file permissions**
```bash
# On node
ls -l /etc/kubernetes/kubelet.conf
# Should be: -rw------- (600)

# Fix:
sudo chmod 600 /etc/kubernetes/kubelet.conf
```

**4.2.2: Kubelet --read-only-port**
```bash
# Check kubelet config
cat /var/lib/kubelet/config.yaml | grep readOnlyPort

# Should be: readOnlyPort: 0 (disabled)
# Edit and restart kubelet
```

---

## 1.3 TLS for Ingress

Ingress must use TLS (HTTPS), not plain HTTP.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-cert>
  tls.key: <base64-key>

---

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 443
```

### Create Self-Signed Cert for Testing

```bash
# Generate private key
openssl genrsa -out tls.key 2048

# Create cert
openssl req -new -x509 -key tls.key -out tls.crt -days 365 \
  -subj "/CN=api.example.com"

# Create secret
k create secret tls tls-secret --cert=tls.crt --key=tls.key

# Verify
k get secret tls-secret -o yaml
k describe secret tls-secret
```

---

## 1.4 Protect Node Metadata

Nodes can access cloud metadata (AWS, GCP, Azure). Restrict this.

### Disable AWS Metadata Access

```bash
# On node, restrict iptables
sudo iptables -I OUTPUT -p tcp -d 169.254.169.254 -j DROP

# Verify pods can't access
k run test --image=curlimages/curl -it -- curl 169.254.169.254
# Should timeout
```

### Kubelet Read-Only Port (Disable It)

```bash
# Kubelet exposes metrics on :10255 by default (INSECURE)
# Check kubelet config
cat /var/lib/kubelet/config.yaml | grep readOnlyPort

# Should be: readOnlyPort: 0

# Edit and restart
sudo systemctl restart kubelet
```

---

## 1.5 Verify Binary Checksums

Ensure kubelet, API server, etcd are official binaries.

```bash
# Download checksum file
curl -L https://dl.k8s.io/v1.35.0/bin/linux/amd64/SHA256SUMS -o checksums

# Verify kubelet
sha256sum -c SHA256SUMS 2>/dev/null | grep kubelet
# Output: ./kubelet: OK

# If mismatch: binary was tampered with, DO NOT USE
```

---

# Domain 2: Minimize Microservice Vulnerabilities (20%)

## 2.1 SecurityContext

Controls what a Pod/Container can do at OS level.

### Pod-Level SecurityContext

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000           # Non-root user ID
    runAsNonRoot: true        # Enforce non-root
    fsGroup: 2000             # Group ID for volumes
    seccompProfile:
      type: RuntimeDefault    # SELinux/AppArmor profile
  containers:
  - name: app
    image: myapp:v1
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL                  # Drop ALL Linux capabilities
        add:
        - NET_BIND_SERVICE     # Add back only needed
      readOnlyRootFilesystem: true
      runAsUser: 1000
```

### Key Fields

| Field | Value | Why |
|-------|-------|-----|
| `runAsNonRoot` | true | Prevents running as root |
| `allowPrivilegeEscalation` | false | Prevents sudo escape |
| `capabilities.drop` | ALL | Remove all Linux powers |
| `readOnlyRootFilesystem` | true | Can't write to disk |
| `runAsUser` | >=1000 | Non-root UID |

### Verify SecurityContext Applied

```bash
# Check running container
k exec <pod> -- id
# Output: uid=1000 (not uid=0)

k exec <pod> -- touch /test
# Should fail: Read-only file system
```

---

## 2.2 Pod Security Standards (PSS)

Replaces deprecated PodSecurityPolicy.

### Three Levels

**Restricted** (Most secure):
- No root
- No privileged mode
- No capability add
- Read-only root filesystem
- Drop ALL capabilities

**Baseline** (Minimal):
- Allows privilege escalation
- Allows some capabilities
- Can write to filesystem

**Unrestricted** (No enforcement):
- Anything goes (UNSAFE)

### Enforce PSS on Namespace

```bash
# Label namespace for Pod Security Standards
k label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Now any pod that violates restricted will be rejected
k run unsafe --image=nginx --as root
# Error: violates PodSecurityStandard "restricted"
```

### Pod Security Standards Manifest

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
  - ALL
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'MustRunAs'
    seLinuxOptions:
      level: "s0:c123,c456"
  readOnlyRootFilesystem: true
```

---

## 2.3 Secrets Management

Never hardcode secrets. Use proper storage.

### Create Secrets

```bash
# From literal
k create secret generic db-password --from-literal=password=supersecret

# From file
k create secret generic api-key --from-file=api.key

# From env file
k create secret generic creds --from-env-file=creds.env

# Generic (base64 encoded)
k create secret generic tls-secret --from-file=tls.crt --from-file=tls.key
```

### Use Secrets in Pods (SAFE)

```yaml
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-password
          key: password
    volumeMounts:
    - name: api-key
      mountPath: /etc/api-key
      readOnly: true
  volumes:
  - name: api-key
    secret:
      secretName: api-key
      defaultMode: 0400  # Read-only
```

### Encrypt Secrets at Rest

```bash
# Secrets are base64 encoded by default (NOT encrypted!)
k get secret db-password -o yaml
# Shows: password: c3VwZXJzZWNyZXQ=

# To encrypt, configure etcd encryption
# Add to kube-apiserver manifest:
# --encryption-provider-config=/etc/kubernetes/encryption.yaml

# Create encryption config
cat <<EOF > /etc/kubernetes/encryption.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}
EOF

# Restart API server
sudo systemctl restart kubelet
```

### Don't Do This

```yaml
# ❌ WRONG - Secret in environment variable (visible in ps/logs)
env:
- name: PASSWORD
  value: "supersecret"

# ❌ WRONG - Secret in ConfigMap (base64, not encrypted)
# Use Secret instead

# ❌ WRONG - Secret in container image
# FROM ubuntu
# RUN echo "password=secret" > /app/creds.txt

# ✅ RIGHT - Use Secret volume, mounted at runtime
```

---

## 2.4 RBAC (Recap from CKA, but Hardened)

Only grant minimum permissions.

```yaml
# Principle of least privilege
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
# Only read pods, NOT modify
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
# Only pods/logs, NOT exec
- apiGroups: [""]
  resources: ["pods/logs"]
  verbs: ["get"]
# Explicitly NOT:
# - create, delete, patch
# - pods/exec
# - pods/attach
# - secrets
```

### Dangerous Role (DON'T USE)

```yaml
# ❌ Overly permissive
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

---

# Domain 3: Supply Chain Security (20%)

## 3.1 Image Scanning with Trivy

Scan container images for vulnerabilities.

```bash
# Install Trivy
wget https://github.com/aquasecurity/trivy/releases/download/v0.46.0/trivy_0.46.0_Linux-64bit.tar.gz
tar xzf trivy_0.46.0_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin

# Scan local image
trivy image nginx:1.25
# Output: shows CVEs, severity (CRITICAL, HIGH, MEDIUM, LOW)

# Scan from registry
trivy image --remote docker.io/library/nginx:latest

# Check if CRITICAL CVEs exist
trivy image --severity CRITICAL nginx:1.25
```

### Scan in CI/CD (Before Push)

```bash
#!/bin/bash
# .github/workflows/build.yml
- name: Scan image with Trivy
  run: |
    trivy image myapp:${{ github.sha }}
    if [ $? -ne 0 ]; then
      echo "Image has vulnerabilities!"
      exit 1
    fi
```

---

## 3.2 Image Signing & Verification (Cosign)

Sign images so you know they're authentic.

### Sign an Image

```bash
# Generate signing key
cosign generate-key-pair

# Sign image (requires Docker login first)
cosign sign --key cosign.key docker.io/myrepo/myapp:v1

# Image is signed, pushed to registry
```

### Verify Signature

```bash
# Verify signature
cosign verify --key cosign.pub docker.io/myrepo/myapp:v1

# Output: Signature verified ✓

# Rejected if tampered
cosign verify --key cosign.pub docker.io/myrepo/myapp:v2-tampered
# Error: signature verification failed
```

---

## 3.3 Admission Controllers (Enforce Policy)

Control what gets deployed.

### Image Policy Admission

```yaml
# /etc/kubernetes/admission/admission-config.yaml
kind: ImagePolicyWebhook
kubeConfigFile: /etc/kubernetes/admission/kubeconfig.yaml
allowTTL: 50
denyTTL: 50
retryBackoff: 5
defaultAllow: true  # Allow if webhook fails
```

### Enforce Signed Images Only

```yaml
# Gatekeeper policy (OPA)
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: signed-images-only
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    labels: ["image-signed"]
```

---

## 3.4 Software Bill of Materials (SBOM)

Document what's in your images.

```bash
# Generate SBOM with Trivy
trivy image --format cyclonedx -o sbom.json nginx:1.25

# Or with syft
syft docker.io/library/nginx:1.25 -o json > sbom.json

# Attach SBOM to image
cosign attach sbom --sbom sbom.json docker.io/myrepo/myapp:v1

# Retrieve SBOM
cosign download sbom docker.io/myrepo/myapp:v1
```

---

# Domain 4: Compliance, Auditing & Logging (15%)

## 4.1 Audit Logging

Track who did what in cluster.

### Enable Audit Logging

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log all requests at Metadata level
- level: Metadata
  omitStages:
  - RequestReceived

# Log Pod exec commands
- level: RequestResponse
  omitStages:
  - RequestReceived
  verbs: ["create"]
  resources:
  - group: ""
    resources: ["pods/exec", "pods/attach"]

# Log Secret access
- level: Metadata
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["secrets"]

# Catch-all
- level: Metadata
```

### Configure API Server

```bash
# Edit kube-apiserver manifest
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml

# Add flags:
# --audit-policy-file=/etc/kubernetes/audit-policy.yaml
# --audit-log-maxage=30
# --audit-log-maxbackup=10
# --audit-log-maxsize=100

# Restart kubelet (auto-restarts pod)
sudo systemctl restart kubelet
```

### Query Audit Logs

```bash
# Logs written to /var/log/kubernetes/audit/audit.log

# Search for secret access
tail -f /var/log/kubernetes/audit/audit.log | grep "secrets" | jq .

# Search for exec commands
cat /var/log/kubernetes/audit/audit.log | jq 'select(.verb=="create" and .objectRef.resource=="pods/exec")'

# Parse with jq
cat /var/log/kubernetes/audit/audit.log | jq '{user: .user.username, action: .verb, resource: .objectRef.resource, time: .requestReceivedTimestamp}'
```

---

## 4.2 Falco (Runtime Security Monitoring)

Detect suspicious container behavior in real-time.

```bash
# Install Falco
curl -s https://falco.org/repo/falcosecurity-3672BA8F.asc | apt-key add -
echo "deb https://download.falco.org/packages/deb stable main" | tee /etc/apt/sources.list.d/falcosecurity.list
sudo apt-get update
sudo apt-get install -y falco

# Start Falco
sudo systemctl start falco

# View alerts
sudo journalctl -u falco -f
# Output: Unauthorized process, Unauthorized file write, etc.
```

### Falco Rules (Detect Threats)

```yaml
# /etc/falco/rules.yaml
- rule: Unauthorized Process
  desc: Detects unauthorized process execution
  condition: >
    spawned_process and container and
    proc.name not in (bash, nginx, python)
  output: >
    Process started (user=%user.name process=%proc.name container=%container.name)
  priority: WARNING

- rule: Write to System File
  desc: Write to sensitive files
  condition: >
    write and container and
    fd.name glob /etc/* and not fd.name glob /etc/nginx/*
  output: >
    File written (user=%user.name file=%fd.name)
  priority: CRITICAL
```

---

## 4.3 Pod Security Policies (Deprecated, but Still Tested)

```yaml
# Restrict pods
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
  - ALL
  runAsUser:
    rule: MustRunAsNonRoot
  seLinux:
    rule: MustRunAs
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'secret'
  hostNetwork: false
  hostIPC: false
  hostPID: false
```

### Bind PSP to ServiceAccount

```bash
k create clusterrole psp-restricted --verb=use --resource=podsecuritypolicies --resource-name=restricted

k create clusterrolebinding psp-restricted-sa \
  --clusterrole=psp-restricted \
  --serviceaccount=default:default
```

---

# Domain 5: System Hardening (15%)

## 5.1 AppArmor & SELinux (Container Confinement)

Restrict what containers can do at OS level.

### AppArmor

```bash
# Check if AppArmor is enabled
systemctl status apparmor

# View loaded profiles
sudo aa-status

# Create AppArmor profile
cat <<EOF | sudo tee /etc/apparmor.d/k8s-profile
#include <tunables/global>

profile k8s-profile flags=(attach_disconnected) {
  #include <abstractions/base>
  
  # Allow read from /app
  /app/** r,
  
  # Allow write only to /tmp
  /tmp/** rw,
  
  # Deny everything else
  deny /etc/** rwk,
  deny /proc/** rwk,
}
EOF

# Load profile
sudo apparmor_parser /etc/apparmor.d/k8s-profile

# Use in Pod
apiVersion: v1
kind: Pod
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/k8s-profile
spec:
  containers:
  - name: app
    image: myapp
```

### SELinux

```bash
# Check if enabled
getenforce

# Set to enforcing (strict)
sudo setenforce Enforcing

# Check pod's SELinux context
k describe pod <pod> | grep seLinuxOptions
```

---

## 5.2 Node Access Control

Restrict who can SSH/access nodes.

```bash
# Disable SSH root login
sudo vi /etc/ssh/sshd_config
# PermitRootLogin no

# Use SSH keys only (no password)
# PasswordAuthentication no

# Disable SSH entirely (if not needed)
sudo systemctl stop ssh
sudo systemctl disable ssh

# Restrict kubelet read-only port
# kubelet --read-only-port=0

# Check open ports on node
sudo netstat -tlnp
# Should see: kubelet (10250), metrics (10250), kube-proxy
# Should NOT see: 22 (SSH), 10255 (read-only kubelet)
```

---

## 5.3 Restrict Kernel Modules

Prevent loading malicious kernel modules.

```bash
# Disable kernel module loading
echo "install floppy /bin/true" | sudo tee /etc/modprobe.d/floppy.conf
echo "install dccp /bin/true" | sudo tee /etc/modprobe.d/dccp.conf

# Verify
lsmod | grep floppy  # Should be empty
```

---

## 5.4 Restrict Kubelet API

```bash
# Kubelet serves API on :10250 (HTTPS)
# Restrict access via firewall

sudo ufw allow from 10.0.0.0/8 to any port 10250
sudo ufw deny from any to any port 10250

# Or iptables
sudo iptables -A INPUT -p tcp --dport 10250 -s 10.0.0.0/8 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 10250 -j DROP
```

---

# Domain 6: Runtime Security (15%)

## 6.1 Falco Deep Dive

Already covered in Compliance section. Key points:

```bash
# Install in container
k create deployment falco --image=falcosecurity/falco

# Mount /var/run/docker.sock for container monitoring
volumeMounts:
- name: docker-sock
  mountPath: /var/run/docker.sock

# Output suspicious activity to stdout
k logs -f deployment/falco

# Common alerts:
# - Unauthorized process execution
# - Shell spawned in container
# - Write to system files
# - Network connections to external IPs
```

---

## 6.2 Container Runtime Security (containerd/CRI-O)

Ensure runtime itself is secure.

```bash
# Check container runtime
crictl version

# Verify runtime is not running as root (best practice)
ps aux | grep containerd
# Should be in docker group, not root

# Restrict container runtime access
sudo chmod 660 /run/containerd/containerd.sock

# Audit runtime operations
journalctl -u containerd -f
```

---

## 6.3 Admission Controllers for Runtime Security

```yaml
# ValidatingAdmissionPolicy to enforce runtime constraints
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingAdmissionPolicy
metadata:
  name: runtime-security
spec:
  failurePolicy: Fail
  validationActions:
  - audit
  matchResources:
    resourceRules:
    - apiGroups: [""]
      resources: ["pods"]
  rules:
  - expression: "object.spec.securityContext.runAsNonRoot == true"
    message: "Pods must run as non-root"
```

---

# Domain Expansion: Critical Missing Topics

## RuntimeClass (Container Isolation)

Different container runtimes for different security needs.

```yaml
# Use gVisor for untrusted workloads (sandboxed)
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: gvisor

---

# Use Kata for maximum isolation (lightweight VM)
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata

---

# Pod uses gVisor
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed-pod
spec:
  runtimeClassName: gvisor
  containers:
  - name: app
    image: untrusted:latest
```

### Why It Matters

- **Default (runc)**: Fast, shared kernel (less isolation)
- **gVisor**: Sandbox, intercepts syscalls, slower but secure
- **Kata**: Lightweight VM, highest isolation, slowest

**Use**: Untrusted user code → gVisor. Multi-tenant cluster → Kata.

---

## Seccomp (Syscall Filtering)

Block dangerous system calls at kernel level.

### Default Seccomp Profile

```bash
# Kubernetes has default profile
# Allow most syscalls, block dangerous ones

# Check what's blocked:
# - ptrace (process tracing)
# - perfopen (performance monitoring)
# - chroot (root change)
# - keyctl (key management)
```

### Custom Seccomp Profile

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 1,
  "archMap": [
    {
      "architecture": "SCMP_ARCH_X86_64",
      "subArchitectures": []
    }
  ],
  "syscalls": [
    {
      "names": ["read", "write", "open", "close"],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": ["ptrace"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
```

### Apply Seccomp to Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: my-profile.json  # Must be in /var/lib/kubelet/seccomp/
  containers:
  - name: app
    image: myapp
```

### Verify Syscall Blocking

```bash
# Inside container, try ptrace syscall
k exec <pod> -- strace -e trace=ptrace echo
# Should fail: ptrace is blocked
```

---

## OPA/Gatekeeper (Policy Engine)

Define custom security policies beyond RBAC.

### Install Gatekeeper

```bash
k apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml

# Verify
k get deployment -n gatekeeper-system
```

### Constraint Template (Require Resource Limits)

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlimits
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLimits
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlimits
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.limits
        msg := sprintf("Container %v must have resource limits", [container.name])
      }
```

### Constraint (Enforce the Template)

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLimits
metadata:
  name: require-limits
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    limits: ["cpu", "memory"]
```

**Result**: Any Pod without limits is REJECTED at admission time.

---

## etcd Hardening

etcd stores all cluster data. Secure it!

### Restrict etcd Access

```bash
# etcd listens on :2379 (encrypted, authenticated)
# and :2380 (peer communication)

# Restrict network access
sudo ufw allow from 10.0.0.0/8 to any port 2379
sudo ufw deny from any to any port 2379

# Verify only control plane nodes access etcd
sudo iptables -A INPUT -p tcp --dport 2379 -s 10.0.0.5 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 2379 -j DROP
```

### Backup etcd Regularly

```bash
# Snapshot backup
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Automate with cron
# 0 2 * * * ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db
```

### etcd Encryption (At Rest)

Already covered in Secrets section—configure API server encryption provider.

---

## RBAC for System Components

Harden permissions of control plane components.

### kubeadm Auto-Generates RBAC

```bash
# Check default roles
k get clusterroles | grep system:

# Example: system:kubelets
k describe clusterrole system:kubelets
# Allows kubelet to read/write Node, Pod resources
```

### Least Privilege for Kubelet

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kubelet-minimal
rules:
# Only what kubelet NEEDS
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["list"]
```

---

## Authorization Modes

Configure API server to enforce auth.

### Check Current Mode

```bash
# In kube-apiserver manifest
grep authorization-mode /etc/kubernetes/manifests/kube-apiserver.yaml
# Output: --authorization-mode=Node,RBAC
```

### Modes Explained

| Mode | Purpose |
|------|---------|
| **Node** | Kubelet can only access its own Pod/Node |
| **RBAC** | Role-based access control (our main tool) |
| **Webhook** | External authorization service |
| **AlwaysAllow** | ❌ NEVER USE (insecure) |
| **AlwaysDeny** | ❌ NEVER USE (nothing works) |

---

## mTLS for Inter-Service Communication

Encrypt traffic BETWEEN services.

### Without mTLS
```
Pod A ──(plain HTTP)──> Pod B
       ↑
       Anyone can intercept
```

### With Istio mTLS
```bash
# Install Istio
curl -L https://istio.io/downloadIstio | sh
./istio-1.x/bin/istioctl install

# Enable mTLS for namespace
k label namespace prod istio-injection=enabled

# Deploy policy (sidecar auto-injects)
# All traffic now encrypted + authenticated
```

### Verify mTLS

```bash
# Check sidecar injection
k get pods <pod> -o yaml | grep istio-proxy

# View TLS status
istioctl authn tls-check

# All should show: MUTUAL
```

---

## Audit Log Severity Levels

Not all audit events are equal.

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# CRITICAL: Secret access
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["secrets"]

# HIGH: Privilege escalation via RBAC
- level: RequestResponse
  verbs: ["create", "update", "patch"]
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["clusterrolebindings", "clusterroles"]

# MEDIUM: Configuration changes
- level: Metadata
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: "apps"
    resources: ["deployments", "statefulsets"]

# LOW: Normal operations
- level: RequestReceivedTimestamp
  omitStages:
  - RequestReceived
```

---

## Disable Insecure Kubelet Flags

```bash
# Common insecure configurations (all should be disabled)

# ❌ --allow-privileged=true
#    √ Fix: --allow-privileged=false

# ❌ --anonymous-auth=true
#    √ Fix: --anonymous-auth=false

# ❌ --authorization-mode=AlwaysAllow
#    √ Fix: --authorization-mode=Webhook

# ❌ --read-only-port=10255
#    √ Fix: --read-only-port=0

# Check current settings
cat /var/lib/kubelet/config.yaml | grep "anonymous\|allow\|read-only\|authorization"
```

---

## Encrypted ConfigMaps (Not Just Secrets)

ConfigMaps are NOT encrypted by default!

```bash
# Check: ConfigMaps are base64, NOT encrypted
k get configmap app-config -o yaml
# Shows: key: dXNlcm5hbWU=bGVhZA==
# Base64 decodable!

# If you need to encrypt ConfigMaps:
# Option 1: Use Secrets instead
# Option 2: Apply etcd encryption to all resources
```

### Encrypt All Resources

```yaml
# /etc/kubernetes/encryption.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  - configmaps  # Also encrypt ConfigMaps!
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-32-byte-key>
  - identity: {}
```

---

## Workload Identity (External Auth)

Allow Pods to authenticate to external services (cloud APIs, etc).

### AWS IRSA (IAM Roles for Service Accounts)

```bash
# Create OIDC provider (AWS)
eksctl utils associate-iam-oidc-provider --cluster=my-cluster

# Create IAM role
aws iam create-role --role-name my-pod-role \
  --assume-role-policy-document file://trust-policy.json

# Attach policies
aws iam attach-role-policy --role-name my-pod-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Annotate ServiceAccount
k annotate serviceaccount my-sa \
  eks.amazonaws.com/role-arn=arn:aws:iam::123456789:role/my-pod-role

# Pod automatically assumes role
# Can access AWS APIs without hardcoding credentials
```

---

## Image Pull Secrets (Private Registries)

```bash
# Create secret with registry credentials
k create secret docker-registry regcred \
  --docker-server=docker.io \
  --docker-username=myuser \
  --docker-password=mypass

# Use in Pod
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: docker.io/myuser/private-app:v1
```

---

## Network Policies for Egress to External APIs

Common pattern: Pod needs to reach external API.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
spec:
  podSelector:
    matchLabels:
      app: processor
  policyTypes:
  - Egress
  egress:
  # Allow DNS to resolve
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
  # Allow HTTPS to external API
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24  # External API IP range
    ports:
    - protocol: TCP
      port: 443
```



## Time Management

- **2 hours** for ~15-20 tasks
- ~6-7 minutes per task
- **Always flag hard ones** and come back

## Common Question Types

**1. Fix insecure Pod**
```bash
# Modify Pod SecurityContext
# Usually: remove root, drop capabilities, make read-only
k get pod <pod> -o yaml > pod.yaml
# Edit: add securityContext
k apply -f pod.yaml
```

**2. Create NetworkPolicy**
```bash
# Template: deny all, then allow specific
k apply -f networkpolicy.yaml
# Test: k exec <pod> -- curl <blocked-service>
```

**3. Scan image for vulnerabilities**
```bash
trivy image myapp:latest
# Fix: update base image, rebuild
```

**4. Enable audit logging**
```bash
# Edit kube-apiserver manifest
# Add: --audit-policy-file, --audit-log-*
# Restart kubelet
```

**5. Configure encryption**
```bash
# Create encryption-config.yaml
# Add to kube-apiserver
# Restart
```

---

# Quick Reference

## Must-Know Commands

```bash
# Image scanning
trivy image <image>
cosign sign --key key.key <image>
cosign verify --key key.pub <image>

# NetworkPolicy
k apply -f networkpolicy.yaml
k describe networkpolicy <name>

# SecurityContext
k set securitycontext pod <pod> --run-as-user=1000
k describe pod <pod> | grep securityContext

# Audit logs
tail -f /var/log/kubernetes/audit/audit.log
cat /var/log/kubernetes/audit/audit.log | jq .

# Falco
sudo systemctl start falco
sudo journalctl -u falco -f

# CIS Benchmark
kube-bench run --targets master,node

# Pod Security Standards
k label namespace prod pod-security.kubernetes.io/enforce=restricted
```

## Gotchas

❌ **Running as root**: Most pods still do this (FIX: add SecurityContext)
❌ **NetworkPolicy missing DNS**: Pods can't resolve external names (ADD: UDP 53)
❌ **Secrets not encrypted**: Base64 is NOT encryption (CONFIGURE: etcd encryption)
❌ **Privileged containers**: Always running (FIX: allowPrivilegeEscalation: false)
❌ **No audit logging**: Can't trace who did what (ENABLE: audit-policy-file)

---

---

# Real Exam Scenarios (Actual Questions)

## Scenario 1: Secure Insecure Deployment

**Question**: A deployment is running nginx with security vulnerabilities. Secure it.

```bash
# Initial state
k get deployment nginx -o yaml
# Shows: runAsUser: 0 (ROOT!), no SecurityContext, no resource limits

# Fix:
k set resources deployment nginx --limits=cpu=200m,memory=256Mi
k patch deployment nginx -p '{"spec":{"template":{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":101,"fsGroup":101},"containers":[{"name":"nginx","securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"readOnlyRootFilesystem":true}}]}}}}'

# OR safer: patch via YAML
k get deployment nginx -o yaml > nginx-patch.yaml
# Edit: add securityContext at Pod level
k apply -f nginx-patch.yaml

# Verify
k get deployment nginx -o yaml | grep -A 10 securityContext
```

## Scenario 2: Create Zero-Trust NetworkPolicy

**Question**: Default deny all traffic. Allow only frontend→backend on 8080.

```yaml
# 1. Deny all ingress (cluster-wide)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---

# 2. Allow frontend to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080

---

# 3. Allow backend to external database (egress)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
  # Allow database
  - to:
    - ipBlock:
        cidr: 10.0.0.50/32  # Database IP
    ports:
    - protocol: TCP
      port: 5432
```

**Test**:
```bash
# Frontend should reach backend
k exec deployment/frontend -- curl http://backend-service:8080
# Should succeed

# Untrusted pod should NOT reach backend
k run untrusted --image=busybox -it -- wget http://backend-service:8080
# Should timeout

# Backend should reach database
k exec deployment/backend -- psql -h 10.0.0.50 -U admin
# Should connect

# Backend should NOT reach external internet
k exec deployment/backend -- curl https://example.com
# Should timeout (if not in allow-egress rule)
```

## Scenario 3: Enforce Image Scanning Before Deployment

**Question**: Only allow images that have been scanned with Trivy and have no CRITICAL CVEs.

```yaml
# Using OPA/Gatekeeper
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sscannedimagess
spec:
  crd:
    spec:
      names:
        kind: K8sScannedImages
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sscannedimages
      
      violation[{"msg": msg}] {
        image := input.review.object.spec.containers[_].image
        # Check if image has annotation indicating it was scanned
        not input.review.object.metadata.annotations["image-scanned"] == "true"
        msg := sprintf("Image %v must be scanned before deployment", [image])
      }

---

# Enforce constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sScannedImages
metadata:
  name: enforce-scanned-images
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]

---

# Deployment with scanned image
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    image-scanned: "true"  # Indicate image was scanned
spec:
  template:
    metadata:
      annotations:
        image-scanned: "true"
    spec:
      containers:
      - name: app
        image: myrepo/myapp:v1  # Only deploy if scanned
```

**Workflow**:
```bash
# 1. Build image
docker build -t myrepo/myapp:v1 .

# 2. Scan before push
trivy image myapp:v1
# Check: no CRITICAL CVEs

# 3. If passes, push
docker push myrepo/myapp:v1

# 4. Deploy (with annotation proving scanned)
k apply -f deployment.yaml
# Gatekeeper checks annotation, allows it

# 5. If not scanned:
# k apply -f deployment-no-scan.yaml
# Rejected: "Image must be scanned before deployment"
```

## Scenario 4: Enable Audit Logging & Find Suspicious Activity

**Question**: Configure audit logging. Find who created a secret.

```bash
# 1. Configure audit policy
cat <<EOF > /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log all Secret operations
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["secrets"]
  
# Catch-all
- level: Metadata
EOF

# 2. Add to kube-apiserver
# Edit /etc/kubernetes/manifests/kube-apiserver.yaml
# Add flags:
# --audit-policy-file=/etc/kubernetes/audit-policy.yaml
# --audit-log-maxage=7
# --audit-log-maxbackup=10
# --audit-log-maxsize=100
# Restart kubelet

# 3. Query audit logs
tail -f /var/log/kubernetes/audit/audit.log | \
  jq 'select(.objectRef.resource=="secrets" and .verb=="create")'

# Output:
{
  "level": "RequestResponse",
  "timestamp": "2026-07-28T10:15:30.123456Z",
  "user": {
    "username": "alice",
    "uid": "1234"
  },
  "verb": "create",
  "objectRef": {
    "resource": "secrets",
    "name": "db-password"
  },
  "requestObject": {...secret content...}
}

# Answer: User 'alice' created secret 'db-password' at 10:15:30
```

## Scenario 5: Runtime Security with Falco

**Question**: Detect and alert on unauthorized process execution in containers.

```bash
# 1. Deploy Falco
k create -f https://raw.githubusercontent.com/falcosecurity/falco/master/deploy/kubernetes/falco-daemonset.yaml

# 2. Configure Falco rules
cat <<EOF > /etc/falco/rules.d/custom-rules.yaml
- rule: Unauthorized Shell in Container
  desc: Shell spawned in production container
  condition: >
    spawned_process and container and 
    container.labels["environment"] == "production" and
    proc.name in (bash, sh, /bin/bash, /bin/sh)
  output: >
    ALERT: Shell spawned in production (container=%container.name user=%user.name process=%proc.name)
  priority: WARNING

- rule: Write to Sensitive File
  desc: Attempt to write to /etc
  condition: >
    write and container and
    fd.name glob /etc/*
  output: >
    ALERT: File write to /etc (container=%container.name file=%fd.name user=%user.name)
  priority: CRITICAL
EOF

# 3. Monitor alerts
k logs -f -l app=falco -n falco | grep ALERT

# Example output:
# ALERT: Shell spawned in production (container=api-pod user=root process=/bin/sh)
# ALERT: File write to /etc (container=api-pod file=/etc/passwd user=root)
```

---

# Common Mistakes (Learn to Avoid)

| Mistake | Impact | Fix |
|---------|--------|-----|
| Pod running as root | HIGH - Can escape container | Add `runAsUser: 1000` |
| No resource limits | MEDIUM - Can OOM node | Add `resources.limits` |
| NetworkPolicy missing DNS | HIGH - Pods can't reach external APIs | Add egress UDP port 53 |
| Secrets not encrypted | HIGH - Readable in etcd | Configure `encryption.yaml` |
| ServiceAccount with wildcard RBAC | CRITICAL - Pod can do anything | Use least-privilege roles |
| No audit logging | MEDIUM - Can't trace breaches | Enable audit-policy-file |
| Kubelet read-only port enabled | HIGH - Exposes metrics | Set `readOnlyPort: 0` |
| Images not scanned | MEDIUM - Deploy with CVEs | Use Trivy + admission policy |
| Falco not running | MEDIUM - No runtime visibility | Deploy Falco daemonset |
| mTLS disabled | MEDIUM - Traffic not encrypted | Enable Istio/mTLS |

---

# Pre-Exam Checklist

## Knowledge
- ✅ CKA + CKAD fundamentals (pods, deployments, services, RBAC)
- ✅ All 6 CKS domains deeply
- ✅ SecurityContext + Pod Security Standards
- ✅ NetworkPolicy (ingress + egress + DNS)
- ✅ Image scanning (Trivy) + signing (Cosign)
- ✅ Audit logging + jq for parsing
- ✅ AppArmor/SELinux basics
- ✅ OPA/Gatekeeper syntax
- ✅ Falco rules + runtime security
- ✅ etcd encryption + backup
- ✅ Kubelet hardening flags
- ✅ RBAC for system components
- ✅ TLS for all APIs

## Hands-On Practice
- ✅ Create Pod with SecurityContext in <2 min
- ✅ Create NetworkPolicy with egress rules in <3 min
- ✅ Scan image with Trivy + interpret output
- ✅ Enable audit logging + query logs with jq
- ✅ Deploy Falco + see real alerts
- ✅ Run kube-bench + fix CIS failures
- ✅ Configure pod-security.kubernetes.io labels
- ✅ Patch deployment to remove root
- ✅ Create OPA constraint + watch rejection
- ✅ Set up encryption-config for etcd
- ✅ Configure kubelet with secure flags

## Tools Installed
- ✅ trivy (image scanning)
- ✅ cosign (image signing)
- ✅ kube-bench (CIS auditing)
- ✅ falco (runtime security)
- ✅ jq (JSON parsing)
- ✅ openssl (certificates)
- ✅ crictl (container CLI)

## Exam Mindset
- ✅ Read FULL question before starting
- ✅ Flag complex questions, do easy ones first
- ✅ Always verify SecurityContext applied: `k exec <pod> -- id`
- ✅ Test NetworkPolicy: `k exec <pod> -- curl <service>`
- ✅ When in doubt: check logs + describe
- ✅ Time remaining? Verify everything works

---

# Last Minute Study Tips

**If you have 1 week**:
- Day 1-2: SecurityContext + Pod Security Standards (practice 5x)
- Day 3: NetworkPolicy (including egress + DNS gotchas)
- Day 4: Image scanning + OPA basics
- Day 5: Audit logging + Falco
- Day 6: CIS hardening + etcd encryption
- Day 7: Take killer.sh mock, review weak areas

**If you have 2 weeks**:
- Week 1: Learn each domain in detail
- Week 2: Practice labs + killer.sh x2

**If you have 4 weeks**:
- Week 1-2: Deep study + hands-on labs
- Week 3: Killer.sh + gap analysis
- Week 4: Final killer.sh + confidence building

---

# What's NOT on CKS

❌ Cloud-specific security (AWS IAM, GCP IAM) - K8s-native only  
❌ Machine learning/AI security  
❌ Compliance frameworks (HIPAA, PCI) - general concepts only  
❌ Dockerfile security (that's dev, not cluster admin)  
❌ Database security (outside K8s scope)  
❌ Physical security  

---

# Final Validation

Before exam day:

```bash
# 1. Verify you can create secure workload quickly
time k run test --image=nginx -o yaml | \
  sed 's/name: test/name: test\n      securityContext:\n        runAsNonRoot: true\n        readOnlyRootFilesystem: true\n        allowPrivilegeEscalation: false\n        capabilities:\n          drop:\n          - ALL/' | \
  k apply -f -
# Should take <3 minutes

# 2. Verify NetworkPolicy speed
cat <<EOF | time k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-policy
spec:
  podSelector:
    matchLabels:
      app: test
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
EOF
# Should take <2 minutes

# 3. Verify image scanning
trivy image nginx:latest
# Should understand severity output instantly

# If all take less time than estimated → you're ready 💪
```

---

**You have everything you need. CKS is achievable. Go crush it!** 🚀



