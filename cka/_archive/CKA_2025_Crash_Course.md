# CKA 2026 Comprehensive Crash Course
## End-to-End K8s Administrator Mastery

**Kubernetes Version**: v1.35  
**Format**: Performance-based, 2 hours, 15-20 tasks, 66% pass  
**Last Updated**: July 2026

This is NOT a strategy guide. This is raw knowledge you need to know, broken down by real exam topics.

---

## 🎯 Your Strategic Advantage

**What you already have:**
- ✅ K8s fundamentals (KCNA)
- ✅ Security concepts (KCSA)  
- ✅ 5 years DevOps experience
- ✅ Conceptual understanding

**Your ONLY gap:**
- 🔥 **Hands-on speed** (kubectl muscle memory)
- 🔥 **Vim proficiency** (YAML editing)
- 🔥 **Doc navigation** (find answers fast)
- 🔥 **New topics** (Helm, Kustomize, Gateway API, CRDs)

**Your plan is SMART**: CKA + CKAD same day/week
- ~70% content overlap
- Same exam format (performance-based, 2 hours, docs allowed)
- CKA harder, do it first while fresh
- CKAD faster after CKA (more app-focused, less cluster admin)

---

## 📋 Exam Overview - CKA 2025

**Format**: Performance-based (terminal tasks, NO multiple choice)  
**Duration**: 2 hours  
**Questions**: 15-20 tasks  
**Passing Score**: 66% (aim for 80%+)  
**Cost**: $445 (includes 1 free retake)  
**Validity**: 2 years  
**Environment**: PSI remote browser, 6 different clusters, context switching

**Allowed Resources**:
- kubernetes.io/docs
- kubernetes.io/blog  
- github.com/kubernetes (limited)
- ONE extra browser tab for docs
- NO Google, Stack Overflow, notes, etc.

**KEY DIFFERENCES from MCQ exams:**
- You actually DO tasks, not just answer questions
- Speed matters (2 hours flies by)
- Partial credit possible
- Can skip and return
- Must verify your work

---

## 🎯 CKA 2025 Domains & Weights (Updated Feb 18, 2025)

| Domain | Weight | Questions | Your Focus |
|--------|--------|-----------|------------|
| **Troubleshooting** | 30% | ~5-6 | HIGHEST - Practice debugging |
| **Cluster Architecture** | 25% | ~4-5 | HIGH - kubeadm, RBAC, upgrades |
| **Workloads & Scheduling** | 15% | ~2-3 | MEDIUM - Deployments, scaling |
| **Services & Networking** | 20% | ~3-4 | HIGH - Services, NetworkPolicy, Gateway API |
| **Storage** | 10% | ~1-2 | MEDIUM - PV, PVC, StorageClass |

---

## 🔧 Domain 1: Storage (10%)

### Core Concepts

**PersistentVolume (PV)**: Cluster resource, provisioned by admin  
**PersistentVolumeClaim (PVC)**: User request for storage  
**StorageClass**: Dynamic provisioning template

### Access Modes
- **ReadWriteOnce (RWO)**: Single node, read-write
- **ReadOnlyMany (ROX)**: Multiple nodes, read-only
- **ReadWriteMany (RWX)**: Multiple nodes, read-write

### Reclaim Policies
- **Retain**: Keep data after PVC deleted (manual cleanup)
- **Delete**: Delete PV and backing storage when PVC deleted
- **Recycle**: Deprecated (use Delete + dynamic provisioning)

### NEW 2025: Dynamic Volume Provisioning (Emphasis)

**StorageClass with provisioner**:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
```

**Key fields**:
- `provisioner`: Cloud provider or CSI driver
- `volumeBindingMode`: 
  - `Immediate`: Bind PV when PVC created
  - `WaitForFirstConsumer`: Bind when Pod scheduled (better for topology)
- `allowVolumeExpansion`: Allow PVC resize

### Hands-on Commands

```bash
# Create PVC (dynamic provisioning)
k create -f pvc.yaml

# Check PVC status
k get pvc
k describe pvc my-pvc

# Check which PV bound
k get pv

# Create Pod using PVC
k run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
# Edit to add volume mount
k apply -f pod.yaml

# Verify mount
k exec nginx -- df -h
k exec nginx -- ls /data

# Resize PVC (if allowVolumeExpansion: true)
k edit pvc my-pvc
# Change storage size, save

