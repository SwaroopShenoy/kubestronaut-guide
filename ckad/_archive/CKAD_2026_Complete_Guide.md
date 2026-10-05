# CKAD 2026 Complete Crash Course
## Raw Knowledge for Application Developers

---

## 🎯 Multi-Container Patterns

**The CKAD differentiator. Master all 4 patterns.**

### Pattern 1: Sidecar

Helper container shares volumes with main container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-sidecar
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx  # App writes here
  
  - name: log-shipper  # Sidecar
    image: filebeat:latest
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx  # Sidecar reads here
      readOnly: true
    
  volumes:
  - name: shared-logs
    emptyDir: {}  # Shared between containers
```

**Real exam question**: "Ship nginx logs to ELK stack using sidecar"
- Add second container
- Share volume
- Configure log shipping tool

### Pattern 2: Init Container

Runs before main container, must complete successfully.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:latest
    command: 
    - sh
    - -c
    - |
      until nslookup postgres.default.svc.cluster.local; do
        echo "Waiting for DB..."
        sleep 2
      done
  
  - name: migrate-db
    image: myapp:v1
    command: ["./migrate.sh"]
  
  containers:
  - name: app
    image: myapp:v1
    ports:
    - containerPort: 8080
```

**Rules**:
- Init containers run sequentially, MUST succeed
- If one fails, pod never starts (restarts init)
- Only main container logs show in `k logs <pod>`
- Check init logs: `k logs <pod> -c init-container-name`

```bash
# Debug init container
k describe pod <pod>
# Look at "Init Containers:" section - shows status

k logs <pod> -c wait-for-db  # Init logs
k logs <pod> -c app          # Main logs
```

### Pattern 3: Adapter

Transforms/normalizes output from main container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-adapter
spec:
  containers:
  - name: app
    image: legacy-app:v1
    volumeMounts:
    - name: shared-data
      mountPath: /app/logs
    # App writes logs in CSV format to /app/logs/app.log
  
  - name: log-adapter
    image: log-normalizer:v1
    volumeMounts:
    - name: shared-data
      mountPath: /app/logs
    # Reads CSV, converts to JSON, outputs to stdout
    command: ["./normalize-logs.sh"]
  
  volumes:
  - name: shared-data
    emptyDir: {}
```

**Use case**: Old app outputs weird format → adapter normalizes for monitoring.

### Pattern 4: Ambassador

Acts as proxy to external service.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-ambassador
spec:
  containers:
  - name: app
    image: myapp:v1
    # Connects to localhost:6379 (looks local)
    # Actually proxied by ambassador
    env:
    - name: REDIS_HOST
      value: localhost
    - name: REDIS_PORT
      value: "6379"
  
  - name: redis-ambassador
    image: redis-proxy:v1
    ports:
    - containerPort: 6379
    # Proxies incoming localhost:6379 → actual-redis-cluster.external:6379
    env:
    - name: REDIS_TARGET
      value: actual-redis-service.example.com
```

**Why**: Decouple app from external service location. Change Redis host? Update only ambassador.

### Native Sidecar Pattern (v1.35)

Init container with `restartPolicy: Always` runs as sidecar:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-native-sidecar
spec:
  containers:
  - name: app
    image: myapp:v1
  
  initContainers:
  - name: monitoring-sidecar
    image: prometheus-exporter:v1
    restartPolicy: Always  # ← Key! Makes it restart like sidecar
    ports:
    - containerPort: 9090
```

---

## 🔍 Probes: Liveness, Readiness, Startup

**Probes determine if container is healthy.**

### Probe Types

**Liveness Probe**: Is container alive? If fails → restart
- Use for: Detect hung processes, deadlocks
- Too aggressive can cause flapping

**Readiness Probe**: Can container handle traffic? If fails → remove from service endpoints
- Use for: Detect dependency issues (DB not ready, cache warming)
- Less disruptive than liveness

**Startup Probe**: Has container started? Blocks liveness/readiness until passes
- Use for: Slow-starting apps (Java, heavy frameworks)
- Prevents restart loops during startup

### Probe Methods

**HTTP GET** (most common):
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: X-Custom-Header
      value: Awesome
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

**TCP Socket** (for non-HTTP services):
```yaml
livenessProbe:
  tcpSocket:
    port: 5432
  initialDelaySeconds: 5
  periodSeconds: 10
