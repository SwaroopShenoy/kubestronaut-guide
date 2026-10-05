# KCSA Crash Course 2025
## Kubernetes & Cloud Native Security Associate - Your Security Sprint

**Target Audience**: Mid-CKA level DevOps engineers taking KCNA + KCSA same day  
**Time Required**: 2-3 days focused study  
**Goal**: Security fundamentals mastery for 90%+ score

---

## 🎯 Your Gap Analysis

**What you already know from KCNA prep:**
- ✅ K8s architecture & components
- ✅ RBAC basics
- ✅ NetworkPolicy concepts  
- ✅ Pod security fundamentals
- ✅ Basic security context

**What's NEW in KCSA (your focus areas):**
- 🔥 **4Cs Security Model** (deep dive)
- 🔥 **Threat modeling** & attack vectors
- 🔥 **Compliance frameworks** (CIS, PCI-DSS, SOC2)
- 🔥 **Supply chain security** (SLSA, SBOM, Sigstore)
- 🔥 **Security tools** (Falco, OPA, Trivy, Cosign)
- 🔥 **Admission controllers** & policy enforcement
- 🔥 **Secrets management** (etcd encryption, external secrets)
- 🔥 **Security scanning** & vulnerability management
- 🔥 **Audit logging** & forensics

**Difficulty assessment**: KCSA > KCNA (harder than KCNA, easier than CKS)
- Described as "conceptual CKS"
- More security depth than KCNA
- Trickier questions, requires understanding not just memorization

---

## 📋 Exam Overview

**Format**: Multiple Choice (60 questions)  
**Duration**: 90 minutes (1.5 min per question)  
**Passing Score**: 75% (aim for 85%+)  
**Cost**: $250 (includes 1 free retake)  
**Validity**: 3 years  
**Environment**: Online, remotely proctored (PSI platform)

**Pro tip**: Join 30 mins early for PSI checks. Questions can be tricky - mark for review and look for hints in later questions.

---

## 🎯 Exam Domains & Approximate Weights

*Note: CNCF doesn't publish exact percentages for KCSA, but based on curriculum:*

| Domain | Est. Weight | Your Priority |
|--------|-------------|---------------|
| **Overview of Cloud Native Security** | ~25% | HIGH - 4Cs, threat modeling |
| **Kubernetes Cluster Component Security** | ~25% | MEDIUM - You know K8s, add security lens |
| **Kubernetes Security Fundamentals** | ~20% | MEDIUM - Deepen RBAC, PSS, NetworkPolicy |
| **Kubernetes Threat Model** | ~15% | HIGH - Attack vectors, STRIDE |
| **Platform Security** | ~10% | HIGH - Supply chain, image security |
| **Compliance and Security Frameworks** | ~5% | HIGH - CIS benchmarks, regulations |

---

## 📚 Domain 1: Overview of Cloud Native Security (~25%)

### 1.1 The 4Cs of Cloud Native Security

**CRITICAL CONCEPT - Memorize this layered model:**

```
Code
  ↓
Container  
  ↓
Cluster
  ↓
Cloud
```

**Each layer builds on the security of layers below it.**

#### **Cloud (Foundation)**
- Infrastructure security (VPC, firewalls, IAM)
- Physical security of data centers
- Network isolation
- Compliance certifications (SOC2, ISO 27001)

**Shared Responsibility Model**:
- Cloud Provider: Physical infrastructure, hypervisor, network
- You: OS, K8s cluster, containers, application code

#### **Cluster (Kubernetes)**
- Control plane security (API server auth/authz)
- Node security (OS hardening, minimal attack surface)
- Network policies (Pod-to-Pod communication)
- RBAC (least privilege)
- Secrets management
- etcd encryption
- Audit logging

**Key Controls**:
- TLS everywhere
- Strong authentication (OIDC, X.509 certs)
- Authorization (RBAC, not ABAC)
- Admission controllers (validating, mutating)
- Pod Security Standards

#### **Container (Image & Runtime)**
- Image security (scan for vulnerabilities)
- Image signing & verification (Cosign, Notary)
- Minimal base images (distroless, scratch)
- Non-root user
- Read-only root filesystem
- Drop capabilities
- Runtime security (Falco for anomaly detection)

**Best Practices**:
- Use trusted registries (private, signed images)
- Scan images in CI/CD pipeline
- Regular image updates
- Implement image pull policies
- Use SecurityContext to restrict containers

#### **Code (Application)**
- Secure coding practices (OWASP Top 10)
- Dependency scanning (SCA - Software Composition Analysis)
- Static Application Security Testing (SAST)
- Dynamic Application Security Testing (DAST)
- Secrets in code = ❌ (use external secret managers)
- Input validation, output encoding
- Least privilege for app