# Delete (reclaim policy applies)
k delete pvc my-pvc
```

### Speed Techniques

**Template PVC fast**:
```bash
cat <<EOF | k apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
EOF
```

**Or use docs**:
- Search "persistent volumes"
- Copy PVC example
- Modify quickly

### Common Exam Scenarios

1. **Create PVC with specific StorageClass**
2. **Attach PVC to Pod**
3. **Resize PVC**
4. **Troubleshoot PVC stuck in Pending** (no matching PV or StorageClass)
5. **Create StorageClass for dynamic provisioning**

### Pro Tips

- **PVC stays Pending**: Check StorageClass exists, provisioner working
- **Pod can't mount**: Check PVC bound, access mode compatible
- **Volume not expanding**: Check `allowVolumeExpansion: true` in StorageClass
- **Data persists**: Use `Retain` reclaim policy

---

## ⚙️ Domain 2: Workloads & Scheduling (15%)

### NEW 2025: Helm & Kustomize (Major Addition!)

#### **Helm Basics**

**What is Helm?**
- Package manager for K8s
- Chart = collection of K8s manifests
- Templating with values
- Release management (install, upgrade, rollback)

**Common Commands**:
```bash
# Add repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search charts
helm search repo nginx

# Install chart
helm install my-release bitnami/nginx

# List releases
helm list

# Get values
helm get values my-release

# Upgrade with custom values
helm upgrade my-release bitnami/nginx --set replicaCount=3

# Rollback
helm rollback my-release 1

# Uninstall
helm uninstall my-release

# Create chart
helm create mychart

# Lint chart
helm lint mychart

# Template (dry-run)
helm template mychart

# Install with values file
helm install my-release mychart -f values.yaml
```

**Exam scenarios**:
- Install chart from repo
- Upgrade release with custom values
- Rollback to previous version
- Debug failed release

#### **Kustomize Basics**

**What is Kustomize?**
- K8s-native config management
- Overlay-based (base + overlays)
- No templates, pure YAML
- Built into kubectl (`kubectl apply -k`)

**Directory structure**:
```
base/
  deployment.yaml
  service.yaml
  kustomization.yaml
overlays/
  dev/
    kustomization.yaml
    patch.yaml
  prod/
    kustomization.yaml
    patch.yaml
```

**Common patterns**:
```bash
# Apply base
kubectl apply -k base/

# Apply overlay
kubectl apply -k overlays/dev/

# View generated YAML
kubectl kustomize overlays/prod/

# Diff before apply
kubectl diff -k overlays/dev/
```

**Example kustomization.yaml**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: myapp
commonLabels:
  app: myapp
resources:
  - deployment.yaml
  - service.yaml
replicas:
  - name: myapp
    count: 3
images:
  - name: nginx
    newTag: 1.21
```

**Exam scenarios**:
- Apply kustomization
- Modify replicas/image via kustomization
- Use overlays for different environments

---

## 🔥 NEW in v1.35 (July 2026 Changes)

### Init Containers as Sidecars (TESTED ON EXAM)

**What changed**: Init containers with `restartPolicy: Always` now run as **sidecars** - they restart if they fail.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  - name: app
    image: app:v1
  initContainers:
  - name: logging-sidecar
    image: filebeat:latest
    restartPolicy: Always  # ⚠️ This is the change!
    volumeMounts:
    - name: logs
      mountPath: /var/log
  volumes:
  - name: logs
    emptyDir: {}
```

**Real exam question pattern**: "Setup a logging sidecar that collects app logs and doesn't fail the Pod"

### Gateway API (More Emphasis)

**Why it matters**: Gradually replacing Ingress. You WILL see exam questions.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: my-gateway-class
spec:
  controllerName: io.kubernetes/gce

---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: my-gateway-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80

---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /app
    backendRefs:
    - name: app-service
      port: 8080
```

**Know for exam**:
- GatewayClass defines controller
- Gateway listener config
- HTTPRoute = path routing
- Much more flexible than Ingress

### cgroup v2 (Infrastructure Change)

**Impact**: All nodes now use cgroup v2 (instead of v1)

**What you need to know**:
- Affects resource limits and monitoring
- CPU/memory limits work as you expect, just internally different
- For exam: Treat it same as v1 from a user perspective
- Might see a question on "why are my kubelet metrics not showing" → answer: cgroup v2 query syntax differs

```bash
# Check which cgroup version node uses
k get node -o yaml | grep cgroupDriver
# Should see: "systemd"

# Or on node:
mount | grep cgroup2
```

### CEL-based Admission (Optional for exam)

**What is it**: Validation rules without webhooks

**Don't stress this**, but know it exists:
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.example.com
spec:
  # ...
  validationRules:
  - rule: "self.metadata.name.startsWith('widget-')"
    message: "Widget name must start with 'widget-'"
```

---

### Deployments & Scaling

**Create Deployment**:
```bash
# Imperative
k create deploy nginx --image=nginx --replicas=3