```

**Exec** (for files, scripts):
```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 10
```

### Probe Parameters

| Parameter | Meaning | Default |
|-----------|---------|---------|
| `initialDelaySeconds` | Wait before first probe | 0 |
| `periodSeconds` | Check every N seconds | 10 |
| `timeoutSeconds` | Timeout per probe | 1 |
| `failureThreshold` | Failures before action | 3 |
| `successThreshold` | Successes to mark healthy | 1 |

### Real Examples

**Web app with graceful shutdown**:
```yaml
spec:
  containers:
  - name: web
    image: django:latest
    ports:
    - containerPort: 8000
    
    startupProbe:
      httpGet:
        path: /startup
        port: 8000
      periodSeconds: 5
      failureThreshold: 30  # 150 seconds max startup
    
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8000
      initialDelaySeconds: 10
      periodSeconds: 10
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 8000
      initialDelaySeconds: 5
      periodSeconds: 5
    
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]  # Grace period
```

**Database with TCP probe**:
```yaml
spec:
  containers:
  - name: postgres
    image: postgres:15
    env:
    - name: POSTGRES_PASSWORD
      value: secret
    
    livenessProbe:
      exec:
        command:
        - /bin/sh
        - -c
        - pg_isready -U postgres
      initialDelaySeconds: 30
      periodSeconds: 10
    
    readinessProbe:
      tcpSocket:
        port: 5432
      initialDelaySeconds: 5
      periodSeconds: 5
```

**Slow-starting Java app**:
```yaml
spec:
  containers:
  - name: java-app
    image: myapp:java
    ports:
    - containerPort: 8080
    
    startupProbe:
      httpGet:
        path: /actuator/health
        port: 8080
      periodSeconds: 10
      failureThreshold: 60  # 10 minutes max startup
    
    # Only checked after startup succeeds
    livenessProbe:
      httpGet:
        path: /actuator/health
        port: 8080
      periodSeconds: 10
```

### Probe Debugging

```bash
# See probe status
k describe pod <pod>
# Look at "Conditions:" → "Ready", "Initialized"
# Look at "Events:" → ProbeWarning, ProbeError

# Check app logs
k logs <pod>

# Test probe manually
k exec <pod> -- curl http://localhost:8080/healthz
k exec <pod> -- nc -zv localhost 5432

# If probe keeps failing
# 1. Check endpoint exists
k exec <pod> -- curl http://localhost:8080/healthz

# 2. Check port is right
k port-forward <pod> 8080:8080
curl localhost:8080/healthz

# 3. Check timing
# Increase initialDelaySeconds, periodSeconds
k edit pod <pod>  # NOT recommended on exam! Use YAML template instead
```

---

## 📋 Jobs & CronJobs

### Job Basics

Runs task to completion, then stops (unlike Deployment which runs forever).

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculation
spec:
  completions: 3          # Run 3 times successfully
  parallelism: 2          # Run 2 in parallel (max)
  backoffLimit: 2         # Retry 2 times if fails
  activeDeadlineSeconds: 300  # Kill if takes > 5 min
  
  template:
    spec:
      containers:
      - name: pi
        image: perl:latest
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never  # Don't restart on failure
```

```bash
# Create job
k apply -f job.yaml

# Check progress
k get job
k describe job pi-calculation
k logs job/pi-calculation  # Logs from all pods

# Wait for completion
k wait --for=condition=complete job/pi-calculation --timeout=600s

# Clean up
k delete job pi-calculation
```

### CronJob

Job on a schedule.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup:v1
            command: ["/backup.sh"]
          restartPolicy: OnFailure
  
  successfulJobsHistoryLimit: 3    # Keep 3 successful jobs
  failedJobsHistoryLimit: 1        # Keep 1 failed job
```

**Cron schedule format**: `minute hour day month weekday`
- `0 2 * * *` = 2 AM every day
- `*/5 * * * *` = Every 5 minutes
- `0 0 * * 0` = Midnight every Sunday
- `0 9-17 * * 1-5` = 9 AM to 5 PM, Mon-Fri

```bash
# Create cronjob
k apply -f cronjob.yaml

# Check scheduled jobs
k get cronjob
k get jobs  # Shows triggered jobs

# Trigger manually
k create job backup-manual --from=cronjob/backup

# View CronJob schedule
k describe cronjob backup
```

### Job Troubleshooting

```bash
# Job pods not running
k get pods -l job-name=pi-calculation

# Check job events
k describe job pi-calculation