**Tools**:
- SAST: SonarQube, Checkmarx
- Dependency scanning: Snyk, Dependabot
- Secret scanning: git-secrets, TruffleHog

### 1.2 Threat Modeling

**STRIDE Framework** (Microsoft model):

| Threat | Description | Example |
|--------|-------------|---------|
| **S**poofing | Pretending to be something/someone else | Fake service accounts, token theft |
| **T**ampering | Modifying data or code | Container escape, image tampering |
| **R**epudiation | Denying actions | No audit logs, anonymous access |
| **I**nformation Disclosure | Exposing information | Secret leakage, exposed etcd |
| **D**enial of Service | Making system unavailable | Resource exhaustion, API flooding |
| **E**levation of Privilege | Gaining unauthorized access | Privilege escalation, RBAC bypass |

**Applying STRIDE to Kubernetes**:
- **Spoofing**: ServiceAccount token theft, compromised credentials
- **Tampering**: Malicious image injection, manifest modification
- **Repudiation**: Disabled audit logging, no monitoring
- **Information Disclosure**: Exposed Secrets, unencrypted etcd
- **DoS**: Resource bombs, API server overload
- **Elevation**: Container escape, kubelet compromise

### 1.3 Attack Vectors in Cloud Native

**Common Attack Scenarios**:

**Initial Access**:
- Exposed API server (no authentication)
- Compromised credentials
- Vulnerable application
- Supply chain attack (malicious image)

**Execution**:
- Malicious container deployed
- Exec into running container
- Compromised CI/CD pipeline

**Persistence**:
- Backdoor container
- Modified RBAC (elevated permissions)
- Malicious admission webhook
- Compromised node

**Privilege Escalation**:
- Container escape (kernel exploit)
- Overly permissive Pod (privileged, hostPath, hostNetwork)
- RBAC misconfiguration
- Insecure kubelet API

**Defense Evasion**:
- Disable audit logging
- Delete evidence (logs, pods)
- Living off the land (use existing binaries)

**Credential Access**:
- ServiceAccount token theft (`/var/run/secrets/kubernetes.io/serviceaccount/token`)
- etcd access (unencrypted Secrets)
- Cloud provider metadata service (IMDSv1)

**Discovery**:
- `kubectl auth can-i --list`
- Service discovery (DNS enumeration)
- API server probing

**Lateral Movement**:
- Pod-to-Pod without NetworkPolicy
- Compromised service to another namespace
- Node access to other nodes

**Impact**:
- Data exfiltration
- Cryptomining
- Ransomware
- Resource exhaustion (DoS)

**MITRE ATT&CK for Containers**: Framework mapping tactics to techniques

### 1.4 Defense in Depth

**Layered Security Strategy**:

1. **Prevention**: Stop threats before they happen
   - Image scanning, admission control, RBAC

2. **Detection**: Identify threats in real-time
   - Runtime security (Falco), audit logs, monitoring

3. **Response**: React to incidents
   - Automated remediation, incident response plan

4. **Recovery**: Restore after incident
   - Backups, disaster recovery, forensics

**Zero Trust Principles**:
- Never trust, always verify
- Least privilege access
- Assume breach (defense in depth)
- Verify explicitly (strong authentication)

---

## 🔧 Domain 2: Kubernetes Cluster Component Security (~25%)

### 2.1 Control Plane Security

#### **API Server**
- **Authentication**: Who are you?
  - X.509 client certificates
  - Static token files (deprecated)
  - Bearer tokens (ServiceAccounts)
  - Authentication proxy
  - OpenID Connect (OIDC) - recommended

- **Authorization**: What can you do?
  - RBAC (recommended) ✅
  - ABAC (attribute-based, deprecated)
  - Webhook
  - Node authorization

- **Admission Control**: Should this request be allowed?
  - **Validating**: Check if request is valid
  - **Mutating**: Modify request before persisting
  
**Common Admission Controllers**:
- PodSecurity (replaces PodSecurityPolicy)
- ResourceQuota
- LimitRanger
- NamespaceLifecycle
- ServiceAccount
- NodeRestriction

**Security Best Practices**:
- Enable audit logging (`--audit-log-path`)
- Encrypt API server to kubelet communication (TLS)
- Disable anonymous auth (`--anonymous-auth=false`)
- Disable insecure port (`--insecure-port=0`)
- Enable admission controllers
- Rate limiting (`--max-requests-inflight`)

#### **etcd**
- **Stores all cluster data** (Secrets, ConfigMaps, etc.)
- **Security concerns**:
  - Unencrypted data at rest by default
  - Direct access = full cluster compromise

