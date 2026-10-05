# CKAD 2026 Crash Course - Application Developer Edition
## Certified Kubernetes Application Developer - Post-CKA Speed Run

**Target Audience**: Engineers who just completed CKA, taking CKAD same day/week  
**Time Required**: 1-2 weeks (you already know 70%!)  
**Goal**: 85%+ score, leverage CKA speed skills  
**Kubernetes Version**: v1.35  
**Last Updated**: July 2026 (Real exam feedback integrated)

**Your Status**: CKA ✅ → **CKAD (next)** → CKS

> ⚠️ **2026 UPDATE**: CKAD is TRICKIER than 2025 mocks. ConfigMap merge logic and egress rules are harder. NetworkPolicy DNS gotchas (UDP 53) caught many. See exam feedback below.

---

## 🎯 Your Massive Advantage

**What you ALREADY have from CKA:**
- ✅ kubectl speed (imperative commands, dry-run)
- ✅ vim proficiency (YAML editing)
- ✅ Doc navigation (find answers fast)
- ✅ Pods, Deployments, Services, ConfigMaps, Secrets
- ✅ RBAC, NetworkPolicy basics
- ✅ Storage (PV, PVC)
- ✅ Troubleshooting methodology
- ✅ Helm & Kustomize (NEW in CKA 2025)
- ✅ **You're already FAST**

**What's NEW/DIFFERENT in CKAD (~30%):**
- 🔥 **Multi-container Pods** (sidecar, init, adapter, ambassador patterns)
- 🔥 **Probes** (liveness, readiness, startup - deeper than CKA)
- 🔥 **Jobs & CronJobs** (more emphasis, completion/parallelism)
- 🔥 **Deployment strategies** (blue/green, canary - manual implementation)
- 🔥 **Container image building** (Dockerfile, build process)
- 🔥 **Resource quotas** (namespace-level limits)
- 🔥 **Application debugging** (more app-focused vs cluster-focused)
- 🔥 **API deprecations** (identify and fix)

**Key Difference: Application vs Infrastructure**
- CKA: "Keep the cluster running" (admin focus)
- CKAD: "Deploy and maintain apps" (developer focus)
- CKA: Cluster troubleshooting, upgrades, RBAC
- CKAD: App configs, probes, multi-container patterns

---

## 📋 Exam Overview - CKAD 2025