# If job failed
k describe pod <job-pod>
k logs <job-pod>

# Common: restartPolicy wrong
# Jobs MUST have restartPolicy: Never or OnFailure
# NOT Always (that's for Deployments)

# Check completions
k get job -o wide
# COMPLETIONS column shows: 1/3 (1 done, 3 needed)
```

---

## 📝 ConfigMaps & Secrets

### ConfigMap

Store configuration (non-sensitive).

```bash
# Create from literal
k create configmap app-config \
  --from-literal=DB_HOST=postgres \
  --from-literal=LOG_LEVEL=debug

# Create from file
k create configmap app-config --from-file=app.conf

# Verify
k get configmap
k describe configmap app-config
k get configmap app-config -o yaml
```

**Use in Pod**:
```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    
    # Method 1: As environment variables
    envFrom:
    - configMapRef:
        name: app-config
    
    # Method 2: Specific key as env var
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    
    # Method 3: As file volume
    volumeMounts:
    - name: config
      mountPath: /etc/config
  
  volumes:
  - name: config
    configMap:
      name: app-config
      # /etc/config/DB_HOST contains "postgres"
      # /etc/config/LOG_LEVEL contains "debug"
```

### Secret

Store sensitive data (passwords, tokens).

```bash
# Create from literal
k create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t

# Create from file
k create secret generic tls-secret \
  --from-file=tls.crt=/path/to/cert.crt \
  --from-file=tls.key=/path/to/cert.key

# View (base64 encoded)
k get secret db-secret -o yaml
# password: czMKcjN0  (base64)

# Decode
k get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

**Use in Pod**:
```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    
    # As env var
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    
    # As volume
    volumeMounts:
    - name: secret
      mountPath: /etc/secret
      readOnly: true
  
  volumes:
  - name: secret
    secret:
      secretName: db-secret
      # /etc/secret/username
      # /etc/secret/password
```

### ConfigMap Merging (2026 Exam Gotcha)

```yaml
# If you have multiple ConfigMaps and need to merge, or
# use specific keys from ConfigMap with volumes:

spec:
  containers:
  - name: app
    volumeMounts:
    - name: config
      mountPath: /etc/config
  
  volumes:
  - name: config
    configMap:
      name: app-config
      items:           # ← Only include specific keys
      - key: app.conf
        path: app.conf
      - key: logging.conf
        path: logging.conf
```

---

## 🌐 Services & NetworkPolicy

### Service Types

**ClusterIP** (default, internal only):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: api
```

**NodePort** (accessible on node IP):
```yaml
spec:
  type: NodePort
  ports:
  - port: 80              # Service port (inside cluster)
    targetPort: 8080      # Container port
    nodePort: 30080       # Node port (must be 30000-32767)
  selector:
    app: api
```

**LoadBalancer** (external IP):
```yaml
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: api
```

### Debugging Services

```bash
# Pod not in endpoints?
k get endpoints <svc-name>  # Should list pod IPs

# Service selector matches pod labels?
k describe svc <svc>        # Check Selector:
k get pods --show-labels    # Check Labels:

# Fix: labels must match exactly
k label pod <pod> app=api
k get endpoints <svc-name>  # Should populate

# Test connectivity
k exec <pod> -- curl http://service-name:80
```

### NetworkPolicy (v1.35 Exam Trap)

**CRITICAL**: Egress rules need DNS!

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

---

# Allow ingress from frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
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

# Allow egress to external service (DON'T FORGET DNS!)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
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
      port: 443      # HTTPS
    - protocol: UDP
      port: 53       # ← DNS! Without this, names won't resolve
```

```bash
# Test NetworkPolicy
k exec <pod> -- ping 8.8.8.8  # Should fail if egress blocked
k exec <pod> -- nslookup google.com  # Needs UDP 53
```

---

## 🔧 Deployment Strategies

### Rolling Update (default)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 1 extra pod during update
      maxUnavailable: 1  # 1 pod can be unavailable
  minReadySeconds: 10    # Wait 10s after pod ready before mark ready
```

### Blue/Green (manual)

```bash
# 1. Run "green" (new version) alongside "blue" (current)
k create deployment app-green --image=myapp:v2

# 2. Route traffic to green
k patch service app -p '{"spec":{"selector":{"version":"green"}}}'

# 3. If bad, rollback
k patch service app -p '{"spec":{"selector":{"version":"blue"}}}'