**Securing etcd**:
- Enable encryption at rest (`EncryptionConfiguration`)
- Mutual TLS between API server and etcd
- Restrict network access (firewall)
- Regular backups (encrypted)
- Consider external etcd cluster (separate nodes)

**Encryption at Rest**:
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64 secret>
      - identity: {}
```

#### **Controller Manager & Scheduler**
- Communicate only with API server (not directly with etcd)
- Use ServiceAccount tokens
- Require TLS
- Disable profiling endpoints in production

### 2.2 Node Security

#### **kubelet**
- Runs on each node
- Manages containers
- **Security risks**:
  - Exposed API (unauthenticated read access)
  - Can exec into containers
  - Access to node filesystem

**Hardening kubelet**:
- Enable authentication (`--anonymous-auth=false`)
- Enable authorization (`--authorization-mode=Webhook`)
- Read-only port disabled (`--read-only-port=0`)
- Rotate certificates (`--rotate-certificates`)
- TLS for serving (`--tls-cert-file`, `--tls-private-key-file`)
- Protect kernel (`--protect-kernel-defaults`)

#### **kube-proxy**
- Network proxy on each node
- Less critical but still secure

#### **Container Runtime**
- containerd, CRI-O
- Security considerations:
  - Run as non-root
  - Use user namespaces
  - Seccomp, AppArmor, SELinux profiles
  - cgroup limits

### 2.3 Node OS Hardening

**CIS Benchmarks**: Industry standard for OS security
- Minimal OS (Container-Optimized OS, Flatcar, Bottlerocket)
- Disable unnecessary services
- Regular patching
- Firewall rules
- SSH hardening (key-based auth, disable root login)
- Audit logging (auditd)

**Kernel Security**:
- Seccomp: Restrict syscalls
- AppArmor/SELinux: Mandatory Access Control
- User namespaces: UID/GID mapping

---

## 🛡️ Domain 3: Kubernetes Security Fundamentals (~20%)

### 3.1 RBAC (Deep Dive)

**Four Resources**:
1. **Role**: Permissions within namespace
2. **ClusterRole**: Cluster-wide permissions
3. **RoleBinding**: Grants Role to subjects in namespace
4. **ClusterRoleBinding**: Grants ClusterRole cluster-wide

**Subjects**: User, Group, ServiceAccount

**Verbs**: get, list, watch, create, update, patch, delete, deletecollection

**Best Practices**:
- Least privilege (minimum permissions needed)
- Namespace-level roles when possible
- Avoid wildcard permissions (`*`)
- Regular RBAC audits
- Use groups instead of individual users
- ServiceAccounts for Pods, not humans

**Common Patterns**:
```bash
# View-only access
verbs: [get, list, watch]

# Full access
verbs: [get, list, watch, create, update, patch, delete]

# Dangerous (avoid)
verbs: ["*"]
resources: ["*"]
```

**Aggregated ClusterRoles**: Combine multiple ClusterRoles
- `view` (read-only)
- `edit` (read-write, no RBAC changes)
- `admin` (full namespace control, can modify RBAC)
- `cluster-admin` (god mode, full cluster control)

**RBAC for Specific Use Cases**:
- Read-only user: `view` ClusterRole
- Developer: `edit` ClusterRole
- Namespace admin: `admin` ClusterRole
- Cluster admin: `cluster-admin` ClusterRole

### 3.2 Pod Security Standards

**Replaces PodSecurityPolicy (deprecated in 1.21, removed in 1.25)**

**Three Levels** (enforced via admission controller):

1. **Privileged**: Unrestricted (no restrictions)
   - For system/infrastructure pods
   
2. **Baseline**: Minimally restrictive
   - Prevents known privilege escalations
   - Default for most workloads

3. **Restricted**: Heavily restricted, hardened
   - Best practice, defense in depth
   - For security-sensitive workloads

**Applying PSS** (namespace labels):
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Modes**:
- `enforce`: Reject violating Pods
- `audit`: Log violations (allow Pod)
- `warn`: Warning to user (allow Pod)

**Restricted Profile Restrictions**:
- Must run as non-root
- No privileged containers
- No host namespaces (network, PID, IPC)
- No hostPath volumes
- Drop ALL capabilities, add only required
- Seccomp profile (RuntimeDefault or Localhost)
- Read-only root filesystem (recommended)

### 3.3 SecurityContext (Pod & Container Level)

**Pod-level**:
```yaml
securityContext:
  runAsUser: 1000          # UID
  runAsGroup: 3000         # GID
  fsGroup: 2000            # File ownership
  fsGroupChangePolicy: "OnRootMismatch"
  runAsNonRoot: true       # Enforce non-root
  seccompProfile:
    type: RuntimeDefault
  supplementalGroups: [4000]
  sysctls:                 # Kernel parameters
    - name: net.ipv4.ip_forward
      value: "1"