**Format**: Performance-based (same as CKA!)  
**Duration**: 2 hours (same!)  
**Questions**: 15-20 tasks  
**Passing Score**: 66% (easier than CKA's 66%, but aim for 85%+)  
**Cost**: $445 (includes 1 free retake)  
**Validity**: 2 years  
**Environment**: PSI remote browser, multiple clusters

**Allowed Resources**: Same as CKA
- kubernetes.io/docs
- kubernetes.io/blog  
- github.com/kubernetes

**CRITICAL**: Questions are typically FASTER than CKA
- More straightforward tasks
- Less cluster debugging, more app deployment
- You should finish with 20-30 min to spare (if CKA prepared)

---

## 🚨 2026 Exam Feedback (Real Test Taker Data)

**Where 2026 CKAD was HARDER than mocks:**

1. **NetworkPolicy Egress** (HIGH PRIORITY)
   - ⚠️ Candidates forgot DNS (UDP 53) multiple times
   - Config maps merging logic was **much more complex** than expected
   - **Action**: Practice egress rules with explicit DNS allowance
   ```yaml
   - to:
     - namespaceSelector: {}
     ports:
     - protocol: UDP
       port: 53  # ← People forgot this!
   ```

2. **Selector Mismatches** 
   - More endpoint debugging required
   - "Why isn't traffic reaching the Pod?" scenarios
   - Practice troubleshooting service→pod connectivity

3. **CronJob Schedule Syntax**
   - Typos in `.spec.schedule` were PENALIZED
   - Know cron format cold: `minute hour day month weekday`
   - Test locally: `0 2 * * *` = 2 AM daily

4. **Native Sidecars (v1.35)**
   - Appeared but were simple (if you know the init container change)
   - Just remember `restartPolicy: Always` for init containers

5. **What was EASIER than expected:**
   - ServiceAccount setup: straightforward
   - Rollbacks: mock exams prepared well
   - ConfigMap/Secret basics: standard questions

---

## 🎯 CKAD 2025 Domains & Weights

| Domain | Weight | Questions | CKA Overlap | Your Focus |
|--------|--------|-----------|-------------|------------|
| **Application Environment, Configuration & Security** | 25% | ~4-5 | 60% overlap | MEDIUM - Review + new topics |
| **Application Design and Build** | 20% | ~3-4 | 40% overlap | HIGH - Multi-container, images |
| **Application Deployment** | 20% | ~3-4 | 70% overlap | LOW - You know this |
| **Services & Networking** | 20% | ~3-4 | 80% overlap | LOW - Quick review |
| **Application Observability & Maintenance** | 15% | ~2-3 | 50% overlap | HIGH - Probes, debugging |

---

## 🔐 Domain 1: Application Environment, Configuration & Security (25%)

### What You Already Know from CKA
✅ ConfigMaps & Secrets (creation, mounting)  
✅ SecurityContext basics  
✅ ServiceAccounts  
✅ Resource requests/limits  
✅ RBAC

### NEW/DEEPER Topics for CKAD

#### 1.1 Custom Resource Definitions (CRDs) - Quick Review

You learned this in CKA, but CKAD may ask you to **use** custom resources:

```bash
# Apply existing CRD (given in question)
k apply -f crd.yaml

# Create custom resource
k apply -f my-custom-resource.yaml

# List custom resources
k get <plural-name>
k get crontabs  # example
```

**Exam tip**: Usually just need to apply existing CRD and create instance

#### 1.2 Resource Quotas (NEW emphasis)

**ResourceQuota**: Limits resource consumption per namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "10"
    services: "5"
    persistentvolumeclaims: "4"
```

**LimitRange**: Default/min/max for containers in namespace

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
  namespace: dev
spec:
  limits:
  - default:  # Default limits
      cpu: 500m
      memory: 512Mi
    defaultRequest:  # Default requests
      cpu: 200m
      memory: 256Mi
    max:  # Max limits
      cpu: 1000m
      memory: 1Gi
    min:  # Min requests
      cpu: 100m
      memory: 128Mi
    type: Container
```

**Common exam scenarios**:
- Create ResourceQuota for namespace
- Pod won't start because quota exceeded
- Set default resource limits with LimitRange

```bash
# Create ResourceQuota fast
kubectl create quota my-quota --hard=cpu=10,memory=20Gi,pods=10 -n dev

# Check quota usage
kubectl describe quota -n dev
```

#### 1.3 SecurityContext - Application Focus

You know this from CKA, but CKAD emphasizes **why** for app security:

**Common app security patterns**:
```yaml
# Run as specific user (app requirement)
securityContext:
  runAsUser: 1000
  runAsGroup: 3000
  fsGroup: 2000

# Read-only filesystem (immutable app)
securityContext:
  readOnlyRootFilesystem: true

# Drop all capabilities (minimal privileges)
securityContext:
  capabilities:
    drop:
      - ALL
```

**Exam tip**: If app needs to write, use emptyDir volume with readOnlyRootFilesystem

```yaml
spec:
  containers:
  - name: app
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

#### 1.4 ServiceAccounts & RBAC (Application Perspective)

**From CKA**: You know how to create Role, RoleBinding  
**For CKAD**: Focus on **app-to-API** permissions

**Common pattern**: App needs to list/watch Pods
```bash
# Create ServiceAccount
k create sa app-reader

# Create Role (list pods)
k create role pod-reader --verb=get,list,watch --resource=pods

# Bind to ServiceAccount
k create rolebinding app-reader-binding --role=pod-reader --serviceaccount=default:app-reader

# Use in Pod
k run myapp --image=myapp --serviceaccount=app-reader
```

**Exam scenario**: App crashes with "forbidden" error → Fix RBAC

---

## 🏗️ Domain 2: Application Design and Build (20%)

### NEW/CRITICAL Topics

#### 2.1 Multi-Container Pod Patterns (VERY IMPORTANT!)

**This is a KEY CKAD differentiator. Master these patterns.**

**Pattern 1: Sidecar**
- Helper container runs alongside main app
- Examples: Log shipping, config syncing, proxy

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-example
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  - name: log-shipper  # Sidecar
    image: fluentd
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
      readOnly: true
  volumes:
  - name: shared-logs
    emptyDir: {}
```

**Pattern 2: Init Container**
- Runs before main container
- Must complete successfully
- Examples: Pre-populate data, wait for service

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-example
spec:
  initContainers:
  - name: init-db
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do sleep 2; done']
  containers:
  - name: app
    image: myapp
```

**Pattern 3: Adapter**
- Transforms output for consumption
- Example: Convert app logs to standard format

```yaml
spec:
  containers:
  - name: app
    image: legacy-app
    # Writes logs in custom format to /var/log/app.log
  - name: adapter
    image: log-adapter
    # Reads /var/log/app.log, converts to JSON, outputs to stdout
```

**Pattern 4: Ambassador**
- Proxy to external service
- Example: Connect to database via ambassador

```yaml
spec:
  containers:
  - name: app
    image: myapp
    # Connects to localhost:6379
  - name: redis-ambassador
    image: redis-proxy
    # Proxies localhost:6379 to actual Redis cluster
```

**Exam tips**:
- Sidecar: Shared volume for logs/data
- Init: Use initContainers (different YAML section!)
- Adapter: Read from volume, transform, output
- Ambassador: App connects to localhost, ambassador proxies

**Common exam scenarios**:
1. Add sidecar to collect logs
2. Add init container to wait for dependency
3. Debug why init container failing (check logs!)

```bash
# Check init container logs
k logs mypod -c init-container-name

# Check if init container completed
k describe pod mypod  # Look at Init Containers section
```

**NEW in v1.35: Native Sidecars**

Init containers with `restartPolicy: Always` now run as true sidecars:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-native-sidecar
spec:
  containers:
  - name: app
    image: myapp
  initContainers:
  - name: logging-agent  # Runs as sidecar due to restartPolicy
    image: fluentd:latest
    restartPolicy: Always  # ← This makes it a sidecar!
    volumeMounts:
    - name: logs
      mountPath: /app/logs
  volumes:
  - name: logs
    emptyDir: {}
```

**Why it matters**: Cleaner than dual-container sidecar pattern. Know the difference for the exam.

#### 2.2 Container Image Building (NEW!)

**Dockerfile Basics** (may need to write/fix):

```dockerfile
# Multi-stage build (best practice)
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

FROM alpine:latest
COPY --from=builder /app/myapp /usr/local/bin/
CMD ["myapp"]
```

**Common Dockerfile commands**:
- `FROM`: Base image
- `WORKDIR`: Set working directory
- `COPY`: Copy files from build context
- `ADD`: Like COPY, but can extract tar files, download URLs
- `RUN`: Execute command (creates new layer)
- `CMD`: Default command (can be overridden)
- `ENTRYPOINT`: Main command (not easily overridden)
- `ENV`: Set environment variables
- `EXPOSE`: Document port (doesn't actually expose!)
- `USER`: Set user for RUN, CMD, ENTRYPOINT
- `VOLUME`: Create mount point

**Best practices** (exam may ask):
- Use multi-stage builds (smaller final image)
- Run as non-root user
- Use specific version tags (not `latest`)
- Minimize layers (combine RUN commands)
- Use `.dockerignore` (exclude unnecessary files)
- Order layers by change frequency (deps first, code last)

**Build commands** (may need to know):
```bash
# Build image
docker build -t myapp:v1 .

# Build with specific Dockerfile
docker build -t myapp:v1 -f Dockerfile.prod .

# Multi-platform build (newer)
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:v1 .
```

**Exam scenarios**:
- Fix Dockerfile error
- Build image with specific tag
- Optimize Dockerfile (reduce size, layers)
- Change base image to non-root user

#### 2.3 Choosing Workload Resources (Review from CKA)

You know these, but CKAD emphasizes **when to use what**:

| Use Case | Resource |
|----------|----------|
| Stateless web app, scaling | Deployment |
| Database, ordered deployment | StatefulSet |
| Log collector (one per node) | DaemonSet |
| Batch processing | Job |
| Scheduled backup | CronJob |
| Long-running worker | Deployment (1 replica) |

**Exam tip**: Read question carefully for clues (scaling? stateful? one-time?)

#### 2.4 Volumes (Application Perspective)

**From CKA**: PV, PVC, StorageClass  
**For CKAD**: More emphasis on **ephemeral** volumes

**emptyDir**: Temporary, Pod lifetime
```yaml
volumes:
- name: cache
  emptyDir: {}
```

**hostPath**: Mount from node (avoid in prod!)
```yaml
volumes:
- name: data
  hostPath:
    path: /data
    type: Directory
```

**configMap as volume**:
```yaml
volumes:
- name: config
  configMap:
    name: app-config
    items:
    - key: app.conf
      path: app.conf
```

**secret as volume**:
```yaml
volumes:
- name: certs
  secret:
    secretName: tls-secret
```

**Exam pattern**: Use emptyDir for shared data between containers in Pod

---

## 🚀 Domain 3: Application Deployment (20%)

### You Already Know This from CKA!

✅ Deployments (create, scale, update)  
✅ Helm (install, upgrade, rollback)  
✅ Kustomize (apply overlays)

### NEW Emphasis: Deployment Strategies

#### 3.1 Blue/Green Deployment (Manual with K8s)

**Concept**: Run two versions, switch traffic instantly

**Implementation**:
```bash
# 1. Deploy blue version
k create deploy blue --image=app:v1 --replicas=3

# 2. Create service pointing to blue
k expose deploy blue --name=app-service --port=80 --selector=app=myapp,version=blue

# 3. Deploy green version
k create deploy green --image=app:v2 --replicas=3

# 4. Test green
k run test --image=busybox -it --rm -- wget -O- http://green-svc

# 5. Switch traffic (update service selector)
k set selector svc app-service 'version=green'

# 6. Rollback if needed
k set selector svc app-service 'version=blue'

# 7. Delete blue when confident
k delete deploy blue
```

**Exam tip**: Use labels to distinguish versions (`version=blue`, `version=green`)

#### 3.2 Canary Deployment

**Concept**: Gradually shift traffic to new version

**Implementation** (using replicas):
```bash
# 1. Deploy stable version (9 replicas)
k create deploy app-stable --image=app:v1 --replicas=9

# 2. Deploy canary (1 replica = 10% traffic)
k create deploy app-canary --image=app:v2 --replicas=1

# 3. Service selects both (same labels)
# selector: app=myapp (both deployments have this)

# 4. Monitor canary metrics

# 5. Gradually increase canary replicas
k scale deploy app-canary --replicas=5  # 50% traffic

# 6. Full rollout
k scale deploy app-canary --replicas=10
k delete deploy app-stable
```

**Exam scenarios**:
- Implement blue/green using labels
- Canary rollout (10% then 50%)
- Rollback deployment

#### 3.3 Rolling Updates (Default)

You know this from CKA:
```bash
# Update image (rolling update)
k set image deploy myapp app=app:v2

# Check rollout status
k rollout status deploy myapp

# Pause rollout
k rollout pause deploy myapp

# Resume
k rollout resume deploy myapp

# Rollback
k rollout undo deploy myapp

# Rollback to specific revision
k rollout undo deploy myapp --to-revision=2

# History
k rollout history deploy myapp
```

**maxSurge** and **maxUnavailable**:
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max extra Pods during update
      maxUnavailable: 0  # Max unavailable Pods
```

---

## 🌐 Domain 4: Services & Networking (20%)

### You ALREADY Know This from CKA!

✅ Service types (ClusterIP, NodePort, LoadBalancer)  
✅ Ingress  
✅ NetworkPolicy  
✅ CoreDNS

**For CKAD**: Just quick review, maybe 1-2 questions

### Quick Review

**Create Services** (fast):
```bash
# ClusterIP (default)
k expose deploy app --port=80 --target-port=8080

# NodePort
k expose deploy app --type=NodePort --port=80

# LoadBalancer
k expose deploy app --type=LoadBalancer --port=80

# Headless (for StatefulSet)
k create svc clusterip app --clusterip=None
```

**Ingress** (from docs):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

**NetworkPolicy** (deny all, then allow):
```yaml
# Deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress

# Allow from specific pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
```

**Exam tip**: If NetworkPolicy question, copy from docs and modify selectors

**⚠️ 2026 EXAM TRAP - DNS Egress Rule**

MANY candidates in 2026 failed egress policies because they forgot DNS. If you allow egress to a namespace but block DNS, external names won't resolve:

```yaml
# WRONG - Forgets DNS!
kind: NetworkPolicy
metadata:
  name: allow-external
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443

---

# CORRECT - Includes DNS!
kind: NetworkPolicy
metadata:
  name: allow-external-with-dns
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443
    - protocol: UDP
      port: 53  # ← DNS! Don't forget!
```

**Know this pattern cold.** It appeared on multiple 2026 exams.

---

## 🔍 Domain 5: Application Observability & Maintenance (15%)

### NEW/CRITICAL Topics

#### 5.1 Probes (VERY IMPORTANT for CKAD!)

**Three types**:

**Liveness Probe**: Is container alive? (restart if fails)
**Readiness Probe**: Is container ready for traffic? (remove from endpoints if fails)
**Startup Probe**: Has container started? (for slow-starting apps)

**Probe Methods**:

**HTTP GET**:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: Value
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 1
  failureThreshold: 3
  successThreshold: 1
```

**TCP Socket**:
```yaml
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

**Exec Command**:
```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 10
```

**Key Parameters**:
- `initialDelaySeconds`: Wait before first probe
- `periodSeconds`: How often to probe
- `timeoutSeconds`: Probe timeout
- `failureThreshold`: Failures before action (restart/remove from service)
- `successThreshold`: Successes to mark as healthy (usually 1)

**Common patterns**:

**Web app**:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 3
```

**Database**:
```yaml
livenessProbe:
  exec:
    command:
    - pg_isready
    - -U
    - postgres
  initialDelaySeconds: 30
  periodSeconds: 10
```

**Slow-starting app**:
```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  periodSeconds: 5
  failureThreshold: 30  # 30 * 5 = 150s max startup time

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
```

**Exam scenarios**:
- Add liveness probe to restart unhealthy container
- Add readiness probe to avoid traffic during startup
- Fix probe causing too many restarts (increase failureThreshold or initialDelay)
- Slow app keeps restarting → Add startup probe

**Debugging probe issues**:
```bash
# Check events
k describe pod mypod  # Look for Liveness/Readiness probe failed

# Check probe endpoint manually
k exec mypod -- wget -O- http://localhost:8080/healthz

# See probe config
k get pod mypod -o yaml | grep -A 10 livenessProbe
```

#### 5.2 Monitoring & Logging

**kubectl top** (requires metrics-server):
```bash
# Node usage
k top nodes

# Pod usage
k top pods

# Pod usage sorted
k top pods --sort-by=memory
k top pods --sort-by=cpu

# Specific container
k top pod mypod --containers
```

**Logs**:
```bash
# Basic logs
k logs mypod

# Specific container (multi-container pod)
k logs mypod -c container-name

# Previous container (after crash)
k logs mypod --previous
k logs mypod -c container-name --previous

# Follow logs
k logs -f mypod

# Last N lines
k logs mypod --tail=50

# Since time
k logs mypod --since=1h
k logs mypod --since=2024-01-01T00:00:00Z

# All containers
k logs mypod --all-containers=true

# Deployment logs
k logs deploy/myapp

# Labels
k logs -l app=myapp
```

**Exam scenarios**:
- App crashing → Check previous logs
- High memory → Check top pods
- Debug multi-container → Specify container with -c

#### 5.3 Debugging Applications

**Application won't start**:
```bash
# 1. Check pod status
k get pod mypod

# 2. Check events
k describe pod mypod

# 3. Check logs
k logs mypod
k logs mypod --previous  # If CrashLoopBackOff

# 4. Check container exit code
k get pod mypod -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'

# 5. Exec into pod (if running)
k exec -it mypod -- /bin/sh

# 6. Check environment
k exec mypod -- env

# 7. Check config/secrets mounted
k exec mypod -- ls /etc/config
k exec mypod -- cat /etc/config/app.conf
```

**App running but not accessible**:
```bash
# 1. Check service
k get svc myapp

# 2. Check endpoints (should match pod IPs)
k get endpoints myapp

# 3. Test from another pod
k run test --image=busybox -it --rm -- wget -O- http://myapp:80

# 4. Check readiness probe
k describe pod mypod  # Readiness section

# 5. Check network policy
k get netpol
```

**Performance issues**:
```bash
# 1. Check resource usage
k top pod mypod

# 2. Check resource limits
k describe pod mypod | grep -A 5 Limits

# 3. Check events (OOMKilled?)
k get events --sort-by='.lastTimestamp' | grep mypod

# 4. Check node resources
k top nodes
k describe node worker1
```

#### 5.4 API Deprecations (NEW emphasis)

**Identify deprecated APIs**:
```bash
# Check deprecation warnings
k get deploy -o yaml | grep -i deprecated

# Convert to new API version
k convert -f old-deployment.yaml --output-version apps/v1
```

**Common deprecations** (know these!):
- Deployment `extensions/v1beta1` → `apps/v1`
- Ingress `networking.k8s.io/v1beta1` → `networking.k8s.io/v1`
- PodSecurityPolicy (removed) → Pod Security Standards

**Exam scenario**: Fix deprecated API version in YAML

```yaml
# Old (deprecated)
apiVersion: extensions/v1beta1
kind: Deployment

# New
apiVersion: apps/v1
kind: Deployment
```

---

## ⚡ CKAD-Specific Speed Techniques

### You Already Have Speed from CKA!

Your CKA prep gave you:
- ✅ kubectl aliases
- ✅ vim config
- ✅ Imperative commands
- ✅ Dry-run templates
- ✅ Doc navigation

### Additional CKAD-Specific Shortcuts

#### Multi-Container Pod Template (Fast)

```bash
# Generate single-container pod
k run myapp --image=nginx --dry-run=client -o yaml > pod.yaml

# Edit to add second container (vim)
vim pod.yaml
# Duplicate containers section, change name/image
# If shared volume needed, add volumes section
```

**Or from docs** (faster for complex patterns):
1. Search "multi-container pod"
2. Copy sidecar example
3. Modify names/images
4. Apply

#### Probes (Fast)

```bash
# Generate pod
k run myapp --image=nginx --dry-run=client -o yaml > pod.yaml

# Edit to add probes
vim pod.yaml
# Add under spec.containers[0]
```

**Or from docs**:
1. Search "configure liveness readiness startup probes"
2. Copy example
3. Modify path/port
4. Paste into your YAML

#### Job/CronJob (Fast)

```bash
# Job
k create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)'

# CronJob
k create cj backup --image=backup:v1 --schedule="0 2 * * *" -- /backup.sh

# Job with completions/parallelism
k create job test --image=busybox --dry-run=client -o yaml -- echo "hello" > job.yaml
# Edit to add:
# spec:
#   completions: 5
#   parallelism: 2
k apply -f job.yaml
```

### Time-Saving Patterns

**Pattern: Init Container**
1. Generate pod
2. Add `initContainers:` section before `containers:`
3. Copy container template, modify for init

**Pattern: Sidecar with Shared Volume**
1. Generate pod with container 1
2. Add container 2 in `containers:` array
3. Add `volumes:` (emptyDir)
4. Add `volumeMounts:` to both containers

**Pattern: Blue/Green**
1. Create deployment with version label
2. Expose as service
3. Create second deployment with different version label
4. Update service selector to switch

---

## 📊 CKA vs CKAD: Key Differences

### Spend Your Prep Time on These

| Topic | CKA | CKAD |
|-------|-----|------|
| Multi-container Pods | Basic | **Deep (15% of exam)** |
| Probes | Mentioned | **Critical (10% of exam)** |
| Deployment strategies | Not emphasized | **Manual implementation** |
| Container images | Not tested | **Dockerfile knowledge** |
| Jobs/CronJobs | Basic | **Completions, parallelism** |
| Resource Quotas | Mentioned | **Namespace limits** |
| Application debugging | Less emphasis | **Primary focus** |
| Cluster setup/upgrade | **Major focus** | Not tested |
| RBAC deep dive | **Cluster-wide** | App-specific only |
| etcd backup/restore | **Tested** | Not tested |
| Node troubleshooting | **Major focus** | Minimal |

---

## 🎓 Study Plan - 1-2 Weeks Post-CKA

### You're 70% Done Already!

**Your CKA prep covered**:
- Pods, Deployments, Services ✅
- ConfigMaps, Secrets ✅
- Storage (PV, PVC) ✅
- RBAC basics ✅
- NetworkPolicy ✅
- Helm, Kustomize ✅
- Troubleshooting methodology ✅
- **Speed skills** ✅

**Focus ONLY on CKAD-specific**:

### Option 1: Same Day (CKA morning, CKAD afternoon)

**Morning**: Take CKA (9 AM - 11 AM)

**Break (11 AM - 2 PM)**:
- ✅ Lunch (light, protein)
- ✅ Walk outside (15 min)
- ✅ Review CKAD cheat sheet (30 min):
  - Multi-container patterns
  - Probe types
  - Job completions/parallelism
- ✅ Relax (don't cram!)

**Afternoon**: Take CKAD (2 PM - 4 PM)

### Option 2: Same Week (3-5 days apart)

**Day 1-2 (After CKA)**:
- Multi-container patterns (sidecar, init)
- Probes (liveness, readiness, startup)
- Practice adding probes to existing pods

**Day 3**:
- Jobs & CronJobs (completions, parallelism)
- Deployment strategies (blue/green, canary)
- Resource Quotas & LimitRange

**Day 4**:
- Container image building (Dockerfile review)
- API deprecations
- Application debugging scenarios

**Day 5**:
- killer.sh CKAD simulator (1st session)
- Identify gaps
- Practice weak areas

**Day 6-7** (if needed):
- killer.sh 2nd session
- Final review
- **Take CKAD**

### Daily Practice (30-60 min)

**Multi-container drill** (15 min):
1. Create pod with nginx + fluentd sidecar
2. Create pod with init container that waits for service
3. Time yourself: <5 min per pod

**Probes drill** (15 min):
1. Add liveness probe (HTTP) to pod
2. Add readiness probe (TCP) to pod
3. Add startup probe for slow app
4. Time yourself: <3 min per probe

**Jobs drill** (10 min):
1. Create Job with 5 completions, 2 parallel
2. Create CronJob that runs every 5 minutes
3. Time yourself: <2 min each

**Blue/Green drill** (20 min):
1. Implement blue/green deployment
2. Switch traffic
3. Rollback

---

## 🎯 Exam Day Strategy (CKAD)

### Pre-Exam (Same as CKA)

```bash
# Set aliases (if not from CKA session)
alias k=kubectl
complete -o default -F __start_kubectl k

# vim config (if needed)
echo "set nu et ts=2 sw=2" >> ~/.vimrc

# Verify contexts
k config get-contexts
```

### During Exam

**Time Management** (CKAD is faster than CKA):
- 2 hours / 15-20 questions = 6-8 min avg
- Easy questions (create pod, add probe): 2-3 min
- Medium (multi-container, debugging): 5-7 min
- Hard (blue/green, complex troubleshooting): 10 min

**Strategy**:
1. **Read all questions** (5 min) - Note easy ones
2. **Do easy wins first** (30 min) - Quick points
3. **Medium questions** (50 min)
4. **Hard questions** (25 min)
5. **Review/verify** (10 min)

**You should be FASTER than CKA** because:
- More straightforward tasks
- Less cluster debugging
- More kubectl create, less YAML editing

**Target**: Finish with 20-30 min to spare for review

### Common CKAD Question Patterns

**Pattern 1: Create pod with probes**
- Generate pod → Add liveness/readiness probes → Verify

**Pattern 2: Multi-container pod**
- Check pattern (sidecar/init) → Use docs template → Modify → Apply

**Pattern 3: Fix broken deployment**
- Check logs → Check events → Fix config/secret/probe → Verify

**Pattern 4: Implement blue/green**
- Create 2 deployments (different labels) → Service selector → Switch

**Pattern 5: Jobs with specific requirements**
- Create job → Add completions/parallelism → Verify completion

**Pattern 6: Add sidecar to existing deployment**
- Edit deployment → Add container → Add shared volume → Verify

---

## 📚 Essential Resources

### Official
- **CKAD Curriculum**: https://github.com/cncf/curriculum
- **K8s Docs**: https://kubernetes.io/docs/
- **kubectl Cheat Sheet**: (same as CKA)

### Courses (if you need)
- **KodeKloud CKAD**: Mumshad's course (updated 2025)
- **Udemy CKAD**: Same instructor

### Practice
- **killer.sh CKAD**: 2 sessions with exam (ESSENTIAL!)
- **Killercoda CKAD scenarios**: Free practice

### YouTube
- Search: "CKAD multi-container pod"
- Search: "CKAD probes tutorial"

---

## 💪 Final Pep Talk - CKAD Edition

**You're in an AMAZING position**:
- ✅ You just crushed CKA
- ✅ You have kubectl speed
- ✅ You have vim skills
- ✅ You know 70% of CKAD content already

**Your ONLY gaps**:
- 🔥 Multi-container patterns (2-3 days practice)
- 🔥 Probes (1 day practice)
- 🔥 Application-specific scenarios (2-3 days)

**CKAD is EASIER than CKA** (really!):
- More straightforward questions
- Less cluster-level complexity
- More kubectl create, less troubleshooting
- Faster to complete

**Your advantage from CKA**:
- Speed skills transfer 100%
- Docs navigation same
- kubectl muscle memory
- vim proficiency
- **You're already FAST**

**Same day strategy is SMART**:
- Same tools, same environment
- Momentum from CKA
- No time to forget kubectl commands
- CKAD feels easier after CKA

**Your sensei's prediction**:
- CKA: 80-85% (harder, more time pressure)
- CKAD: 85-90% (easier, you'll finish early)

**1-2 weeks focused on gaps = CKAD certified**

Now go crush it! Come back when you're ready for CKS (the final boss).

**Ganbatte, gakusei! You're almost Kubestronaut!** 🥋🚀

---

## 📋 CKAD Quick Reference Cheat Sheet

### Multi-Container Patterns
```yaml
# Sidecar (log collector)
spec:
  containers:
  - name: app
    image: nginx
  - name: sidecar
    image: fluentd
  volumes:
  - name: shared
    emptyDir: {}

# Init Container (wait for service)
spec:
  initContainers:
  - name: init
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do sleep 2; done']
  containers:
  - name: app
    image: myapp
```

### Probes
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5

readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 3

startupProbe:
  exec:
    command: ['cat', '/tmp/started']
  periodSeconds: 5
  failureThreshold: 30
```

### Jobs
```bash
# Job with completions
k create job myjob --image=busybox --dry-run=client -o yaml -- echo done > job.yaml
# Add: spec.completions: 5, spec.parallelism: 2

# CronJob
k create cj mycron --image=busybox --schedule="*/5 * * * *" -- echo hello
```

### Resource Quotas
```bash
k create quota myquota --hard=cpu=10,memory=20Gi,pods=10 -n dev
```

### Deployment Strategies
```bash
# Blue/Green: Create 2 deployments, switch service selector
k set selector svc myapp 'version=green'

# Canary: Scale replicas for traffic %
k scale deploy canary --replicas=3  # 30% if stable has 7
```

---

*Last Updated: January 2026*  
*CKAD Exam Version: 2025 (K8s 1.34)*  
*Post-CKA Optimization Guide*  
*Sensei Mode: Victory Lap* 🎉