# With resource limits
k create deploy nginx --image=nginx --dry-run=client -o yaml > deploy.yaml
# Edit to add resources
k apply -f deploy.yaml
```

**Scaling**:
```bash
# Manual scale
k scale deploy nginx --replicas=5

# Autoscaling (HPA)
k autoscale deploy nginx --min=2 --max=10 --cpu-percent=80

# Check HPA
k get hpa
```

### StatefulSets

**When to use**: Databases, stateful apps needing stable identity

**Key differences from Deployment**:
- Ordered creation/deletion (pod-0, pod-1, pod-2)
- Stable network identity (pod-0.service.namespace.svc.cluster.local)
- Stable storage (PVC per Pod)

```bash
# Create from docs (no imperative command)
# Search "statefulset" in docs, copy example

# Scale StatefulSet
k scale sts mysql --replicas=5

# Delete (cascading=false keeps Pods)
k delete sts mysql --cascade=orphan
```

### DaemonSets

**When to use**: One Pod per node (logging, monitoring, storage daemons)

```bash
# Create from docs
# Search "daemonset", copy example

# Update image
k set image ds/my-daemon my-container=nginx:1.21
```

### Jobs & CronJobs

**Job**: Run to completion
```bash
# Create job
k create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)'

# Check completion
k get jobs

# See logs
k logs job/pi
```

**CronJob**: Scheduled jobs
```bash
# Create cronjob
k create cj backup --image=backup:v1 --schedule="0 2 * * *" -- /backup.sh

# Check cronjobs
k get cj

# Trigger manually
k create job --from=cronjob/backup backup-manual
```

### Scheduling

**Node Selector** (simple):
```yaml
spec:
  nodeSelector:
    disktype: ssd
```

**Node Affinity** (complex):
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
```

**Taints & Tolerations**:
```bash
# Taint node
k taint node node1 key=value:NoSchedule

# Pod toleration
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"

# Remove taint
k taint node node1 key:NoSchedule-
```

**Resource Limits**:
```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

### Speed Techniques

```bash
# Deployment with resources (fast)
k create deploy nginx --image=nginx --dry-run=client -o yaml | \
  k set resources -f - --requests=cpu=200m,memory=256Mi --limits=cpu=500m,memory=512Mi --local -o yaml | \
  k apply -f -

# Or just edit after creation (often faster)
k create deploy nginx --image=nginx
k edit deploy nginx
# Add resources, save
```

---

## 🌐 Domain 3: Services & Networking (20%)

### NEW 2025: Gateway API (Major Addition!)

**What is Gateway API?**
- Next-gen Ingress (more expressive, role-oriented)
- Graduated to v1 in K8s 1.32
- Separates concerns: Infrastructure vs Route config

**Core Resources**:
- **GatewayClass**: Defines type of gateway (like IngressClass)
- **Gateway**: Instance of GatewayClass, listens on port
- **HTTPRoute**: Routes HTTP traffic to services
- **TCPRoute**: L4 TCP routing
- **TLSRoute**: TLS SNI routing

**Example**:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 8080
```

**vs Ingress**:
- Ingress: Simpler, one resource
- Gateway API: More powerful, multiple resources, role separation

**Exam tip**: Know how to create HTTPRoute, reference Gateway

### Services

**ClusterIP** (default, internal):
```bash
k expose deploy nginx --port=80 --target-port=8080
```

**NodePort** (external, static port):
```bash
k expose deploy nginx --type=NodePort --port=80
```

**LoadBalancer** (cloud LB):
```bash
k expose deploy nginx --type=LoadBalancer --port=80
```

**ExternalName** (DNS CNAME):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-db
spec:
  type: ExternalName
  externalName: db.example.com
```

**Headless Service** (no ClusterIP, for StatefulSet):
```yaml
spec:
  clusterIP: None
```

### NetworkPolicy

**Default Deny All**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**Allow specific**:
```yaml
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
```

### CoreDNS

**DNS for K8s services**:
- Service: `<service>.<namespace>.svc.cluster.local`
- Pod: `<pod-ip-dashes>.<namespace>.pod.cluster.local`

**Troubleshoot DNS**:
```bash
# Test DNS from Pod
k run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default

# Check CoreDNS pods
k get pods -n kube-system -l k8s-app=kube-dns