```

**Container-level** (overrides Pod-level):
```yaml
securityContext:
  runAsUser: 2000
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE  # Bind to ports < 1024
  seccompProfile:
    type: RuntimeDefault
  seLinuxOptions:
    level: "s0:c123,c456"
```

**Capabilities**:
- Linux capabilities split root privileges
- Drop ALL, add only what's needed
- Common: NET_BIND_SERVICE, CHOWN, DAC_OVERRIDE

**Dangerous Capabilities (avoid)**:
- CAP_SYS_ADMIN (near root)
- CAP_NET_ADMIN (network control)
- CAP_SYS_PTRACE (debug any process)

### 3.4 Network Policies

**Default**: All traffic allowed (no policies = allow all)
**Once a NetworkPolicy exists**: Default deny, then allow specific

**Types**:
- **Ingress**: Incoming traffic to Pod
- **Egress**: Outgoing traffic from Pod

**Selectors**:
- `podSelector`: Target Pods by labels
- `namespaceSelector`: Allow from specific namespaces
- `ipBlock`: Allow from CIDR ranges

**Example: Default Deny All**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}  # All pods in namespace
  policyTypes:
    - Ingress
    - Egress
```

**Example: Allow Frontend → Backend**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
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
```

**Best Practices**:
- Default deny all, then whitelist
- Separate policies for ingress and egress
- Use namespaceSelector for multi-namespace apps
- Test policies in non-prod first
- CNI plugin must support NetworkPolicy (Calico, Cilium)

### 3.5 Secrets Management

**K8s Secrets** (built-in):
- Base64 encoded (NOT encrypted)
- Stored in etcd
- Can be mounted as volumes or env vars

**Security Issues**:
- Not encrypted in etcd by default
- Base64 is encoding, not encryption
- Visible to anyone with RBAC access

**Securing K8s Secrets**:
- Enable etcd encryption at rest
- RBAC: Restrict Secret access (`get`, `list` verbs)
- Immutable Secrets (prevent modification)
- Short-lived Secrets
- Avoid env vars (use volume mounts instead)
  - Env vars visible in `docker inspect`, process list
  - Volumes: More secure, atomic updates

**External Secret Management** (better):
- **HashiCorp Vault**: Secret storage, dynamic secrets
- **AWS Secrets Manager**: Cloud-native
- **Google Secret Manager**
- **Azure Key Vault**
- **External Secrets Operator**: Sync external → K8s Secrets

**ServiceAccount Tokens**:
- Mounted in Pods (`/var/run/secrets/kubernetes.io/serviceaccount/token`)
- Used for Pod → API server auth
- Projected volume (more secure than auto-mounted secret)
- Token expiration (time-bound tokens)

**Best Practices**:
- Use external secret managers
- Enable encryption at rest
- Rotate secrets regularly
- Principle of least privilege (RBAC)
- Audit Secret access

---

## ⚠️ Domain 4: Kubernetes Threat Model (~15%)

### 4.1 Attack Surface Analysis

**Entry Points**:
1. **API Server**: Primary attack target
2. **kubelet API**: Node access
3. **Applications**: Vulnerable app code
4. **Supply Chain**: Malicious images, dependencies
5. **Users**: Compromised credentials

**Attack Chains**:

**Example 1: Container Escape**:
1. Exploit vulnerable app → RCE in container
2. Privileged container or kernel exploit
3. Escape to node
4. Access kubelet, steal creds
5. Access API server
6. Cluster takeover

**Example 2: RBAC Escalation**:
1. Compromised ServiceAccount with `create pods` permission
2. Create Pod with `hostPath` mounting node filesystem
3. Access node's ServiceAccount token
4. Escalate to higher privileges
5. Cluster compromise

### 4.2 Common Vulnerabilities

**Misconfigurations** (most common):
- Overly permissive RBAC
- No NetworkPolicies
- Privileged containers
- Exposed API server (no auth)
- Unencrypted etcd
- Disabled admission controllers
- Host namespaces enabled

**Container Vulnerabilities**:
- CVEs in base images
- Outdated dependencies
- Malicious images
- Image pulled from untrusted registry

**Supply Chain Attacks**:
- Compromised CI/CD pipeline
- Malicious dependencies (typosquatting)
- Backdoored images
- Unsigned/unverified images

### 4.3 Data Flow & Trust Boundaries

**Trust Zones**:
1. **Untrusted**: Internet, external users
2. **DMZ**: Ingress controllers, load balancers
3. **Application**: Pods, services
4. **Data**: Databases, persistent storage
5. **Control Plane**: API server, etcd (most trusted)

**Data Flows to Secure**:
- External → Ingress (TLS, WAF)
- Ingress → Services (mTLS with service mesh)
- Pod → Pod (NetworkPolicy, mTLS)
- Pod → API server (RBAC, TLS)
- API server → etcd (TLS, encryption at rest)
- Node → Node (network isolation)

### 4.4 Incident Response

**Phases**:
1. **Preparation**: IR plan, runbooks, tools
2. **Detection**: Monitoring, alerts, anomaly detection
3. **Containment**: Isolate compromised resources
4. **Eradication**: Remove threat
5. **Recovery**: Restore services
6. **Lessons Learned**: Post-mortem, improve

**Forensics**:
- Audit logs (API server audit)
- Container logs
- Runtime security alerts (Falco)
- Network traffic analysis
- Node forensics (if compromised)

**Containment Actions**:
- Isolate Pod (NetworkPolicy deny all)
- Delete compromised Pods
- Revoke ServiceAccount tokens
- Block egress to C2 servers
- Cordon/drain compromised nodes

---

## 🏗️ Domain 5: Platform Security (~10%)

### 5.1 Supply Chain Security

**Software Supply Chain**: Code → Build → Package → Deploy

**SLSA Framework** (Supply-chain Levels for Software Artifacts):
- L1: Documentation of build process
- L2: Tamper resistance (signed builds)
- L3: Extra resistance (hardened build platform)
- L4: Highest level (two-party review)

**Key Concepts**:

**SBOM (Software Bill of Materials)**:
- List of all components in software
- Includes dependencies, versions, licenses
- Used for vulnerability tracking
- Tools: Syft, SPDX

**Image Signing & Verification**:
- **Cosign** (Sigstore project): Sign container images
- **Notary** (Docker Content Trust): Image signing
- **Policy Controller**: Enforce signed images only

**Example Flow**:
1. Build image in CI/CD
2. Scan for vulnerabilities (Trivy, Grype)
3. Sign image with Cosign
4. Push to registry
5. Admission controller verifies signature
6. Only signed images allowed to run

### 5.2 Image Security

**Vulnerability Scanning**:
- **Trivy**: Comprehensive scanner (images, filesystems, Git repos)
- **Grype**: Fast, accurate
- **Clair**: Container scanning
- **Anchore**: Policy-based scanning

**Scanning in CI/CD**:
- Scan on build (fail if high/critical CVEs)
- Scan on push to registry
- Continuous scanning in registry
- Admission webhook blocks vulnerable images

**Image Best Practices**:
- Use minimal base images (distroless, Alpine, scratch)
- Multi-stage builds (separate build and runtime)
- Non-root user
- No secrets in layers
- Pin versions (no `latest` tag)
- Regular updates
- Signed and verified

**Private Registries**:
- Harbor: CNCF project, scanning, signing, RBAC
- Docker Registry: Basic, self-hosted
- Cloud registries: ECR, GCR, ACR (scanning built-in)

### 5.3 Admission Control & Policy Enforcement

**OPA (Open Policy Agent)**:
- General-purpose policy engine
- Rego policy language
- Can enforce policies on K8s resources

**Gatekeeper**: OPA for Kubernetes
- CustomResourceDefinitions for policies
- Constraint Templates (reusable)
- Constraints (instances of templates)

**Example: Require labels**:
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-app-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    labels:
      - key: "app"
```