# 4. Delete green when stable
k delete deployment app-green
```

### Canary (manual)

```bash
# 1. Keep current deployment
k get deployment app  # Running v1, 10 replicas

# 2. Add canary pod (v2)
k create deployment app-canary --image=myapp:v2 --replicas=1

# 3. Both v1 and v2 are in service (selector matches both)
k get pods -l app=app  # 10 v1 + 1 v2

# 4. Monitor metrics

# 5. If good, scale up v2
k scale deployment app-canary --replicas=5

# 6. If bad, delete canary
k delete deployment app-canary
```

---

## 🐛 Debugging Applications

### Pod Logs

```bash
# Current logs
k logs <pod>

# Follow (tail -f)
k logs <pod> -f

# Previous pod (if crashed and restarted)
k logs <pod> --previous

# From specific container (multi-container pod)
k logs <pod> -c container-name

# All pods with label
k logs -l app=myapp
```

### Port Forward

```bash
# Localhost:8080 → Pod:8080
k port-forward <pod> 8080:8080

# Then in another terminal
curl localhost:8080

# Specific pod by name
k port-forward pod/my-pod 8080:8080
```

### Exec

```bash
# Shell access
k exec -it <pod> -- /bin/bash

# Or sh if no bash
k exec -it <pod> -- /bin/sh

# Run command
k exec <pod> -- ps aux
k exec <pod> -- env
k exec <pod> -- curl http://localhost:8080
```

### Describe

```bash
# Show all details + events
k describe pod <pod>

# Look for:
# - State: Running, CrashLoopBackOff, ImagePullBackOff
# - Ready: True/False
# - Conditions section
# - Events section (recent problems)
```

### Events

```bash
# Sort by time
k get events -n <ns> --sort-by='.lastTimestamp'

# Filter by pod
k get events -n <ns> --field-selector involvedObject.name=<pod>

# Filter by type
k get events -n <ns> --field-selector type=Warning
k get events -n <ns> --field-selector type=Normal
```

### Troubleshoot Common Pod States

**ImagePullBackOff**: Can't download image
```bash
k describe pod <pod> | grep -A 5 "Events:"
# Usually: wrong image name, tag doesn't exist, private repo

# Check image name
k describe pod <pod> | grep image:
# Correct it in deployment/pod yaml
```

**CrashLoopBackOff**: Container keeps crashing
```bash
# Check logs
k logs <pod> --previous

# Check command/args are correct
k get pod <pod> -o yaml | grep -A 5 "command:\|args:"

# Increase graceful termination
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
```

**Pending**: Pod can't be scheduled
```bash
k describe pod <pod> | grep "Events:" -A 10
# Usually: resource requests too high, no nodes available

# Check resource requests
k get pod <pod> -o yaml | grep -A 3 "resources:"

# Check node resources
k describe nodes
k top nodes
```

---

## 🔐 RBAC for Application Devs

### Create ServiceAccount

```bash
k create serviceaccount app-user

# Verify
k get sa
k describe sa app-user
```

### Create Role

```bash
k create role app-role \
  --verb=get,list \
  --resource=pods,services

# Add more permissions
k edit role app-role
```

### Bind Role to ServiceAccount

```bash
k create rolebinding app-user-binding \
  --role=app-role \
  --serviceaccount=default:app-user

# Verify
k get rolebindings
k describe rolebinding app-user-binding
```

### Use in Pod

```yaml
spec:
  serviceAccountName: app-user  # Use the ServiceAccount
  
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      readOnly: true
  
  # Token automatically mounted
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          expirationSeconds: 3600
```

---

## 🎯 Real Exam Questions

**Multi-container sidecar logging**:
- "Ship nginx logs using Filebeat sidecar"
- Create second container, share emptyDir volume, mount log dirs

**Probe troubleshooting**:
- "Pod keeps failing readiness probe"
- `k logs <pod>`, check endpoint, fix app or probe timing

**Service connectivity**:
- "Pod can't reach service"
- `k get endpoints`, check selector labels, test with curl

**NetworkPolicy egress**:
- "Pod can't reach external API"
- Add DNS rule: protocol: UDP, port: 53

**ConfigMap update not reflected**:
- ConfigMaps don't auto-reload
- Pod must restart to see changes: `k rollout restart deployment`

**CronJob not running**:
- Check schedule syntax: `k describe cronjob`
- Check job pods are being created: `k get jobs`