# CoreDNS logs
k logs -n kube-system -l k8s-app=kube-dns
```

### NEW 2025: CNI, CSI, CRI (Extension Interfaces)

**CNI (Container Network Interface)**:
- Plugins: Calico, Flannel, Cilium, Weave
- Provides Pod networking
- Check: `/etc/cni/net.d/` on nodes

**CSI (Container Storage Interface)**:
- Storage plugins
- Allows external storage providers
- Check StorageClass provisioner

**CRI (Container Runtime Interface)**:
- Runtime: containerd, CRI-O
- Check: `crictl` commands
```bash
# On node
crictl ps
crictl images
crictl logs <container-id>
```

### Speed Techniques

```bash
# Service expose (fastest)
k expose deploy nginx --port=80 --target-port=8080

# Or imperative
k create svc clusterip nginx --tcp=80:8080

# NetworkPolicy from docs (fastest)
# Search "network policy", copy, modify

# Gateway API from docs
# Search "gateway api", copy example
```

---

## 🔨 Domain 4: Cluster Architecture, Installation & Configuration (25%)

### Cluster Components (Review)

**Control Plane**:
- kube-apiserver
- etcd
- kube-scheduler
- kube-controller-manager
- cloud-controller-manager (if cloud)

**Node**:
- kubelet
- kube-proxy
- Container runtime (containerd, CRI-O)

### kubeadm Cluster Setup (IMPORTANT)

**Initialize control plane**:
```bash
# On control plane node
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# Setup kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install CNI (Calico example)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

**Join worker node**:
```bash
# On worker node (use token from init output)
sudo kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

# If token expired, generate new
kubeadm token create --print-join-command
```

**Cluster upgrade** (LESS emphasis in 2025, but know basics):
```bash
# On control plane
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.32.0

# Upgrade kubelet
sudo apt-mark unhold kubelet kubectl
sudo apt-get update && sudo apt-get install -y kubelet=1.32.0-00 kubectl=1.32.0-00
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Repeat for worker nodes (use kubeadm upgrade node)
```

### etcd Backup & Restore (LESS emphasis, but good to know)

**Backup**:
```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Restore**:
```bash
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore
```

### RBAC (Critical!)

**Create User with Certificate**:
```bash
# Generate key
openssl genrsa -out user.key 2048

# Create CSR
openssl req -new -key user.key -out user.csr -subj "/CN=user/O=developers"

# Sign with K8s CA
sudo openssl x509 -req -in user.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out user.crt -days 365

# Create kubeconfig
kubectl config set-credentials user --client-certificate=user.crt --client-key=user.key
kubectl config set-context user@kubernetes --cluster=kubernetes --user=user
```

**Create Role & RoleBinding**:
```bash
# Role
k create role developer --verb=create,get,list,update,delete --resource=pods

# RoleBinding
k create rolebinding developer-binding --role=developer --user=user

# Test
k auth can-i get pods --as user
```

**ClusterRole & ClusterRoleBinding** (cluster-wide):
```bash
k create clusterrole pod-reader --verb=get,list,watch --resource=pods
k create clusterrolebinding pod-reader-binding --clusterrole=pod-reader --user=user
```

### ServiceAccounts

**Create SA**:
```bash
k create sa my-sa

# Use in Pod
k run nginx --image=nginx --serviceaccount=my-sa

# Or in YAML
spec:
  serviceAccountName: my-sa
```

**Bind Role to SA**:
```bash
k create rolebinding sa-binding --role=developer --serviceaccount=default:my-sa
```

### NEW 2025: CRDs & Operators (Major Addition!)

**CustomResourceDefinition (CRD)**:
- Extend K8s API with custom resources
- Define schema, validation

**Example CRD**:
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

**Use custom resource**:
```yaml
apiVersion: stable.example.com/v1
kind: CronTab
metadata:
  name: my-cron
spec:
  cronSpec: "*/5 * * * *"
  image: my-cron-image
  replicas: 3
```

```bash
# Apply CRD
k apply -f crd.yaml

# Create custom resource
k apply -f crontab.yaml

# List
k get crontabs
k get ct  # short name
```

**Operators**:
- Software that uses CRDs
- Automates management of complex apps
- Example: Prometheus Operator, MySQL Operator

**Exam tip**: Know how to apply CRD, create custom resource, list them

### Speed Techniques

```bash
# RBAC fast
k create role dev --verb=get,list --resource=pods
k create rolebinding dev-bind --role=dev --user=john

# ServiceAccount
k create sa mysa
k set sa deploy nginx mysa
```

---

## 🐛 Domain 5: Troubleshooting (30% - LARGEST!)

**This is the MOST IMPORTANT domain. Master this.**

> ⚠️ **2026 UPDATE**: Troubleshooting is SIGNIFICANTLY HARDER than 2025 and mock exams. Real candidates report complex scenarios, not straightforward debugging. Spend 1/3 of your study time here.

### Troubleshooting Methodology

1. **Identify symptoms**: What's broken?
2. **Gather information**: Logs, events, describe
3. **Isolate cause**: Control plane? Node? Pod? Network?
4. **Fix**: Apply solution
5. **Verify**: Confirm fixed

### NEW 2025: Expanded Network Troubleshooting

**Common network issues**:
- Pod can't reach service
- Service not accessible
- DNS not resolving
- External connectivity broken
- NetworkPolicy blocking traffic

**Debug steps**:
```bash
# 1. Check Pod network
k exec mypod -- ping 8.8.8.8
k exec mypod -- nslookup kubernetes.default