**Kyverno**: K8s-native policy engine
- YAML policies (no Rego)
- Validate, Mutate, Generate resources
- Simpler than OPA/Gatekeeper

**Example: Require non-root**:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: enforce
  rules:
    - name: check-runAsNonRoot
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Pods must run as non-root"
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true
```

**Policy Types**:
- Image source (only trusted registries)
- Image signing (must be signed)
- Resource limits (must have requests/limits)
- Security contexts (non-root, capabilities)
- Labels (required metadata)
- Ingress rules (TLS required)

### 5.4 Runtime Security

**Falco** (CNCF graduated):
- Detects anomalous activity
- Uses eBPF or kernel module
- Rule-based detection
- Alerts on suspicious behavior

**What Falco Detects**:
- Shell spawned in container
- Sensitive file access (`/etc/shadow`, `/etc/passwd`)
- Unexpected network connections
- Privilege escalation
- Container escape attempts
- Crypto mining

**Example Falco Rule**:
```yaml
- rule: Shell in Container
  desc: Detect shell spawned in container
  condition: >
    spawned_process and
    container and
    proc.name in (shell_binaries)
  output: Shell spawned in container (user=%user.name container=%container.name shell=%proc.name)
  priority: WARNING
```

**Other Runtime Security Tools**:
- **Sysdig**: Commercial, based on Falco
- **Aqua Security**: Container security platform
- **Prisma Cloud**: Palo Alto, runtime protection

---

## 📜 Domain 6: Compliance & Security Frameworks (~5%)

### 6.1 CIS Kubernetes Benchmark

**Center for Internet Security** standards:
- K8s benchmark: Security best practices
- Scored (must implement) vs Not Scored (should implement)
- Covers control plane, worker nodes, policies

**Key Sections**:
1. Control Plane Components
2. etcd
3. Control Plane Configuration
4. Worker Nodes
5. Policies (RBAC, PSS, NetworkPolicy)

**Automated Scanning**:
- **kube-bench**: CIS benchmark checker
  - Runs on nodes, checks configuration
  - Reports PASS/FAIL/WARN
  - Remediation instructions

**Example kube-bench**:
```bash
kube-bench run --targets master
kube-bench run --targets node
```

### 6.2 Regulatory Compliance

**PCI-DSS** (Payment Card Industry):
- For handling credit card data
- Encryption, access control, monitoring
- K8s considerations: Secrets encryption, audit logs, NetworkPolicy

**HIPAA** (Healthcare):
- Protect health information
- Encryption at rest and in transit
- Access controls, audit logs
- Data retention policies

**SOC 2** (Service Organization Control):
- Trust principles: Security, Availability, Confidentiality
- Vendor security assessments
- K8s: RBAC, monitoring, incident response

**GDPR** (General Data Protection Regulation):
- EU data privacy
- Data encryption, right to erasure
- Data residency (region-specific storage)

### 6.3 Security Auditing

**Kubernetes Audit Logs**:
- Records API server requests
- Who, what, when
- Levels: None, Metadata, Request, RequestResponse

**Audit Policy**:
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  - level: Metadata
    verbs: ["get", "list", "watch"]
```

**Audit Backends**:
- Log (file-based)
- Webhook (send to SIEM)
- Dynamic audit (flexible routing)

**What to Audit**:
- Secret access
- RBAC changes
- Privileged Pod creation
- Namespace deletion
- ConfigMap modifications
- Exec into Pods

**Tools**:
- **Falco**: Runtime security, audit logs
- **Splunk**: SIEM, log aggregation
- **ELK Stack**: Elasticsearch, Logstash, Kibana

---

## 🔥 Key Security Tools & Projects (Know These!)

### Image Scanning
- **Trivy**: Comprehensive, easy to use
- **Grype**: Fast, accurate
- **Clair**: Docker registry scanner

### Image Signing
- **Cosign**: Sigstore project
- **Notary**: Docker Content Trust

### Policy Enforcement
- **OPA/Gatekeeper**: Flexible, Rego language
- **Kyverno**: K8s-native, YAML policies

### Runtime Security
- **Falco**: Anomaly detection, eBPF
- **Tetragon**: eBPF-based, Cilium project

### Secrets Management
- **HashiCorp Vault**: Industry standard
- **External Secrets Operator**: K8s integration
- **Sealed Secrets**: Encrypt Secrets in Git

### Compliance
- **kube-bench**: CIS benchmark scanner
- **Popeye**: K8s cluster sanitizer (best practices)

### Vulnerability Management
- **Snyk**: Developer-first, SCA
- **Aqua Security**: Container platform

### Network Security
- **Cilium**: eBPF-based CNI, NetworkPolicy++
- **Calico**: Network policy enforcement
- **Istio**: Service mesh, mTLS

---

## 🎯 Study Strategy - KCNA + KCSA Same Day

### Scheduling Strategy

**Option 1: KCNA morning, KCSA afternoon** (recommended)
- 9 AM: KCNA (90 min + 30 min buffer)
- 11 AM: Break, review KCSA notes
- 2 PM: KCSA (90 min)
- **Pros**: Fresh for both, easier exam first
- **Cons**: Mental fatigue by afternoon

**Option 2: KCSA morning, KCNA afternoon**
- KCSA is harder, take it fresh
- **Cons**: If KCSA stresses you, affects KCNA

**Recommendation**: KCNA first. It builds foundation for KCSA.

### Pre-Exam Prep (3 days)

**Day 1: KCNA Focus + KCSA Intro**
- Morning: KCNA crash course
- Afternoon: KCSA domains 1-2 (4Cs, threat modeling)
- Evening: KCNA practice exam