# 2. Check Service endpoints
k get svc
k get endpoints my-service

# 3. Check DNS
k exec mypod -- nslookup my-service.default.svc.cluster.local

# 4. Check CoreDNS
k get pods -n kube-system -l k8s-app=kube-dns
k logs -n kube-system -l k8s-app=kube-dns

# 5. Check NetworkPolicy
k get netpol
k describe netpol my-policy

# 6. Check kube-proxy
k get pods -n kube-system -l k8s-app=kube-proxy
k logs -n kube-system -l k8s-app=kube-proxy

# 7. Check CNI
# On node
ls /etc/cni/net.d/
cat /etc/cni/net.d/*
```

### Cluster Component Troubleshooting

**Control plane issues**:
```bash
# Check component status
k get componentstatuses  # Deprecated but may still work
k get pods -n kube-system

# API server
k logs -n kube-system kube-apiserver-<node>

# Scheduler
k logs -n kube-system kube-scheduler-<node>

# Controller manager
k logs -n kube-system kube-controller-manager-<node>

# etcd
k logs -n kube-system etcd-<node>

# On control plane node
sudo journalctl -u kubelet
sudo systemctl status kubelet
```

**Node issues**:
```bash
# Check node status
k get nodes
k describe node worker1

# On node
sudo systemctl status kubelet
sudo journalctl -u kubelet -f

# Check disk pressure
df -h

# Check memory
free -h

# Check container runtime
sudo systemctl status containerd
crictl ps
```

### Pod Troubleshooting

**Pod states**:
- Pending: Not scheduled (resources, taints, etc.)
- Running: Executing
- Failed: Terminated with error
- CrashLoopBackOff: Keeps crashing
- ImagePullBackOff: Can't pull image
- Unknown: Node lost contact

**Debug commands**:
```bash
# Check Pod status
k get pod mypod
k describe pod mypod  # Check Events section!

# Check logs
k logs mypod
k logs mypod -c container-name  # Multi-container
k logs mypod --previous  # Previous crash

# Exec into Pod
k exec -it mypod -- /bin/sh

# Check resource usage
k top pod mypod

# Check if Pod can run on node
k get pod mypod -o wide  # See node
k describe node <node>  # Check resources

# Check events
k get events --sort-by='.lastTimestamp'
```

**Common fixes**:

**ImagePullBackOff**:
- Check image name/tag
- Check imagePullSecrets
- Check registry connectivity

**CrashLoopBackOff**:
- Check logs (`k logs pod --previous`)
- Check liveness/readiness probes
- Check application config (env vars, ConfigMap, Secret)

**Pending**:
- Check resources (requests too high)
- Check node taints/tolerations
- Check PVC (if using, must be bound)
- Check scheduling constraints (nodeSelector, affinity)

**OOMKilled** (Out of Memory):
- Increase memory limits
- Fix memory leak in app

### Service Troubleshooting

```bash
# Check service
k get svc my-service
k describe svc my-service

# Check endpoints (should match Pod IPs)
k get endpoints my-service

# If no endpoints:
# - Check selector matches Pod labels
# - Check Pods are Running and Ready
# - Check Pod port matches targetPort

# Test from another Pod
k run test --image=busybox -it --rm -- wget -O- http://my-service:80
```

### Application Troubleshooting

**Logs & Events**:
```bash
# Application logs
k logs deploy/myapp
k logs deploy/myapp --all-containers=true
k logs -l app=myapp --tail=100

# Events
k get events -n default --sort-by='.lastTimestamp'
```

**Resource usage**:
```bash
# Top pods
k top pods
k top pods --sort-by=memory
k top pods --sort-by=cpu

# Top nodes
k top nodes
```

**HPA not scaling**:
```bash
# Check HPA
k get hpa
k describe hpa my-hpa

# Check metrics-server
k get pods -n kube-system -l k8s-app=metrics-server
k top nodes  # If this fails, metrics-server broken
```

### Worker Node Troubleshooting

**Node NotReady**:
```bash
# Check node
k describe node worker1

# On node
sudo systemctl status kubelet
sudo journalctl -u kubelet

# Check disk space
df -h

# Check container runtime
sudo systemctl status containerd
crictl info

# Restart kubelet
sudo systemctl restart kubelet
```

**Kubelet issues**:
```bash
# On node
sudo systemctl status kubelet
sudo journalctl -u kubelet -f

# Check kubelet config
cat /var/lib/kubelet/config.yaml

# Check certificates
ls /var/lib/kubelet/pki/
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -text -noout
```

### Exam Troubleshooting Patterns

**Pattern 1: Pod won't start**
1. `k get pod` → Check state
2. `k describe pod` → Check Events
3. `k logs pod` → Check application logs
4. Fix: Image, config, resources, etc.

**Pattern 2: Service not accessible**
1. `k get svc` → Check service exists
2. `k get endpoints` → Check endpoints populated
3. If empty: Check Pod selector, Pod status
4. Test: `k run test --image=busybox -- wget http://svc`

**Pattern 3: Node NotReady**
1. `k describe node` → Check conditions
2. SSH to node
3. `sudo systemctl status kubelet`
4. `sudo journalctl -u kubelet`
5. Fix: Disk space, kubelet config, certificates

**Pattern 4: DNS not working**
1. Test: `k exec pod -- nslookup kubernetes.default`
2. Check CoreDNS: `k get pods -n kube-system -l k8s-app=kube-dns`
3. Check logs: `k logs -n kube-system -l k8s-app=kube-dns`

### Speed Techniques

```bash
# Quick aliases (set these up in exam)
alias k=kubectl
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias kx='kubectl exec -it'

# Fast event checking
k get events --sort-by='.lastTimestamp' | tail -20

# Fast log checking
k logs deploy/myapp --tail=50

# Fast describe (events at bottom)
k describe pod mypod | tail -20
```

---

## 🚀 Speed Optimization Strategies

### vim Mastery (CRITICAL)

**Essential vim commands**:
```vim
# Navigation
gg          # Top of file
G           # Bottom of file
10G         # Line 10
/search     # Search forward
n           # Next match
N           # Previous match

# Editing
i           # Insert mode
Esc         # Normal mode
dd          # Delete line
5dd         # Delete 5 lines
yy          # Copy line
p           # Paste below
P           # Paste above
u           # Undo
Ctrl+r      # Redo

# Find/Replace
:%s/old/new/g    # Replace all
:%s/old/new/gc   # Replace with confirmation

# Copy/Paste blocks
V           # Visual line mode
5j          # Select 5 lines down
y           # Copy
p           # Paste

# Indentation
>>          # Indent right
<<          # Indent left
5>>         # Indent 5 lines

# Save/Quit
:w          # Save
:q          # Quit
:wq         # Save and quit
:q!         # Quit without saving
ZZ          # Save and quit (fast)
```

**YAML editing in vim**:
```vim
# Set in exam
:set nu                 # Line numbers
:set expandtab          # Spaces not tabs
:set tabstop=2          # 2 spaces per tab
:set shiftwidth=2       # Indent 2 spaces

# Or all at once
:set nu et ts=2 sw=2
```

**Practice drill**: Edit 10 YAML files, add sections, delete lines, indent blocks. Get to <30 sec per file.

### kubectl Speed Tricks

**Aliases** (set these up immediately in exam):
```bash
alias k=kubectl
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'
alias kdp='kubectl describe pod'
alias kds='kubectl describe svc'
alias kl='kubectl logs'
alias kx='kubectl exec -it'

# Verify alias works
complete -o default -F __start_kubectl k
```

**Autocomplete** (should already be enabled):
```bash
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
```

**Short names**:
```bash
po        # pods
svc       # services
deploy    # deployments
rs        # replicasets
ds        # daemonsets
sts       # statefulsets
cm        # configmaps
ns        # namespaces
pv        # persistentvolumes
pvc       # persistentvolumeclaims
sa        # serviceaccounts
netpol    # networkpolicies
```

**Dry-run templates**:
```bash
# Generate YAML without creating
k create deploy nginx --image=nginx --dry-run=client -o yaml

# Pipe to file
k create deploy nginx --image=nginx --dry-run=client -o yaml > deploy.yaml

# Apply from stdin
cat <<EOF | k apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```

**Imperative commands** (learn these cold):
```bash
# Pods
k run nginx --image=nginx
k run nginx --image=nginx --port=80
k run nginx --image=nginx --env="KEY=VALUE"
k run nginx --image=nginx --labels="app=nginx,tier=frontend"
k run nginx --image=nginx --command -- sleep 3600

# Deployments
k create deploy nginx --image=nginx --replicas=3

# Services
k expose deploy nginx --port=80 --target-port=8080
k expose pod nginx --port=80 --type=NodePort
k create svc clusterip nginx --tcp=80:8080

# ConfigMaps
k create cm myconfig --from-literal=key=value
k create cm myconfig --from-file=config.txt

# Secrets
k create secret generic mysecret --from-literal=password=secretpass
k create secret docker-registry regcred --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD

# Jobs
k create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)'

# CronJobs
k create cj backup --image=backup:v1 --schedule="0 2 * * *" -- /backup.sh

# ServiceAccounts
k create sa mysa

# Roles
k create role dev --verb=get,list --resource=pods
k create clusterrole dev --verb=get,list --resource=pods

# RoleBindings
k create rolebinding dev-bind --role=dev --user=john
k create clusterrolebinding dev-bind --clusterrole=dev --user=john

# Namespaces
k create ns dev
```

**Quick edits**:
```bash
# Edit resource in place
k edit deploy nginx

# Set image
k set image deploy nginx nginx=nginx:1.21

# Set resources
k set resources deploy nginx --requests=cpu=200m,memory=256Mi --limits=cpu=500m,memory=512Mi

# Set env
k set env deploy nginx KEY=VALUE

# Set serviceaccount
k set sa deploy nginx mysa
```

### Doc Navigation Speed

**Critical pages to bookmark mentally**:
1. kubectl Cheat Sheet
2. Workloads (Pods, Deployments, StatefulSets)
3. Services & Networking
4. Storage (PV, PVC, StorageClass)
5. RBAC
6. NetworkPolicy
7. Gateway API
8. Helm
9. CRDs

**Search strategy**:
1. Use search bar (top right)
2. Keywords: "persistent volume", "network policy", "rbac"
3. Copy example YAML
4. Modify quickly
5. Apply

**Practice**: Find example YAML for each common resource in <60 seconds.

### Time Management

**2 hours = 120 minutes for 15-20 questions**
- ~6-8 minutes per question average
- Easy questions (1-2 min): Deployments, Services, Pods
- Medium questions (5-10 min): RBAC, NetworkPolicy, Troubleshooting
- Hard questions (10-15 min): Cluster issues, complex debugging

**Strategy**:
1. Read all questions quickly (5 min)
2. Do easy ones first (quick wins, build confidence)
3. Mark hard ones for later
4. Return to medium/hard
5. Leave 10-15 min for review

**Flags in terminal**:
```bash
# Mark question for review (built into exam)
# Just click flag icon on question
```

**Verify your work**:
```bash
# Always check
k get <resource>
k describe <resource>
k logs <pod>  # If applicable

# Don't assume it worked!
```

---

## 📚 Learning Plan - 3-4 Weeks

### Week 1: Foundations + New Topics
**Goals**: Refresh concepts, learn Helm/Kustomize/Gateway API/CRDs

**Day 1-2: Storage & Workloads**
- PV, PVC, StorageClass (hands-on)
- Deployments, StatefulSets, DaemonSets
- **NEW**: Helm basics (install, upgrade, rollback)

**Day 3-4: Networking**
- Services (all types)
- **NEW**: Gateway API (create Gateway, HTTPRoute)
- NetworkPolicy

**Day 5-7: Cluster Architecture**
- kubeadm cluster setup (practice on VMs)
- RBAC (create roles, bind to users/SA)
- **NEW**: CRDs (create, apply, list custom resources)
- **NEW**: CSI, CNI, CRI concepts

**Practice**:
- 2 hours hands-on daily
- KodeKloud labs (if subscribed)
- Local Minikube/Kind cluster

### Week 2: Troubleshooting Deep Dive
**Goals**: Master debugging (30% of exam!)

**Day 1-2: Pod Troubleshooting**
- All Pod states (Pending, CrashLoop, ImagePull, etc.)
- Logs, describe, exec
- Resource issues

**Day 3-4: Network Troubleshooting**
- Service not accessible
- DNS issues
- NetworkPolicy blocking
- CoreDNS debugging

**Day 5-7: Cluster/Node Issues**
- Control plane component failures
- Node NotReady
- kubelet issues
- etcd problems

**Practice**:
- Break things intentionally
- Fix them
- Time yourself
- Repeat until fast (<5 min per scenario)

### Week 3: Speed Building
**Goals**: Get FAST with kubectl and vim

**Day 1-3: kubectl Drills**
- Imperative commands (all)
- Generate YAML templates
- Quick edits
- Context switching
- Time: <30 sec per task

**Day 4-5: vim Drills**
- Edit YAML files
- Add sections
- Indent blocks
- Find/replace
- Time: <30 sec per file

**Day 6-7: Doc Navigation**
- Find examples fast
- Copy, modify, apply
- All common resources
- Time: <60 sec to find and apply

**Practice**:
- Timed drills (use timer!)
- Mock exams (killer.sh simulator!)
- Repeat slow tasks until fast

### Week 4: Mock Exams & Final Prep
**Goals**: Exam simulation, identify gaps

**Day 1-2: killer.sh Session 1**
- Full 2-hour exam
- Note time per question
- Identify slow areas

**Day 3-4: Gap Filling**
- Focus on weak topics from mock
- More practice on slow tasks
- Review new topics (Helm, Gateway API, CRDs)

**Day 5: killer.sh Session 2**
- Full 2-hour exam
- Aim for 80%+
- Verify all answers

**Day 6: Light Review**
- Cheat sheet creation
- Alias setup practice
- vim config practice

**Day 7: REST**
- Don't study
- Relax
- Early sleep

---

## 🎯 Exam Day Checklist

### 30 Minutes Before

✅ Bathroom  
✅ Water bottle (allowed)  
✅ Clear desk (only allowed items)  
✅ ID ready  
✅ Room lighting good  
✅ Test webcam/mic  
✅ Close all apps/tabs  

### First 5 Minutes in Exam

```bash
# 1. Set aliases
alias k=kubectl
complete -o default -F __start_kubectl k

# 2. Set vim defaults
echo "set nu et ts=2 sw=2" >> ~/.vimrc

# 3. Verify k8s contexts
k config get-contexts

# 4. Test kubectl
k get nodes
```

### During Exam

✅ **Read question fully** (don't assume)  
✅ **Note context/namespace** (switch if needed)  
✅ **Use imperative when possible**  
✅ **Verify your work** (always check)  
✅ **Skip hard questions** (come back later)  
✅ **Leave 10 min for review**  
✅ **Don't panic** (you got this!)

### Question Format

**Example**:
```
Context: k8s-cluster1
Namespace: production
Weight: 7%

Task: Create a deployment named 'webapp' with 3 replicas using image 'nginx:1.21'.
The deployment should have resource requests of 200m CPU and 256Mi memory.
Expose it as a ClusterIP service on port 80.

Verify: kubectl get deploy webapp -n production
```

**Your approach**:
1. Switch context: `k config use-context k8s-cluster1`
2. Set namespace: `k config set-context --current --namespace=production`
3. Create deployment: `k create deploy webapp --image=nginx:1.21 --replicas=3`
4. Edit for resources: `k edit deploy webapp` (add resources)
5. Expose: `k expose deploy webapp --port=80`
6. Verify: `k get deploy,svc -n production`

**Time**: ~3 minutes

---

## 🎓 Essential Resources

### Official
- **CKA Curriculum**: https://github.com/cncf/curriculum (February 2025 version!)
- **K8s Docs**: https://kubernetes.io/docs/
- **kubectl Cheat Sheet**: https://kubernetes.io/docs/reference/kubectl/cheatsheet/

### Courses
- **KodeKloud CKA**: Updated for 2025 (highly recommended)
- **Udemy Mumshad CKA**: Also updated

### Practice
- **killer.sh**: 2 sessions with exam purchase (ESSENTIAL!)
- **KodeKloud Labs**: Lightning labs + mock exams
- **GitHub CKA Labs**: https://github.com/simonbbbb/CKA-Hand-on-lab

### YouTube (2025 Updates)
- **JayDemy CKA 2025**: Real exam questions walkthrough
- Search: "CKA 2025 February changes"

### Hands-on
- **Minikube**: Local single-node cluster
- **Kind**: Kubernetes in Docker (faster)
- **kubeadm VMs**: Practice cluster setup (3 VMs: 1 control, 2 workers)

---

## 💪 Final Pep Talk

**You're in a GREAT position**:
- ✅ You have K8s fundamentals (KCNA/KCSA)
- ✅ You have DevOps experience
- ✅ You understand the concepts

**Your ONLY challenge**:
- 🔥 Build hands-on speed
- 🔥 Practice, practice, practice

**3-4 weeks of focused practice = CKA certified**

**CKA + CKAD same day/week is SMART**:
- Overlap is ~70%
- Same format
- Same docs
- Do CKA first (harder, requires fresh mind)
- CKAD after (easier, more app-focused)

**Your sensei's promise**:
- Follow this guide
- Do the drills
- Crush killer.sh
- You'll pass CKA with 80%+

Now get to work! Come back when you're ready for CKAD guide.

**Ganbatte, gakusei!** 🥋🚀

---

*Last Updated: January 2026*  
*CKA Exam Version: February 18, 2025 update*  
*Sensei Mode: Maximum Overdrive* 💪