**Day 2: KCSA Deep Dive**
- Morning: Domains 3-4 (K8s security fundamentals, threat model)
- Afternoon: Domains 5-6 (platform security, compliance)
- Evening: KCSA practice exam

**Day 3: Review & Practice**
- Morning: KCNA weak areas
- Afternoon: KCSA weak areas
- Evening: Both practice exams

### Between Exams (2-hour break)

**Don't**:
- ❌ Study new material
- ❌ Take practice exams
- ❌ Stress about KCNA results

**Do**:
- ✅ Eat lunch (protein, avoid heavy carbs)
- ✅ Walk outside (fresh air, movement)
- ✅ Review KCSA flashcards (quick refresh)
- ✅ Hydrate
- ✅ Relax (confidence boost)

---

## 📝 Quick Reference - KCSA Edition

### 4Cs One-Liner
**Cloud → Cluster → Container → Code** (security in layers)

### STRIDE
**S**poofing, **T**ampering, **R**epudiation, **I**nfo Disclosure, **D**oS, **E**levation

### Pod Security Standards
- **Privileged**: No restrictions
- **Baseline**: Basic security
- **Restricted**: Hardened (production)

### RBAC Quick Matrix
| Subject | Binding | Role | Scope |
|---------|---------|------|-------|
| User/Group | RoleBinding | Role | Namespace |
| User/Group | ClusterRoleBinding | ClusterRole | Cluster |
| ServiceAccount | RoleBinding | Role | Namespace |

### Image Security Checklist
- ✅ Scan for CVEs
- ✅ Sign with Cosign
- ✅ Minimal base image
- ✅ Non-root user
- ✅ Read-only filesystem
- ✅ Drop ALL capabilities

### NetworkPolicy Pattern
1. Default deny all
2. Allow specific ingress
3. Allow specific egress
4. Test in non-prod

### etcd Security
- ✅ Encryption at rest
- ✅ TLS communication
- ✅ Restrict network access
- ✅ Regular backups (encrypted)

### Admission Control Flow
**Request → Authentication → Authorization → Admission (Mutating → Validating) → Persist to etcd**

---

## 🚨 Common Traps & Gotchas

### KCSA-Specific Mistakes

1. **Confusing 4Cs order**: It's **Code → Container → Cluster → Cloud** (bottom-up), but security model is **Cloud → Cluster → Container → Code** (top-down)

2. **STRIDE vs CIA Triad**: 
   - STRIDE = Threat modeling
   - CIA (Confidentiality, Integrity, Availability) = Security goals

3. **Pod Security Standards vs SecurityContext**:
   - PSS = Admission controller (namespace-level policy)
   - SecurityContext = Pod/container spec (what Pod runs as)

4. **OPA vs Gatekeeper vs Kyverno**:
   - OPA = General policy engine
   - Gatekeeper = OPA for K8s (Rego)
   - Kyverno = K8s-native (YAML, easier)

5. **Falco vs Audit Logs**:
   - Falco = Runtime anomaly detection (process, network, file)
   - Audit logs = API server request logs (what happened in K8s API)

6. **Image scanning vs Runtime security**:
   - Scanning = Static analysis (find CVEs before deploy)
   - Runtime = Dynamic (detect attacks during execution)

7. **Secrets encryption vs Secrets management**:
   - Encryption = etcd encryption at rest
   - Management = External (Vault, Sealed Secrets)

8. **RBAC is additive**: No deny rules, only allow

9. **NetworkPolicy default**:
   - No policies = allow all
   - One policy = default deny

10. **ServiceAccount token location**: `/var/run/secrets/kubernetes.io/serviceaccount/token`

### Question Patterns to Watch

**Pattern 1: Best practice questions**
"What is the BEST way to secure Secrets?"
→ External secret manager (not just etcd encryption)

**Pattern 2: Scenario-based**
"An attacker gained access to a Pod. What should you check?"
→ Audit logs, Falco alerts, RBAC permissions, Pod SecurityContext

**Pattern 3: Tool selection**
"Which tool detects runtime anomalies?"
→ Falco (not Trivy, which is scanning)

**Pattern 4: Compliance**
"What does CIS benchmark recommend for API server?"
→ Disable anonymous auth, enable audit logging, etc.

**Pattern 5: Defense in depth**
"Multiple layers of security include..."
→ RBAC + NetworkPolicy + PSS + Image scanning

---

## 📚 Essential Resources

### Official
- **CNCF KCSA Curriculum**: https://github.com/cncf/curriculum/blob/master/KCSA_Curriculum.pdf
- **K8s Security Docs**: https://kubernetes.io/docs/concepts/security/
- **CIS Benchmark**: https://www.cisecurity.org/benchmark/kubernetes

### Courses
- **LinkedIn Learning**: Michael Levan's KCSA Cert Prep (highly recommended)
- **O'Reilly**: KCSA Crash Course

### Practice
- **GitHub KCSA Mock**: https://kubernetes-security-kcsa-mock.vercel.app/ (290+ questions!)
- **Udemy**: KCSA Practice Exams (multiple vendors)

### Reading
- **Kubernetes Security** by Liz Rice & Michael Hausenblas
- **4Cs of Cloud Native Security**: https://kubernetes.io/docs/concepts/security/overview/
- **OWASP K8s Top 10**: https://owasp.org/www-project-kubernetes-top-ten/

### Tools to Try (Optional but Helpful)
- **Trivy**: `trivy image nginx:latest`
- **kube-bench**: Check CIS compliance
- **Falco**: Runtime security rules
- **Cosign**: Sign container images

---

## 🎓 Final 24 Hours Checklist - KCSA

### Must Review
- [ ] 4Cs model (explain each layer)
- [ ] STRIDE threat model
- [ ] Pod Security Standards (3 levels)
- [ ] RBAC (4 resources, how they work)
- [ ] NetworkPolicy (default behavior, selectors)
- [ ] SecurityContext (common fields)
- [ ] etcd encryption at rest
- [ ] Admission controllers (validating, mutating)
- [ ] Falco (what it detects)
- [ ] Image signing (Cosign, Notary)
- [ ] Supply chain security (SLSA, SBOM)
- [ ] CIS benchmarks (kube-bench)
- [ ] OPA/Kyverno (policy enforcement)
- [ ] Common attack vectors
- [ ] kubelet security
- [ ] ServiceAccount tokens

### Quick Drills
- [ ] 4Cs → STRIDE mapping (20 questions)
- [ ] Tool → Use case quiz (Falco vs Trivy vs kube-bench)
- [ ] SecurityContext scenarios
- [ ] RBAC troubleshooting
- [ ] Threat scenarios (what would you check?)

### Mental Prep
- [ ] Sleep 8 hours before each exam
- [ ] KCSA is harder than KCNA, expect it
- [ ] Both are conceptual (no terminal work)
- [ ] Mark tricky questions, come back
- [ ] 75% to pass, aim for 85%+

---

## 💪 You've Got This!

**Your advantages**:
- ✅ Mid-CKA level = strong K8s foundation
- ✅ KCNA prep overlaps ~40% with KCSA
- ✅ DevOps experience = practical security exposure

**Your focus areas**:
- 🎯 4Cs security model (memorize)
- 🎯 Threat modeling (STRIDE)
- 🎯 Security tools (Falco, Trivy, Cosign, OPA)
- 🎯 Supply chain security
- 🎯 Compliance frameworks

**Time allocation**:
- KCNA prep: 40% of study time (you're already solid)
- KCSA prep: 60% of study time (new concepts)

**Exam day**:
- KCNA first (confidence boost)
- 2-hour break (relax, don't cram)
- KCSA second (fresh on security)

With 3 focused days, you'll crush both. Let's get you those certs! 🚀

---

## 📋 Pointer to Deep Dive Topics

When you need more depth on specific topics, read these sections of official docs:

### Critical Deep Dives
1. **4Cs of Cloud Native Security**: 
   - https://kubernetes.io/docs/concepts/security/overview/
   - Read 3x, understand each layer

2. **Pod Security Standards**:
   - https://kubernetes.io/docs/concepts/security/pod-security-standards/
   - Know restricted profile cold

3. **RBAC**:
   - https://kubernetes.io/docs/reference/access-authn-authz/rbac/
   - Practice examples

4. **Network Policies**:
   - https://kubernetes.io/docs/concepts/services-networking/network-policies/
   - Understand selectors

5. **Secrets Management**:
   - https://kubernetes.io/docs/concepts/configuration/secret/
   - External Secrets Operator docs

6. **Admission Controllers**:
   - https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/
   - Know common ones

7. **Audit Logging**:
   - https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/
   - Understand levels

8. **CIS Benchmark**:
   - Download PDF from CIS website
   - Skim sections, know categories

9. **Falco Rules**:
   - https://falco.org/docs/rules/
   - Understand rule structure

10. **SLSA Framework**:
    - https://slsa.dev/
    - Know 4 levels

---

*Last Updated: December 2025*  
*Version: 1.0*  
*Target Exams: KCNA + KCSA 2025*  
*Sensei Mode: Activated* 🥋
